# pocx_sorcerer 0.1.2 — demo binaries

**Sorcerer** is a single, unified tool for Bitcoin-PoCX (Proof-of-Capacity)
storage mining. One binary — `pocx_sorcerer` — covers the whole lifecycle:

* **plot** — write plots to disks (GPU-accelerated hashing + on-GPU XOR
  compression, automatic CPU fallback), filesystem-backed *or* raw
  "filesystem-less" (fsless) directly on a block device;
* **mine** — a long-lived, multi-disk miner against a Bitcoin-PoCX node,
  with multi-network / multi-destination routing;
* **mine --pow** *(new in 0.1.2)* — **GPU mining**: regenerate the plot on
  the GPU on demand and scan it in VRAM, with no disk at all. Validated
  end-to-end against a live node; it runs at the GPU's plotting speed, so
  it is a capability/proof rather than a substitute for disk-backed mining;
* **plot toolbox** — `verify` a plot byte-for-byte against regeneration,
  `transform` it (convert ∙ extend ∙ reduce ∙ upgrade in one pass, with
  pure-XOR compression upgrades that need no rehashing), and `info` to
  inventory every plot on a box.

Linux only, any architecture. No GPU needed for mining; OpenCL is used only
when present and wanted.

## New in 0.1.2

* **GPU mining (`mine --pow`)** — regenerate-and-scan mining entirely in GPU
  VRAM, no disk. Add a `pow:` block (address + warps) to the miner config and
  run `mine --pow`.

## Builds in this drop

```
┌────────────────────────────────────────┬─────────┬────────┬────────────────────────────┐
│                  File                  │  Arch   │ Flavor │           Notes            │
├────────────────────────────────────────┼─────────┼────────┼────────────────────────────┤
│ pocx_sorcerer-0.1.2-x86_64-glibc_demo  │ x86_64  │ glibc  │ dynamic, OpenCL/GPU        │
├────────────────────────────────────────┼─────────┼────────┼────────────────────────────┤
│ pocx_sorcerer-0.1.2-x86_64-musl_demo   │ x86_64  │ musl   │ static, CPU-only, portable │
├────────────────────────────────────────┼─────────┼────────┼────────────────────────────┤
│ pocx_sorcerer-0.1.2-aarch64-glibc_demo │ aarch64 │ glibc  │ Pi 5, dynamic, Pi OpenCL   │
├────────────────────────────────────────┼─────────┼────────┼────────────────────────────┤
│ pocx_sorcerer-0.1.2-aarch64-musl_demo  │ aarch64 │ musl   │ Pi 5, static, CPU-only     │
└────────────────────────────────────────┴─────────┴────────┴────────────────────────────┘
```

* **glibc** builds are dynamically linked and can load OpenCL at runtime for
  GPU plotting/mining (and Pi OpenCL on the aarch64 build). Use these on a
  normal Linux install when you want the GPU.
* **musl** builds are fully static and self-contained — they run on any Linux
  regardless of its glibc version, but are **CPU-only** (a static binary can't
  load the GPU's OpenCL driver). Ideal for portability and locked-down hosts.

Pick by your CPU architecture (`uname -m` → `x86_64` or `aarch64`) and whether
you want GPU acceleration (glibc) or maximum portability (musl).

## Demo limitations

These are **demo** binaries. The following limits are baked in:

* **Plotting and plot transforms are capped at 4 TiB** per operation.
* **Mining is limited to 4 distinct payee addresses.**
* The build **ceases to operate after block height 50000** (help still works).

Operating within these limits behaves identically to the full product. Exceed
them and the affected operation produces unusable output rather than extra
capacity.

## Getting started

```sh
chmod +x pocx_sorcerer-0.1.2-<arch>-<flavor>_demo
./pocx_sorcerer-0.1.2-<arch>-<flavor>_demo info        # inventory this box (read-only)
./pocx_sorcerer-0.1.2-<arch>-<flavor>_demo --help
```

Full documentation is in [`../docs/`](../docs/):

* [Operator guide](../docs/guide.md) — disks + an address → mining.
* [Plot operations](../docs/plot_operations.md) — verify, transform, info, the
  on-disk format, and the algorithms.

## Integrity (SHA-256)

```
9f7246f1a47167376f4bebcab6d5f1004f75a1037e75cd62206dcc5e1e7f4efb  pocx_sorcerer-0.1.2-aarch64-glibc_demo
54e48d7cd411f5ff0e2c7e4ea99cb759e77ed4b8b3498dc28d3d5831978d11c1  pocx_sorcerer-0.1.2-aarch64-musl_demo
7b96b2dadc795fefeb333d91b6828d9f8b568ff8f5a4e72cc5d3e90207a8cc10  pocx_sorcerer-0.1.2-x86_64-glibc_demo
3fba3c1690e64ec786e5dfc5599dfffd657c16f781f8412064e62567533c169e  pocx_sorcerer-0.1.2-x86_64-musl_demo
```
