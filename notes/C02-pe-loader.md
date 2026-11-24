# C02 PE loading, import resolution, stubs, TLS, exceptions

Files: `crates/xcl/src/eh.rs`, `src/hooks.rs`, `src/loader.rs`, `src/modules.rs`, `src/pe.rs`, `src/resolver.rs`, `src/stubs.rs`, `src/tls.rs` · ~4.3k lines · commit c4d37be

## Purpose
This is XCL's own Windows loader (design §5.1/§5.2). It maps the console game's x64 PE files into the xcl.exe host process without the Windows loader, binds every import to an HLE function, a game DLL, a real Windows DLL or a generated logging stub, and recreates what the OS loader would normally provide: static TLS, DllMain and TLS-callback notifications, unwind data for C++ exceptions, and crash diagnostics. It also emulates the console CPU's AMD-only instructions when the PC CPU lacks them.

## Main pieces
- `Pe` (`src/pe.rs:193`): a bounds-checked, allocation-light PE32+ parser over raw file bytes (`Layout::File`) or a mapped image (`Layout::Mapped`). It reads imports, delay imports (including old VA-based descriptors), relocations, TLS directory and exports. Malformed data becomes a `PeError`, never a panic or a runaway loop (#106). `OwnedExports` (`pe.rs:599`) keeps exports with forwarders decoded.
- `loader::map_image_ex` (`src/loader.rs:64`): the 8-step mapping. (1) reserve+commit at the preferred base, else anywhere if relocatable; (2) copy headers and sections; (3) apply DIR64/HIGHLOW relocations; register the module; (4) resolve all imports eagerly, delay-loads included, into the IAT; (5) set up static TLS; (6) set per-section page protections and flush the I-cache; (7) `RtlAddFunctionTable` for `.pdata`; (8) hook `__report_gsfailure` in executable sections. It does not run any image code.
- `loader::load_game_dll` / `init_dll` (`loader.rs:291`, `:257`): map plus TLS callbacks (PROCESS_ATTACH) plus `DllMain`. Idempotent by name. A DLL whose DllMain fails is unregistered, so a later load retries, as Windows does.
- `loader::run_main` (`loader.rs:337`): writes the game's base into `PEB.ImageBaseAddress` (skipped under Wine), runs the exe's TLS callbacks, then starts the entry point on a new thread with the stack reserve from the PE header, passing the PEB as the argument. It waits for that thread.
- `loader::on_tls_event` (`loader.rs:384`): called from xcl.exe's TLS callback (C01). THREAD_ATTACH fills TLS blocks, then runs TLS callbacks and DllMain(2) in load order. THREAD_DETACH does the same in reverse. PROCESS_DETACH writes the final report.
- `modules` (`src/modules.rs`): registry of `MappedModule`s (name, base, size, exports, TLS callbacks, `initialized`, `no_thread_calls`, stack reserve). A **lock-free fixed array of 512 ranges** serves address→module lookups from exception paths (`modules.rs:41-77`). `hle_handle` (`:111`) makes a fake HMODULE for an HLE DLL: a committed page that starts with "MZ". `describe_address` (`:152`) gives `module+offset` and is made safe for arbitrary stack words.
- `resolver` (`src/resolver.rs`): the import policy. `resolve_inner_opt` (`:211`) applies, in order: a config/registry route (HLE module → that module's exports → any other HLE module → system fallback DLLs; or `@system`; or `@stub`); an already mapped game DLL; a game DLL found in the search dirs (mapped on demand); a real Windows DLL; a known HLE function name imported from an unexpected DLL; and finally an auto-stub. `Source` (`:24`) records which of these won, for the report and `xcl inspect`.
- `resolver::load_library` / `get_proc_address` (`:370`, `:414`): the game-facing LoadLibrary and GetProcAddress, used by kernelx shims. GetProcAddress returns NULL rather than a stub, so a game probing for optional APIs sees them as absent. Kernel-family modules get the real module handle, but GetProcAddress on them still prefers XCL's overrides.
- `stubs` (`src/stubs.rs`): `make_stub_info` (`:116`) writes a 48-byte x64 thunk into an RWX arena (`alloc_code`, `:84`). The thunk passes its `StubInfo*` and the caller's return address to `stub_report`, which counts the call, logs the first one with its caller, runs an optional hook, and returns the configured value. `StubDb` (`:176`) holds return values per function, per module default (with an alias→canonical-module fallback) and ordinal→name maps. It comes from `data/stubdb.toml` (61 lines: `module_defaults`, `functions`, and ordinals for gameinput, xaudio2_9 and d3d12_x), merged with `stub_returns` from `xcl.toml`.
- `tls` (`src/tls.rs`): the **TLS carrier** (see below). `install` (`:451`), `on_thread_attach` (`:385`), `take_slot_zero` (`:310`).
- `eh` (`src/eh.rs`): exception support and diagnostics. `register_function_table` (`:20`); a `RtlPcToFileHeader` detour (`:34`) so MSVC `throw`/`catch` can find the image base of mapped modules; an unhandled-exception filter (`:64`) that logs registers (with vtable owners), stack callers and the report, then chains to the title's own filter; an `NtTerminateProcess` detour (`:128`) that logs how the process ends; a UCRT invalid-parameter handler (`:536`); a vectored handler (`:440`) that names threads from the debugger exception 0x406D1388, logs the first 16 C++ exceptions thrown by title code, and emulates console-only instructions (`emulate_console_only`, `:315`).
- `hooks::hook` (`src/hooks.rs:13`): a thin `retour::RawDetour` wrapper. The detour is leaked on purpose and the trampoline address is stored in an `AtomicUsize`.

## How it connects
- Uses: `hle::registry()` with `Route`, `HleModule` (`exports`, `ordinals`, `fallback`, `SELF`, `aliases_are_system`, `find_any`, `hle_for`, `module_by_name`) → C16; `vfs` (`real_path_for_read`, `is_game_path`) → C03; `insn::decode`/`ConsoleOnly` → C03; `gpu::shader::readable` → C04; `gpu::uma::in_committed_window` → C05; `hle::d3d12x::guard::make_call_tracer` → C19; `d3d12x::real::log_device_removed` → C20; `report`, `log`, `state` → C01.
- Used by: `host_main` (C01) calls `load_main`, `run_main`, `set_cache_dir`, `set_search_dirs`, `install_pc_hook`, `install_crash_handler`, `start_probes`, `stubs::init_db`; `report::build` reads `resolver::records`, `stubs::all_stubs` and `modules::mapped_modules`; `inspect` uses `Pe` and `classify`; kernelx (C16) is expected to call `load_library`, `get_proc_address`, `set_title_filter`, and set `no_thread_calls`. `stubs::new_info`/`alloc_code` are reused by D3D12.X slot guards (C19).
- Global state: `MAPPED`/`MAIN`/range arrays, `HLE_HANDLES`, `RECORDS`, `SEARCH_DIRS`, `SYS_CACHE`, the stub arena and `BY_KEY`, `tls::IMAGES`/`XCL_SLOT`/`MOVED`, `LOAD_LOCK` plus the thread-local `IN_LOAD`, and the hook trampolines. Threads: `xcl-watch` and `xcl-probe`, both for diagnostics only.

## Flow
1. `load_main(exe)` takes the load guard. `map_image_ex(is_main=true)` reads the file, parses it and maps it (steps 1–3), registers the module and sets it as main, so DLLs initialised during import resolution can already ask `GetModuleHandle(NULL)`.
2. For each import, `resolve_import` normalises the DLL with `dll_key` and runs the policy. A game DLL reached here is mapped and initialised recursively. The IAT slot is written. Under `XCL_TRACE_IMPORTS` the address is wrapped in a call tracer.
3. Static TLS: `tls::install` writes a carrier DLL to `cache\xcl_tls_<pid>_<n>_<name>.dll` and loads it with `LoadLibraryW`. It reads the slot index that Windows wrote into the carrier, stores it in the game's `_tls_index`, and copies the template into every existing thread's block. For the main exe, `take_slot_zero` then swaps so the title owns TLS slot 0.
4. Protections, `.pdata` registration and the GS-failure hook. Back in `host_main`, kernelx adopts title hooks and the report is written.
5. `run_main` patches the PEB, runs TLS callbacks, sets `READY`, and starts the entry thread. Every later thread start goes through xcl.exe's TLS callback → `on_tls_event`.

## Design decisions and invariants
- **The TLS carrier.** Windows only allocates static-TLS blocks for modules it loaded itself. XCL synthesises a tiny real DLL whose TLS template is at least as large as the game's (`build_carrier_dll_with`, `tls.rs:86`), so Windows owns and manages a slot for all threads, present and future. Two findings are hard-coded from Windows 11 experiments: the carrier needs an import table (it imports `KERNEL32!GetTickCount`) or ntdll ignores its TLS directory, and `SizeOfZeroFill` is ignored for dynamically loaded DLLs, so the whole size is put in the initialised range (`tls.rs:17-23`). Carrier files are deleted at the next launch (`cleanup_cache`).
- **Slot 0 for the title exe.** Code built with `/GA` reads `ThreadLocalStoragePointer[0]` directly (Fortnite's thread-safe statics). XCL moves its own TLS to the carrier's slot k and gives slot 0 to the title. Running threads first get xcl.exe's block in both slots; after a 20 ms grace sleep, slot 0 gets the title's block. New threads swap in `exchange_for_new_thread`. A `MOVED` list stops threads from being swapped twice (`tls.rs:303-371`).
- **A thread callback in xcl.exe replaces loader notifications.** Mapped DLLs never sit on the loader list, so DllMain(THREAD_ATTACH/DETACH) and TLS callbacks are driven from xcl.exe's `.CRT$XLY` callback, gated by `READY`.
- **Fail like Windows.** A broken import table frees the image and unregisters it; a failing DllMain unregisters it but keeps its memory, since its code may have left threads or callbacks behind (`loader.rs:160-170`, `:301-307`). Unit tests check both cases.
- **Re-entrant loads.** A global `LOAD_LOCK` is held only by the outermost load on a thread; nested loads during import resolution use a thread-local depth counter, which avoids self-deadlock. Modules are registered *before* import resolution so cyclic game-DLL imports terminate.
- **Lock-free paths for exception time.** `image_base_for_addr` reads atomics only. `describe_address` uses `try_read`. Memory reads in handlers go through `readable` checks page by page (`bounded_cstr`). Thread dumps hold a thread suspended only while reading its context, then unwind after resuming it, so the dump cannot deadlock on the loader or heap lock (#106).
- **Stub semantics.** Unknown functions never crash at load time. They return a configured value: E_NOTIMPL for xgameruntime and d3d12 aliases, 0 for the kernel family. Each stub logs once, with the caller. Raw ordinals are only resolved in their own DLL (`resolver.rs:129`). Forwarders are followed up to depth 8, splitting on the *last* dot so dotted DLL names work.
- **Console CPU emulation.** The launcher refuses PCs without the Zen 2 base extensions (C01). The vectored handler emulates and steps over AMD-specific instructions that a PC CPU may lack: CLZERO (single, and a recognised `clzero/add/cmp/jb` loop done in one step with correct EFLAGS), RDPRU (from QPC), RDPID, MONITORX/MWAITX, and SSE4a EXTRQ/INSERTQ (high quadword zeroed to match Zen 5). CLWB is patched to NOPs in place. It acts only for addresses inside mapped title images (`eh.rs:315-438`).
- **Making silent deaths loud.** `__fastfail` paths (stack-cookie failure, CRT invalid parameter, uncaught C++ via abort) leave no trace by default. XCL pattern-matches MSVC's `__security_check_cookie` (PC and console variants with `bnd` prefixes) to hook `__report_gsfailure`, installs a CRT invalid-parameter handler, logs first-chance C++ exceptions, and logs `NtTerminateProcess` with a full thread dump when the status is non-zero.
- **Debug-register diagnostics.** `XCL_WATCH=rva` sets a write watchpoint on DR0. `XCL_PROBE=rva,...` sets up to 3 execute breakpoints on DR1–3, logging the arguments. Both arm every thread by suspend/set-context polling, and arm the current thread with a private exception code (`eh.rs:705-971`). `XCL_TRACE_FAULTS` logs the first 32 access violations as they happen.

## Open questions
- What `HleModule.fallback`, `SELF`, `ordinals` and `aliases_are_system()` hold for each module, and how `Route` is built from config aliases → C16 (`hle/mod.rs`)
- Do kernelx's `DisableThreadLibraryCalls`, `SetUnhandledExceptionFilter`, `GetModuleHandle`/`GetModuleFileName` and `FreeLibrary` map onto `no_thread_calls`, `set_title_filter`, `modules::main_module` and (no unload visible here)? → C16
- Who sets `StubInfo.hook` (the "present heuristics")? → C19 / C24
- `make_call_tracer` and how D3D12.X slot guards reuse `stubs::new_info` → C19 (`guard.rs`)
- How is a clash avoided with a non-relocatable game exe whose preferred base overlaps xcl.exe's own image (`loader.rs:74-79` only errors out)? → C30 (`low_va.rs`), maybe the design doc
- Generated thunks (stubs, GS thunk) have no unwind info; does stack walking through a stub frame matter in practice? → not obviously answered; check C19/C30 for guard thunks
- `insn::decode` coverage and the `ConsoleOnly` variants → C03 (`insn.rs`)
- Is the 20 ms grace sleep in `take_slot_zero` the only protection for TLS accesses in flight? Is it ever revisited? → maybe `HANDOFF.md` (grep "slot 0")

## Pointers
- `loader.rs:64`: `map_image_ex`, the whole mapping pipeline in numbered steps
- `loader.rs:337`: `run_main`, PEB patch and the entry thread
- `loader.rs:384`: `on_tls_event`, replacement thread notifications
- `resolver.rs:211`: `resolve_inner_opt`, the import policy in one function
- `resolver.rs:106`: `resolve_hle`, own module → cross-HLE → system fallback
- `resolver.rs:414`: `get_proc_address`, the "NULL, not stub" rule
- `stubs.rs:1-14` and `:116`: the stub thunk layout and how it is emitted
- `tls.rs:1-23`: why the TLS carrier exists (read first)
- `tls.rs:310`: `take_slot_zero`
- `modules.rs:41-77`: lock-free address ranges
- `eh.rs:34`: `RtlPcToFileHeader` detour (C++ exceptions in mapped images)
- `eh.rs:315`: `emulate_console_only`
- `eh.rs:187`: `hook_gs_failure`
- `pe.rs:490`: hardened relocation parsing
