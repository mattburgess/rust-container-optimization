# Hello World

To get started, we'll create the stereotypical "Hello World" program and build a container image that can be run locally.

```sh
$ cargo init hello-world
    Creating binary (application) package
note: see more `Cargo.toml` keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
```

Confirm that the program compiles and runs:

```sh
$ cd hello-world
$ cargo build
   Compiling hello-world v0.1.0 (/home/mattburgess/dev/hello-world)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.09s
$ cargo run
cargo run
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.00s
     Running `target/debug/hello-world`
Hello, world!
```

## Our first container image

In order to compile our Rust source code into a container image, we'll need a source image that contains the Rust compiler and associated tools. Create a  `Containerfile` that references the upstream Rust project's container image and compiles the "Hello World" program:

```dockerfile
{{#include Containerfile.hello-world}}
```

Now build our first image and run it to confirm it works as expected:

```sh
$ docker build -t localhost/hello-world:latest .
STEP 1/4: FROM rust
Resolved "rust" as an alias (/etc/containers/registries.conf.d/000-shortnames.conf)
Trying to pull docker.io/library/rust:latest...
Getting image source signatures
Copying blob 4846f2900730 done   | 
Copying blob 4dc3950125eb done   | 
Copying blob 62fe2e747f69 done   | 
Copying blob b2eb680928d4 done   | 
Copying blob 7daf4a4778d1 done   | 
Copying config 73cdc62447 done   | 
Writing manifest to image destination
STEP 2/4: COPY . .
--> e88f80054553
STEP 3/4: RUN cargo build
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.10s
--> 2892840000aa
STEP 4/4: ENTRYPOINT [ "/target/debug/hello-world" ]
COMMIT localhost/hello-world:latest
--> b9310700b81e
Successfully tagged localhost/hello-world:latest
b9310700b81e9701ced36e3364d7799997d97af6eabcdbc5f50a8be75a1f7c0d

$ docker run localhost/hello-world:latest
Hello, world!
```

## Initial Observations

The good news is that the container successfully builds and runs, but there are a number of issues to note:

1. Developers may be running different versions of container build tools. For example, some might use Docker's buildkit whilst others are using Podman. Different versions of those tools may not understand all of the syntax in a given `Containerfile`.
1. `Trying to pull docker.io/library/rust:latest...` - without an explicit tag on the image reference, the `latest` version at the time of building will be used. Whilst the Rust project is very careful to ensure backward compatibility of releases, bugs can creep in leading to linting and compilation failures.
1. docker.io implements some quite restrictive rate limiting for unauthenticated users. This can cause build failures for developers building locally and for automated CI systems
1. ``Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.10s`` - by default, `cargo build` will produce a `debug` build.
1. And last, but certainly by no means least, the image is huge!

   ```sh
   $ docker image ls -a localhost/hello-world:latest
   REPOSITORY             TAG         IMAGE ID      CREATED        SIZE
   localhost/hello-world  latest      b9310700b81e  6 minutes ago  1.69 GB
   ```

   Note that the binary itself is relatively small but with some simple tweaks to the compilation process can be shrunk:

   ```sh
   $ docker run -it --entrypoint /bin/sh localhost/hello-world:latest \
     -c 'ls -lh /target/debug/hello-world'
   -rwxr-xr-x. 2 root root 4.4M Oct 12 20:48 /target/debug/hello-world
   ```

   Not only does the additional size mean _very_ long times to pull the image, it represents a huge security issue; the image has a full Debian OS on it including compilers (obviously) and, as shown above, full shell access. Any vulnerability in any of the packages installed in that image could potentially cause a compromise of your system(s)!

The next section will start to address each of the above points.
