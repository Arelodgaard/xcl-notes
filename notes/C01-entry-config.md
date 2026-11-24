# C01 Entry point, launcher, config, reporting

Files: `crates/xcl/build.rs`, `src/config.rs`, `src/host.rs`, `src/inspect.rs`, `src/launcher.rs`, `src/lib.rs`, `src/log.rs`, `src/main.rs`, `src/manifest.rs`, `src/preflight.rs`, `src/report.rs`, `src/state.rs`, `src/util.rs` · ~3.0k lines · commit c4d37be

## Purpose
This is the outer shell of XCL: the `xcl.exe` command line, the two-process start of `xcl run`, the per-title configuration and data folders, the PC capability check, logging, and the `report.json` written at the end of a run. It decides *which* game file runs, *where* its writable data lives, and *with what settings*, then hands off to the loader (C02) inside a fresh "host" process.

## Main pieces
- `main` (`src/main.rs:51`): subcommand dispatch (`run`, `inspect`, `disasm`, `shaders`, `rpf`, `--version`). If env `XCL_HOST` is set, the process is the host and goes straight to `host::host_main`. Double-clicked with no arguments, it shows a small task-dialog menu and file picker (`run_picked_game`, `main.rs:101`).
- `XCL_TLS_CALLBACK` (`main.rs:16-18`): xcl.exe's own TLS callback placed in `.CRT$XLY`. ntdll calls it for every thread start/exit; in host mode it forwards to `loader::on_tls_event` (C02). This is why the game must run *inside xcl.exe's own process*.
- `launcher::run` (`src/launcher.rs:288`): resolves the package root, title name and exe; creates `%LOCALAPPDATA%\XCL\<title>\{logs,cache,overlay,saves,pls,temp}`; loads/creates `xcl.toml`; resolves "auto" settings from `MicrosoftGame.config`; writes `cache\boot.json`; rotates the previous log/report to `*.prev.*`; runs preflight and the CPU check; spawns the host; prints a summary.
- `launcher::spawn_host` (`launcher.rs:433`): `CreateProcessW` of xcl.exe itself, with the *game's* command line (argv[0] = game exe), cwd = package root, `XCL_HOST` = path of `boot.json`. Waits for it and returns its exit code.
- `Config` (`src/config.rs:10`): the `xcl.toml` schema, with `#[serde(default)]` so partial files work. Covers console identity (device type, console id, gamertag, user id, sandbox), memory budgets (`title_memory`, `uma_mb`, `pls_mb`), console path `mounts`, DLL routing `aliases`, `stub_returns`, graphics/adapter/raytracing/window, Xbox Live token policy, keyboard-and-mouse mapping.
- `Config::resolve` (`config.rs:80`): fills "auto" values from the manifest (title memory mode per device, PLS size, TitleId, UMA size: 4095 MiB for "advanced" memory, else 2048).
- `Boot` (`config.rs:264`): the record handed from launcher to host (title, exe path, game root, data dir, resolved config), plus `{game}`/`{data}` expansion for mounts.
- `manifest::parse` / `load` (`src/manifest.rs:40`, `:61`): pulls the few `MicrosoftGame.config` elements XCL uses (PLS size, Anaconda/Lockhart title memory, TitleId, resource languages) with plain string search.
- `host::host_main` (`src/host.rs:11`): the host process. Starts logging, stores the boot record globally, optionally starts diagnostic threads, then initialises stub DB → HLE → timers → loader cache dir → DLL search dirs → VFS (with mounts) → VFS hooks → `RtlPcToFileHeader` hook → crash handler → probes, sets `HOST_MODE`, maps the main exe, lets kernelx adopt title hooks, writes the report, and runs the entry point.
- `state` (`src/state.rs`): `OnceLock<Boot>` with `config()` and `data_dir()` accessors that fall back to defaults in unit tests. This is the global config everything else reads.
- `preflight::run` / `checks` (`src/preflight.rs:157`, `:57`): probes the GPU (via the same adapter selection and Agility device creation as the real backend), DXC location and version, and `System32\xgameruntime.dll`. Produces leveled `Check`s (Ok/Note/Warning/Error) with a fix hint; stored in the report.
- `log` (`src/log.rs`): process-wide logger. File opened once at startup, before any file hooks, so logging never re-enters the VFS. Lines carry uptime and thread id. Warnings and errors flush immediately. `xlog_*!` macros; debug and trace are checked before formatting.
- `report::build` / `write_final` (`src/report.rs:98`, `:171`): assembles `report.json` from import records (C02), called stubs, mapped modules, VFS unmapped paths and redirected writes, plus stats from shaders, DXC, D3D12, audio, XGameRuntime and DirectStorage.
- `inspect::run` / `console_imports` (`src/inspect.rs:68`, `:37`): static, read-only import classification of a PE (HLE / Windows / game DLL / stub) using the resolver's dry-run `classify`.
- `util` (`src/util.rs`): HRESULT constants, `parse_ret` (stub return-value specs), `dll_key` (lower-case file name with `.dll` added), `glob_match` (one `*`), GDK/Win32 string-out helpers with exact truncation semantics, `under_wine`, `SendPtr`.
- `build.rs`: links xcl.exe with high-entropy ASLR off, and hashes the shader translator sources into `XCL_TRANSLATOR_SOURCE_ID` and `XCL_PER_LANE_SOURCE_ID` for cache identity.

## How it connects
- Uses: `loader` (load_main, run_main, on_tls_event, set_cache_dir), `resolver` (search dirs, records, classify), `stubs` (init_db, all_stubs), `eh` (hooks, crash handler, probes, thread dump), `tls::cleanup_cache` (all C02); `vfs` and `vfs_hooks` (C03); `hle::init`, `hle::registry`, `hle::kernelx::{fine_timer, adopt_title_hooks}` (C16); `gpu::adapter`, `gpu::agility` (C04); `gpu::recomp::dxc::{location, toolchain, cache_dir}` (C10); `gpu::dxil::coverage::dump_fallbacks` (C13); D3D12.X `present`, `bind`, `frame`, `real`, `vram`, `sparse` counters (C19–C24); `dstorage` (C25); `xaudio2` stats (C27); `kbm::release` (C29).
- Used by: `main.rs` only; the other subcommands (`disasm`, `shaders`, `rpf`) live in C03/C04.
- Global state: `state::BOOT`, `host::HOST_MODE`, logger statics, `report::{CRASH, WRITE_LOCK}`. Threads: optional `xcl-activity` (2 s activity and memory timeline), `xcl-thread-dump`, and an `XCL_RUN_SECONDS` timer thread that ends the title cleanly.

## Flow
1. `xcl run <folder|exe> [-- args]` → `launcher::run`.
2. `resolve_location`: canonicalise; if a file, the package root is the nearest parent (up to 5 levels) whose `MicrosoftGame.config` names that exe (Unreal titles keep exes in `<Game>/Binaries/XSX`). Title = sanitised `Identity Name`, else the folder name.
3. Exe choice: explicit path > `exe` in `xcl.toml` > the config's `<Executable>` > the only `.exe` at the top level, else up to 2 levels down. xcl.exe itself is never picked, whether matched by name, path or identical bytes.
4. Data folders created; config loaded and resolved; boot record written; previous log and report kept as `.prev`.
5. Warnings: no console imports (a plain Windows program?) or not an x64 PE. Then preflight output, and a hard stop if the CPU lacks Zen 2 extensions (AVX2, FMA, F16C, BMI1/2, LZCNT, POPCNT, MOVBE, SSE4.2), unless `XCL_SKIP_CPU_CHECK` is set.
6. `spawn_host`: the same xcl.exe sees `XCL_HOST` → `host_main` → the init sequence above → `loader::load_main` → `run_main` (C02).
7. After the host exits, the launcher reopens `report.json`, adds the exit code and preflight results, and prints crash, import counts, the top 15 called stubs, unmapped console paths and redirected writes.

## Design decisions and invariants
- **Two processes, one binary.** The launcher stays clean (preflight, summary, survives a crash). The host is a fresh process whose command line looks like the game's, and whose main image (xcl.exe) owns the TLS callback that sees every thread. Because the game shares xcl.exe's process image, xcl.exe deliberately avoids importing UI libraries and avoids an application manifest. The menu's comctl32 v6 goes through a temporary activation context and `GetProcAddress` (`main.rs:133-192`).
- **Game folder read-only; all state under the data root.** `{data}` holds logs, cache, overlay, saves, PLS and temp. Default mounts map `GAME:`, `G:`, `D:` to the game, `T:` to temp and `S:` to PLS (`config.rs:150-159`).
- **Defaults are written once, never over an unreadable file** (`load_or_create`, issue #106). New default mounts are merged into old files, and an old over-long console id is migrated (`config.rs:207-234`).
- **No XML or INI libraries.** `MicrosoftGame.config` is decoded (UTF-8 or BOM-marked UTF-16), comments are stripped, and tags and attributes are found by careful string scanning (`launcher.rs:19-125`).
- **Report robustness.** `write_final` is called after load, at process detach, from the crash filter, from `XCL_RUN_SECONDS` and at normal exit. Locks are taken with bounded `try_lock` loops because the holder may be the crashing thread (`report.rs:159`). The first crash message wins.
- **Logging never re-enters hooked file APIs**: the file handle is opened before the VFS hooks are installed.
- **Low virtual addresses.** `/HIGHENTROPYVA:NO` keeps the title's memory under 2^40, which a retail title's BVH builder needs of its GPU addresses (`build.rs:41-47`, refs #585 / PR #631).
- **Cache identity by source hash.** FNV-style hashes over the translator sources and the per-lane sources let shader caches drop stale entries automatically; `TRANSLATION_GLUE_REVISION` is bumped by hand for `pso.rs` glue changes (`build.rs:3-31`).
- Many behaviours are switched by environment variables (`XCL_DATA_ROOT`, `XCL_UMA_MB`, `XCL_ACTIVITY`, `XCL_THREAD_DUMP_AT`, `XCL_RUN_SECONDS`, `XCL_SKIP_CPU_CHECK`, `XCL_ADAPTER`, `XCL_RAYTRACING`, `XCL_WINDOW_POS`, `XCL_DXC_DIR`), usually overriding `xcl.toml`.
- Exit codes `0x7C10001` (boot record unreadable) and `0x7C10002` (loader failure) mark XCL's own failures, as opposed to the game's.

## Open questions
- How does `hle::init` use `Config` (aliases, identity, memory)? What does `kernelx::adopt_title_hooks` do after the main image is mapped? → C16
- How the VFS uses `mounts`, `own_folder` (shader cache) and the overlay; what `vfs_hooks::install` covers → C03
- Who reads `XCL_TRANSLATOR_SOURCE_ID` / `XCL_PER_LANE_SOURCE_ID` (the shader cache key)? → C10 (`dxc.rs`, `pso.rs`), C07
- Why do BVH GPU addresses have to be below 2^40? → C06 (`bvh.rs`); also `tests/low_va.rs` → C30
- With ASLR still on, how is a clash avoided between xcl.exe's own base and a non-relocatable game exe that wants the same preferred base (often 0x140000000)? → C02 shows only the failure path; maybe C30 (`low_va.rs`)
- Consumers of `title_memory_mb` (TitleMemoryStatus) and `uma_mb` (`gpu::uma`) → C16, C05
- `report.rs` notes say console-format shaders are replaced by fallback pipelines "until the RDNA2 recompiler is finished". Is that still true at this commit? → C10, C24
- `xbox_live_tokens` handling → C17/C18 (`xsign.rs`)

## Pointers
- `main.rs:16`: the `.CRT$XLY` TLS callback, the hook that makes manual TLS and DllMain thread notifications possible
- `main.rs:55`: host-mode branch via `XCL_HOST`
- `launcher.rs:225`: `config_root`, finding the package root above a deep exe
- `launcher.rs:288`: `run`, the whole launcher flow
- `launcher.rs:393`: `check_cpu_features`, the Zen 2 extension list
- `launcher.rs:433`: `spawn_host`
- `config.rs:80`: `Config::resolve`, manifest-driven defaults
- `config.rs:150`: default mounts
- `host.rs:106-143`: the host initialisation order (worth memorising)
- `report.rs:171`: `write_final` and its lock discipline
- `preflight.rs:57`: what counts as Error / Warning / Note for a run
- `build.rs:41`: `/HIGHENTROPYVA:NO`
- `util.rs:136`: `dll_key`, the normalisation used for every DLL lookup
