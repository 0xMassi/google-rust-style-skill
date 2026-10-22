---
name: google-rust-style
description: "Google's Rust style guide for coding agents: paste-ready rustfmt.toml, clippy.toml and [lints], the judgement rules no tool checks (module layout, imports, API shape, panics, naming), and where Google overrides common Rust advice. Use when writing or reviewing Rust to Google style, or setting up a Rust crate's lint config."
license: CC-BY-4.0
---

# Google Rust style

Source: [Google Rust Style Guide](https://google.github.io/styleguide/rustguide.html), read 2026-10-22 at gh-pages `2ac1584`. Section links below use `G#anchor`, short for `https://google.github.io/styleguide/rustguide.html#anchor`.

This skill holds only what Google adds on top of common Rust advice (borrow before you clone, `thiserror` for library errors, `cargo clippy` clean). Pair it with a general Rust skill for the basics. Where common advice disagrees with Google, Google wins (see [Overrides](#overrides)).

Out of scope: rules tagged Chrome, Fuchsia or Android (small crates, flat `rust_library` layout, `chromium::import!`, no `async`, no `std::thread`, `fuchsia_sync::Mutex`, per-codebase test frameworks).

## Library or application

First decide which kind of crate you are in ([G#api-design-context](https://google.github.io/styleguide/rustguide.html#api-design-context)):

- **Library**: used by other projects and solves a general problem. Every API rule here binds.
- **Application**: one app and one specific problem. API rules are strong recommendations, and simplicity and directness win over reuse.

Rules below marked *library* follow this split. Every other rule binds both kinds: an application with no outside users still takes the full rustfmt and clippy config except the library block, and every layout, import, error, naming and unsafe rule.

## 1. Tool config

Run `cargo clippy --all-targets --all-features -- -D warnings` in CI so every `warn` below fails the build.

`rustfmt.toml`. All three options need nightly rustfmt. Stable 1.98 prints "unstable features are only available in nightly" and ignores them, and the guide lets a stable-only codebase skip them. `max_width = 100` is already rustfmt's default, so it is not set.

```toml
imports_granularity = "Module"
hex_literal_case = "Lower"
format_macro_matchers = true
```

`clippy.toml`. `avoid-breaking-exported-api = false` is load-bearing. Without it, `wrong_self_convention` and `ref_option` skip `pub` items, which are exactly the items Google's rules target.

```toml
avoid-breaking-exported-api = false
allow-unwrap-in-tests = true
disallowed-methods = [
  { path = "std::panic::catch_unwind", reason = "Google Rust style forbids catch_unwind in first-party code" },
  { path = "std::sync::PoisonError::into_inner", reason = "Google Rust style: never reach inside a poisoned mutex" },
  { path = "std::sync::PoisonError::get_ref", reason = "Google Rust style: never reach inside a poisoned mutex" },
  { path = "std::sync::PoisonError::get_mut", reason = "Google Rust style: never reach inside a poisoned mutex" },
]
```

`Cargo.toml`, every crate. Drop `unsafe_code = "forbid"` only in the one crate that isolates unsafe code.

```toml
[lints.rust]
unsafe_code = "forbid"
unstable_features = "deny"

[lints.clippy]
ref_option = "deny"
unwrap_used = "deny"
wildcard_imports = "deny"
needless_for_each = "deny"
undocumented_unsafe_blocks = "deny"
derive_partial_eq_without_eq = "allow"
missing_const_for_fn = "allow"
```

*Library* crates merge these keys into the same two tables.

```toml
[lints.rust]
missing_docs = "warn"

[lints.clippy]
exhaustive_enums = "deny"
exhaustive_structs = "deny"
must_use_candidate = "deny"
return_self_not_must_use = "deny"
missing_panics_doc = "deny"
```

To take a guide-sanctioned exception, write `#[expect(<lint>, reason = "...")]` on the item. The `reason` carries the why, so the item needs no comment.

| Setting | Google rule | Section |
|---|---|---|
| `imports_granularity = "Module"` | A `use` imports from one module; no `::` inside `{}` | [G#use-statement](https://google.github.io/styleguide/rustguide.html#use-statement) |
| `hex_literal_case`, `format_macro_matchers` | Lowercase hex literals; formatted macro matchers | [G#formatting-other](https://google.github.io/styleguide/rustguide.html#formatting-other) |
| `wildcard_imports` | No glob imports. It already exempts Google's allowed cases: `use super::*` in tests, `use Enum::*` inside a function, `pub use sub::*` | [G#glob-imports](https://google.github.io/styleguide/rustguide.html#glob-imports) |
| `ref_option`, plus default `borrowed_box` and `ptr_arg` | Take `&T`, `&[T]`, `&str`, `Option<&T>`, not `&Box<T>`, `&Vec<T>`, `&String`, `&Option<T>` | [G#flexible-references](https://google.github.io/styleguide/rustguide.html#flexible-references) |
| default `should_implement_trait` | `impl FromStr` rather than an inherent `from_str` | [G#known-traits](https://google.github.io/styleguide/rustguide.html#known-traits) |
| default `derivable_impls` | Derive when a derive can express it | [G#derive](https://google.github.io/styleguide/rustguide.html#derive) |
| default `new_without_default` | A type with `new()` implements `Default` | [G#important-traits](https://google.github.io/styleguide/rustguide.html#important-traits) |
| `derive_partial_eq_without_eq = "allow"` | Prefer `PartialEq` over `Eq` unless you mean reflexivity. The nursery lint pushes the other way | [G#important-traits](https://google.github.io/styleguide/rustguide.html#important-traits) |
| default `wrong_self_convention` | `as_`, `to_`, `into_` take the matching `self` | [G#naming-methods](https://google.github.io/styleguide/rustguide.html#naming-methods) |
| default `len_without_is_empty` | `len()` comes with `is_empty()` | [G#expected-api-sets](https://google.github.io/styleguide/rustguide.html#expected-api-sets) |
| `disallowed-methods` catch_unwind | `catch_unwind` is prohibited. The defense-in-depth entry-point wrapper takes an `#[expect]` | [G#catch-unwind](https://google.github.io/styleguide/rustguide.html#catch-unwind) |
| `disallowed-methods` PoisonError | Never reach inside a poisoned mutex | [G#poison](https://google.github.io/styleguide/rustguide.html#poison) |
| `exhaustive_enums`, `exhaustive_structs` | Public enums and all-pub-field structs used by other teams are `#[non_exhaustive]` | [G#non-exhaustive-enums](https://google.github.io/styleguide/rustguide.html#non-exhaustive-enums), [G#non-exhaustive-structs](https://google.github.io/styleguide/rustguide.html#non-exhaustive-structs) |
| `must_use_candidate`, `return_self_not_must_use` | `#[must_use]` on pure functions and on types whose value must be consumed | [G#must-use-on-functions](https://google.github.io/styleguide/rustguide.html#must-use-on-functions), [G#must-use-on-types](https://google.github.io/styleguide/rustguide.html#must-use-on-types) |
| default `unnecessary_unwrap` | Pattern-match instead of check-then-unwrap | [G#dont-check-then-unwrap](https://google.github.io/styleguide/rustguide.html#dont-check-then-unwrap) |
| `unwrap_used`, `allow-unwrap-in-tests` | `expect` over `unwrap`; tests may panic | [G#expect-over-unwrap](https://google.github.io/styleguide/rustguide.html#expect-over-unwrap) |
| `needless_for_each` | A `for` loop over `Iterator::for_each` | [G#combinator-chains](https://google.github.io/styleguide/rustguide.html#combinator-chains) |
| `unsafe_code`, `undocumented_unsafe_blocks`, default `missing_safety_doc` | Unsafe code is isolated in one crate, each block carries `// SAFETY:`, and unsafe fns document their invariants | [G#unsafe](https://google.github.io/styleguide/rustguide.html#unsafe), [G#writing-doc](https://google.github.io/styleguide/rustguide.html#writing-doc) |
| `missing_docs`, `missing_panics_doc` | Document public APIs and every panic on malformed input | [G#writing-doc](https://google.github.io/styleguide/rustguide.html#writing-doc) |
| `unstable_features` | No `#![feature]`. On stable it is already a hard error; the lint covers nightly builds | [G#unstable-features](https://google.github.io/styleguide/rustguide.html#unstable-features) |
| `missing_const_for_fn = "allow"` | A public fn is `const` only if it will stay `const`. The nursery lint pushes the other way | [G#const-functions](https://google.github.io/styleguide/rustguide.html#const-functions) |

## 2. Judgement rules

No tool can check these. `non_exhaustive_omitted_patterns` is the lint the guide names for the matching rule, but it is unknown on stable 1.98 and needs a nightly feature, which would break the no-unstable-features rule. That rule stays here as prose.

**Modules and imports**
- A leaf module with one primary item `Foo` is `mod foo` in `foo.rs`. [G#userdefined-item](https://google.github.io/styleguide/rustguide.html#userdefined-item)
- Non-leaf modules hold only `mod` declarations and re-exports, plus at most a little glue code. A `mod tests` or `#[cxx::bridge]` child still counts as a leaf module. [G#non-leaf-modules](https://google.github.io/styleguide/rustguide.html#non-leaf-modules)
- Define no preludes. Using a third-party prelude is the reviewer's call. [G#glob-imports](https://google.github.io/styleguide/rustguide.html#glob-imports)
- Import a trait inside the function body that uses it, unless the whole file uses it. [G#prefer-local-trait-imports](https://google.github.io/styleguide/rustguide.html#prefer-local-trait-imports)
- Call free functions through their module: `use std::mem;` then `mem::swap(..)`. Calling a name directly is fine when the name is already unambiguous (C++ interop) or the module name collides. [G#toplevel-functions](https://google.github.io/styleguide/rustguide.html#toplevel-functions)
- Name types unqualified: `use regex::Regex;` then `Regex`. Use the module path (`io::Error`) or a rename (`IoError`) for an ambiguous name, a large type family (rustc's `hir`), macros and generated code, and huge import lists. [G#type-names](https://google.github.io/styleguide/rustguide.html#type-names)

**Formatting**
- `#[rustfmt::skip]` is for hand alignment (matrices, rows of similar lines) and generated code only. Personal taste is not a reason. [G#formatting-optout](https://google.github.io/styleguide/rustguide.html#formatting-optout)
- Write invisible characters in literals as escapes, such as `\u{200D}`. Clippy 0.1.98 does not flag a raw zero-width joiner. [G#formatting-other](https://google.github.io/styleguide/rustguide.html#formatting-other)

**Ownership and signatures**
- Model ownership as unique owners plus borrows. Reach for `Rc`/`Arc` only when that structure becomes unreasonably complex, and make the case in review. [G#avoid-rc-arc](https://google.github.io/styleguide/rustguide.html#avoid-rc-arc)
- *Library*: return an iterator (`impl Iterator` or a concrete type) instead of a `Vec<T>`. Return the `Vec` when most callers need one, the implementation naturally builds one, the result will be indexed, the function exists to build a collection, or it hands back an existing vector. *Application*: a `Vec` is fine. [G#return-iterators](https://google.github.io/styleguide/rustguide.html#return-iterators)
- *Library*: take `impl IntoIterator` when the function only iterates. *Application*: `&[T]` is fine. [G#accept-iterators](https://google.github.io/styleguide/rustguide.html#accept-iterators)
- `impl Into<T>` parameters belong in library APIs where call-site ergonomics are paramount. Application code takes `T`. [G#accepting-into](https://google.github.io/styleguide/rustguide.html#accepting-into)
- When a `bool` argument's meaning is unclear at the call site, make it a two-variant `enum`. If you keep the `bool`, name it in an inline comment at the call site: `connect(addr, /* retry= */ true)`. [G#enum-vs-bool](https://google.github.io/styleguide/rustguide.html#enum-vs-bool)
- Choose the error contract:
  - `Option` when the cause is obvious (`checked_add`, a missing value).
  - `Result` when the caller needs the cause to recover (I/O, network).
  - `panic!` for bugs and broken preconditions, which get fixed in code rather than handled at runtime.

  Developer-facing binaries and tests may panic. A codebase may choose `Result` even for broken invariants. [G#reporting-errors](https://google.github.io/styleguide/rustguide.html#reporting-errors)

**Types and traits**
- Use a newtype instead of an alias when the semantics differ, the type deserves its own API, the underlying type may change, or values are easy to mix up (`Day`, `Month`, `Year`). [G#newtypes-vs-type-aliases](https://google.github.io/styleguide/rustguide.html#newtypes-vs-type-aliases)
- Use named fields instead of tuples and multi-field tuple structs. Newtypes and very local tuples are fine. [G#prefer-named-fields](https://google.github.io/styleguide/rustguide.html#prefer-named-fields)
- *Library* code anticipates the trait impls users will need. *Application* code adds them on demand. [G#traits-yagni](https://google.github.io/styleguide/rustguide.html#traits-yagni)
- A trait exists for a generic bound, for `impl Trait`/`dyn Trait` in a signature, or as an extension trait. Everything else is an inherent `impl`. [G#trait-vs-inherent-impl](https://google.github.io/styleguide/rustguide.html#trait-vs-inherent-impl)
- An extension trait is named `FooExt` and owned by whoever owns `Foo`. A foundational library may extend foreign types. An input trait (`DateLike`, implemented on foreign types) is not an extension trait and is fine. [G#extension-traits](https://google.github.io/styleguide/rustguide.html#extension-traits), [G#naming-extension-traits](https://google.github.io/styleguide/rustguide.html#naming-extension-traits)
- Trait impls ([G#important-traits](https://google.github.io/styleguide/rustguide.html#important-traits)):
  - Types with public fields implement the traits all their fields implement.
  - Error types are `Copy` whenever no field prevents it.
  - Implement `Copy` only when the type will stay `Copy`.
  - Most types implement `Clone`. Types that represent an external resource don't.
  - `PartialEq`/`PartialOrd` give users `==`/`<`. Add `Eq`/`Ord` only for reflexivity or a total order.
  - A likely map key or cache key implements `Hash` and `Eq`, plus `Ord` when the type has a clear total order.
- If a `From` impl is a primary way to build a type, also add an inherent `new()` or `from_foo()`. [G#from-impls](https://google.github.io/styleguide/rustguide.html#from-impls)
- An all-pub-field options struct implements `Default` so callers can write `Opts { a: 1, ..Default::default() }`. Don't mark it `#[non_exhaustive]`, which blocks that syntax outside the crate. [G#options-should-implement-default](https://google.github.io/styleguide/rustguide.html#options-should-implement-default), [G#non-exhaustive-structs](https://google.github.io/styleguide/rustguide.html#non-exhaustive-structs)
- Take an `#[expect]` on `exhaustive_enums`/`exhaustive_structs` for a fixed ontology (`Color`, `Rectangle`), a bool-like two-variant enum, or a type whose breakage you will fix yourself. [G#non-exhaustive-enums](https://google.github.io/styleguide/rustguide.html#non-exhaustive-enums)
- When matching a foreign `#[non_exhaustive]` enum, list every known variant and fold the ones you ignore into the wildcard (`Shape::Triangle | _ => {}`). When you care about only one or two variants, use `if let` or `let else`. [G#non-exhaustive-enum-matching](https://google.github.io/styleguide/rustguide.html#non-exhaustive-enum-matching)
- Operator overloads keep their mathematical or logical meaning. [G#operator-overloads](https://google.github.io/styleguide/rustguide.html#operator-overloads)

**Macros**
- Express an API with the first option that works: plain functions, types, traits and modules, then `macro_rules!`, then other code generation, then `proc_macro`. Count the cost: weaker IDE and search support, a local dialect, harder large-scale changes. [G#macros](https://google.github.io/styleguide/rustguide.html#macros)

**Naming**
- Lifetimes get descriptive names, especially on types. Use `'_` where it works. Single letters are fine for the input-to-output link of one method or a local `for<'a>`, at most two of them. [G#lifetimes](https://google.github.io/styleguide/rustguide.html#lifetimes)
- Generic parameters get short names that hint at their main bound (`R: Read`). Use `T`/`U` only for nearly unbounded APIs such as `Vec<T>`. [G#generic-parameters](https://google.github.io/styleguide/rustguide.html#generic-parameters)
- Constructors are `new()`, `new_with_foo()` for alternatives, and `try_new()` when they return a `Result`. Conversions are `from_foo()` backed by a `From` impl. [G#naming-initialization-functions](https://google.github.io/styleguide/rustguide.html#naming-initialization-functions)
- Accessors: `foo()`, `foo_mut()`, `set_foo(v)`, `take_foo()` (resets the field and returns the old value), `replace_foo(v)` (returns the old value), and `with_foo(|foo| ..)` / `with_mut_foo(..)`. Use the closure form when the field sits behind a `Mutex` or `RefCell` (so the guard type stays private) or a setter would need to clone a value that can't be cloned. [G#getters-and-setters](https://google.github.io/styleguide/rustguide.html#getters-and-setters), [G#closure-accessors](https://google.github.io/styleguide/rustguide.html#closure-accessors)
- Spell words out. Keep only abbreviations an outsider knows, such as Wikipedia-common ones like `IpAddr`. Don't drop letters from a word (`cnt`). [G#avoid-abbreviations](https://google.github.io/styleguide/rustguide.html#avoid-abbreviations)

**Docs**
- A doc says what the code does not. Omit one that restates the name and put `#[expect(missing_docs, reason = "...")]` there instead. Self-explanatory enum variants may go undocumented. Don't explain language basics. Examples need to cover the crate somewhere, not every function. Write `///` and `//!` by hand rather than generating them from macros. [G#writing-doc](https://google.github.io/styleguide/rustguide.html#writing-doc)
- Use `//` for implementation notes and internal TODOs. One item can carry both: `///` for users, `//` for maintainers. [G#writing-doc](https://google.github.io/styleguide/rustguide.html#writing-doc)

**Implementation**
- Write unsafe code only when no safe version exists, and wrap it in a safe abstraction, ideally in its own crate with few dependencies. Write a technical justification, per block or for a whole file or build target: why it can't be safe, and the benchmark if the reason is speed. [G#unsafe](https://google.github.io/styleguide/rustguide.html#unsafe)
- Treat `#[doc(hidden)]` items you don't own as private. [G#doc-hidden](https://google.github.io/styleguide/rustguide.html#doc-hidden)
- Nothing runs before `main()`: no `ctor` crate, no life-before-main tricks. [G#global-initialization](https://google.github.io/styleguide/rustguide.html#global-initialization)
- Annotate enough types that a reader familiar with the codebase can tell what was inferred. Put the type on the binding rather than in a turbofish, especially after `.into()`, `.parse()` and long iterator chains. Use `::<_>` where the compiler demands a turbofish. [G#type-inference](https://google.github.io/styleguide/rustguide.html#type-inference)
- Give an `assert!` a message when it can say why the invariant holds, show the offending value, or tell API misuse apart from an internal bug. The message replaces a comment and shows up in logs. A message that only restates the condition adds nothing. [G#assertion-context](https://google.github.io/styleguide/rustguide.html#assertion-context)
- Bind a combinator chain to a named variable before the `for` that consumes it. Rewrite long chains with branches or outside mutation as a loop. Keep short chains whose combinator carries the logic (`any`, `sum`). [G#combinator-chains](https://google.github.io/styleguide/rustguide.html#combinator-chains)
- Join three or more strings with `format!`/`write!` rather than `+`. Fill a literal template with `format!` rather than `str::replace` or a homemade template syntax. [G#prefer-format-or-write](https://google.github.io/styleguide/rustguide.html#prefer-format-or-write)

## Overrides

When common Rust advice, in another skill or in the codebase, says otherwise, follow Google:

| Topic | Common advice | Google |
|---|---|---|
| Import granularity | `imports_granularity = "Crate"` | `"Module"` ([G#use-statement](https://google.github.io/styleguide/rustguide.html#use-statement)). `group_imports` and `reorder_imports` can stay |
| `unwrap`/`expect`, panics | No `unwrap()` or `expect()` outside tests; avoid `panic!` in production | `panic!` for bugs and broken invariants, with `expect` naming the invariant ([G#reporting-errors](https://google.github.io/styleguide/rustguide.html#reporting-errors), [G#expect-over-unwrap](https://google.github.io/styleguide/rustguide.html#expect-over-unwrap)). Enable `unwrap_used`, not `expect_used` |
| Loops vs combinators | Prefer iterator chains over `for` loops, including chains in a `for` header | `for` when clearer, never a chain in the header, avoid `for_each` ([G#combinator-chains](https://google.github.io/styleguide/rustguide.html#combinator-chains)) |
| `impl Into<String>` params | Use them in constructors and builders everywhere | Library-only, and only when ergonomics are paramount ([G#accepting-into](https://google.github.io/styleguide/rustguide.html#accepting-into)) |
| Shared state | `Arc<Mutex<T>>` / `Rc<RefCell<T>>` for shared mutable state | Unique ownership first; `Rc`/`Arc` only when the alternative is unreasonably complex ([G#avoid-rc-arc](https://google.github.io/styleguide/rustguide.html#avoid-rc-arc)) |
| Traits | Design trait hierarchies; add extension traits to any type, such as `StringExt for str` | A trait only for generics, `impl`/`dyn`, or extension; extension traits are owned by the type's owner ([G#trait-vs-inherent-impl](https://google.github.io/styleguide/rustguide.html#trait-vs-inherent-impl), [G#extension-traits](https://google.github.io/styleguide/rustguide.html#extension-traits)) |
| Error types | `thiserror` enums carrying `String` payloads | `Copy` whenever possible ([G#important-traits](https://google.github.io/styleguide/rustguide.html#important-traits)). Give a variant a payload only when the caller needs it |
| `clippy::nursery` | Enable `-W clippy::nursery` | Allow `derive_partial_eq_without_eq` and `missing_const_for_fn`, which push against [G#important-traits](https://google.github.io/styleguide/rustguide.html#important-traits) and [G#const-functions](https://google.github.io/styleguide/rustguide.html#const-functions) |
| `missing_docs` level | `#![deny(missing_docs)]` for libraries | `warn` plus `#[expect]` on trivial items ([G#writing-doc](https://google.github.io/styleguide/rustguide.html#writing-doc)) |
