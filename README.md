# Fixing Vagrant on CachyOS (Arch-based Rolling Release)

> A complete troubleshooting guide documenting how to resolve Vagrant binary/AppImage
> compatibility issues on CachyOS and similar cutting-edge Arch-based distributions.

---
## Tested Environment
| Component | Version |
|---|---|
| Distro | CachyOS x86_64 |
| Linux kernel | 6.19.10-1-cachyos |
| Vagrant | 2.4.9 |
| readline | 8.3.003-1.1 |
| ncurses | 6.6-2.1 |
| libxcrypt-compat | 4.5.2-1.1 |
| Date | March 2026 |

---

## Table of Contents

- [Context & Background](#context--background)
- [Key Notions](#key-notions)
- [Problem 1 — `undefined symbol: UP` (libreadline)](#problem-1--undefined-symbol-up-libreadline)
- [Problem 2 — Downgrading readline broke bash](#problem-2--downgrading-readline-broke-bash)
- [Problem 3 — `libcrypt.so.1` not found](#problem-3--libcryptso1-not-found)
- [Problem 4 — Wrong vagrant binary being called](#problem-4--wrong-vagrant-binary-being-called)
- [Final Working Solution (Step by Step)](#final-working-solution-step-by-step)
- [Why This Happens on Rolling Releases](#why-this-happens-on-rolling-releases)

---

## Context & Background

Vagrant distributes its Linux binary as an **AppImage** — a self-contained executable that
bundles its own libraries (readline, ruby, etc.) to avoid depending on the host system.
This works well on stable distros (Ubuntu LTS, Debian), but **breaks on rolling-release
distros** like CachyOS or pure Arch, because the host system's libraries have moved ahead
of what was bundled at build time.

The AUR-built vagrant has the same problem because it also ships a pre-bundled Ruby runtime
that references older ABI symbols.

---

## Key Notions

### AppImage
An AppImage is a portable application format for Linux. It mounts itself as a temporary
filesystem under `/tmp/.mount_<random>/` when executed, and runs from there. Because it
bundles its own libraries, it is supposed to "just work" on any distro — but this
assumption breaks when the bundled libs have ABI incompatibilities with the kernel or
libc on the host.

### ABI (Application Binary Interface)
The ABI defines how compiled code communicates at the binary level — function names,
symbol names, calling conventions. When a library drops or renames a symbol between
versions, binaries compiled against the old version will fail with `undefined symbol: X`
at runtime, even if the library is technically "the same" major version.

### LD_PRELOAD
`LD_PRELOAD` is an environment variable that forces the dynamic linker to load a specific
shared library **before** any others — including those bundled inside an AppImage. It can
be used to override a broken bundled library with a working system one.

### Symbol Lookup Error
`bash: symbol lookup error: /path/to/lib.so: undefined symbol: FOO` means the dynamic
linker found the library, but the specific function or variable `FOO` it was expecting
does not exist in it. This is an ABI mismatch between the caller and the library.

### Rolling Release
A rolling release distro (CachyOS, Arch, Manjaro) continuously updates all packages to
their latest versions, with no fixed "release" points. This means system libraries are
always cutting-edge, which can break applications that were compiled against older ABIs.

---

## Problem 1 — `undefined symbol: UP` (libreadline)

### Symptom

```
bash: symbol lookup error: /tmp/.mount_vagranhlLloe/usr/lib/libreadline.so.8: undefined symbol: UP
```

### Root Cause

`UP` is a global variable exported by `ncurses`/`terminfo` that older versions of
`libreadline` depended on. Newer versions of ncurses changed how this symbol is exported
(it moved into `libncursesw.so` with a versioned symbol), so Vagrant's **bundled**
`libreadline.so.8` — compiled against the old ncurses ABI — cannot find it.

CachyOS ships **readline 8.3** and **ncurses 6.6**, both of which have moved past the ABI
that Vagrant's bundled readline expects.

### What does NOT work

- `LD_PRELOAD=/usr/lib/libreadline.so.8 ./vagrant` — the AppImage loads its own copy
  before LD_PRELOAD can intercept it.
- Installing via `paru -S vagrant` — the AUR package uses the same bundled Ruby/readline.
- Downgrading readline system-wide — **dangerous**, see Problem 2 below.

### What works

Extract the AppImage and remove the bundled readline so it falls back to the system one:

```bash
# Extract the AppImage contents
./vagrant --appimage-extract

# Remove the bundled readline (3 symlinks/files)
rm squashfs-root/usr/lib/libreadline.so*

# Test the extracted version
squashfs-root/AppRun --version
```

**Why this works:** Once the bundled readline is removed, Vagrant's runtime looks for
readline in the host's library paths and finds `/usr/lib/libreadline.so.8` — which is
correctly linked against the system's ncurses and has no missing symbols.

---

## Problem 2 — Downgrading readline broke bash

### Symptom

After running `sudo downgrade readline` (to readline 8.2):

```
/bin/sh: symbol lookup error: /bin/sh: undefined symbol: rl_print_keybinding
/bin/bash: symbol lookup error: bash: undefined symbol: rl_print_keybinding
```

### Root Cause

`rl_print_keybinding` is a symbol **added in readline 8.3**. CachyOS's `bash` package was
compiled against readline 8.3 and requires this symbol. Downgrading readline to 8.2
removed the symbol, breaking bash itself.

This is a cascade failure: downgrading one core library on a rolling release can break
everything that was compiled against the newer version.

### Fix — Restore readline 8.3 immediately (do not close the terminal)

```bash
# Use the still-running shell session to pull readline back from the mirror
sudo pacman -U "$(pacman -Sp readline | head -1)"

# Or from the local cache if available
sudo pacman -U /var/cache/pacman/pkg/readline-8.3*.pkg.tar.zst
```

**Key lesson:** Never downgrade core system libraries (`readline`, `glibc`, `ncurses`,
`openssl`) on a rolling release distro to fix a third-party application. The risk of
breaking the entire system is very high.

---

## Problem 3 — `libcrypt.so.1` not found

### Symptom

After removing the bundled readline:

```
/home/rsh/Downloads/squashfs-root/usr/bin/ruby: error while loading shared libraries:
libcrypt.so.1: cannot open shared object file: No such file or directory
```

### Root Cause

Vagrant's bundled Ruby was compiled against `libcrypt.so.1`, which is the legacy crypt
library. Arch Linux (and CachyOS) migrated from `libxcrypt` compat to the newer
`libxcrypt` which only ships `libcrypt.so.2`. The `.so.1` version is no longer installed
by default.

### Verification

```bash
find /usr/lib -name "libcrypt*"
# Shows: libcrypt.so, libcrypt.so.2, libcrypt.so.2.0.0
# Missing: libcrypt.so.1
```

### Fix

Install the compatibility package that provides `libcrypt.so.1`:

```bash
paru -S libxcrypt-compat
```

**Why this works:** `libxcrypt-compat` ships the old `libcrypt.so.1` alongside the new
`.so.2`, allowing legacy binaries compiled against the old ABI to continue working
without affecting modern packages that use `.so.2`.

---

## Problem 4 — Wrong vagrant binary being called

### Symptom

After setting up `/usr/local/bin/vagrant`, running `vagrant --version` still triggered the
broken AppImage error. Investigation showed:

```bash
which vagrant
# /home/rsh/.local/bin/vagrant   ← wrong! old AUR-installed binary
```

### Root Cause

The AUR-installed vagrant had placed a binary at `~/.local/bin/vagrant`. Because
`~/.local/bin` appears **before** `/usr/local/bin` in the user's `$PATH`, it was being
called instead of the new wrapper.

### Fix

```bash
# Remove the shadowing binary
rm ~/.local/bin/vagrant

# Verify the correct one is now used
which vagrant
# /usr/local/bin/vagrant  ✓

vagrant --version
# Vagrant 2.4.9  ✓
```

**Key lesson:** Always check `which -a vagrant` (lists all matches in PATH order) when a
binary doesn't behave as expected after installation. PATH precedence is a common source
of confusion, especially when mixing system packages, AUR packages, and manual installs.

---

## Final Working Solution (Step by Step)

```bash
# 1. Download the official Vagrant binary AppImage from hashicorp.com
#    (or use the one from paru — same result)

# 2. Install the missing compat library
paru -S libxcrypt-compat

# 3. Extract the AppImage
./vagrant --appimage-extract
# Creates: squashfs-root/

# 4. Remove the bundled readline (the root cause of the UP symbol error)
rm squashfs-root/usr/lib/libreadline.so*

# 5. Test the extracted binary
squashfs-root/AppRun --version
# Vagrant 2.4.9

# 6. Move to a permanent location
sudo mv squashfs-root /opt/vagrant

# 7. Create a system-wide wrapper script
sudo tee /usr/local/bin/vagrant > /dev/null << 'EOF'
#!/bin/bash
exec /opt/vagrant/AppRun "$@"
EOF
sudo chmod +x /usr/local/bin/vagrant

# 8. Remove any shadowing binaries from previous installs
rm ~/.local/bin/vagrant  # if it exists

# 9. Verify
which vagrant          # should be /usr/local/bin/vagrant
vagrant --version      # should print Vagrant 2.4.9
```

---

## Why This Happens on Rolling Releases

| Factor | Stable Distro (Ubuntu LTS) | Rolling Release (CachyOS/Arch) |
|---|---|---|
| readline version | 8.1 or 8.2 | 8.3+ |
| ncurses version | 6.3 | 6.6+ |
| libcrypt | `.so.1` present | `.so.2` only (`.so.1` optional) |
| bash compiled against | readline 8.1 | readline 8.3 |
| AppImage compatibility | ✅ Works out of box | ❌ ABI mismatches |

AppImage vendors test against LTS distros. Rolling release users are on their own — but
the extract-and-patch approach described in this guide works around the issue reliably
without compromising system integrity.

---

*Tested on CachyOS with readline 8.3.003, ncurses 6.6, Vagrant 2.4.9.*
