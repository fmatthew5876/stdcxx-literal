String Literals should be rvalues
==========================================

* Document Number: XXXXX
* Date: 2020-01-01
* Programming Language C++
* Reply-to: Matthew Fioravante <fmatthew5876@gmail.com>

The latest draft, reference header (incomplete), and links to past discussions on github:

* https://github.com/fmatthew5876/stdcxx-literal

Introduction
=============================

The objective of this proposal is to provide a way to discern string literals from other array and string types.
Our proposed solution is to change the type of string literal expressions such as `"Hello World"` from
`const char(&)[N]` to `const char (&&)[N]`.

Impact on the standard
=============================

This proposal is a language level change. It breaks the API for some rare edge cases but does not break API for
idiomatic uses of string literals. This proposal does not affect C compatibility.

A detailed explanation of the API consequences of this proposal are given later in the paper.

Motivation
================

In high performance, low latency systems we have natural contention between needing to be fast but also
needing to be able to log information so we can debug and introspect our systems. These goals are always at
odds, as any logging code added to the fast path will consume CPU cycles that could have been used for
the primary business logic.

In many domains, running these systems with no introspection at all is not feasable due to the difficulty
and cost of debugging production issues. Many applications, such as low latency trading strategies in the
financial domain need to provide feedback to the user so they can observe the behavior and tune the
application as real world conditions change. The scope is not just limited to such applications. Nearly
every application uses some kind of logger, even if it's using `std::cout` directly. Therefore providing
language level enhancements to make building high performance loggers safer and easier is an important goal.

Logging fixed width types such as integers and floating point numbers is pretty straightforward. One can
simply copy them off the hot path to a buffer and let a background logger serialize them. The difficulty comes in when
one has to deal with variable width strings. Some would argue that all logging should be structured and
never use strings. These approaches have their own trade offs and aren't universally applicable. We will not
discuss structured logging approaches any more in this paper.

While we could go onto the various approaches of dealing with string data, there is one class of strings
where we have a very easy way to both get variable width string data and maximum efficiency. Those would be
native string literals like `"Hello World"`. String literals are very common in logging statements, and will
be even moreso with the addition of \[[fmtlib](#fmtlib)\] to the standard in C++20 as `std::format`.

String literals are compile time constants with infinite lifetime.
Because they are `constexpr` there are no initialization or shutdown race conditions with other singleton
objects. Because of the lifetime guarantee, we never need to copy them and we can pass them by reference using
pointers. This of course falls out naturally from the decay behavior inherited from C.

Going back to our logging example, this means that for string literal types the most efficent and convenient
way to ship them to a logging thread is to decay and store a pointer. This is clearly more efficient than copying
the bytes directly for strings of length greater than `sizeof(const char*)`, which is the majority in many
applications. It also uses less memory, even in many cases where the length of the string is smaller than
word size, as the structures we would end up storing this data will likely have to align themselves out to
word or half word size anyway.

Storing a pointer means our storage mechanism can stay in the fixed size domain. This is important because
dealing with fixed size structures is an order of magnitude simpler than dealing with variable length data.
Storing variable length data often requires either templates or complex type erasure techniques.

So we conclude we would like to optimize string literals by storing pointers instead of copying the actual bytes.
In order to achieve this, we need to be able to reliably discern string literals from other string types and
character arrays, and herein lies the problem.


Design Goals and Scope
========================

The goal of this paper is simple:

Allow the programmer to capture a string literal expression like "Hello World" so that they may handle it
differently than other string or character array types.

A secondary goal which falls out of the proposed solution is to simplify the rules of C++ by making all
literal expressions be rvalues.

Analysis of literal types
==============================

First of all, what is a string literal?

```
std::is_same<const char(&)[4],decltype("foo")>::value;
```

This is interesting, because it is different than all other literals. For example:

```
std::is_same<std::string,decltype("foo"s)>::value;
std::is_same<int,decltype(1)>::value;
std::is_same<float,decltype(1.0f)>::value;
```

Presumably, the difference is due to the fact that C arrays cannot be copied, and so in order to make such an
expression assignable, a reference type had to be used instead of a value. In C++ pre-11, all we had were
lvalue references.

There is another difference here.

```
template <typename T> void f(T&);

// Ok
f("foo");

// Compile error
f(1);
```

Unlike all other literal types, string literals are lvalues. This could cause confusion, but admittedly since
string literals are also const, and const lvalue references can also bind to rvalues, it rarely shows up in
practice. In fact the author after being a C++ developer for nearly 2 decades did not know of this difference
until investigating the details of this proposal.

```
template <typename T> void f(const T&);

// Ok
f("foo");

// Ok
f(1);
```

Analysis of the problem of capturing string literals
====================================================

Now that we understand what string literal expressions are, how do we go about capturing them?

We can capture a string literal by array reference like this:

```
template <size_t N> void f(const char (&)[N]);

// Ok, deduces const char(&)[4]
f("foo");
```

We can also capture them using a pointer.

```
void f(const char*);

// Ok, decays to pointer
f("foo");
```

If both are provided, the pointer variant is prefered because it is not a template.

```
template <size_t N> void f(const char (&)[N]);
void f(const char*);

// Ok, decays calls f(const char*)
f("foo");
```

If both are provided and both are templates or both are not templates, we get a compiler error due to
ambiguous overload.

```
template <size_t N> void f(const char (&)[N]);
template <typename T> void f(T*);

void g(const char*);
void g(const char(&)[4]);

// Error ambigous overloads
f("foo");

// Error ambigous overloads
g("foo");
```

So we have only 2 viable options here, either capture by `const char*` or write a template and capture by
`const char(&)[N]`.

Capturing by pointer is obviously too wide. A `const char*` could be anything including something
allocated by `malloc`, a `std::string::c_str()` call, or any array which would also be a candidate for
the reference overload. It also captures any non-const `char*` as well.

Capturing by `const char(&)[N]` is much better, and in practice this is the idiomatic way most programmers
try to capture string literals. This is the best and only viable solution we have today. However,
this solution has big problems, namely:

```
// Assume l has infinite lifetime
template <size_t N>
void capture_literal(const char (&l)[N]);

void f() {
   char buf[32] = /* stuff */;
   // Crash!
   capture_literal(buf);
}
```

Our technique fails us here, because it captures a local C array and then treats it like a literal. This
scenario is pretty common. Here is an idiomatic one we often see

```
// Fancy logger which captures const char(&)[N] and stores them as pointers, assuming infinite lifetime.
template <typename... Args>
void fancy_logger(Args&&... args);

void do_file_io() {
   char path[] = "/some/path/to/file";
   int fd = ::open(path);
   if (fd < 0) {
      // Oops!
      fancy_logger("open({}) failed : {}", path, strerror(errno));
   }
```

The compiler is powerless to provide any help to us here. There is no error or even a warning. Even worse,
the bug happens on a error handling path, which is likely to not be exercised in testing. This bug
will likely show up in rare case in production and either result in a crash or corrupted log message or both.

There is a half-workaround to this problem. When building interfaces which attempt to capture string literals,
one can add some protection by adding both const and non-const overloads.

```
// Treat as a character array and copy
template <size_t N> void logger(char (&)[N]);
// Treat as a string literal and decay
template <size_t N> void logger(const char (&)[N]);

// Ok - decay optimize
logger("foo");

// Ok - copy
void do_file_io1() {
   char path[] = "/some/path/to/file";
   logger(path);
}

// Not Ok
// Maybe ok on your platform if the implementation stores the string in permanent storage.
void do_file_io2() {
   const char path[] = "/some/path/to/file";
   logger(path);
}
```

This is the best we have with the current C++ language for writing an overload which captures a string literal.
The solution is clearly lacking.

Proposed Solution
=================

We propose to continue down the rabbit hole followed in the previous section one more step, by changing the type of
string literal expressions to rvalue references.

```
std::is_same<const char(&&)[4],decltype("foo")>::value;
```

If we made this change to the language, now the idiomatic way to capture a literal would be:

```
template <size_t N> void logger(const char (&&)[N]);
```

Yes, that is a const rvalue reference, a strange beast that is almost never seen in real code and nearly useless,
making it a great candidate for our use case.


```
// Treat as a character array and copy
template <size_t N> void logger(const char (&)[N]);
// Treat as a string literal and decay
template <size_t N> void logger(const char (&&)[N]);

// Ok - decay optimize
logger("foo");

// Ok - copy
void do_file_io1() {
   char path[] = "/some/path/to/file";
   logger(path);
}

// Ok - copy
void do_file_io1() {
   const char path[] = "/some/path/to/file";
   logger(path);
}

// Ok - decay optimize
void do_file_io2() {
   const char path[] = "/some/path/to/file";
   logger(path);
}

// Not ok, but nobody does this
void do_file_io2() {
   const char path[] = "/some/path/to/file";
   logger(std::move(path));
}
```

As we can see this solution is not achieving compile time perfection. However, given the fact that actual use
of `const char (&&)[N]` is meaningless in current code, this is in the author's opinion a suitable compromise
and clearly better than the status quo. Finally, we gain another benefit in simplifying the language as now all
literal expressions are rvalues.

Analysis of API breakage
========================

Clearly, this change doesn't come for free. Any code depends on the exact type of string literal expressions,
(such as `std::is_same<const char(&)[4],decltype("foo")>::value` shown above), or depends on the lvalueness of
these expressions will break. I would argue that both of these are extremely rare and if someone is really
exploiting these behaviors they probably know enough to be able to fix them for the new standard.

The good news is that all of the idiomatic use cases still work.

```
template <size_t N> void f(const char (&)[N]);
template <size_t N> void f(const char (&&)[N]);
```

In our example above, the rvalue overload is preferred while the lalue overload is still a viable candidate.
All old code using `const char(&)[N]` to capture literals still works.

```
template <size_t N> void f(const char (&)[N]);
template <size_t N> void f(const char (&&)[N]);
template <typename T> void f(T*);
```

In this example, a call to `f` with a literal would still be ambiguous as before.

```
template <size_t N> void f(const char (&&)[N]);
void f(const char*);
```

And finally, this still behaves the same. The `const char*` overload is still preferred due to it not being a template.

Alternative Solutions
=====================

Solution 1. Change string literals to be a new template type
-----------------------------------------------------------

We could add a unique templated type to the language, call it `std::basic_string_literal<char,N>`
and have string literal expression return this type.

```
std::is_same<std::basic_string_literal<char,N>,decltype("foo")>::value;
```

This type would need to contain a lot of compiler "magic" in order to work. Namely it would need to:

* Be an rvalue reference or a value, making the expression itself an rvalue.
* Decay to `const char*` pointer
* Convertable to `const char(&)[N]`
* Maintain same overload behavior outlined above.
* Not require standard library header inclusions for every file containing a string literal expression

That is indeed a lot of magic and a lot of unknowns about how such a thing could be specified or even possible.
It also causes the same API breakage as our proposed solution, and perhaps even more as the expression isn't even
a character array type anymore.

This solution however would solve our primary goal and be compile time perfect. In that one could write:
```
template <typename Char, size_t> void logger(const std::basic_string_literal<Char,N>&)
```

And be guaranteed this will only bind to string literal expressions and nothing else.

Solution 2. Add a special type which can overload on string literals
--------------------------------------------------------------------

This is similar to (1). Except that now instead of changing the types of literals themselves,
we just add a new type to the standard library which has some magic help from the compiler
to overload on string literal expressions.

```
template <typename Char, size_t N>
void f(std::basic_string_literal<Char,N>);

// Ok - with the help of some non-standard compiler magic
f("foo");

void g() {
   const char buf[] = "foo";
   // Compile error const char[4] is not convertable to string_literal
   f(buf);
}
```

To be clear, this is not a binding to `const char (&)[N]`. It is a new and unprecedented mechanism
for this type to bind to string literal expressions in addition to the existing rules in C++ for overloads.

This would achieve our primary goal of adding the ability to detect string literals with perfect compile time accuracy.
It also would be a pure addition and not break the API.

This would require some compiler help to actually detect literal expressions from others and perform the correct overload.
It is unclear how this rule would be worded or implemented in the standard. It also doesn't achieve the goal of making
all literals rvalues.

If the committee is excited about this approach, we would be willing to change course and go in this direction. As for
all the alternatives presented this is the one we feel is the most desirable.

In addition with the goal of making all literals rvalues, we could still adopt the proposed solution in this paper of changing
string literals to `const char (&&)[N]` and simulateously propose this variant of `basic_string_literal` in addition to further
harden the primary goal of this paper.

Solution 3. Don't use native string literals
--------------------------------------------

One can add this decay optimization today by simply using `string_view`

For example:
```
using namespace std::literals;

log("This"sv, "is"sv, "extremely"sv, "cumbersome"sv);
log("Oops I forgot the sv, not this string will be copied and pessimized");
```

This has a lot of problems. First, it requires `using namespace std::literals` everywhere which is cumbersome at best and not
viable at all for global scope in header files.

This solution requires us to remember to write the `sv` suffix on every string literal we pass to our logger. Any large code
base using a logger in this way must train their entire org to do this correctly. We can all predict with certainty that this
will be a constant source of bugs as programmers will, as a rule, forget to add the `sv` suffix. The `sv` suffix could be enforced
by adding overloads disable `const char (&)[N]` at compile time, and I would argue anyone going for this approach *must* do that.
But then we lose the ability to log character arrays and still have to pay the syntatic overhead of adding this useless `sv` suffix
everywhere just to work around language limitations which could be fixed.

The author is vehemently against these types of approaches for the above reasons. We should improve string literals at the
language level to empower developers to write more efficient and safer libraries.

Acknowledgments
====================

References
==================

* <a name="fmtlib"></a>[fmtlib] GitHub: fmtlib/fmt A modern formatting library Available online at <https://github.com/fmtlib/fmt>
