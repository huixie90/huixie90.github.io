---
layout: post
title: "Almost Always `const auto&`"
comments_id: 4
---

Looking at the code I've been written in the past, I found that lot's of lines started with `const auto&`. Well, I am not going to debate on whether you should use `auto const&` instead.

Wait a second, I have to say something about `const west` and `east const`. One possible reason why some people prefer `east const` is that (although they won't admit this), when they start typing on the keyboard, they type the class name or the `auto` keyword first, before they finish the class name, they realise that it can be `const`, so they add `const`, without any efforts and it end up with `foo const`. If they were `const west`er, they would type `foo` first and realise it can be `const`, and they had to go back to the beginning and add `const` to make `const foo`. There are the few extra key strokes. However, the real `const west`er won't have this problem. Because they always type `const` first without even knowing what type they are going to type, and at the same time think what they are going to type next.

OK, that is enough ranting about unrelated topic. Let's go back to `const auto&`. I often write code like this

```cpp
const auto& foos = give_me_some_foos();
```

It might be just that I am lazy without thinking why I was writing `const auto&`, as it almost always works.

## `const`

There is no doubt on this one. Everybody loves `const`. There are millions talks about why you should use `const`. I just followed everybody's advice.

## `auto`

The best reference of the benefits of `auto` might be [Almost Always Auto](https://herbsutter.com/2013/08/12/gotw-94-solution-aaa-style-almost-always-auto/).

The main argument against it is the readability. In my opinion, variable and function names play more important roles than the type itself in lots of cases. Compare

```cpp
const std::vector<std::basic_string<char>, std::allocator<std::basic_string<char>>> foo = getBars();
```

to

```cpp
const auto& names = get_employee_names();
```

Is the first one more readable than the second one?

Another argument is that I need to know the type to figure out what it can do. That is the job of the IDE (clangd is getting better and better now). One would say I don't want to go to my IDE when I see a code review. I would suggest to index your code base and make a web based source code viewer in your organisation where you can navigate though the code base in a browser. This is definitely doable and scaleable with the current tools.

The biggest benefit of `auto` in my opinion is that it gives you the flexibility because you don't depend on a concrete type. If the concrete type of the right hand side changes, you don't need to change your code if the interface remains. In Java, people often use interfaces as types.

```java
Collection<Foo> foos = new CopyOnWriteArrayList<Foo>(fs);
```

So when implementations on the right hand side changes, you don't need to change the code.

Another argument is that in C++ using `auto` doesn't self document what the interface is. This is true, with concept, we can do slightly better

```cpp
const range auto& employees = get_employees();
```

We add concept `range` to make a constraint here, so that later we can iterate though it.

However, if we compare this with the Java version,

```java
Collection<? extends Workable> employees = employeeFactory.createEmployees();

for (Workable employee : employees){
    employee.work();
}
```

We are still missing a bit here. We cannot specify in C++ what the interface the element in the range has.

## `&`

I use `&` to avoid copies. This is not for performance reasons. If I change a `const auto&` to `const auto` and send a code review saying for performance reasons, the first comment will be "Have you measured"? I don't have time to measure every single decision I have made.

The main reason to use `&` here is that I don't know if the types on the right hand side is copyable. Let say the right hand side returns `const std::vector<std::unique_ptr<int>>&`, using `const auto` won't work.

Even if the right hand side is copyable (but not documented) for now, and I rely on it being copyable, I will become a problem if the maintainer of the right hand side function wants to return a non-copyable type in the future.

## Putting together `const auto&`

C++ has a feature to extend the lifetime of a temporary object if it is bond to a `const` reference. This will make `const auto&` work for a lot more cases. For example, this is going to work as well.

```cpp
std::vector<int> getInts();

const auto& ints = getInts();
use(ints);
```

One could argument that I don't need the variable at all. Yes, if you can make your expression less than one line, sure. But the fact that it extends life time is sometimes useful.

## Caveat

One caveat is that you can't call non `const` functions on it because it is `const`.

Another caveat is that with `const auto&`, the thing it references to can still change on another thread. Whereas if you do `const auto` to make a copy, it is safer. (Well, sort of, only if the copy is a deep copy).
