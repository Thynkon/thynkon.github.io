---
layout: post
title: "How does the AUR work?"
date:   2021-01-21 21:04:33 +0100
tags: [archlinux, package-management, aur]
---

## What is the AUR?
If you are new to ArchLinux or Linux in general, you may be wondering what AUR is
and how it differs from \"ordinary\" distros repositories.

So, what is [AUR](https://wiki.archlinux.org/index.php/Arch_User_Repository)?
It is ```a community``` repository for users using ArchLinux-based distributions.
Anyone who wants to create or maintain an existing package is free to do so.

Unlike distros repositories, AUR does ```not provide binary packages``` (e.g. .deb or .rpm).
Instead, it contains a file called PKGBUILD that ```tells you how to build a package```.
The installation of a package can be done manually
(using [makepkg](https://wiki.archlinux.org/index.php/Makepkg)) or using
a [helper](https://wiki.archlinux.org/index.php/AUR_helpers).

Think of ```the installation process``` as a ```cake-making procedure``` that
you find in a book. In this case, the ```book``` is the ```AUR```, the ```recipe``` is the ```PKGBUILD```
and the ```baker``` is the ```AUR helper```.

The following image should give you an idea of how an ```AUR helper``` works.

![Build process of a .zst package](/assets/img/pkgbuild.svg){: .normal }

## What does the build process look like?

```sh
#!/bin/bash

pkgname=gitui-git
_filename=gitui
pkgver=0.13.0
pkgrel=1
pkgdesc='Blazing fast terminal-ui for git written in Rust'
arch=('i686' 'x86_64')
url="https://github.com/extrawurst/gitui"
license=('MIT')
depends=('libgit2' 'libxcb')
conflicts=('gitui')
makedepends=('git' 'cargo' 'python') # xcb crate needs python
source=("${pkgname}::git+${url}")
sha256sums=('SKIP')

build() {
  cd "${pkgname}"
  LIBGIT2_SYS_USE_PKG_CONFIG=1 cargo build --release --locked
}

package() {
  install -Dm755 "$srcdir/$pkgname/target/release/$_filename" "$pkgdir/usr/bin/$_filename"
  install -Dm644 "$srcdir/$pkgname/LICENSE.md" "$pkgdir/usr/share/licenses/$_filename/LICENSE"
}
```

When you install a package using an ```AUR helper```, it will download a PKGBUILD file which should look like this.
Once the file is downloaded, it will then run ```makepkg --syncdeps --install --clean```.

The ```makepkg``` command is provided by the ```pacman``` package and it will parse/load the ```PKGBUILD``` file and create the ```.zst``` package.
It will first download the source code. Then it will check if the dependencies listed in the depends and ```makedepends``` arrays are installed
on the system and if the packages that may cause conflicts are not installed. Finally, it executes each function that corresponds to a build step.

During the build process, two directories are created: ```src``` and ```pkg```. They contain, respectively, the source code and the files that the final package will contain.

As you may have already guessed, the build function tells makepkg how to build the package and the package function specifies the files that the ```.zst``` package should contain.

## Using chroots
A ```chroot``` is an operation that changes the apparent root directory for the current running process and their children.
A program that is run in such a modified environment cannot access files and commands outside that environmental directory tree.
This modified environment is called a ```chroot jail```.[^1]

Building a package in a ```clean chroot``` has many advanges to the maintainer. It makes sure that the dendencies listed in the ```PKGBUILD``` file are correct and that the
build instructions will work for any Arch-based distribution since a ```chroot``` ```emulates a minimal Archlinux iso``` (like the iso you download when you first install Archlinux).

To be able to build a package using a chroot, you will need to first create one. The command we will be using is provided by the package ```devtools```.
```sh
mkdir ~/chroot
mkarchroot ~/chroot/root base-devel
```

The last command, installs the ```base-devel``` package under the directory ```/root``` of the chroot we have just created.

Finally, to build a package in a clean chroot, all you need to do is to execute the following command in the directory that contains the ```PKGBUILD``` file:
```sh
makechrootpkg -u -c -r ~/chroot -- --check
```

It will clean the chroot, update its packages and build the ```.zst``` package using the chroot's settings.

You can then install the pacakge by running ```pacman -U <package_name>.zst```.

```sh
makechrootpkg -u -c -r ~/chroot -- --check
```
## References
- [chroot Archlinux wiki page](https://wiki.archlinux.org/index.php/Chroot)
- [makepkg Archlinux wiki page](https://wiki.archlinux.org/index.php/Makepkg)
- [Building in a clean chroot](https://wiki.archlinux.org/index.php/DeveloperWiki:Building_in_a_clean_chroot)

[^1]: This is an extract of the [Archlinux chroot's wiki page](https://wiki.archlinux.org/index.php/Chroot).

