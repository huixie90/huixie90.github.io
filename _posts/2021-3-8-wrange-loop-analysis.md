---
layout: post
title: "-Wrange-loop-analysis"
comments_id: 5
---

If you are building your projects on multiple platforms, you probably noticed that on mac with XCode 12, `-Wrange-loop-analysis` is set on by default, which might cause some code to break if you use `-Werror`.

## What is `-Wrange-loop-analysis` ?

`-Wrange-loop-analysis` is an umbrella warning flag that controls two warnings:

- `-Wrange-loop-construct`
- `-Wrange-loop-bind-reference`

see clang documentation [here](https://clang.llvm.org/docs/DiagnosticsReference.html#wrange-loop-analysis)

### `-Wrange-loop-construct`

This is a useful warning. It catches unnecessary copies. For example,

```cpp
std::vector<std::string> vec{/*...*/};
for(const auto s : vec){
    // use s
}
```

The compiler will warn you

```
loop variable 's' creates a copy from type 'const std::__cxx11::basic_string<char>' [-Wrange-loop-construct]
    for(const auto s: vec){
                   ^
<source>:8:9: note: use reference type 'const std::__cxx11::basic_string<char> &' to prevent copying
    for(const auto s: vec){
        ^~~~~~~~~~~~~
                   &
```

[Godbolt](https://godbolt.org/z/76vb1e)

The compiler is suggesting that you might want a reference `const auto&` to prevent a copy as you are not modifying the original object in the `vector`. Notice that without `const`, if you have just

```cpp
for (auto s : vec){
    // use s
}
```

the compiler cannot make the suggestion. Because you could modify the copy `s` even if you are calling all `const` member functions. (In theory you could `const_cast` away `const` and modify it). However, `const_cast` away `const` object is undefined behaviour so the compiler is happily suggesting you to use a reference when you use `const`.

This is a useful warning and I have no doubt with it.

## `-Wrange-loop-bind-reference`

This is a less useful warning. It warns you if you bind a `const` reference to a temporary object. Consider

```cpp
const auto& obj = getTemporary();
```

This is perfectly legal C++ code. `const` reference can extend the life time of a temporary object. If this is happening inside the range-for loop. The compiler will warn you. A typical example is

```cpp
std::vector<bool> vec{/*...*/};
for(const auto& b : vec){
    // use b;
}
```

You will see the warning:

```
warning: loop variable 'b' binds to a temporary value produced by a range of type 'std::vector<bool>' [-Wrange-loop-bind-reference]
    for(const auto& b: vec){
                    ^
<source>:8:9: note: use non-reference type 'std::_Bit_reference'
    for(const auto& b: vec){
        ^~~~~~~~~~~~~~
```

[Godbolt](https://godbolt.org/z/fbGn6z)

This is because `vector<bool>`'s iterator is a proxy iterator and its dereference operator returns a proxy object by value. Here we are trying to bind a `const` reference to this temporary proxy object, which is fine in my opinion.

## Problem of `-Wrange-loop-bind-reference`

Sometimes it creates problems. Consider the following example,

```cpp

// code can be in another place
auto getEmployees(){
    return std::vector<Employee>{/*...*/};
}

// caller
for (const auto& employee : getEmployees()){
    // use employee;
}

```

[Godbolt](https://godbolt.org/z/aKx71P)

This compiles fine and it is perfectly valid code. The caller of `getEmployees` does not care about the return type of that function. All it expects is a range of `Employee`s. It does not want to copy the object, nor does it want to modify the object, so it uses `const auto&`.

So far so good, but one day the implementation of `getEmployee` gets changed. Instead it returns a generator range.

```cpp
// code can be in another place
auto getEmployees(){
    return ranges::views::iota(25,30) 
    | ranges::views::transform([](auto i){
        return Employee{i};});
}

// caller
for (const auto& employee : getEmployees()){
    // use e;
}
```

Suddenly, the caller get a compiler warning (or error if you have `-Werror`)

```
loop variable 'employee' binds to a temporary value produced by a range of type 'ranges::transform_view<ranges::iota_view<int, int>, (lambda at <source>:11:32)>' [-Wrange-loop-bind-reference]
    for(const auto& employee: getEmployees()){
                    ^
<source>:16:9: note: use non-reference type 'Employee'
    for(const auto& employee: getEmployees()){
        ^~~~~~~~~~~~~~~~~~~~~
```

[Godbolt](https://godbolt.org/z/79Mxdv)

The warning is right, the iterator's dereference operator returns a temporary `Employee`, but I bind a `const` reference to it. Well, but is that a problem? Extending the life time of the temporary by the `const` reference is a language feature. If I follow the compiler's suggestion, remove the reference, I am coupled to this `getEmployee` implementation. If in the future it returns a `vector` again, my code will be broken again because I will be creating unnecessary copies.

## Solution to this problem

In Arthur O'Dwyer's [article](https://quuxplusone.github.io/blog/2020/08/26/wrange-loop-analysis/), he suggested that using `auto&&` solves the problem. This is probably what I am going to use in the future given that this compiler warning will keep annoying me. But this is not a perfect solution, because I lose `const`. I could potentially call a non-const member function accidentally. My reader isn't aware that I really don't need to modify the object because of the missing `const`.

Can I just get my `const auto&` back please?

## When this warning is useful

Consider the following example,

```cpp
using ranges::views::zip;

std::vector<int> ages{1,2,3};
std::vector<std::string> names{"Alice","Bob","Charles"};

for(const auto& [age, name] : zip(ages, names)){
    age = 0;
    name = "null";
}
```

[Godbolt](https://godbolt.org/z/1zPjaT)

It seems that you have `const` reference to `ages` and `names`, but it is not. It is just a `const` reference to a hidden temporary `ranges::common_pair`. The `ranges::common_pair` itself contains non-const reference to elements. In the end, we are able to modify the original objects.

In this case, the compiler gives you a warning

```
loop variable '[age, name]' binds to a temporary value produced by a range of type 'zip_view<all_t<std::vector<int, std::allocator<int>> &>, all_t<std::vector<std::__cxx11::basic_string<char>, std::allocator<std::__cxx11::basic_string<char>>> &>>' (aka 'zip_view<ranges::ref_view<std::vector<int, std::allocator<int>>>, ranges::ref_view<std::vector<std::__cxx11::basic_string<char>, std::allocator<std::__cxx11::basic_string<char>>>>>') [-Wrange-loop-bind-reference]
    for(const auto& [age, name] : ranges::views::zip(ages, names)){
                    ^
<source>:15:9: note: use non-reference type 'ranges::common_pair<int &, std::__cxx11::basic_string<char> &>'
    for(const auto& [age, name] : ranges::views::zip(ages, names)){
        ^~~~~~~~~~~~~~~~~~~~~~~~~
```

In this case, the compiler at least asked you to have a second look of your code. I don't know the best solution for this case. What do you think?