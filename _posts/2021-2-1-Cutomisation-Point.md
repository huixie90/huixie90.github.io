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

This looks like that it is working. But unfortunetely, it doesn't. Because `Foo` is actually not in `user` namespace, but in `std` namespace. `my_cp(const Foo&)` won't be found by ADL as its namespace is not the same as the argument's namespace. So what can the user do? Add `my_cp` in `std` namespace? That is undefined behaviour. So with ADL there is no way to do this.

## `boost::hana` Way

`boost::hana` uses template specialisation throughout its code base. In addition to template specialisation, it uses tag dispatch as another layer of indirection to allow dispatch multiple unrelated types into the same category, and then only provide the speicialisation for that category. To simplify how template speicialisation is used, tag dispatch layer is removed in this example.

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

This works. We are not worried about opening up `std` namespace because we are opening up the algorithm namespace `my_lib`, as the author of this algorithm expects their user to open up its namespace.

## `tag_invoke`