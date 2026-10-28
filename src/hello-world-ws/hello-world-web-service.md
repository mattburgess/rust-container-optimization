# Hello World Web Service

While our canonical "Hello World" application was useful so as not to get bogged down in too many technical details about the app itself, and instead concentrate on the container build process, we'll need something slightly more involved in order to highlight further optimization techniques.

For this, we'll use a very simple web service implemented using the [axum](https://github.com/tokio-rs/axum) library.

Let's start off by initializing our new project and adding the dependencies  and features required to run axum's "Hello World" example:

```shell
cargo add axum@0.8.9 tokio@1.54.0 --no-default-features --features axum@0.8.9/http1,axum@0.8.9/tokio,tokio@1.54.0/rt-multi-thread
```

> [!NOTE]
> Disabling a crate's default features and enabling just those features that your program requires minimizes the amount of work the compiler has to do. Cargo will download and compile any dependencies you specify before moving onto your code. The final binary size is unaffected though as the linker will realise that there are no references to the unused code and can therefore eliminate it from the final build artifact.

Change the auto-generated `src/main.rs` to match [axum's "Hello World" example](https://github.com/tokio-rs/axum/blob/main/examples/hello-world/src/main.rs):

```rust
{{#include main.rs}}
```

> [!NOTE]
> The `bind()` address above is slightly different from Axum's example. When running in a containerised environment, a service listening on `localhost` (`127.0.0.1`) cannot receive network traffic originating from outside the container itself. By binding to `0.0.0.0` instead, the application listens on all interfaces inside its isolated network namespace, allowing it to receive traffic forwarded by the container runtime's ingress routing layer. The `docker run` command below explicitly binds the container's port to the host's loopback interface, meaning it isn't accessible externally.

Copy the `Containerfile` we used from the previous section, making 3 minor adjustments to the filenames on the last few lines:

```dockerfile
{{#include Containerfile.hello-web-service}}
```

Also copy the Cargo release profile we used from the previous section into the new projects `Cargo.toml`:

```toml
[profile.release]
panic = "abort"
debug = "line-tables-only"
split-debuginfo = "packed"
lto = true
codegen-units = 1
```

Now build, run, and test it:

```shell
$ docker build -t localhost/hello-world-ws:latest .
...
Finished `release` profile [optimized + debuginfo] target(s) in 20.05s

$ docker run -d -p 127.0.0.1:3000:3000 localhost/hello-world-ws:latest
34c018b38eb40f727498d4f0a572dc6e3ed4f47a695586f9f72df97fb50b7994

$ curl http://127.0.0.1:3000
<h1>Hello, World!</h1>
```

With all of those dependencies added, how's the container size looking?

```shell
$ docker image ls localhost/hello-world-ws:latest
REPOSITORY                TAG         IMAGE ID      CREATED         SIZE
localhost/hello-world-ws  latest      326936a8aeca  43 seconds ago  4.32 MB
```

While the build times and image size have increased quite substantially (~10x and ~3x respectively), that's somewhat expected; a web service is considerably more complex than our previous "Hello World" command line app.

Somewhat more concerning, though, is that if you make a single line change to the code (e.g. making it print `<h1>Hi, World!</h1>` instead), you'll notice that Cargo builds everything, including our dependencies, from scratch! The next section will address that inefficiency.
