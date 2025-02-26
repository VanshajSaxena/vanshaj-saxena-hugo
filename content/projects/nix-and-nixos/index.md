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
{
  description = "A Simple Nix Flake";
  outputs = { self }: {
    packages.x86_64-linux.hello = with import <nixpkgs> {}; hello;
  };
}
```

Nix Flakes are still and experimental feature in Nix Package Manager, but they
are slowly moving towards stability.

---
### NixOS

I faced this problem of traditionally managing the state (packages) of my Linux
System imperatively. This caused conflicts and inconsistencies in my
environment in various ways. Also I never knew what state of my system is
leading to an issue. It is very hard to pin point it, when you build you system
imperatively rather than declaratively.

``` shell
vanshaj@NIXOS /etc/nixos $ lt
drwxr-xr-x     - root 26 Jul  2024  .
lrwxrwxrwx     - root 26 Jul  2024  ├── configuration.nix -> /home/vanshaj/nixos/configuration.nix
lrwxrwxrwx     - root 26 Jul  2024  ├── flake.lock -> /home/vanshaj/nixos/flake.lock
lrwxrwxrwx     - root 26 Jul  2024  ├── flake.nix -> /home/vanshaj/nixos/flake.nix
lrwxrwxrwx     - root 26 Jul  2024  ├── hardware-configuration.nix -> /home/vanshaj/nixos/hardware-configuration.nix
lrwxrwxrwx     - root 26 Jul  2024  └── home.nix -> /home/vanshaj/nixos/home.nix
```

Now, whenever there is a problem in the system you could literally `git bisect`
your system configuration like a traditional software.

Here is the [source code](https://github.com/VanshajSaxena/nixos-config) for anyone interested in nerding out.
