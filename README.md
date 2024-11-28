# Pacman on LFS Version 12.2-systemd

  Forked from "[Pacman on LFS Version 12.0-systemd](https://github.com/wsdmatty/lfs-pacman)" By Matthew Sexton wich in turn is  
  Forked from "[Pacman on LFS 8.1](https://github.com/benvd/lfs-pacman)" By Ben Van Daele.  
  Added changes and PKGBUILD files from "[LFS-11.2-systemd-with-EFI-and-pacman](https://github.com/ShiroiBara/LFS-11.2-systemd-with-EFI-and-pacman)"  
  Based on [the guide writen by James Kimball](http://lists.linuxfromscratch.org/pipermail/hints/2013-March/003304.html) in 2013.
  Building on [Linux From Scratch](https://linuxfromscratch.org/lfs/view/stable-systemd/index.html)

"Archlinux from Scratch" but without the Arch repo's unless you add them.

I hope you know what Linux from Scratch is, or (GNU, lol) Linux at all. if so, this tries to add arch linux their package manager to Linux from Scratch. by building everything ourselves, and then installing it. But not using Arch or its friends repositories of precompiled software. Because that would defeat the purpose of LFS and slowly replacing LFS with basicly what Arch makes Arch.

So we basicly build LFS, but make every package, a installable arch package. You could then in theory, make your own package repo and host these (beyound the scope of this repo). To start, get familiar with the LFS book, prefferably youve completed the book atleast once and got it working

Licensed under the [Creative Commons Attribution-ShareAlike 3.0 License](https://creativecommons.org/licenses/by-sa/3.0/).

## Introduction

This guide is divided in five stages, the first one of which starts just before installing tools to your final system.

- Stage 1 - Installing pacman to your temporary toolchain
- Stage 2 - Installing pacman with pacman
- Stage 3 - Installing packages of chapter eight of the LFS book
- Stage 4 - Installing pacman to your final system
- Stage 5 - Finishing the book

## Stage 1 - Installing pacman to your temporary toolchain

This stage begins right before **7.13. Cleaning up and Saving the Temporary System**,
after completing **7.12. Util-linux-2.39.3**.
So yes, youve completed the previous chapters of the LFS book. If not, start reading :D

When you finished 7.12, its best to grab these extra wget download list and md5 checksum check. 
adapt to what feels right. Also, make sure $LFS points to your LFS target mount point when doing these steps. And you could make a backup either or both before and or after adding pacman, now be the time to decide.

Probably good to do outside chroot, grab the pacman install scripts:

```sh
 git clone https://github.com/CodeAsm/lfs-pacman $LFS/sources/pacman
```
the work in progress branch:
```sh
git clone --branch LFS-12.2 --single-branch https://github.com/CodeAsm/lfs-pacman $LFS/sources/pacman
```

### Pacman and makepkg dependencies

Pacman and to use makepkg later, they depend on the following packages:

- Python 3.12.5
    - zlib 1.3.1
    - meson 1.5.1
- ninja 1.12.1
- util-linux for getopt 2.40.2
- shadow for su 4.16.0
- fakeroot 1.36 
    - libcap 2.70
- libarchive 3.7.4
    - OpenSSL 3.3.1 (libcryto)
- pkgconf 2.3.0 (and not pkg-config, as dev has stalled)
- pacman 7.0.0

We will also need:

- Vim 9.1.0660 

Some of these are not part of the LFS book, so download their sources manually:

From the BLFS book:
- libarchive: <https://github.com/libarchive/libarchive/releases/download/v3.7.4/libarchive-3.7.4.tar.xz>

Outside of the LFS ecosystem:
- fakeroot: <https://deb.debian.org/debian/pool/main/f/fakeroot/fakeroot_1.36.orig.tar.gz>
- pacman: <https://gitlab.archlinux.org/pacman/pacman/-/releases/v7.0.0/downloads/pacman-7.0.0.tar.xz>

To download all of the packages (including libarchive from BLFS) by using wget-list-pacman as an input to the wget command, use: 
```sh
wget --input-file=$LFS/sources/pacman/install-files/wget-list-pacman --continue --directory-prefix=$LFS/sources
```
Additionally, there is a separate file, md5sums, which can be used to verify that all the correct packages are available before proceeding. Place that file in $LFS/sources and run: 

```sh
pushd $LFS/sources
  md5sum -c $LFS/sources/pacman/install-files/md5sums-pacman
popd
```
This check can be used after retrieving the needed files with any of the methods listed above.

Build these packages using the following commands. Just like the LFS book, these commands assume you've extracted the relevant sources and `cd`'d into the resulting directory.

You need to be inside the chroot environment, if you quit (to backup/restore) and have the virtual kernel file system mounted. (see [Section 7.4, “Entering the Chroot Environment”](https://linuxfromscratch.org/lfs/view/stable-systemd/chapter07/chroot.html)) before continuing. 

#### zlib 1.2.11

```
./configure --prefix=/usr
make
make install
```

#### libarchive 3.7.2

```
./configure --prefix=/usr --without-xml2 --disable-shared
make
make install
```

#### Pkgconf 2.1.1

```
./configure --prefix=/usr              \
            --disable-static           \
            --docdir=/usr/share/doc/pkgconf-2.1.1
make
make install
```

#### libcap 2.69

Prevent static libraries from being installed:

```
sed -i '/install -m.*STA/d' libcap/Makefile
```

Compile and install the package:

```
make prefix=/usr lib=lib
make test
make prefix=/usr lib=lib install
```

#### libtool 2.4.7

 Configure and Build Libtool:

```
./configure --prefix=/usr
make
make install
```
Remove a useless static library:
```
rm -fv /usr/lib/libltdl.a
```

#### autoconf 2.72 

Now do Autoconf:
```
./configure --prefix=/usr
make
make install
```

#### automake 1.16.5 

 Prepare Automake for compilation:
```
./configure --prefix=/usr --docdir=/usr/share/doc/automake-1.16.5
```
Compile and install the package:
```
make
make install
```
#### fakeroot 1.36
As part of its installation, fakeroot calls ldconfig, which is located in /tools/sbin. /tools/sbin is not part of our PATH, so we must add it now.
```
# Don't install docs
sed -i 's/SUBDIRS=doc \(.*\)/SUBDIRS=\1/' Makefile.am

./bootstrap

./configure --prefix=/usr \
  --libdir=/usr/lib/libfakeroot \
  --disable-static \
  --with-ipc=sysv

make
make install

install -dm0755 "/etc/ld.so.conf.d/"
echo '/usr/lib/libfakeroot' > "/etc/ld.so.conf.d/fakeroot.conf"
```
### OpenSSL 3.3.1 

```
./config --prefix=/usr         \
         --openssldir=/etc/ssl \
         --libdir=lib          \
         shared                \
         zlib-dynamic
make
sed -i '/INSTALL_LIBS/s/libcrypto.a libssl.a//' Makefile
make MANSUFFIX=ssl install
mv -v /usr/share/doc/openssl /usr/share/doc/openssl-3.3.1
```

### Pacman 6.0.2

```
PKG_CONFIG=pkgconf ./configure --prefix=/usr   \
            --disable-doc     \
            --disable-shared  \
            --sysconfdir=/etc \
            --localstatedir=/var
make
make install
```

This will have installed, amongst others, the `makepkg.conf` and `pacman.conf` config files in `/etc`; you may want to edit them. For `makepkg.conf`, be sure that `CARCH` and `CHOST` are appropriate, e.g.:

```
CARCH="x86_64"
CHOST="x86_64-pc-linux-gnu"
```

You can set your name and email address as the `PACKAGER` if you want.

### Vim (or any text editor)

We wont bother testing VIM as this is just to our temporary tools and it'll be recompiled later.

```
echo '#define SYS_VIMRC_FILE "/etc/vimrc"' >> src/feature.h
./configure --prefix=/usr
make
make install

```


### Setting up to build packages with pacman

Create a new user by adding the following to `/etc/passwd`:

```
lfs:x:1000:999:LFS:/home/lfs:/bin/bash
```

Where 999 is the ID of the `users` group. Check `/etc/group` for the correct ID, or if `users` is not present in that file, add the following to it.

```
users:x:999:
```

Create a home directory:

```
mkdir -v /home/lfs
chown -Rv lfs:users /home/lfs
```

Exit your current chroot, then chroot into your new user (make sure `1000`, `999` and `lfs` are set to the proper values for your system):

```
chroot --userspec=1000:999 "$LFS" /bin/env -i \
     HOME=/home/lfs     \
     TERM="$TERM"       \
     PS1='(lfs chroot) \u:\w\$ ' \
     PATH=/bin:/usr/bin:/sbin:/usr/sbin \
     /bin/bash --login +h
```

You may want to create a `builds` directory in your home dir. In there, you would then create a directory for each package you're building.

From the chroot environment, create a builds directory:

```
cd /home/lfs
mkdir -v builds
```

## Stage 2 - Installing pacman with pacman

Copy the pacman sources to its build directory, `~/builds/pacman-5.0.2`.

Download the necessary build files (`PKGBUILD`, `makepkg.conf` and `pacman.conf.x86_64`) from the [Install-Files](https://github.com/mssxtn/lfs-pacman/tree/master/install-files/pacman-5.0.2) to the build directory.

The included PKGBUILD has had the following edits made:

* Removed 'groups' and 'depends' sections (We're not using groups, and as far as pacman knows none of the dependencies are installed.)
* Edited the 'source' section to point to local tarball and files
* Removed 'check' section as most of the tests will fail with our current environment
* Added the PKG_CONFIG=pkgconf flag to find the newer pkg config tool

Now run the following command as your non-root user from the build directory:

```
makepkg
```

If all goes well, this should have created a file that you can now install as follows, as root this time (thus, logout, use the chroot command like you would in 7.4. Entering the Chroot Environment):

```
pacman -U pacman-5.0.2-2-x86_64.pkg.tar.gz
```

Before starting with stage 3, you left the LFS book at **7.13. Cleaning up and Saving the Temporary System**
Continue from there
 - backup your tools

Come back after you have read **8.2. Package Management**

## Stage 3 - Installing packages of chapter eight of the LFS book

Now we will continue with the rest of the book, but instead of building and installing the packages manually, we will create packages and then use pacman to install them.

All of the necessary `PKGBUILD` files can be found in the `packages` directory, but if you want to learn how to create these, I suggest you only refer to them when you get stuck.

### Creating a package

The general process goes like this:

1. Create a new directory in the builds directory.
2. Copy all needed files to it (source archive, possibly other files like patches or config files).
3. Write a `PKGBUILD` file.
4. Run `makepkg --skipchecksums`.
5. Check the contents of the `pkg` directory; this is what pacman will install, so you may want to verify that it looks good.
6. Install the package (as root) with `pacman -U $filename`

Writing a `PKGBUILD` file can take some trial and error. Essentially you'll need to copy what the LFS book wants you to do and paste it in the `PKGBUILD` at the right place, but usually you will not be able to copy things verbatim from the book.

If you're stuck, check the [official Arch Linux packages](https://www.archlinux.org/packages/) or the`PKGBUILD` files in the `packages` directory of this repo.

### Tips

#### Source directory
When calling any of the `PKGBUILD` functions, `makepkg` will automatically `cd` you into the source dir, which is where it will have extracted or symlinked all your source files.

#### Source tarball
`makepkg` will automatically extract all archives you specify in the `sources` array. Typically, your source tarball will have been extracted into a separate directory (although that depends on the tarball itself; the `tzdata` tarball for instance, extracts directly, without a subdirectory).

This is why the first step of each function will usually be to `cd` into the directory created during the extraction of the source tarball.

#### Installing files
When installing files, you cannot install them to the system root, but instead you have to install them in `${pkgdir}`. For certain packages this is done by using `make DESTDIR=${pkgdir} install` instead of `make install`. Not all packages use `DESTDIR`, however. If you're not sure how those packages should be built, you can always check the [Arch Linux packages](https://www.archlinux.org/packages/) or the `PKGBUILD` files from this repo.

In some cases, the book will tell you to `mv` a directory. This will work while creating the package, but will cause problems when installing it. You should first create the directory with `install -vdm755 $dir_name`.

#### Post-install
Things that the LFS book wants you to do after having called `make install` will likely need to be placed in an `.install` file. Check the [Arch Linux wiki](https://wiki.archlinux.org/index.php/PKGBUILD#install) for more info.

Pacman will execute the functions in the `.install` file at the end of the installation process.

### Notes for individual packages

Some packages require additional steps besides simply converting the LFS instructions into a  `PKGBUILD`. These steps are documented here.

#### 6.9. Glibc-2.39

The book wants you to create some symlinks; I did that manually, as I felt it would be out of place in the package.

The tzdata part of glibc requires `zic` to be installed to the system, which means after glibc was installed with `pacman -U`. While I suppose you could directly call the `zic` binary residing in the `src` or `pkg` directories, I thought it would be cleaner to split the tzdata installation to a separate package, much [like Arch Linux does](https://www.archlinux.org/packages/core/any/tzdata/).

#### 6.10. Adjusting the Toolchain

I did these steps manually. It didn't feel suitable to (ab)use `makepkg`/`pacman` for this purpose.

#### 6.15. Bc-6.7.5

The book wants you to create symlinks for libncurses. I did this manually before building the package.

#### 6.20. GCC-13.2.0

Before building the package, increase the stack size: `ulimit -s 32768`.

I had to use `--force` when installing this package, since some libraries already existed on the system.

#### 6.28. Shadow-4.14.5

I ran `passwd` manually.

#### 6.4. Bash-5.2.21

When creating the package, `makepkg` told me that "Package contains reference to $srcdir". Using `grep -R "$(pwd)/src" pkg/`, I found out that Bash installs `Makefile.inc` to `/usr/lib/bash/`, which contains a reference to the build directory. On an existing Arch Linux installation, `/usr/lib/bash/Makefile.inc` also contained a reference to a (non-existing) build directory, so I assume this is benign.

Use `--force` to install this package, since `/bin/bash` was created as part of **7.6. Creating Essential Files and Symlinks**.

I then re-chrooted, using `/bin/bash` instead of `/tools/bin/bash`.

#### 7.9. Perl-5.38.2

The LFS book tells you to create `/etc/hosts`. I've chosen to do this manually, rather than to have the Perl package install this file. It doesn't sound right that Perl should own this file. For reference, in Arch Linux, the hosts file is owned by the `filesystem` package, which contains the base Arch Linux files, so it makes sense that this is created manually in the case of LFS.

Use `--force` to install this package, since `/usr/bin/perl` was created as part of **6.6. Creating Essential Files and Symlinks**.

#### 6.5. Coreutils-9.4

For some reason, doing the in place `sed` (`sed -i`) on `chroot.8` resulted in the file having 000 permissions. I replaced it with a regular `sed`, redirecting the result to a new file, then using `install` to copy the file to its destination.

Use `--force` to install this package, since a number of files already exist (amongst others `cat`, `dd`, `echo`) in `/bin` (as symlinks to `/tools`). These were added in **6.6. Creating Essential Files and Symlinks**.

#### 6.8. Findutils-4.9.0

As with coreutils, using in place `sed` set the permissions to 000, so I used the same workaround here.

#### 6.63. Sysklogd-1.5.1 << ??? >>

Sysklogd's makefile doesn't support specifying a destination directory when installing (`make DESTDIR=/path`). I've included a patch that adds this.

## Stage 4 - Installing pacman to your final system

Stage four starts immediately after the last package in chapter six, currently vim. Before continuing with the rest of the chapter, build and install pacman and its dependencies.

As part of chapter six you should already have installed most of these packages, except for:

- libarchive
- fakeroot
- pacman

Use your temporary pacman installation to install them. Remember, we're installing to the final system, not to `/tools`.

## Stage 5 - Finishing the book

Finish up the rest of the book manually.

When restarting my machine, the LFS system wouldn't boot properly. It turned out that many binaries and other files were owned by `ben`, the user with which I built the packages. I have no idea why that happened, but chowning them to `root:root` allowed my system to boot.
