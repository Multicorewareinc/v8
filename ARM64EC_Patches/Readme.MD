# Building V8 for ARM64EC on Windows

This repository contains patch files to build Google's V8 JavaScript engine as an **ARM64EC** binary on Windows.

ARM64EC ("Emulation Compatible") is a Windows ABI that allows ARM64 native code to interoperate seamlessly with x64 emulated code in the same process. This build enables V8 to run natively on ARM64 Windows devices (Surface Pro X, Snapdragon-based PCs) while remaining compatible with x64 host processes.

---

## Patch Files

| File | Description |
|------|-------------|
| `0001-ARM64EC-build-changes.patch` | GN build system changes — new `v8_heap_base_stack` source set, ARM64EC toolchain flags, linker machine type |
| `0001-ARM64EC-support-changes.patch` | Core ARM64EC support — architecture detection macros, virtual memory allocation with EC code flag, stack unwinding, sampler, assembler FFS sequence skip |
| `0001-ARM64EC-ICU-changes.patch` | ICU internationalization library ARM64EC compatibility fixes |
| `0001-ARM64EC-libc-changes.patch` | libc++ threading support for `_M_ARM64EC` mutex size |
| `0001-ARM64EC-protobuf-changes.patch` | Protobuf architecture detection fixes for ARM64EC |
| `0001-ARM64EC-cycleclock-changes.patch` | Google Benchmark cycle clock ARM64EC support |

---

## Prerequisites

- **OS:** Windows 10/11 (ARM64 or x64)
- **Visual Studio 2022** with the following workloads:
  - Desktop development with C++
  - MSVC v143 ARM64/ARM64EC build tools
- **Windows SDK 10.0.26100.0** (Windows 11 SDK)
  - Install via Visual Studio Installer → Individual Components → search "Windows 11 SDK (10.0.26100)"
- **Git** for Windows
- **Python 3** (installed automatically by depot_tools)
- At least **20 GB free disk space**

---

## Build Instructions

### Step 1 — Set up depot_tools

depot_tools is Google's build toolchain manager required to fetch V8 and its dependencies.

```cmd
cd C:\workspace
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
```

Add depot_tools to your system `PATH` permanently via System Properties → Environment Variables, or for the current session:

```cmd
set PATH=C:\workspace\depot_tools;%PATH%
set DEPOT_TOOLS_WIN_TOOLCHAIN=0
```

> `DEPOT_TOOLS_WIN_TOOLCHAIN=0` tells depot_tools to use your locally installed Visual Studio instead of trying to download Google's internal toolchain.

### Step 2 — Fetch V8 source and dependencies

Create a directory for the V8 source and run the fetch command. This downloads approximately 1.5 GB of source and additional dependencies.

```cmd
mkdir C:\workspace\v8_build
cd C:\workspace\v8_build
fetch v8
```

### Step 3 — Checkout the required version

This build targets V8 **10.4.132.24**. Check out that exact commit and sync all dependencies to match.

```cmd
cd C:\workspace\v8_build\v8
git checkout 10.4.132.24
cd ..
set DEPOT_TOOLS_WIN_TOOLCHAIN=0
gclient sync --with_branch_heads
```

> `gclient sync` updates all third-party dependencies (ICU, zlib, etc.) to the versions required by this V8 commit.

### Step 4 — Apply the ARM64EC patches

Copy the `ARM64EC_Patches` directory into the V8 root:

C:\workspace\v8_build\v8\ARM64EC_Patches\

### 1. V8 core support changes
```cmd
cd C:\workspace\v8_build\v8
git am ARM64EC_Patches\0001-ARM64EC-support-changes.patch
```

### 2. Build system changes
```cmd
cd C:\workspace\v8_build\v8\build
git am ..\ARM64EC_Patches\0001-ARM64EC-build-changes.patch
```

### 3. Google Benchmark cycle clock changes
```cmd
cd C:\workspace\v8_build\v8\third_party\google_benchmark\src
git am ..\..\..\ARM64EC_Patches\0001-ARM64EC-cycleclock-changes.patch
```

### 4. libc++ threading changes
```cmd
cd C:\workspace\v8_build\v8\buildtools\third_party\libc++\trunk
git am ..\..\..\..\..\ARM64EC_Patches\0001-ARM64EC-libc-changes.patch
```

### 5. ICU changes
```cmd
cd C:\workspace\v8_build\v8\third_party\icu
git am ..\..\ARM64EC_Patches\0001-ARM64EC-ICU-changes.patch
```

### 6. Protobuf changes
```cmd
cd C:\workspace\v8_build\v8\third_party\protobuf
git am ..\..\ARM64EC_Patches\0001-ARM64EC-protobuf-changes.patch
```

> If any patch fails due to whitespace, apply it with:
> ```cmd
> git am --ignore-whitespace --ignore-space-change <patch-file>
> ```

Each patch is applied inside its own subdirectory because each of those folders (build, third_party/icu, buildtools/third_party/libc++/trunk, etc.) is a separate git repository checked out by gclient — they each have their own .git folder. So git am must be run from within the repo that owns those files, not from the V8 root.

### Step 5 — Configure the build

Generate the Ninja build files using GN. Make sure `DEPOT_TOOLS_WIN_TOOLCHAIN=0` is set in your environment.

```cmd
set DEPOT_TOOLS_WIN_TOOLCHAIN=0
gn gen out/Release_ARM64EC --args="target_cpu=\"arm64\" v8_target_cpu=\"arm64\" is_debug=false is_clang=false is_component_build=true v8_enable_webassembly=false is_arm64ec=true"
```

| GN Argument | Value | Reason |
|---|---|---|
| `target_cpu` | `arm64` | Build for ARM64 architecture |
| `v8_target_cpu` | `arm64` | V8 JIT targets ARM64 |
| `is_debug` | `false` | Release build, optimizations enabled |
| `is_clang` | `false` | Use MSVC (required for ARM64EC) |
| `is_component_build` | `true` | Produce DLLs rather than static libs |
| `v8_enable_webassembly` | `false` | Disable WebAssembly (not needed, reduces build scope) |
| `is_arm64ec` | `true` | Enable ARM64EC ABI mode |

### Step 6 — Build

```cmd
ninja -C out/Release_ARM64EC v8
```

To also build the d8 developer shell:

```cmd
ninja -C out/Release_ARM64EC d8
```

Build time is approximately **30–60 minutes** on a modern machine.

---

## Output Binaries

After a successful build, the following files will be in `out/Release_ARM64EC/`:

| File | Description |
|------|-------------|
| `v8.dll` | Main V8 engine — ARM64EC |
| `v8.dll.lib` | Import library for linking |
| `v8_libbase.dll` | V8 base utilities |
| `v8_libbase.dll.lib` | Import library |
| `v8_libplatform.dll` | V8 platform abstraction layer |
| `v8_libplatform.dll.lib` | Import library |
| `icui18n.dll` | ICU internationalization |
| `icuuc.dll` | ICU unicode core |
| `snapshot_blob.bin` | Pre-compiled JavaScript builtins snapshot |
| `d8.exe` | V8 developer shell (if built) |

---

## Verifying the Build

Confirm the DLLs are built as ARM64EC using `dumpbin` from a Visual Studio Developer Command Prompt:

```cmd
dumpbin /headers out\Release_ARM64EC\v8.dll | findstr /i "machine"
```

Expected output:
```
            8664 machine (x64) (ARM64X)
```

Run a quick smoke test using the d8 shell:

```cmd
out\Release_ARM64EC\d8.exe -e "print('V8 ARM64EC is working: ' + 1+1)"
```
---

## Version Info

| Component | Version |
|-----------|---------|
| V8 | 10.4.132.24 |
| Windows SDK | 10.0.26100.0 |
| Visual Studio | 2022 (MSVC 14.x) |
| Target ABI | ARM64EC |