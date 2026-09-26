# Building HexChat on macOS

This document describes the current procedure for building and
installing **HexChat 2.16.2-1** natively on Apple Silicon macOS.

The build has been verified on an **M4 Mac mini running macOS 27**. The
resulting executable is a native **ARM64 Mach-O** binary; Rosetta, a
virtual machine, and cross-compilation are not required.

> This is currently a Unix-style GTK2 build running natively on macOS.
> It is **not yet a polished macOS `.app` bundle** and does not provide
> full native macOS desktop integration.

## Status

HexChat 2.16.2-1 currently:

-   configures with Meson on macOS;
-   compiles natively for Apple Silicon;
-   installs under `/opt/homebrew`;
-   runs successfully;
-   loads and runs the Sysinfo plugin;
-   reports storage correctly on macOS/BSD `df`.

Known desktop-integration limitations include:

-   no `.app` bundle in `/Applications`;
-   the menu bar is drawn inside the HexChat window rather than exported
    to the macOS global menu bar;
-   no signing or notarization work has been done yet.

## Requirements

The build uses Xcode command-line development tools and Homebrew.

Install the required Homebrew packages:

``` sh
brew install ninja meson openssl@3 perl perl-build gtk+ glib make pkg-config desktop-file-utils
```

### GTK2 note

HexChat 2.16.2 uses GTK2. Homebrew installs `gtk-update-icon-cache` for
`gtk+` under:

``` text
/opt/homebrew/opt/gtk+/libexec/gtk-update-icon-cache
```

This location is not normally in `PATH`, because Homebrew avoids a
conflict with the GTK3 formula. It matters during `meson install`.

## Get the source

Clone the maintained fork:

``` sh
git clone https://github.com/LinuxTechLife247/hexchat.git
cd hexchat
```

The current maintenance revision is **2.16.2-1**.

## Configure

Create an out-of-tree build directory:

``` sh
meson setup build
```

Meson should detect the Homebrew dependencies, including OpenSSL.

## Compile

``` sh
meson compile -C build
```

The build-tree executable can be tested before installation:

``` sh
./build/src/fe-gtk/hexchat
```

## Install

Because Homebrew's GTK2 `gtk-update-icon-cache` is installed in `gtk+`'s
`libexec` directory, add that directory to `PATH` for the install step:

``` sh
PATH="/opt/homebrew/opt/gtk+/libexec:$PATH" meson install -C build
```

`desktop-file-utils` supplies `update-desktop-database`, which the Meson
post-install script also invokes.

After installation:

``` sh
which hexchat
file /opt/homebrew/bin/hexchat
```

A successful Apple Silicon installation should report
`/opt/homebrew/bin/hexchat`, with `file` identifying it as a 64-bit
ARM64 Mach-O executable.

Run it with:

``` sh
hexchat
```

## macOS portability fixes in 2.16.2-1

Two source-level portability problems were exposed while building
upstream HexChat 2.16.2 on current macOS.

### OpenSSL dependency propagation

OpenSSL was detected correctly by `pkg-config`, but compilation of GTK
frontend sources failed with:

``` text
fatal error: 'openssl/ssl.h' file not found
```

The common HexChat headers expose OpenSSL headers to consumers, while
the Meson dependency exported by `hexchat_common_dep` did not propagate
the OpenSSL dependency. Linux commonly masks this because OpenSSL
headers are available through normal compiler search paths; Homebrew's
keg-only OpenSSL makes the missing dependency propagation visible.

The Meson configuration in 2.16.2-1 propagates OpenSSL to consumers of
the common dependency rather than hard-coding a Homebrew include path.

### Sysinfo storage reporting

The Sysinfo plugin originally invoked:

``` sh
df -k -l -P --exclude-type=squashfs --exclude-type=devtmpfs --exclude-type=tmpfs
```

Those `--exclude-type` options are GNU `df` options. The BSD `df`
supplied by macOS rejects them, causing Sysinfo to receive no usable
filesystem data and report:

``` text
Storage: 0 bytes / 0 bytes (0 bytes Free)
```

On macOS, using the supported BSD command:

``` sh
df -k -l -P
```

allows the existing Sysinfo parser to work correctly.

A verified result after the fix was:

``` text
Client: HexChat 2.16.2-1 • OS: OS X 27.0.0 • Memory: 24.0 GiB Total (10.8 GiB Free) • Storage: 3.4 TB / 17.1 TB (13.6 TB Free) • Uptime: 2h 3m 18s
```

Do **not** add `-h` to the command used by Sysinfo. Its parser expects
the numeric 1024-block output produced by `df -k`.

## Cleaning and rebuilding

Because Meson uses an out-of-tree build directory, the source tree can
be returned to a clean pre-build state simply by removing the build
directory:

``` sh
rm -rf build
```

Then configure again with:

``` sh
meson setup build
```

## What this proves

The current result demonstrates that HexChat's existing GTK2 codebase
can still **configure, compile, install, and run natively on current
Apple Silicon macOS** with small portability corrections.

It does not yet constitute a conventional macOS application
distribution. Creating a `.app` bundle, improving macOS menu
integration, bundling runtime dependencies, and handling code
signing/notarization are separate future work.
