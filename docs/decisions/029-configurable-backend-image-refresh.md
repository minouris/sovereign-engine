# ADR 029: Configurable backend image references and refresh

**Status:** Proposed
**Date:** 2026-09-30
**Related:** [ADR 028 — Configurable ROCm backend image](028-configurable-rocm-backend-image.md), [gh#4](https://github.com/agenticsnz/sovereign-engine/issues/4), [ADR 015 — Per-container API keys](015-per-container-api-keys.md)

## WH(Y) Decision Statement

**In the context of** the proxy selecting a llama.cpp backend image from two hard-coded constants, and refreshing neither because a local image is never re-pulled,

**facing** a reported fault in which the engine serves a llama.cpp build roughly 1,500 releases behind current, which cannot load a model using a four-month-old architecture and intermittently returns HTTP 500 on constrained JSON requests, and which is therefore pinned not by policy but by a short-circuit in the prefetch path,

**we decided for** environment-backed image references per device type, with an explicit pull policy that distinguishes a floating tag from a pinned one, so that upgrading llama.cpp is a configuration change rather than a code change and is not silently skipped when a stale image is already present locally,

**and neglected** relying on the floating upstream tags alone, which appear to promise currency but deliver the opposite once an image is cached, and separately declined to pin a specific build number in source, since that would reintroduce the hard-coding this ADR removes,

**to achieve** a deployment where the running llama.cpp version is known, chosen deliberately, and reportable, and where a new backend image variant can be trialled by configuration rather than by rebuild,

**accepting that** a pull policy introduces a second configuration surface that can be set inconsistently with the image reference, that refresh behaviour must be documented because a floating tag and a pinned tag now mean different things, and that this ADR does not itself upgrade the deployed image.

## Context

Two independent faults motivate this, and only one of them is a version problem.

### The image is stale because prefetch skips it

`LLAMACPP_IMAGE_CPU` and `LLAMACPP_IMAGE_VULKAN` (`proxy/src/docker/llamacpp.rs:15-16`) reference the floating upstream tags `ghcr.io/ggml-org/llama.cpp:server` and `:server-vulkan`. Upstream's [docker documentation](https://raw.githubusercontent.com/ggml-org/llama.cpp/master/docs/docker.md) lists these as current variant tags with no version qualification, and separately documents that GPU images "are not currently tested by CI beyond being built". A floating tag should therefore track upstream, and a fresh deployment would receive a current build.

Deployed installations do not. `pull_image` (`docker/mod.rs:546-583`) returns before pulling when the image is already present:

```rust
// Check if image already exists locally
let full_ref = format!("{}:{}", repo, tag);
if docker.inspect_image(&full_ref).await.is_ok() {
    info!(image = %full_ref, "Image already present");
    return;
}
```

`pull_backend_images()` (`docker/mod.rs:196-214`) is the only caller, invoked once at startup from `main.rs:109`. So the image is fetched on first run and never again, regardless of how far the tag has moved since. The result is that a floating tag produces a **pinned** deployment by accident: the build present at first install persists indefinitely, and nothing reports the drift.

This is the mechanism behind [gh#4](https://github.com/agenticsnz/sovereign-engine/issues/4). The reporter determined from `system_fingerprint` that the engine serves `b8925-0adede866` (`parser: fix structured output bug (#22302)`, 2026-04-24) against a current `b10434` (2026-08-14) — approximately 1,500 builds and four months behind. The reporter could only establish this indirectly because the proxy exposes no `/props`, `/health` or `/version` route; all three return 404, and the build had to be read out of a chat-completion response.

Two consequences follow. First, the version is not pinned by any deliberate decision, so there is no recorded intent to preserve or update it. Second, the version is not observable, so drift is only discoverable by inference from model output. The `check_llamacpp_health` probe (`llamacpp.rs:361-371`) queries `/health` and returns only a status, and upstream's `/health` is a public endpoint that reports 503 while loading and 200 with `{"status":"ok"}` when ready — it carries no build identifier.

### The version gap has two distinct causes

**Reason 1 — a model that cannot load at all.** [gh#4](https://github.com/agenticsnz/sovereign-engine/issues/4) reports that Qwen3.8-27B introduced a new `qwen35` architecture, published as `unsloth/Qwen3.8-27B-GGUF`. llama.cpp requires explicit per-architecture support to load a GGUF and refuses an unknown `general.architecture` at load time. `b8925` predates the model, so it cannot support it. The reporter cites [ggml-org/llama.cpp#27090](https://github.com/ggml-org/llama.cpp/issues/27090) as evidence the architecture works on `b10430` and `b10434`, and explicitly declines to name a minimum version, since the introducing build was not pinned. `b10434` or later is a known-good target.

**Reason 2 — an intermittent server-side fault on the current build.** On `b8925`, a request whose prompt asks for JSON but which does not set `response_format` is answered through gpt-oss's harmony constrained-JSON channel, and the server then fails to parse its own model output and returns HTTP 500. The reporter measured ten runs: four of five returned 500 without `response_format`, five of five returned 200 with `response_format: {"type":"json_object"}`. The intermitcy is what made it expensive to diagnose, since it presents as a flaky server rather than a parser fault, and the workaround is a client-side patch for a server-side failure.

The reporter also flags a caveat on the target build: [ggml-org/llama.cpp#27090](https://github.com/ggml-org/llama.cpp/issues/27090) is open against `b10434`, where `llama-server` exits silently at approximately 520K prefill tokens with Qwen3.8-27B under YaRN rope-scale ×4, just under twice the native 262144 context. That affects extreme long-context use only and is recorded here so it is not a surprise after upgrading.

## Decision

Three changes, separable, the first two of which can land before any image decision.

**One: image references become configuration.** Each device type gets an environment-backed reference, defaulting to the value currently hard-coded. `config.rs` has no image setting today, so this introduces the first. With nothing configured the behaviour is unchanged, so the change is behaviour-preserving and carries no migration risk.

`pull_backend_images()` (`docker/mod.rs:196-214`) hard-codes the same two constants and must read the configured values in the same change. If it does not, the proxy prefetches one image and launches another, and the divergence is silent.

**Two: a pull policy, and a refresh path.** The `inspect_image` short-circuit is the direct cause of the staleness in [gh#4](https://github.com/agenticsnz/sovereign-engine/issues/4), and removing it unconditionally would be worse: it would re-pull a multi-gigabyte image on every proxy start. The distinction needed is between an image whose reference is a floating tag and one whose reference is pinned.

- A **pinned** reference — a specific tag or a digest — is not re-pulled when present. The existing short-circuit is correct for it and is retained.
- A **floating** reference is re-pulled when a refresh is requested, and the refresh is explicit rather than automatic at startup.

Because both cases are decided by inspecting the configured string, the policy needs no separate flag: a reference that names a version is pinned by construction. A `--pull` style option on the container-start request, or a periodic refresh, are both defensible; the choice is implementation detail and is not settled here.

The digest case is a known limitation to handle deliberately. `pull_image` splits the reference on the last colon (`docker/mod.rs:548`), which misparses a digest reference: `repo@sha256:abc` yields repository `repo@sha256` and tag `abc`. The reassembly at `docker/mod.rs:554` reproduces the original string, so the local-presence check still succeeds and a digest-pinned image resolves when present — but the pull path receives an invalid repository name and cannot fetch one. Any work here must correct that parse before digest pinning can be offered as a policy.

**Three: the running build becomes observable.** A configured deployment should be able to report which llama.cpp build it is running without inferring it from model output. Two options, both sufficient:

- Log the image reference and its resolved digest at container start, so the deployed version appears in proxy logs without a new endpoint.
- Expose the digest of the image backing a loaded model through the existing model listing, so the UI and API can display it.

The first is smaller and removes the need to guess. The second is what makes the drift visible to an operator who does not read logs. Neither requires a proxy route that upstream does not already provide.

## Consequences

- **Positive:** Upgrading llama.cpp becomes a configuration change, which is the stated motivation in [gh#4](https://github.com/agenticsnz/sovereign-engine/issues/4) and the reason this is separable from ADR 028.
- **Positive:** The pull policy removes an accidental pin, so a deployment's llama.cpp version follows an expressed intent rather than its install date.
- **Positive:** The change is behaviour-preserving when nothing is configured, so it can ship ahead of any image decision.
- **Positive:** Removes a hard-coded third-party image reference from source.
- **Positive:** Makes the version discoverable, which removes the inference from `system_fingerprint` that [gh#4](https://github.com/agenticsnz/sovereign-engine/issues/4) had to rely on.
- **Negative:** A pull policy is a second configuration surface that can disagree with the image reference, and the failure mode is silent — a re-pull that does not happen, or a pin that was not intended.
- **Negative:** Digest pinning is not available until the parse at `docker/mod.rs:548` is corrected, so tag-based pinning is the only immediately usable form.
- **Negative:** Refresh on start is expensive for a multi-gigabyte image, so the policy has to be explicit rather than automatic. An operator who expects a fresh image on restart will not get one unless the reference is floating.
- **Negative:** Fixing the drift in [gh#4](https://github.com/agenticsnz/sovereign-engine/issues/4) is not automatic. Once the policy exists, moving to a build supporting `qwen35` is still a deliberate operator action, and the constrained-JSON fault in reason 2 is only fixed by that same move.
- **Not addressed here:** the `qwen35` architecture support and the constrained-JSON parse fault are fixed by upgrading the image, not by this ADR. This ADR makes the upgrade a configuration change; it does not perform one.

## Dependencies

| ADR | Relationship | Note |
|---|---|---|
| [028](028-configurable-rocm-backend-image.md) | Supports | Supplies the motivation and a second consumer. ADR 028 defers its image selection to measurement and needs per-device-type references to compare candidates without a rebuild; the configurability decided here is the mechanism it assumes. Neither supersedes the other, and neither is complete without the other. |
| [015](015-per-container-api-keys.md) | Relates To | The per-container API key is passed to `llama-server` as `--api-key` at start. The same start path carries the image reference, so both changes touch `build_llamacpp_cmd` and `start_llamacpp`. |

No circular references. This ADR supersedes nothing. The dependency on ADR 028 is mutual by construction and is stated rather than hidden: 028 is the reason the mechanism is wanted, and 029 is how 028's deferral becomes affordable.

## Governance

| | |
|---|---|
| **Decision-maker** | Project maintainer |
| **Review date** | *Placeholder.* To be set when [gh#4](https://github.com/agenticsnz/sovereign-engine/issues/4) is closed, or 2026-12-30, whichever is sooner. |

## Status History

| Date | Status | Note |
|---|---|---|
| 2026-09-30 | Proposed | Drafted against [gh#4](https://github.com/agenticsnz/sovereign-engine/issues/4). The pull-policy mechanism and the version-observability option are not settled and are recorded as open. |
