- Feature Name: (fill me in with a unique ident, my_awesome_feature)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Allow struct definitions to supply default values for individual fields,
and then allow those fields to be omitted from struct initialisation:

```rust
struct Foo {
    a: &'static str,
    b: bool = true,
    c: i32,
}

let foo = Foo {
    a: "Hello",
    c: 42,
}
```

# Motivation
[motivation]: #motivation

Today, Rust allows you to create an instance of a struct using a literal syntax.
This requires all fields in the struct be assigned a value, and can't be used
with private fields.
It can also be inconvenient for large structs that usually receive the same values.

With the `..` syntax, values for missing fields can be taken from another struct.
This is convenient for using `Default` to override just a few values on initialisation:

```rust
#[derive(Default)]
struct Foo {
    a: &'static str,
    b: bool,
    c: i32,
}

let foo = Foo {
    a: "Hello",
    c: 42,
    ..Default::default()
}
```

However, this still requires an already initialised struct after the `..`.

To work around these shortcomings, users can create constructor functions or more elaborate builders.
The problem with a constructor is that you need one for each combination of fields a caller can supply.
Builders enable more advanced initialisation, but need additional boilerplate.

# Detailed design
[design]: #detailed-design

With field defaults a caller can initialise a struct with default values without needing builders
or a constructor function:

```rust
struct Foo {
    a: &'static str,
    b: bool = true,
    c: i32,
}

let foo = Foo {
    a: "Hello",
    c: 42,
}
```

Supplied field values take precedence over field defaults:

```rust
struct Foo {
    a: &'static str,
    b: bool = true,
    c: i32,
}

// `b` is `false`, even though the field default is `true`
let foo = Foo {
    a: "Hello",
    b: false,
    c: i32,
};
```

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

The `#[derive(Default)]` above is functionally equivalent to:

```rust
impl Default for Foo {
    fn default() -> Self {
        Foo {
            a: <&'static str>::default(),
            c: i32::default(),
        }
    }
}
```

The field default should only be used when the caller doesn't supply a value,
to avoid unnecessarily assigning values.

## Allowable Values

Field defaults are modelled on `const`s.
So the type must be supplied, and the value must be a compile-time expression, or a call to `Default::default()`.
This is to ensure default values have an expectation of being cheap.
It also supports default values for collection types like `Vec`.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Alternatives
[alternatives]: #alternatives

## Stick with builders and constructor functions

## Allow arbitrary expressions instead of const

There's an expectation that field defaults will be cheap.
This is especially important for private fields that a caller can't override.

Allowing arbitrary expressions would make this feature more powerful, but at the expense of allowing
functionality to leak into the struct's data.

## Use explicit syntax for defaults

Additional fields can be added to a struct in a non-breaking fashion.
Say we have the following API and consumer:

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

We can add a new field b to this struct with a default value, and the calling code
doesn't need to change:

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

This is the key use-case for avoiding special opt-in syntax like `..` to utilise
default fields.
If it's unclear whether or not a particular caller will use a default field value then
its addition can't be treated as a non-breaking change.

The goal of this syntax is to let users build a struct as if it default fields weren't there.

# Unresolved questions
[unresolved]: #unresolved-questions