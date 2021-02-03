---
layout: post
title: "What If Everything Is a Customisation Point ?"
---

Following up my [previous post](https://huixie90.github.io/Cutomisation-Point/), there are some arguments saying that I should just write a wrapper of the `std` types. OK, let's do it.

Let's say I have the following definition

```cpp
namespace user {

using MyObject = std::variant<Foo, Bar, Buz>;

} // namespace user
```

Imagine this code is there for a while, and we have some usages of the alias definition. Obviously most of them are just calling `std::visit` with some visitors.

For some reasons, we need to make it an actual class today instead of a type alias to `std::variant`. Those reasons can be:

- Gain full control of the type. e.g. Not exposing some behaviours such as `std::get`. If the alias is used in public APIs to encapsulate the implementation details, it will never work. Because the clients will depend on every single observable behaviour of `std::variant`, rather than your documented API.
- Allow adding customisation points to some algorithms that requires ADL. If it is an alias, it is actually in `std` namespace. Adding customisation points to `std` namespace is not allowed. Adding behaviours to our own type is safer that adding behaviours to a type we don't own.
- Allow forward declaration `class MyObject;`. This is useful when you work in a large legacy code base. For this alias, we need to include the header `variant`, `Foo`, `Bar`, and `Buz`. Imagine including the definition header of `Foo` will pull up a million of lines of code (I am not joking). Having a forward declaration of `MyObject` will be very useful when you don't need the full definition of it.

At the same time, while we are refactoring, we don't want to break all the code that uses this class. Let's make an attempt.

> All SFINAE and `noexcept` propagations have been omitted

```cpp
namespace user {

class MyObject{

    using variant_type = std::variant<Foo,Bar,Buz>;

  public:
    template <typename T> requires 
        std::convertible_to<T&&, variant_type>
    /* implicit */ constexpr MyObject(T&& t) 
        : variant_{static_cast<T&&>(t)}{}

  private:
    variant_type variant_;

    template <typename Visitor, typename... MyObjects>
        requires (std::same_as<std::decay_t<MyObjects>, MyObject> && ...)
    friend constexpr decltype(auto) visit(Visitor&& visitor, 
                                          MyObject&&... objs){
        return std::visit(static_cast<Visitor&&>(visitor), 
                          static_cast<MyObject&&>(objs).variant_ ...);
    }
};

} // namespace user
```

We are injecting a hidden friend `visit`, expecting the clients to call `visit` with visitors as if it were a `std::variant`.

```cpp
int main(){
    constexpr user::MyObject obj{Foo{}};
    constexpr auto get_name = hana::overload(
        [](const Foo& foo) {return foo.name;},
        [](const Bar& bar) {return getName(bar);},
        [](const Buz& buz) {return buz.getName();}
    );
    constexpr auto name = visit(get_name, obj);
}
```

The interface looks almost exactly like the `std::variant`, except that the client is calling `visit` instead of `std::visit`. This is annoying, we still have to update the client code. While we are updating, we may find that some clients are using `visit` already, and that worked because of ADL (`std::visit` is in the same namespace as `std::variant`), and after our refactoring it still works because hidden friend `visit` is found for `user::MyObject`. Some clients are writing `using std::visit;` then use `visit`, which worked before and after our refactoring as well.

OK, wait a second, the two cases work for both `std::variant` and our `user::MyObject`. This sounds familiar. Yes, this looks like a basic version of customisation point. What if `std::visit` is actually a customisation point, and with improved implementations such as `tag_invoke`? Let's imagine how it might look like.

```cpp
namespace std {

inline constexpr struct visit_fn {
    template <typename Visitor, typename... Variants>
    constexpr decltype(auto) operator()(Visitor&& visitor, 
                                        Variants&& variants...) const {
        return tag_invoke(*this, 
                          static_cast<Visitor&&>(visitor),
                          static_cast<Variants&&> variants...);
    }
} visit{};

template <typename... T>
class variant {
    // some definition here

    
    template <typename Visitor, typename... Variants> 
    // SFINAE omitted 
    friend constexpr decltype(auto) tag_invoke(tag_of<visit>, 
                                               Visitor&& visitor, 
                                               Variants&&... variants){
        // implementation here
    }
};

} // namespace std
```

The public interface of `std::visit` for `std::variant` remains the same. There is one exception. As a side effect `std::visit` now becomes a function object rather than a function template. This makes it easier to pass into higher order functions, which is useful. For example,

```cpp
int main(){
    const std::vector<std::variant<Foo, Bar, Buz>> objs = ...;
    constexpr auto get_name = hana::overload(
        [](const Foo& foo) {return foo.name;},
        [](const Bar& bar) {return getName(bar);},
        [](const Buz& buz) {return buz.getName();}
    );
    const auto names = hana::transform(objs, hana::partial(std::visit, get_name));
}
```

Anyway that is just a side effect. Let's focus on our `user::MyObject`.

```cpp
namespace user {

class MyObject{

    using variant_type = std::variant<Foo,Bar,Buz>;

  public:
    template <typename T> requires 
        std::convertible_to<T&&, variant_type>
    /* implicit */ constexpr MyObject(T&& t) 
        : variant_{static_cast<T&&>(t)}{}

  private:
    variant_type variant_;

    template <typename Visitor, typename... MyObjects>
        requires (std::same_as<std::decay_t<MyObjects>, MyObject> && ...)
    friend constexpr decltype(auto) tag_invoke(tag_of<std::visit>,
                                               Visitor&& visitor, 
                                               MyObject&&... objs){
        return std::visit(static_cast<Visitor&&>(visitor), 
                          static_cast<MyObject&&>(objs).variant_ ...);
    }
};

} // namespace user
```

Now we can call `std::visit` on our type as if it were a `std::variant`. Is it useful? Well it is useful in this particular refactoring example. One could argue the refactoring can be avoided by not using `std::variant` in the API at the first place. But that argument cannot convince me completely as what it suggested is that one should never use any types that they don't own in public APIs, including `std::variant`, `std::optional`, `std::vector`, `std::string`, ...

If we continue explore the possibility of customisation points, `std::variant` is just a start. Next obvious thing is `std::optional`. It seems that we can never add customisation points to the API because all APIs are member functions. There are two things which I think make `optional` an `optional`: `operator bool` and `operator*`. Perhaps we can have these two as customisation points

- `is_just := Optional<T> -> bool`
- `get_just := Optional<T> -> T`

The downside is that this will make the API harder to use.

Next let's look at `std::vector`. The first impression might be, omg, the interface is too rich and it is impossible to make all of them customisation points. Well, look at `boost::graph` library, you will find that this is not impossible. Lots of operations in `std::vector` are actually already customisation points, such as `ranges::begin`, `ranges::end`, `ranges::size`, etc ... We can start with simple concepts, such as `range`. And then adding more behaviours to it and make it `forward_range`, `random_access_range`, `container`, etc ...

What if everything is a customisation point? I don't know. One thing I know is that this will never happen.
