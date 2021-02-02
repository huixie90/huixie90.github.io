---
layout: post
title: "What If Everything Is a Customisation Point"
---

Following up my [previous post](https://huixie90.github.io/Cutomisation-Point/), there are some arguments saying that I should just write a wrapper of the `std` types. OK, let's do it.

I have the following definition

```cpp
namespace user {

using MyObject = std::variant<Foo, Bar, Buz>;

} // namespace user
```

Imagine this code is there for a while, and we have some usages of the definition. Obviously most of them are just calling `std::visit` with some visitors.

For some reasons, we need to make it an actual class today instead of a type alias to `std::variant`. Those reasons can be:

- Gain full control of the class. e.g. Not exposing some behaviours such as `std::get`
- Allow adding customisation points to some algorithms that requires ADL. If it is an alias, it is actually in `std` namespace. Adding customisation points to `std` namespace is not allowed
- Allow forward declaration. This is useful when you work in a large legacy code base. Imagine including the definition header of `Foo` will pull up a million of lines of code (I am not joking). Having a forward declaration of `MyObject` will be very useful when you don't need the full definition of it.

At the same time, while we are refactoring, we don't want to break all the code that uses this class. Let's make an attempt.

> All SFINAE and `noexcept` propagations have been omitted

```cpp
namespace user {

struct MyObject{
    /* implicit */ MyObject(Foo);
    /* implicit */ MyObject(Bar);
    /* implicit */ MyObject(Buz);

    template <typename Visitor, typename... MyObjects>
        requires std::same_as<std::decay_t<MyObjects>, MyObject> && ...
    decltype(auto) 
};

} // namespace user
```
