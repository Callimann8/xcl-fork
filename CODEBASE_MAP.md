# XCL codebase map

A dense index of the XCL code for people and agents who need to find something without reading
the tree. It points at files and functions; the code and `HANDOFF.md` stay the source of truth.
Written against upstream `ivall/xcl` at `7ebf1b8` (2026-11-24). Paths are relative to the repo
root, and `src/` means `crates/xcl/src/`. Function names are given rather than line numbers, so
search for them (`rg 'fn name'`).

## 1. What XCL is, in one paragraph

XCL runs Xbox Series X|S (GDK) titles on Windows **without CPU emulation**. `xcl.exe` maps the
title's x64 PE into its own process, routes the console DLL imports (`kernelx`, XGameRuntime,
`d3d12_xs`, …) to Rust "HLE" stand-ins, hooks the Win32 file APIs into a VFS, and recompiles the
console's RDNA2 GPU machine code to HLSL, then to DXIL with DXC. GPU memory is shared with the CPU
as on the console ("UMA"). Windows only (MSVC toolchain, a D3D12 SM 6.6 GPU with mesh shaders,
DXC). On Linux/macOS it neither builds nor runs.

## 2. Repository layout

| Path | What |
|---|---|
| `crates/xcl/src/` | the runtime + the `xcl` CLI (one crate, ~90k lines incl. unit tests) |
| `crates/xcl/src/hle/` | console library stand-ins (kernelx, XGameRuntime, D3D12.X, audio, input, storage) |
| `crates/xcl/src/gpu/` | RDNA2 decoder, shader recompiler, tiling, textures, UMA, BVH |
| `crates/xcl/tests/` | integration tests (e2e with the test title, GPU tests, game-data tests) |
| `crates/xcl/data/stubdb.toml` | default return values for auto-stubs, per DLL / function |
| `tests/testtitle`, `tests/testgamedll` | our own fake console title + a "game-shipped" DLL, driven by `tests/e2e.rs` |
| `scripts/` | `test.ps1`, `smoke.ps1`, `pr_check.py`, generators (`gen_*.py`), `re/` reverse-engineering helpers |
| `docs/specs/2026-11-20-xcl-design.md` | design (sections §5.1–5.10 map to the modules below) |
| `docs/wave64-on-wave32.md` | how console wave64 programs run on 32-lane GPUs (NVIDIA) |
| `HANDOFF.md` | status, measurements, findings per title, "Open" lists, "Next steps". Long: search it |
| `.github/workflows/` | `ci.yml` (Windows build + tests, no GPU), `pr-check.yml` (runs `scripts/pr_check.py` on the PR body) |

Generated files (never edit by hand; rerun the script): `src/gpu/rdna2_spec.rs`
(`scripts/gen_rdna2.py`), `src/gpu/tiling_tables.rs` (`gen_tiling.py`), `src/hle/d3d12_layout.rs`
(`gen_d3d12_layout.py`), `src/hle/xgameruntime/gdk_table.rs` (`gen_gdk_table.py`).

## 3. The boot path, step by step

Two processes: the **launcher** (`xcl run`) prepares everything and starts a copy of itself as the
**host**, which becomes the game process.

1. **`src/main.rs` `main()`**: if `XCL_HOST` is set, this is the host: `host::host_main(boot.json)`.
   Otherwise it dispatches the CLI (`run`, `inspect`, `disasm`, `shaders`, `rpf`), or shows the
   Explorer double-click menu (`run_picked_game`). `main.rs` also registers XCL's own TLS callback
   (`.CRT$XLY` → `loader::on_tls_event`), which is how every new thread gets the game's static TLS.
2. **`src/launcher.rs` `run()`**:
   - `resolve_location` / `pick_exe`: game folder or exe → package root, exe, title name
     (`MicrosoftGame.config` read by `read_game_config`).
   - Data dir `%LOCALAPPDATA%\XCL\<title>\` (`data_root()`, `XCL_DATA_ROOT` overrides) with
     `logs cache overlay saves pls temp`.
   - `Config::load_or_create(xcl.toml)` then `Config::resolve(manifest)` (`src/config.rs`,
     `src/manifest.rs`: title memory mode, PLS quota, TitleId from the manifest).
   - Writes the **boot record** `cache\boot.json` (`config::Boot`), rotates `xcl.log` →
     `xcl.prev.log` and `report.json` → `report.prev.json`.
   - Prints console imports (`inspect::console_imports`), runs **preflight** (`src/preflight.rs`:
     DXC, system XGameRuntime, GPU SM 6.6 / mesh / wave width), and checks Zen 2 CPU features
     (`missing_console_cpu_features`, bypass `XCL_SKIP_CPU_CHECK`).
   - `spawn_host` starts `xcl.exe` again with `XCL_HOST=<boot.json>` and the game's command line
     (argv[0] = game exe, cwd = game root), waits, then `summarize` prints the crash line, called
     stubs, unmapped paths and the `Report:` / `Log:` paths (smoke.ps1 parses `Log:`).
3. **`src/host.rs` `host_main()`**, in order:
   1. read `boot.json`; `log::init` (file opened **before** any hook so logging never re-enters
      the VFS); `state::set_boot`.
   2. optional diagnostic threads: `XCL_ACTIVITY`, `XCL_THREAD_DUMP_AT`, `XCL_RUN_SECONDS`.
   3. `stubs::init_db` (stubdb.toml + `[stub_returns]`), `hle::init(config)` (module registry and
      routes), `loader::set_cache_dir`, `resolver::set_search_dirs`.
   4. `vfs::init` (mounts) and `vfs_hooks::install()` (KERNELBASE file API detours).
   5. `eh::install_pc_hook()` (`RtlPcToFileHeader` for C++ exceptions in mapped images),
      `eh::install_crash_handler()`, `eh::start_probes()` (`XCL_PROBE`/`XCL_WATCH`).
   6. `HOST_MODE = true`; `loader::load_main(exe)`: map image, relocate, resolve imports (each
      import goes through `resolver::resolve_import`), TLS carrier, `.pdata`, game DLLs + their
      `DllMain`s.
   7. `hle::kernelx::adopt_title_hooks(&main)` (the title's exported `__XMemAllocationHooks__`).
   8. `report::write_final(None)`; `loader::run_main` runs the entry point on a new thread with the
      PE's stack size (sets `PEB.ImageBaseAddress` to the game, except under Wine). It usually
      never returns; the process ends through `ExitProcess`, which `eh` hooks
      (`hk_nt_terminate_process`) to log status + callers and flush the log.

Exit codes from the host itself: `0x7C10001` (boot.json unreadable), `0x7C10002` (loading the game
failed, reason in the log and `report.json` → `crash`).

## 4. Module map: `src/` top level

| File | Role | Key entry points |
|---|---|---|
| `main.rs` | CLI, host switch, TLS callback, double-click menu | `main`, `run_picked_game` |
| `launcher.rs` | `xcl run`: data dir, config, boot record, preflight, spawn host, summary | `run`, `data_root`, `spawn_host`, `summarize` |
| `host.rs` | host process setup (see §3) | `host_main`, `HOST_MODE` |
| `config.rs` | `xcl.toml` (`Config`) + `Boot` record; default mounts | `Config::resolve`, `Boot::{logs_dir,report_path,expand}` |
| `manifest.rs` | the few `MicrosoftGame.config` fields used | `load`, `parse` |
| `state.rs` | global boot/config accessors | `boot()`, `config()`, `data_dir()` |
| `loader.rs` | PE mapping into the process, DLL init, entry point | `map_image_ex`, `load_main`, `load_game_dll`, `init_dll`, `run_main`, `on_tls_event` |
| `pe.rs` | bounds-checked PE32+ parser (file or mapped layout) | `Pe`, `ImportName`, `Exports` |
| `resolver.rs` | import policy: HLE → game DLL → Windows DLL → auto-stub; import tracer | `resolve_import`, `classify`, `load_library`, `get_proc_address`, `find_game_dll` |
| `modules.rs` | table of mapped images + HLE pseudo-handles; lock-free addr→module | `describe_address` (module+RVA strings in logs), `find_mapped_by_addr`, `hle_handle` |
| `stubs.rs` | auto-stubs: x64 thunk logging each call + configured return value | `make_stub`, `record_call`, `StubDb`, `all_stubs` |
| `tls.rs` | static TLS for mapped images via a synthesized "carrier" DLL | `install`, `on_thread_attach`, `build_carrier_dll` |
| `eh.rs` | exceptions: `.pdata`, `RtlPcToFileHeader` hook, crash filter, Zen 2 insn emulation, thread dumps, watchpoints, probes | `unhandled_filter`, `emulate_console_only`, `cpp_exception_logger`, `dump_threads`, `watch_writes`, `start_probes`, `hook_gs_failure` |
| `insn.rs` | decoder for Zen 2-only instructions (CLWB, CLZERO, RDPRU, MONITORX/MWAITX, RDPID) | used by `eh::emulate_console_only` |
| `hooks.rs` | `retour` inline-hook helper | `hook(module, func, detour, &orig)` |
| `vfs.rs` | console path → host path, read-only game dir + overlay + whiteouts | `Vfs`, `Decision`, `full_path` |
| `vfs_hooks.rs` | KERNELBASE file API detours feeding the VFS (CreateFile*, Find*, Move/Copy/Replace, disk free) | `install` (gap: direct `Nt*` syscalls are not hooked) |
| `log.rs` | process logger, `xlog_{error,warn,info,debug,trace}!` macros | `init`, `flush`, `uptime` |
| `report.rs` | `report.json` (`Report` struct) | `build`, `write_final`, `load` |
| `preflight.rs` | environment checks before a run | `run`, `checks`, `print` |
| `inspect.rs` | `xcl inspect`: static import analysis | `run`, `console_imports` |
| `disasm.rs` | `xcl disasm` (RDNA2, console containers) | `run` |
| `shaders.rs` | `xcl shaders`: pack statistics, `--recomp`, `--compile N`, `--precompile [--only cs] [--pipelines]` | `run`, `precompile` |
| `rpf.rs`, `rpf_probe.rs` | `xcl rpf`: RPF9 record listing/extraction (TOC encrypted; records found by scan) | `run` |
| `util.rs` | HRESULT constants, wide/ANSI string helpers, GDK string outputs, `under_wine` | `wide`, `from_wide_ptr`, `write_cstr_out`, `dll_key`, `glob_match` |

## 5. HLE layer: `src/hle/`

### 5.1 Registry and routing (`hle/mod.rs`)

An `HleModule` is a name → address table plus `fallback` DLLs for names XCL does not implement.
`build(config)` creates the modules and the **routes** (glob on the lowercase DLL name; user
`aliases` in `xcl.toml` win):

| Imported DLL pattern | HLE module | Fallback for unimplemented names |
|---|---|---|
| `kernelx`, `kernel32`, `kernelbase`, `advapi32`, `*legacy`, `minappmodelx`, `api-ms-win-core-*`, `ext-ms-win-core-*` | `kernelx.dll` (`kernelx.rs`) | the real DLL of the same name, KERNELBASE, KERNEL32, advapi32, sechost |
| `xgameruntime*.dll`, `xgameplatform.dll` | `xgameruntime.dll` (`xgameruntime/`) | none (stub) |
| `d3d12_xs.dll`, `d3d12_x.dll`, `d3d12x.dll` | `d3d12_xs.dll` (`d3d12x/`) | none |
| `xaudio2_9*.dll` | `xaudio2.rs` | system `xaudio2_9.dll` |
| `mfplat.dll` | `mfplat.rs` | system `mfplat.dll` |
| `minuserx.dll`, `ext-ms-win-rtcore-ntuser-window-*` | `user.rs` | `user32.dll` |
| `xcrdapi.dll` | `dstorage.rs` (console DirectStorage) | none |
| `xg_xs.dll` | `xg.rs` (texture layout library) | none |
| `acphal.dll`, `xapu.dll`, `chad.dll` | `acphal.rs` (audio processor, decoder, DSP) | none |
| `gameinput.dll` | `gameinput.rs` (+ `kbm.rs`) | none |
| `dxcompiler_xs.dll`, `dxcompiler_x.dll` | `dxcompiler.rs` (the PC DXC) | none |
| `winhttp.dll`, `ws2_32.dll` (only with `XCL_NETWORK=0`) | `netoff.rs` | real DLL |

Resolution order (`resolver.rs`): route → HLE export (or fallback / `@system` / `@stub`) → DLL in the
game folder (mapped by XCL's loader) → real Windows DLL → **auto-stub** (`stubs.rs`, default return
from `data/stubdb.toml`: E_NOTIMPL for most console DLLs). Every stub call is counted and lands in
`report.json` → `called_stubs` with its first caller.

### 5.2 kernelx (`hle/kernelx.rs`, `kernelx/lazy.rs`, `kernelx/placeholders.rs`)

Only what differs from Windows; everything else falls through to KERNELBASE.

- **Memory**: `VirtualAlloc`/`VirtualFree`/`VirtualProtect` (console alloc-type and protect flags
  sanitized: `sanitize_alloc_type`, `sanitize_protect`; graphics allocations go to UMA or the
  coherent window, `XCL_COHERENT_HEAPS=0` turns that off), `XMemAlloc*`/`XMemFree*`,
  `XMemVirtualAlloc`, `XMemSetAllocationHooks`, `TitleMemoryStatus` (budget from config: 13 824 MiB
  Series X, 8 192 Series S).
- **Physical pages**: `AllocateTitlePhysicalPages`, `MapTitlePhysicalPages`,
  `XMemMapPhysicalPages(Scatter)`; pages are 4 KiB frames; a page's "home" vs. mapped-away copies.
  `kernelx/lazy.rs`: zero pages are mapped reserved-only and committed by a vectored handler on
  first CPU touch; kernel calls don't fault, so ReadFile buffers / DirectStorage destinations call
  `lazy::ensure` first. `kernelx/placeholders.rs`: placeholder reservations inside the sparse GPU
  address space.
- **Modules**: `LoadLibrary*`, `GetProcAddress`, `GetModuleHandle*`, `GetModuleFileName*`,
  `FreeLibrary` (HLE pseudo-handles for console DLLs, real handles for kernel-family aliases).
- **Threads / system**: `SetThreadName`/`GetThreadName`, affinity masks fitted to the PC
  (`fit_affinity_mask`), `GetConsoleType`, `GetSystemOSVersion`, registry reads the GDK thunks use
  (`RegOpenKeyExW`/`RegQueryValueExW` answer the console mode for XSystemGetDeviceType),
  app-state notifications (no-ops), `SetUnhandledExceptionFilter` (title filter runs after XCL's).

### 5.3 XGameRuntime (`hle/xgameruntime/`)

- **`api.rs`: the thunk protocol** real titles use. The GDK static thunk lib loads
  `xgameruntime.dll`, calls `InitializeApiImpl(Ex/Ex2)`, then `QueryApiImpl(clsid, iid)` per API
  family and calls vtable slots. Slot layout comes from `gdk_table.rs` (396 functions, generated).
  XCL answers XUser (one offline local user), XGameSave, persistent local storage, XPackage, XGame
  (title id), XSystem (Series X / S), runtime features, display deferrals, XGameUi dialogs without a
  user; refuses XStore / broadcast / capture; **passes the rest to the PC's
  `System32\xgameruntime.dll`** (task queues, async, networking) when Gaming Services is installed.
  `report.json` → `xgameruntime` lists classes asked for, who answered, and unimplemented slots.
- `mod.rs`: by-name exports for titles that import them directly.
- `taskqueue.rs` (XTaskQueue), `xasync.rs` (XAsync / XAsyncProvider; goes to the PC runtime's when
  present), `gamesave.rs` (`<data>/gamesave/...` files), `networking.rs` (XNetworking),
  `xsign.rs` (offline XSTS request signing; tokens only with `xbox_live_tokens = true`),
  `accessibility.rs`.

### 5.4 D3D12.X (`hle/d3d12x/`)

Entry: `mod.rs` exports `D3D12XboxCreateDevice`, `D3D12CreateDevice`, root signature (de)serializers,
`D3D12ConfigureVirtualMemoryX`. Backend chosen once: `real` (default) or `null` (`graphics = "null"`,
or no device could be created).

| File | What it owns |
|---|---|
| `real.rs` (largest) | genuine PC D3D12 objects **"adopted"**: vtable replaced with an extended copy (`adopt`, `readopt`, `Kind`). Layout: `ext[-1]` original vtable, PC methods (some overridden), guard thunks for internal virtuals, logging stubs for console-only slots. Overrides every `Create*`, `QueryInterface` (logs unknown IIDs), presentation hooks. `console_remap` (slot moves per console layout), `this_first` (struct-return argument order), `create_device` (adapter, Agility SDK, debug layer, DRED), `texture_memory`, `video_memory`, `end_title`. `D3d12Stats` → `report.json` `d3d12` |
| `device_x.rs` | console device extensions from slot 44 (CreatePlacedResourceX #44, CreateCommandQueueX #48, Create*PipelineStateX #60/#62, CreateCommandListX #73, page pools #74/#75, …); `pc_shader_view_format` (console formats → valid PC view formats) |
| `list_x.rs` | console command list extensions from #60 (CopyMemoryToMemoryX, FillMemoryWith*X, Write*BitValue{Top,Bottom}OfPipeX, Wait*X, DispatchX, DrawIndexedX, SetPredicationBufferX, …); `gpu_va` (IB/VB outside UMA mirrored in) |
| `title_list.rs` | command lists of **console code** (SDK inline writers write records straight into the list object at +0x18/+0x28/+0x30); `XCL_LIST_TRACE` |
| `console_list.rs` | console-shaped compute lists and PSOs, replayed at `close` (opt-in `XCL_CONSOLE_LISTS=1`) |
| `bind.rs` | root arguments of console code → XCL shader ABI (user SGPRs in `io`, arg table), draws/dispatches with translated pipelines, `draw_rects` (rect lists as 2 triangles), draw tracing (`XCL_TRACE_SHADERS`, `XCL_TRACE_MANY`, `XCL_TRACE_UI_ANY`, `XCL_TRACE_ADDRESSES`) |
| `console_heap.rs` | console descriptor heaps: 32-byte T#/V# slots and 16-byte S# living in UMA, PC descriptors beside each |
| `frame.rs` | frame clock: `SetFrameIntervalX`, `ScheduleFrameEventX`, `WaitFrameEventX`, `GetFrameStatisticsX` (device slots 78–81). `XCL_TRACE_FRAME_EVENTS` |
| `present.rs` | `PresentX` (queue slot 30) → XCL window; frame dumps (`XCL_FRAME_DUMP`), window size/pos |
| `pso_async.rs` | non-blocking PSOs: stand-in at once, real one built on a worker; compute PSOs waited for at first bind (`XCL_ASYNC_PSO=0`, `XCL_LAZY_PSO=0`) |
| `cpu_writes.rs` | bottom-of-pipe writes into memory the GPU can't see, done by a CPU thread after a fence |
| `fence_x.rs` | fence #12 (address of the completed value the title polls with `monitorx`/`mwaitx`) |
| `vram.rs` | GPU-owned RT/DS/UAV textures at title addresses: reserved resources on shared 64 KiB pages (aliasing as on console), arenas for what can't be reserved (D32S8, MSAA; `XCL_VRAM_ARENAS=1` forces arenas) |
| `sparse.rs` | the title's own GPU-visible reservations placed in UMA sparse windows (4 GiB tiled buffers); `XCL_EARLY_PAGES` |
| `title_textures.rs` | textures placed in title memory: detiled PC copy, re-uploaded when written pages change (write-watch); `XCL_TEXTURE_SKIP_UNCHANGED`, `XCL_SYNC_SMALL_TARGETS` |
| `guard.rs` | guard thunks for extended vtables (caller in game image → console method, else original) |
| `dxgi_device.rs` | device as IDXGIDevice (adapter → outputs for `SetFrameIntervalX`) |
| `null.rs` | the render-nothing backend |
| `layout.rs` + `../d3d12_layout.rs` | generated PC D3D12 interface slot table |

### 5.5 Other HLE modules

| File | What |
|---|---|
| `dstorage.rs` | console DirectStorage (`DStorageGetFactory` → factory / queue / file / status array in console vtable order); BCPack requests refused (`XCL_DSTORAGE_BCPACK_FILL` fills instead); `counters()` feeds `XCL_ACTIVITY` |
| `xaudio2.rs` | console XAudio2 on Windows XAudio2.9 (vtables patched in place); XMA via `xma2/` (pure Rust, ported from FFmpeg), Opus via `opus.rs` (`MSOpusDecoder.dll`, or libopus) |
| `acphal.rs` | audio processor / xapu / chad: acknowledges commands, ticks frames, decodes Opus packets; `XCL_TRACE_APU` |
| `gameinput.rs`, `kbm.rs` | one always-connected gamepad from XInput + keyboard/mouse; `XCL_PRESS_AT`, `XCL_PRESS_A_AT` for scripted presses |
| `xg.rs` | `XGCreateTextureComputer` & co. from XCL's GFX10 tiling model |
| `user.rs` | console window functions (ANSI calls made as Unicode) |
| `mfplat.rs` | `MFResetDXGIDeviceManagerX` + system Media Foundation |
| `netoff.rs` | `XCL_NETWORK=0`: WinHTTP/Winsock fail as offline; loopback kept |
| `dxcompiler.rs` | console DXC → the PC DXC XCL loaded |

## 6. GPU: `src/gpu/`

### 6.1 Shader path, from title to GPU

1. Title calls `Create*PipelineState(X)` with a DXBC container whose DXIL part holds console code
   (`gpu/xbox.rs`: XDAT/XBCH/FSSM parts). `gpu/shader.rs` classifies it (`classify`) and with
   `XCL_DUMP_SHADERS=1` writes it to `<data>\shaders\` with a disassembly.
2. `gpu/recomp/pso.rs`: `is_console` → `console_to_dxil_in(blob, stage, sampler_base,
   split_wave64, windows)`; mesh/NGG `console_mesh`, VS+GS `console_geometry`, rect lists
   `rect_list_gs`. The pipeline gets **XCL's ABI root signature** (`abi_root_signature_with`):
   u0 `io` (initial SGPRs = user data), u1 `res`, u2.. `mem` (flat memory windows), plus directly
   indexed descriptor heaps and the shader's static samplers.
3. `gpu/rdna2.rs` decodes (tables in generated `rdna2_spec.rs`, extras in `rdna2_xbox.rs`).
4. `gpu/recomp/mod.rs` `translate()`: a `switch(pc)` dispatcher loop over basic blocks (branches are
   scalar), structured control flow when possible (`XCL_RECOMP_NO_STRUCTURED` disables;
   `XCL_RECOMP_WHY` says why a program wasn't structured). `recomp/emit.rs`: one instruction →
   HLSL. Wave64 on 32-lane GPUs: `x64_masks.rs`, `upper_lanes.rs`, `mesh_wave.rs`,
   `mesh_dispatch.rs` (+ `docs/wave64-on-wave32.md`).
5. `gpu/recomp/dxc.rs`: DXC loaded at run time (`XCL_DXC_DIR`, next to xcl.exe, or newest Windows
   SDK), `compile_cached` with a disk cache (`XCL_DXIL_CACHE`, default `%LOCALAPPDATA%\xcl\dxil`,
   plus `translated\` for HLSL translations, `XCL_TRANSLATED_CACHE=0` off); `wait_for_memory`
   throttles compiles when commit is low; `XCL_DXC_THREADS`.
6. `hle/d3d12x/pso_async.rs` builds the PSO in the background; `bind.rs` binds root args at draw.

Warm the caches offline (no title running):
`xcl shaders <shaders.rpf> --precompile [--only cs] [--pipelines]` (`XCL_PIPELINE_THREADS`, 4).

### 6.2 Other GPU files

| File | What |
|---|---|
| `uma.rs` | unified memory: up to 4 windows of <4 GiB opened as D3D12 heaps from address (`OpenExistingHeapFromAddress`); coherent window sized from the last run (`cache/uma_coherent.txt`, `coherent_hint`); `XCL_UMA_MB` (default 2048, or 4095 with "advanced" title memory), `XCL_COHERENT_MB`, `XCL_UMA_CHECK` |
| `desc.rs` | GFX10 T#/S#/V# descriptors → DXGI formats, SRVs, samplers |
| `tiling.rs` (+ generated `tiling_tables.rs`) | GFX10 swizzle (2D, 3D thick/thin, linear), `Chain` = mip/slice layout |
| `texture.rs` | console texture memory → detiled PC subresources |
| `bvh.rs` | RDNA2 BVH node format, `build`/`build_top` |
| `adapter.rs` | adapter pick: `d3d12_adapter` / `XCL_ADAPTER` (`auto`, `warp`, index, name substring) |
| `agility.rs` | Agility SDK `D3D12Core.dll` loading (Win10), `XCL_D3D12_DIR` |
| `dxbc.rs` | input/output signatures (VS→PS linkage) |
| `fence.rs` | completion checks for XCL-owned fences |
| `pack.rs` | RPF9 shader-pack record scan (no TOC) |
| `recomp/runner.rs` | runs recompiled kernels on the GPU (test bench); `skip_test(reason)` for GPU tests |
| `recomp/tests.rs` | per-instruction GPU tests |

## 7. Configuration

### 7.1 `xcl.toml` (`%LOCALAPPDATA%\XCL\<title>\xcl.toml`, struct `config::Config`)

Created with defaults on first run; `exe` is rewritten on every run. Fields: `device_type`
(`anaconda` = Series X, `lockhart` = Series S), `console_type_raw`, `gamertag`, `user_id`,
`console_id`, `sandbox_id`, `title_memory` (`auto`/`standard`/`advanced`), `title_memory_mb`,
`pls_mb`, `uma_mb`, `log_level` (`error|warn|info|debug|trace`), `log_to_stderr`, `exe`,
`package_identifier`, `title_id`, `[mounts]` (defaults: `GAME:`, `G:`, `D:` → `{game}`; `T:` →
`{data}\temp`; `S:` → `{data}\pls`), `[aliases]` (DLL pattern → HLE module / `@system` / `@stub`),
`[stub_returns]` (`dll!func` or `*!func` → value), `graphics` (`d3d12`/`null`), `d3d12_adapter`,
`d3d12_debug`, `present` (`auto`/`off`), `raytracing`, `window_width`/`window_height`,
`window_pos`, `xbox_live_tokens`, `keyboard_mouse`, `mouse_sensitivity`.

### 7.2 Files a run produces (`%LOCALAPPDATA%\XCL\<title>\`)

`logs\xcl.log` (+ `xcl.prev.log`), `logs\report.json` (+ `report.prev.json`), `logs\frame_<n>.bmp`
(frame dumps), `cache\boot.json`, `cache\uma_coherent.txt`, `overlay\` (writes to game files),
`saves\`, `gamesave\`, `pls\` (`S:\`), `temp\` (`T:\`), `shaders\` (with `XCL_DUMP_SHADERS=1`).

`report.json` (`report::Report`): `crash`, `exit_code`, `environment` (preflight),
`imports_by_source`, `stubbed_imports`, **`called_stubs`** (calls + first caller), `modules`,
`vfs` (unmapped paths, redirected writes), `shaders` (formats, mnemonics), `dxc`, **`d3d12`**
(adapter, unknown IIDs, fallbacks, presents, `last_frame_dump`), `audio`, **`xgameruntime`**
(classes, unimplemented slots), `dstorage`, `notes`.

## 8. Environment variables

All are read with `std::env::var` at the place named; search the name to find the code.

### 8.1 Run control and setup

| Variable | Effect | Where |
|---|---|---|
| `XCL_DATA_ROOT` | root for per-title data (default `%LOCALAPPDATA%\XCL`) | `launcher.rs` |
| `XCL_HOST` | internal: set by the launcher to start host mode | `main.rs`, `launcher.rs` |
| `XCL_SKIP_CPU_CHECK` | run although Zen 2 CPU features are missing | `launcher.rs` |
| `XCL_RUN_SECONDS=n` | end the title cleanly after n s (GPU drained, log + report written) | `host.rs` |
| `XCL_NETWORK=0` | offline mode (always use it for a title with a crash uploader) | `hle/netoff.rs` |
| `XCL_ADAPTER` | GPU choice (`1`, `radeon`, `warp`), overrides `d3d12_adapter` | `gpu/adapter.rs` |
| `XCL_DXC_DIR` | DXC folder (`dxcompiler.dll` + `dxil.dll`) | `gpu/recomp/dxc.rs` |
| `XCL_D3D12_DIR` | Agility SDK folder | `gpu/agility.rs` |
| `XCL_UMA_MB`, `XCL_COHERENT_MB` | unified memory size / coherent window size | `gpu/uma.rs` |
| `XCL_RAYTRACING` | reported raytracing tier | `config.rs` |
| `XCL_WINDOW_POS` | window position | `hle/d3d12x/present.rs` |
| `XCL_DXIL_CACHE`, `XCL_TRANSLATED_CACHE=0` | DXIL cache folder / turn off the translation cache | `gpu/recomp/dxc.rs`, `pso.rs` |
| `XCL_DXC_THREADS`, `XCL_PIPELINE_THREADS` | compile parallelism | `dxc.rs`, `shaders.rs` |
| `XCL_PRESS_AT`, `XCL_PRESS_A_AT=s1,s2` | scripted gamepad presses (e.g. get past a "press A" screen in an unattended run) | `hle/gameinput.rs` |

### 8.2 Feature switches (mostly to bisect a regression)

`XCL_ASYNC_PSO=0`, `XCL_LAZY_PSO=0` (`pso_async.rs`); `XCL_COHERENT_HEAPS=0` (`kernelx.rs`);
`XCL_VRAM_ARENAS=1` (`vram.rs`); `XCL_TEXTURE_SKIP_UNCHANGED=0`, `XCL_SYNC_SMALL_TARGETS=0`
(`title_textures.rs`); `XCL_CONSOLE_LISTS=1` (`console_list.rs`); `XCL_SPLIT_WAVE64=1`,
`XCL_RECOMP_NO_STRUCTURED` (`recomp/pso.rs`, `mod.rs`); `XCL_X64_ALL_MASKS`, `XCL_WAVE64_SPIN`
(recompiler); `XCL_UMA_CHECK=0` (`real.rs`); `XCL_DSTORAGE_BCPACK_FILL` (`dstorage.rs`).

### 8.3 Diagnostics

| Variable | What it logs / does | Where |
|---|---|---|
| `XCL_ACTIVITY=1` | every 2 s: presents, submits, draws, dispatches, frame-event waits, DirectStorage requests, process commit, video memory, pages/arenas/pools. **First thing to turn on for a hang** | `host.rs` |
| `XCL_THREAD_DUMP_AT=s1,s2` | at those seconds, every thread's RIP + call stack (module+RVA) | `host.rs`, `eh::dump_threads` |
| `XCL_TRACE_FAULTS=1` | first 32 access violations as raised, before the title's own handlers (finds crashes the title's crash filter swallows) | `eh.rs` |
| `XCL_TRACE_IMPORTS=…` | call tracer with args + result: `1`/`hle`, `all`, `prefix`, `-prefix` (e.g. `hle,-kernelx!GetTickCount`) | `resolver.rs` |
| `XCL_TRACE_MEM=n` | first n allocations ≥ 64 KiB | `kernelx.rs`, `uma.rs` |
| `XCL_WATCH=rva` | hardware watchpoint: every write to 8 bytes at that exe RVA, with the writing code | `eh.rs` |
| `XCL_PROBE=rva,…` | up to 3 hardware breakpoints in the exe: registers at each new hit | `eh.rs` |
| `XCL_D3D12_DEBUG=1` | D3D12 debug layer (needs Graphics Tools / `d3d12SDKLayers.dll`); finds most device removals | `real.rs` |
| `XCL_DRED=1` | on device removal: faulting GPU address and last GPU operations | `real.rs` |
| `XCL_D3D12_TRACE` | log D3D12 calls from the game image | `real.rs` |
| `XCL_WATCH_DEVICE=1`, `XCL_WATCH_FENCE=n`, `XCL_WATCH_DMA` | watch writes over the device / n-th fence / DMA list objects | `real.rs`, `device_x.rs` |
| `XCL_LIST_TRACE` | last slots called on console command lists | `title_list.rs` |
| `XCL_CONSOLE_LISTS_LOG` | log replayed console list records | `console_list.rs` |
| `XCL_TRACE_FRAME_EVENTS` | first 40 frame-event calls (frame pacing stalls) | `frame.rs` |
| `XCL_TRACE_FRAME` | view-slot → resource mapping at present | `present.rs` |
| `XCL_TRACE_SHADERS=hash,…`, `XCL_TRACE_MANY=n`, `XCL_TRACE_UI_ANY`, `XCL_TRACE_ADDRESSES=1` | draw tracing with root arguments | `bind.rs` |
| `XCL_FRAME_DUMP=n`, `XCL_FRAME_DUMP_FULL=1` | every n-th shown frame to `logs\frame_<n>.bmp`; `report.json` `d3d12.last_frame_dump` | `present.rs` |
| `XCL_DUMP_SHADERS=1` | write every submitted shader + disassembly to `<data>\shaders\` | `gpu/shader.rs` |
| `XCL_DUMP_GS=dir` | dump VS+GS pairs that fail to translate | `real.rs` |
| `XCL_RECOMP_WHY` | why a program stays on the unstructured block chain | `recomp/` |
| `XCL_DEBUG_MAGENTA`, `XCL_DEBUG_LOD0`, `XCL_DEBUG_PS_EXPR`, `XCL_DEBUG_PS_SUB` | per-pixel-shader overrides by dump hash (find which shader draws what) | `recomp/pso.rs` |
| `XCL_DEBUG_NO_STENCIL`, `XCL_DEBUG_NO_CULL` | per-PS pipeline overrides | `real.rs` |
| `XCL_TRACE_APU=1` | audio processor commands | `acphal.rs` |
| `XCL_OPUS_WAV`, `XCL_OPUS_DIR` | write decoded Opus / where to find the decoder | `opus.rs` |

### 8.4 Tests and tools only

`XCL_REQUIRE_GPU=wave64,mesh` (turn GPU skips into failures), `XCL_SAMPLE_{RPF,DLLS,TTPF,BK2,BVH4,AWC,XG,DXC,FSPX}`
(game data the owner points at; tests do nothing when unset), `XCL_DUMP_HLSL`, `XCL_DUMP_EVERY`,
`XCL_RECOMP_ONLY`, `XCL_RECOMP_WINDOWS`, `XCL_RECOMP_DXC`, `XCL_SURVEY_*`, `XCL_CENSUS_*`, `XCL_BLOB`,
`XCL_DXC_ARGS`, `XCL_RAW`, `XCL_TIME_HLSL`, `XCL_PROBE_BVH_*`, `XCL_PROBE_FSR*`, `XCL_SCATTER_PAGES`,
`XCL_EARLY_PAGES`, `XCL_FUZZ_ITERS`, `XCL_GALLERY_DIR`.

## 9. Diagnosing a hang or crash

Always start with `XCL_NETWORK=0` for a title under bring-up. Read the `xcl.log` tail and
`report.json` first; they usually name the problem.

**The run ends with `CRASH:`** (XCL's crash filter caught it). The log line has the exception
code, module+RVA (`modules::describe_address`), registers with the vtable module of objects they
point at, code bytes and stack callers. Then:
- `0xC000001D` illegal instruction: a Zen 2 instruction the CPU lacks. `insn.rs` +
  `eh::emulate_console_only` handle CLWB/CLZERO/RDPRU/MONITORX/MWAITX/RDPID; SHA-NI is not
  emulated (code bytes `0F 38 C8..CD`).
- `0xC0000005` in game code right after a stub was called: check `called_stubs` (an E_NOTIMPL
  stub the title doesn't check, or an output pointer left unset). Implement the function in the
  right HLE module, or try `[stub_returns]` in `xcl.toml` as a quick experiment.
- Reads of `vtable+N` on a strange object, or a D3D12 call that does nothing: a console-only slot.
  These show up in `called_stubs` as `<Interface>::#slot (not on PC)`; the slot tables are in
  `device_x.rs` / `list_x.rs` / `real::console_remap`.
  HANDOFF's "newer list layout's shift by two" section shows how slot numbers moved between
  console SDK versions.

**The process ends with no XCL crash line**: the title's own crash filter ended it. Rerun with
`XCL_TRACE_FAULTS=1` to see the first access violations before its handler runs. The
`hk_nt_terminate_process` line gives the exit status, the calls that led there, and (for status ≠ 0)
all threads.

**Device removed** (`device removed` in the log, e.g. 0x887A0001): rerun with `XCL_D3D12_DEBUG=1` (the debug layer
logs 8 messages per id) and `XCL_DRED=1`. Past causes: a view format the PC refuses
(`device_x::pc_shader_view_format`), out of video memory (look at `XCL_ACTIVITY` memory lines),
raw pipeline streams with console shaders.

**A hang or a black screen**:
1. `XCL_ACTIVITY=1`: are presents / submits / draws still moving? Is DirectStorage reading? Is
   commit near the system limit (shader compiles wait in `dxc::wait_for_memory` when commit is low)?
2. `XCL_THREAD_DUMP_AT=30,60`: where every thread waits (module+RVA). Usual culprits are a frame
   event never fired (`XCL_TRACE_FRAME_EVENTS`, `frame.rs`), a GPU-written value the title polls
   (`cpu_writes.rs`, `fence_x.rs`), an XAsync that never completes (`xasync.rs`), or a PSO still
   compiling (many minutes on a first run: warm with `xcl shaders --precompile`).
3. Black but presenting: `XCL_FRAME_DUMP=n` to see what is presented, then the draw tracing vars
   and `XCL_DEBUG_MAGENTA` to find the shader. Untranslated pipelines and skipped draws/dispatches
   are logged as `D3D12.X: … skipped` / `… without a translation` warnings.

**Unattended runs**: `XCL_RUN_SECONDS=n` ends cleanly; `XCL_PRESS_A_AT=s1,s2` presses A.
`XCL_WATCH=rva` / `XCL_PROBE=rva` find who writes a bad field or what a function gets called with.

## 10. Testing and the PR gate

- Build + all tests (Windows): `.\scripts\test.ps1` or `cargo build --workspace && cargo test
  --workspace -- --test-threads=4`. One area: `cargo test -p xcl --lib <filter>`.
- GPU tests print `SKIPPED <test>: <reason>` when the adapter can't run them (they still pass);
  report them, don't count them as coverage.
- Integration tests in `crates/xcl/tests/`: `e2e.rs` (runs `tests/testtitle` through `xcl run`; the
  title prints `[PASS]/[FAIL]`, exit code = failures), `gpu_pso_e2e.rs`, `gpu_compute_e2e.rs`,
  `predication_e2e.rs`, `texture_states.rs`, `heap_probe.rs`, `pls_quota.rs`, plus game-data tests
  (`console_dlls_e2e.rs`, `console_shaders.rs`, `xg_oracle.rs`, `rpf_e2e.rs`) gated on `XCL_SAMPLE_*`.
- **Regression run** (any change to `src/` outside tests): `scripts/smoke.ps1 -Game <folder>
  [-Seconds 60]` with a release build from `main` and from the PR head (no uncommitted changes). It
  sets `XCL_NETWORK=0`, `XCL_RUN_SECONDS`, `XCL_ACTIVITY=1`, reads the log and prints an
  `xcl-smoke v1` block: commit, gpu (from `report.json`), exit, frames shown, presents per 2 s,
  device removals, crashes, peak commit. `scripts/pr_check.py` (the `PR checks` workflow) requires
  both blocks, no crash or removal in the PR's run, ≥ 80 % of main's frames, ≤ 125 % of its peak
  commit. Never write a block by hand.
- PR rules (CONTRIBUTING.md): one logical change, < ~400 lines, `<Area>: <what>` commit subjects,
  **AI use** section naming the tool and exact model, a test that fails without a bug fix.

## 11. Where to look for X

| I need to… | Look at |
|---|---|
| see why an import resolves the way it does | `resolver::resolve_inner_opt`, `hle::build` routes, `xcl inspect <exe>` |
| implement a missing kernelx function | `hle/kernelx.rs`: write `pub unsafe extern "system" fn Name`, add it to `exports()` |
| implement a missing XGameRuntime API | thunk path: `xgameruntime/api.rs` (`QueryApiImpl`, slot from `gdk_table.rs`); by-name: `xgameruntime/mod.rs` `exports()` |
| change an auto-stub's default return | `crates/xcl/data/stubdb.toml` (or per title `[stub_returns]`) |
| handle a console-only D3D12 device/list/queue slot | `hle/d3d12x/device_x.rs`, `list_x.rs`, `real::console_remap`; `called_stubs` entries `(not on PC)` |
| fix a "not translated" pipeline | `gpu/recomp/pso.rs` (`console_to_dxil_in`, `console_mesh`, `console_geometry`), then `recomp/mod.rs` / `emit.rs` |
| add an RDNA2 instruction | `recomp/emit.rs` + a per-instruction GPU test in `recomp/tests.rs`; decoder gaps in `rdna2_xbox.rs` |
| fix a texture that looks wrong | `gpu/tiling.rs`, `gpu/texture.rs`, `hle/xg.rs`, `hle/d3d12x/title_textures.rs`; format mapping in `gpu/desc.rs` |
| fix render targets / aliasing / VRAM blow-up | `hle/d3d12x/vram.rs`, `sparse.rs`, `gpu/uma.rs` |
| fix memory/commit problems | `hle/kernelx.rs` (VirtualAlloc, physical pages), `kernelx/lazy.rs`, `gpu/uma.rs`; measure with `XCL_ACTIVITY`, `XCL_TRACE_MEM` |
| fix frame pacing / stuck after N frames | `hle/d3d12x/frame.rs`, `present.rs`, `cpu_writes.rs`, `fence_x.rs` |
| fix file-not-found / path problems | `vfs.rs`, `vfs_hooks.rs`, mounts in `config.rs`; `report.json` `vfs.unmapped_paths` |
| fix asset streaming | `hle/dstorage.rs` (+ `kernelx/lazy::ensure` for destinations) |
| fix audio | `hle/xaudio2.rs`, `acphal.rs`, `opus.rs`, `xma2/` |
| fix input | `hle/gameinput.rs`, `kbm.rs` |
| fix saves / PLS / users | `xgameruntime/gamesave.rs`, `api.rs`, `vfs_hooks.rs` (disk free for `S:`) |
| fix C++ exceptions / TLS in the game | `eh.rs` (`RtlPcToFileHeader` hook, `.pdata`), `tls.rs`, `loader.rs` |
| find a log line's source | `rg '<distinctive words>' crates/xcl/src` (messages are plain `xlog_*!` format strings) |
| read status and open issues per title | `HANDOFF.md`: "Retail game bring-up", "XDefiant … first runs", "Next steps (priority)" |

## 12. Known gaps and open items worth knowing before bring-up

From `HANDOFF.md` and module docs at `7ebf1b8`; check HANDOFF for the latest.

- Direct `Nt*` file syscalls are not hooked by the VFS (`vfs_hooks.rs`).
- Memory the GPU does not see: buffers in a title's plain `VirtualAlloc` memory outside UMA give
  shaders empty V#s (bottom-of-pipe writes there are done on the CPU; IB/VB are mirrored).
- Mesh pipelines with a console MS container in the AS slot are refused (the "newer retail game"'s
  minimap); MSAA layouts are incomplete; D32S8 targets still use arenas.
- The per-access memory-window `switch` doubles DXC and driver time on big shaders; first runs of a
  big title spend minutes compiling, so warm the caches with `xcl shaders --precompile`.
- SHA-NI is not emulated for pre-Ice Lake Intel CPUs.
- BCPack DirectStorage requests are refused (no decoder).
- XCL was developed on AMD RDNA4; NVIDIA (wave32 only) runs wave64 programs split in halves
  (slower, and VS/PS that read lanes 32–63 can differ).
