# Rust Language Server (RLS) - Architecture

## Preface
In addition to the document below, an architecture overview can be found at @nrc's blog post [How the RLS works](https://www.ncameron.org/blog/how-the-rls-works/) (2017). While some bits have changed, the gist of it stays the same.

Here we aim to explain in-depth how RLS obtains the underlying data to drive its indexing features as the context for the upcoming IDE planning and discussion at the 2019 Rust All-Hands.

Also the [rust-analyzer](https://github.com/rust-analyzer/rust-analyzer/blob/1d53f695f0408f47c5cce5cefa471eb0e86b0db7/docs/dev/guide.md) guide is a great resource as it covers a lot of common ground.

## High-level overview

At the time of writing, at the highest level RLS compiles your package/workspace (similar to `cargo check`) and reuses resulting internal data structures of the Rust compiler to power its indexing features.

When initialized, (unless overriden by custom build command) RLS `cargo check`s the current project and collects inter-crate [1] dependency graph along with exact crate compilation invocations, which is used later to run the compiler again itself (but in-process).

In-process compilation runs return populated internal data structures (`rls_data::Analysis`), which are further lowered and cross-referenced to expose a low-level indexing API (`rls_analysis::Analysis`) to finally be consumed by the Rust Language Server in order to answer relevant LSP [2] queries.

The main reason we execute the compilation in-process is to optimize the
latency - we pass the resulting data structures in-memory. For dependencies that
don't change often (non-primary/path dependencies) we perform the compilation out of process
once, where we dump and cache the resulting data into a JSON file, which only needs to be read once at the start of indexing.

[1] *crate* is a single unit of compilation as compiled by `rustc`. For example, Cargo package with bin+lib has *two* crates (sometimes called *targets* by Cargo).

[2] [*Language Server Protocol*](https://microsoft.github.io/language-server-protocol/specification) is a language-agnostic JSON-RPC protocol which serves to expose common language "smartness" operations via standardized interface, regardless of the IDE/editor used.

## Information flow (in-depth)
The current flow is as follows:
```
rustc -> rustc_save_analysis -> rls_data -> rls_analysis -> rls
```

### [rustc_save_analysis](https://github.com/rust-lang/rust/tree/master/compiler/rustc_save_analysis)

The Rust compiler includes the [`rustc_save_analysis`](https://github.com/rust-lang/rust/tree/master/compiler/rustc_save_analysis) crate, which allows to dump the knowledge about the currently compiled crate. The main entry point is [`process_crate`](https://github.com/rust-lang/rust/blob/e08d5693609a659e45025b8ea4dbd9efa342fa68/compiler/rustc_save_analysis/src/lib.rs#L978), which walks the post-macro-expansion AST and [saves](https://github.com/rust-lang/rust/blob/e08d5693609a659e45025b8ea4dbd9efa342fa68/compiler/rustc_save_analysis/src/lib.rs#L1011) the collected knowledge either by [dumping to a JSON file](https://github.com/rust-lang/rust/blob/e08d5693609a659e45025b8ea4dbd9efa342fa68/compiler/rustc_save_analysis/src/lib.rs#L953-L965) or by [calling back with resulting data structure](https://github.com/rust-lang/rust/blob/e08d5693609a659e45025b8ea4dbd9efa342fa68/compiler/rustc_save_analysis/src/lib.rs#L967-L976).

### [rls_data](https://github.com/rust-lang/rls/tree/master/rls-data)

As mentioned previously, the returned data structure is [`rls_data::Analysis`](https://github.com/rust-lang/rls/blob/3df74381f37617ec800537c11fb0c3130f5f3616/rls-data/src/lib.rs#L11-L26) inside the [`rls_data`](https://github.com/rust-lang/rls/tree/master/rls-data) crate:

```rust
/// Basically a type alias, we refer to nodes with HIR ids.
/// All of the below nodes are either identified by or refer to those IDs.
type Id = rustc::hir::def_id::DefId;

pub struct Analysis {
    ...
    /// Contains rustc invocation producing this save-analysis data. Currently
    /// used to support RLS in custom build system context, namely Buck).
    pub compilation: Option<CompilationOptions>,
    /// Path to current crate root file and current crate `GlobalCrateId`. Also
    /// includes externally referred crates' `GlobalCrateId` and their local
    /// `CrateNum` index as stored by rustc from this crate compilation PoV.
    pub prelude: Option<CratePreludeData>,
    /// Nodes of use tree forests, incl. relation, span, optional alias value
    /// and kind (extern crate, simple use or glob).
    pub imports: Vec<Import>,
    /// Main data nodes. Roughly correspond to post-expansion AST nodes, incl.
    /// span, qualified name, relations, optional signature (if a function),
    /// docs and attributes.
    pub defs: Vec<Def>,
    /// Nodes for `impl` items, incl. kind (inherent, trait impl, ...), span,
    /// children ids, docs, attributes and signature.
    pub impls: Vec<Impl>,
    /// Span which refers an `Id` of function/module/type/variable kind.
    pub refs: Vec<Ref>,
    /// Contains callsite and callee span along with macro qualified name.
    pub macro_refs: Vec<MacroRef>,
    /// Impl/SuperTrait relation between `Id`s with an associated span.
    pub relations: Vec<Relation>,
}
```

### [rls_analysis](https://github.com/rust-lang/rls/tree/master/rls-analysis)

This [crate](https://github.com/rust-lang/rls/tree/master/rls-analysis) is responsible for loading and stitching multiple of
the `rls_data::Analysis` data structures into a single, coherent interface.

Whereas `rls_data` format can be considered an implementation detail that might
change, this crate aims to provide a 'stable' API.

Another reason behind that is that each of those structures contains data centric
to the crate that was being compiled - this [lowering](https://github.com/rust-lang/rls/blob/3df74381f37617ec800537c11fb0c3130f5f3616/rls-analysis/src/lowering.rs#L161)
cross-references the data
and indexes it, resulting in a database spanning multiple crates that can be
queried like 'across known crates, find all references to a definition at a
given span' or similarly.

We are capable of updating the index with new crate data. Whenever we encounter
a new crate, we [record and translate](https://github.com/rust-lang/rls/blob/3df74381f37617ec800537c11fb0c3130f5f3616/rls-analysis/src/lowering.rs#L122-L144)
the crate id into our database-wide crate id mapping.

However, if data for an already lowered crate is loaded again, we simply
replace the definitions for a given crate and re-index.

One interesting edge case is when we lower data for crates having the same name, such
as binary and `#[cfg(test)]`-compiled version of it. We need to ensure we lower a given definition
[only once](https://github.com/rust-lang/rls/blob/3df74381f37617ec800537c11fb0c3130f5f3616/rls-analysis/src/lowering.rs#L271-L276)
, even if it technically is repeated across multiple crates.

### rls

With all the data lowering logic in place, all we have to do is actually fetch
the data - that happens inside the RLS.

In general, apart from being an LSP server, the RLS is also concerned with
build orchestration, and coordination of other components, such as

* Racer for autocompletion
* Cargo for project layout detection and initial build coordination
* internal virtual file system (VFS) for handling in-memory text buffers,
* rls-analysis serving as our knowledge database
* Clippy performing additional lints
* Rustfmt driving our formatting capabilities

After doing initial compilation with Cargo, we cache a subgraph of the inter-crate
dependency graph along with compilation invocations and input files for the
crates we're interested in (inside the [primary or path-based packages](https://github.com/rust-lang/rls/blob/3df74381f37617ec800537c11fb0c3130f5f3616/rls/src/build/cargo.rs#L358-L363))
which we later rerun manually.

In our case Cargo is configured to use a separate target directory
(`$target-dir/rls`) so the analysis does not interfere with regular builds.

We hijack the Cargo build process not only to record the build orchestration
data mentioned above but also to inject additional compiler flags forcing the compiler
to dump the JSON save-analysis files for each dependency. This serves as an
optimization so that we don't have to re-check dependencies in a fresh run.

Because RLS aims to provide a truthful view of the compilation and to maintain
parity with regular `cargo check` flow, our Cargo runs also run `build.rs` and
proc macros initially and when needed (e.g. by modifying a file causing
`build.rs` to be rerun).

## Build scheduling

On every relevant file change we [mark files as dirty](https://github.com/rust-lang/rls/blob/3df74381f37617ec800537c11fb0c3130f5f3616/rls/src/actions/notifications.rs#L113) and schedule a normal build.

We currently discern two [build priorities](https://github.com/rust-lang/rls/blob/3df74381f37617ec800537c11fb0c3130f5f3616/rls/src/build/mod.rs#L114-L124):

* Normal
* Cargo

The latter is scheduled whenever a change happened that can impact entire
project. This includes:

* [initial build](https://github.com/rust-lang/rls/blob/3df74381f37617ec800537c11fb0c3130f5f3616/rls/src/actions/mod.rs#L328)
* [configuration change](https://github.com/rust-lang/rls/blob/3df74381f37617ec800537c11fb0c3130f5f3616/rls/src/actions/notifications.rs#L116)
(can potentially build different set of packages)
* [Cargo.toml change](https://github.com/rust-lang/rls/blob/3df74381f37617ec800537c11fb0c3130f5f3616/rls/src/actions/notifications.rs#L264)
(ditto)
* [build directory](https://github.com/rust-lang/rls/blob/3df74381f37617ec800537c11fb0c3130f5f3616/rls/src/build/mod.rs#L468-L472) changed
* [modified file](https://github.com/rust-lang/rls/blob/3df74381f37617ec800537c11fb0c3130f5f3616/rls/src/build/cargo_plan.rs#L350-L354) in a package we didn't build
* [build.rs](https://github.com/rust-lang/rls/blob/3df74381f37617ec800537c11fb0c3130f5f3616/rls/src/build/cargo_plan.rs#L359-L360) modification

On a normal build, we map from dirty files to dirty crates, sort those
topologically and run rustc in-process for each crate ourselves.
With each compilation in-process we directly
[receive `rls_data::Analysis`](https://github.com/rust-lang/rls/blob/3df74381f37617ec800537c11fb0c3130f5f3616/rls/src/build/rustc.rs#L293-L305)
in a callback,
[mark corresponding files as built](https://github.com/rust-lang/rls/blob/3df74381f37617ec800537c11fb0c3130f5f3616/rls/src/build/mod.rs#L478-L490)
and finally
[update our analysis database](https://github.com/rust-lang/rls/blob/3df74381f37617ec800537c11fb0c3130f5f3616/rls/src/actions/post_build.rs#L180-L184)
with currently built data for each rebuilt crate.

If there are still files that are modified after we scheduled a build (user kept
typing), we don't mark it as done yet and schedule a regular build again).
It's worth noting that we squash builds whenever user happened to type before
a build kicked off (we buffer build requests not to waste work on something we
might potentially invalidate).

## I/O

As mentioned previously, we run Cargo using a separate target directory so we
do the same kind of work that Cargo does, in addition to also saving
JSON save-analysis files for our non-path dependencies.

### VFS

To allow running analysis on unsaved in-memory text buffers, we use the
[`rls-vfs`](https://github.com/rust-lang/rls/tree/master/rls-vfs)
crate to act as our virtual file system.

The Rust compiler supports using custom file providers via [`FileLoader`](https://github.com/rust-lang/rust/blame/f19851069efd6ee1fe899a469f08ad2d66e76050/compiler/rustc_span/src/source_map.rs#L98-L105) trait, which [we use](https://github.com/rust-lang/rls/blob/3df74381f37617ec800537c11fb0c3130f5f3616/rls/src/build/rustc.rs#L355-L367).

It delegates to the real file system
whenever there are no buffered changes to a file but serves the unsaved buffers [from the VFS](https://github.com/rust-lang/rls/blob/3df74381f37617ec800537c11fb0c3130f5f3616/rls/src/build/rustc.rs#L54) otherwise.
