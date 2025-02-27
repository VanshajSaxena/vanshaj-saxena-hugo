---
title: 'Nix and NixOS'
#date: 2024-02-10T03:36:02+05:30
draft: false
showtoc: false
weight: 98
showtoc: true
---
---

[NixOS](https://nixos.org/) is a GNU/Linux distribution that focuses on
reproducibility, declarative configuration, and robust package management.

If you have ever used GNU/Linux Operating System you must know a large part of
maintaining an installation of a Linux Distribution is having to edit some
configuration files.

The GNU/Linux Operating System is so malleable that you can break it into
pieces of independent software and put them back together in a different way,
and it would work if done correctly. But this power comes with the
responsibility of having to manage the complexity of the underlying system.

**NixOS** gives the user the power to handle this complexity in a declarative
way. NixOS does this with the use of its **purely functional package manager**,
**Nix**.

---
### Nix
Nix is a powerful package manager for Linux and other Unix systems that makes
package management reliable and reproducible.

So if you have ever used a package manager like npm, it maintains a lock file
that tracks the exact version of the packages you are using in your project.
This is to ensure that the project will work the same way on different
machines.

In a similar fashion [Nix Flakes](https://nixos.wiki/wiki/flakes) provide a
standard way to write Nix expressions (and therefore packages) whose
dependencies are version-pinned in a lock file, improving reproducibility of
Nix installations.

Below is an example of a really simple Nix Flake:
``` nix
#  flake.nix
{
  description = "A Simple Nix Flake"; # Human readable string that describes the flake
  outputs = { self }: {
    packages.x86_64-linux.hello = with import <nixpkgs> {}; hello;
  };
}
```
This entire file is a Nix expression that evaluates to an attribute set.
Looking closely you'll find the expression in itself is an attribute set.

Here `outputs` is a function that takes an attribute set as an argument, we are
only using one parameter here, `self`, which is a reference to the current flake.

`packages.x86_64-linux.hello` is just a reference to the package `hello`
through the `x86_64-linux` key contained in the `packages` attribute set.

---
### NixOS

I faced this problem of traditionally managing the state (packages) of my Linux
System imperatively. This always caused conflicts and inconsistencies in my
environment in various ways. Also I never knew what state of my system is
leading to an issue. It is very hard to pin point it, when you build you system
imperatively rather than declaratively.

NixOS allows you to **interact with your operating system programmatically**, more
specifically you can declare your packages and configurations and **build your
system like any other software project that you build**.

``` nix
# /etc/nixos/flake.nix
{
  description = "NixOS System Flake";
  # Input to the flake
  inputs = {
    nixpkgs-stable.url = "github:NixOS/nixpkgs/nixos-24.11";
    nixpkgs-unstable.url = "github:NixOS/nixpkgs/nixos-unstable";
  };
  # Flake output
  outputs =
    {
    nixpkgs-stable,
    nixpkgs-unstable,
    ...
    }:
    {
      # `NIXOS` is hostname here, can be anything
      nixosConfigurations.NIXOS = nixpkgs-unstable.lib.nixosSystem rec {
        # Architecture
        system = "x86_64-linux";
        # Used to provide special arguments to the flake output
        specialArgs = {
          nixos-stable = import nixpkgs-stable {
            inherit system;
            config.allowUnfree = true;
          };
        };
        # Nix module to be included in the flake
        modules = [
          ./configuration.nix
        ];
      };
    };
}
```
As you see, this is how we define a flake that helps to define our dependencies
declaratively, in any project. For this system flake, the dependencies are the
packages that we require in our environment.

``` nix
# /etc/nixos/home.nix
{ inputs-from-flake, ... }:
{
  home.username = "vanshaj";
  home.homeDirectory = "/home/vanshaj";
  home.packages = with inputs-from-flake; [
    # Define your dependencies
  ];
  # basic configuration of git
  programs.git = {
    enable = true;
    userName = "Vanshaj Saxena";
    userEmail = "vs110405@outlook.com";
  };
  home.stateVersion = "24.05";
  # Let home Manager install and manage itself.
  programs.home-manager.enable = true;
}
```
You can further extend your flake with the module system provided with Nix.

``` shell
vanshaj@NIXOS /etc/nixos $ lt
drwxr-xr-x     - root 26 Jul  2024  .
lrwxrwxrwx     - root 26 Jul  2024  ├── configuration.nix -> /home/vanshaj/nixos/configuration.nix
lrwxrwxrwx     - root 26 Jul  2024  ├── flake.lock -> /home/vanshaj/nixos/flake.lock
lrwxrwxrwx     - root 26 Jul  2024  ├── flake.nix -> /home/vanshaj/nixos/flake.nix
lrwxrwxrwx     - root 26 Jul  2024  ├── hardware-configuration.nix -> /home/vanshaj/nixos/hardware-configuration.nix
lrwxrwxrwx     - root 26 Jul  2024  └── home.nix -> /home/vanshaj/nixos/home.nix
```

This makes debugging and maintaining your project much easier.
Now, whenever there is a problem in the system you could literally `git bisect`
your system configuration like a traditional software.

Not only Nix allows to you build declaratively, it guarantees **reproducibility
and easy rollbacks**.

---

{{<img src="Systemdboot-Screenshot.jpg">}}


---

This **immutability** and **reproducibility** is the cause I use Nix for my projects
and my system.


[Source code](https://github.com/VanshajSaxena/nixos-config)

