# pocx_sorcerer 0.1.0 — demo binaries

**Sorcerer** is a single, unified tool for Bitcoin-PoCX (Proof-of-Capacity)
storage mining. One binary — `pocx_sorcerer` — covers the whole lifecycle:

* **plot** — write plots to disks (GPU-accelerated hashing + on-GPU XOR
  compression, automatic CPU fallback), filesystem-backed *or* raw
  "filesystem-less" (fsless) directly on a block device;
* **mine** — a long-lived, multi-disk miner against a Bitcoin-PoCX node,
  with multi-network / multi-destination routing;
* **plot toolbox** — `verify` a plot byte-for-byte against regeneration,
  `transform` it (convert ∙ extend ∙ reduce ∙ upgrade in one pass, with
  pure-XOR compression upgrades that need no rehashing), and `info` to
  inventory every plot on a box.

Linux only, any architecture. No GPU needed for mining; OpenCL is used only
when present and wanted.

## Builds in this drop

```
┌────────────────────────────────────────┬─────────┬────────┬────────────────────────────┐
│                  File                  │  Arch   │ Flavor │           Notes            │
├────────────────────────────────────────┼─────────┼────────┼────────────────────────────┤
│ pocx_sorcerer-0.1.0-x86_64-glibc_demo  │ x86_64  │ glibc  │ dynamic, OpenCL/GPU        │
├────────────────────────────────────────┼─────────┼────────┼────────────────────────────┤
│ pocx_sorcerer-0.1.0-x86_64-musl_demo   │ x86_64  │ musl   │ static, CPU-only, portable │
├────────────────────────────────────────┼─────────┼────────┼────────────────────────────┤
│ pocx_sorcerer-0.1.0-aarch64-glibc_demo │ aarch64 │ glibc  │ Pi 5, dynamic, Pi OpenCL   │
├────────────────────────────────────────┼─────────┼────────┼────────────────────────────┤
│ pocx_sorcerer-0.1.0-aarch64-musl_demo  │ aarch64 │ musl   │ Pi 5, static, CPU-only     │
└────────────────────────────────────────┴─────────┴────────┴────────────────────────────┘
```

* **glibc** builds are dynamically linked and can load OpenCL at runtime for
  GPU plotting (and Pi OpenCL on the aarch64 build). Use these on a normal
  Linux install when you want the GPU.
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
chmod +x pocx_sorcerer-0.1.0-<arch>-<flavor>_demo
./pocx_sorcerer-0.1.0-<arch>-<flavor>_demo info        # inventory this box (read-only)
./pocx_sorcerer-0.1.0-<arch>-<flavor>_demo --help
```

Full documentation is in [`../docs/`](../docs/):

* [Operator guide](../docs/guide.md) — disks + an address → mining.
* [Plot operations](../docs/plot_operations.md) — verify, transform, info, the
  on-disk format, and the algorithms.

## Integrity (SHA-256)

```
264a0bdad33e3d6af5d531d28b8399b22fbe140ae4293b6d05319523be3d00be  pocx_sorcerer-0.1.0-aarch64-glibc_demo
0081a21a7be2746838868d91e5b690f323f5921f5399c2e9b453b9833f0abf2f  pocx_sorcerer-0.1.0-aarch64-musl_demo
f941e7ea2f9215ac4f666b18f834712c65c611fcbdaedff8479155819bbe0d1a  pocx_sorcerer-0.1.0-x86_64-glibc_demo
9bcdf1fbf790e1e813f09d6acffdeb46f9163e68ad0c3c8f5ad99caff381386d  pocx_sorcerer-0.1.0-x86_64-musl_demo
```
