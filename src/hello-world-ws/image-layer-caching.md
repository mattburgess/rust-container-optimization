# Image Layer Caching

As explained in Docker's own [Docker build cache](https://docs.docker.com/build/cache/) documentation, Docker images are built up as a series of layers:

* Each command in the Dockerfile translates to a layer in the final image
* If a layer changes, because either the command in the Dockerfile changed or the file contents it refers to changes, Docker will rebuild that layer.
* If a layer changes, all subsequent layers are also rebuilt because they might depend on the the changed layer

In order to minimise the amount of work performed during a container build, we we therefore need to structure the Dockerfile / the image layers, such that we invalidate as few layers as possible as infrequently as possible.

As an example of Docker's build caching at work, let's rebuild the image from scratch:

```shell
$ docker image rm localhost/hello-world-ws:latest
$ time docker build -t localhost/hello-world-ws:latest .
[1/2] STEP 1/5: FROM public.ecr.aws/docker/library/rust:1.99.1-slim-trixie AS build
[1/2] STEP 2/5: RUN rustup target add x86_64-unknown-linux-musl
... Lots of compilation steps omitted
Successfully tagged localhost/hello-world-ws:latest
real    0m33.723s
```

And now build the image again:

```shell
$ time docker build -t localhost/hello-world-ws:latest .
[1/2] STEP 1/5: FROM public.ecr.aws/docker/library/rust:1.99.1-slim-trixie AS build
[1/2] STEP 2/5: RUN rustup target add x86_64-unknown-linux-musl
--> Using cache c936677250ecbf5b822d0b10230a60786a21fd0ce276b57ecfa3ed85202be5d4
--> c936677250ec
[1/2] STEP 3/5: COPY . .
--> Using cache f1f9c1faa8733fb0f5c7d9a835ae08406d452f42dd76546fb3fe340b29319d4a
...
real    0m1.866s
```

All steps should have a corresponding `Using cache...` line and with all layers cached our build is pretty quick.

If you make a trivial change to the contents of the working directory then the `COPY . .` layer will be invalidated and that layer and all subsequent layers will be rebuilt:

```shell
$ touch dummy.txt
$ time docker build -t localhost/hello-world-ws:latest .
[1/2] STEP 1/5: FROM public.ecr.aws/docker/library/rust:1.99.1-slim-trixie AS build
[1/2] STEP 2/5: RUN rustup target add x86_64-unknown-linux-musl
--> Using cache 14a2546c4a9cc46331a9ada0fb9e41ad1e89c7627eb5c7d8f8df54aa79934c81
--> 14a2546c4a9c
[1/2] STEP 3/5: COPY . .
--> e863d0111157
...
real    0m24.902s
```

Note how the `COPY . .` step, and all subsequent steps, now no longer have a related `Using cache` message and the build time has shot back up again.

## Ignore unnecessary files

The first way we have of preventing unnecesary cache invalidation is by not copying files that are unrelated to the image build process into the Docker build context. In the example above, we created a `dummy.txt` file that had nothing at all to do with the Rust program that was being compiled, yet doing so caused the `RUN cargo build...` layer (and subsequent layers) to be rebuilt.

The `.dockerignore` file can be used to explicitly omit files and directories from being copied into the Docker build context. Instead of maintaining a deny list of files that we don't want to be copied, we instead explicitly allow files that we know are required to build and run the application. This prevents any accidental or temporary local files from invalidating the build cache:

```txt
# Prevent cache-busting Docker layers by omitting unrelated files from the
# build context

# Ignore everything by default
*

# Allow just enough for the Rust code to build
!Cargo.toml
!Cargo.lock
!src/
```

## Cache dependency builds

As mentioned in the previous section, changing any of our source code will cause a full rebuild of the binary, including all of its dependencies.

In order to avoid that, we can carve out the build of the dependencies into their own image layer so that they only get rebuilt if their version or selected features change. To do that, we'll use a tool called [cargo-chef](https://github.com/LukeMathWalker/cargo-chef).

Copy and paste the following into your `Containerfile`:

```dockerfile
{{#include Containerfile.image-layer-caching}}
```

We'll walk through it section by section:

### Version numbers

```dockerfile
{{#include Containerfile.image-layer-caching:3:6}}
```

Here we just collect all of the version numbers of the various build tools we need into one convenient location at the top of the `Containerfile`. This makes upgrading them much easier.

### The chef image

```dockerfile
{{#include Containerfile.image-layer-caching:8:17}}
```

The chef image is simply the upstream Rust image we've been using all along with the addition of the `cargo-chef` tool. We use another neat tool, `cargo-binstall` to install a binary copy of `cargo-chef` so we don't have to wait for it to be compiled.

### The planner image

```dockerfile
{{#include Containerfile.image-layer-caching:19:21}}
```

I can't explain this better than the upstream project:

> `cargo chef prepare` examines your project and builds a recipe that captures the set of information required to build your dependencies.

The `recipe.json` file it produces is the only input that `cargo chef cook` needs to decide what needs to be built.

### The builder image

```dockerfile
{{#include Containerfile.image-layer-caching:23:30}}
```

This is similar to our previous `builder` image except that compilation has been split into two distinct steps:

* We first copy the `recipe.json` file from the `planner` stage and then have `cargo chef cook` use that to build all of the project's dependencies. Note how the only layer before it is the `recipe.json` file. So it doesn't matter whether we change any of our source code or not; the only thing that will trigger `cargo chef cook` to run is a change to `recipe.json` which, in turn, will only change if the project's dependencies change.
* Finally we copy the full source code and run `cargo build` as normal. With our dependencies already built (or reused from the cache), this step should be relatively quick.

### The scratch image

```dockerfile
{{#include Containerfile.image-layer-caching:32:37}}
```

This is as simple as it was previously. Even though the steps before it have grown a bit more complicated, the end result of them is the same 2 files that need copying to the final image.

## Timings

What's the net effect of the use of `cargo-chef` then?

A "cold" build of the image can be forced using the `--no-cache` option to `docker build`:

```shell
$ time docker build --no-cache -t localhost/hello-world-ws:latest
...
real    0m42.676s
```

If we then change the output of our program from `<h1>Hello, World!</h1>` to `<h1>Hi, World!</h1>` the image builds in a few seconds (be sure not to use the `--no-cache` option from the previous command!):

```shell
$ time docker build -t localhost/hello-world-ws:latest
...
[3/4] STEP 3/6: COPY --from=planner /app/recipe.json recipe.json
--> Using cache b625608b848825320610a9d47c65ad0d0713662f02cc3c4493ead235d9562026
--> b625608b8488
[3/4] STEP 4/6: RUN cargo chef cook --release --target=x86_64-unknown-linux-musl --recipe-path recipe.json
--> Using cache 0eb116fafd54d0eacc64c60e556436648180ea6f98b6f953483cc7d823c08d9b
--> 0eb116fafd54
[3/4] STEP 5/6: COPY . .
--> c8a1da1712e8
[3/4] STEP 6/6: RUN cargo build --release --target=x86_64-unknown-linux-musl
   Compiling hello-world-ws v0.1.0 (/app)
...
real    0m10.970s
```

Note how all steps in the `builder` stage use the cached dependency layers and only the actual `hello-world-ws` binary compilation is performed due to the change in source code, exactly what we wanted. It results in a huge 75% reduction in build time.
