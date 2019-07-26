---
title: Messing with Macro in Rust
permalink: blog/rust_macros/
---

# Messing with Macros and Enums in Rust

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

So let's try making the implementation of traits less painfull:

```rust
macro_rules! apply_on_match {
    ($self:ident, $function:ident, $($type:ident),*) => {
        match &$self {
            BigEnum::$type(x) => $function(x),
        }
    }
}
```

So, yeah, pretty simple !

We first take the name of the function to apply,
and then the names of the different types, and for each type we generate a ``match`` arm, and apply the function.

So now the implementation of traits would look like:

```rust
impl FooTrait for BigEnum {
    fn foo(&self) {
        let tmp_foo = |x| x.foo();
        apply_on_match!(self, tmp_foo, Type1, Type2, ..., TypeN);
    }
}
```

Unfortunately, this does not work because Rust ask for a type for the function's input...

Let's try to find a workaround..

A solution for the function, would be to pass the function's names and its arguments to the macro:

```rust
macro_rules! apply_function {
    ($selem:ident, $function:ident, $($arg:expr),*) => {
        $elem.$function($($arg),*)
    }
}
```

The thing is that having the types and the functions arguments in the same macro call looks kind of ugly....

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
            ($self:ident, $function_name:ident, $($arg:expr),*) => {
                match &$self {
                    $(
                        BigEnum::$type(x) => x.$function($($arg),*),
                    )*
                }
            }
        }
    }
}
```

And we would have to call once: 

```rust
generate_macro_trait_impl!(Type1, Type2, ..., TypeN);
```

to generate the macro ``apply_on_match!``.
We could then apply this macro during trait implementations:

```rust
impl FooTrait for BigEnum {
    fn foo(&self) {
        apply_on_match!(self, foo, )// In our case, `foo` does not take any argument
    }
}
```

And this does look very good !

But ...

It does not work ....

[This issue](https://github.com/rust-lang/rust/issues/35853) explains that it is not possible to have nested macros with repetition patterns...

However, this issue also give a trick to avoid this problem: passing the ``$`` sign as a token to escape the nested ``$`` sign.

The magical macro given in the isssue is the following:

```rust
macro_rules! with_dollar_sign {
    ($($body:tt)*) => {
        macro_rules! __with_dollar_sign { $($body)* }
        __with_dollar_sign!($);
    }
}
```

Now, to adapt our ``generate_macro_trait_impl!`` macro, we (almost) just need to replace every ``$`` sign in the nested macro by the token ``$``:

```rust
macro_rules! generate_macro_trait_impl {
    ($($type:ident),*) => {
        with_dollar_sign! {
            ($d:tt) => {
                macro_rules! apply_on_match {
                    ($d self:ident, $d function_name:ident, $d ($d arg:expr),*) => {
                        match &$d self {
                            $(
                                BigEnum::$type(x) => x.$d function($d ($d arg),*),
                            )*
                        }
                    }
                }
            }
        }
    }
}
```

And that's it ! It works now !!

## Example

Imagine that the ``Struct``s implement ``Serialize``.
You can easily derive the ``Serialize`` trait on ``BigEnum``, but you would also have the ``Type`` wrapping the ``Struct``:

If ``Struct1`` is :

```rust
struct Struct1 {
    x: f64,
    y: f64,
}
```
then the JSON resulting of the serialization of ```Type1(Struct1)``` would be:

```json
Type1({ x: 3.14, y: 2.78 })
```

But we would like to keep only the JSON of ``Struct1``.

So, by looking at the [Serialize trait](https://docs.serde.rs/serde/ser/trait.Serialize.html), we see that we can use our macro to generate ta matching pattern to solve this problem:

```rust
impl Serialize for BigEnum {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error> 
    where
        S: Serializer
    {
        match &self {
            BigEnum::Type1(x) => x.serialize(serializer),
            BigEnum::Type2(x) => x.serialize(serializer),
            // ...
            BigEnum::TypeN(x) => x.serialize(serializer),            
        }
    }   
}
```

But this is the perfect use for our new macros !

Let's suppose that we already have called the ``generate_macro_trait_impl!`` macro.

The implementation of the trait is reduced to:

```rust
impl Serialize for BigEnum {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error> 
    where
        S: Serializer
    {
        apply_on_match!(self, serialize, serializer)
    }   
}
```

## One step further ...

As we said, we have here a lot of elements in the enum.
We managed to reduce the number of times we had to write those elements, and now we have at least two pieces of code with all the elements :

* the definition of the enum

* the call to the ``generate_macro_trait_impl!`` macro

And, as writing something twice is too much, we can add the defintion of the enum in the ``generate_macro_trait_impl!`` macro.

```rust
macro_rules! generate_enum_and_apply_macro {
    ($name:ident, $($type:ident),*) => {
        pub enum $name {
            $(
                $type($type),
            )*
        }
        
        with_dollar_sign! {
            ($d:tt) => {
                macro_rules! apply_on_match {
                    ($d self:ident, $d function_name:ident, $d ($d arg:expr),*) => {
                        match &$d self {
                            $(
                                $name::$type(x) => x.$d function($d ($d arg),*),
                            )*
                        }
                    }
                }
            }
        }
    }
}
```

With this macro, we would only need one call to define everything:

```rust
generate_enum_and_apply_macro!(BigEnum, Struct1, Struct2, ..., StructN);
```


So now the ``enum`` would look like:

```rust
pub enum BigEnum {
    Struct1(Struct1),
    Struct2(Struct2),
    // ...
    StructN(StructN)
}
```

And we achieved our goal: We only have one instance of all the ``Struct``s !!

[Link to the playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=7edd006a46d71b6cf8666ec4b0eddda5)
