---
title: Evolving a Translation System with Reflection in C++
category: C++
tags:
    - c++
    - c++26
    - c++29
    - reflection

default_lang: cpp
---

Lately, I've been using C++26 reflection to create [some crazy and cursed stuff](/posts/2026/03/08/exploring-mutable-consteval-state). But even though I quite enjoy that work, it is as well quite far from the norm of what reflection is going to offer us in our everyday code.

Reflection is definitely not just that craziness, and so I want to present reflection in a more grounded environment, and in a way that will probably land as more typical usage as our codebases gradually come into contact with it.

So then, in this blogpost, I will be exploring a spectrum of options for how reflection can upgrade a translation system that I already use in one of my projects. We'll look at where it's at now without reflection, identify places in which reflection could plausibly help, and then explore a series of modifications we could make to soothe those problem points.

The purpose of looking at each of these options will not be to declare that one is clearly the best option or the one that makes the most sense, but rather to get a better feel for what *could* make sense to do, and whether some things are really worth the effort. We're trying to gauge the benefits that reflection can bring to our code.

And who knows, even if one option is less appealing for this particular situation, maybe in a different situation it could be the perfect fit.

## The Present Day

Let's take a look at my translation system as it is today, without reflection.

A lot of other translation systems read from files at runtime, which is a good choice for those systems, but my project wouldn't really benefit from that, so mine gets to stay all in code. The skeleton of it looks basically like this:

```cpp
namespace lang {

    /* A type-safe wrapper for a string that's been translated. */
    struct translated_string {
        std::string _underlying;
    };

    enum class language : std::uint8_t {
        en_US,
        en_GB,
        en_AU,

        de_DE,
        de_CH,
    };

    template<typename... Args>
    struct translation_set {
        using string = std::format_string<const Args &...>;

        string en_US;
        string en_GB;
        string en_AU;

        string de_DE;
        string de_CH;

        constexpr auto string_for_language(lang::language language) const -> string {
            switch (language) {
                using enum lang::language;

                default:
                case en_US: return this->en_US;

                case en_GB: return this->en_GB;
                case en_AU: return this->en_AU;

                case de_DE: return this->de_DE;
                case de_CH: return this->de_CH;
            }
        }

        constexpr auto translate(lang::language language, const Args &... args) const -> lang::translated_string {
            const auto fmt_string = this->string_for_language(language);

            return lang::translated_string{
                std::format(fmt_string, args...)
            };
        }
    };

}
```

It also has some slight optimizations that it does when `Args` is an empty pack, and `translated_string` has a bit more to it in reality, but this is the gist of it.

You could then define a `translation_set` like so:

```cpp
namespace translations {

    constexpr inline auto example = lang::translation_set<>{
        .en_US = "Imagine this is American English.",
        .en_GB = "Imagine this is British English.",
        .en_AU = "Imagine this is Australian English.",

        .de_DE = "Imagine this is German.",
        .de_CH = "Imagine this is Swiss German."
    };

}
```

And then you could call `translations::example.translate(lang::language::de_CH)` and get back a `translated_string` of `"Imagine this is Swiss German."`, for example. Here it is [in Compiler Explorer](https://compiler-explorer.com/z/7z519q61q).

And this is nice enough. It's simple, it works, it's basically pleasant to use. But... it's also pretty clear to me that there's something pretty unideal about this, which is that we're repeating the language names four different times.

We spell out the language names once when defining the `lang::language` enum, again when defining the fields of a `lang::translation_set`, and then two more times in the `string_for_language` method, when we have to do the `case en_US: return this->en_US;` lines.

This poses a very real maintenance burden, even if it's smaller-scoped. Whenever we add a language, we have to edit 4 different places in the code (in addition to all the translation sets), and make sure we didn't make any typos when doing so.

It's as well a fairly rote and unexciting process, and one that would plausibly be better passed off to some automation to save us some peace of mind.

***

But even besides that, there is a slight quality-of-life feature that this also presently lacks, which is defaulting one language's string to another's, for instance defaulting `en_GB` to `en_US`, or `de_CH` to `de_DE`.

We could of course give the string members defaults like this:

```cpp
string en_US;
string en_GB = en_US;
string en_AU = en_GB;

string de_DE;
string de_CH = de_DE;
```

And that would work, but then when we define a translation set, it would be like this:

```cpp
constexpr inline auto with_defaults = lang::translation_set<>{
    .en_US = "Default for English",

    .de_DE = "Default for German"
};
```

And that wouldn't be unreasonable, but when looking at it, it's not necessarily clear that any given string was actually intended to be defaulted, or could rather have just been forgotten about instead.

One could add some comments in there like `/* .en_GB = defaulted; */` or something to that effect, and that would make it better. However, we could end up forgetting to add the appropriate comment, and that would result in zero diagnostics from the compiler to inform us of our mistake.

And too, if one were to add a new language, and have it default to another language, then there would again be no diagnostic when constructing any of these `translation_set`s, and no indication given that they should pay any thought to the newly-added language, because it's just been implicitly defaulted everywhere.

So it would be nice too if we could have defaulting like this, but somehow force it to be made explicit.

***

Besides that though, everything else seems fine to me. The `translated_string` wrapper type is good, and the `translate` method looks alright.

And so from now on then, we'll just forget about those and just stick to these problem points of repeating the language names and lacking explicit defaulting.

## Seeking Validation

Now, it might be tempting to immediately try to go all out with reflection and completely rewrite all the guts of our code to get it exactly how we dream of it being. And don't worry, we'll get there.

But, it's not always the practical option to do a big overhaul of a region of our codebase. There are certain realities of development that can make that the worse choice.

One of those realities is that other developers ought to be able to understand and maintain the code we write. With reflection being such a new feature, we could run up against that reality if we were to go all out with it.

Maybe, then, we could work something out with reflection that is instead more minimal, and maybe even doesn't impact our preexisting code at all.

A lot of our concerns with the present code have to do with our repeating the language names in several places, and worrying about what could happen if we add a new language and forget to update those places, or even update them incorrectly.

Maybe then we could just simply *validate* that all these repetitions are correct and agree with each other. That would require no changes to our existing code, and could just end up as a test that we can `static_assert` on somewhere.

Let's try it.

### Validating Members

First let's validate that the fields of a `translation_set` match the `lang::language` enumerators.

We can do that by just making sure all their identifiers match:

```cpp
consteval auto validate_translation_set_fields() -> bool {
    const auto enumerators = enumerators_of(^^lang::language);

    const auto fields = nonstatic_data_members_of(
        ^^lang::translation_set<>,
        std::meta::access_context::unchecked()
    );

    return std::ranges::equal(fields, enumerators, [](auto field, auto enumerator) {
        return identifier_of(field) == identifier_of(enumerator);
    });
}
```

{:.prompt-info}
> It should be noted that we need to operate on a particular instantiation of `lang::translation_set`, since templates aren't actual types and don't have members. It would of course be fairly unlikely that different instantiations would behave differently here, but conceivably a poorly-implemented specialization could have different members defined.

We could do some extra validation, like checking that each field has the right type and even the right access rules, and all sorts of other things about the structure of the class. I don't feel much need to do that here, but it would be relatively simple to do if we wanted to.

But anyways, after we define that function, we can just add a simple assert directly afterwards:

{:.no-line-numbers}
```cpp
static_assert(validate_translation_set_fields());
```

And then with just that, we'd have already covered half of the places we repeat the language names, and ensured that they won't be out of sync with each other.

### Validating `string_for_language`

However, that still leaves our `string_for_language` method. To remind, that looks like this:

```cpp
constexpr auto string_for_language(lang::language language) const -> string {
    switch (language) {
        using enum lang::language;

        default:
        case en_US: return this->en_US;

        case en_GB: return this->en_GB;
        case en_AU: return this->en_AU;

        case de_DE: return this->de_DE;
        case de_CH: return this->de_CH;
    }
}
```

So... how would we go about validating this?

Well, we can't actually inspect the body of a function, so we'll have to make do by calling the method and validating its results. So basically, we'll be constructing a unit test using reflection.

For that, we'll need a translation set that we can call the `string_for_language` method on and get results we can expect. We'll aim for one that looks like this:
```cpp
lang::translation_set<>{
    .en_US = "en_US",
    .en_GB = "en_GB",
    /* And so on... */
}
```

With reflection, this is relatively simple to do:

```cpp
template<std::meta::info... Languages>
constexpr inline auto test_translation_set_helper = lang::translation_set<>{
    identifier_of(Languages)...
};

constexpr inline auto test_translation_set = [:
    []() {
        auto enumerators = enumerators_of(^^lang::language);

        for (auto &enumerator : enumerators) {
            enumerator = std::meta::reflect_constant(enumerator);
        }

        return substitute(^^test_translation_set_helper, enumerators);
    }()
:];
```

We define a `test_translation_set_helper` variable template that takes in a pack of enumerators of `lang::language`, and then initializes each field to the identifier of each enumerator. And we know that this will be in the correct order and all because we just validated that with the `validate_translation_set_fields` test.

And then we just substitute into that helper template and splice the result in to get our desired translation set.

{:.prompt-info}
> We wrap each enumerator with a call to `std::meta::reflect_constant`, and that's because our template receives `std::meta::info` objects. If we were to instead pass the result of `enumerators_of` directly to `substitute`, then the enumerators would be unwrapped into their actual `lang::language` values, which we don't want.

With that, we can then write the function that validates `string_for_language`:

```cpp
consteval auto validate_string_for_language() -> bool {
    for (const auto enumerator : enumerators_of(^^lang::language)) {
        const auto language = extract<lang::language>(enumerator);
        const auto string = test_translation_set.string_for_language(language);

        if (string.get() != identifier_of(enumerator)) {
            return false;
        }
    }

    const auto unenumerated = lang::language{0xFF};
    const auto string = test_translation_set.string_for_language(unenumerated);

    if (string.get() != "en_US") {
        return false;
    }

    return true;
}
```

{:.prompt-info}
> We have to call the `get` method on the returned strings because they're actually of type `std::format_string<>`, which doesn't have any comparison operators of its own.

Here, we don't just check all of the enumerators of `lang::language`, but we as well check the `default` case of our `switch` statement and ensure that it falls back to returning the `en_US` string.

Though, we *do* just assume that `0xFF` will be an unenumerated value for `lang::language`. With reflection we wouldn't have to assume that, and could go through the enumerators and make a best effort to find an unenumerated value.

But today at least, I'm not gonna bother with that, and will leave it as an exercise for the reader.

### Results

Putting this all together, we can now check it out [in Compiler Explorer](https://compiler-explorer.com/z/xcoozdYjM) and see that it does what we want.

If we add a new language without updating the rest of our code, or force a typo in our implementation, then the compiler will yell at us just like we wanted.

And that's pretty nice, I think. We solved a real and annoying problem that we had with our code, and we did it without even changing any of the code we had before. It's all added on top.

But, of course, it doesn't solve all our problems.

Even if we can be confident that we've updated the code correctly when a new language is added, we do still need to actually update all the code ourselves. And that's still annoying.

But it is still an improvement, and too it is code that we could plausibly keep around even as our implementation could change. They do just function as unit tests, after all.

## Expanding Our Horizons

If we *were* willing to change our implementation though, then where could we start?

Well, our `string_for_language` method contains half of the repetitions of the language names, and it's very self-contained, so it would seem a good place to try to reduce those repetitions.

We can't, however, just synthesize a `switch` statement in C++26. What we can do instead, though, is create a series of `if` statements that accomplish the same thing.

We'll do that with an expansion statement, i.e. a `template for`:

```cpp
constexpr auto string_for_language(lang::language language) const -> string {
    static constexpr auto Enumerators = std::define_static_array(
        enumerators_of(^^lang::language)
    );

    template for (constexpr auto Enumerator : Enumerators) {
        if (language == [: Enumerator :]) {
            static constexpr auto Field = impl::field_for_enumerator(
                ^^translation_set, Enumerator
            );

            return this->[: Field :];
        }
    }

    return this->en_US;
}
```

For each enumerator of `lang::language`, we just check if the passed-in parameter `language` equals the enumerator, and if so then we return the corresponding field. And if we don't find a match, then we fall back to returning the `en_US` string.

The logic for `impl::field_for_enumerator` just looks for the field of same name as the enumerator:

```cpp
consteval auto field_for_enumerator(std::meta::info type, std::meta::info enumerator) -> std::meta::info {
    const auto fields = nonstatic_data_members_of(
        type, std::meta::access_context::unchecked()
    );

    for (const auto field : fields) {
        if (identifier_of(field) == identifier_of(enumerator)) {
            return field;
        }
    }

    throw "Couldn't find the field for the given enumerator";
}
```

{:.prompt-info}
> If we find no corresponding field, then we just `throw`. Here I throw a string literal because it's blogware, but in real code you could throw a `std::meta::exception` or your own exception, or use some other erroring scheme.

We could implement this in some different ways, like zipping the fields together with the enumerators and not having to go searching for them, but this works just fine for us.

And, to spoil a little, this `impl::field_for_enumerator` function will be useful for us later anyways.

### Results

We can again look at this [in Compiler Explorer](https://compiler-explorer.com/z/bW5j93vj6).

And we can use our previously-crafted `validate_string_for_language` test to see that it does in fact result in the same behavior.

So, how do we like this?

Well, I quite like it. To me at least, it reads fairly clearly and appears pretty simple. And of course, we reduced our repeating of the language names by half, which is great.

There is a *slight* difference with this though, which is in the machine-code the compiler generates for us.

With GCC optimizing at `-O3`{:.language-console} at least, our implementation with the expansion statement leads to *very* similar machine-code as with the `switch` statement, but differs slightly in how it's ordered. See here [in Compiler Explorer](https://compiler-explorer.com/z/1rKa99oPE).

From some fiddling, this appears to be from how the compiler tries to assume the likelihood of each case, which can affect where it decides to place certain code.

{:.prompt-info}
> Amir Kirsh and Tomer Vromen have [a good talk](https://www.youtube.com/watch?v=RjPK3HKcouA) which explores quite well how the compiler handles perceived likeliness when generating code.

I wouldn't make any assumptions about whether this slight difference matters all that much, at least here. But it's something you may want to explore with benchmarking if this is something that you think could matter for you.

Regardless, to my eye, this is a massive improvement for our code, and from a very simple change. Great.

## Leveling The Field(s)

Of course, this does still leave the issue of having to keep the fields of a `translation_set` in sync with the `lang::language` enum.

It would be nice if we could eliminate one of these two places, and thus have one single source of truth for what languages we support.

Between the enumerators and the fields, it'll be better for us to eliminate spelling out the fields. That's because, in C++26, we can't just programmatically create an `enum`. It'll be much easier for us to work something out with `define_aggregate`, as things currently stand.

There will, however, be a slight issue with that. Instead of being able to initialize a `translation_set` like so:

```cpp
lang::translation_set<>{
    .en_US = "Imagine this is American English.",
    /* And so on... */
}
```

We will have to change this to the following:

```cpp
lang::translation_set<>({
    .en_US = "Imagine this is American English.",
    /* And so on... */
})
```

With parentheses surrounding the braces.

This is because we can't add methods onto a type that we define with `define_aggregate`, or dynamically add fields to a class which we're in the process of defining.

There is a trick where can *inherit* from a `define_aggregate`-ed type and then add methods in the inheriting class, but that would still require us to change how we initialize a `translation_set`, so that's of no use to us.

For my project, this is perfectly fine since I control every usage point of all this code. But in other situations, breaking the API like this might not be as acceptable.

***

Once we've come to terms with that though, getting what we want is pretty simple:

```cpp
template<typename... Args>
struct translation_set {
    using string = std::format_string<const Args &...>;

    struct strings_t;

    consteval {
        define_aggregate(
            ^^strings_t, impl::fields_for_enum(^^string, ^^lang::language)
        );
    }

    strings_t strings;

    /* ... */
};
```

The compiler will generate an aggregate constructor for us, which is nice here. So we can still keep triviality, if that's something we wanted.

The `impl::fields_for_enum` function just returns a bunch of `data_member_spec`s that have the first argument as their type, and that have the same names as the enumerators of the second argument:

```cpp
consteval auto fields_for_enum(std::meta::info field_type, std::meta::info enum_type) -> std::vector<std::meta::info> {
    auto fields = enumerators_of(enum_type);

    for (auto &field : fields) {
        field = data_member_spec(field_type, {
            .name = identifier_of(field),

            /* Just to silence '-Wmissing-field-initializers'. */
            .alignment = {},
            .bit_width = {}
        });
    }

    return fields;
}
```

Kind of funnily, we get to reuse the vector returned from `enumerators_of`, because `data_member_spec` also just returns a `std::meta::info`.

It is also nice, though, that we can very easily and clearly express this one-to-one relationship between each enumerator and each field, by just overwriting each element in the vector.

### Results

We can yet again now put this together [in Compiler Explorer](https://compiler-explorer.com/z/n3hE57Pnz) and see it work.

And, you know what, call me crazy, but I actually find this to be extremely simple.

To me, it appears even more simple than what we had to put together in order to validate our code previously, particularly compared to the `validate_string_for_language` test. Those tests are probably still good to have around maybe, but they seem like they add much less value now too, don't they?

I think I even find this more simple than our original code without any reflection. Now we don't have to look at four different places for each and every language to make sure everything's alright. We've reduced a lot of mental overhead.

Sure, we had to change the interface, but that seems a small price to pay here. And it could plausibly be a benefit too, in that we now have more control over the internal representation of a `translation_set`. We could store the strings however want to, if we added our own constructor.

I am very much pleased with this.

## Explicating Our Defaults

So that's all awesome, we've eliminated all the repetitions of the language names and kept *basically* the same API we had before.

But if you'll recall, we also wanted to find a way to be able to default one language's string to another, but with that forced to be explicit when constructing a `translation_set`.

So basically, we want to be able to construct a `translation_set` in a way that looks somewhat like this:

```cpp
constexpr inline auto example = lang::translation_set<>({
    .en_US = "Imagine this is English.",
    .en_GB = defaulted,
    .en_AU = defaulted,

    .de_DE = "Imagine this is German.",
    .de_CH = defaulted
});
```

So where would we start with that?

### Describing Defaults

Well, first we'll want a way to describe what the default of a given language should be. And since we have our single source of truth about the languages in the `lang::language` enum, then describing our defaults there too would seem appropriate.

We can do that with annotations:

```cpp
enum class language : std::uint8_t {
    en_US,
    en_GB [[=defaults_to(en_US)]],
    en_AU [[=defaults_to(en_GB)]],

    de_DE,
    de_CH [[=defaults_to(de_DE)]],
};
```

For each language that we want a default for, we add an annotation of `defaults_to(/* some language */)` onto the given enumerator. For languages that we don't want to be defaultable, we just leave them plain.

{:.prompt-info}
> For reasons unknown to me, the position for attributes and annotations on enumerators comes after the identifier, instead of before it. It seems an odd placement to me, but in this specific case at least it actually might help aesthetically.

Annotations are just values, so this `defaults_to` expression needs to yield an actual object. Here's what that looks like:

```cpp
struct defaults_to {
    lang::language _default_value;

    consteval explicit defaults_to(std::uint8_t underlying)
    :
        _default_value{underlying}
    {}

    consteval auto enumerator() const -> std::meta::info {
        return impl::enumerator_for(
            std::meta::reflect_constant(this->_default_value)
        );
    }
};
```

Note that in the constructor, we accept the underlying `std::uint8_t` value of a language instead of the `lang::language` type itself. This is because, in the context of the `lang::language` definition where we're constructing these `defaults_to` objects, when we write e.g. `en_US`, it actually refers to the underlying value of that enumerator.

As well, because we're defining this before we actually define the enumerators of `lang::language`, we have to add an "opaque" enum definition before defining the `defaults_to` struct:

{:.no-line-numbers}
```cpp
enum class language : std::uint8_t;
```

And then we have an `enumerator` method to get the reflection of the actual enumerator of the stored `lang::language` value. It calls into `impl::enumerator_for`, which looks like this:

```cpp
consteval auto enumerator_for(std::meta::info enum_value) -> std::meta::info {
    enum_value = constant_of(enum_value);

    for (const auto enumerator : enumerators_of(type_of(enum_value))) {
        if (enum_value == constant_of(enumerator)) {
            return enumerator;
        }
    }

    throw "Could not find enumerator for enum value";
}
```

And then after all that, we can then write an `impl::default_for_language` function like this:

```cpp
consteval auto default_for_language(std::meta::info enumerator) -> std::optional<std::meta::info> {
    for (const auto annotation : annotations_of_with_type(enumerator, ^^lang::defaults_to)) {
        const auto defaults_to = extract<lang::defaults_to>(annotation);

        return defaults_to.enumerator();
    }

    return std::nullopt;
}
```

So, for instance, calling `impl::default_for_language(^^lang::language::en_GB)` will yield a `std::optional` holding `^^lang::language::en_US`. And calling `impl::default_for_language(^^lang::language::en_US)` will yield an empty optional.

I will say though, there is a design decision I made here to return a `std::optional` instead of a `std::meta::info`. `std::meta::info` does already have an "empty" value, which is the null relection, spelled `std::meta::info()`, and it's analogous to a null pointer.

So we could, if we wanted, return a `std::meta::info` directly and just return the null reflection in the case where we find no default.

But, conceptually, that feels a little awkward to me. And we will be needing to actually check this later, so being able to use the familiar `std::optional` logic to do that seems sensible to me.

But it wouldn't be unreasonable to use the null reflection for this, either.

### Acting On Our Defaults

So now we've described our defaults, but how can we actually use them?

Well, as was previously stated, now our `translation_set`s don't actually need to be structured in the same way as their inputs. So instead of accepting a `strings_t` that we then just directly store in a `translation_set`, we can accept some `input_t` object that we can then process into a `strings_t` object that we store.

We can sketch that out like this:

```cpp
template<typename... Args>
struct translation_set {
    using string = std::format_string<const Args &...>;

    using string_input = /* ... */;

    struct strings_t;
    struct input_t;

    consteval {
        define_aggregate(
            ^^strings_t, impl::fields_for_enum(^^string, ^^lang::language)
        );

        define_aggregate(
            ^^input_t, impl::fields_for_enum(^^string_input, ^^lang::language)
        );
    }

    strings_t strings;

    consteval explicit translation_set(input_t input) {
        /* ... */
    }
};
```

We'll take in a type that's shaped like our `strings_t`, but with a different field type of `string_input`, and then we'll use that to fill in our `strings` field.

And as for what that `string_input` type should be, well we want to either pass a string, or say it should be defaulted. We can express that with `std::variant`:

{:.no-line-numbers}
```cpp
using string_input = std::variant<string, lang::defaulted_t>;
```

This `lang::defaulted_t` type is in the same vein as e.g. `std::from_range_t` or `std::in_place_t`, where it's just an empty type for some object that gets passed in a constructor:

```cpp
struct defaulted_t {};

constexpr inline auto defaulted = defaulted_t{};
```

{:.prompt-info}
> Since we do have some languages that don't have defaults, we could conceivably leave off the `lang::defaulted_t` alternative for their respective `input_t` fields. However, doing so would add a lot of complexity to our code, and this way too we can end up giving a better diagnostic than the compiler itself would if we tried to default a non-defaultable field.

Our constructor for a `translation_set` can then look like this:

```cpp
consteval explicit translation_set(input_t input)
:
    strings([:
        impl::defaulted_strings(^^strings_t)
    :])
{
    for (const auto target_language : enumerators_of(^^lang::language)) {
        const auto input_value = resolve_input_string(input, target_language);

        using StringPtr = string (strings_t::*);

        const auto target_string_ptr = extract<StringPtr>(
            impl::field_for_enumerator(^^strings_t, target_language)
        );

        this->strings.*target_string_ptr = input_value;
    }
}
```

There is... a bit going on here.

First of all, we have to fill in our `strings` field in with something. It would be nice if we could just default-construct it, but unfortunately we can't because `std::format_string`s have no default constructor.

So, instead we splice in our own "defaulted" `strings_t` object that we construct in the `impl::defaulted_strings` function:

```cpp
template<typename Strings, std::size_t NumFields>
constexpr inline auto defaulted_strings_helper = []<std::size_t... Indices>(std::index_sequence<Indices...>) {
    return Strings{
        ((void) Indices, "")...
    };
}(std::make_index_sequence<NumFields>{});

consteval auto defaulted_strings(std::meta::info strings_type) -> std::meta::info {
    const auto num_fields = nonstatic_data_members_of(
        strings_type, std::meta::access_context::unchecked()
    ).size();

    return substitute(^^impl::defaulted_strings_helper, {
        strings_type, std::meta::reflect_constant(num_fields)
    });
}
```

We end up creating an object that looks like `strings_t{"", "", /* And so on... */}`, and using that as our default. An empty string will thankfully work just fine for any `std::format_string<...>`, so we don't have to do anything *too* weird.

{:.prompt-info}
> In `defaulted_strings_helper`, we do the familiar deduction step with `std::make_index_sequence`, but in C++26 you should be able to use structured binding packs to simplify that code.
>
> I don't do that here because GCC trunk was behaving strangely with it, so I decided to instead just do what we're used to.

And then once we have that, we're free to move forward with our constructor:

```cpp
for (const auto target_language : enumerators_of(^^lang::language)) {
    const auto input_value = resolve_input_string(input, target_language);

    using StringPtr = string (strings_t::*);

    const auto target_string_ptr = extract<StringPtr>(
        impl::field_for_enumerator(^^strings_t, target_language)
    );

    this->strings.*target_string_ptr = input_value;
}
```

We hide the meat of the logic away in `resolve_input_string`, which I'll get to, but first I want to talk about this `extract` thing we're doing here.

We're looping over the enumerators of `lang::language` with a normal `for` loop instead of a `template for` expanded loop. That means that we can't get the `target_language` as a constexpr variable.

That then means that we can't get the field to set on our `strings` member as a constexpr variable either, which means we can't just splice it in to assign to it.

So instead what we do is `extract` the member pointer out of the given field, and then use that to assign to the field.

We could use a `template for` here of course, but this feels like it *might* be easier on the compiler? Since it gets to just step through the constant evaluation without having to unroll our loop.

And we'll see anyways too that we need to do a similar thing in our `resolve_input_string` function:

```cpp
static consteval auto resolve_input_string(const input_t &input, std::meta::info target_language) -> string {
    auto current_language = target_language;

    while (true) {
        using InputPtr = string_input (input_t::*);

        const auto input_ptr = extract<InputPtr>(
            impl::field_for_enumerator(^^input_t, current_language)
        );

        const auto &input_field = input.*input_ptr;

        const auto resolved = input_field.visit(impl::overloaded{
            [&](lang::defaulted_t) -> std::optional<string> {
                const auto default_language = impl::default_for_language(
                    current_language
                );

                if (not default_language) {
                    throw "Cannot default language which has no default";
                }

                current_language = *default_language;

                return std::nullopt;
            },

            [](string str) -> std::optional<string> {
                return str;
            }
        });

        if (resolved) {
            return *resolved;
        }
    }
}
```

Here we start with the given "target" language, and then we keep hopping from one input field to the next until we find a field that has its own string given.

And because here we use a `while (true)` loop, we have to do our `extract` trick where we get the member pointer out of the field, since we'd certainly not be able to get it as a `constexpr` variable to splice in.

We then visit the variant at that field, using this (likely familiar) `impl::overloaded` construct that puts a bunch of callables into an overload set:

```cpp
template<typename... Callables>
struct overloaded : Callables... {
    using Callables::operator ()...;
};
```

When we see a `lang::defaulted_t`, then we get the default for the current language we're checking, and if there is no default for the language then we throw an error. If it does have a default though, then we replace the current language with that default, and continue onto the next loop iteration.

And if we see a `string`, then we return out of the function, returning that string.

### Results

And that's it. Now we can construct our translation set like so:

```cpp
constexpr inline auto example = lang::translation_set<>({
    .en_US = "Imagine this is English.",
    .en_GB = lang::defaulted,
    .en_AU = lang::defaulted,

    .de_DE = "Imagine this is German.",
    .de_CH = lang::defaulted
});
```

You can see it all here [in Compiler Explorer](https://compiler-explorer.com/z/1dzrao6Ms).

So... how do we like this?

Well. I definitely like the end product of the API we get to use. It's a nice API, and maybe that's all that ultimately matters.

But I'm not so much a fan of how our implementation just kind of exploded in order to add this. It did feel nicer while writing it than while explaining it, but that wouldn't seem a good signal for maintainability.

Personally, I'm torn. Normally I wouldn't feel too bad about what I had to do to achieve a desired API, but our prior code's implementation felt so pleasant, so simple.

This does not scratch that same itch for me, but that might be alright. It's still reasonable code, I think. And we achieve a clear benefit with it.

It's up to personal taste and project needs perhaps, then.

## Even More Explicit Language

But if we were to stick with this design, then is there more we could do with it?

For instance, what if we wanted to give the user more control over how a given string gets defaulted?

Maybe there's a scenario where in a translation set, both the `en_US` and the `en_GB` strings are specified, but for whatever reason the `en_US` string is the better default for the `en_AU` string than the `en_GB` string would be.

We might want then to be able to write something like this:

```cpp
constexpr inline auto example = lang::translation_set<>({
    .en_US = "Imagine this is American or Australian English.",
    .en_GB = "Imagine this is British English",
    .en_AU = en_US,

    .de_DE = "Imagine this is German.",
    .de_CH = lang::defaulted
});
```

We can actually write exactly that, and it's a pretty small change too.

First we add the `lang::language` type to our `string_input` variant:

{:.no-line-numbers}
```cpp
using string_input = std::variant<string, lang::defaulted_t, lang::language>;
```

And then we accordingly need to add a handler for a `lang::language` in our `resolve_input_field` function:

```cpp
const auto resolved = input_field.visit(impl::overloaded{
    [&](lang::defaulted_t) -> std::optional<string> {
        /* ... */

        return std::nullopt;
    },

    [&](lang::language language) -> std::optional<string> {
        current_language = impl::enumerator_for(
            std::meta::reflect_constant(language)
        );

        return std::nullopt;
    },

    [](string str) -> std::optional<string> {
        return str;
    }
});
```

In the handler, we just set the current language to the enumerator for the given `lang::language`, and continue onto the next loop iteration.

And then we just have to add a `using enum` statement to be able to use `en_US` unqualified, and we get this:

```cpp
namespace translations {

    using enum lang::language;

    constexpr inline auto example = lang::translation_set<>({
        .en_US = "Imagine this is American or Australian English.",
        .en_GB = "Imagine this is British English",
        .en_AU = en_US,

        .de_DE = "Imagine this is German.",
        .de_CH = lang::defaulted
    });

}
```

### Results

Here it is [in Compiler Explorer](https://compiler-explorer.com/z/Tocrr5dnM).

I think that's pretty cute. And in terms of implementation complexity, I don't find it any more complicated than what we had already written to get our explicit defaulting.

The question more becomes whether this actually adds any value. Even if this were to ease any real friction, it does add more to what the user needs to think about when reading their code.

And, it is actually possible with this for the user to get themselves into an infinite loop, if they set two language strings to be equal to each other. That can be diagnosed, but then that adds more implementation complexity.

Regardless, it's another option that we have available to us, and it does feel empowering that we can achieve it.

## The Next Generation

Before we close out, I also wanted to show off code generation a little bit.

Unfortunately, code generation didn't make the cut for C++26, but as far as I know the hope is to get it into C++29.

We can do a lot with what we have in C++26, and what I've shown in this blogpost is honestly just one small part of that. But code generation holds even more potential to empower us and our codebases.

And there's a fair few things we could conceivably do with it here, but specifically I want to show how it can solve two slight issues we ran into earlier.

***

The first issue is that we needed to go from constructing a `translation_set` like this:

```cpp
lang::translation_set<>{
    .en_US = "Imagine this is American English.",
    /* And so on... */
}
```

To like this, with the parentheses surrounding the braces:

```cpp
lang::translation_set<>({
    .en_US = "Imagine this is American English.",
    /* And so on... */
})
```

And that was because in C++26 we can't dynamically add members to a class which we're in the process of defining. But with code generation as is currently proposed [in P3294](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p3294r2.html), we can do just that:

```cpp
template<typename... Args>
struct translation_set {
    using string = std::format_string<const Args &...>;

    consteval {
        for (const auto enumerator : enumerators_of(^^lang::language)) {
            const auto name = identifier_of(enumerator);

            queue_injection(^^{
                string \id(name);
            });
        }
    }

    /* ... */
};
```

Here we just enter into a `consteval` block and inject all the field declarations using the `queue_injection` function, which queues the given token sequence to be injected at the end of the `consteval` block.

And so the fields all get added as if we had just written all that code ourselves. There's no need for any interceding parentheses to construct a `translation_set` then, just as there isn't with our original code.

***

The second issue that code generation can help us with is the slight machine-code difference between our `string_for_language` implementations.

We originally had it defined like this:

```cpp
constexpr auto string_for_language(lang::language language) const -> string {
    switch (language) {
        using enum lang::language;

        default:
        case en_US: return this->en_US;

        case en_GB: return this->en_GB;
        case en_AU: return this->en_AU;

        case de_DE: return this->de_DE;
        case de_CH: return this->de_CH;
    }
}
```

And we replaced it with a `template for`-based implementation so that we didn't have to manually write each case. And we found that these two implementations produced very similar but *slightly* different machine-code.

But with code generation, we can just write the `switch` statement how we had it originally:

```cpp
template<typename... Args>
struct translation_set {
    /* ... */

    consteval {
        auto cases = std::meta::list_builder(^^{});

        for (const auto enumerator : enumerators_of(^^lang::language)) {
            const auto name = identifier_of(enumerator);

            cases += ^^{
                case [: \(enumerator) :]:
                    return this->\id(name);
            };
        }

        queue_injection(^^{
            constexpr auto string_for_language(lang::language language) const -> string {
                switch (language) {
                    /* NOTE: The first case will be the default case. */
                    default:
                    \tokens(cases)
                }
            }
        });
    }
};
```

Here we just build up all the cases for each enumerator, and then we paste them into the larger token sequence we inject, which is the whole `string_for_language` method outright.

{:.prompt-info}
> You should be able to inject the `switch` statement while *inside* the `string_for_language` method, instead of injecting the whole method definition. But when I tried it, the (experimental) compiler was somewhat confused about it.

We don't even need to use any `impl::field_for_enumerator` function or anything. We can just paste in the name of the enumerator, like we would have if we had written this normally. There's no need for us to go searching through the fields.

### Results

We can put this together and see it work [in Compiler Explorer](https://compiler-explorer.com/z/aWWfjMnor) thanks to the experimental reflection version of the EDG compiler.

We can't see the assembly because that's not what EDG produces, but conceptually it should result in the same machine-code as if we had written all the injected code ourselves.

This is really cool, and frankly a lot simpler than even our most simple code we had earlier. We want to materialize code, so we just... materialize the code.

I really really hope something like this gets into C++29.

## Conclusion

As we can see, reflection gives us a whole host of options for how we might improve our code as it stands today.

It really is just programming, and just like with the sort of programming we're used to already, there are a lot of ways one can approach a given problem using reflection programming. That provides us with a lot of opportunities, and that's great and that's empowering.

Some of these options are definitely more complex than the others. And that's true of all programming, we as developers are already empowered to introduce arbitrary complexity to our code.

But reflection also allows us to simplify our code. We can use it to add useful bits and bobs onto our code sure, but it is such a simplifying force as well.

And that is something that we as C++ developers weren't empowered to do before. Reflection lowers the complexity-floor of our codebases in a way that we would struggle to accomplish without it.

And reflection's not just for library code, it's not just for generic code. This translation system I showed here is for a GUI application, and there are a couple other places in its codebase that I want to bring reflection to, like in order to simplify defining and then loading image assets.

We can dream big with reflection, and we can dream small with it too.

[And you're the one who decides the dream.](https://www.youtube.com/watch?v=Ose28F3D6TQ)
