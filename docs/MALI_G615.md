# Mali-G615 Graphics Support in ARMSX3

Target: Android ARM64 device with an ARM Mali-G615 GPU (e.g. MediaTek Dimensity 8350).
Status: system-driver path with Mali-specific hardening and diagnostics. No bundled
Mali driver exists in this project and none is required.

> **Hardware honesty:** ADB/device access was NOT available while this was written.
> Everything marked **UNVERIFIED UNTIL HARDWARE TESTING** below must be confirmed on
> the physical device before it is treated as fact.

## 1. What "Mali support" actually means here

 investigated the Winlator Mali/Lodashi tree (`WinlatorMali/`, single squashed
commit `fbf42d2`) for anything resembling a Mali GPU driver to port. Finding: there
is none, and the architecture shows why none is needed.

| Layer | Winlator Mali path | ARMSX3 path |
|---|---|---|
| Kernel GPU driver | Stock Android (`mali_kbase`, `/dev/mali0`) — untouched | Same (untouched, out of scope) |
| Userspace Vulkan driver | Stock Android Mali blob (`/system/lib64/libvulkan.so`) | Same, via `vk_android_loader` |
| Extra ICD/layer | `wrapper` Vulkan layer from `graphics_driver/wrapper.tzst`, forced via `VK_ICD_FILENAMES` (`XServerDisplayActivity.java:2215-2379`) | None — native `libvulkan.so` directly |
| GL-on-VK / translators | Gladio (`gladio-1.1.tzst`), DXVK/VKD3D/WineD3D, VirGL/Zink present | None — RSX is implemented natively in Vulkan (`RSX/VK/`) with a GLES fallback (`RSX/GL/`) |
| Mali-specific code | Tuning only: DXVK `relaxedBarriers/ignoreGraphicsBarriers/useEarlyDiscard/maxQueuedFrames=1` for non-Adreno (`DXVKConfigDialog.java:283-348`), `BOX64_MMAP32=0` on Mali renderer strings, `GLADIO_NO_ERROR=1`, Apex GLES post-FX presets | Tiler-aware renderer behaviour + workarounds (see §2) |
| Presentation | `ANativeWindow`/`AHardwareBuffer` → EGL surface (`winlator/renderer/egl.cpp`) | `ANativeWindow` → `vkCreateAndroidSurfaceKHR` (`vkutils/swapchain_android.hpp:22-26`) |

Data flow on device (both projects, same shape at the bottom):

```
app → Vulkan API → system loader/ICD → Mali userspace blob → Android GPU stack
    → kernel Mali driver → Mali-G615 hardware
```

**Porting consequence:** there is no Winlator Mali driver code to copy
(no Panfrost/PanVK/Mesa in the tree — zero hits; no `mali0`/`mali_kbase` hits;
`dmabuf` appears only in a generic helper). The Winlator pieces that transfer are
*behavioural*: tile-aware barrier/render-pass discipline, single queued frame,
`AHardwareBuffer` zero-copy presentation, and explicit GPU/driver logging. ARMSX3
already implements the equivalents natively (see §2). What was added here is
diagnostics + documentation, not a driver.

Intentionally NOT ported: `wrapper` ICD, Gladio, DXVK/VKD3D/Wine/Proot/Box64
chain, Turnip/AdrenoTools switching (Adreno-only), BCN/leegao layers, `WINE_*`
registry hacks, Apex frame-gen overlay. None of these have a role in a native
Vulkan PS3 emulator; the custom-driver loader path stays Adreno-gated.

## 2. Mali handling already present in ARMSX3 (prior to this change)

- Vendor detection: `ARM_MALI` / `PANVK` via driver ID or device name, `is_MALI()`
  / `is_MOBILE()` helpers (`vkutils/chip_class.h:104-116`, `device.cpp:578-585,637-640`).
- `r44p1` feedback-loop disable: that Mali-G615 blob faults inside the in-tile
  feedback-loop blend path with no `DEVICE_LOST`, so the feature is disabled by
  driver build, never by vendor (`device.cpp:189-208`).
- Mobile tiler path in `VKGSRender` ctor: async compute and passthrough DMA
  disabled on all mobile tilers incl. Mali (`VKGSRender.cpp:820-848`).
- fp16 safe path: native float16 disabled on mobile unless proven OK
  (Turnip / new Adreno); Mali keeps fp32 emulation (`device.cpp:448-469`).
- `forceMaliFbFetch` setting for MediaTek Mali framebuffer-fetch
  (`android/.../config/Settings.kt:555-566`).
- FSR Mali format fix (`upscalers/fsr1/fsr_pass.cpp:243-264,348`).
- ANGLE GLES fallback for weak native Mali GL drivers; Mali/Xclipse/PowerVR run
  the system driver, never a custom pack (`GpuInfo.kt`, `DriverManagerSection.kt`).
- Unified-memory budget (`VK_EXT_memory_budget` + `/proc/meminfo` clamp,
  `device.cpp:736-769`, `memory.cpp:201-225`); shader-cache `-nofbl` suffix so a
  cache written with feedback loops is never replayed without them
  (`VKGSRender.cpp:671-677`).
- Driver-identity + framegen-requirement logging
  (`device.cpp:360-396`, `device.cpp:239-243`).

## 3. What this change adds

1. `rpcs3/Emu/RSX/VK/vkutils/device.cpp` — Mali startup summary in
   `physical_device::create()`: one `rsx_log.notice` reporting framebuffer-loop,
   float16, memory-budget, sync2, unsized-array and descriptor-UAB state, plus an
   `rsx_log.error` if a custom (Adreno-only) driver was requested but Mali
   answered — i.e. a silent fallback to the system driver. Log-only; zero
   behaviour change.
2. This document.

## 4. Vulkan extension audit (from `device.cpp` device-create path)

"Required by ARMSX3" = requested at `vkCreateDevice` when probed present.
"Exposed by Mali G615" = **UNVERIFIED UNTIL HARDWARE TESTING** (no ADB yet);
typical G615 blobs expose 1.3 + most below, but per-device blobs vary.

| Extension | Required? | Notes / fallback |
|---|---|---|
| `VK_KHR_swapchain` | Yes (always) | No fallback; fatal without it |
| `VK_EXT_shader_uniform_buffer_unsized_array` | Yes, when probed | All game shaders declare it; without it pipelines fail (`device.cpp:1159-1181`) |
| `VK_EXT_custom_border_color` | When probed | Degraded sampling without it |
| `VK_EXT_multi_draw` | When probed | Disabled if `maxMultiDrawCount==0` |
| `VK_EXT_conditional_rendering` | When probed (NOT disabled on Mali; Adreno/Turnip only) | CPU-side fallback `thread::begin_conditional_rendering` |
| `VK_EXT_depth_range_unrestricted` | When probed | Clamped path otherwise |
| `VK_EXT_robustness2` (`nullDescriptor`) | When probed | Framegen shaders need it; framegen unavailable without it |
| `VK_EXT_external_memory_host` | When probed | Passthrough DMA off (always off on mobile tilers anyway) |
| `VK_EXT_memory_budget` | When probed | **Strongly recommended on Mali**; without it eviction judges heap size, not real headroom (`memory.cpp:201-225`) |
| `VK_EXT_shader_stencil_export` | When probed | Fallback path in shader gen |
| `VK_EXT_attachment_feedback_loop_layout` | When probed, except `r44p1` blob | Per-primitive barrier fallback; no measurable slowdown per prior report |
| `VK_KHR_fragment_shader_barycentric` | When probed | Flat-shading fallback |
| `VK_KHR_synchronization2` | When probed (1.3 feature as ext, since instance caps at 1.2) | Legacy barrier path |
| `VK_EXT_device_fault` | When probed | Diagnostics only |
| `VK_EXT_extended_dynamic_state` | When probed | Affects shader-cache version suffix |
| `VK_EXT_provoking_vertex` | When probed | Smooth-interpolation fallback with warning |
| `VK_ANDROID_external_memory_android_hardware_buffer` | **Deliberately NOT requested** (probe + log only) | Former framegen sharing path removed; requesting it at apiVersion 1.2 trips VUID `-01387` (`device.cpp:897-911`) |
| Instance `VK_KHR_android_surface` | Yes | Fatal without it; system loader provides it |

Core features requested unconditionally (`device.cpp:971-979`): `robustBufferAccess`,
`fullDrawIndexUint32`, `independentBlend`, `logicOp`, `depthClamp`, `depthBounds`,
`wideLines`, `largePoints`, `shaderFloat64`, plus MSAA/occlusion/clip-distance/aniso
under guards — each degrades with an `rsx_log.error` if the driver lacks it.
Whether Mali-G615 blobs expose all of these is **UNVERIFIED UNTIL HARDWARE TESTING**;
the errors above are the tripwires that will say so on device.

## 5. Build instructions (clean checkout)

```sh
git clone https://github.com/<you>/ARMSX3.git && cd ARMSX3
git checkout mali-g615-port
# Android (arm64-v8a, NDK 29, minSdk 33): see android/configure.sh and
# android/armsx3-ui/app/build.gradle.kts. The Vulkan loader is header-only NDK
# + system libvulkan.so at runtime — no ICD/.so packaging step for Mali.
```

No hidden machine paths; no proprietary binaries are bundled or required for Mali
(system driver is used). Custom-driver packs remain Adreno-only.

## 6. Hardware validation (once ADB is available)

```sh
adb shell getprop ro.build.version.release
adb shell uname -r
adb shell getprop ro.soc.model            # expect Dimensity-class SoC
adb shell dumpsys gpu | grep -i -m5 mali
adb shell ls -l /dev/mali0
adb shell cmd gpu help 2>/dev/null | head
adb shell dumpsys SurfaceFlinger | grep -i -m5 -e vulkan -e mali
# Vulkan caps on device (via ARMSX3 log after installing this build):
adb logcat -c; adb logcat | grep -i -e rsx -e vk_loader -e Mali -e "driver identity"
```

Acceptance ladder: (1) Mali GPU enumerated + system loader resolves;
(2) instance/device/queue/command submission;
(3) clear → triangle → texture → framebuffer → shader;
(4) ARMSX3 renderer init → game boot → frame present → surface recreation;
(5) sustained workload, no GPU hang/reset/leak;
(6) real PS3 title renders 3D (menu alone is NOT success).
Compare against the Winlator NFS:MW 2012 mid-30s-FPS reference only as proof the
silicon can do real work — never as an ARMSX3 FPS promise.

## 7. Test report

Verified without hardware: full-tree `grep` architecture audit of WinlatorMali
(system-driver + wrapper model, no portable driver code) and ARMSX3 (native
Vulkan + existing Mali workarounds); `git diff --check` clean; changed hunk
re-read against `device.h` accessors; no new build inputs (header-only log
change + docs). No host compiler run claims device behaviour.

Verified on Mali-G615 hardware: **nothing yet — UNVERIFIED UNTIL HARDWARE
TESTING.** Do not treat APK assembly as acceptance.
