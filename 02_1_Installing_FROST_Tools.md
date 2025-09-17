# 2.1: Installing the FROST Tools

The [ZF FROST Tools](https://github.com/ZcashFoundation/frost-tools)
allow you to create FROST secret shares and to sign with FROST. This
chapter provides hands-on tutorials for using them. By following these
tutorials, you can solidfy your understanding of how FROST works.

As demonstrated here, these tools can allow signing of documents, but
you won't be able to Bitcoin transactions yet. That will require
additional support, as detailed in [Chapter
3](03_0_FROST_and_Bitcoin.md).

## Installing Rust

The ZF Frost Tools were built with Rust using Cargo. If you've already
installed Rust and Cargo on your system, you can continue on to
["Installing ZF FROST Tools"](). Otherwise, you'll need to install
Rust and Cargo first.

> :book: ***What is a Rust?*** Rust is a programming language focused
on performance and memory safety. Its speed and efficiency and tight
control over memory management have made it a favorite among developers.

> :book: ***What is Cargo?*** Cargo is Rust's package management
system. It makes it simple to accurately install Rust projects
... such as the FROST Tools. (Cargo is another reason for Rust's
popularity.)

### Installing with Rustup

The preferred method of installing Rust and Cargo is
[`rustup`](https://www.rust-lang.org/tools/install).

#### Installing with Rustup on Linux & MacOS

On MacOS, Linux, and other UNIX OSes, you can run the following:
```
% curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

You should choose to "Proceed with standard installation". Afterward,
you will need to restart your command-line shell before `cargo` will
be visible.

#### Installing with Rustup on Windows

A [`rustup-init.exe`](https://rustup.rs/) installer is available if
you run WIndows.

### Installing with Package Managers

You can also install Rust and Cargo with a standard package
manager. You might prefer this if you use a single package manager to
control most of your third-party installations. Rust is supported on
Chocolatey, Homebrew, MacPorts, pkgsrc, and Scoop.

### Testing Your Install

When your install is complete, you should see `cargo` in your standard
path.

This is an example of a `rust-up` install:

```
% which cargo
/Users/shannona/.cargo/bin/cargo
```

This is an example of a Homebrew install:

```
% which cargo
/opt/homebrew/bin/cargo
```

## Installing ZF FROST Tools

The ZF FROST Tools should be installed with `git`. If you do not have
`git` installed, visit the [Git Downloads
site](https://git-scm.com/downloads).

Once you have `git`, choose a directory where you want to install
`frost-tools` and then clone the repository:

```
% git clone https://github.com/ZcashFoundation/frost-tools.git
Cloning into 'frost-tools'...
remote: Enumerating objects: 3295, done.
remote: Counting objects: 100% (54/54), done.
remote: Compressing objects: 100% (51/51), done.
remote: Total 3295 (delta 25), reused 3 (delta 3), pack-reused 3241 (from 2)
Receiving objects: 100% (3295/3295), 1.08 MiB | 1.37 MiB/s, done.
Resolving deltas: 100% (1987/1987), done.
```
This should create a `frost-tools` directory:
```
% ls frost-tools 
Cargo.lock		frost-client		README.md
Cargo.toml		frostd			rust-toolchain.toml
codecov.yml		LICENSE-APACHE		supply-chain
CONTRIBUTING.md		LICENSE-MIT		tests
DEVELOPER.md		Makefile.toml		zcash-sign
```
You can then change into the crate root and run `cargo build`:
```
% cd frost-tools
% cargo install --path frost-client  
``
You should see the client-side ZF FROST Tools installed:
```
Installed package `frost-client v0.1.0 (/Users/ShannonA/Documents/GitHub/FROST/frost-tools/frost-client)` (executables `coordinator`, `dkg`, `frost-client`, `participant`, `trusted-dealer`)
````
You're now ready to use the ZF FROST Tools!

## Summary: Installing the FROST Tools

All you need to do to start using the ZF FROST Tools is to `clone` the
`frost-tools` repo and build it with Rust. If you've never used Git or
Rust before, you might need to download that software first.

## What's Next

Continue onward with "Signing with FROST" by [§2.2: Creating FROST Secret Shares](02_2_Creating_FROST_Secret_Shares.md).
