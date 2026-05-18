# CLAUDE.md — Vortex GPGPU

## About the human you're working with

The user is preparing to contribute to **Vortex** (open-source RISC-V GPGPU) and is in a **deep-learning phase**: they want to understand each module and microarchitectural decision thoroughly, not just get tasks done. They have general GPU microarchitecture background (SIMT, warps, threads, caches, memory hierarchies) but are new to Vortex's specific implementation.

**What this means for how you should respond:**

- Default to **explanatory depth**, not minimal answers. When asked about a module, walk through the *why* of design choices, not just *what* the code does.
- Reconcile docs vs. RTL when they disagree. The high-level docs in [docs/](docs/) are useful but sometimes lag behind the RTL or omit critical details (arbitration priorities, MSHR coalescing, pipeline depths, deadlock-avoidance signals). Always cross-check against the actual SystemVerilog.
- Use precise terms (write-through vs. write-back, MSHR allocation vs. finalization, replay vs. fill, etc.) and define them on first use.
- Cite **specific files and line numbers** as `[filename.sv:N](hw/rtl/.../filename.sv#LN)` so the user can jump straight to source.
- When the user shows a partial mental model, **point out which parts are right and which are wrong** rather than re-explaining from scratch. They're calibrating, not starting from zero.
- Prefer concrete signal names from the RTL over generic textbook language. The user is going to read this code.
- Don't shy away from "this is more complex than the diagram suggests" — that's exactly the kind of insight they need.

## Project at a glance

Vortex is a configurable, multi-core, multi-warp, multi-thread RISC-V GPGPU (RV32IMAF / RV64IMAFD) targeting FPGAs (Altera & Xilinx) and simulation. It implements the SIMT model with a custom RISC-V ISA extension for thread-mask control, warp spawning, control-flow divergence (SPLIT/JOIN/PRED), and barriers.

Key configurability knobs: number of cores, warps/core, threads/warp, ALU/FPU/LSU/SFU units, issue width, local mem, L1/L2/L3 caches.

## Directory map (RTL focus)

- [hw/rtl/](hw/rtl/) — all SystemVerilog sources
  - [core/](hw/rtl/core/) — 6-stage pipeline (Schedule, Fetch, Decode, Issue, Execute, Commit), scoreboard, IPDOM stack, warp scheduler, functional units (ALU/FPU/LSU/SFU)
  - [cache/](hw/rtl/cache/) — banked non-blocking write-through/writeback cache used for I-cache, D-cache, L2, L3 (same module, parameterized)
  - [mem/](hw/rtl/mem/) — memory subsystem glue, arbiters
  - [fpu/](hw/rtl/fpu/) — FP unit wrappers (uses external cvfpu)
  - [interfaces/](hw/rtl/interfaces/) — `VX_mem_bus_if` and other inter-module SV interfaces
  - [libs/](hw/rtl/libs/) — reusable building blocks: `VX_stream_xbar`, `VX_stream_omega`, `VX_stream_arb`, `VX_elastic_buffer`, `VX_fifo_queue`, `VX_pipe_register`, `VX_pending_size`, etc. These show up everywhere — learn them once.
  - [Vortex.sv](hw/rtl/Vortex.sv), [VX_cluster.sv](hw/rtl/VX_cluster.sv), [VX_socket.sv](hw/rtl/VX_socket.sv) — top-level hierarchy: Vortex → Clusters (share L2) → Sockets (share L1) → Cores
  - [VX_config.vh](hw/rtl/VX_config.vh), [VX_define.vh](hw/rtl/VX_define.vh), [VX_gpu_pkg.sv](hw/rtl/VX_gpu_pkg.sv) — parameters, macros, package types
- [hw/syn/](hw/syn/) — FPGA synthesis flows (altera, xilinx) and ASIC (synopsys, yosys)
- [hw/unittest/](hw/unittest/) — module-level RTL tests (e.g. `cache_top`)
- [sim/](sim/) — three simulator backends: `simX` (C++ cycle-approximate, for fast prototyping), `rtlsim` (Verilator-driven RTL), `opaesim` (Intel OPAE AFU sim)
- [runtime/](runtime/) — host drivers: simx, rtlsim, opae, xrt, stub
- [kernel/](kernel/) — GPU-side runtime APIs linked into kernel binaries
- [tests/](tests/) — riscv conformance, kernel, regression, opencl
- [ci/](ci/), [docs/](docs/), [miscs/](miscs/)

## Where the docs live (and what to trust)

- [docs/microarchitecture.md](docs/microarchitecture.md) — pipeline + SIMT model + ISA extension. Good high-level overview, accurate.
- [docs/cache_subsystem.md](docs/cache_subsystem.md) — accurate but **abbreviated**; for real understanding read the RTL alongside it (the 5-way arbiter, MSHR allocate/finalize/replay flow, and the fact that "Core Rsp Merger / DRAM Req Arbiter" is *two independent structures* are not obvious from the doc alone).
- [docs/codebase.md](docs/codebase.md) — directory map.
- [docs/coding_guidelines_verilog.md](docs/coding_guidelines_verilog.md), [docs/coding_guidelines_cpp.md](docs/coding_guidelines_cpp.md) — style. Read before submitting any contribution.
- [docs/contributing.md](docs/contributing.md) — contributor process.
- [docs/debug_mode.md](docs/debug_mode.md), [docs/debugging.md](docs/debugging.md) — wave-dumps, scope, trace macros (`DBG_TRACE_*`).
- [docs/simulation.md](docs/simulation.md), [docs/testing.md](docs/testing.md) — running the sims and tests.
- [docs/environment_setup.md](docs/environment_setup.md), [docs/install_vortex.md](docs/install_vortex.md), [docs/fpga_setup.md](docs/fpga_setup.md) — build infra.

## Conventions worth knowing up-front

- **Signal naming**: `_st0`/`_st1` are pipeline stage suffixes, `_sel` is the pre-pipeline mux output, `valid`/`ready` is standard ready/valid handshake, `*_fire = valid && ready`.
- **Trace macros**: `\`TRACE(level, msg)` for debug prints (gated by `DBG_TRACE_*` defines). Use them when reading RTL — grepping for trace lines is the fastest way to learn what a block does.
- **Performance counters**: anything inside `\`ifdef PERF_ENABLE` is observability infrastructure, not functional logic — skip on first reads.
- **`UUID_WIDTH`**: optional per-request unique ID for debugging in-flight requests through the pipeline. Often `0` in production builds.
- **Almost-full backpressure**: deadlock avoidance throughout — queues stop accepting at *almost-full*, not full, leaving slots for requests already in-flight.

## When the user asks "explain X module"

The expected pattern:
1. Read the relevant `.sv` file end-to-end (or in chunks for the long ones — bank pipeline is ~770 lines).
2. Cross-reference the matching doc.
3. Identify any place the doc undersells or the diagram misleads.
4. Explain with cited line numbers, define terms, and call out design rationale (why this priority, why this buffering, why this almost-full).
5. End with a "quick mental model" paragraph the user can re-derive without re-reading the code.
