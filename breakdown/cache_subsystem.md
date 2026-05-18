# Vortex Cache Subsystem — Deep-dive Breakdown

> Reading order: skim this alongside [docs/cache_subsystem.md](../docs/cache_subsystem.md), then read the RTL in [hw/rtl/cache/](../hw/rtl/cache/). The diagram in the official docs is correct but compresses several distinct structures into single blocks, which causes confusion on first read.

## TL;DR mental model

> Thread requests → address-decoded into banks → per-bank: arbitrate {init, replay, fill, flush, new req} → tag check → on hit, read/write data array and respond; on miss, keep an MSHR entry, issue a fill, wait. When DRAM responds → demux back to the right bank → enters arbiter as a fill → writes data array → wakes up all MSHR entries for that line as replays → those re-walk the pipe and now hit. **Two output streams leave the cache independently**: core responses (back to threads) and memory requests (out to DRAM).

## Common misreadings of the diagram

1. **"Core Response Merger / DRAM Request Arbiters" is a single block.** No — it's *two separate, independent* structures sharing a visual label. One routes responses back to the thread that asked; the other funnels miss/writeback requests out to memory. They never interact.
2. **"DRAM Rsp goes straight to the arbiter."** No — it first lands in an elastic buffer (`mem_rsp_queue`, sized `MRSQ_SIZE`), then a `VX_stream_omega` demuxes responses to the right bank using the bank-id bits embedded in the memory tag.
3. **"MSHR is just where misses get noted, then replayed."** Half right. MSHR slots are *allocated up-front for every core request* (before hit/miss is known), then either released on hit or kept on miss. The "replay" only happens after the fill response arrives — that's how miss coalescing works: many misses to the same line share one DRAM fill and replay in order.
4. **"Arbiter is a 2-input mux."** It's a 5-way priority arbiter (init > replay > fill > flush > creq).

## The structures, left to right

### 1. Core Request Bank Select (input crossbar)

[VX_cache.sv:323-346](../hw/rtl/cache/VX_cache.sv#L323-L346) — `VX_stream_xbar` of size `NUM_REQS → NUM_BANKS`.

- Bank selection is pure address decoding: bits `[WORD_SEL_BITS +: BANK_SEL_BITS]` of the word address pick the bank ([VX_cache.sv:297-303](../hw/rtl/cache/VX_cache.sv#L297-L303)).
- "Bank collisions resolved with stalls" = if two thread requests in the same cycle hash to the same bank, one wins arbitration and the other is held with backpressure.
- The "FIFO" you see in the diagram on the bank input is actually the xbar's `OUT_BUF` skid/elastic buffer, not a deep FIFO.

### 2. DRAM Response demux

[VX_cache.sv:151-218](../hw/rtl/cache/VX_cache.sv#L151-L218).

- `mem_rsp_queue` (elastic buffer, `MRSQ_SIZE`) per memory port.
- `VX_stream_omega` (`MEM_PORTS → NUM_BANKS`) routes each response back to the bank that issued the original miss — identified by bank-id bits stamped into the memory tag on the way out.

### 3. Flush Unit

[VX_cache_flush.sv](../hw/rtl/cache/VX_cache_flush.sv), wired in [VX_cache.sv:124-138](../hw/rtl/cache/VX_cache.sv#L124-L138).

Two responsibilities:
- Post-reset **tag-array initialization** walk (tag SRAMs come up undefined; this clears them).
- Explicit **cache flushes** — invalidate-all, or writeback-and-invalidate when `WRITEBACK=1`.

### 4. Inside one bank (the yellow box)

#### 5-way priority arbiter

[VX_cache_bank.sv:207-234](../hw/rtl/cache/VX_cache_bank.sv#L207-L234). Priority order:

| # | Source | Why this priority |
|---|--------|-------------------|
| 1 | **init** | Must finish tag init before any traffic |
| 2 | **replay** (MSHR dequeue) | Drain pending misses to free MSHR slots before admitting more |
| 3 | **fill** (memory response) | Must land fill data before any replay of that line can hit |
| 4 | **flush** | Sequenced ahead of new traffic during flush ops |
| 5 | **core_req** | New thread traffic — lowest priority |

The replay > core_req ordering matters: once a fill lands, you want to drain every pending miss for that line *before* admitting fresh traffic, otherwise the MSHR fills up and stalls the pipe.

#### 2-stage pipeline

- **st0 = Tag Access** ([VX_cache_bank.sv:363-388](../hw/rtl/cache/VX_cache_bank.sv#L363-L388)) — read tag array, compute hit/miss (via `tag_matches_st0`), pick replacement victim for fills (via `VX_cache_repl`).
- **st1 = Data Access** ([VX_cache_bank.sv:433-462](../hw/rtl/cache/VX_cache_bank.sv#L433-L462)) — read or write the data array using the way decided in st0.

State and decisions from st0 are carried to st1 via `VX_pipe_register` instances (`pipe_reg0`, `pipe_reg1`).

#### MSHR — the key structure

See header comment in [VX_cache_mshr.sv:16-43](../hw/rtl/cache/VX_cache_mshr.sv#L16-L43) and integration in [VX_cache_bank.sv:464-544](../hw/rtl/cache/VX_cache_bank.sv#L464-L544).

The actual lifecycle of an MSHR entry:

1. **Allocate on entry (st0).** Every incoming core request grabs a free MSHR slot *before* hit/miss is known ([VX_cache_bank.sv:529-536](../hw/rtl/cache/VX_cache_bank.sv#L529-L536)). The bank refuses to admit a new request when MSHR is almost-full ([VX_cache_bank.sv:231-234](../hw/rtl/cache/VX_cache_bank.sv#L231-L234)) — this is the early-full deadlock prevention mentioned in the doc.
2. **Finalize in st1.** If it hit, *release* the slot. If it missed, *keep* the slot and link it (via `mshr_previd`) to any earlier pending entry for the same line — forming a per-line linked list. This is how miss coalescing works.
3. **Fill request emitted** only if there's no other in-flight miss for that line (`~mshr_pending_st1`) — no duplicate fills.
4. **Fill returns from DRAM** → MSHR uses `fill_id` to find the head entry and start dequeuing every linked entry for that line. These enter the arbiter as **replays** (priority #2) and walk the pipeline again, this time hitting.

So MSHR does two things at once:
- (a) reservation station for in-flight misses,
- (b) miss-coalescing structure so 100 threads missing the same line → 1 DRAM fill + 100 replays.

### 5. Core Response Merger (output crossbar)

[VX_cache.sv:448-465](../hw/rtl/cache/VX_cache.sv#L448-L465) — `VX_stream_xbar` of size `NUM_BANKS → NUM_REQS`.

- "Merge" here means *many-to-many routed gather*, not "fused with the DRAM-request stream."
- Each bank can produce a response in the same cycle; each response routes back to the thread-lane port that issued it. The `req_idx` carried through the pipeline tells the xbar where to send it.

### 6. DRAM Request Arbiter (output arbiter)

[VX_cache.sv:507-522](../hw/rtl/cache/VX_cache.sv#L507-L522) — `VX_stream_arb` of size `NUM_BANKS → MEM_PORTS`.

- Banks emit (a) fill requests on miss, (b) write-throughs / writebacks.
- Funnels them down to however many memory ports the cache exposes upstream (commonly `MEM_PORTS=1`).
- Independent of the response merger above.

## Other RTL-level facts worth knowing

- **Write-through is the default** (`WRITEBACK=0`). On a write hit, the data array is updated *and* a memory write is issued. Writeback configs only write to memory on dirty eviction.
- **Non-blocking** — up to `MSHR_SIZE` outstanding misses per bank.
- **Deadlock mitigation everywhere** — uses *almost-full* (not full) signals on queues so backpressure activates one cycle early, leaving room for the in-flight request. See `mreq_queue_alm_full`, `mshr_alm_full` usage.
- **Per-bank vs per-port memory tags**: requests carry a `BANK_MEM_TAG_WIDTH = UUID_WIDTH + MSHR_ADDR_WIDTH` tag internally. On the way out through the arbiter, additional bank-id bits get appended so the response demux can route the fill back to the right bank.
- **The diagram shows DRAM ports per-bank** for clarity, but with `MEM_PORTS < NUM_BANKS` (common), all banks share the final memory port via the arbiter shown above.
- **Pipeline stall plumbing**: `pipe_stall = crsp_queue_stall` — i.e. the response queue fills up → pipeline halts.

## Same module, four roles

The exact same `VX_cache` module is parameterized to serve as **Icache, Dcache, L2, and L3** — the difference is just `NUM_REQS`, `NUM_BANKS`, `LINE_SIZE`, `CACHE_SIZE`, `MSHR_SIZE`, `WRITEBACK`, etc. Read [VX_cache_wrap.sv](../hw/rtl/cache/VX_cache_wrap.sv) and [VX_cache_cluster.sv](../hw/rtl/cache/VX_cache_cluster.sv) for the wrappers that instantiate it at each level.

## Files in this subsystem

| File | Role |
|------|------|
| [VX_cache.sv](../hw/rtl/cache/VX_cache.sv) | Top: input xbar, output xbar/arb, response demux, bank instantiation |
| [VX_cache_bank.sv](../hw/rtl/cache/VX_cache_bank.sv) | One bank: 5-way arbiter, 2-stage pipeline, queues |
| [VX_cache_tags.sv](../hw/rtl/cache/VX_cache_tags.sv) | Tag SRAM + hit/miss compare + dirty bits |
| [VX_cache_data.sv](../hw/rtl/cache/VX_cache_data.sv) | Data SRAM + byte-enables + evict path |
| [VX_cache_mshr.sv](../hw/rtl/cache/VX_cache_mshr.sv) | MSHR with per-line linked-list coalescing |
| [VX_cache_repl.sv](../hw/rtl/cache/VX_cache_repl.sv) | Replacement policy (FIFO / PLRU) |
| [VX_cache_flush.sv](../hw/rtl/cache/VX_cache_flush.sv) | Init walk + explicit flush |
| [VX_cache_init.sv](../hw/rtl/cache/VX_cache_init.sv) | Pre-xbar input wiring + flush trigger |
| [VX_cache_bypass.sv](../hw/rtl/cache/VX_cache_bypass.sv) | Bypass-mode (no-cache) path |
| [VX_cache_wrap.sv](../hw/rtl/cache/VX_cache_wrap.sv) | Optional-bypass wrapper |
| [VX_cache_cluster.sv](../hw/rtl/cache/VX_cache_cluster.sv) | Multi-cache clustering (L2/L3) |
| [VX_cache_top.sv](../hw/rtl/cache/VX_cache_top.sv) | Top-level for synthesis/test |
| [VX_cache_define.vh](../hw/rtl/cache/VX_cache_define.vh) | Macros, derived params |
