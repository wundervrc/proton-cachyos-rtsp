# CachyOS Proton + RTSP — build notes

Goal: a Proton build that keeps **CachyOS's optimizations** but gains **SpookySkeletons'
RTSP/livestream media patchset**, so VRChat can play RTSP and other stream types
*without* regressing normal video playback (the earlier GE-Proton-based attempt broke
"movie world" playback).

Started 2026-09-02. Working dir: `/home/wunder/Projects/cachyosProtonRTSP`.

## User directives (standing)

1. **Always run `pwd` before any `rm`/delete.**
2. **For anything video-related, prefer the RTSP side.** Merge both sides only when it's
   straightforward; otherwise take RTSP wholesale. RTSP is known-working in game today;
   CachyOS's value here is its *non-media* optimizations.
3. Build is for VRChat specifically, with the user's own launch flags.

## Published forks

| what | repo | branch | tag |
|---|---|---|---|
| superproject | <https://github.com/wundervrc/proton-cachyos-rtsp> | `cachyos-rtsp` | `cachyos-rtsp-built-20260902` |
| wine | <https://github.com/wundervrc/wine-cachyos-rtsp> | `rtsp-merge` | `cachyos-rtsp-built-20260902` |

Both are public forks of the CachyOS repos, so upstream history is intact and our work sits
on top as normal commits. `.gitmodules` in the superproject points `wine` at the
**wine-cachyos-rtsp** fork, so a fresh `git clone --recurse-submodules` of the superproject
gets the RTSP wine, not CachyOS's stock wine.

To rebuild from scratch on another machine:

```bash
git clone --recurse-submodules https://github.com/wundervrc/proton-cachyos-rtsp.git
cd proton-cachyos-rtsp && git checkout cachyos-rtsp-built-20260902
mkdir ../build && cd ../build
../proton-cachyos-rtsp/configure.sh --container-engine=podman --enable-ccache \
    --build-name=proton-cachyos-rtsp
make -j$(nproc) && make install
```

## Layout

```
cachyosProtonRTSP/
├── BUILD-NOTES.md          <- this file
├── proton-rtsp/            <- SpookySkeletons/proton-rtsp (reference only, not built)
└── proton-cachyos/         <- THE BUILD TREE. branch: cachyos-rtsp
    └── wine/               <- merged wine. branch: rtsp-merge
                               (this is CachyOS/wine-cachyos + RTSP series, moved in
                                from ../wine-cachyos; `wine` submodule is deliberately
                                skipped via `git config submodule.wine.update none`)
```

## Upstream references

| | repo | tag/branch | commit |
|---|---|---|---|
| RTSP proton | `SpookySkeletons/proton-rtsp` | `proton-rtsp-11.0-20260609-3` | `f3f25735` |
| RTSP wine | `SpookySkeletons/wine` | `rtsp-11.0-20260609-3` | `a7fcac55` |
| CachyOS proton | `CachyOS/proton-cachyos` | `cachyos-11.0-20260703-slr` | `ea0053a9` |
| CachyOS wine | `CachyOS/wine-cachyos` | `cachyos_11.0_20260702/main` | `b5f2dc7b` |

**The key fact that made this tractable:** both wine forks contain Proton
experimental-bleeding-edge 11.0-20260609 wine (`e3b1bcb2`) as a common ancestor.
`git merge-base b5f2dc7b a7fcac55` == `e3b1bcb2`, and the RTSP series is exactly
**72 commits** on top of it. So it rebases cleanly rather than being force-fitted.

`git cherry` reported **no** exact patch-id duplicates between the two sides.

## What the RTSP patchset actually is

- **Superproject: one commit** (`bb45aaac` "enable rtsp"), which only:
  - adds `-Drtsp=enabled -Drtp=enabled -Drtpmanager=enabled` to `GST_GOOD_MESON_ARGS`
  - makes `configure.sh`'s container-engine UID detection robust to extra output lines
- **Everything else is in the wine fork**: 72 commits, mostly openglfreak's
  (Torge/Reyka Matthies) media patchset — `winegstreamer`, `mf`, `mfmediaengine`,
  `mfplat`, `mfsrcsnk`, `quartz` — plus some ntdll/server perf patches, `qcap` webcam
  fixes (VRChat selfie / upper-body tracking), and an opt-in yt-dlp redirect.

Ground truth on plugins: the user's installed `proton-rtsp-11.0-20260609-2` ships
`libgstrtsp.so`, `libgstrtp.so`, `libgstrtpmanager.so` and **no** `libgstudp.so` —
so RTSP runs interleaved over TCP. Our flags match exactly.

## Work completed

### wine (`proton-cachyos/wine`, branch `rtsp-merge`)

Base `b5f2dc7b` → HEAD `b9eeaf79b5a`. 72 cherry-picked + 1 of mine.

```bash
git cherry-pick --empty=keep e3b1bcb2..a7fcac55
```

**Four conflicts, all resolved preferring RTSP semantics:**

1. `dlls/winegstreamer/wg_parser.c` — commit `546384a7` (caps negotiation).
   Kept RTSP's `next_desired_caps` machinery **and** CachyOS's two additions:
   the `PROTON_GST_VIDEO_ORIENTATION` env override (now also sets `need_reconf = TRUE`
   so it still forces a reconfigure, matching the old unconditional `push_event`), and
   `stream->fix_align = fix_nv12 || fix_yv12` (their YV12/I420 alignment fix) in place
   of RTSP's `stream->fix_nv12`.

2. `dlls/winegstreamer/media_source.c` — commit `61c71744` (buffering events).
   CachyOS refactored the fail path into `media_source_release_streams()` /
   `media_source_destroy_parser()`. Taught `media_source_destroy_parser()` to tear down
   the RTSP `event_thread` right after `wg_parser_disconnect`, matching RTSP's order.
   **This also fixes a latent bug in CachyOS's retry path**, which re-created the event
   thread on `goto retry` without ever tearing it down.

3. `dlls/winegstreamer/media_source.c` — commit `9d3c10dc` (per-stream work queues).
   Moved `MFUnlockWorkQueue(stream->async_commands_queue)` into
   `media_source_release_streams()`.

4. `dlls/winegstreamer/media_source.c` + `wg_parser.c` — commit `a1776b1b` (hang fix).
   Adopted RTSP's teardown semantics inside `media_source_destroy_parser()`: set both
   shutdown flags *first*, then disconnect whenever a parser exists (RTSP's companion
   `wg_parser.c` hunks NULL-harden `wg_parser_disconnect`). Dropped the now-redundant
   `connected` parameter and the `parser_connected` variable to avoid an unused-var
   warning under `-Werror`.

5. `dlls/winevulkan/make_vulkan` — commit `fe09d4a2` (`treewide: -Werror` fixes).
   **Took CachyOS's side.** Their `UNIX_CALL_CHECKED` macro (`vulkan_loader.h:148`)
   already consumes `status`, so RTSP's `(void)status;` hunk is obsolete. Non-media,
   so directive 2 doesn't apply.

**Plus one deliberate behavioral change of mine** — commit `b9eeaf79b5a`
`winegstreamer: Default to decoded streams for the RTSP media path.`

CachyOS carries `b18ed3f9440 winegstreamer: Retry media source creation with decoded
streams` (NelloKudo — **CachyOS-only, not upstream Proton**), which makes the media
source expose *native compressed* stream types first and only fall back to decoded
output if the app rejects them with `MF_E_INVALIDMEDIATYPE` at creation time. A live
RTSP source can't renegotiate later, and if the app accepts compressed types but then
can't decode them there is no fallback. That is exactly the shape of "broke other video
playback." So `media_source_create()` now defaults `compressed = FALSE` (RTSP's
behavior) via a new `use_compressed_streams()` helper; set
**`PROTON_MEDIA_COMPRESSED_STREAMS=1`** to get CachyOS's behavior back.

**This is the #1 suspect if video regresses — flip that env var to test.**

### superproject (`proton-cachyos`, branch `cachyos-rtsp`)

Uncommitted working-tree changes (2 files, matches Spooky's `bb45aaac`):
- `Makefile.in`: added `-Drtsp/-Drtp/-Drtpmanager=enabled` to the second
  `GST_GOOD_MESON_ARGS +=` block
- `configure.sh`: container UID detection reads `tail -n1` of output, and the
  "Permission denied" / "Emulate Docker CLI" checks test the full output

## Verification done (static review, pre-build)

- `wg_parser_connect()` is consistently 4-arg (`..., BOOL has_bytestream`) across
  `main.c`, `gst_private.h`, `quartz_parser.c`, `wm_reader.c`, `media_source.c`,
  `wg_parser.c` incl. the wow64 thunk. RTSP changed this signature; CachyOS was 3-arg.
- `fix_align` / `fix_nv12` / `fix_yv12` consistent in `wg_parser.c`.
- No conflict markers anywhere in `dlls/`.
- **Routing**: `winegstreamer.rgs` registers `http:`, `https:`, `rtsp:`, `rtspu:`,
  `rtspt:`, `rtsph:`, `rtsp-sdp:`, `rtsps:`, `rtspsu:`, `rtspst:`, `rtspsh:` to the
  GStreamer Scheme Handler. `mf.rgs` only claims `file:`. So VRChat's URL playback goes
  through the RTSP-patched winegstreamer path. CachyOS's `winedmo`-first default
  (`use_gst_byte_stream_handler()` in `dlls/mfsrcsnk/media_source.c`) only affects
  **byte-stream / local-file** handling, not URLs.
- `eef101d3` (libsoup/souphttpsrc hook) uses `dlopen("libsoup-3.0.so")` — no build-time
  dep; CachyOS already builds libsoup as a `GST_GOOD_DEPENDS`.
- `48e004af` (yt-dlp redirect) is opt-in via `PROTON_YTDLP_PATH`; inert if unset.
- Both trees build `gst-libav`; CachyOS's ffmpeg enables all decoders + explicit h264/hevc.
- CachyOS's other risky-looking media patches were checked and left alone:
  `abf4e04790b` (quartz/DirectShow — not VRChat's path) and `924c786e740`
  (`wg_transform.c` B-frame PTS fix — RTSP doesn't touch that function).

## Build-time verification (attempt 1)

- `obj-gst_good-x86_64/meson-info/intro-buildoptions.json` reports
  `rtsp = enabled`, `rtp = enabled`, `rtpmanager = enabled`, `soup = enabled`,
  `udp = auto` (→ disabled, because `-Dauto_features=disabled`; matches Spooky's build,
  which ships no `libgstudp.so`).
- `libgstrtsp-1.0.so` (the gst-plugins-base RTSP library) builds for both i386 and x86_64.

## Build environment

- podman rootless, overlay driver, storage at `~/.local/share/containers/storage`
- SDK image **pulled**: `registry.gitlab.steamos.cloud/proton/steamrt4/sdk/x86_64:4.0.20260331.220802-0` (7.02 GB)
- `ccache` 4.13.6 installed (via `sudo pacman -S ccache` in tmux session `protonbuild`)
- 16 cores, 31 GB RAM, ~970 GB free on `/home`
- Sudo needs a password → use the `protonbuild` tmux session and ask the user to attach.

## Build command

Already configured; `build/Makefile` exists. To rebuild:

```bash
cd /home/wunder/Projects/cachyosProtonRTSP/build
make -j16 > build.log 2>&1
```

To re-configure from scratch:

```bash
cd /home/wunder/Projects/cachyosProtonRTSP
mkdir -p build && cd build
../proton-cachyos/configure.sh \
    --container-engine=podman \
    --enable-ccache \
    --build-name=proton-cachyos-rtsp
```

Then `make install` drops it into `~/.steam/steam/compatibilitytools.d/` as
`proton-cachyos-rtsp`. Restart Steam afterwards.

Build runs in the SteamRT4 SDK container via podman, with `~/.ccache` and `~/.cargo`
bind-mounted in. Host flags baked in by configure: `-O2 -march=nocona -mtune=core-avx2`.

## Status / next steps

- [x] Merge RTSP wine series onto CachyOS wine
- [x] Superproject RTSP enablement
- [x] podman + SDK image + ccache ready
- [x] Submodule checkout — all 51 populated, `wine/` holds the merged tree. Tree is 9.9 GB.
- [x] `configure.sh` succeeded (podman auto-detected, ccache enabled)
- [x] **Build SUCCEEDED first try** — `make -j16`, exit 0, zero compile errors. ~2h wall.
- [x] `make install` → `~/.steam/steam/compatibilitytools.d/proton-cachyos-rtsp` (1.6 GB)
- [x] **Tested in VRChat — WORKS.** RTSP streams play, and YouTube + movie worlds all still
      work. Confirmed by the user 2026-09-02. No regression in normal video playback, which
      was the thing that broke on the previous GE-Proton-based attempt.
- [x] Pushed to the user's GitHub (see "Published forks" above)

### Build attempts log

| # | date | result | notes |
|---|---|---|---|
| 1 | 2026-09-02 | **success, exit 0** | first build of the merged tree; no compile errors at all |

### Post-build verification (all passed)

- `dist/files/lib/{x86_64,i386}-linux-gnu/gstreamer-1.0/` contains `libgstrtsp.so`,
  `libgstrtp.so`, `libgstrtpmanager.so` for **both** architectures. No `libgstudp.so`
  (matches Spooky's build).
- Bonus plugins CachyOS ships that Spooky's build does not, useful for "other stream types":
  `libgstrtmp2.so`, `libgstrist.so`, `libgstsdpelem.so`, `libgstrtponvif.so`,
  `libgstrtpmanagerbad.so`, `libgstsrtp`-adjacent bad plugins, `libgstdash.so`,
  `libgstsmoothstreaming.so`, `libgsthls.so`.
- `strings` on the built `winegstreamer.dll` finds the RTSP scheme registrations
  (`rtspsh`, `rtsp-sdp`, `rtspu`, `rtspt`) → the `.rgs` patch is compiled in.
- `strings` finds `PROTON_MEDIA_COMPRESSED_STREAMS` → our decoded-streams change is live.
- `strings` on unix `winegstreamer.so` finds `libsoup-3.0.so` → the RTSP libsoup hook is in.
- `dlls/winegstreamer/media_source.c` (the most heavily hand-merged file) compiled clean for
  both `x86_64-windows` and `i386-windows` with **no warnings**.

### How to test / roll back

- Steam → VRChat → Properties → Compatibility → **proton-cachyos-rtsp**
- Existing builds are untouched and still selectable for A/B:
  `proton-rtsp-11.0-20260609-1`, `proton-rtsp-11.0-20260609-2`, `GE-Proton11-5`,
  `GE-Proton10-33-rtsp24-1`
- `PROTON_MEDIA_COMPRESSED_STREAMS=1 %command%` restores CachyOS's compressed-first media
  source. **Tested both ways: streams, YouTube and movie worlds work either way.** So the
  default-flip is not load-bearing for VRChat — it stays as insurance for other titles and
  because it's the only thing that still reaches CachyOS's compressed-first code path.
- Useful debug: `PROTON_LOG=1`, and `GST_DEBUG=3` (or `GST_DEBUG=rtspsrc:5`) in the launch
  options; log lands in `~/steam-<appid>.log`.

## Known differences / open risks

**GStreamer version gap — the biggest remaining unknown.**

| tree | gstreamer submodule | version |
|---|---|---|
| proton-rtsp `20260609-3` | `bf6ce1d64a` | **1.22.5** (installed build ships `libgstreamer-1.0.so.0.2205.0`) |
| proton-cachyos `20260703-slr` | `43421c2a5b` | **1.28.2** |

Spooky's RTSP patches were written and tested against **1.22**; we are building them against
**1.28**. `-Drtsp/-Drtp/-Drtpmanager` still exist as meson options in 1.28, so the build
should be fine, but runtime behaviour of `rtspsrc` / `rtpjitterbuffer` / `uridecodebin` has
moved a lot between those releases. Patches most exposed to this:

- `72b22c8943` mark wg_parser container bin as streams-aware
- `868cdd96d3` PLAYING state before the no-more-pads callback
- `61c7174461` buffering events
- `61ec30a1b8` don't fail `autoplug-continue`
- `d9b6992a1c` don't seek live sources

Deliberately **not** downgrading CachyOS's GStreamer — that would throw away CachyOS work and
create a huge conflict surface. Build and test first; only revisit if RTSP playback misbehaves
in a way that smells like negotiation/buffering.

## CPU targeting / optimization levels — settled

CachyOS's release CI (`.github/workflows/release.yml`) builds three x86 variants:

| variant | cflags | rustflags |
|---|---|---|
| `x86_64` (they recommend this) | `-O3 -march=nocona -mtune=core-avx2` | `-Copt-level=3 -Ctarget-cpu=nocona` |
| `x86_64_v3` | `-O3 -march=x86-64-v3 -mtune=core-avx2` | `-Copt-level=3 -Ctarget-cpu=nocona` (unchanged!) |
| `arm64` | default | default |

CI passes these as `CFLAGS` / `RUSTFLAGS` env vars to `configure.sh`, which honours both
(`configure.sh:210-211`).

**Our release build uses the `x86_64` settings verbatim.** Attempt 1 accidentally used the
Makefile default `-O2`; the release build (`build-release/`) is `-O3`, matching CachyOS.

### The user's own system, for reference

- `proton-cachyos-slr 1:11.0.20260703-1` is installed from the plain **`[cachyos]`** repo.
  The `cachyos-v3` / `cachyos-core-v3` / `cachyos-extra-v3` repos **do not carry Proton at
  all**, even though the rest of their system is v3.
- Verified at the instruction level: `objdump -d` on that package's
  `files/lib/wine/x86_64-unix/ntdll.so` finds **0** AVX2 instructions and 1164 SSE.
  It is a baseline build.
- Their CPU supports up to x86-64-v3 (not v4).

## Documentation convention

Keep this file current as work proceeds. When something is tried and **ruled out**, don't
delete it — move it to "Investigated and ruled out" and ~~strike it through~~ with the reason,
so a future session doesn't burn time re-testing the same dead end.

## Investigated and ruled out — do NOT re-check these

Struck-through items were considered, checked, and dismissed. Don't spend time on them again.

- ~~"CachyOS and RTSP both carry a `mf/topology_loader` 'restore media type' fix — do they
  double-restore?"~~ **No.** CachyOS's `04e86314f5d` is **tests-only** (`dlls/mf/tests/topology.c`,
  48 lines); the source fix landed upstream as the merge base `e3b1bcb2` (Rémi Bernon) and
  RTSP's `19d7c45cad` supersedes it. Verified the merged
  `topology_branch_connect_optional_chain()` is exactly RTSP's version.
- ~~"CachyOS's `abf4e04790b` (dynamic format changes on raw parser connections) fights RTSP's
  caps logic."~~ **No.** It's in `quartz_parser.c` — the DirectShow path. VRChat uses Media
  Foundation. Left alone.
- ~~"CachyOS's `924c786e740` (prefer decoder output PTS) fights RTSP's timestamp patches."~~
  **No.** It's in `wg_transform.c::set_sample_flags_from_buffer()`; the RTSP series never
  touches that function. It's a genuine B-frame reorder fix. Kept.
- ~~"Does the RTSP build need the GStreamer `udp` plugin for RTP transport?"~~ **No.** The
  user's installed `proton-rtsp-11.0-20260609-2` ships no `libgstudp.so` — only rtsp/rtp/
  rtpmanager. It runs interleaved over TCP. Our flags match it exactly. Don't add `-Dudp`.
- ~~"Are there duplicate patches between the two trees that need dropping?"~~ **No.**
  `git cherry -v b5f2dc7b a7fcac55 e3b1bcb2` reported `+` for all 72 — zero patch-id matches.
- ~~"Does CachyOS's `winedmo`-first default bypass the RTSP path?"~~ **No, not for VRChat.**
  `use_gst_byte_stream_handler()` only gates **byte-stream / local-file** handling. URLs go by
  *scheme* handler, and `winegstreamer.rgs` claims `http:`/`https:`/`rtsp*:`. Don't set
  `PROTON_MEDIA_FORCE_GST=1` expecting it to matter for stream URLs.
- ~~"Does the libsoup hook (`eef101d3`) need a build-time dependency added?"~~ **No.** It's
  `dlopen("libsoup-3.0.so")` at runtime, and CachyOS already builds libsoup as a `GST_GOOD_DEPENDS`.
- ~~"We should ship an `x86_64_v3` build too, since the user runs CachyOS."~~ **No — decided
  against, 2026-09-02, user agreed.** Three reasons, in order of weight:
  1. `Makefile.in:104-123` appends `-mno-avx -mno-avx2 -mno-avx512f` to `x86_64_CFLAGS`
     *unconditionally and after* `-march=`, and later GCC flags win. So **AVX2 — the
     headline feature of x86-64-v3 — is disabled even in CachyOS's own v3 build.** What
     remains over `nocona` is SSE4.1/4.2, POPCNT, BMI1/2, LZCNT, MOVBE.
  2. CachyOS's v3 CI job keeps `rustflags` at `-Ctarget-cpu=nocona`, so Rust components
     gain nothing either. DXVK is separately pinned at `-O3 -mno-avx`.
  3. Proton's hot paths are the GPU driver, DXVK/vkd3d shader translation and the game —
     not this C code. Unmeasurable in VRChat, and it would halve who can run the release.
  Also note CachyOS never actually *recommended against* v3 — their release note says
  "we have a lot of different packages that might cause confusion... be conservative and
  use `x86_64`. Feel free to experiment." That's about choice-overload, not performance.
  If it's ever wanted: `CFLAGS="-O3 -march=x86-64-v3 -mtune=core-avx2"` at configure time.
- ~~"Building Proton yourself gives you CPU-optimized binaries because you're on CachyOS."~~
  **False.** The arch flags are pinned in `Makefile.in`; the host CPU and the distro's
  repo tier have no effect. You must override `CFLAGS` explicitly.
- ~~"`libtool: error: Could not determine the host path corresponding to ...` in the build log
  means the build is broken."~~ **No.** These come from vkd3d's libtool and always end with
  `Continuing, but uninstalled executables may not work.` They are benign and appear in stock
  Proton builds too. Grep for `Error [0-9]+` / `make.*Error` instead when triaging.
- ~~"Spooky's `configure.sh` container-UID fix is required."~~ **Not on this machine** — podman
  was auto-detected fine without it. Kept anyway since it's harmless and keeps us aligned
  with upstream RTSP.

## Gotchas for future me

- The `wine` submodule is intentionally **not** a normal submodule checkout. `git status`
  in `proton-cachyos` will show `wine` as modified/untracked-ish — that's expected.
  Don't run `git submodule update` on `wine`, it would blow away the merge.
- `proton-cachyos/wine` is a **partial clone** (`--filter=blob:none`) with `origin` =
  CachyOS/wine-cachyos and a `rtsp` remote = SpookySkeletons/wine. Working tree is fully
  materialized, so the build needs no network.
- The user's existing installed builds for comparison:
  `~/.steam/steam/compatibilitytools.d/{proton-rtsp-11.0-20260609-1,-2,GE-Proton11-5,GE-Proton10-33-rtsp24-1}`
- Spooky's release notes say to run `steam steam://unlockh264/` once, to unlock proper
  YouTube/Twitch functionality. That's a Steam-client-side thing, not a build flag —
  tell the user to do it after installing.
- Steam must be restarted after `make install` for the new tool to appear.
