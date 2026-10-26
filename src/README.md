# Introduction

Welcome to Rust Container Optimization! This book is intended to offer tips and tricks for optimizing container images, and their build processes, with a specific focus on Rust.

Whilst the code examples throughout the book are relatively simple, some familiarity with the Rust programming language is useful, as is prior experience with the `docker`, `podman` or similar container tools.

## Pre-Requisites

In order to run the examples in this book you will need:

* The Rust compiler and associated tooling (e.g. cargo). See the [Installation](https://doc.rust-lang.org/book/ch01-01-installation.html) section of The Rust Programming Language book.
* Either [Docker](https://www.docker.com/products/docker-desktop/) or [Podman](https://podman.io/) depending on your OS and tooling preference. For consistency, all commands throughout this book refer to the `docker` command line tool, but substituting `podman` will work as they have compatible command line interfaces.

## Contributing

Rust Container Optimization is free and open source. You can find the source code on [GitHub](https://github.com/mattburgess/rust-container-optimization). Feel free to raise issues there if you spot any technical, spelling or grammatical errors.

## License

This project is dual-licensed:

* The **text, explanations, and diagrams** are licensed under the [Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)](https://creativecommons.org/licenses/by-sa/4.0/legalcode.txt) license.
* All **code snippets, Dockerfiles, configurations, and scripts** are licensed under the [MIT License](https://opensource.org/license/mit).
