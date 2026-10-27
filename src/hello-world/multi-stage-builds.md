# Multi-Stage Builds

A multi-stage container build process allows our final container image to use a different base image than the one used to compile the Rust code. This should allow us to substantially reduce the image size by omitting any unnecessary compiler toolchain and other OS binaries.

The `Containerfile` below shows a simple multi-stage build:

```dockerfile
{{#include Containerfile.multi-stage-builds}}
```

Note that the build process remains the same as before:

```shell
$ docker build -t localhost/hello-world:latest .
[1/2] STEP 1/3: FROM public.ecr.aws/docker/library/rust:1.99.1-slim-trixie AS build
[1/2] STEP 2/3: COPY . .
--> fd4b02e7ac79
[1/2] STEP 3/3: RUN cargo build --release
   Compiling hello-world v0.1.0 (/)
    Finished `release` profile [optimized + debuginfo] target(s) in 2.22s
--> c4989fbdf675
[2/2] STEP 1/3: FROM scratch
[2/2] STEP 2/3: COPY --from=build     target/release/hello-world     target/release/hello-world.dwp     /
--> d094873dcc01
[2/2] STEP 3/3: ENTRYPOINT [ "/hello-world" ]
[2/2] COMMIT localhost/hello-world:latest
--> da01a2314163
Successfully tagged localhost/hello-world:latest
```

The image is now incredibly small, and as we'd expect given the `Containerfile` above, the same size as our combined binary + debuginfo:

```shell
$ docker image ls -a localhost/hello-world:latest
REPOSITORY             TAG         IMAGE ID      CREATED        SIZE
localhost/hello-world  latest      d93833c4e004  3 minutes ago  1.31MB
```

Unfortunately, there's just one minor problem...it doesn't actually work:

```shell
$ docker run localhost/hello-world:latest
{
  "msg": "exec container process (missing dynamic library?) `\/hello-world`: No such file or directory",
  "level": "error",
  "time": "2026-10-13T21:38:03.601702Z"
}
```

The `missing dynamic library?` error is the smoking gun! By default, the Rust toolchain creates binaries that are dynamically linked to any required supporting libraries. Even a program as simple as our "Hello World" needs some assistance to print a message to the console. So, how do we find out what libraries we also need to copy into the image? The `ldd` program is designed for exactly this purpose, and we can run it from inside the `build` stage after the binary has been compiled:

```dockerfile
{{#include Containerfile.multi-stage-builds-ldd}}
```

Running the usual `docker build` command prints out the following:

```shell
[1/2] STEP 4/4: RUN ldd target/release/hello-world
 linux-vdso.so.1 (0x00007faf0be60000)
 libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007faf0bdd4000)
 libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007faf0bbe0000)
 /lib64/ld-linux-x86-64.so.2 (0x00007faf0be62000)
```

Now we have a full list of what the binary requires, we can adjust the Containerfile and try again:

```dockerfile
{{#include Containerfile.multi-stage-builds-libs}}
```

```shell
$ docker build -t localhost/hello-world:latest .
...
$ docker run localhost/hello-world:latest
Hello, world!
```

Excellent, it works again! But what about the size?

```shell
$ docker image ls -a localhost/hello-world:latest
REPOSITORY             TAG         IMAGE ID      CREATED        SIZE
localhost/hello-world  latest      fd31f13955bf  2 minutes ago  3.73 MB
```

Just under 4MiB. Definitely not bad compared to the 1.69GiB initial image, but there's still room for improvement. The above approach has a couple of drawbacks:

1. We need to keep track of which dynamic libraries our binary depends on and add the relevant `COPY` directives to the `Containerfile`; as we add more Rust dependencies (crates) then that list may well grow, e.g. pulling in SSL or database client libraries.
1. We copy the whole of each dynamic library even if we only use a small portion of its functionality. e.g. `libc.so.6` comprises 2.0MB of our total 3.73MB image size.

In the next section, we'll see how statically linking our binary can address both of those drawbacks.
