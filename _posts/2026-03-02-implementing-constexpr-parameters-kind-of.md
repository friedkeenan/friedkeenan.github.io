---
title: "Implementing Constexpr Parameters Using C++26 Reflection (Kind Of)"
category: C++
tags:
    - c++
    - c++26
    - reflection
    - friend-injection

default_lang: cpp
---

{:.prompt-info}
> This blogpost is refurbished from [a Reddit post I made](https://www.reddit.com/r/cpp/comments/1rjhg4g/implementing_constexpr_parameters_using_c26/) shortly before creating this blog.

First off, here's the [compiler explorer link showing it off](https://compiler-explorer.com/z/6nWd1Gvjc).

The fundamental mechanic behind how it works is using `std::meta::substitute` to trigger friend-injection, which accumulates state and associates a template parameter with a value passed as a parameter to a function.

For instance, take the following function:

```cpp
template<const_param First = {}, const_param Second = {}>
auto sum(decltype(First), decltype(Second), int x) -> void {
    if constexpr (First.value() == 1) {
        std::printf("FIRST IS ONE\n");
    }

    if constexpr (Second.value() == 2) {
        std::printf("SECOND IS TWO\n");
    }

    std::printf("SUM: %d\n", First.value() + Second.value() + x);
}
```

This function `sum` can be called very simply like `sum(1, 2, 3)`, and is able to use the values of the first and second parameters in compile-time constructs like `if constexpr`, `static_assert`, and splicing using `[:blah:]` (if a `std::meta::info` were passed).

The third parameter, in contrast, acts just like any other parameter, it's just some runtime `int`, and doesn't need a compile-time known value.

## ...What Is Even Going On Here?

Let me try my best to explain. It's contrived, so please bear with me.

What precisely happens when one calls `sum(1, 2, 3)` is first that the `First` and `Second` template parameters get defaulted. If we look at the definition of `const_param`:

```cpp
template<typename = decltype([]{})>
struct const_param {
    /* ... */
};
```

Then we can see that the lambda in the default for the template parameter keeps every `const_param` instantiation unique, and so when the `First` and `Second` template parameters get defaulted, they are actually getting filled in with distinct types, and not only distinct from each other but, critically, distinct from each call-site of `sum` as well.

Now at this point, a defaulted `const_param` has no value associated with it. But `const_param` also has a `consteval` implicit constructor which takes in any value, and in this constructor, we can pull off the friend-injection with the help of `std::meta::substitute`.

***

If you don't know what friend-injection is, I'm probably the wrong person to try to explain it in detail, but basically it allows us to *declare* a `friend` function, and critically we get to delay *defining* the body of that function until we later know what to fill it in with. And the way we later define that body is by instantiating a class template, which then is able to define the same `friend` function, filling in the body with whatever present knowledge it has at its disposal.

And so basically, by instantiating a template, one can accumulate global state, which we can access by calling a certain `friend` function. And well... with C++26 reflection, we are able to programmatically substitute into templates, and not only that, we are able to use function parameters to do so.

And so that's exactly what the implicit constructor of `const_param` does:

```cpp
consteval explicit(false) const_param(const auto value) {
    /* Substitute into the template which does the friend injection. */
    const auto set_value = substitute(^^set_const_param_value, {
        std::meta::reflect_constant(value),
    });

    /* Needed so that the compiler won't just ignore our substitution. */
    extract<std::size_t>(substitute(
        ^^ensure_instantiation, {set_value}
    ));
}
```

Here we need to also do this weird `extract<std::size_t>(substitute(^^ensure_instantiation, ...))` thing, and that's because if we don't then the compiler just doesn't bother instantiating the template, and so our `friend` function never gets defined. `ensure_instantiation` here is just a variable template that maps to `sizeof(T)`, and that's good enough for the compiler.

And so now we can specify the types of `First` and `Second` as the types of the first and second function parameters (and remember, they each have their own fully unique type). And so when a value gets passed in those positions, the compiler will execute their implicit constructors, and we will then have a value associated with those `First` and `Second` template parameters via the friend-injection.

And we can get their associated values by just calling the `friend` function we just materialized. `const_param` has a helper method to do that for us:

```cpp
static consteval auto value() -> auto {
    /* Retrieve the value from the injected friend function. */
    return const_param_value(const_param{});
}
```

Note that the method can be `static` because it relies solely on the type of the particular `const_param`, there's no other data necessary.

So great, we've successfully curried a function parameter into something we can use like a template parameter. That's really cool. But... here's where the (kind of) in the title comes in. There are a few caveats.

## Of Course There Had To Be A Catch...

The first and probably most glaring caveat is probably that we can't actually change the *interface* of the function based on these `const_param`s. Like for instance, we can't make the return type depend on them, and indeed we can't specify `auto` as the return type at all. If one tries, then the compiler will error and say that it hasn't deduced the return type of the friend function. I believe that's because it needs to form out the full interface of the function when it's called, which it does before running the implicit constructors which power our `const_param`s. So they can only affect things within the function's body. And that's still cool, but it does leave them as strictly less powerful than a normal template parameter.

The second is that, because each `const_param` has a distinct type at each new call-site, it does not deduplicate template instantiations that are otherwise equivalent. If you call `sum(1, 2, 3)` at one place, that will lead to a different template than what `sum(1, 2, 3)` leads to at a different place. I can't imagine that that's easy on our compilers and linkers.

And the third is well, it's just hacky. It's unintuitive, it's obscure, it relies on stateful metaprogramming via friend-injection which is an unintended glitch in the standard. When investigating this I received plenty of errors and segfaults in the compiler, and sometimes had to switch compilers because some things only worked on GCC while other things only worked on Clang (this current version however works on both GCC trunk and the experimental Clang reflection branch). The compiler isn't very enthused about the technique. I wouldn't be at all surprised if this relies on some happenstance of how the compiler works rather than something that's actually specified behavior. You probably shouldn't use it in production.

And too, it's all pretty involved just so we can avoid writing `std::cw<>` at the call-site, or just passing them directly as template parameters. I'm someone who will put a lot of effort into trying to make the user-facing API really pretty and pleasing and ergonomic, and I do think that this basically achieves that. But even though I place a severe amount of importance on the pleasantness of interfaces, I still really don't think this is something that should be genuinely used. I think it's really cool that it's possible, and maybe there will be some case where it's the perfect fit that solves everything and makes some interface finally viable. But I'm not holding my breath on that.

***

But all that said, it was still very fun to get this cooked up. I spent hours trying to figure out if there was a way with `define_aggregate` to do this, instead of having to use friend-injection, and I think the answer is just no, at least not today. I wonder if with less restricted code generation it could someday be possible to do this better, that'd be neat.

But that something *like* this is already possible in just C++26 I think does speak to the power that reflection is really bringing to us. I'm really excited to see what people do with it, both the sane and insane things.
