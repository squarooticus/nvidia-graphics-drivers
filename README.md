# Building updated NVIDIA drivers on Debian bookworm

If you game under Linux, you probably find yourself wanting something closer to the game-ready drivers available on Windows. If so, this repo may be for you.

While we are unlikely to see anything so up-to-date on Linux anytime soon, I wanted to at least remain closer to the latest upstream Linux driver package than anything Debian offers, and I wanted to do it without polluting my runtime environment with a bunch of unstable or experimental packages that are likely to result in major brokenness for an install I also use for productivity work.

**WARNING:** Don't attempt this if you aren't prepared to recover your system back to the official NVIDIA packages via virtual console or an SSH connection, should something go unexpectedly. I have zero time to help people debug problems with this (significantly cargo-cult) procedure.

## Instructions

This procedure assumes you are building on a multi-architecture Debian bookworm install using amd64/x86\_64 as your primary architecture. (i386 libraries are really only required at this point for Steam itself, but it's straightforward to build them with minimal additional work.)

1. Start from the root of this repository as your working directory. Create a new branch if you are not building 550.

2. Make sure you have `EMAIL` or `DEBEMAIL` set in your environment. Create an updated changelog entry with the version number of the upstream package you are intending to use, followed by a stable suffix you can use to pin the package via `apt_preferences`:

```
debchange --distribution UNRELEASED --newversion 550.78-1krose1
```

3. Either download the upstream package and manually place it in the `amd64/` subdirectory, or do the following:

```
./debian/rules get-orig-source-download/amd64
mv get-orig-source amd64
```

4. Build for amd64:

```
DEB_BUILD_OPTIONS=nocheck dpkg-buildpackage -b --no-sign
```

The build may fail, and usually indicates the kind of thing you need to update to get it working again. For instance, I had to update `debian/nv-readme.ids` by simply copying the generated file over the one checked into the repo. It's likely you will need to install some build dependencies if this is your first time building the NVIDIA driver packages.

5. Build for i386:

```
DEB_BUILD_OPTIONS=nocheck dpkg-buildpackage -a i386 --target-arch i386 -b --no-sign
```

6. Dump the list of installed packages for whatever your current Debian-supported version of the NVIDIA driver is. For example:

```
apt-show-versions | fgrep 525.147 >/tmp/nv
```

7. Attempt to install this same set of packages using the new versions you just built, plus `nvidia-modprobe` and `nvidia-settings`:

```
sudo apt install $(cat /tmp/nv | \
    sed -e 's/^\([^:]*\):\([^ /]*\).*/.\/\1_550.78-1krose1_\2.deb/' | xargs) \
    ./nvidia-settings_550.78-1krose1_amd64.deb \
    ./nvidia-modprobe_550.78-1krose1_amd64.deb
```

   This may fail with an error indicating additional packages you need to install. (I, for instance, needed to add `./libnvidia-pkcs11-openssl3_550.78-1krose1_amd64.deb`, `./libnvidia-gpucomp_550.78-1krose1_amd64.deb`, and `./libnvidia-gpucomp_550.78-1krose1_i386.deb`.) Add those missing packages to your `apt install` command line and repeat until the install commences.

8. Hopefully the install succeeds! 🤞

9. Finally, pin this version so it isn't replaced by upstream. I pin all packages with my custom version suffix using the following lines in `/etc/apt/preferences.d/01krose`:

```
Package: *:any
Pin: version *krose?
Pin-Priority: 1011
```

## Changes from Debian

* Revert to packaging upstream `nvidia-settings` and `nvidia-modprobe` instead of using out-of-band source packages.

* Remove backport patches.

* Fix the kbuild patch for changed context in upstream.
