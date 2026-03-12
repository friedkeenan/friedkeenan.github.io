---
title: "Exploring Mutable Consteval State in C++26"
category: C++
tags:
    - c++
    - c++26
    - reflection
    - friend-injection

default_lang: cpp
---

Recently I wrote a post titled ["Implementing Constexpr Parameters Using C++26 Reflection (Kind Of)"](/posts/implementing-constexpr-parameters-kind-of). The primary mechanic behind that post was using C++26's `std::meta::substitute` in order to trigger friend injection and insert some global state during constant evaluation, and it left me wondering what other crazy stuff we might be able to do with that technique.

As well, shortly after making that post, I read Barry Revzin's blogpost ["Behold the power of `meta::substitute`"](https://brevzin.github.io/c++/2026/03/02/power-of-substitute/), where he actually shows off some really cool stuff that leads to a result very similar to my previous post, but in its own way. However, what really caught my eye was that, at the end of his blogpost, he brings up the idea of a `consteval mutable` variable.

With a `consteval mutable` variable, you would be able to gradually update the state of that variable at compile time, as the compiler is building your program, so you wouldn't have to figure out ahead of time all the information you might need. You could just figure it out as you go along building your program.

And very quickly I realized that... that's just stateful metaprogramming. And we can do that with friend injection. And we can even use C++26 reflection to inject that sort of state, like in my previous post.

And so that's basically what I've done: [Take a look](https://compiler-explorer.com/z/rGxjYavxa).

In Barry's blogpost as well, he lays out some speculative syntax to use a `consteval mutable` variable to define a loop:

```cpp
template for (consteval mutable int i = 0; i < 4; ++i) {
    /* ... */
}
```

Which is like an expansion statement we have now, but instead of just expanding over a range, it can keep expanding until some arbitrary condition is met.

And by the end of this post, we'll be able to construct much the same thing in library code:

```cpp
expand_loop([]<auto Loop>() {
    /* ... */

    consteval {
        if (Loop.index() >= 3) {
            Loop.push_break();
        }
    }
});
```

It's not quite the same as what Barry showed, it's more like a `template while (true)`, but we can accomplish basically the same thing with it.

## So How Do We Get There?

Well, the first thing we should know is that `define_aggregate` will decidedly *not* get us there. The P2996 paper that introduced reflection has [an example](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p2996r13.html#compile-time-ticket-counter) that uses `define_aggregate` to modify and accumulate state, so it appears at first promising. Unfortunately however, `define_aggregate` is in fact quite restricted on where it can be called from. It can only be called within a `consteval` block, but more importantly, that `consteval` block must be at the same scope as the type passed to `define_aggregate`. This ends up being much too limiting for our purposes.

But that's fine, because we have our trusty friend injection. We don't need `define_aggregate` anyways. The man can't keep us down.

So then I guess the first thing to do now is to try to model a `consteval mutable` variable, and see about modifying its state using friend injection.

***

First, we'll need a type to associate the state to:

```cpp
template<typename Tag>
struct consteval_state {
    /* ... */
};
```

The `Tag` template parameter here is just some type that we can claim as unique to a given state. We won't actually reference it anywhere in our implementation, we just use it to keep different `consteval_state`s distinct from each other. We don't need to worry about that right now though, just know that it won't be a big bother in the end to come up with a tag type.

So now we can put our friend-injection machinery in here. I'll come right out and say though, we can't just associate this `consteval_state` type itself with a friend function. If we only wanted to store up to one value for a given state, then we could. But of course, we want to keep modifying our state an arbitrary number of times.

If we were to try to do that with just this `consteval_state` type though, then the moment we try to set a second value, the compiler would yell at us and tell us that we can't redefine the injected friend. We can only perform friend injection once for a given type, so we need a unique type for each n-th state. We can get that by adding an `index_t` class template that will be associated with the injected friend function:

```cpp
template<typename Tag>
struct consteval_state {
    template<std::size_t>
    struct index_t {
        friend auto consteval_state_value(index_t) -> auto;
    };
};
```

Eventually, this `consteval_state_value` friend function will be defined as returning some value. The important thing is that these `index_t<N>` types are particular to our `consteval_state<Tag>` type. Each `consteval_state` type will have its own particular family of `index_t` types, which don't collide with another `consteval_state` type's.

And from here, it's pretty simple to add the template which will host the friend injection:

```cpp
template<typename Tag>
struct consteval_state {
    template<std::size_t>
    struct index_t {
        friend constexpr auto consteval_state_value(index_t) -> auto;
    };

    template<std::size_t Index, auto Value>
    struct store_consteval_state {
        friend constexpr auto consteval_state_value(index_t<Index>) -> auto {
            return Value;
        }
    };
};
```

When the `store_consteval_state` template gets instantiated, it will *define* the friend function we only *declared* earlier within the `index_t` class, and it will return the `Value` that was passed in its template parameters. We will then be able to retrieve this `Value` from just calling the `consteval_state_value` friend function, using only a given `index_t<N>` type. In effect, a given `index_t<N>` type will then serve as a key to retrieve a certain value that we stored elsewhere.

And so now with this, we can write much the same code as my previous post to insert some state at a given index:

```cpp
template<typename Tag>
struct consteval_state {
    /* ... */

    static consteval auto insert(const std::size_t index, const auto value) -> void {
        /* Substitute into the template which does the friend injection. */
        const auto store_state = substitute(^^store_consteval_state, {
            std::meta::reflect_constant(index),
            std::meta::reflect_constant(value)
        });

        /* Ensure that the template gets instantiated. */
        extract<std::size_t>(substitute(
            ^^impl::ensure_instantiation, {store_state}
        ));
    }
};
```

{:.prompt-info}
> As in the previous post, the `extract<std::size_t>(substitute(...))` expression here is just to convince the compiler to instantiate our substituted template, otherwise it just won't bother.

Retrieving the state has a little more to it, but not much more. First we're going to define a helper variable template for retrieving the state value from a given index type:

```cpp
namespace impl {

    template<typename Index>
    constexpr inline auto get_consteval_state = consteval_state_value(Index{});

}
```

And then we can write our getter:

```cpp
template<typename Tag>
struct consteval_state {
    /* ... */

    static consteval auto get(const std::size_t index) -> std::meta::info {
        const auto index_type = substitute(^^index_t, {
            std::meta::reflect_constant(index)
        });

        const auto state = substitute(^^impl::get_consteval_state, {index_type});

        return constant_of(state);
    }
};
```

Here, we have to return a `std::meta::info` because we don't know what the type of the state at `index` will be, so we've type-erased it.

Additionally, we wrap our susbtitution into our helper template with `constant_of`. This is so that the returned reflection won't refer specifically to our implementation-detail variable, but rather its underlying value. This will also allow it to compare more intuitively with other reflections.

##  Something We Can Work With

That all gets us something we can play around with, so now let's see about how to define a particular `consteval_state`. Basically, there are two ways. The first is defining it as a type:

```cpp
struct my_state : consteval_state<my_state> {};
```

Where we use the just-defined `my_state` type as the tag type for our state, akin to CRTP. And then we would call `my_state::insert` and `my_state::get`.

The other way is defining it as a value:

```cpp
constexpr auto my_state = consteval_state<struct my_state>{};
```

Where we use the declared-inline `struct my_state` of the same name as our variable as the tag type for our state. Then we could call `my_state.insert` and `my_state.get`.

I don't know that one is necessarily any better than the other, but it's helpful to know of both. I'll be sticking to the first way of defining it for this post, as a type, just because I think it's a little bit more clear what's happening with its definition.

***

Let's check out [our current work so far](https://compiler-explorer.com/z/MqcKW5xKK).

Here we can see how we would insert and retrieve state:

```cpp
consteval {
    my_state::insert(0, ^^int);
}

static_assert([: my_state::get(0) :] == ^^int);

consteval {
    my_state::insert(1, 'A');
}

static_assert([: my_state::get(1) :] == 'A');
```

We have to splice the result of `my_state::get` in order to get at the actual values, but that's still pretty cool. And we can even have different types for each n-th state. That's a little freaky, but it is at the same time nice that we don't have to be restricted in that way.

If we did restrict the state values to only a single type, then we would actually be able to `extract` out their values in our `get` method and avoid needing to return a `std::meta::info`. That'd be useful, and certainly something to explore if one were to make this into a library. For now, however, we'll stick with this current version.

Now, I should also say that this example code we have here *could* feasibly be accomplished using `define_aggregate`. What distinguishes our code, however, is that these `consteval` blocks which store the state don't need to all be at the same scope. We could for instance add the following code to our usage:

```cpp
auto some_function() -> void {
    consteval {
        my_state::insert(2, 42);
    }
}

static_assert([: my_state::get(2) :] == 42);
```

As far as I'm aware, there is no way to accomplish this using `define_aggregate`.

## Still Not Quite There Yet

We do have one issue, though, which is that we have to know the index beforehand to insert our state at and to later get our state from. If we wanted to use a `consteval_state` like a variable, then we'd probably want to somehow just push the latest state, and be able to retrieve the latest state too. For that I guess we'd have to be able to detect whether we've injected the friend function for a given index, so... can we do that?

Well, as far as I know, the normal way one would detect an injected friend is with either a `requires` expression or with SFINAE. However, that won't work for us. Our compilers are smart, and they've learned to cache these sorts of things. That means that the moment we query it, we'd be locking in its value. So that's no good. But, C++26 reflection has a few examples where some consteval functions will return different results depending on what the compiler has seen up to that point. Maybe we can take a look at those.

And trust me, I did. I tried a whole bunch of stuff. As one example of what I looked at, there's the `is_complete_type` function, which will give different results at different points of building your program, depending on whether a type has been defined. For example:

```cpp
struct some_type;

static_assert(not is_complete_type(^^some_type));

struct some_type {};

static_assert(is_complete_type(^^some_type));
```

This is actually how `define_aggregate` can be used to gradually accumulate state. You can check whether a class has been completed yet, and if not, then you can call `define_aggregate` on it, making it then complete.

But I'm not sure how we could later define a type in concert with friend injection without running into the inherent limitations of `define_aggregate`. So I don't think `is_complete_type` will work for us.

I'm going to cut out a lot of trial and error here that I went through. In short, I only ended up finding one thing that will work for us. It's definitely possible I've missed something, and I greatly encourage you to try to figure out another way, because this way is... weird.

***

Function parameter names are a little strange. When you declare a function, you can leave a parameter without a name, or you could give it any sort of name you want. You can give it one name when you first declare it, and then a completely different one if you decide to declare it again. The world's your oyster.

But with reflection, this poses a slight issue. We want to be able to reflect over function parameters and see what names they have. But if you have a situation like this:

```cpp
auto function(int original_name) -> void;

/* Redeclare and change the parameter name. */
auto function(int altered_name) -> void;
```

Then when you ask for the identifier of that first parameter, which name should you get? Well, neither name really stands out as the unambiguously correct option. So what the standard says about this situation is that you shouldn't be able to get the identifier of a function parameter once it has been redeclared with a different name.

That means that `identifier_of` will fail, and `has_identifier` will return false, and this behavior will change depending on what the compiler has seen so far:

```cpp
auto function(int original_name) -> void;

/* All good, only one name so far. */
static_assert(has_identifier(parameters_of(^^function)[0]));

/* Redeclare and change the parameter name. */
auto function(int altered_name) -> void;

/* We can no longer get the identifier of the parameter. */
static_assert(not has_identifier(parameters_of(^^function)[0]));
```

And this works with declarations of friend functions as well, so it'll work for us.

## Success In Sight

Let's put this all to use, then. First we'll need a function to redeclare, and it'll need to be associated with a particular `index_t`, so that it can be associated with our n-th state we inserted. We can achieve that with a function template:

```cpp
namespace impl {

    template<typename Index>
    auto track_consteval_state(int original_name) -> void;

}
```

Next, we'll need to redeclare this function when we inject the `consteval_state_value` friend function, so that the result of `has_identifier` will change at the same time we insert our state. We can just do that right next to where we already inject our friend function:

```cpp
template<typename Tag>
struct consteval_state {
    /* ... */

    template<std::size_t Index, auto Value>
    struct store_consteval_state {
        friend constexpr auto consteval_state_value(index_t<Index>) -> auto {
            return Value;
        }

        /* Redeclare with an altered parameter name. */
        friend auto impl::track_consteval_state<index_t<Index>>(int altered_name) -> void;
    };
};
```

Then we can write a method to query whether a state has been stored at a given index, leveraging our `has_identifier` trick:

```cpp
template<typename Tag>
struct consteval_state {
    /* ... */

    static consteval auto has_inserted_at(const std::size_t index) -> bool {
        const auto index_type = substitute(^^index_t, {
            std::meta::reflect_constant(index)
        });

        const auto tracker = substitute(^^impl::track_consteval_state, {index_type});

        /* If the parameter has no identifier, then state has been inserted. */
        return not has_identifier(parameters_of(tracker)[0]);
    }
};
```

And from there it's pretty straightforward to just write a `push(...)` method that calls our `insert` method with the next available index, and a `last()` method that calls our `get` method with the last-inserted index.

***

Now we can put this all together and check out [the progress we've made](https://compiler-explorer.com/z/865TrTrvs).

We get to replace our previous calls to `insert` and `get` with calls to `push` and `last`, which get us functionality analogous to setting and accessing a variable. This already is really awesome:

```cpp
struct my_state : consteval_state<my_state> {};

consteval {
    my_state::push(^^int);
}

static_assert([: my_state::last() :] == ^^int);

consteval {
    my_state::push('A');
}

static_assert([: my_state::last() :] == 'A');
```

There is one tiny hitch, though. It doesn't work on GCC right now.

Currently at least, GCC has [a bug](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=124382) with specifically just this construct of querying `has_identifier` with this specific sort of rigamarole. But it works on the experimental Clang reflection branch, so we'll be sticking to that in this post.

But otherwise, this is great. We've successfully designed a `consteval mutable` variable, like we originally set out to do.

## So What Can We Do With This?

Well, theoretically you could use this for all sorts of things that you can use variables for. But what I really want to do with it is to pull off something like that `template for` that Barry showed in his blogpost, which to remind, looks like this:

```cpp
template for (consteval mutable int i = 0; i < 4; ++i) {
    /* ... */
}
```

At the start of this post, I showed that eventually we'd be able to make something that accomplishes the same thing, that looks like this:
```cpp
expand_loop([]<auto Loop>() {
    /* ... */

    consteval {
        if (Loop.index() >= 3) {
            Loop.push_break();
        }
    }
});
```

So let's work towards that.

***

If you were someone who kept up with the C++26 reflection proposals as they were progressing, you'll probably be aware of a workaround they often used to account for not relying on the proposal which added expansion statements.

They used library code to fill the gap, and made a little helper facility they called `expand`, which is [shown off in P2996](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p2996r13.html#implementation-status) for example.

Basically how that `expand` works is it takes in a range, gathers its elements as template arguments, and hands them off to a helper type that will substitute those arguments into a lambda's call operator template. It would end up being used like so:

```cpp
[: expand(members_of(^^some_type, ctx)) :] >> []<auto Member>() {
    /* Do something with each constexpr 'Member'... */
};
```

Whereas now with expansion statements we could write that instead like this:

```cpp
template for (constexpr auto Member : std::define_static_array(members_of(^^some_type, ctx))) {
    /* Do something with each constexpr 'Member'... */
}
```

Our `expand_loop` will operate similarly, but it will have to be ordered somewhat differently.

Their `expand` helper uses an implementation-detail called `replicator` which effectively stores the template arguments from the passed-in range, which will then later be passed to the lambda's call operator template in the `replicator`'s `operator >>`. So let's start out with our own equivalent of that:

```cpp
namespace impl {

    template<typename Body, std::meta::info... CallOperators>
    struct replicator_t {
        static constexpr auto execute(const Body body) -> void {
            template for (constexpr auto CallOperator : {CallOperators...}) {
                body.[: CallOperator :]();
            }
        }
    };

    template<typename Body, std::meta::info... CallOperators>
    constexpr inline auto replicator = replicator_t<Body, CallOperators...>{};

}
```

`Body` here will be the type of the lambda object that we pass to `expand_loop`, and `CallOperators` will be all the substituted call operator templates of the lambda that our `expand_loop` will gather.

And then in the `execute` method we just call all those call operators on the lambda `body`.

So basically this `replicator` is just taking in a list of functions to call, and then calling them in its `execute` method. Simple on its own.

Now let's take a peek at implementing `expand_loop`. We're going to separate out the meat of this into a separate function, so the function we actually call gets to look like this:

```cpp
template<typename Body>
constexpr auto expand_loop(const Body body) -> void {
    return [: impl::expand_loop<Body>() :].execute(body);
}
```

Here we end up calling an `impl::expand_loop` function template, which will return a `std::meta::info` of the `impl::replicator` object. We can then splice that object in, and then call the `execute` method on it.

So now let's see about this `impl::expand_loop` function. Let's sketch it out first:

```cpp
namespace impl {

    template<typename Body>
    consteval auto expand_loop() -> std::meta::info {
        /* We'll accumulate the args to 'impl::replicator' here. */
        std::vector<std::meta::info> args = ...;

        return substitute(^^impl::replicator, args);
    }

}
```

It accepts the `Body` template parameter, and returns a `std::meta::info` representing the `impl::replicator` object, which we will then splice in and call `execute` on, in order to enter our expanded loop.

But now we'll need to figure out how to fill in those `args`. The first one is just `Body`, so we can start off with that:

```cpp
auto args = std::vector{^^Body};
```

As for the rest of the arguments, basically what we'll do is we'll make our own `consteval_state` for the state of the loop, which we can substitute into the lambda's call operator template. Then, in the lambda it can interact with that loop state to signal that we should end the loop, and thus stop accumulating arguments.

Let's sketch out that state:

```cpp
namespace impl {

    template<typename Body>
    consteval auto expand_loop() -> std::meta::info {
        /* ... */

        struct loop_state : consteval_state<loop_state> {
            std::size_t _index;

            constexpr auto index(this loop_state self) -> std::size_t {
                return self._index;
            }

            static consteval auto push_break() -> void {
                consteval_state<loop_state>::insert(0, 0);
            }

            static consteval auto should_break() -> bool {
                return consteval_state<loop_state>::has_inserted_at(0);
            }
        };
    }

}
```

Here we go beyond the normal `consteval_state` definition and add some methods and even a member. We need the `_index` member because we'll be passing an object of this `loop_state` type as a template parameter to the lambda, and every loop iteration needs to be distinct. As a side effect, it becomes nice that we can just have that loop index handy.

Also note that because we're templating `expand_loop` on the `Body` type, each `loop_state` will be unique for each `Body` we get passed, which will be a unique type each time for a lambda.

In the `push_break` method, we just insert some state at index `0`, and when we check whether we `should_break`, we just check whether any state has been inserted at index `0`.

And now that we have that, it's not too much more bother to fill in the rest of the `args`:

```cpp
auto index = 0uz;
while (true) {
    const auto state = loop_state{._index = index};

    args.push_back(std::meta::reflect_constant(
        substitute(^^Body::template operator (), {
            std::meta::reflect_constant(state)
        })
    ));

    if (loop_state::should_break()) {
        break;
    }

    ++index;
}
```

With Clang at least, simply substituting into the call operator template will step through its consteval stuff, updating the loop state accordingly.

From my experimenting with GCC, I don't think this is necessarily guaranteed to be the case, but I'm pretty positive that we can do a similar thing as with the earlier `impl::ensure_instantiation`. We should be able to extract out the resulting member function pointer of each call operator, and then that would guarantee the function template gets instantiated at that point in our constant evaluation.

But I'll leave it as-is for now, since it works and looks nicer this way.

***

Anyways though, we're then able to use our `expand_loop` like so:

```cpp
expand_loop([]<auto Loop>() {
    std::printf("LOOP: %zu\n", Loop.index());

    consteval {
        if (Loop.index() >= 3) {
            Loop.push_break();
        }
    }
});
```

We can even manage and update our own separate states in each loop iteration too, like is shown in the initial [Compiler Explorer link I gave](https://compiler-explorer.com/z/rGxjYavxa).

As well, one can forgo using our `push` and `last` functions on our `consteval_state`s, and instead only `insert` at and `get` from a given index, and if we do then we can [get this to work on GCC](https://compiler-explorer.com/z/MoEor1a5G), with some other slight changes. We can't use a `consteval_state` quite like a variable on GCC currently, but it proves that the `expand_loop` can work even there.

## And That Just... Works?

Yeah, it does appear to just work.

When I first wrote this code, I fully expected the compiler to just yell my ear off or maybe get stuck in some infinite loop somehow. But no, it just... works.

And that's awesome. That's actually genuinely really cool. Sure, it's really not the easiest on the eyes, but it seems super powerful, doesn't it? My previous post had me telling you not to use what it showed off, but this one? Genuinely, maybe there are circumstances where you should use this.

I probably wouldn't tell you to use this all over the place, but there could be some real places where this sort of stuff would genuinely simplify and aid your code. The circumstances that Barry showed off at the end of his blogpost seem compelling to me, at least.

And this `expand_loop` is just one low-level example of this sort of behavior. We could also write different versions, like an `expand_until_equal<some_state, sentinel_value>`. I feel like there's potentially a whole family of functions to explore here.

***

But even besides whatever value this all might hold for your code today, what I think is maybe more interesting is that this library code implementing something like a `consteval mutable` variable can possibly give a foundation for future proposals to the language to work off of.

Not just in a "Hey, we can already do this, could we get a nicer way to do it, please?" way, but also in terms of allowing people to more broadly explore the mechanics of this mutable consteval state and figure out where and when it's useful, which could be used to build a base of experience for a language proposal to work off of.

So in that sense, I very much would encourage you to use this and explore its capabalities. In doing so, you could end up pushing C++ forward in a meaningful way.

Because, obviously, it would be better to have a `consteval mutable` variable rather than all this crazy stuff, even as fond of the craziness as I am.
