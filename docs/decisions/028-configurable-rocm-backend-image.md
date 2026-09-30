# ADR 028: Configurable ROCm backend image

**Status:** Proposed
**Date:** 2026-09-30
**Related:** [ADR 001 — llama.cpp over vLLM](001-llamacpp-over-vllm.md), [ADR 003 — Model host path](003-model-host-path.md), [ADR 006 — Strix Halo GPU tuning](006-strix-halo-gpu-tuning.md), [ADR 008 — Container UID allocation](008-container-uid-allocation.md), [ADR 029 — Configurable backend image references and refresh](029-configurable-backend-image-refresh.md)

## WH(Y) Decision Statement

**In the context of** the proxy selecting a llama.cpp backend image from two hard-coded constants, with a single AMD Ryzen AI Max (gfx1151) target and Vulkan as the only GPU backend today,

**facing** an untested question of whether ROCm would serve that target faster than Vulkan, and a set of candidate images that differ so widely in cost that the choice materially changes the deployment — 0.31 GB for Vulkan against 7.71 GB for the official ROCm image,

**we decided for** an environment-backed image reference per device type, defaulting to the current constants, so that candidate images can be compared by configuration rather than by code change, and deferred the image choice itself to the measurements below,

**and neglected** pre-selecting a ROCm image now, on the strength of a size comparison and a performance figure that a source comment misattributes to images this project never tested,

**to achieve** a deployment where the GPU backend is chosen on evidence gathered on the target hardware, and where comparing candidates costs a configuration change rather than a rebuild,

**accepting that** this ADR selects no image, that ROCm may measure slower than Vulkan and leave Vulkan in place, that the candidates remain unevaluated at the time of writing, and that a configurable reference is a slightly larger change than a third constant.

## Context

`proxy/src/docker/llamacpp.rs:15-16` hard-codes the two images the proxy can launch, and `llamacpp.rs:176-180` selects between them from a single field:

```rust
// Select image based on GPU type
let image = match config.gpu_type {
    GpuType::Vulkan => LLAMACPP_IMAGE_VULKAN,
    GpuType::None => LLAMACPP_IMAGE_CPU,
};
```

`GpuType` has only `None` and `Vulkan` variants (`llamacpp.rs:27-41`). Every candidate in this ADR is gated behind adding a third variant, and each candidate needs a different image, so the constants cannot stay hard-coded if more than one is to be tried.

Three claims in the current source bear on that choice, and all three are unreliable.

**The performance objection is misattributed.** `llamacpp.rs:19-20` cites "0.6–2.4 t/s vs 4.5 t/s from Vulkan" as a reason ROCm was removed. That figure originates at `docs/decisions/001-llamacpp-over-vllm.md:16`, where it describes custom gfx1151 **vLLM** ROCm images. No measurement of any `llama.cpp` ROCm image on gfx1151 exists in this repository. The comparison the comment appears to make has not been made.

**The seccomp objection is ambiguous rather than wrong.** `llamacpp.rs:19` states "ROCm requires seccomp=unconfined". AMD's [install-on-Linux guide](https://rocm.docs.amd.com/projects/install-on-linux/en/develop/how-to/docker.html) annotates the flag **"(optional)"**, describing it as "Recommended for HPC workloads that use `numactl`"; `llama-server` does not use `numactl`. AMD's [llama.cpp-specific guide](https://rocm.docs.amd.com/projects/llama-cpp/en/latest/install/llama-cpp-install.html) does include it in the standard run command, bundled with `--privileged` and `--cap-add=SYS_PTRACE`. Whether `llama-server` alone needs a relaxed profile is **unverified**. No `SecurityOpt` field is set anywhere in `proxy/src/`.

**The "gfx1151 is unsupported" position is outdated.** ADR 001 lists as a rejection reason that "The gfx1151 architecture (Strix Halo) is not in the ROCm supported GPU list". AMD's [ROCm 10.0.0 compatibility matrix](https://rocm.docs.amd.com/en/latest/compatibility/compatibility-matrix.html) now lists gfx1151 explicitly, under AMD Ryzen AI Max PRO 300 Series, RDNA 3.5, codename Gorgon Halo. The same matrix lists supported operating systems for that APU series as **Ubuntu 26.04 and 24.04.4 only — Debian is not listed**, whereas Instinct MI rows do list Debian 12 and 13. This project's proxy image is `debian:bookworm-slim` (`Dockerfile:18`). That gap is recorded as a known risk. **This ADR proposes no change to the host.**

**The candidates differ enough to be worth measuring.** Compressed layer totals from the GHCR registry API: `server` 0.31 GB, `server-vulkan` 0.31 GB, `server-rocm` **7.71 GB**. The cause is in upstream's [`.devops/rocm.Dockerfile`](https://raw.githubusercontent.com/ggml-org/llama.cpp/master/.devops/rocm.Dockerfile), where the final `server` stage inherits a `base` stage that inherits the ROCm **development** base:

```
ARG BASE_ROCM_DEV_CONTAINER=docker.io/rocm/dev-ubuntu-24.04:7.2.1-complete
ARG ROCM_DOCKER_ARCH='gfx908;gfx90a;gfx942;gfx1030;gfx1100;gfx1101;gfx1102;gfx1151;gfx1150;gfx1200;gfx1201'
```

The corresponding runtime base without `-complete` is 1.22 GB, so most of the excess is attributable to the development base and to eleven compiled architectures where one is needed.

**Vendor guidance conflicts with the project's isolation model, for every ROCm candidate.** The project attaches each backend to `sovereign-internal` (`internal: true`, `docs/DEPLOYMENT.md:196-206`), binds no host ports (`llamacpp.rs:198-208`), grants no capabilities, and runs each backend as a non-root UID in 10000–65000. AMD's llama.cpp run command prescribes `--network=host`, `--ipc=host`, `--privileged` and `--cap-add=SYS_PTRACE`, plus `--shm-size 16G`. Docker's `run` reference states that omitting `--shm-size` yields `64m`, and no `shm_size` is set anywhere in `proxy/src/`. Upstream separately states the GPU images "are not currently tested by CI beyond being built."

## Options Considered

No option is selected. Each is a candidate for the measurements below, and the criteria for choosing between them are stated in Decision.

| Candidate | Compressed | gfx1151 | Isolation | Notes |
|---|---|---|---|---|
| **Vulkan, status quo** | 0.31 GB | Yes | Already compliant | The incumbent. Wins if ROCm is not faster. |
| **Official `server-rocm`, unmodified** | 7.71 GB | Yes | Unproven | Maintained upstream, pinnable per build, not CI-tested. Carries the development base. |
| **Slim ROCm, `gfx1151`-only, runtime base** | ~1.5–2 GB (projection) | Yes | Unproven | Build and publish in a separate project we oversee. Size is a projection, not a measurement. |
| **Repackage official image, no recompile** | Unmeasured | Yes | Unproven | Extracts `/app` onto the runtime base. Most of the bulk is the fatbin, so it captures little benefit for real cost. |

Two observations constrain the set. The official image already compiles `gfx1151`, so gfx1151 support is not by itself an argument for a custom build — the argument is size. And the isolation question is **common to every ROCm candidate**, so it is answered once for all of them rather than per option.

## Decision

Two things are decided now. They are separable, and the first is a prerequisite for the second.

**One: image references become configuration.** Each device type gets an environment-backed image reference, defaulting to the value currently hard-coded. This is a superset of today's behaviour: with nothing configured, the proxy pulls and launches exactly what it does now, so the change is behaviour-preserving and can land independently of any image decision. `config.rs` has no image setting today, so this introduces the first. `pull_backend_images()` (`docker/mod.rs:196-214`) hard-codes the same two constants and must read the configured values in the same change, or the proxy will prefetch one image and launch another.

The supervisor needs no modification. `rebuild_llamacpp_config` (`supervisor/restart.rs:76-130`) reconstructs the same `LlamacppConfig` and reaches the same match, so a restarted backend resolves whatever the configuration says at that moment.

Three separate variables, one per device type, is the shape consistent with the existing `AppConfig` style of flat fields with environment defaults. A single name-to-reference mapping would generalise further, at the cost of a parsing step and weaker static typing for what is currently a closed set of three. The pull behaviour that accompanies a configured reference is settled separately in [ADR 029](029-configurable-backend-image-refresh.md), which this decision depends on and which is motivated independently by [gh#4](https://github.com/agenticsnz/sovereign-engine/issues/4).

**Two: the image is chosen by measurement, not by this ADR.** No candidate above is adopted. `GpuType::Rocm` is not added as part of this decision; it is added when a ROCm image is selected, and the addition is mechanical once selection is made.

### Measurements that select an image

All are taken on the existing local hardware and the existing container configuration. No host modification is proposed or implied.

1. **Size.** Record the actual compressed size of each candidate. The 0.31 GB, 1.22 GB and 7.71 GB figures are measured; ~1.5–2 GB for the slim build is a **projection** and is unverified.
2. **Throughput.** Compare Vulkan against each ROCm candidate on local gfx1151 hardware, with identical model, context and sampling settings, reporting tokens per second. This is the measurement that does not currently exist, and the one `llamacpp.rs:19-20` appears to assert on the strength of a vLLM result. **Vulkan remaining in place is a legitimate outcome.**
3. **Isolation.** Establish the minimum flags `llama-server` actually requires, starting from the project's current configuration and adding only what proves necessary. Record specifically whether it runs with: no `--privileged`; no `--network=host`; no `--ipc=host`; no `SYS_PTRACE`; and Docker's default seccomp profile. **If AMD's full flag set proves required, no ROCm candidate is viable** and the isolation model would need renegotiating in a separate decision, which this ADR does not authorise.
4. **Shared memory.** Establish the minimum `--shm-size`, against the current unset value and Docker's 64 MB default, and whether the default suffices.
5. **Device permissions.** Determine whether `rw` on `/dev/dri` and `/dev/kfd` (`llamacpp.rs:217`, `:223`) is required, or whether narrower permissions work.
6. **Group inheritance side effect.** Audit what `gpu_device_gids()` (`llamacpp.rs:386-413`) grants. It adds the host GIDs owning `/dev/dri` and `/dev/kfd` as supplementary groups. If any of those GIDs owns host resources beyond the GPU device nodes, the backend inherits access to them. Record the GIDs and what they own.

Items 1 and 2 select the image. Items 3 to 6 qualify any ROCm result, and item 3 can veto it.

## Consequences

- **Positive:** Comparing candidates costs a configuration change rather than a rebuild, which is what makes the deferral defensible.
- **Positive:** The change is behaviour-preserving when unset, so it carries no migration risk and can ship before any image decision.
- **Positive:** Removes a hard-coded third-party image reference from source.
- **Positive:** Corrects a misattributed performance figure and an ambiguous seccomp claim in the source comment at `llamacpp.rs:18-23`, both of which currently read as settled findings.
- **Negative:** Configuration surface grows by three variables, and `pull_backend_images()` must be kept consistent with `start_llamacpp()` or prefetch and launch will diverge.
- **Negative:** The GPU backend question stays open past this ADR. Nothing is gained operationally until the measurements are run.
- **Negative:** A misconfigured reference produces a pull or create failure at container start rather than at proxy startup, so a bad value surfaces late.
- **Risk, unquantified:** Read-write GPU device access and host-GID group inheritance are existing properties of the Vulkan path, not new ones. ROCm may exercise the KFD interface more heavily; the extent is item 6.
- **Unresolved:** Vulkan-versus-ROCm throughput remains unmeasured at the time of writing.

Independent of this ADR and not waiting on it: `docs/API.md:403` and `:733` publish `"gpu_type": "rocm | cuda | none"` while `from_str` (`llamacpp.rs:35-40`) maps anything other than `vulkan` to `None`. A client requesting `rocm` today receives HTTP 200, a container name, and a CPU-only container. That published contract should be corrected regardless of which image is eventually selected.

## Dependencies

| ADR | Relationship | Note |
|---|---|---|
| [001](001-llamacpp-over-vllm.md) | Refines | Corrects two claims originating there: the 0.6–2.4 t/s figure describes vLLM images, and "gfx1151 is not in the ROCm supported GPU list" is outdated. Removing vLLM still stands. |
| [003](003-model-host-path.md) | Depends On | Backend images bind-mount `MODEL_HOST_PATH` read-only at `/models` regardless of variant; the host-path distinction applies unchanged. |
| [006](006-strix-halo-gpu-tuning.md) | Relates To | Unified-memory reporting sums `mem_info_vram_total` and `mem_info_gtt_total`. The backend variant does not change how memory is observed. |
| [008](008-container-uid-allocation.md) | Relates To | Backends run as non-root UIDs in 10000–65000 and need `/dev/dri` and `/dev/kfd` plus `group_add` (`llamacpp.rs:211-238`, `:386-413`). All variants share this mechanism. |
| [029](029-configurable-backend-image-refresh.md) | Depends On | Settles the pull policy and version observability for configured image references. The configurability decided above is the mechanism 029 assumes, and 029 supplies the upgrade path that makes deferring an image choice affordable. |

**External dependency.** The slim ROCm candidate is produced by a separate repository we oversee. That repository is not an ADR and is not versioned by this one; the coupling is tracked by image reference rather than by an ADR link. No circular references. This ADR supersedes nothing.

## Governance

| | |
|---|---|
| **Decision-maker** | Project maintainer |
| **Review date** | *Placeholder.* To be set on completion of the six measurements, or 2026-12-30, whichever is sooner. |

## Status History

| Date | Status | Note |
|---|---|---|
| 2026-09-30 | Proposed | Configuration change decided; image selection deferred to measurement. No candidate selected. Size and throughput figures for the slim ROCm candidate are projections. |
