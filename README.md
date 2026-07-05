# F1TV UHD Patcher

Automated pipeline that patches the F1TV Android TV app to enable UHD/4K playback on Android TV devices. It now ships separate profiles for NVIDIA Shield-quality output, smoother generic Android TV playback, and a safer reduced target for TVs that cannot sustain full 4K.

## How it works

1. **Checks** APKPure every 3 hours for new F1TV Android TV releases
2. **Downloads** the app bundle — Google Play primary (arm64 native profile), APKPure and APKMirror as fallbacks
3. **Patches** the smali with a selected profile (Shield quality, Android TV smooth, or Android TV safe) — see [Patch profiles](#patch-profiles)
4. **Signs** all APKs with a consistent keystore
5. **Publishes** the patched bundle as a GitHub Release
6. **Notifies** via Pushover when a new patch is ready (or if it fails)

> **How 4K gets unlocked (and why stock skips the Shield).** F1TV picks a stream manifest from
> the `x-f1-device-info` header. The Android app sends `device=android_tv`, which returns the
> Widevine `HDR-UHD-CMAF-WV` manifest where **2160p exists only as HDR; SDR is hard-capped at
> 1620p server-side**. ClearVR only requests the 2160p tier if the content's HDR type is in
> *both* what the EGL renderer supports *and* what the display reports
> (`Display.getHdrCapabilities()`). F1's UHD is **HLG**; a panel that reports only HDR10 (like
> most setups behind an NVIDIA Shield) makes ClearVR reject HLG upstream and fall back to
> 1620p SDR. The pipeline forces the EGL HDR advertise **and** spoofs the display-capability
> check, so the core serves the 2160p tiles; the device's video pipeline then plays them as
> HDR10 or a clean 4K SDR downconvert (the same path YouTube HLG uses). **Needs the arm64 build**
> — see [Verify 4K is working](#verify-4k-is-working). Genuinely HLG-capable devices get full HDR;
> HDR10-only panels get accurate 4K. An opt-in `F1TV_PQ_REROUTE` can push some setups to true
> HDR10 output — see [Build options](#build-options).

## Installing on your Android TV

### Prerequisites: Enable Developer Options & ADB

1. On your Android TV, go to **Settings > Device Preferences > About**
2. Scroll to **Build** and click it **7 times** to enable Developer Options
3. Go back to **Settings > Device Preferences > Developer Options**
4. Enable **USB debugging** (and **ADB over network** if installing wirelessly)
5. Note the **IP address** shown under Settings > Network & Internet, or Device Preferences > About > Status

### Option 1: ADB from a computer (recommended and tested)

Install ADB on your computer ([download platform-tools](https://developer.android.com/tools/releases/platform-tools)) and add it to your PATH.

**Via USB:**
```bash
# Connect your Android TV via USB cable, then:
adb devices  # Confirm it shows up — approve the prompt on your TV
```

**Via WiFi:**
```bash
adb connect 192.168.1.100:5555  # Replace with your TV's IP
# Approve the connection prompt on your TV
```

Then download the APKM for your profile from the release page and install it. For most non-Shield Android TVs, start with `f1tv-uhd-smooth-tv-patched.apkm`; if full 4K still advances frame-by-frame, use `f1tv-uhd-android-tv-safe-patched.apkm`.

```bash
# Uninstall the original F1TV first (required — different signing key)
adb uninstall com.formulaone.production

# Use the install script (auto-extracts and auto-detects device config)
./scripts/install.sh f1tv-uhd-smooth-tv-patched.apkm
# With ADB over WiFi:
./scripts/install.sh f1tv-uhd-smooth-tv-patched.apkm 192.168.1.100:5555
```

Or install manually:

```bash
# Unzip the bundle
mkdir f1tv && cd f1tv && unzip ../f1tv-uhd-patched.apkm

# Install (most Android TVs — NVIDIA Shield, Chromecast, etc. — are arm64).
# Use the arm64_v8a split; the 32-bit armeabi_v7a native libs can't sustain 4K.
adb install-multiple base.apk \
  config.arm64_v8a.apk \
  config.en.apk \
  config.xhdpi.apk
```

> **Install the `arm64_v8a` split, not `armeabi_v7a`.** The ClearVR decoder/renderer ships per
> ABI. On an arm64 device (NVIDIA Shield included) the 32-bit split forces ClearVR to run in
> 32-bit, which commonly can't allocate the 4K secure HEVC decoder — you get `TM4014` /
> "acquireVdecResource not enough" errors. `install.sh` picks arm64 automatically when it's in
> the bundle; a bundle with only `armeabi_v7a` (e.g. the APKPure fallback source) can't do
> reliable 4K on Android TV. `install.sh` now refuses a 32-bit fallback on arm64 devices unless
> you explicitly set `F1TV_ALLOW_32BIT=1`.

The install script accepts `.apkm`, `.xapk` files, or a directory of extracted APKs.

### Option 2: Send & install directly on the TV (not tested yet)

No computer needed after the initial download.

1. Download the APKM for your selected profile from the release page on your phone
2. Install [Split APKs Installer (SAI)](https://play.google.com/store/apps/details?id=com.mtv.sai&hl=en) on your Android TV (available on Play Store)
3. Transfer the `.apkm` file to your TV via:
   - **USB drive** — copy the file to a USB stick, plug it into the TV
   - **Send Files to TV** — install [this app](https://play.google.com/store/apps/details?id=com.yablio.sendfilestotv) on both your phone and TV, send the file over WiFi
   - **Google Drive / cloud** — upload to Drive, open it from the TV's file manager
4. Open SAI on the TV, select the `.apkm` file, and install

### Option 3: Wireless ADB apps (not tested yet)

If you don't have a computer but want a one-tap solution:

1. Install [Bugjaeger](https://play.google.com/store/apps/details?id=eu.sisik.hackendebug) on your Android phone
2. Enable ADB over network on your TV (see prerequisites above)
3. Connect Bugjaeger to your TV via its IP address
4. Use Bugjaeger to install the individual APK files

### Common split APKs

The bundle contains split APKs for different device configurations. You need `base.apk` plus the correct splits. Split names vary by source (`config.*` from APKPure, `split_config.*` from APKMirror, `com.formulaone.production.config.*` from Google Play):

| Split (any prefix) | When to include |
|---|---|
| `*.arm64_v8a.apk` | Most modern Android TVs (NVIDIA Shield, Chromecast, etc.) |
| `*.armeabi_v7a.apk` | Older 32-bit devices |
| `*.x86.apk` | Some emulators |
| `*.en.apk` | English — replace `en` with your language code |
| `*.xhdpi.apk` | Standard TV density — almost always needed |

> **Note:** You must uninstall the original F1TV app before installing the patched version (different signing key). This means you'll need to log in again.

## Setup your own pipeline

### 1. Fork and enable Actions

Fork this repo and enable GitHub Actions in the Actions tab.

### 2. Custom apkeep fork

The pipeline uses a custom build of [apkeep](https://github.com/EFForg/apkeep) with Android TV device profiles added to [rs-google-play](https://github.com/EFForg/rs-google-play). This allows downloading native split APKs directly from Google Play. The workflow currently exposes the profiles known by the bundled custom apkeep release: `nvidia_shield_tv` and `generic_armv7_tv`. The repo also includes `generic_arm64_tv.properties` as a template for a future custom apkeep build.

To set up your own:

1. Fork [EFForg/rs-google-play](https://github.com/EFForg/rs-google-play), add your device profile to `gpapi/device.properties`, delete `gpapi/src/device_properties.bin`, commit & push
2. Fork [EFForg/apkeep](https://github.com/EFForg/apkeep), change `Cargo.toml` to point `gpapi` at your rs-google-play fork, commit & push
3. Tag a release (`git tag v0.18.0-shield && git push origin v0.18.0-shield`) — the included workflow builds the binary automatically
4. Update `APKEEP_CUSTOM_TAG` in `patch.yml` to match your tag

A device profile dump script is included at `scripts/dump_device_props.sh` — connect your Android TV via ADB and run it to generate the profile.

### 3. Secrets

In **Settings > Secrets > Actions**, add:

| Secret | Purpose | Required |
|---|---|---|
| `GOOGLE_EMAIL` | Google account email for Play Store downloads | For Google Play |
| `GOOGLE_AAS_TOKEN` | Google AAS token ([how to obtain](https://github.com/EFForg/apkeep/blob/master/USAGE-google-play.md)) | For Google Play |
| `KEYSTORE_B64` | Base64-encoded signing keystore (persistent key across builds) | Recommended |
| `KEYSTORE_PASS` | Keystore password | Recommended |
| `KEYSTORE_ALIAS` | Key alias | Recommended |
| `PUSHOVER_APP_TOKEN` | Pushover app token for notifications | Optional |
| `PUSHOVER_USER_KEY` | Pushover user key for notifications | Optional |

Without Google Play credentials, the pipeline falls back to APKPure (armeabi-v7a only) and APKMirror automatically.

**Generate a persistent keystore:**

```bash
keytool -genkeypair -keystore f1tv.keystore -storepass yourpass \
  -keypass yourpass -alias f1tvpatch -keyalg RSA -keysize 2048 \
  -validity 10000 -dname "CN=F1TV UHD Patch"

# Encode and add as KEYSTORE_B64 secret
base64 -w0 f1tv.keystore
```

Without a persistent keystore, a new key is generated each build — you'll need to uninstall before each update.

### 4. Manual trigger

If the automatic download fails, you can trigger the workflow manually:

- **With direct URL**: Go to Actions > F1TV UHD Patch > Run workflow, paste an `.apkm` URL
- **Force rebuild**: Check the "Force rebuild" option to re-patch an existing version
- **Patch profile**: choose `both-android-tv` to build both Android TV variants, `android-tv-smooth` for full 4K direct rendering, `android-tv-safe` if full 4K still stalls, or `shield-quality` for NVIDIA Shield/strong GPUs.
- **Download profile**: choose the Google Play device profile. Use `nvidia_shield_tv` for arm64 bundles with the bundled custom apkeep release.

## Project structure

```
.github/workflows/patch.yml   # CI pipeline (check, download, patch, release)
scripts/
  check_version.py             # APKMirror RSS parser (fallback version check)
  download_apkm.py             # Playwright-based APKMirror downloader (fallback)
  patch.sh                     # Smali patching, signing, bundling
  install.sh                   # ADB install helper (accepts .apkm, .xapk, or directory)
  dump_device_props.sh         # Dump Android TV device profile for rs-google-play
device_profiles/
  nvidia_shield_tv.properties  # NVIDIA Shield TV profile for Google Play downloads
  generic_arm64_tv.properties  # Generic 64-bit Android TV profile template
  generic_armv7_tv.properties  # Generic 32-bit Android TV profile template
```

## Requirements (local use)

Only needed if running scripts locally outside CI:

- Python 3.10+, Playwright (`pip install playwright && playwright install chromium`)
- Java, apktool, zipalign, apksigner
- ADB (for install.sh)

## Verify 4K is working

After installing (on an **arm64** bundle, connected to an **HDR-capable TV**), start a session and
watch the live decoder stats over ADB:

```bash
./scripts/stream_stats.sh 192.168.1.100:5555   # your TV's IP, or omit if on USB
```

You want to see, once the stream ramps up:

- **Decoder Resolution: `3840x2160`** — full 4K. If it plateaus at `2880x1620`, the 2160p tier isn't
  being served (see the checklist below).
- **Video Codec: HEVC** and a healthy bitrate (≈15–25 Mbps).
- On a genuinely HDR-capable device you'll also get **HDR10/HLG** output; on an HDR10-only panel
  (e.g. many NVIDIA Shield setups) the 4K plays as a clean **SDR downconvert** — full resolution,
  accurate colours, just not HDR's extra brightness/gamut. See [Build options](#build-options) to
  push those setups toward true HDR10.

The patched build also exposes the in-app **quality selector** so you can confirm `3840×2160` is
offered (debug overlays are left off in release builds).

If you're stuck at 1620p, check in order:

1. **ABI** — `adb shell getprop ro.product.cpu.abi` should be `arm64-v8a`, and you must have installed
   the `config.arm64_v8a.apk` split (not `armeabi_v7a`). A 32-bit install can't do 4K.
2. **Build flags** — the bundle must be built with `F1TV_HLG_BYPASS` **and** `F1TV_DISPLAY_HDR_SPOOF`
   on (both default; CI sets them). Public releases from before these defaults do **not** have them.
3. **4K TV** — the panel must actually be 4K (the decode target follows the display). An HDR-capable
   panel additionally gets HDR output.

## Applied patches

All patches are applied to every device (no runtime device gating) unless noted. The two that
actually unlock 2160p are **HDR advertise** and **display-capability spoof**.

| Patch | File · method | What it does |
|---|---|---|
| UHD / device unlock | `DeviceSupportImpl` · `validateIsUhdSupportedDevice`, `validateTmSdkSupport`, `validateLowRamDeviceSupport`, `validateApiLevelSupport` | Each returns `Pair(true, null)`, so the device passes every UHD-capability gate (brand/product whitelist, secure-decoder probe, low-RAM check, API-level check). |
| **Display-capability spoof** *(2160p unlock)* | `DeviceParameters` · `doesDisplaySupport` | Returns `true` so the core believes the panel accepts F1's HLG and serves the **2160p tiles** instead of rejecting HLG upstream and dropping to 1620p SDR. Default on; disable with `F1TV_DISPLAY_HDR_SPOOF=0`. |
| **HDR advertise** *(2160p unlock)* | `EGLRenderTarget` · `getIsBt2020HlgExtensionSupported` | Returns `true`, so `DeviceParameters` reports PQ+HLG EGL support and the backend offers the HDR 2160p tier. Default on; disable with `F1TV_HLG_BYPASS=0`. |
| Quality selector | `DiagnosticsPreferenceManagerImpl` · `isVideoQualityEnabled` | Returns `true` so the in-app quality picker is visible. (The debug overlays are intentionally left off.) |
| Display size target | `TrueTVDisplaySizeHelper` · `getDefaultDisplaySize` | Reports the selected profile target via `getTrueDisplaySizeIfTV`, lifting or tuning ClearVR's ~1.5× display-size cap. Shield/smooth report `3840×2160`; safe reports `1920×1080`, which caps the effective stream target around `2880×1620`. |
| PQ colour reroute *(correct colours)* | `RenderTargetConfig` · `requireHLG`, `require2020PQ` | Routes F1's HLG content through the EGL **PQ** colorspace the device supports (`requireHLG→false`, `require2020PQ→PQ‖HLG`) so the 4K tiles are correctly gamut-converted instead of shown washed-out. Default on; disable with `F1TV_PQ_REROUTE=0`. |
| Render path | `RenderAPIConfig` · `getNRPTextureBlitMode` | **Default: EGL/GL path** (patch skipped) so ClearVR composites and does a correct BT.2020→Rec.709 conversion. Set `F1TV_DIRECT_TO_VIEW=1` to force `NATIVE_ANDROID_DIRECT_TO_VIEW` (decoder→SurfaceView) on weak/Amlogic GPUs that drop frames — at the cost of washed-out HDR colours. |
| Decoder capability spoof | `DecoderCapability` · `getAsCoreProtobuf` | Reports profile-selected secure tile slots/rows/cols. Shield/smooth use `16/5/5`; safe uses `8/4/4`. |
| NVIDIA workaround off | `Quirks` · `deviceNeedsNoPostProcessWorkaround` | Returns `false` only when the profile enables `F1TV_DISABLE_NVIDIA_QUIRK=1`. Android TV profiles leave it untouched. |
| Device-model spoof | `TvApplication` · `getRequestHeader` | Sends a configurable `model` in the `x-f1-device-info` header. Default is `Chromecast`; disable with `F1TV_MODEL_SPOOF=0`. |
| Version tag | `apktool.yml`, `BuildConfig` | Appends the profile suffix, e.g. `-UHD`, `-UHD-SMOOTH`, or `-UHD-SAFE`. |

## Patch profiles

| Profile | Output | Render path | Display target | Decoder capability | Best for |
|---|---|---|---|---|---|
| `shield-quality` | `f1tv-uhd-patched.apkm` | EGL/GL + PQ reroute | `3840×2160` | `16/5/5` | NVIDIA Shield or stronger GPUs where colour correctness matters most. |
| `android-tv-smooth` | `f1tv-uhd-smooth-tv-patched.apkm` | Direct-to-view | `3840×2160` | `16/5/5` | TVs that can decode full 4K but drop frames on the EGL/GL path. |
| `android-tv-safe` | `f1tv-uhd-android-tv-safe-patched.apkm` | Direct-to-view | reports `1920×1080` for an effective cap around `2880×1620` | `8/4/4` | TVs that show 2160p but barely advance frames or run out of decoder resources. |

Recommended order for generic Android TV: try `android-tv-smooth` first, then `android-tv-safe` if playback still stalls.

## Build options

`patch.sh` reads these environment variables. `F1TV_SMOOTH_TV=1` is kept as a backward-compatible local alias for `F1TV_PATCH_PROFILE=android-tv-smooth`; the GitHub workflow uses `patch_profile`.

| Variable | Default | Effect |
|---|---|---|
| `F1TV_PATCH_PROFILE` | `shield-quality` | Selects `shield-quality`, `android-tv-smooth`, or `android-tv-safe`. The workflow also offers `both-android-tv`, which builds smooth and safe releases from the same source APKM. |
| `F1TV_HLG_BYPASS` | `1` | Advertise EGL HDR (PQ+HLG) so the backend offers the 2160p tier. Required for 4K. |
| `F1TV_DISPLAY_HDR_SPOOF` | `1` | Force the display-capability check to accept all HDR types so the core serves 2160p on HDR10-only panels. Required for 4K on the Shield. |
| `F1TV_DIRECT_TO_VIEW` | profile-based | Forces decoder-to-SurfaceView rendering when `1`. Android TV profiles default to `1`. |
| `F1TV_PQ_REROUTE` | profile-based | Enables EGL PQ colour reroute when `1`. Shield-quality defaults to `1`; Android TV profiles default to `0`. |
| `F1TV_DISABLE_NVIDIA_QUIRK` | profile-based | Disables the NVIDIA no-post-process workaround only for profiles that need it. |
| `F1TV_MODEL_SPOOF` | `Chromecast` | Device model sent in the F1 request header. Set `0` to keep the real model. |
| `F1TV_DECODER_TILE_SLOTS` | profile-based | Secure tile slot count reported to ClearVR. |
| `F1TV_DECODER_TILE_ROWS` / `F1TV_DECODER_TILE_COLUMNS` | profile-based | Tile grid dimensions reported to ClearVR. |
| `F1TV_DISPLAY_WIDTH` / `F1TV_DISPLAY_HEIGHT` | profile-based | Display target reported to ClearVR. |
| `F1TV_OUTPUT_BASENAME` | profile-based | Output APKM filename. |

> **Result on an NVIDIA Shield + HDR10 TV:** true **3840×2160** with accurate colours, output as a
> clean SDR downconvert (the Shield's GPU lacks the HLG EGL colorspace, so full HDR10 to the panel
> isn't reliable — but the 4K resolution and colour accuracy are the wins).

For TVs that show `3840×2160` but barely advance frames, build the smooth TV profile:

```bash
F1TV_PATCH_PROFILE=android-tv-smooth ./scripts/patch.sh f1tv-source.apkm output/
./scripts/install.sh output/f1tv-uhd-smooth-tv-patched.apkm [device-ip:5555]
```

If it still stalls, build the safer reduced target:

```bash
F1TV_PATCH_PROFILE=android-tv-safe ./scripts/patch.sh f1tv-source.apkm output/
./scripts/install.sh output/f1tv-uhd-android-tv-safe-patched.apkm [device-ip:5555]
```

In GitHub Actions, run the workflow manually and choose the **patch_profile**. Release tags get a
profile suffix such as `-smooth-tv` or `-android-tv-safe` so variants do not overwrite each other.

## License

For personal/educational use only.
