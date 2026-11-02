# Sccache

## Introduction

[Sccache] is a compiler caching tool. It is used to avoid recompiling anything that hasn't changed since the last invocation of the compiler. In Rust this can largely be broken down into to areas:

* If we can cache the already compiled dependencies, if we then make any unrelated changes to Cargo.toml then those dependencies shouldn't be recompiled. For example, bumping the version of Axum shouldn't trigger a recompilation of Tokio. Similarly, introducing a brand new dependency shouldn't trigger a recompilation of any existing dependencies.
* If we can cache parts of the main binary compilation process then changing the source of only one part of it should only cause that part to be recompiled. This, however, requires us to structure the code in a specific way which we'll go into later with a more complex example than our current `hello-world-ws`. The use of sccache and the associated `Containerfile` instructions in this section will give you that caching with no additional effort.

## The Containerfile

The full `Containerfile` is provided below, but the specific changes of interest are highlighted in the following sections:

```dockerfile
{{#include Containerfile.sccache}}
```

## Installing and configuring sccache

Sccache is available as a prebuilt binary that can be installed using `cargo binstall`, just like we do with `cargo-chef`. We add the `SCCACHE_VERSION` argument to the top section, and add the installation of sccache in the `RUN cargo binstall...` step of the `chef` stage. Also in the `chef` stage we configure 2 environment variables:

* `RUSTC_WRAPPER=sccache` - instructs `cargo` to use `sccache` instead of `rustc` directly when compiling Rust code
* `SCCACHE_DIR=/sccache` - tells `sccache` where to store its cached output.

## Using cache mounts to store and reuse the cache

The `RUN cargo chef cook` and `RUN cargo build` steps in the `builder` stage now have some `--mount` options passed to them:

```dockerfile
{{#include Containerfile.sccache:29:40}}
```

* `/usr/local/cargo/registry` - Cache Cargo's package/dependency registry
* `/usr/local/cargo/git` - Cache Cargo''s Git-based dependency registry
* `$SCCACHE_DIR` (`/sccache`) - Cache our crate's compilation artifacts

The behaviour of our image builds is now:

* Any change to any dependency in `Cargo.toml` (e.g. features, version number) will recompile that dependency (and any of its sub-dependencies) and our crate only. Unrelated dependencies will, instead, be pulled from the cache. Our crate is recompiled because the rebuild of the `cargo chef build` layer invalidates the `cargo build` layer that follows it.
* Given the trivial nature of our `hello-world-ws` binary crate, any change to `main.rs` will cause recompilation of our app.
