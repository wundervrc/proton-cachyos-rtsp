# proton-cachyos-rtsp

**CachyOS Proton plus the Proton-RTSP media patchset. 
So RTSP and other live streams play in VRChat without breaking normal video playback.**

This is a merge of two existing projects. **Pretty much none of the work here is mine.**
See [Credits](#credits) 
Please go star and support the people who actually built all of this.

---

## Credits

### CachyOS Proton — the base

Everything about this build's performance comes from
**[CachyOS/proton-cachyos](https://github.com/CachyOS/proton-cachyos)**. This fork is
based on their `cachyos-11.0-20260703-slr` release and carries their entire tree:
their Wine patches, scheduler and thread-priority work, DXVK/VKD3D variants,
`dxvk-sarek`, `d7vk`, low-latency layers, FEX, NVIDIA libraries, protonfixes and much
more.

Huge thanks to the CachyOS team and to everyone whose patches land in their Wine tree.
Including **Etaash Mathamsetty**, **Stelios Tsampas** (loathingkernel), **M0n7y5**,
**NelloKudo**, **Rémi Bernon**, **Paul Gofman**, and many others.

CachyOS Proton is itself built on **[Valve's Proton](https://github.com/ValveSoftware/Proton)**
and **[Wine](https://www.winehq.org/)**
Thanks to Valve, CodeWeavers, and the Wine project.

### Proton-RTSP — the streaming support

The RTSP/live-media support is
**[SpookySkeletons/proton-rtsp](https://github.com/SpookySkeletons/proton-rtsp)**.
Thanks to **SpookySkeletons** for maintaining, rebasing and packaging the patchset, and 
for doing the work that makes VRCDN livestreams play in VRChat at all.

The media patchset itself is overwhelmingly the work of **Torge Matthies / Reyka
Matthies** ([openglfreak](https://github.com/openglfreak)) — 69 of the 72 commits.
Also **zhineng cao** (the `qcap` webcam fixes behind VRChat desktop selfie / upper-body
tracking) and **kazu0617**. The superproject `enable rtsp` change is by
**BabbleBones**.

This repo exists solely because I wanted CachyOS's optimizations *and* Spooky's RTSP support in a single build.
Claude helped to achieve that. This repo stands on the shoulders of giants.

---

## What this actually adds

Compared to stock `proton-cachyos`, exactly two things:

1. The **`rtsp`, `rtp` and `rtpmanager`** GStreamer plugins are enabled
   (`Makefile.in`), matching what Proton-RTSP ships.
2. The **72-commit RTSP Wine patchset** is rebased on top of CachyOS's Wine.

Both trees are based on the same upstream Proton Wine
(`experimental-bleeding-edge-11.0-20260609`), so this is a genuine rebase rather than a
graft.

Tested in VRChat: RTSP streams work, and YouTube and movie worlds still work.

---

## Install

Download the `.tar.xz` from [Releases](../../releases) and extract it into your Steam
compatibility tools directory:

```bash
mkdir -p ~/.steam/steam/compatibilitytools.d
tar -xf proton-cachyos-rtsp-*.tar.xz -C ~/.steam/steam/compatibilitytools.d/
```

Flatpak Steam:

```bash
tar -xf proton-cachyos-rtsp-*.tar.xz -C ~/.var/app/com.valvesoftware.Steam/data/Steam/compatibilitytools.d/
```

Then **restart Steam**, and set VRChat → Properties → Compatibility →
*proton-cachyos-rtsp-…*.

### One extra step for YouTube/Twitch

Run this once, as Proton-RTSP also instructs:

```bash
steam steam://unlockh264/
```

---

## Launch options / flags

**This build adds exactly one flag of its own.** Everything else comes from CachyOS and
from the projects they bundle.

### Added here

| Flag | Default | What it does |
|---|---|---|
| `PROTON_MEDIA_COMPRESSED_STREAMS=1` | off | Restores CachyOS's *compressed-first* media source. By default this build always hands decoded streams to the game, which is what the RTSP patchset expects. **You almost certainly do not need this** — it exists as a diagnostic if some non-VRChat title has video trouble. |

### Everything else

Do **not** treat the list below as complete. CachyOS frequently ships a new flag in a
release before it reaches any README:

- **Read the [CachyOS Proton releases page](https://github.com/CachyOS/proton-cachyos/releases)
  first.** Each release documents its new options, and links out to the projects it
  bundles — each of which has its own flags you can use here.
- [CachyOS Proton repo](https://github.com/CachyOS/proton-cachyos)
- [Proton-RTSP releases](https://github.com/SpookySkeletons/proton-rtsp/releases) — for
  RTSP-specific notes and known issues

Some of the bundled projects and their own documentation:

- [Wayland / EM additions](https://github.com/Etaash-mathamsetty/Proton/blob/em-10/docs/EM-ADDITIONS.md)
- [FSR4](https://github.com/Etaash-mathamsetty/Proton/blob/em-10/docs/FSR4.md)
- [dxvk-sarek](https://github.com/pythonlover02/DXVK-Sarek#shader-compilation)
- [dxvk-low-latency](https://github.com/netborg-afps/dxvk-low-latency#dxvk-low-latency)
- [vkd3d-low-latency](https://github.com/netborg-afps/vkd3d-low-latency#vkd3d-low-latency)


---

## Building

Needs Podman or Docker; everything else builds inside the Steam Runtime SDK container.

```bash
git clone --recurse-submodules https://github.com/wundervrc/proton-cachyos-rtsp.git
mkdir build && cd build
CFLAGS="-O3 -march=nocona -mtune=core-avx2" \
RUSTFLAGS="-Copt-level=3 -Ctarget-cpu=nocona" \
../proton-cachyos-rtsp/configure.sh \
    --container-engine=podman --enable-ccache \
    --build-name=proton-cachyos-rtsp
make -j$(nproc)          # or: make -j$(nproc) redist   to get a .tar.xz
make install             # installs into ~/.steam/root/compatibilitytools.d/
```

> The `wine` submodule points at
> [wundervrc/wine-cachyos-rtsp](https://github.com/wundervrc/wine-cachyos-rtsp)
> (branch `rtsp-merge`), which is CachyOS's Wine with the RTSP patchset rebased on top.
> Cloning **without** `--recurse-submodules` and initialising submodules from elsewhere
> will silently give you a build with no RTSP support.

Upstream Proton's original build documentation is preserved at
[docs/README.upstream-proton.md](docs/README.upstream-proton.md). Detailed notes on how
the merge was done — every conflict resolution, what was ruled out and why — are in
[BUILD-NOTES.md](BUILD-NOTES.md).

---

## Licensing

Unchanged from upstream Proton and Wine. Each component is used under the terms of its
own license; see `LICENSE`, `dist.LICENSE`, and the `LICENSE`/`COPYING` files in each
submodule. If you redistribute a build, you must comply with those terms.
