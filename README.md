# WIDE MODE — stereo widening plugin (VST3, JUCE 8 / C++17)

Transparent stereo widener for music production and mastering.
Windows 64-bit VST3 (a Standalone app is built too, handy for testing without a DAW).

## Project status — read this first

| Part | Status |
|---|---|
| DSP core (`Source/DSP/WideModeDSP.h`) | Written, compiled and **unit-tested** (14 tests, 0 warnings, also clean under ASan/UBSan) |
| JUCE plugin layer, UI, presets | Syntax/type-checked against the real JUCE 8.0.6 headers with g++ and clang++ (Linux; one name-clash error found and fixed). **Not yet built with MSVC / not yet run in a host.** A first Windows build may still need a small fix. |
| Windows `.vst3` binary | **Not built yet.** The GitHub Actions workflow builds it and checks it is a real 64-bit DLL; it counts as built only once that job is green. |

---

## 1. Get the VST3 (pick one)

### Option A — no local tools: GitHub Actions (easiest)
1. Create a new GitHub repository and push this folder to it.
2. Open the **Actions** tab → *Build WIDE MODE (Windows VST3)* → it runs automatically (or press *Run workflow*).
3. When it finishes, download the **WIDE-MODE-VST3-win64** artifact. Inside is `WIDE MODE.vst3`.
   The same job also runs the DSP unit tests on Windows/MSVC.

### Option B — build locally on Windows 10/11
Install once:
- **Visual Studio 2022** (Community is free) with the workload *Desktop development with C++*
- **CMake ≥ 3.22** and **Git** (both on PATH)

Then, in a *x64 Native Tools / normal PowerShell* inside this folder:

```powershell
cmake -B build -G "Visual Studio 17 2022" -A x64
cmake --build build --config Release --target WideMode_VST3 --parallel
ctest --test-dir build -C Release --output-on-failure      # DSP tests
```

The first configure downloads JUCE 8.0.6 automatically (needs internet, ~1–2 min).
Result: `build\WideMode_artefacts\Release\VST3\WIDE MODE.vst3`

(Optional) Quick test without a DAW: `cmake --build build --config Release --target WideMode_Standalone`
→ `build\WideMode_artefacts\Release\Standalone\WIDE MODE.exe`

> **Licensing:** JUCE is dual-licensed (AGPLv3 / commercial). Using it to build a plugin you distribute
> publicly requires either releasing under AGPLv3 or buying a JUCE license. VST® is a Steinberg trademark;
> check Steinberg's current VST3 SDK license terms before distributing.

## 2. Install in Windows

Copy the whole **`WIDE MODE.vst3`** folder/bundle to:

```
C:\Program Files\Common Files\VST3\
```

(or run `scripts\install_windows.ps1` from an elevated PowerShell). Then rescan plugins in your DAW:

| DAW | Where |
|---|---|
| **FL Studio** | Options → Manage plugins → *Find plugins* (scan), then add from the plugin database → *Installed → Effects* |
| **Ableton Live** | Preferences → Plug-ins → enable *Use VST3 Plug-in System Folders*, press *Rescan* |
| **Reaper** | Options → Preferences → Plug-ins → VST → *Re-scan…* (clear cache and re-scan) |
| **Cubase / Nuendo** | Studio → VST Plug-in Manager → ↻ Update |
| **Studio One** | Preferences → Locations → VST Plug-ins → *Reset Blacklist* → restart |
| **Bitwig** | Settings → Locations → Plug-ins → rescan |

Put WIDE MODE as an **insert effect on a stereo track or the master bus** (last in chain, or before your limiter).
The plugin is stereo-in / stereo-out.

## 3. Using it

Signal order: `Mid/Side gain → Stereoize → Width → Bass Mono → Mix → Output → (Mono Check)`

| Control | Range | What it does |
|---|---|---|
| **WIDTH** (big knob) | 0–200 % | Scales the side signal. 0 % = true mono, 100 % = untouched, 200 % = side ×2. Double-click = reset. |
| **STEREOIZE** | 0–100 % | Adds a *decorrelated* copy of the mid (all-pass phase network) into the side channel. Adds air/size to narrow or mono-ish sources without delays, so there is no latency and no comb-filtering in mono. |
| **BASS MONO** + **CROSSOVER** | on/off, 60–300 Hz | Removes the side signal below the crossover (24 dB/oct Linkwitz-Riley), keeping lows centered. Cross-faded on toggle, no clicks. |
| **MID / SIDE** | −24…+12 dB | Independent level of the centre and the stereo-difference content. |
| **MIX** | 0–100 % | Dry/wet. Dry path is latency-free, so there is no phase mismatch at partial mix. |
| **OUTPUT** | −18…+18 dB | Output gain (widening can raise level slightly; trim here). |
| **MONO CHECK** | on/off | Folds the output to mono instantly (smoothed) so you can verify the mix still holds up. |
| **Correlation meter** | −1…+1 | Output correlation (bar) with the input marked by ▲. Near +1 = mono, 0 = wide, **below 0 = phase problems**. |
| **Input / Output meters** | −60…+6 dBFS | Peak meters with hold; click a meter to reset its max-peak readout. |
| **Presets** | | ◀ ▶ step, dropdown to pick, **SAVE** stores the current settings, **FOLDER** opens `%APPDATA%\WideMode\Presets` (user presets are plain `.wmpreset` XML files — delete or share them freely). The box shows *Custom* when settings differ from any preset. Mono Check is not stored in presets. |

Tips: start at Width 110–130 % with Bass Mono at 120 Hz. For very mono sources use Stereoize 20–40 % instead of Width.
Always flip **Mono Check** at the end. The mono sum is mathematically unchanged by Width/Stereoize/Bass Mono.

## 4. Verified DSP behaviour (from `Tests/DspTests.cpp`, run it with `ctest` or `g++ -O2 -std=c++17 Tests/DspTests.cpp`)

| Property | Result |
|---|---|
| Default settings | Bit-exact bypass (0 error) at 44.1–192 kHz |
| Width 0 % | Exact mono (L = R = (L+R)/2) |
| Width 200 % | Side ×2, mid unchanged (error < 1e-7) |
| **Mono compatibility** | Mono sum (L+R)/2 equals the input's to **−143 dB** with width 200 %, stereoize 100 %, bass mono and +6 dB side all on, at every sample rate tested → widening cannot cause mono phase cancellation |
| Bass Mono response | Matches the ideal LR4 high-pass to < 0.01 dB (−6.02 dB at the crossover, −24.6 dB at ½ fc, −46 dB at 40 Hz for a 150 Hz crossover); centred bass untouched |
| Stereoize | Decorrelates mono source to r ≈ 0.67; mid magnitude unchanged at all frequencies; average level +0.8 dB, L/R tilt < 1 dB |
| Clicks | Worst sample-to-sample step across a simultaneous change of *every* parameter: 0.065, i.e. no larger than the test sine's own slope (no discontinuity) |
| Correlation meter | +1.00 mono, −1.00 anti-phase, ≈ 0 independent noise |
| Block-size independence | Bit-exact for blocks of 1, 7, 64, 480, 4096 |
| Robustness | NaN/Inf input is sanitised; stable under abusive crossover modulation |
| CPU | ≈ 0.3 % of one core @ 48 kHz (≈ 370× realtime), zero latency, no allocations in the audio thread |

## 5. Source layout

```
CMakeLists.txt                  JUCE via FetchContent, VST3 + Standalone, static MSVC runtime
Source/DSP/WideModeDSP.h        All DSP (pure C++, no JUCE) + metering
Source/PluginProcessor.*        Parameters (APVTS), state save/load, audio callback
Source/PluginEditor.*           Editor layout, preset bar, meter polling (60 Hz)
Source/UI/WideModeUI.h          Knob, segmented meters, correlation meter, toggle, look & feel
Source/PresetManager.*          Factory + user presets (.wmpreset XML)
Tests/DspTests.cpp              14 DSP tests (no JUCE needed)
.github/workflows/              Windows build + tests in CI
scripts/install_windows.ps1     Copies the VST3 to Common Files\VST3
```

## 6. Troubleshooting

- **DAW doesn't see the plugin** → the `.vst3` must be in `C:\Program Files\Common Files\VST3\` (the *whole* `WIDE MODE.vst3` folder), then rescan/clear the blacklist.
- **CMake can't download JUCE** → check internet/proxy, or clone JUCE yourself and configure with `-DFETCHCONTENT_SOURCE_DIR_JUCE=C:\path\to\JUCE`.
- **Compile error on first build** → please send the first error line (the plugin layer has not yet been through MSVC).
