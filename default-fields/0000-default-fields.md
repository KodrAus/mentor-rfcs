- Feature Name: (fill me in with a unique ident, my_awesome_feature)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Allow struct definitions to supply default values for individual fields,
and then allow those fields to be omitted from construction:

```rust
struct Foo {
    a: &'static str,
    b: bool = true,
    c: i32
}

let foo = Foo {
    a: "Hello",
    c: 42
}
```

# Motivation
[motivation]: #motivation

Today, Rust allows you to create an instance of a struct using a literal syntax:

```rust
struct Foo {
    a: &'static str,
    b: bool,
    c: i32
}

let foo = Foo {
    a: "Hello",
    b: true,
    c: 42
};
```

This requires all fields in the struct be assigned a value, unless the struct implements
the Default trait.
In this case, values can be optionally provided, and rest assigned their default value:

```rust
#[derive(Default)]
struct Foo {
    a: &'static str,
    b: bool,
    c: i32
}

let foo = Foo {
    a: "Hello",
    c: 42,
    ..Default::default()
}
```

The problem with this is that it requires Foo to derive Default, which may not always
be desirable.

The existing struct construction also can't be used when a struct has private fields:

```rust
mod data {
    pub struct Foo {
        pub a: &'static str,
        b: bool,
        pub c: i32
    }
}

// Error: field `b` is private
let foo = data::Foo {
    a: "Hello",
    b: true,
    c: 42
}
```

To work around these shortcomings, users can create constructor functions or more elaborate builders.
The problem with a constructor is that it needs to be kept in sync with the fields of the struct, or
require the user reassign values they want to change.
Builders enable more advanced construction, but also need to kept in-line with the struct they build,
and can be overkill for large, but uncomplicated structs.

With field defaults a caller can instantiate a struct without needing builders
or a constructor function:

```rust
struct Foo {
    a: &'static str,
    b: bool = true,
    c: i32
}

let foo = Foo {
    a: "Hello",
    c: 42
}
```

This also means additional fields can be added to a struct in a non-breaking fashion.
Say we have the following API and consumer:

```rust
mod data {
    pub struct Foo {
        pub a: &'static str,
        pub c: i32
    }
}

let foo = data::Foo {
    a: "Hello",
    c: 42
}
```

We can add a new field b to this struct with a default value, and the calling code
doesn't need to change:

```rust
mod data {
    pub struct Foo {
        pub a: &'static str,
        pub b: bool = true
        pub c: i32
    }
}

let foo = data::Foo {
    a: "Hello",
    c: 42
}
```

This is the key use-case for avoiding special opt-in syntax like `..` to utilise
default fields.
If it's unclear whether or not a particular caller will use a default field value then
its addition can't be treated as a non-breaking change.

The goal of this syntax is to let users build a struct as if it default fields weren't there.

# Detailed design
[design]: #detailed-design

The semantics of field defaults are the same as consts.
The type must be supplied, and the value must be a valid compile-time constant.

Field defaults take precedence over `Default::default()`.

```rust
#[derive(Default)]
struct Foo {
    a: &'static str,
    b: bool = true
    c: i32
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
			c: i32::default()
		}
	}
}
```

Supplied field values take precedence over field defaults:

```rust
struct Foo {
    a: &'static str,
    b: bool = true
    c: i32
}

// `b` is `false`, even though the field default is `true`
let foo = Foo {
    a: "Hello",
    b: false,
    c: i32
};
```

That means if `Default` is manually implemented for a struct and default fields are
given different values, then those values take precedence:

```rust
struct Foo {
    a: &'static str,
    b: bool = true
    c: i32
}

impl Default for Foo {
	fn default() -> Self {
		Foo {
			a: "Hello",
			b: false,
			c: 42
		}
	}
}

// `b` is `false`, even though the field default is `true`
let foo = Foo::default();
```

This makes sense when you think of `default()` as _just another function_.

The field default should only be used when the caller doesn't supply a value,
to avoid unnecessarily assigning values.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Alternatives
[alternatives]: #alternatives

Just stick with builders and constructor functions.

# Unresolved questions
[unresolved]: #unresolved-questions

## `const`

Is `const` the right feature to base off?
It means field values are just as constrained as `const` is now,
but also prevents struct definitions from becomes bloated by complex logic that
maybe belongs in a builder instead.

## When all fields have defaults

An interesting case is where all fields are given default values:

```rust
struct Foo {
    a: &'static str = "",
    b = true
    c: i32 = 42
}
```

Should this type automatically derive `Default`?
Should we be able to construct this type as if it's a ZST, like `let foo = Foo;`?
