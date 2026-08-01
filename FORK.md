# About this fork

Fork of [SysSec-KAIST/LTESniffer](https://github.com/SysSec-KAIST/LTESniffer) carrying a
small number of local changes. Upstream is AGPLv3; so is this.

`main` is kept **byte-identical to upstream** (currently `a694803`) so every branch below is
an exact, reviewable diff against pristine upstream. Rebases onto a future upstream release
stay clean. Do not commit to `main`.

## Branches

| Branch | Base | Contents |
|---|---|---|
| `main` | — | pristine upstream `a694803` |
| `fix/aarch64-simd-detection` | `main` | build fix: LTESniffer cannot configure on 64-bit ARM |
| `bts/phy-telemetry` | `main` | two additive PHY/DCI stderr traces (23 insertions, 0 deletions) |
| `bts/pi-build` | merge of the two above | what we actually build |

## `fix/aarch64-simd-detection` — build fix

LTESniffer currently **cannot be configured on any 64-bit ARM host** (Raspberry Pi 4/5 on a
64-bit OS, Jetson, Ampere, arm64 Docker).

`CMakeLists.txt` gates NEON on `if(${CMAKE_SYSTEM_PROCESSOR} MATCHES "arm")`. CMake's
`MATCHES` is a regex search, and this fails two different ways:

- **`aarch64`** (Linux) does *not* contain the substring `arm`, so it never matches.
  `HAVE_NEON` stays `False`, `HAVE_SSE` is also false, and configuration aborts at
  `message(FATAL_ERROR "no SIMD instructions found")`.
- **`arm64`** (macOS/Apple silicon, some BSDs) *does* contain `arm`, so it matches — and is
  then handed `-mfpu=neon`, which is armv7-only syntax that 64-bit compilers reject.

The fix tests `aarch64|arm64` **first** (ordering matters — a naive
`MATCHES "arm" OR MATCHES "aarch64"` still mis-routes `arm64`), defines `HAVE_NEONv8`, and
keeps `-mfpu=neon` for 32-bit ARM only. No behaviour change on x86.

The NEON code paths themselves are already comprehensive — the bundled srsRAN's `simd.h`
carries ~180 `HAVE_NEON` paths against ~206 `HAVE_SSE` — so only this build-system gate was
blocking ARM.

> **Status: branch logic verified across `aarch64`, `arm64`, `armv7l`, `armv6l`, `x86_64` and
> `AMD64`; not yet confirmed by a completed aarch64 compile+link.** Treat as unproven until
> that is done.

## `bts/phy-telemetry` — two stderr traces

Local instrumentation for offline analysis. Both are purely additive and emit to **stderr**,
deliberately: stdout is block-buffered, so trailing lines are lost when a time-boxed capture
is stopped with `SIGINT`.

**1. `DCIMCS,tti,rnti,mcs,mod,prb,layers,fmt,tbs`** — `DL_Sniffer_PDSCH.cc`, at the three
CRC-OK `write_pcap` sites in `decode_dl_mode()`.

Stock LTESniffer exposes no per-UE MCS or PRB in downlink-only mode (`-m 0`): the MAC pcap's
DL-PHY context fields (`mac-lte.dl-phy.mcs-index`, `rb-length`) are written to zero frames,
and the stdout activity table reports modulation as 100 % `Unknown`. This looks like the same
symptom as upstream issue #101. The values are present in srsRAN's grant struct at decode
time — they are simply never emitted.

**2. `DCIDL,tti,rnti` and `DCIUL,tti,rnti,is_rar`** — `SubframeWorker.cc`, in `run_dl_mode()`.

LTESniffer's DCI blind search already decodes **format-0 uplink grants** from the downlink
PDCCH even in downlink-only mode (`getULSnifferDCI_UL()` holds them), but discards them,
because format-0 carries no PDSCH and so never reaches the decode path. That is a latent
capability: uplink *scheduling* activity is observable from a downlink-only capture, with no
second radio.

DL is counted here at the raw blind-search stage rather than reusing the `DCIMCS` trace, and
that is load-bearing: `DCIMCS` fires only after a successful PDSCH decode, which under-counts
DL by roughly an order of magnitude (measured 4787 raw vs 269 decoded on one live capture).
Comparing raw UL against decoded DL would label nearly every device as "upload".

> **Honest limit:** this counts *grants*, not bytes. A downloading device still receives many
> small uplink grants (TCP ACKs), so any per-device DL/UL lean derived from it is coarse; the
> cell-level DL:UL grant ratio is the cleaner signal.

> **Status: not upstream-ready as-is.** Before proposing these upstream they need a CLI flag
> instead of unconditional output, an integer RNTI-type comparison instead of per-DCI string
> compares in the decode hot path, named constants for the C-RNTI bounds, and routing through
> the project logger rather than raw `fprintf(stderr, …)`.

## Verifying a working tree

```bash
grep -c MCS-PATCH  src/src/DL_Sniffer_PDSCH.cc   # expect 3
grep -c ULDL-PATCH src/src/SubframeWorker.cc     # expect 2
```

After a build, all three traces must appear on a cell with active phones:

```bash
./src/LTESniffer -A 1 -W 4 -f <DL_Hz> -C -m 0 -g 40 2>&1 | grep -cE '^(DCIMCS|DCIDL|DCIUL),'
```

## Note on the vendored srsRAN

`external/cmake/srsRAN.CMakeLists.txt.in` fetches srsRAN from a third-party personal fork at
a **floating tag**:

```cmake
GIT_REPOSITORY https://github.com/ShaoPaoLao/srsRAN2.git
GIT_TAG        master
```

Builds are therefore not reproducible, and depend on a repository outside the project's
control. The commit that produces our known-good binary is
**`0acc79d3fe5b153a18b62e8ef5af1a0fb2327a18`** (2024-09-29). Pin it if you need a
reproducible build.
