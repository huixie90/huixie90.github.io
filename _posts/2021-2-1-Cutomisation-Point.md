---
layout: post
title: "Customisation point for types (or aliases to) in `std` namespace"
---

There are lots of ways of doing customisation points in C++. What is the best way to do customisation points to allow types (or aliases to types) in `std` namespace? Should we ever allow it?

Let's say we are writing a generic algorithm called `my_algo`, which calls a customistion point function `my_cp`. Let's see how we can (or cannot) do it in different ways.

To simplify the code, all SFINAE and `noexcept` propergations have been omitted.

## ADL

This is probably the worst option. But let's use it as a baseline. First, we have our algorithm header `my_algo.hpp`

```cpp
namespace my_lib {

template <typename T>
auto my_algo(const T& t){
    // code here
    decltype(auto) r = my_cp(t); // ADL call to my_cp
    // more code here
    return something;
}

} // namespace my_lib
```

The algorithm makes an unqualified call to the customisation point function `my_cp`. It requires its users to provide the definition of `my_cp` though ADL.

Now we have a user to use the algorithm with a type `Foo`. They may or maynot own this type. Anyway, they might try to add the customisation point `my_cp` to the namespace of `Foo` to enable ADL.

```cpp
namespace user {

using Foo = std::optional<int>;

int my_cp(const Foo& foo){
    // some code here
    return foo ? *foo : 42;
}

} // namespace user

int main(){
    user::Foo foo{5};
    const auto result = my_lib::my_algo(foo);
    // doesn't work
}

```

This looks like that it is working. But unfortunetely, it isn't. Because `Foo` is actually not in `user` namespace, but in `std` namespace. `my_cp(const Foo&)` won't be found by ADL as its namespace is not the same as the argument's namespace. So what can the user do? Add `my_cp` in `std` namespace? That is an undefined behaviour. So with ADL there is no way to do this.

## `boost::hana` Way

`boost::hana` uses template specialisation throughout its code base. In addition to template specialisation, it uses tag-dispatch as another layer of indirection to allow dispatching multiple unrelated types into the same category, and then only provide the speicialisation for that category. To simplify how template speicialisation is used, tag-dispatch layer is removed in this example.

Let's look at the algorithm header `my_algo.hpp` first.

```cpp
namespace my_lib {

template <typename T, typename = void>
struct my_cp_impl : hana::default_ {
    static auto apply(const T&) = delete;
};

inline constexpr struct my_cp_fn {
    template <typename T>
    decltype(auto) operator()(const T& t) const{
        using impl = my_func_impl<std::decay_t<T>>;
        return impl::apply(t);
    }
} my_cp{};

template <typename T>
auto my_algo(const T& t){
    // code here
    decltype(auto) r = my_cp(t);
    // more code here
    return something;
}

} // namespace my_lib
```

First of all, regardless what it is trying to do, `my_cp` becomes a function object, which can be passed into higher-order functions. This is already a big win as it enables functional programming without having to write lots of lambdas.

Let's look at how it works. The function object `my_cp` calls the static function `my_cp_impl<T>::apply`. But default, this static function is declared `delete`. So the user must specialise the template `my_cp_impl` to supply the implementation of the customisation point.

```cpp
namespace user {

using Foo = std::optional<int>;

} // namespace user

namespace my_lib{

template <>
struct my_cp_impl<user::Foo>{
    static int apply(const user::Foo& foo){
        return foo ? *foo : 42;
    }
};

} // namespace my_lib

int main(){
    user::Foo foo{5};
    const auto r = my_lib::my_algo(foo);
}
```

This works. We are not worried about opening up `std` namespace because we are opening up the algorithm namespace `my_lib`, as the author of this algorithm expects their users to adding behaviours for their types in the algorithm namespace.

## `tag_invoke`

`tag_invoke` becomes a very popular topic recently. It is an improvement to the niebloids. In all other presentations, `tag_invoke` itself is a function object. To simplify our example, I will make `tag_invoke` itself an ADL function call.

First, let's make a small utility to get the type of an object.

```cpp
namespace ti {

template <auto& obj>
using tag_of = std::decay_t<decltype(obj)>;

}
```

Now let's look at the algorithm's header.

```cpp
namespace my_lib {

inline constexpr struct my_cp_fn {
    template <typename T>
    decltype(auto) operator()(const T& t) const {
        return tag_invoke(*this, t);
    }
} my_cp{};

template <typename T>
auto my_algo(const T& t){
    // code here
    decltype(auto) r = my_cp(t);
    // more code here
    return something;    
}

} // my_lib
```

Similar to the `hana` way, `my_cp` is a function object, which is much nicer than a function overload set or a function template, which you can pass it around. Its `operator()` does nothing, but call `tag_invoke` with `*this` and the argument. It expects the user to define a function `tag_invoke` that takes `my_cp_fn` and the argument `T` as a customisation point.

Now let's look at how the user would use it.

```cpp
namespace user {

using Foo = std::optional<int>;

// we cannot add tag_invoke in this namespace

} // namespace user
```

We have the same problem with ADL, we cannot add customisation points inside `user` namespace because our type `Foo` is actually not in `user` namespace but `std` namespace. And obviously we cannot add anything to `std` namespace. But as [Arthur O'Dwyer](https://quuxplusone.github.io/blog/) pointed out in this [Stack Overflow answer](https://stackoverflow.com/a/65331023/10915786), the customisation point now takes two arguments, if we cannot put it in the namespace of our argument, we can put it into the namespace of the other argument `*this`, which is of type `my_cp_fn`, which is in `my_algo` namespace.

```cpp
namespace my_algo {

int tag_invoke(ti::tag_of<my_cp>, const user::Foo& foo){
    return foo ? *foo : 42;
}

} // namespace my_algo
```

This works. Although we can't add `tag_invoke` in `user` namespace, or in `std` namespace, we can still add it in the namespace `my_algo`, where `my_cp` lives.

## Template specialisation or `tag_invoke`?

People claim that the template specialisation approach is too verbose. When you define a type, instead of provide the customisation point as a hidden friend inside the class, you have to close the class curly brace, close the namespace curly brace, and open up the algorithm's namespace.

But on the other hand, I think it is most flexible. What do you think?

## Should we do it?

As [Arthur O'Dwyer](https://quuxplusone.github.io/blog/) pointed out in this [Stack Overflow answer](https://stackoverflow.com/a/65331023/10915786), adding customisation points to types we don't own, such as `std::optional` is the original sin. This can easily lead to ODR violation as other people can use the same type for something else. To make it safe, the user need to wrap the type so he can have full control of, and other people won't be using it for other purposes.

In my example, it is `std::optional<int>`. It will lead to ODR violation at some point, because it is type that everyone will use. But in the real world, it won't be `int`. It can be some custom types. The class template in `std` namespace is not only `optional`, it can be anything, e.g. `variant`

There are different senarios.

### We own the types

```cpp
namespace user {

struct Foo {
    // some definition here
};

struct Bar {
    // some definition here   
}

struct Buz {
    // more definitions
};

using MyObject = std::variant<Foo, Bar, Buz>;

} // namespace user
```

I am using `MyObject` with visitors thoughout the code base and I have control of `Foo`, `Bar`, and `Buz`. And now I'd like to use it in a generic algorithm `my_algo`. Nothing should stop me directly using this alias in the algorithm. Indeed, ADL in this case will just work because the template parameters' namespaces are also in the ADL namespace set.

```cpp
namespace user {

int tag_invoke(tag_of<my_lib::my_cp>, const MyObject& obj){
    // ...
}

} //namespace user

int main(){
    user::MyObject obj{...};
    my_lib::my_algo(obj); // it just works
}
```

### We don't own the types

```cpp
namespace team1 {

struct Foo {
    // some definition here
};

} // namespace team1

namespace team2 {

struct Bar {
    // definition
};

} // namespace team2

namespace team3 {

struct Buz {
    // definition
};

} // namespace team3

namespace user {

using MyObject = std::variant<team1::Foo, team2::Bar, team3::Buz>;

} // namespace user
```

Things get trickier. Let's say I've already used `MyObject` thoughout my code base (with `std::visit` everywhere). Wrapping it into my own type and update the usage is not that easy. If we'd like to add customisation point to `my_cp`, where should we add it? Options are:

- `user` namespace. This won't work because `MyObject` is in `std` namespace
- `std` namespace. This is no no
- `team1` namespace. This will work, but why choose this one?
- `team2` namespace. This will work, but why choose this one?
- `team3` namespace. This will work, but why choose this one?
- `my_algo` namespace. This will work because the `my_cp_fn` is in `my_algo` namespace

So it seems that the reasonable choice is `my_algo` namespace

```cpp
namespace my_algo{

int tag_invoke(tag_of<my_cp>, const user::MyObject& obj){
    // ...
}

} // namespace my_algo
```

Now the question is "Is it safe to do so"?  

The argument against it is that it is possible that another person create the same `variant` and provide its own customisation point for it and we have ODR violation. Here is an example, which I shamelessly copied from the [stack overflow answer mentioned earlier](https://stackoverflow.com/a/65331023/10915786). One person can do this and use it happily.

```cpp
using IntSet = std::set<int>;
template<> struct std::hash<IntSet> {
    size_t operator()(const IntSet& s) const { return s.size(); }
};
```

At the same time, their colleague does this:

```cpp
using MySet = std::set<int>;
template<> struct std::hash<MySet> {
    size_t operator()(const MySet& s, size_t h = 0) const {
        for (int i : s) h += std::hash<int>()(i);
        return h;
    }
};
```

Boom. ODR Violation.

OK. But on the other hand, unnlike `std::optional<int>` (or `std::set<int>`), `MyObject` is a `variant` of specific set of 3 different types. So the question really is, can I claim the ownership of this `variant`? I tend to believe that I can claim the ownership. I think if we are going down the ODR violation route, nothing can stop ODR violation. Even if you write a wrapper to the `std::variant`, and provide the customisation point for your wrapper somewhere (and possibly in a cpp file where you call the algorithm). Another person can still include your wrapper header and add a customisation point in his own cpp file. Boom, ODR violation.

Working in a large code base with tens of millions of loc, one can only own a handle of types and uses a large number of other people's types. If we are going to wrap every single other people's class, it is going to make our already bloated code base even more bloated. One thing nice about generic programming and customisation point is that you can take any types and add behaviours to it. If we are going to wrap every single class we are using, it will become identical to the tranditional Java OO style.

```cpp
namespace user {

struct MyObject{

    /* implicit */ MyObject(team1::Foo);
    /* implicit */ MyObject(team2::Bar);
    /* implicit */ MyObject(team3::Buz);

    std::variant<team1::Foo, team2::Bar, team3::Buz> obj_;

    friend int tag_invoke(tag_of<my_lib::my_cp>, const MyObject& obj){
        // ...
    }

    // attempt to make it like a variant except it doesn't work
    template<typename Visitor, typename... Obj>
    friend decltype(auto) visit(Visitor&& v, Obj&&... obj){
        return std::visit(v, static_cast<Obj&&>(obj).obj_...);
    }

};

} // namespace user
```

Does this look familar? Yes, it is just a wrapper. And it looks similar to the Java code here except that it doesn't work

```java
public class MyObject implements my_cpable, visitable{
    private final visitable fVariant;

    @override
    public void visit(Visitor v){
        fVariant.visit(v);
    }

    @override
    public int my_cp(){
        // ...
    }
}
```
