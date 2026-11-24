# PROGRESS

Pinned commit: `c4d37bec1bb4286eb373aea963404a23f13401fa` (xcl-dev/xcl, 2026-11-24).
Focus: architecture. Instructions for each run: `RUNBOOK.md`.

Mark a chunk `[x]` only once its note is committed. The line after each chunk lists its files.

## Chunks

- [x] **C01** `entry-config`: Entry point, launcher, config, reporting (13 files, ~3.0k lines)
  - files: `crates/xcl/build.rs`, `crates/xcl/src/config.rs`, `crates/xcl/src/host.rs`, `crates/xcl/src/inspect.rs`, `crates/xcl/src/launcher.rs`, `crates/xcl/src/lib.rs`, `crates/xcl/src/log.rs`, `crates/xcl/src/main.rs`, `crates/xcl/src/manifest.rs`, `crates/xcl/src/preflight.rs`, `crates/xcl/src/report.rs`, `crates/xcl/src/state.rs`, `crates/xcl/src/util.rs`
- [x] **C02** `pe-loader`: PE loading, import resolution, stubs, TLS, exceptions (8 files, ~4.3k lines)
  - files: `crates/xcl/src/eh.rs`, `crates/xcl/src/hooks.rs`, `crates/xcl/src/loader.rs`, `crates/xcl/src/modules.rs`, `crates/xcl/src/pe.rs`, `crates/xcl/src/resolver.rs`, `crates/xcl/src/stubs.rs`, `crates/xcl/src/tls.rs`
- [ ] **C03** `vfs-disasm`: Virtual file system, file hooks, RPF archives, disassembly (6 files, ~3.9k lines)
  - files: `crates/xcl/src/disasm.rs`, `crates/xcl/src/insn.rs`, `crates/xcl/src/rpf.rs`, `crates/xcl/src/rpf_probe.rs`, `crates/xcl/src/vfs.rs`, `crates/xcl/src/vfs_hooks.rs`
- [ ] **C04** `gpu-entry`: Shader front door and GPU basics (adapter, Agility, DXBC, fences, packs) (8 files, ~3.8k lines)
  - files: `crates/xcl/src/gpu/adapter.rs`, `crates/xcl/src/gpu/agility.rs`, `crates/xcl/src/gpu/dxbc.rs`, `crates/xcl/src/gpu/fence.rs`, `crates/xcl/src/gpu/mod.rs`, `crates/xcl/src/gpu/pack.rs`, `crates/xcl/src/gpu/shader.rs`, `crates/xcl/src/shaders.rs`
- [ ] **C05** `gpu-memory-textures`: GPU memory (UMA), descriptors, textures, GFX10 tiling (5 files, ~4.1k lines)
  - files: `crates/xcl/src/gpu/desc.rs`, `crates/xcl/src/gpu/texture.rs`, `crates/xcl/src/gpu/tiling.rs`, `crates/xcl/src/gpu/tiling_tables.rs`, `crates/xcl/src/gpu/uma.rs`
- [ ] **C06** `rdna2-isa`: RDNA2 instruction set, Xbox GPU specifics, BVH (5 files, ~3.6k lines)
  - files: `crates/xcl/src/gpu/bvh.rs`, `crates/xcl/src/gpu/rdna2.rs`, `crates/xcl/src/gpu/rdna2_spec.rs`, `crates/xcl/src/gpu/rdna2_xbox.rs`, `crates/xcl/src/gpu/xbox.rs`
- [ ] **C07** `recomp-core`: Shader recompiler core: module, SSA, wave lowering, lane masks (5 files, ~3.9k lines)
  - files: `crates/xcl/src/gpu/recomp/lane_masks.rs`, `crates/xcl/src/gpu/recomp/lower/wave.rs`, `crates/xcl/src/gpu/recomp/mod.rs`, `crates/xcl/src/gpu/recomp/ssa.rs`, `crates/xcl/src/gpu/recomp/x64_masks.rs`
- [ ] **C08** `recomp-lower`: Shader recompiler: lowering RDNA2 to IR (1 files, ~4.3k lines)
  - files: `crates/xcl/src/gpu/recomp/lower.rs`
- [ ] **C09** `recomp-emit`: Shader recompiler: emitting HLSL (1 files, ~5.9k lines)
  - files: `crates/xcl/src/gpu/recomp/emit.rs`
- [ ] **C10** `recomp-backends`: Recompiler back ends: DXC, DXIL lowering, pipeline state objects (5 files, ~3.4k lines)
  - files: `crates/xcl/src/gpu/recomp/dxc.rs`, `crates/xcl/src/gpu/recomp/dxil/assemble.rs`, `crates/xcl/src/gpu/recomp/dxil/mod.rs`, `crates/xcl/src/gpu/recomp/lower_dxil.rs`, `crates/xcl/src/gpu/recomp/pso.rs`
- [ ] **C11** `recomp-runner-mesh`: Recompiler runner, upper lanes, mesh shaders (4 files, ~3.5k lines)
  - files: `crates/xcl/src/gpu/recomp/mesh_dispatch.rs`, `crates/xcl/src/gpu/recomp/mesh_wave.rs`, `crates/xcl/src/gpu/recomp/runner.rs`, `crates/xcl/src/gpu/recomp/upper_lanes.rs`
- [ ] **C12** `dxil-container`: Direct DXIL: bitstream, container, module, ops, validation (9 files, ~4.5k lines)
  - files: `crates/xcl/src/gpu/dxil/bitstream.rs`, `crates/xcl/src/gpu/dxil/container.rs`, `crates/xcl/src/gpu/dxil/mod.rs`, `crates/xcl/src/gpu/dxil/module.rs`, `crates/xcl/src/gpu/dxil/module/opt.rs`, `crates/xcl/src/gpu/dxil/ops.rs`, `crates/xcl/src/gpu/dxil/ops_core.rs`, `crates/xcl/src/gpu/dxil/prelude.rs`, `crates/xcl/src/gpu/dxil/validate.rs`
- [ ] **C13** `dxil-shader`: Direct DXIL: shader, CFG, SSA, kernels, graphics, coverage, lanes (7 files, ~3.8k lines)
  - files: `crates/xcl/src/gpu/dxil/cfg.rs`, `crates/xcl/src/gpu/dxil/coverage.rs`, `crates/xcl/src/gpu/dxil/graphics.rs`, `crates/xcl/src/gpu/dxil/kernel.rs`, `crates/xcl/src/gpu/dxil/lanes.rs`, `crates/xcl/src/gpu/dxil/shader.rs`, `crates/xcl/src/gpu/dxil/ssa.rs`
- [ ] **C14** `dxil-lower-a`: Direct DXIL lowering, part 1 (core, wide, window, graphics) (4 files, ~3.8k lines)
  - files: `crates/xcl/src/gpu/dxil/lower.rs`, `crates/xcl/src/gpu/dxil/lower_gfx.rs`, `crates/xcl/src/gpu/dxil/lower_wide.rs`, `crates/xcl/src/gpu/dxil/lower_window.rs`
- [ ] **C15** `dxil-lower-b`: Direct DXIL lowering, part 2 (memory, VALU, images) (3 files, ~4.5k lines)
  - files: `crates/xcl/src/gpu/dxil/lower_image.rs`, `crates/xcl/src/gpu/dxil/lower_mem.rs`, `crates/xcl/src/gpu/dxil/lower_valu.rs`
- [ ] **C16** `hle-kernelx`: HLE core and kernelx (console kernel library) (6 files, ~4.2k lines)
  - files: `crates/xcl/src/hle/kernelx.rs`, `crates/xcl/src/hle/kernelx/lazy.rs`, `crates/xcl/src/hle/kernelx/placeholders.rs`, `crates/xcl/src/hle/mod.rs`, `crates/xcl/src/hle/netoff.rs`, `crates/xcl/src/hle/user.rs`
- [ ] **C17** `xgameruntime-a`: XGameRuntime, part 1 (dispatch, API, GDK table, accessibility) (4 files, ~3.8k lines)
  - files: `crates/xcl/src/hle/xgameruntime/accessibility.rs`, `crates/xcl/src/hle/xgameruntime/api.rs`, `crates/xcl/src/hle/xgameruntime/gdk_table.rs`, `crates/xcl/src/hle/xgameruntime/mod.rs`
- [ ] **C18** `xgameruntime-b`: XGameRuntime, part 2 (saves, networking, task queues, async, signing) (5 files, ~4.7k lines)
  - files: `crates/xcl/src/hle/xgameruntime/gamesave.rs`, `crates/xcl/src/hle/xgameruntime/networking.rs`, `crates/xcl/src/hle/xgameruntime/taskqueue.rs`, `crates/xcl/src/hle/xgameruntime/xasync.rs`, `crates/xcl/src/hle/xgameruntime/xsign.rs`
- [ ] **C19** `d3d12x-device`: D3D12.X: device, DXGI, fences, layouts, guards, pools, null device (10 files, ~4.0k lines)
  - files: `crates/xcl/src/hle/d3d12_layout.rs`, `crates/xcl/src/hle/d3d12x/alloc_pool.rs`, `crates/xcl/src/hle/d3d12x/device_x.rs`, `crates/xcl/src/hle/d3d12x/dxgi_device.rs`, `crates/xcl/src/hle/d3d12x/fence_x.rs`, `crates/xcl/src/hle/d3d12x/guard.rs`, `crates/xcl/src/hle/d3d12x/late_release.rs`, `crates/xcl/src/hle/d3d12x/layout.rs`, `crates/xcl/src/hle/d3d12x/mod.rs`, `crates/xcl/src/hle/d3d12x/null.rs`
- [ ] **C20** `d3d12x-real`: D3D12.X: the real (PC D3D12) backend (1 files, ~6.5k lines)
  - files: `crates/xcl/src/hle/d3d12x/real.rs`
- [ ] **C21** `d3d12x-list`: D3D12.X: command lists (1 files, ~6.2k lines)
  - files: `crates/xcl/src/hle/d3d12x/list_x.rs`
- [ ] **C22** `d3d12x-binding`: D3D12.X: binding, console heaps and lists, CPU writes (5 files, ~3.9k lines)
  - files: `crates/xcl/src/hle/d3d12x/bind.rs`, `crates/xcl/src/hle/d3d12x/console_heap.rs`, `crates/xcl/src/hle/d3d12x/console_list.rs`, `crates/xcl/src/hle/d3d12x/cpu_writes.rs`, `crates/xcl/src/hle/d3d12x/title_list.rs`
- [ ] **C23** `d3d12x-memory`: D3D12.X: VRAM and sparse resources (2 files, ~3.7k lines)
  - files: `crates/xcl/src/hle/d3d12x/sparse.rs`, `crates/xcl/src/hle/d3d12x/vram.rs`
- [ ] **C24** `d3d12x-frame`: D3D12.X: title textures, frames, present, async PSOs, shader cache (5 files, ~4.8k lines)
  - files: `crates/xcl/src/hle/d3d12x/frame.rs`, `crates/xcl/src/hle/d3d12x/present.rs`, `crates/xcl/src/hle/d3d12x/pso_async.rs`, `crates/xcl/src/hle/d3d12x/shader_cache_session.rs`, `crates/xcl/src/hle/d3d12x/title_textures.rs`
- [ ] **C25** `dstorage-media`: DirectStorage, DXC shim, Media Foundation (4 files, ~3.6k lines)
  - files: `crates/xcl/src/hle/dstorage.rs`, `crates/xcl/src/hle/dstorage/parallel.rs`, `crates/xcl/src/hle/dxcompiler.rs`, `crates/xcl/src/hle/mfplat.rs`
- [ ] **C26** `bcpack-xg`: BCPack texture decompression and XG (texture layout library) (4 files, ~3.6k lines)
  - files: `crates/xcl/src/hle/bcpack/mod.rs`, `crates/xcl/src/hle/bcpack/tables.rs`, `crates/xcl/src/hle/bcpack/victory.rs`, `crates/xcl/src/hle/xg.rs`
- [ ] **C27** `audio-a`: Audio, part 1: XAudio2 and ACP HAL (2 files, ~3.1k lines)
  - files: `crates/xcl/src/hle/acphal.rs`, `crates/xcl/src/hle/xaudio2.rs`
- [ ] **C28** `audio-b`: Audio, part 2: XMA2 and Opus decoders (3 files, ~4.8k lines)
  - files: `crates/xcl/src/hle/opus.rs`, `crates/xcl/src/hle/xma2/mod.rs`, `crates/xcl/src/hle/xma2/tables.rs`
- [ ] **C29** `input`: Input: GameInput, keyboard and mouse (2 files, ~2.0k lines)
  - files: `crates/xcl/src/hle/gameinput.rs`, `crates/xcl/src/hle/kbm.rs`
- [ ] **C30** `integration-tests`: Integration tests (skim: what they prove about the design) (16 files, ~5.5k lines)
  - files: `crates/xcl/tests/bcpack_parity.rs`, `crates/xcl/tests/console_dlls_e2e.rs`, `crates/xcl/tests/console_shaders.rs`, `crates/xcl/tests/disasm_e2e.rs`, `crates/xcl/tests/e2e.rs`, `crates/xcl/tests/gpu_compute_e2e.rs`, `crates/xcl/tests/gpu_pso_e2e.rs`, `crates/xcl/tests/heap_probe.rs`, `crates/xcl/tests/low_va.rs`, `crates/xcl/tests/pls_quota.rs`, `crates/xcl/tests/predication_e2e.rs`, `crates/xcl/tests/rpf_e2e.rs`, `crates/xcl/tests/shaders_e2e.rs`, `crates/xcl/tests/sparse_reinit.rs`, `crates/xcl/tests/texture_states.rs`, `crates/xcl/tests/xg_oracle.rs`
- [ ] **C31** `harness-tooling`: Test title, test DLL, scripts and CI (26 files, ~4.5k lines)
  - files: `.github/workflows/ci.yml`, `.github/workflows/pr-check.yml`, `scripts/gen_bcpack_victory_tables.py`, `scripts/gen_d3d12_layout.py`, `scripts/gen_gdk_table.py`, `scripts/gen_rdna2.py`, `scripts/gen_tiling.py`, `scripts/pr_check.py`, `scripts/re/bvhb_shaders.py`, `scripts/re/check_layout.py`, `scripts/re/pe_exports.py`, `scripts/re/record_writers.py`, `scripts/smoke-title.tests.ps1`, `scripts/smoke.ps1`, `scripts/test-native-stderr.tests.ps1`, `scripts/test.ps1`, `scripts/test_bvhb_shaders.py`, `scripts/test_pe_exports.py`, `scripts/test_pr_check.py`, `scripts/test_record_writers.py`, `tests/testgamedll/Cargo.toml`, `tests/testgamedll/src/lib.rs`, `tests/testtitle/Cargo.toml`, `tests/testtitle/src/audio.rs`, `tests/testtitle/src/gpu.rs`, `tests/testtitle/src/main.rs`

## Synthesis

- [ ] **S1** Write `ARCHITECTURE.md` from the notes (no source reading beyond spot checks)

## Run log

<!-- one line per run: date, chunks done, anything left half-done -->
- 2026-11-24: C01, C02 done (7.3k lines read). Unclear: how a non-relocatable game exe avoids xcl.exe's own image base (see C02 open questions).
