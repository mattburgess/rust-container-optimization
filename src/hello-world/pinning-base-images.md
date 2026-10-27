# Pinning Base Images and Tooling

## Pinning image references

The first 2 issues mentioned in [Initial Observations](./hello-world.md#initial-observations) concerned implicit and explicit build-time dependencies that weren't pinned to specific versions. Both of those can lead to build issues that depend on either the time the build is invoked or the machine on which it is invoked.

### Implicit docker/dockerfile dependency

Different container build engines (e.g. BuildKit, Podman), and different versions thereof, support various features of the `Containerfile`/`Dockerfile` format. If no explicit version of that format is specified, the build engine's default will be used. Therefore, by specifying the exact version of the Dockerfile syntax frontend to use, any differences in support between local developer's machines and CI can be eliminated.

Adding `# syntax=docker/dockerfile:1.28.0` (for example) as the very first line in the Container file will pin the syntax frontend.

### Implicit Rust version

The initial `Containerfile` used `FROM rust` as the source image. With no tag specified, future builds will use the latest version available at that time which can lead to changes in behaviour of various Rust tools.

Changing `FROM rust` to `FROM rust:1.99.1` (for example) will pin the Rust compiler and associated tools to a fixed version.

It's also important to note that the `rust` image is available in both Debian and Alpine variants, and by default any tag that doesn't contain an explicit variant (e.g. `latest`, `1.99.1`) will point to the equivalent "full" Debian variant. For example, `1.99.1` will point at `1.99.1-trixie`.

## Mitigating docker.io rate limits

docker.io rate limits unauthenticated image pulls to 10 per hour per IPv4 address / IPv6 /64 subnet. That is incredibly restrictive for pretty much anyone, but is a particularly acute problem for developers whose machines are  sat behind a corporate proxy and for CI systems.

There are a number of techniques that can be used to mitigate docker.io's throttling:

1. Use either Red Hat's Quay registry (quay.io) or Amazon's Public ECR (public.ecr.aws) which have far more lenient throttling controls.
2. Setup a Personal/Free account on Docker Hub, or for development teams, a Pro/Team/Business account.
3. Implement a private registry that mirrors/caches source images.

While a private registry is recommended for production systems, it's out of scope for this guide. Instead, we'll simply use Amazon's Public ECR.

Our `FROM rust` line can now change to `public.ecr.aws/docker/library/rust:1.99.1-trixie` to both avoid docker.io rate limits and to pin to a fixed variant and version of the image.

## Image size reduction round one

All of the suggestions above only deal with ensuring consistent build dependencies are used, regardless of which machine the image is being built on. We haven't yet done anything to reduce the size of the resultant image.

For our initial attempt at size reduction, we'll use the `slim` variant of the Rust image.

Given all of the advice above, the resultant `Containerfile` is given below:

```dockerfile
{{#include Containerfile.dependencies}}
```

And the results?

```shell
$ docker build -t localhost/hello-world:latest .
STEP 1/4: FROM public.ecr.aws/docker/library/rust:1.99.1-slim-trixie
STEP 2/4: COPY . .
STEP 3/4: RUN cargo build --release
STEP 4/4: ENTRYPOINT [ "/target/release/hello-world" ]

$ docker image ls -a localhost/hello-world:latest
REPOSITORY             TAG         IMAGE ID      CREATED         SIZE
localhost/hello-world  latest      997759c4f16d  10 seconds ago  929 MB
```

The `slim` version of the container reduces the size by ~47%. But 929MiB is still huge for a "Hello World" application, and the security issues mentioned before are still present.

In the next section, we'll shrink the image significantly by using a multi-stage build.
