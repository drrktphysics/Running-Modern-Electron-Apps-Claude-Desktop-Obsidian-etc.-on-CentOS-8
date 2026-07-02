# Running Modern Electron Apps (Claude Desktop, Obsidian, etc.) on CentOS 8

CentOS 8 is EOL, and most modern Electron-based desktop apps (Claude Desktop, Obsidian, and others) are built and tested against Ubuntu/Debian. This causes two recurring, predictable failure classes. This guide documents both, with a general methodology you can reuse for *any* similar app.

## Symptom class 1: Missing shared library at launch

```
error while loading shared libraries: libffmpeg.so: cannot open shared object file: No such file or directory
```

**Cause:** The app's RPATH/search path doesn't include the directory where its bundled library actually lives (common in Snap and some Electron packaging).

**Fix:**
```bash
ldd /path/to/binary | grep <missing-lib>      # confirm it's actually unresolved
find / -name "<missing-lib>" 2>/dev/null      # locate where it actually is
sudo cp /path/to/found/lib.so /usr/lib/x86_64-linux-gnu/   # or wherever ldconfig scans
sudo ldconfig
```
If the app is Snap-packaged, environment variables like `LD_LIBRARY_PATH` usually **will not help** — Snap confinement resets the environment before the binary runs. Prefer non-Snap packaging (`.rpm`, `.deb`, AppImage) for legacy distros.

## Symptom class 2: App launches but no window appears / GPU init errors

```
MESA-LOADER: failed to open swrast: /usr/lib/x86_64-linux-gnu/dri/swrast_dri.so: cannot open shared object file
ANGLE Display::initialize error 12289: Failed to get system egl display
```

**Cause:** The Electron binary is hardcoded to look for Mesa DRI drivers at the **Debian/Ubuntu path** (`/usr/lib/x86_64-linux-gnu/dri/`). CentOS/RHEL ships drivers at a **different path** (`/usr/lib64/dri/`). Installing Mesa packages via `dnf` doesn't fix this — the drivers are installed correctly, the app just isn't looking there.

**Fix — symlink the CentOS driver path into the path the app expects:**
```bash
sudo mkdir -p /usr/lib/x86_64-linux-gnu/dri
sudo ln -sf /usr/lib64/dri/*.so /usr/lib/x86_64-linux-gnu/dri/
```

**Also launch with these flags** (harmless on any system, and necessary if the host has no real GPU/driver stack, e.g. a VM):
```bash
./App.AppImage --disable-gpu --no-sandbox
```

## Why Snap packages are the wrong choice here

If the app is available as a Snap, it's tempting to use it — but Snap runs in a confined mount namespace with **its own filesystem view**. Symlinks and environment variables you set on the host (like the DRI path fix above) are invisible to the sandboxed process. We confirmed this directly: the same DRI errors persisted inside Snap even after the symlink fix worked perfectly for the unsandboxed AppImage.

**Rule of thumb for old/unsupported distros: prefer AppImage or a native `.rpm`/`.deb` over Snap or Flatpak.** Unsandboxed formats let you patch around distro mismatches directly; sandboxed formats don't.

## Flatpak-specific gotcha

Old Flatpak versions (CentOS 8 ships ~1.6.x) can't parse Flathub's newer, larger summary index format:
```
error: Unable to load summary from remote flathub: URI ... exceeded maximum size of 10485760 bytes
```
This requires upgrading Flatpak itself (nontrivial on CentOS 8) — usually faster to skip Flatpak and use AppImage instead.

## General reusable methodology (apply to any new machine/app)

1. **Try the official install method first** (apt repo, rpm repo, or `.deb`/`.rpm` from vendor). Note the target distro it was built for (check filenames — e.g. `.fc42.x86_64` means Fedora 42, likely to have a newer glibc than your CentOS 8 box; may still work, may not).
2. **If it fails to launch — read the exact error.** `cannot open shared object file` = missing/misplaced library. Find it with `find` and `ldd`, then symlink/copy it to where the loader expects it, then `ldconfig`.
3. **If it launches but shows no window — check for MESA/EGL/GPU errors in the terminal output.** This is almost always the `/usr/lib/x86_64-linux-gnu/dri` vs `/usr/lib64/dri` path mismatch. Symlink one into the other, and add `--disable-gpu --no-sandbox`.
4. **If using Snap or Flatpak and stuck — switch to AppImage or native package.** Sandboxing defeats host-level fixes.
5. **Wrap the final working launch command in a script**, symlink it into `~/.local/bin` (ensure it's on `PATH`), and optionally create a `.desktop` file so it appears in your app menu:
   ```bash
   # ~/Applications/app.sh
   #!/bin/bash
   ~/Applications/App.AppImage --disable-gpu --no-sandbox "$@"
   ```
   ```bash
   chmod +x ~/Applications/app.sh
   ln -sf ~/Applications/app.sh ~/.local/bin/appname
   ```

This methodology was verified end-to-end on CentOS 8 (Ubuntu 24.04-era Snap builds, Fedora 42-targeted RPM builds, and official AppImages), fixing both Claude Desktop and Obsidian installs using the same two root causes.
