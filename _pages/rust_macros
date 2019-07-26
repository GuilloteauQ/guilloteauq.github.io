---
title: Messing with Macro in Rust
permalink: /blog/rust_macros/
---

# Messing with Macro in Rust

## Intro

Ok, some context first.

For a project, I was working with a big ``enum``, like about 20 elements in it.

Something like: 

```rust
pub enum BigEnum {
    Type1(Struct1),
    Type2(Struct2),
    // ...
    TypeN(StructN),
}
```

The thing was that all the ``Struct``s implement the same set of traits.
So I had to write something like:

```rust
impl TraitFoo for BigEnum {
    fn foo(&self) {
        match &self {
            BigEnum::Type1(x) => x.foo(),
            BigEnum::Type2(x) => x.foo(),
            // ...
            BigEnum::TypeN(x) => x.foo(),
        }
    }
}
```

But this was waaaay too much pain for me, so I tried to wrap some of this in a macro.

## First Try

My first thought was: "This ``match`` is so boring to write !"

So here is my first attempt at making the implementation of traits less painfull:

```rust
macro_rules! apply_on_match {
    ($function:ident, $($type:ident),*) => {
        match &self {
            BigEnum::$type(x) => function(x),
        }
    }
}
```

So, yeah, pretty simple !

We first take the name of the function to apply,
and then the names of the different types, and for each type we generate a ``match`` arm, and apply the function.

So now the implementation of traits will look like:

```rust
impl FooTrait for BigEnum {
    fn foo(&self) {
        let tmp_foo = |x| x.foo();
        apply_on_match!(tmp_foo, Type1, Type2, ..., TypeN);
    }
}
```

So at this point, we can feel like that we may have written less code, but writing again all those ``Type``s was boring as hell...

And writing more traits implementation will require to write (or copy/paste) this sequence of ``Type``s ... 
it does not really sounds fun to maintain if we have to add or remove a ``Type``....

## Second try: Let's produce a macro with ... a macro

So now the objective if to write only once this sequence of ``Type``s.

So we will try to write a macro that will, itself, write a macro that will produce the ``match``.

```rust
macro_rules! generate_macro_trait_impl {
    // It will only take the Types
    ($($type:ident),*) => {
        // we define the new macro
        macro_rules! apply_on_match {
            // It is now the same macro as in the first try
            ($function:ident) => {
                match &self {
                    BigEnum::$type(x) => function(x),
                }
            }
        }
    }
}
```


