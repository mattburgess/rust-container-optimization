# Static Linking

Static linking instructs the linker to copy all of the required library code into the final executable file. As a result, no dynamic libraries need to be copied into our final container image.

Reliably statically linking our executable is made much easier by using the `musl` C library instead of Glibc, which is the default when using Debian as the compilation host. Glibc makes use of `dlopen()` to dynamically load some libraries at runtime, for example for DNS resolution. So, even if instructed to do so, the Rust toolchain can't guarantee a truly 100% static binary when using Glibc. `musl` doesn't use `dlopen()` so a statically linked binary using that library is the only file required within the container image.

The `Containerfile` below installs the `musl` target for Rust and instructs the compiler to cross-compile to that target.

```dockerfile
{{#include Containerfile.static-linking}}
```

Build the image as usual:

```sh
$ docker build -t localhost/hello-world:latest .
...
Successfully tagged localhost/hello-world:latest

$ docker run localhost/hello-world:latest
Hello, world!

$ docker image ls -a localhost/hello-world:latest
REPOSITORY             TAG         IMAGE ID      CREATED        SIZE
localhost/hello-world  latest      f6e9a5a631f4  9 seconds ago  1.47MB
```

And there we have it; a fully statically linked binary at just under 1.5MiB.

Astute readers may have realised that back in [Implicit Rust Version](./dependencies.md#implicit-rust-version) we mentioned that the Rust image is available in an Alpine Linux variant, which natively targets the `musl` C library. So, why do we go to the trouble of cross-compiling here, instead of just using the Alpine variant?

The main reason is compilation speed. The `musl` library's default memory allocator is notoriously slower than `glibc`'s under heavy multi-threaded workloads[^1]. Because the Rust compiler performs a large number of memory allocations when building any non-trivial codebase, running the compiler inside an Alpine (`musl`) container can significantly slow down your build times. By cross-compiling from a Debian-based (`glibc`) Rust image instead, we get the best of both worlds: a compiler toolchain that is as fast as possible and still outputs a perfectly portable, lightweight `musl` binary. Later on in the book we'll discuss how to avoid the `musl` memory allocator's performance issues in your own applications.

That's about as far as we can take things for the image itself. In the next section, we'll compile a simple web service as a more realistic containerized application before seeing how we can optimize for local build speed/fast feedback loops.

[^1]: This allocation overhead is documented in the upstream Rust Issue Tracker: [Performance issues with musl binaries](https://github.com/rust-lang/rust/issues/70108), which outlines the comparative latency of the default allocator under high parallel loads.
