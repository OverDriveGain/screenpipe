# Linux build notes (OverDriveGain fork)

Two workarounds were needed to build the Tauri desktop app on Ubuntu 24.04 (Linux Mint 22). One is a source patch on this branch; two are environment setup.

## 1. Source patch — `apps/screenpipe-app-tauri/scripts/find_tools.js`

Upstream's `downloadFile` uses `fetch()` + `Bun.write(destination, response)`. On Linux, Bun pegged a CPU core at 96% with no network traffic and no syscalls; the 30 s abort timer never fired. Replaced with a `curl` shell-out via `bun's $\`…\`` helper.

## 2. Apt deps — install before `bun tauri build`

The CONTRIBUTING.md list misses `libopenblas-dev` (provides `cblas.h`, needed by `antirez-asr-sys` for Qwen ASR):

```bash
sudo apt-get install -y \
  g++ tesseract-ocr cmake \
  libavformat-dev libavfilter-dev libavdevice-dev libavutil-dev \
  libssl-dev libtesseract-dev libxdo-dev libsdl2-dev \
  libclang-dev libxtst-dev libpipewire-0.3-dev \
  libayatana-appindicator3-1 libayatana-appindicator3-dev \
  librsvg2-dev libwebkit2gtk-4.1-dev \
  pkg-config build-essential libglib2.0-dev libgtk-3-dev clang \
  libopenblas-dev
```

## 3. Symlink workaround — `antirez-asr-sys` emits wrong link directive

The crate's `build.rs` emits `cargo:rustc-link-lib=libopenblas` (with redundant `lib` prefix). The linker then searches for `liblibopenblas.so` which doesn't exist. Workaround:

```bash
cd /usr/lib/x86_64-linux-gnu
sudo ln -sf libopenblas.so   liblibopenblas.so
sudo ln -sf libopenblas.so.0 liblibopenblas.so.0
sudo ln -sf libopenblas.a    liblibopenblas.a
```

A cleaner fix would be patching upstream `antirez-asr-sys` to use `cargo:rustc-link-lib=openblas` (no `lib` prefix). Source repo: github.com/divanshu-go/audiopipe.git.

## Build command

```bash
cd apps/screenpipe-app-tauri
bun install
bun tauri build --bundles deb
```

Output: `apps/screenpipe-app-tauri/src-tauri/target/release/bundle/deb/screenpipe - Development_2.4.285_amd64.deb`
