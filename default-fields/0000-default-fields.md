- Feature Name: (fill me in with a unique ident, my_awesome_feature)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Allow struct definitions to supply default values for individual fields, and then allow those fields to be omitted from struct initialisation:

```rust
struct Foo {
    a: &'static str,
    b: bool = true,
    c: i32,
    d: Vec<i32> = Vec::default()
}

let foo = Foo {
    a: "Hello",
    c: 42,
}
```

# Motivation
[motivation]: #motivation

*summary para*

*backwards compatibility*

Today, Rust allows you to create an instance of a struct using a literal syntax. This requires all fields in the struct be assigned a value, and can't be used with private fields. It can also be inconvenient for large structs whose fields usually receive the same values.

With the `..` syntax, values for missing fields can be taken from another struct. For example, using `Default` to override just a few values on initialisation:

```rust
#[derive(Default)]
struct Foo {
    a: &'static str,
    b: bool,
    c: i32,
    d: Vec<i32>
}

let foo = Foo {
    a: "Hello",
    c: 42,
    ..Default::default()
}
```

However, this still requires an already initialised struct after the `..`. It also isn't valid if the struct has private fields.

To work around these shortcomings, users can create constructor functions or more elaborate builders. The problem with a constructor is that you either need one for each combination of fields a caller can supply or expect callers to override values after initialisation, which is inefficient. Builders enable more advanced initialisation, but need additional boilerplate.

# Detailed design
[design]: #detailed-design

With field defaults a caller can initialise a struct with default values without needing builders or a constructor function:

```rust
struct Foo {
    a: &'static str,
    b: bool = true,
    c: i32,
    d: Vec<i32> = Vec::default()
}

let foo = Foo {
    a: "Hello",
    c: 42,
}
```

Field defaults are sugar for the 'real' initialiser, where values for missing fields are added with the supplied default expression.

Supplied field values take precedence over field defaults:

```rust
// `b` is `false`, even though the field default is `true`
let foo = Foo {
    a: "Hello",
    b: false,
    c: 42,
};
```

Supplied field values in functional updates take precedence over field defaults:

```rust
// `b` is `false`, even though the field default is `true`
let foo = Foo {
    a: "Hello",
    c: 0,
    ..Foo {
        a: "Hello",
        b: false,
        c: 0
    }
};
```

The field default should only be used when the caller doesn't supply a value, to avoid unnecessarily assigning values.

When deriving `Default`, field defaults are used instead of the type default.

```rust
#[derive(Default)]
struct Foo {
    a: &'static str,
    b: bool = true,
    c: i32,
}

// `b` is `true`, even though `bool::default()` is `false`
let foo = Foo::default();
```

## Allowable Values

Field defaults use the same syntax as `const`s. So the type must be supplied, and the value must be a compile-time expression, or a call to `Default::default()`. Default field values are expected to be cheap to produce and have basic support for collection types like `Vec`.

# Drawbacks
[drawbacks]: #drawbacks

Field defaults are limited to `const` expressions and calls to `default()`. This means there are values that can't be used as defaults, such any value that requires allocation, including non-empty `Vec`s.

Allowing functionality to be injected into data initialisation through `default()` means struct literals may have runtime costs that aren't ever surfaced to the caller. This goes against the expectation that literal expressions have small, predictable runtime cost and are totally deterministic (and trivially predictable).

# Alternatives
[alternatives]: #alternatives

## Allow arbitrary expressions instead of constants

Allowing arbitrary expressions as field defaults would make this feature more powerful. However, limiting field defaults to `const`s and `default()` maintains the expectation that struct literals are cheap and deterministic. The same isn't true when arbitrary expressions that could reasonably panic or block on io are allowed.

It could be argued that supporting `default()` is an artificial constraint that doesn't prevent arbitrary expressions. The difference is that `Default` has an expectation of being cheap, so using it to inject logic into field initialisation is an obvious code smell.

Ultimately, the combination of `const`s and `default()` strikes the best balance between expressiveness of allowable field default values and constraints on their cost. When using `const` values, there is no runtime overhead for default fields. When using `default()`, the expected overhead is small.

For complex initialisation logic, builders are the preferred option because they don't need to carry this same expectation.

## Don't allow `default()`

## Allow type inference for `default()`

## Explicit syntax for opting into field defaults

Field defaults could require callers to use an opt-in syntax like `..`. This would make it clearer to callers that additional code could be run on struct initialisation, weakening arguments against more powerful default expressions. However it would prevent field default from being used to maintain backwards compatibility, and reduce overall ergonomics.

With no special syntax, additional fields can be added to a struct in a non-breaking fashion. Say we have the following API and consumer:

```rust
mod data {
    pub struct Foo {
        pub a: &'static str,
        pub c: i32,
    }
}

let foo = data::Foo {
    a: "Hello",
    c: 42,
}
```

We can add a new field b to this struct with a default value, and the calling code doesn't need to change:

```rust
mod data {
    pub struct Foo {
        pub a: &'static str,
        pub b: bool = true,
        pub c: i32,
    }
}

let foo = data::Foo {
    a: "Hello",
    c: 42,
}
```

If it's unclear whether or not a particular caller will use a default field value then its addition can't be treated as a non-breaking change.

The goal of this design is to let users build a struct as if its default fields weren't there.

# Unresolved questions
[unresolved]: #unresolved-questions