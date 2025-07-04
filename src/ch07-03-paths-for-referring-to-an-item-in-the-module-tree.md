## Paths for Referring to an Item in the Module Tree

To show Rust where to find an item in a module tree, we use a path in the same
way we use a path when navigating a filesystem. To call a function, we need to
know its path.

A path can take two forms:

- An _absolute path_ is the full path starting from a crate root; for code
  from an external crate, the absolute path begins with the crate name, and for
  code from the current crate, it starts with the literal `crate`.
- A _relative path_ starts from the current module and uses `self`, `super`, or
  an identifier in the current module.

Both absolute and relative paths are followed by one or more identifiers
separated by double colons (`::`).

Returning to Listing 7-1, say we want to call the `add_to_waitlist` function.
This is the same as asking: what’s the path of the `add_to_waitlist` function?
Listing 7-3 contains Listing 7-1 with some of the modules and functions
removed.

We’ll show two ways to call the `add_to_waitlist` function from a new function,
`eat_at_restaurant`, defined in the crate root. These paths are correct, but
there’s another problem remaining that will prevent this example from compiling
as is. We’ll explain why in a bit.

The `eat_at_restaurant` function is part of our library crate’s public API, so
we mark it with the `pub` keyword. In the [“Exposing Paths with the `pub`
Keyword”][pub]<!-- ignore --> section, we’ll go into more detail about `pub`.

<Listing number="7-3" file-name="src/lib.rs" caption="Calling the `add_to_waitlist` function using absolute and relative paths">

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch07-managing-growing-projects/listing-07-03/src/lib.rs}}
```

</Listing>

The first time we call the `add_to_waitlist` function in `eat_at_restaurant`,
we use an absolute path. The `add_to_waitlist` function is defined in the same
crate as `eat_at_restaurant`, which means we can use the `crate` keyword to
start an absolute path. We then include each of the successive modules until we
make our way to `add_to_waitlist`. You can imagine a filesystem with the same
structure: we’d specify the path `/front_of_house/hosting/add_to_waitlist` to
run the `add_to_waitlist` program; using the `crate` name to start from the
crate root is like using `/` to start from the filesystem root in your shell.

The second time we call `add_to_waitlist` in `eat_at_restaurant`, we use a
relative path. The path starts with `front_of_house`, the name of the module
defined at the same level of the module tree as `eat_at_restaurant`. Here the
filesystem equivalent would be using the path
`front_of_house/hosting/add_to_waitlist`. Starting with a module name means
that the path is relative.

Choosing whether to use a relative or absolute path is a decision you’ll make
based on your project, and it depends on whether you’re more likely to move
item definition code separately from or together with the code that uses the
item. For example, if we moved the `front_of_house` module and the
`eat_at_restaurant` function into a module named `customer_experience`, we’d
need to update the absolute path to `add_to_waitlist`, but the relative path
would still be valid. However, if we moved the `eat_at_restaurant` function
separately into a module named `dining`, the absolute path to the
`add_to_waitlist` call would stay the same, but the relative path would need to
be updated. Our preference in general is to specify absolute paths because it’s
more likely we’ll want to move code definitions and item calls independently of
each other.

Let’s try to compile Listing 7-3 and find out why it won’t compile yet! The
errors we get are shown in Listing 7-4.

<Listing number="7-4" caption="Compiler errors from building the code in Listing 7-3">

```console
{{#include ../listings/ch07-managing-growing-projects/listing-07-03/output.txt}}
```

</Listing>

The error messages say that module `hosting` is private. In other words, we
have the correct paths for the `hosting` module and the `add_to_waitlist`
function, but Rust won’t let us use them because it doesn’t have access to the
private sections. In Rust, all items (functions, methods, structs, enums,
modules, and constants) are private to parent modules by default. If you want
to make an item like a function or struct private, you put it in a module.

Items in a parent module can’t use the private items inside child modules, but
items in child modules can use the items in their ancestor modules. This is
because child modules wrap and hide their implementation details, but the child
modules can see the context in which they’re defined. To continue with our
metaphor, think of the privacy rules as being like the back office of a
restaurant: what goes on in there is private to restaurant customers, but
office managers can see and do everything in the restaurant they operate.

Rust chose to have the module system function this way so that hiding inner
implementation details is the default. That way, you know which parts of the
inner code you can change without breaking outer code. However, Rust does give
you the option to expose inner parts of child modules’ code to outer ancestor
modules by using the `pub` keyword to make an item public.

### Exposing Paths with the `pub` Keyword

Let’s return to the error in Listing 7-4 that told us the `hosting` module is
private. We want the `eat_at_restaurant` function in the parent module to have
access to the `add_to_waitlist` function in the child module, so we mark the
`hosting` module with the `pub` keyword, as shown in Listing 7-5.

<Listing number="7-5" file-name="src/lib.rs" caption="Declaring the `hosting` module as `pub` to use it from `eat_at_restaurant`">

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch07-managing-growing-projects/listing-07-05/src/lib.rs:here}}
```

</Listing>

Unfortunately, the code in Listing 7-5 still results in compiler errors, as
shown in Listing 7-6.

<Listing number="7-6" caption="Compiler errors from building the code in Listing 7-5">

```console
{{#include ../listings/ch07-managing-growing-projects/listing-07-05/output.txt}}
```

</Listing>

What happened? Adding the `pub` keyword in front of `mod hosting` makes the
module public. With this change, if we can access `front_of_house`, we can
access `hosting`. But the _contents_ of `hosting` are still private; making the
module public doesn’t make its contents public. The `pub` keyword on a module
only lets code in its ancestor modules refer to it, not access its inner code.
Because modules are containers, there’s not much we can do by only making the
module public; we need to go further and choose to make one or more of the
items within the module public as well.

The errors in Listing 7-6 say that the `add_to_waitlist` function is private.
The privacy rules apply to structs, enums, functions, and methods as well as
modules.

Let’s also make the `add_to_waitlist` function public by adding the `pub`
keyword before its definition, as in Listing 7-7.

<Listing number="7-7" file-name="src/lib.rs" caption="Adding the `pub` keyword to `mod hosting` and `fn add_to_waitlist` lets us call the function from `eat_at_restaurant`">

```rust,noplayground,test_harness
{{#rustdoc_include ../listings/ch07-managing-growing-projects/listing-07-07/src/lib.rs:here}}
```

</Listing>

Now the code will compile! To see why adding the `pub` keyword lets us use
these paths in `eat_at_restaurant` with respect to the privacy rules, let’s look
at the absolute and the relative paths.

In the absolute path, we start with `crate`, the root of our crate’s module
tree. The `front_of_house` module is defined in the crate root. While
`front_of_house` isn’t public, because the `eat_at_restaurant` function is
defined in the same module as `front_of_house` (that is, `eat_at_restaurant`
and `front_of_house` are siblings), we can refer to `front_of_house` from
`eat_at_restaurant`. Next is the `hosting` module marked with `pub`. We can
access the parent module of `hosting`, so we can access `hosting`. Finally, the
`add_to_waitlist` function is marked with `pub` and we can access its parent
module, so this function call works!

In the relative path, the logic is the same as the absolute path except for the
first step: rather than starting from the crate root, the path starts from
`front_of_house`. The `front_of_house` module is defined within the same module
as `eat_at_restaurant`, so the relative path starting from the module in which
`eat_at_restaurant` is defined works. Then, because `hosting` and
`add_to_waitlist` are marked with `pub`, the rest of the path works, and this
function call is valid!

If you plan on sharing your library crate so other projects can use your code,
your public API is your contract with users of your crate that determines how
they can interact with your code. There are many considerations around managing
changes to your public API to make it easier for people to depend on your
crate. These considerations are beyond the scope of this book; if you’re
interested in this topic, see [The Rust API Guidelines][api-guidelines].

> #### Best Practices for Packages with a Binary and a Library
>
> We mentioned that a package can contain both a _src/main.rs_ binary crate
> root as well as a _src/lib.rs_ library crate root, and both crates will have
> the package name by default. Typically, packages with this pattern of
> containing both a library and a binary crate will have just enough code in the
> binary crate to start an executable that calls code defined in the library
> crate. This lets other projects benefit from the most functionality that the
> package provides because the library crate’s code can be shared.
>
> The module tree should be defined in _src/lib.rs_. Then, any public items can
> be used in the binary crate by starting paths with the name of the package.
> The binary crate becomes a user of the library crate just like a completely
> external crate would use the library crate: it can only use the public API.
> This helps you design a good API; not only are you the author, you’re also a
> client!
>
> In [Chapter 12][ch12]<!-- ignore -->, we’ll demonstrate this organizational
> practice with a command line program that will contain both a binary crate
> and a library crate.

{{#quiz ../quizzes/ch07-03-paths-sec1.toml}}

### Starting Relative Paths with `super`

We can construct relative paths that begin in the parent module, rather than
the current module or the crate root, by using `super` at the start of the
path. This is like starting a filesystem path with the `..` syntax that means
to go to the parent directory. Using `super` allows us to reference an item
that we know is in the parent module, which can make rearranging the module
tree easier when the module is closely related to the parent but the parent
might be moved elsewhere in the module tree someday.

Consider the code in Listing 7-8 that models the situation in which a chef
fixes an incorrect order and personally brings it out to the customer. The
function `fix_incorrect_order` defined in the `back_of_house` module calls the
function `deliver_order` defined in the parent module by specifying the path to
`deliver_order`, starting with `super`.

<Listing number="7-8" file-name="src/lib.rs" caption="Calling a function using a relative path starting with `super`">

```rust,noplayground,test_harness
{{#rustdoc_include ../listings/ch07-managing-growing-projects/listing-07-08/src/lib.rs}}
```

</Listing>

The `fix_incorrect_order` function is in the `back_of_house` module, so we can
use `super` to go to the parent module of `back_of_house`, which in this case
is `crate`, the root. From there, we look for `deliver_order` and find it.
Success! We think the `back_of_house` module and the `deliver_order` function
are likely to stay in the same relationship to each other and get moved
together should we decide to reorganize the crate’s module tree. Therefore, we
used `super` so we’ll have fewer places to update code in the future if this
code gets moved to a different module.

### Making Structs and Enums Public

We can also use `pub` to designate structs and enums as public, but there are a
few extra details to the usage of `pub` with structs and enums. If we use `pub`
before a struct definition, we make the struct public, but the struct’s fields
will still be private. We can make each field public or not on a case-by-case
basis. In Listing 7-9, we’ve defined a public `back_of_house::Breakfast` struct
with a public `toast` field but a private `seasonal_fruit` field. This models
the case in a restaurant where the customer can pick the type of bread that
comes with a meal, but the chef decides which fruit accompanies the meal based
on what’s in season and in stock. The available fruit changes quickly, so
customers can’t choose the fruit or even see which fruit they’ll get.

<Listing number="7-9" file-name="src/lib.rs" caption="A struct with some public fields and some private fields">

```rust,noplayground
{{#rustdoc_include ../listings/ch07-managing-growing-projects/listing-07-09/src/lib.rs}}
```

</Listing>

Because the `toast` field in the `back_of_house::Breakfast` struct is public,
in `eat_at_restaurant` we can write and read to the `toast` field using dot
notation. Notice that we can’t use the `seasonal_fruit` field in
`eat_at_restaurant`, because `seasonal_fruit` is private. Try uncommenting the
line modifying the `seasonal_fruit` field value to see what error you get!

Also, note that because `back_of_house::Breakfast` has a private field, the
struct needs to provide a public associated function that constructs an
instance of `Breakfast` (we’ve named it `summer` here). If `Breakfast` didn’t
have such a function, we couldn’t create an instance of `Breakfast` in
`eat_at_restaurant` because we couldn’t set the value of the private
`seasonal_fruit` field in `eat_at_restaurant`.

In contrast, if we make an enum public, all of its variants are then public. We
only need the `pub` before the `enum` keyword, as shown in Listing 7-10.

<Listing number="7-10" file-name="src/lib.rs" caption="Designating an enum as public makes all its variants public.">

```rust,noplayground
{{#rustdoc_include ../listings/ch07-managing-growing-projects/listing-07-10/src/lib.rs}}
```

</Listing>

Because we made the `Appetizer` enum public, we can use the `Soup` and `Salad`
variants in `eat_at_restaurant`.

Enums aren’t very useful unless their variants are public; it would be annoying
to have to annotate all enum variants with `pub` in every case, so the default
for enum variants is to be public. Structs are often useful without their
fields being public, so struct fields follow the general rule of everything
being private by default unless annotated with `pub`.

There’s one more situation involving `pub` that we haven’t covered, and that is
our last module system feature: the `use` keyword. We’ll cover `use` by itself
first, and then we’ll show how to combine `pub` and `use`.

{{#quiz ../quizzes/ch07-03-paths-sec2.toml}}

[pub]: ch07-03-paths-for-referring-to-an-item-in-the-module-tree.html#exposing-paths-with-the-pub-keyword
[api-guidelines]: https://rust-lang.github.io/api-guidelines/
[ch12]: ch12-00-an-io-project.html
