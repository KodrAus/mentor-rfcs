- Feature Name: (fill me in with a unique ident, my_awesome_feature)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Allow structure definitions to supply default values for individual fields, and then allow those fields to be omitted from struct initialisation:

```rust
struct Foo {
    a: &'static str,
    b: bool = true,
    c: i32,
}

let foo = Foo {
    a: "Hello",
    c: 42,
};
```

# Motivation
[motivation]: #motivation

Field defaults address two issues with structure initialisation in Rust: privacy and boilerplate. This is achieved by letting callers omit fields from initialisation when a default is specified for that field. This syntax also allows allows fields to be added to structures in a backwards compatible way, by providing defaults for new fields.

Rust allows you to create an instance of a structure using a literal syntax. This requires all fields in the structure be assigned a value, and can't be used with private fields. It can also be inconvenient for large structures whose fields usually receive the same values.

With the `..` syntax, values for missing fields can be taken from another structure. However, this still requires an already initialised struct after the `..`. It also isn't valid if the struct has innaccessible private fields.

To work around these shortcomings, users can create constructor functions or more elaborate builders. The problem with a constructor is that you need one for each combination of fields a caller can supply. Builders enable more advanced initialisation, but need additional boilerplate.

Field defaults allow a caller to initialise a structure with default values without needing builders or a constructor function:

```rust
struct Foo {
    a: &'static str,
    b: bool = true,
    c: i32,
}

let foo = Foo {
    a: "Hello",
    c: 42,
};
```

# Detailed design
[design]: #detailed-design

## Grammar

In the initialiser for a structure, a default value expression can be optionally supplied for a field:

```
struct_field : ident ':' type_path |
               ident ':' type_path '=' expr
```

The syntax is modeled after `const` expressions. Field defaults are only valid for classic C structures:

```
structure : 'struct' ident '{' struct_field
                               [ ',' struct_field ] *
                           '}' ;
```

## Interpretation

The value of a field default must be a compile-time expression. So any expression that's valid as `const` can be used as a field default. This ensures values that aren't specified by the caller are deterministic and cheap to produce.

Valid:

```rust
struct Foo {
    a: &'static str,
    b: bool = true,
    c: i32,
}
```

Invalid:

```rust
struct Foo {
    a: &'static str,
    b: Vec<bool> = Vec::new(),
                   ^^^^^^^^^^
                   // error: calls in field defaults are limited to struct and enum constructors
    c: i32,
}
```

Field defaults are sugar for the 'real' initialiser, where values for missing fields are added with the supplied default expression. 

```rust
let foo = Foo {
    a: "Hello",
    c: 42,
};
```

Is equivalent to:

```rust
let foo = Foo {
    a: "Hello",
    b: true,
    c: 42,
};
```

When a field isn't supplied, and there is no default available, then the standard _missing field_ error applies.

## Order of Precedence

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

## Deriving Default

When deriving `Default`, supplied field defaults are used instead of the type default. This is a feature of `#[derive(Default)]`.

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

## Examples

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

We can add a new field `b` to this structure with a default value, and the calling code doesn't need to change:

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

By using field defaults, callers can use structure literals without having to know about any private fields:

```rust
mod data {
    pub struct Foo {
        pub a: &'static str,
        pub c: i32,
        private_field: bool = true
    }
}

let foo = data::Foo {
    a: "Hello",
    c: 42,
}
```

# Drawbacks
[drawbacks]: #drawbacks

Field defaults are limited to `const` expressions. This means there are values that can't be used as defaults, such any value that requires allocation, including common collections like `Vec`.

# Alternatives
[alternatives]: #alternatives

## Allow arbitrary expressions instead of just constants

Allowing arbitrary expressions as field defaults would make this feature more powerful. However, limiting field defaults to `const`s maintains the expectation that struct literals are cheap and deterministic. The same isn't true when arbitrary expressions that could reasonably panic or block on io are allowed.

For complex initialisation logic, builders are the preferred option because they don't need to carry this same expectation.

## Allow `Default::default()` instead of just constants

An alternative to allowing any expression as a default value is allowing `Default::default()`, which is expected to be cheap and deterministic:

```rust
struct Foo {
    a: &'static str,
    b: Vec<i32> = Vec::default(),
    c: i32,
}

let foo = Foo {
    a: "Hello",
    c: 42,
}
```

It could be argued that supporting `default()` is an artificial constraint that doesn't prevent arbitrary expressions. The difference is that `Default` has an expectation of being cheap, so using it to inject logic into field initialisation is an obvious code smell.

Allowing functionality to be injected into data initialisation through `default()` means struct literals may have runtime costs that aren't ever surfaced to the caller. This goes against the expectation that literal expressions have small, predictable runtime cost and are totally deterministic (and trivially predictable).

## Type inference for fields with defaults

Reduce the amount of code needed to define a structure by allowing type inference for field defaults:

```rust
struct Foo {
    a: &'static str,
    b = 42,
    c: i32,
}
```

This has the effect of simplifying the definition, but also requiring readers to manually work out what the type of the right-hand-side is. This could be less of an issue with IDE support.

## Explicit syntax for opting into field defaults

Field defaults could require callers to use an opt-in syntax like `..`. This would make it clearer to callers that additional code could be run on struct initialisation, weakening arguments against more powerful default expressions. However it would prevent field default from being used to maintain backwards compatibility, and reduce overall ergonomics.

If it's unclear whether or not a particular caller will use a default field value then its addition can't be treated as a non-breaking change.

The goal of this design is to let users build a struct as if its default fields weren't there.

# Unresolved questions
[unresolved]: #unresolved-questions