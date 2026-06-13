# Sorcerer operator guide

Task-oriented: from "I have disks and an address" to mining. The plot
operations (verify / transform / info) have their own reference in
[plot_operations.md](plot_operations.md).

## TL;DR

```sh
# plot two disks, fsless, one process (GPU auto-selected, CPU fallback):
sudo pocx_sorcerer plot -i <your-address> --device /dev/sdX /dev/sdY -m 40GB

# mine them:
pocx_sorcerer mine -c miner_config.yaml

# what's on this box? (read-only, safe anytime)
pocx_sorcerer info

# spot-check / prove a plot:
pocx_sorcerer verify /dev/sdX            # 16 random warps
pocx_sorcerer verify --full /dev/sdX     # exhaustive

# rehouse a plot: convert + extend + upgrade in one pass
pocx_sorcerer transform old.pocx --to /dev/sdY -w <warps> -x 2 -f
```

Linux only, any architecture. No GPU? Same command — it falls back to
CPU and says so. Want CPU deliberately? Add `-c` (that *disables* the
GPU; `-g` and `-c` are mutually exclusive).

## Plotting

What happens when you run the plot command, per device:

1. **Calibration** — a full-disk sweep (64 zones, pooled access latency,
   scatter penalty; ~1.5 min on 500 GB, ~13 min on 10 TB; devices
   calibrate in parallel). Result is stamped into the on-device
   superblock; the miner can consume it later.
2. **Seek-aware ETA** is printed — an upper bound until auto-escalation
   resolves.
3. **Plot data** flows: hashing + folding on the GPU, write-ready data
   to the disks, until each device's plot is FINALIZED.

Knobs that matter (everything else is auto):

* `-i <address>` — the mining address. The only thing you must supply.
* `--device A B C…` — block devices, any count, one process.
* `-m <budget>` — host RAM you're willing to spend. Bigger budget →
  bigger write batches → fewer seeks → faster plotting. GPU mode costs
  `(disks+1)` GiB per escalation step.
* `-w N` — limit the plot to N warps (1 warp = 1 GiB). Default: fill
  the disk. Oversize requests are rejected before calibration.
* `--vmem <budget>` — cap GPU memory (ring + 1 GiB output). Default
  sizes from the device's compute units (~3 GiB minimum).

**Interrupted?** Re-run with `--plotonly`: seeds and progress live on
the devices, plotting resumes where it stopped. (Re-running *without*
`--plotonly` also resumes, but wastes a recalibration first.)

**`-f`** is needed only to overwrite a FINALIZED plot or replot to a
different address. Fresh disks, old filesystems, interrupted plots:
no `-f`.

**Completed device?** Re-running prints
`already complete per resume marker — finalized, skipping.`

## Mining

The config is one hierarchical YAML: what this machine **has** (`machine`,
`plots`) and what it **does with blocks** (`networks` = where mining-info
comes from, `targets` = named submission destinations, `accounts` = which
payee address routes to which target). Minimal config against a local node:

```yaml
machine:
  cpu_threads: 0            # 0 = auto-detect (all logical cores)
  cpu_thread_pinning: true

plots:                      # what this machine HAS
  dirs: []                  # filesystem-backed .pocx plot directories
  devices: [/dev/sdb, /dev/sdd]   # raw fsless plots (one per device)
  direct_io: true
  read_cache_in_warps: 16
  on_the_fly_compression: false   # set explicitly — absent defaults to TRUE!

networks:                   # where blocks come from (one entry per network)
  - id: 'Bitcoin-PoCX Mainnet'
    mining_info:
      url: 'http://127.0.0.1:8332'
      auth: { type: cookie, cookie_path: '/root/.bitcoin-pocx/.cookie' }
    block_time: 120

targets:                    # named, reusable submission destinations
  - name: solo
    network: 'Bitcoin-PoCX Mainnet'
    mode: solo              # 'solo' (submit to the network's own node) or 'pool'

accounts:                   # route each plot's payee address to a target
  - id: <your-pocx-address>
    targets: [ solo ]
```

Full, fully-commented examples ship next to these docs:
[`miner_config.example.yaml`](miner_config.example.yaml) and
[`miner_config.fsless.example.yaml`](miner_config.fsless.example.yaml).

Round behavior: one seek + one sequential zone read per disk per block
(zone position follows the scoop → round times vary with platter zone,
e.g. 9–19 s across a 10 TB disk), hashing overlaps the reads. The
hasher pool is demand-scheduled — "16 threads" in the banner is pool
size, not a utilization promise.

**Sizing a box:** `benchmark: 'CPU'` with no plots configured
fabricates an in-memory plot array and prints the box's hash ceiling:

```
CPU BENCHMARK: 14.50 Mnonce/s — serviceable capacity ≈ 35 TiB @ 10s ...
```

Treat it as conservative — real scans measure cheaper per nonce.

## Plot operations (verify / transform / info)

Full doc with format diagrams and costs: [plot_operations.md](plot_operations.md).
All of these auto-select the GPU like the plotter does (`-c` forces CPU).

* **Prove a plot is good** — `verify --full /dev/sdX` regenerates every byte
  from (address, seed) and compares; `verify /dev/sdX` spot-checks 16 random
  warps. Read-only, safe beside a running miner.
* **Move / upgrade plots without replotting** —
  `transform <src> --to /dev/sdY [-w N] [-x X] -f` builds a fresh fsless plot
  from any source in one pass: fs→fsless conversion is a copy, **X-upgrades
  are pure XOR** (no hashing — only an extend tail is regenerated, on the
  GPU). The target comes out FINALIZED and immediately mineable. Field
  number: 20 GiB X1 file → 30 GiB X2 disk in ~5.7 min on a 110 MB/s
  test HDD — disk-write-bound, i.e. as fast as that disk could be plotted
  at all.
* **What's on this box?** — `info` shows CPU/RAM/GPUs and every POCX
  superblock found on the block devices, including resume progress of
  interrupted plots.

## Numbers from the field

| Box | Role | Measured |
|---|---|---|
| Xeon E-2276M (6P+6HT, AVX2) + RTX 5000 | plotter | 10 TB pair in ~19 h, ~150 MB/s/disk, ~1 host thread busy |
| Atom C3958 (16P, SSE2, fixed clock) | miner, 4×10 TB | rounds 16.5–17.5 s, ~5.7 cores during scans, idle between |
| WD5002ABYS 500 GB (2008) | test ground | 85 MiB/s plot, zones 110→66 MB/s |

## FAQ

* **Device letters moved after reboot/reattach.** They do. Identify by
  serial (`lsblk -dno SERIAL /dev/sdX`) before anything destructive.
* **Multiple GPUs?** Not yet — one `-g` per process. Run one process per
  GPU with disjoint `--device` sets.
* **`-g` together with `-c`?** Refused. The GPU ring and the CPU
  pipeline don't mix; pick one engine per process.
* **Does mining need the GPU?** No. The unified binary runs fine on
  GPU-less hosts — OpenCL is loaded at runtime only when wanted, and a
  host with no GPU falls back to CPU automatically.
* **Turn the GPU off entirely (`--nogpu`).** A global flag on every
  command — `pocx_sorcerer --nogpu <cmd> …` or `pocx_sorcerer <cmd>
  --nogpu …` — makes the binary do **no OpenCL probing at all**:
  `libOpenCL` is never loaded, and plot/mine/info stay on the CPU path.
  The env var `SORCERER_NO_GPU=1` does the same (handy in scripts/units).
  Use it if a host's OpenCL stack misbehaves, or on a fully-static
  (musl) build where loading the GPU driver at runtime isn't available.
  (For plotting you can also just pass `-c` to force CPU; `--nogpu` is
  the binary-wide hard switch that also covers `info` and `mine`.)
* **glibc errors on old distros? / a portable binary.** Use the static
  **musl** flavor (`…-x86_64-musl_demo` / `…-aarch64-musl_demo`): a
  self-contained binary that runs on any Linux regardless of its glibc.
  Caveat: it's **CPU-only** — a static binary can't load the GPU's OpenCL
  driver at runtime (use a **glibc** flavor for GPU plotting; `--nogpu`
  forces CPU explicitly).
* **Raspberry Pi 5 / ARM64.** Use the `…-aarch64-…` flavors. The
  **glibc** one is dynamically linked and can use the Pi's OpenCL (Mesa
  rusticl / v3d); the **musl** one is static and CPU-only but needs no
  glibc.
