# Release Builds

At the moment, the compiled binary inside our image is 4.4MiB. By providing a few parameters to the Rust compiler, though, we can improve things quite considerably. The main gains are achieved by using Rust's `release` profile which aims to optimize the compilation and linking process to produce smaller binaries suitable for production use. It does this by, amongst other things, removing information that's only required for debugging during development.

## Release profile tuning

Cargo's configuration file, `cargo.toml` can be used to change the options that the Rust toolchain is invoked with and some of these can have substantial effects on the resultant binary size. Append the following to your hello-world's default `cargo.toml` file:

```toml
[profile.release]
panic = "abort"
debug = "line-tables-only"
split-debuginfo = "packed"
lto = true
codegen-units = 1
```

### panic = "abort"

Instead of unwinding the stack during a panic, this setting immediately terminates the process. Because the binary is running within a container, the application will exit instantly, allowing the container runtime to quickly detect the failure, log the exit code, and start a fresh container.

Using `abort` instead of the default `unwind` removes structural constraints that would otherwise limit compiler optimization:

* Enables Aggressive Code Inlining: The compiler no longer needs to preserve distinct, sequential stack frames for unwinding cleanup, freeing it up to inline functions more aggressively.
* Unlocks Advanced Loop Optimizations: The compiler can freely reorder or eliminate instructions within loops, as it is no longer bound by the strict requirement to drop local variables in a precise sequence if a panic occurs.

### debug = "line-tables-only"

The default for the `release` profile is `debug = false`, which strips all debug-related information from the binary, which also includes human-readable file and line information that would otherwise appear in crash logs. Setting this to `line-tables-only` retains the absolute minimum amount of debug information required to produce meaningful backtraces with filenames and line numbers, without bloating the binary size with variable or type metadata. If the application crashes in production then this information alongside other telemtry (metrics, logs, and traces) can prove invaluable in understanding the root cause.

### split-debuginfo = "packed"

Instead of retaining the above debug information in the binary itself, this option tells the compiler to save the information to a separate file. This generally reduces the size of the image as the information is stored more efficiently in the split out file than in the binary itself.

### lto = true

Instructs the compiler to perform whole-program analysis across the local crate, the standard library, and all third-party dependencies, allowing for deep optimization and dead-code elimination across the entire dependency graph.

### codegen-units = 1

The default for the `release` profile is `codegen-units = 16`, which splits the crate into 16 separate chunks to parallelise compilation at the expense of optimization depth. By forcing a single unit (`codegen-units = 1`), we prevent the codebase from being partitioned. This gives the compiler a completely unified view of our code, allowing the whole-program analysis of `lto = true` to work at maximum efficiency.

### Other options

There are some other options that can help produce smaller binaries. However, they've been explicitly omitted from this guide as their tradeoff against performance or observability are deemed to be not worth it:

* `strip = true` - this would result in the fielename and line number information provided by `debug = "line-tables-only"` being removed
* `opt-level = "z"` - this tells the compiler to optimize for binary size at the expense of speed.

As we're not targeting a resource constrained execution environment, such as embedded hardware, where every byte counts, we opt for observability and execution speed over absolute minimal file size.

## Using the release profile

Adjust the `cargo build` command in your `Containerfile` to instruct it to use the `release` profile. Note that the `ENTRYPOINT` needs to be updated too, as the profile name is included in the output directory structure:

```dockerfile
{{#include Containerfile.release-builds}}
```

Build the container again and confirm it works:

```sh
$ docker build -t localhost/hello-world:latest .
...
Successfully tagged localhost/hello-world:latest


$ docker run localhost/hello-world:latest
Hello, world!
```

And what about its size?

```sh
$ docker run -it --entrypoint /bin/sh localhost/hello-world:latest \
     -c 'ls -lh /target/release/hello-world*'
  -rwxr-xr-x. 2 root root 689K Oct 22 19:51 /target/release/hello-world
  -rw-r--r--. 1 root root   42 Oct 22 19:51 /target/release/hello-world.d
  -rw-r--r--. 2 root root 591K Oct 22 19:51 /target/release/hello-world.dwp
```

We've shrunk the binary from 4.4MiB -> 1.25MiB (taking the split debuginfo file into account), representing ~72% improvemtn whilst maintaining reasonable debugging and speed optimizations for production use.

The image itself is still massive though:

```sh
$ docker image ls -a localhost/hello-world:latest
REPOSITORY             TAG         IMAGE ID      CREATED         SIZE
localhost/hello-world  latest      0528b9293f88  28 seconds ago  1.69 GB
```

The next section will start to make inroads on that by changing the source image used for compilation as well as make our builds more reliable and reproducible.
