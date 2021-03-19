---
layout: post
title: "Never Use Unconstrained `auto&&` or `T&&` in an Overload Set"
comments_id: 6
---

The story starts from the following code

```cpp
struct Foo{};
struct Bar{};
struct Buz{};

using MyClass = std::variant<Foo, Bar, Buz>;

int main(){
    MyClass obj{Foo{}};

    auto const r = std::visit(boost::hana::overload(
        [](const Foo&){ return 5; },
        boost::hana::always(42) // fallback
    ), obj);

    std::cout << r;
}
```

What do you think what is printed out? Yes it is 42. It is not selecting the `const Foo&` overload.

[godbolt](https://godbolt.org/z/8xv5jY)

## `std::variant` Visitor

If you are not familiar with `boost::hana`, that is not a problem. `boost::hana::overload` is almost the same as the `overload` thing that appears in every C++17 text book, but slightly better because itself is a function object and it supports C++14. `boost::hana::always` is just a function that takes an argument and returns a function that returns that argument. So the code is equivalent to

```cpp
template<typename... F>
struct overload : F...{
    using F::operator()...;
};

template<typename... F>
overload(F...) -> overload<F...>;

struct Foo{};
struct Bar{};
struct Buz{};

using MyClass = std::variant<Foo,Bar,Buz>;

int main(){
    MyClass obj{Foo{}};

    auto const r = std::visit(overload{
        [](const Foo&){return 5;},
        [](auto&&){return 42;} // fallback
    }, obj);

    std::cout << r;
}
```

[godbolt](https://godbolt.org/z/eTs7b7)

Same result: 42 is printed. So the question is, why does it not select the lambda that takes `const Foo&`?

This is because our `obj` is not `const`. We are passing a `Foo&`, and the overload that takes `auto&&` is actually a better match than the overload that takes `const Foo&` because `auto&&` does not have `const`.

If our `obj` is `const`, 5 will be printed.

```cpp
    const MyClass obj{Foo{}};
```

[godbolt](https://godbolt.org/z/bqM5EK)

So which bit in our code is wrong? 

Is it we are not declaring our `obj` a `const`? Well, in this toy example, it can be definitely `const`. But in reality, it is not always possible to make it `const` because later we might want to mutate it. 

Is the first lambda that takes `const Foo&` wrong? No. If it does not mutate `Foo`, it should take it by `const Foo&`. We should not change it to take `Foo&`.

The obvious remaining thing is the second lambda which takes `auto&&`. It matches everything and that is why we use it as a fallback. But how should we change it? We could add constraints to it but it is too much boilerplate code to write (either with concepts or with SFINAE). The easiest thing to do is to change it to take argument by `const auto&`

```cpp
[](const auto&){return 42;} // fallback
```

Now both overloads takes arguments by `const` reference. But the first overload is more specific than the second one so it will be selected. And 5 will be printed.

[godbolt](https://godbolt.org/z/deKzW4)

If the fallback needs to modify the obj, then you need to do more work

So the first take away is: never use `boost::hana::always` inside `boost::hana::overload`

## Normal Function Overload Set

The same can happen in normal function overload set.

```cpp
struct Foo{};

void call(const Foo&){
    std::cout << "call(const Foo&)\n";
}

template <typename T>
void call(T&&){
    std::cout << "call(T&&)\n";
}

int main(){
    Foo foo{};
    call(foo);
}
```

Yes it will print "call(T&&)".

This is the exact same problem. Our `foo` is not `const` and the overload that takes `T&&` will be a better match.

What is going wrong in this case?

Same argument for the `const`. It would be nice if we can declare `foo` as a `const` at the first place but it is not always possible.

Some would argue mixing functions with function templates is not a good idea. But this is very hard to avoid especially when they are declared in different headers and even more annoying, when ADL is involved.

Imagine this code

```cpp
// in lib.hpp
namespace lib{

struct Foo{};

} // namespace lib

// in client.hpp
// #include "lib.hpp"
namespace client{

void call(const lib::Foo&){
    std::cout << "call client call(const Foo&)\n";
}

void run(){
    lib::Foo foo{};
    call(foo);
    // use foo a bit more maybe call non-const functions
}

} // namespace client

// in main.cpp
int main(){
    client::run();
}
```

[godbolt](https://godbolt.org/z/ha8MTr)

Our `client::call(const Foo&)` is called and everything works, until one day, the owner of `lib.hpp` decide to add a `call` that takes `T&&`.

```cpp
// in lib.hpp
namespace lib{

struct Foo{};

template <typename T>
void call(T&&){
    std::cout << "lib::call(T&&)\n";
}

} // namespace lib

// in client.hpp
// #include "lib.hpp"
namespace client{

void call(const lib::Foo&){
    std::cout << "call client::call(const Foo&)\n";
}

void run(){
    lib::Foo foo{};
    call(foo);
    // use foo a bit more
}

} // namespace client

// in main.cpp
int main(){
    client::run();
}
```

[godbolt](https://godbolt.org/z/8n4EKn)

Now the `client::run` silently call the `lib::call(T&&)` without any compiler errors/warnings.

Are there other options? Well, if the author of `call(const Foo&)` know that there is already an overload that takes `T&&`, they could add another overload `call(Foo&)`, which does not look very nice neither.

The only thing we can do is probably to make the function template not to take unconstrained `T&&`. We can add constraints to the type `T` either with concepts or SFINAE. But this is also hard because the function template writer usually don't know the existence of type `Foo`. The constraints might not also exclude the type `Foo`.

So the take away for this case is that using unconstrained T&& judiciously.

## Constructors

The most common cases for overload sets are constructors. Because our compiler will generate a bunch of overloads.

If you are writing an implicit constructor that takes unconstrained `T&&`, you are in trouble. Consider,

```cpp
struct Foo{
    template <typename T>
    Foo(T&&){
        std::cout << "Foo(T&&)\n";
    }
    Foo(const Foo&) = default;
};

void call(Foo){
    // take Foo by copy
}

int main(){
    Foo foo{5};
    call(foo);
}
```

Unsurprisingly the output is

```cpp
Foo(T&&)
Foo(T&&)
```

Yes, our template constructor is called twice. This is because, when we call `call(foo)`, it will try to make a copy because the function is taking argument by value. Our template constructor `Foo(T&&)` is a better match than the compiler generated copy constructor, because, obviously, our input `foo` is not `const`, and copy constructor takes argument by `const Foo&`. So our `Foo(T&&)` is called when the copy happens.