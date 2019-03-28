# Fixing Reference Initialization

## 1 Motivation

### 1.1 Silent lvalue to rvalue conversion

A lot of code assumes that lvalues passed as arguments to a function will outlive the function call and thus may be stored and accessed after the function exits, while rvalues are invalid after the function exits.

For example, the standard library makes this assumption in the range library. `std::ranges::view::all` or functions returning iterators cannot be called on rvalues to warn the caller of lifetime issues. `std::cref` has a `const&&` overload to avoid binding rvalues.

Unfortunately, in a function call, an rvalue argument can bind to a const& parameter. The function may return the parameter as a `const&` and thus silently turn an rvalue reference into an lvalues reference, breaking lifetime semantics.

`std::min/max` is one group of functions that has this defect. Any typical member accessor of the form

```cpp
class Foo {
  Bar m_bar;
public:
  Bar const& get_bar() const& { // or const, same semantics
    return m_bar;
  }
};
```

has it, too.

### 1.2 No lifetime extension for xvalues

When a prvalue is assigned to a const& variable, the prvalue is lifetime-extended to the lifetime of the reference:

```cpp
Bar foo();
...
Bar const& b=foo();
```

But if the prvalue has been forwarded through a function taking its argument by rvalue reference, which returns an xvalue to the argument or a part of it, there is no lifetime extension, resulting in a dangling pointer:

```cpp
Bar const& b=std::move(foo());
```

The same problem exists when creating a `Bar&&` or `Bar const&&`, which also triggers lifetime extension for prvalues but not for xvalues.

This paper provides a starting point for a discussion on how to solve these two problems.

## 2 Idea of a Solution

What are the guarantees that various references provide?

We will leave out `volatile` for now because it is rarely used, and P1152 even proposes deprecation.

Lvalue references indicate long lifetime, rvalue reference restricted lifetime.

Non-const references can be mutated, while const references cannot.

Given these semantics and disregarding compatibility issues for now, what is the desired binding behavior for each kind of reference?

I propose treating references used in functions parameters slightly differently because the lifetime of arguments is usually at least ensured for the duration of the function call.

### 2.1 Initializing References in the Context of Function Argument Binding

#### 2.1.1 Non-Template Argument Binding

Let‘s start with binding arguments to function parameters:

```cpp
return_type foo(Bar const& b);
```

To avoid the problem described in 1.1, `const&` must only bind to lvalues. It also must not allow temporaries, because they only live for the duration of the call.

Of course, binding to rvalues as well as lvalues is often desirable. If `const&` does not have this role anymore, which kind of reference do we use instead?

```cpp
return_type foo(Bar const&& b);
```

`const&&` gives no lifetime guarantee and no mutability guarantee. Presently, only rvalues can bind to it. I propose to allow all types of expressions to be able to bind to it, adopting the semantics of present day's `const&`. A consequence of this proposal would be to use `const&&` instead of `const&` as parameter type as long as this reference does not leave the function.

```cpp
return_type foo(Bar&& b);
```

Non-`const` `&&` provides mutability, but no lifetime guarantee. Presently, only mutable rvalues can bind to it. But since `&&` provides no lifetime guarantee and no expectation of visible side effects, I propose allowing binding of lvalues unless there is a better match by creating a temporary copy. Its role would be similar to today‘s pass-by-value + move. It provides a private copy to be consumed. It would be more efficient than today’s pass-by-value because it can be passed down a call hierarchy without invoking the move constructor.

```cpp
return_type foo(Bar& b);
```

`&` keeps its current binding behavior, binding only to mutable lvalues, with temporaries not allowed.

#### 2.1.2 Template Argument Binding

```cpp
template<typename T>
return_type foo(T const&&);
```
would deduce T in the same way as T const& does now.

For

```cpp
template<typename T>
return_type foo(T&&);
```

we have two options:

##### Option 1: keep it as the universal reference.

Pro: unchanged from today

Pro: same rules as auto (see below)

Con: arguably confusing syntax: similar to non-template `Bar&&`, but very different semantics.

##### Option 2: Deduce T in the same way as T const& does now, and then create temporaries as described for the non-template case.

Pro: similar to non-template Bar&&

Con: rules different from auto (see below)

Con: need a different syntax for perfect forwarding.

Since the role of pass-by-value would be largely taken over by `&&`, for perfect forwarding, we could use the syntax

```cpp
template<typename T>
return_type foo(T t);
```

where `T` is no longer forced to be a value. This is probably better explainable than the current reference collapse. `std::forward<T>(...)` would work as before.

```cpp
template<typename T>
return_type foo(T const&);
```

would deduce `T` in the same way as `T const&` does now, but only for lvalues. For rvalues, it does not match.

```cpp
template<typename T>
return_type foo(T&);
```

would work unchanged.

### 2.2 Initializing References in Other Contexts

#### 2.2.1 Deprecate Lifetime Extension

With the introduction of move semantics, lifetime extension has become problematic:

```cpp
Bar foo();
...
Bar const& b=foo();
ConsumesBar( std::move(b) );
```

The std::move is ineffectual, Bar will not be moved from, although there is an instance of `B` created by lifetime extension that is available to be moved from. The problem is that lifetime extension is not reflected in the type system. Regardless of lifetime extension, `b`‘s type is always `const&`.

Why not modify the type depending on lifetime extension being in effect? I would like to focus on the functionality rather than the syntax, so I just pick a suggestive syntax for now (I am aware it may collide with lambdas):

```cpp
Bar [const&] b=foo();
```

If `foo()` returns an rvalue, `b` would be a value. If `foo()` returns an lvalue, `b` would be `const&`. In either case,

ConsumesBar( std::move(b) );

would do the right thing, passing `&&` or `const&&`, respectively.

If no move semantics are needed, programmers may prefer to have `const`ness in either case, for example with a syntax of

```cpp
Bar const[&] b=foo();
```

If `foo()` returns an rvalue, `b` would be of type `Bar const`.

#### 2.2.2 Initializing References with Given Single Type

##### 2.2.2.1 Lvalue References

In the absence of lifetime extension, to avoid dangling references,

```cpp
Bar const& b=...;
```

and

```cpp
Bar& b=...;
```

must only bind to lvalues (only mutable ones in case of `&`).

##### 2.2.2.2 Rvalue References

Without lifetime extension, binding any references to prvalue expressions would immediately create a dangling reference and should be forbidden.

Rvalue references do bind to xvalues.

```cpp
Bar&& b=...;
```

thus binds only to mutable xvalues.

For consistency with argument binding,

```cpp
Bar const&& b=...;
```

could also bind to lvalues because it gives the weakest guarantees, although I have no example where this is useful.

#### 2.2.3 auto

auto references work like template type deduction described in 2.1.2, with the following differences: all cases requiring the creation of temporaries are forbidden instead, and `auto&&` and `auto` are treated according to option 1 above. I do not like the inconsistency between choosing option 1 here and option 2 above, any ideas welcome.

To replace lifetime extension, the new syntax described in 2.2.1 can be extended to `auto`.

```cpp
auto [const &] a=...;
```

would work like

```cpp
auto const& a=expr;
```

if expr is an lvalue, and like

```cpp
auto a=...
```

otherwise.

```cpp
auto const [&] a=...;
```

would work like

```cpp
auto const& a=expr;
```

if expr is an lvalue, and like

```cpp
auto const a=...
```

otherwise.

### 2.3 Backward compatibility

To interoperate with existing code, new code would opt in to the new behavior using something like a `#pragma`.

Include files would need to be able to return to the setting of the inclusion site, either explicitly or implicitly with the end of the file.

Implicit return may be undesirable for some fancy uses of `#include` files. An explicit

```cpp
#pragma binding_v2(push, on)
...
#pragma binding_v2(pop)
```

may be best.

If a function declaration, definition or reference variable initialization (including members in a constructor) is in a block of code with new behavior, the new rules would apply. Function declarations and corresponding definitions are required to have the same type of behavior, either both old or both new.