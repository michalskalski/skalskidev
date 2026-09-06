+++
title = "Understanding Nix: A Purely Functional Approach to Package Management"
date = 2025-05-10
+++

_This article is based on the presentation I made a while ago about Nix and NixOS. The presentation [can be seen here](/pub/nix/nix.pdf). I used LLM to help extract information from the slides_

Over the past few years, I've been exploring different approaches to managing software environments and deployments. Among them, Nix has stood out as a fascinating solution that takes a fundamentally different approach. In this post, I'll share what I've learned about Nix and why it might be worth your attention.

## What is Nix?

Nix is simultaneously three interconnected things:

- A **domain-specific language** with functional programming characteristics
- A powerful **[package manager](https://github.com/NixOS/nixpkgs)** with unique properties
- The main pillar of **[NixOS](https://nixos.org/)**, a Linux distribution built around Nix principles

## Origins

Nix began as the PhD project of Eelco Dolstra, crystallized in these academic papers:

- **[Imposing a Memory Management Discipline on Software Deployment (2004)](https://edolstra.github.io/pubs/immdsd-icse2004-final.pdf)**
- **[The Purely Functional Software Deployment Model (2005)](https://edolstra.github.io/pubs/phd-thesis.pdf)**

Initially, Nix was intended to be an alternative to traditional build systems like ```make``` and package managers like ```rpm```. However, it has evolved into something much more comprehensive.

For those interested in a deeper historical perspective, I recommend watching [Eelco Dolstra's talk on The Evolution of Nix](https://www.youtube.com/watch?v=h8hWX_aGGDc).

## Problems Nix Attempts to Solve

Traditional package managers face several persistent challenges that anyone who has managed systems will recognize:

- **DLL hell** - conflicting shared libraries causing application failures
- **Destructive upgrades** - installing a new version overwrites the previous one
- **No rollbacks** - difficult or impossible to return to previous states
- **Not atomic** - system left in inconsistent state if updates are interrupted
- **Hard to prevent undeclared dependencies** - applications might depend on libraries that aren't explicitly listed

These issues stem from the fundamental design of how software is installed and managed in conventional systems.

## Understanding the Traditional Approach: Filesystem Hierarchy Standard

To understand what makes Nix different, let's first look at how traditional systems handle software installation. In standard Linux distributions, packages install files in fixed locations according to the [Filesystem Hierarchy Standard](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard):

```bash
# list of files installed on filesystem by wget pkg
~ ❯ dpkg -L wget
...
/etc/wgetrc
/usr/bin/wget
/usr/share/doc/wget/README
/usr/share/man/man1/wget.1.gz
```

Programs often rely on dynamically linked libraries that are shared across the system:

```bash
# dynamically linked libraries
~ ❯ ldd /usr/bin/wget
libpcre2-8.so.0 => /lib/x86_64-linux-gnu/libpcre2-8.so.0
libuuid.so.1 => /lib/x86_64-linux-gnu/libuuid.so.1
libidn2.so.0 => /lib/x86_64-linux-gnu/libidn2.so.0
libssl.so.3 => /lib/x86_64-linux-gnu/libssl.so.3
libcrypto.so.3 => /lib/x86_64-linux-gnu/libcrypto.so.3
libz.so.1 => /lib/x86_64-linux-gnu/libz.so.1
libpsl.so.5 => /lib/x86_64-linux-gnu/libpsl.so.5
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
/lib64/ld-linux-x86-64.so.2
libunistring.so.2 => /lib/x86_64-linux-gnu/libunistring.so.2
```

## Filesystem as Memory: A Novel Approach

One of Nix's breakthrough insights was drawing a parallel between memory management in programming languages and package management in operating systems. Dolstra visualized the filesystem as analogous to program memory.

Here's how concepts map between programming languages and deployment systems:

| Programming Language Domain | Deployment Domain |
|----------------------------|-------------------|
| memory | disk |
| value, object | file |
| address | path name |
| pointer dereference | file access |
| pointer arithmetic | string operations |
| dangling pointer | path to absent file |
| object graph | dependency graph |
| calling constructed object with reference to other object | runtime dependency |
| calling constructor with reference to other object, not stored | build-time dependency |
| calling constructor with reference to other object, stored | retained dependency |
| languages without pointer discipline (e.g. assembler) | typical Unix-style deployment |
| languages with enough pointer discipline to support conservative garbage collection (e.g. C, C++) | Nix |
| languages with full pointer discipline (e.g. Java, Haskell) | as-yet unknown deployment style not enabled by contemporary operating systems |


This model suggests viewing the file system as memory and package management as memory management. That reshapes how we can think about software deployment.

## Isolation and Reliable Identification

One of the challenge in package management is avoiding file path collisions. When multiple packages want to install files to the same location, conflicts arise. Traditional approaches have limitations:

- Using component name and version alone isn't sufficient (what about different build options?)
- Random allocation would be inefficient and generate duplicates

## The Hash Solution

Nix's approach is to compute a cryptographic hash of all inputs that affect a package, including:

- **The sources** of the components
- **The build script** that performed the build
- **Any arguments or environment variables** passed to the build script
- **All build time dependencies**, including compilers, linkers, libraries, standard Unix tools, shells, etc.

This hash becomes part of the path where the package is stored, ensuring that different versions or builds of the same package can coexist without conflicts.

## The Nix Store

The cornerstone of Nix is the **/nix/store** directory, where all packages are stored using their unique hash identifiers. For example:

```bash
~ ❯ which wget
/nix/store/d3kkv9vjb3ljh7hr5v38gls8iykvwkny-wget-1.21.4/bin/wget

~ ❯ ldd /nix/store/d3kkv9vjb3ljh7hr5v38gls8iykvwkny-wget-1.21.4/bin/wget
libpcre.so.1 => /nix/store/pxl4n1lrns2xhc8f1s04srb4cphlg5cz-pcre-8.45/lib/libpcre.so.1
libuuid.so.1 => /nix/store/y5975fancsig22f6xw22mmmffn19n8zp-util-linux-minimal-2.39-lib/lib/libuuid.so.1
libidn2.so.0 => /nix/store/vh4pdds47783g12fmywazdx3v3kx0j4x-libidn2-2.3.4/lib/libidn2.so.0
libssl.so.3 => /nix/store/ix7cb1isdcdl4gq9hl4pdk6gyc4wrk14-openssl-3.0.9/lib/libssl.so.3
libcrypto.so.3 => /nix/store/ix7cb1isdcdl4gq9hl4pdk6gyc4wrk14-openssl-3.0.9/lib/libcrypto.so.3
libz.so.1 => /nix/store/mgz7sp9jxnk7c3rn1hvich9n0k2rjr7m-zlib-1.2.13/lib/libz.so.1
libc.so.6 => /nix/store/ayg065nw0xi1zsyi8glfh5pn4sfqd8xg-glibc-2.37-8/lib/libc.so.6
libunistring.so.5 => /nix/store/aw137ya6rvy61zw8ydsz22xwarsr8ynf-libunistring-1.1/lib/libunistring.so.5
libdl.so.2 => /nix/store/ayg065nw0xi1zsyi8glfh5pn4sfqd8xg-glibc-2.37-8/lib/libdl.so.2
libpthread.so.0 => /nix/store/ayg065nw0xi1zsyi8glfh5pn4sfqd8xg-glibc-2.37-8/lib/libpthread.so.0
```

Notice how each library has its own unique path in the store. This isolation prevents conflicts between different versions of libraries and enables multiple versions to coexist peacefully.

## Closures: Complete Dependency Tracking

A "closure" in Nix represents the complete set of dependencies required by a package. Using cryptographic hashes allows Nix to identify the **exact** build and runtime dependencies of any package:

```bash
~ ❯ nix-store -qR $(which wget)
/nix/store/6kyaqlxcmfadiiq0mcdj1symv1jsp58w-xgcc-12.3.0-libgcc
/nix/store/aw137ya6rvy61zw8ydsz22xwarsr8ynf-libunistring-1.1
/nix/store/vh4pdds47783g12fmywazdx3v3kx0j4x-libidn2-2.3.4
/nix/store/ayg065nw0xi1zsyi8glfh5pn4sfqd8xg-glibc-2.37-8
/nix/store/ix7cb1isdcdl4gq9hl4pdk6gyc4wrk14-openssl-3.0.9
/nix/store/mgz7sp9jxnk7c3rn1hvich9n0k2rjr7m-zlib-1.2.13
/nix/store/pxl4n1lrns2xhc8f1s04srb4cphlg5cz-pcre-8.45
/nix/store/y5975fancsig22f6xw22mmmffn19n8zp-util-linux-minimal-2.39-lib
/nix/store/d3kkv9vjb3ljh7hr5v38gls8iykvwkny-wget-1.21.4
```

We can even visualize the dependency graph of a package:

```bash
~ ❯ nix-store -q --graph $(which wget)
```

This will generate a graph showing all relationships between packages:

![nix closure for wget package](wget-closure.png "nix closure")


The beauty of closures is that they can be distributed across hosts, which enables powerful distributed build and cache systems:

```bash
~ ❯ nix copy --to ssh-ng://remote-host /nix/store/qh4y4iw...
```

This is the foundation for Nix's [distributed builds](https://nixos.org/manual/nix/stable/advanced-topics/distributed-builds.html) and [binary cache](https://nixos.wiki/wiki/Binary_Cache) systems.

## Garbage Collection

Since the Nix store can accumulate packages over time, Nix provides a garbage collection mechanism to reclaim space by removing packages that aren't referenced by any active profiles:

```bash
~ ❯ nix-collect-garbage
finding garbage collector roots...
removing stale temporary roots file '/nix/var/nix/temproots/1023955'
deleting garbage...
deleting '/nix/store/mvqj8avzhkqabkg51cyz617qnhzzawhl-anstyle-wincon-1.0.1'
deleting '/nix/store/xzspb26l48b7hlhmlp6ac6sbivil0kgj-rust-operator-deps-0.1.0'
deleting '/nix/store/q512fyfmpmdw0ap391j8vkdd8j435545-rust-operator-deps-0.1.0'
deleting '/nix/store/gdnzfmns1ryh2pg5z9zbl0jgdspmmmx0-vendor-cargo-deps'
...
deleting unused links...
note: currently hard linking saves -0.00 MiB
1855 store paths deleted, 7729.65 MiB freed
```

## The Nix Expression Language

Nix has its own domain-specific language for defining packages and configurations. It has three key characteristics:

- **Pure functional** - functions always produce the same output given the same input
- **Domain specific** - designed specifically for describing packages and their dependencies
- **Lazy evaluation** - expressions are only evaluated when their results are needed

### Syntax Basics

Let's explore some basic syntax of the Nix language:

```nix
# operators
1 + 2
> 3
[ 1 2 ] ++ [ 3 ]
> [ 1 2 3 ]
```

```nix
# let ... in ..., allow repeated use of variables in scope
# string interpolation
# nix-repl>
let
    name = "World";
in
    "hello ${name}!"
> hello World!
```

```nix
# attribute set, attributes accessible by '.'
# with ..., expose attributes directly
let
    attrs = { a = "str"; b = false; i = 3; };
in
    with attrs; [ a attrs.b i ]
> [ "str" false 3 ]
```

We can merge attribute sets and use the ```inherit``` keyword to bring variables into scope:

```nix
# merging attr sets
# dynamic typing
let
    attrs1 = { a = "str"; b = false; };
    attrs2 = { b = 10; i = 3; };
in
    attrs1 // attrs2
> { a = "str"; b = 10; i = 3; }
```

```nix
# inherit, assign existing values in nested scope
let
    x = { b = 1; };
    y = 2;
    z = false;
in
    {
      inherit x y;
      z = z;
    }
> { x = { b = 1; }; y = 2; z = false; }
```

### Functions

In Nix, functions are nameless (lambdas) and always take exactly one argument:

```nix
# argument: function body
let
    f = x: x + 1;
in
    {
      type = builtins.typeOf f;
      result = f 1;
    }
> { result = 2; type = "lambda"; }
```

```nix
# nested functions, x: (y: x + y)
let
    f = x: y: x + y;
in
    f 1 2
> 3
```

Functions can also take attribute sets as arguments, with optional default values:

```nix
# attr set as argument, defined attr must be passed
# ?, default value
# ... , extra attrs
# @name, named attr set
let
    f = {a, b ? 1, ...}@args: a + b + args.c;
in
    f { a = 1; c = 1; }
> 3
```

### Domain-Specific Features

Nix has some unique behaviors due to its domain-specific nature. For example, division symbols in paths are interpreted literally:

```nix
6/3
> /Users/mskalski/org/6/3
```

```nix
let
    r = 6/3;
in
    builtins.typeOf r
> "path"
```

### Lazy Evaluation

Nix uses lazy evaluation, meaning expressions are only evaluated when their results are needed:

```nix
let
    f = builtins.fetchurl "http://127.0.0.1:8000/f";
    b = 3;
in
    b
> 3 # no request has been made to http server
```

In this example, even though we defined a function to fetch a URL, no network request was made because the result wasn't needed.

## Practical Nix: What Can I Do With It?

Now that we understand the basics, let's explore some practical applications of Nix.

### Task Shells: Ephemeral Environments

One of my favorite features is creating ephemeral shells with specific packages:

```
~ ❯ cowsay "nix is awesome!"
Unknown command: cowsay

~ ❯ nix-shell -p cowsay
[nix-shell:~]$ cowsay "nix is awesome!"
 _________________
< nix is awesome! >
 -----------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||

[nix-shell:~]$ exit

~ ❯ cowsay "nix is awesome!"
Unknown command: cowsay
```

This makes it incredibly easy to try out packages without permanently installing them. You can even create ad-hoc environments with specific Python modules:

```bash
~ ❯ python3 -c "import bcrypt; print(bcrypt.__version__)"
Traceback (most recent call last):
  File "<string>", line 1, in <module>
ModuleNotFoundError: No module named 'bcrypt'

~ ❯ nix-shell -p "python3.withPackages(p: [ p.bcrypt ])"
[nix-shell:~]$ python3 -c "import bcrypt; print(bcrypt.__version__)"
4.0.1
```

### Profiles: Persistent Environments with History

Profiles allow you to maintain persistent environments with full rollback capability and atomic updates:

```bash
# Install btop package in user environment (new generation)
~ ❯ nix profile install nixpkgs#btop

# Compare changes between generations
~ ❯ nix profile diff-closures
Version 6 -> 7:
  btop: ∅ → 1.2.13, +1637.7 KiB
  gcc: ∅ → 12.3.0, +15845.2 KiB
  glibc: ∅ → 2.37-8, +29536.3 KiB
  libidn2: ∅ → 2.3.4, +350.4 KiB
  libunistring: ∅ → 1.1, +1813.8 KiB
  xgcc: ∅ → 12.3.0, +139.1 KiB

# Revert to previous generation
~ ❯ nix profile rollback
switching profile from version 7 to 6
```

### Flakes: Reproducible Definitions

Flakes introduce ```flake.nix``` and ```flake.lock``` files to provide clear definitions of inputs and their versions. Think of them as similar to package.json and package-lock.json in the Node.js ecosystem, but more powerful.

Here is a short recording of me playing with nix flakes:

{{ asciinema(id="604780", speed="1.5") }}

#### Composing Projects

Flakes can produce a variety of outputs: binaries, container images, development environments, and more:

```bash
~ ❯ nix flake show github:michalskalski/axact
github:michalskalski/axact/9ca2f50dc4fb836af6e16dc03190cd2055d9f24b
├───devShell
│   ├───aarch64-darwin omitted (use '--all-systems' to show)
│   ├───aarch64-linux omitted (use '--all-systems' to show)
│   ├───x86_64-darwin omitted (use '--all-systems' to show)
│   └───x86_64-linux: development environment 'nix-shell'
└───packages
    ├───aarch64-darwin
    │   ├───bin omitted (use '--all-systems' to show)
    │   ├───default omitted (use '--all-systems' to show)
    │   └───ociImage omitted (use '--all-systems' to show)
    ├───aarch64-linux
    │   ├───bin omitted (use '--all-systems' to show)
    │   ├───default omitted (use '--all-systems' to show)
    │   └───ociImage omitted (use '--all-systems' to show)
    ├───x86_64-darwin
    │   ├───bin omitted (use '--all-systems' to show)
    │   ├───default omitted (use '--all-systems' to show)
    │   └───ociImage omitted (use '--all-systems' to show)
    └───x86_64-linux
        ├───bin: package 'axact-0.1.0'
        ├───default: package 'axact-0.1.0'
        └───ociImage: package 'docker-image-axact.tar.gz'
```

Building a binary from a flake is straightforward:

```bash
~ ❯ nix build github:michalskalski/axact#packages.x86_64-linux.bin

# by default it produce 'result' symlink in current directory
~ ❯ ls -l
result -> /nix/store/qh4y4iwh0q40q5xxlp61bimhx8i6dp9i-axact-0.1.0

~ ❯ nix path-info --json $(realpath result) | jq .
[
  {
    "deriver": "/nix/store/gkxa58jxq5a9z7187afx0lywkckxr05b-axact-0.1.0.drv",
    "narHash": "sha256-cLMwsb0BQCRGXB1M+KGruZB+lR0gZRV3UKOFalkgONE=",
    "narSize": 6295640,
    "path": "/nix/store/qh4y4iwh0q40q5xxlp61bimhx8i6dp9i-axact-0.1.0",
    "references": [
      "/nix/store/c50v7bf341jsza0n07784yvzp5fzjpn5-gcc-12.3.0-lib",
      "/nix/store/vq3sdi8l15rzfl5zvmwpafrzis4sm6xf-glibc-2.37-8"
    ],
    "registrationTime": 1692975733,
    "ultimate": true,
    "valid": true
  }
]
```

You can also build container images:

```bash
~ ❯ nix build github:michalskalski/axact#packages.x86_64-linux.ociImage

~ ❯ ls -l result
result -> /nix/store/sn1yivqy1c1qjhypq3n515g4r47rgp0k-docker-image-axact.tar.gz

# load image to local docker instance
~ ❯ docker load < result
941c04e2c681: Loading layer [=======================================>]  47.59MB/47.59MB
Loaded image: axact:latest

# but docker is not needed
~ ❯ skopeo copy "docker-archive://$(realpath result)" "docker://registry-address/img:tag"
```

#### Development Shells

Flakes can define development environments where all dependencies for your application are available, providing a consistent experience for all developers:

```bash
~ ❯ nix develop github:michalskalski/axact
```

These development shells are typically defined in the flake.nix file:

```bash
# flake.nix
...
devShell = mkShell {
    inputsFrom = [ bin ];
    buildInputs = [dive skopeo] ++ darwinPkgs;
};
```

For an even more seamless experience, you can use [direnv](https://direnv.net/), which many code editors also understand:

```bash
~ ❯ ls -a project/
.envrc  flake.nix ..

~ ❯ cat project/.envrc
use flake

~ ❯ cd project/
direnv: error project/.envrc is blocked.
Run `direnv allow` to approve its content

~ ❯ direnv allow
direnv: loading project/.envrc
direnv: using flake
direnv: nix-direnv: using cached dev shell
```

#### Overlays

If your project depends on a specific version of a system library or needs extra patches, you can easily modify packages at your project level using [overlays](https://nixos.wiki/wiki/Overlays):

```bash
example_overlay = final: prev: {
  package = prev.package.overrideAttrs (old: {
    version = "";
    src = prev.fetchurl {
      url = "";
      hash = "";
    };
  });
}
```

## NixOS: One System to Rule Them All

While Nix itself is a powerful package manager, **NixOS** takes the concept further by applying the same principles to the entire operating system configuration.

### What Makes NixOS Special?

NixOS is fully integrated with Nix, both from packages and configuration standpoints. By default, it uses "channels" as a source of package versions:

- **Stable channels** are released every six months (for example, 22.11, 23.05)
- **Unstable** channel is a rolling release with the latest packages

### Declarative System Configuration

The entire system can be described through [declarative configuration](https://nixos.org/manual/nixos/stable/options.html). This means you can:

- Store your configuration in a repository
- Make it a flake and describe multiple hosts as separate outputs
- Share common configurations between hosts, making your setup more modular

### Safe System Changes

Before applying changes to your system, you can test them in a [virtual machine](https://nixos.wiki/wiki/NixOS:nixos-rebuild_build-vm):

```bash
# will start local vm with current system configuration
~ ❯ nixos-rebuild build-vm
```

Every package installation in the system profile creates a new "generation" and an entry for it in the bootloader. If your system doesn't start after an upgrade, you can simply boot from a previous generation.

![grub generation entries](nixos-boot.png)

If you need to reinstall your system, you can run from a live CD and restore your system from an existing configuration with a single command, or generate a disk image ahead of time.

### Alternatives for Other Platforms

If you're not ready to switch to NixOS, there are options for other operating systems:

- **[nix-darwin](https://github.com/LnL7/nix-darwin)** tries to replicate NixOS behavior on macOS
- **[Home Manager](https://github.com/nix-community/home-manager)** can be used standalone or as a module for NixOS or nix-darwin and provides a rich library of software configurations

## Additional Resources

If you're interested in learning more about Nix, here are some valuable resources:

- [Nix learning resources](https://nixos.org/learn)
- [Zero to Nix](https://zero-to-nix.com/)
- [Nix Pills](https://nixos.org/guides/nix-pills/)

For those interested in build systems more generally:
- [Build Systems à la Carte (2018)](https://www.microsoft.com/en-us/research/uploads/prod/2018/03/build-systems.pdf)
- YouTube: [Build Systems à la Carte](https://www.youtube.com/watch?v=BQVT6wiwCxM)

## Conclusion

Nix represents a paradigm shift in how we think about package management and system configuration. By treating the filesystem as memory and applying functional programming principles to deployment, it solves many long-standing issues in software management.

While the learning curve can be steep, the benefits - reproducibility, atomic upgrades, rollbacks, and isolation - make it worth considering for both development environments and production systems.
