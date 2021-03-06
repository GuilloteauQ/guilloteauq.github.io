---
title: Tex-rs
permalink: /projects/tex_rs
---

# Tex-rs
A crate to generate LaTeX files in Rust

Work in progress

Link to the repository: [tex-rs](https://github.com/GuilloteauQ/tex-rs)

## Example of use

```rust
// Defining the output file
let mut f = new_latex_file("output.tex");
// Adding a title to the document
f.title("Example of use of tex-rs");
// Adding an author
f.author("GuilloteauQ");
// Begin the core of the document
f.begin_document();

// Writing an abstract
let mut abstract_bloc = Core::bloc("abstract");
abstract_bloc.add(Core::text("This document is an example of use of Tex-rs"));
abstract_bloc.write_latex(&mut f);

// Creating a new section
let mut sec = Core::section("Examples");
// Creating an itemize bloc
let mut itemize = Core::bloc("itemize");

let countries = vec!["France", "UK", "Germany", "Italy"];
sec.add(Core::text("Here are some countries in Europe"));
for country in countries.iter() {
    itemize.add(Core::item(Core::text(*country)));
}
// Adding the itemize to the section
sec.add(itemize);
// Writing the section in the file
sec.write_latex(&mut f);

f.write_footer();
```

[See the result here !](https://github.com/GuilloteauQ/tex-rs/blob/master/examples/out.pdf)
