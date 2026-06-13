# Plot operations — verify, transform, info

Operator documentation for sorcerer's plot toolbox: what the verbs do, the
on-disk format they operate on, and the algorithms behind them. Task recipes
live in the [operator guide](guide.md).

**One engine everywhere.** Every operation that produces plot bytes — plotting
itself, verification, transformation — runs the same generation machinery: the
GPU ring pipeline (hash + fused scatter/compress kernels) when a GPU is
present, the SIMD CPU path otherwise. The contract is the plotter's layer-0
default in every verb:

| You pass | Engine |
|----------|--------|
| *(nothing)* | best available GPU (discrete > VRAM > CUs), CPU fallback |
| `-c` | CPU, deliberately |
| `-g platform:device[:cores]` | that GPU, or an error |

## The format in one picture

A plot is identified by `(address, seed, warps, X)`. The same plot data can
live in a `.pocx` file on a filesystem or raw on a block device ("fsless"):

```text
byte 0          ┌─────────────────────────────────┐
                │ superblock (4 KiB)              │  magic POCX-FSLESS, status,
                │                                 │  address, seed, warps, X,
byte 4096       ├─────────────────────────────────┤  calibration buckets, CRC
                │                                 │
                │ plot data — warps × 1 GiB       │  scoop-major, byte-identical
                │                                 │  to the fs-backed .pocx
                │                  ┌──────────────┤
                │                  │ marker (8 B) │  resume progress + magic
4096 + W·1 GiB  ├──────────────────┴──────────────┤
                │ slack (unused)                  │
end of device   └─────────────────────────────────┘
```

(FS plots are the same minus the superblock — identity lives in the filename,
the marker at the same relative position.)

The **warp** is the indivisible unit: a 4096 × 4096 matrix of 64-byte
double-hashes = 1 GiB. Storage is **scoop-major** — scoop `s` of every nonce
is contiguous, so one mining round is one seek plus one sequential read:

```text
              ◀────────── one scoop band: W · 256 KiB, contiguous ──────────▶
             ┌────────────┬────────────┬────────────┬───────┬────────────┐
  scoop    0 │  warp 0    │  warp 1    │  warp 2    │   …   │  warp W−1  │
             ├────────────┼────────────┼────────────┼───────┼────────────┤
  scoop    1 │            │            │            │       │            │
             ├────────────┼────────────┼────────────┼───────┼────────────┤
      ⋮      │                              ⋮                            │
             ├────────────┼────────────┼────────────┼───────┼────────────┤
  scoop 4095 │            │            │            │       │         ▣  │
             └────────────┴────────────┴────────────┴───────┴────────────┘
   one cell = 256 KiB: warp w's 4096 nonces at scoop s (64 B each)   ▣ = marker
```

This drives every toolbox I/O shape: a single warp's data is *scattered* (one
256 KiB cell per band — expensive to sample on HDD), while a *range* of warps
read band-by-band is sequential (cheap to stream).

## The fold algebra — why upgrades are free

A warp at compression X is built from `2^X · 4096` generated nonces in two
steps. First the **helix fold** pairs generated warps with a 90°-transposed
XOR (`h[y][x] = gᵢ[y][x] ⊕ gᵢ₊₁[x][y]`) — this is the anti-Helix measure that
makes on-the-fly *de*compression impossible. Then, for X > 1, helix warps are
folded together by **plain cell-wise XOR** at constant (scoop, nonce)
position:

```text
 generated:    g₀  g₁     g₂  g₃     g₄  g₅     g₆  g₇
                ╲ ╱        ╲ ╱        ╲ ╱        ╲ ╱
 helix fold      ╳ᵀ⊕        ╳ᵀ⊕        ╳ᵀ⊕        ╳ᵀ⊕      (transpose ⊕)
 helix warps:    h₀         h₁         h₂         h₃
                 │          │          │          │
 X1 stores:    [ h₀ ]     [ h₁ ]     [ h₂ ]     [ h₃ ]
                  ╲        ╱            ╲         ╱
 X2 stores:    [ h₀ ⊕ h₁ ]              [ h₂ ⊕ h₃ ]        (cell-wise ⊕)
                    ╲                       ╱
 X3 stores:           [ h₀ ⊕ h₁ ⊕ h₂ ⊕ h₃ ]
```

X1 *stores the helix warps directly*, so the **upgrade theorem** follows:

```text
   Xb[w]  =  Xa[2ᵇ⁻ᵃ·w]  ⊕  Xa[2ᵇ⁻ᵃ·w + 1]  ⊕ … ⊕  Xa[2ᵇ⁻ᵃ·w + 2ᵇ⁻ᵃ − 1]
```

* **Upgrading a plot needs no hashing** — read `2^ΔX` stored warps, XOR them
  cell-wise, write one warp. The transpose cancels out of the merge entirely.
  (Proven byte-exact and validated on device end-to-end.)
* The miner uses the same identity to **mine an Xa plot at Xb on the fly**
  (XOR paired band reads, at proportionally reduced capacity).
* **Downgrading is impossible** — XOR drops its summands. Not a use case.
* Upgrading `2^ΔX · W` warps yields `W` warps covering the same nonce range in
  `1/2^ΔX` of the bytes — pair the upgrade with an **extend** to refill the
  freed space at the higher density.

## verify — integrity check

```sh
pocx_sorcerer verify /dev/sdX                      # sample 16 random warps
pocx_sorcerer verify --full /dev/sdX               # every warp, exhaustive
pocx_sorcerer verify --scoop 1234 --nonce 567 p.pocx   # one cell, milliseconds
```

Read-only (no locks, plain reads) — safe beside a live miner. The plot is
ground truth against *regeneration*: the engine replays the plotter's exact
pipeline from (address, seed) and the result is compared byte-for-byte. The
marker cell's 8 metadata bytes are excluded.

| Mode                     | Reads                                  | Hashes              | Use it for                              |
|--------------------------|----------------------------------------|---------------------|-----------------------------------------|
| `--scoop --nonce`        | one 64 B cell                          | 1 scoop             | a specific reported-bad nonce           |
| *(default)* `--sample N` | N scattered warps (seek-bound on HDD)  | `2^X·4096·N` nonces | spot check; reproducible (seeded)       |
| `--full`                 | whole plot, sequential                 | `2^X·4096·W` nonces | proof of integrity (regeneration-bound) |

`-m` sizes the warp-block (bigger = fewer seeks; default: auto — 85% of
available RAM with a ≥2 GiB headroom floor).

## transform — convert ∙ extend ∙ reduce ∙ upgrade, one pass

```sh
pocx_sorcerer transform <SOURCE> --to <DEVICE> [-w N] [-x X] [-m SIZE] -f
```

Builds a **fresh fsless plot** on `--to` (device contents overwritten — `-f`
required; mounted devices refused) from any source plot (`.pocx` file or
fsless device, read-only, never touched). Target warp count and X default to
the source's; what you change selects the operation, and all combinations are
one single pass:

| Change         | Operation                 | Data source                                |
|----------------|---------------------------|--------------------------------------------|
| container only | **convert** (fs ↔ fsless) | copy                                       |
| `-w` larger    | **extend**                | copy/derive overlap + regenerate tail      |
| `-w` smaller   | **reduce**                | copy/derive prefix, drop the surplus       |
| `-x` larger    | **upgrade**               | XOR-merge `2^ΔX` stored warps (no hashing) |

The derivation window: a source of `S` warps at Xa covers the first
`⌊S / 2^ΔX⌋` target warps at Xb; everything beyond is regenerated from
(address, seed) — identical bytes either way:

```text
 source  20 × X1   [s₀ s₁] [s₂ s₃] [s₄ s₅]  …  [s₁₈ s₁₉]
                      ⊕       ⊕       ⊕            ⊕        derive: read + ⊕
 target  30 × X2   [ t₀ ]  [ t₁ ]  [ t₂ ]  …  [  t₉  ]  [ t₁₀ ……………… t₂₉ ]
                                                          regenerate: GPU ring
```

Two details handled for you: the cell where the *source's* resume marker sat
is re-derived (the target carries true plot data where the source carried
metadata — a transformed plot is in this sense *cleaner* than its source), and
the target gets a complete marker plus FINALIZED superblock, so it is
immediately mineable. (The target carries no calibration data yet —
calibration on transform is a future option.)

### The pipeline

Derivation and target I/O overlap, the plotter's producer/writer shape:

```text
 ┌───────────────────────── producer (calling thread) ────────────────────────┐
 │                                                                            │
 │   source bands ──read──▶ ⊕-merge ──────────┐                               │
 │                                            ├──▶  block of K warps          │
 │   GPU ring ──regenerate the tail───────────┘                               │
 │                                                                            │
 └──────────────────────────────────────┬─────────────────────────────────────┘
                                        │  rendezvous handoff, depth 1
                                        │  (at most one block assembling
                                        ▼   while one is being written)
 ┌──────────────────────────────────────┴─────────────────────────────────────┐
 │                               writer (thread)                              │
 │                                                                            │
 │   for each scoop band: write K · 256 KiB sequentially ──▶  target device   │
 └────────────────────────────────────────────────────────────────────────────┘
```

Wall time tends to `max(derivation, target write)` — on rotational targets
that is the disk's sequential write rate, i.e. the same floor as plotting the
device fresh, *minus* everything the source already paid for.

`-m` sizes K (auto default as in verify); the budget prices one block in
assembly + one in flight + the engine's scratch, so it cannot OOM the box.

### The regeneration engine

```text
            ┌───────────────────────── GPU ─────────────────────────┐
            │   ring buffer (R = W + C − gcd(W,C) nonces)           │
 (address,  │   ┌───────────────────┐     ┌──────────────────────┐  │
  seed,     ──▶ │ calculate_nonces  │ ──▶ │ fused scatter +      │  │
  start     │   │ (Shabal, W/batch) │     │ compress (helix ⊕,   │  │
  nonce)    │   └───────────────────┘     │ ⊕-accumulate         │  │
            │            ▲                │ 2^(X−1) passes/warp) │  │
            │            └── refill ──────┴──────────┬───────────┘  │
            └────────────────────────────────────────┼──────────────┘
                                                     ▼ DMA, scoop-major
                                          finished warp → host block
```

The CPU fallback (`-c`) runs the same algorithm on SIMD cores; its
parallelism spans the warps of one block, so it scales with `-m`
(`2^X + 3` GiB of budget per concurrently regenerated warp) — usable for
spot checks, slow for bulk work. That asymmetry is why the GPU is the
default.

## info — machine and plot summary

```sh
pocx_sorcerer info
```

Read-only, lock-free, safe beside running plotters and miners: CPU topology
and SIMD level, RAM, GPUs, and a scan of all block devices for POCX-FSLESS
superblocks — per device: status, address, seed, warps, X, calibration state,
media type, and (for in-progress plots) resume-marker progress.

## Field-measured costs

One reference job — 20-warp X1 FS plot → 30-warp X2 fsless — on a Quadro
RTX 5000 with a 2008-era 500 GB WD (outer zone ≈ 110 MB/s) as target:

| Operation                                | Wall time               | Bound by                               |
|------------------------------------------|-------------------------|----------------------------------------|
| transform (upgrade + extend + convert)   | **341 s**               | target disk write                      |
| — same, CPU engine, 8 GiB budget         | ≈ 70 min (extrapolated) | single-threaded hashing                |
| verify `--full` of the 30-warp X2 result | 612 s                   | regen + sequential read (serial today) |
| verify (sample 16 warps) of the same     | 392 s                   | HDD seeks (scattered cells)            |
| verify `--scoop --nonce`                 | < 1 s                   | —                                      |

Scaling intuition: the upgrade/convert/reduce parts are pure I/O; the extend
tail and any verify are generation work at the plotter's rate (this GPU:
~2400 nonces/s ≈ 18 stored X1-warps/min ≈ 1071 warps/h). On a 240 MB/s
IronWolf the same transform would be ~2.5 min.
