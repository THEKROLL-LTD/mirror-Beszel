# mirror-Beszel

Hardened container mirror of the [henrygd/beszel](https://github.com/henrygd/beszel) **hub** (dashboard server). THEKROLL's internal Beszel image, published publicly as a reference build.

## What this repository does

Each night, a GitHub Actions pipeline in this repository:

1. Checks `henrygd/beszel` for a new release tag
2. If there is one, clones the upstream source and **rebuilds the hub from source** on a pinned, current Go toolchain (and builds the web UI with a pinned bun) instead of re-publishing henrygd's prebuilt image
3. Detects upstream drift in the files that define the build contract (`internal/dockerfile_hub`, `internal/site/embed.go`, `internal/site/package.json`, `.github/workflows/docker-images.yml`); blocks with an issue if anything changed since the last review
4. Scans the cloned source with Trivy (filesystem) and the built image with Trivy (container); produces SBOMs for both
5. If clean: pushes to `ghcr.io/thekroll-ltd/beszel` and opens a digest-pin PR
6. If findings: blocks the push, opens an issue, retains the full audit bundle for 90 days

## Scope: hub only

This mirror builds **only** the Beszel hub image — the central dashboard server you self-host once. It does **not** build the agent (`beszel-agent`) that runs on each monitored host, nor the GPU agent variants (`-nvidia`, `-intel`). Use the upstream agent images for those, or a separate mirror.

## Image

`ghcr.io/thekroll-ltd/beszel:<tag>` and `ghcr.io/thekroll-ltd/beszel@sha256:<digest>`.

Tags track upstream Beszel releases. Digests are the authoritative pin. The image is **linux/amd64 only**.

Drop-in for `ghcr.io/henrygd/beszel` (hub): same `ENTRYPOINT`/`CMD`, same `EXPOSE 8090`, same `VOLUME /beszel_data`, runs as root by default.

## Why rebuild instead of re-publish?

Beszel's hub is already a `scratch` image — a single static Go binary plus CA certificates. The usual mirror hardening levers (slimming a fat base, patching apt/apk CVEs, avoiding an opaque prebuilt base) **do not apply**: there is no OS layer to harden. The CVE classes that can appear in a scratch + Go-binary image are:

| CVE class | What the mirror does |
| --- | --- |
| **Go stdlib** (TLS / x509 / net/http) | **Remediated by nightly rebuild** on a pinned, current Go — a Go security release lands in our image on Go's cadence, not when henrygd next cuts a Beszel release |
| Go module deps (`go.sum`) | Detected + gated; not patched (rewriting `go.mod` would diverge from upstream) |
| OS / apt / apk | None — the image is `scratch` |
| Base image | We build only from the source at a tag, not from henrygd's published bytes; bases are digest-pinned |

So the value here is narrower than the OS-heavy mirrors: **Go-stdlib remediation + full supply-chain control**, not base hardening.

## The frontend-embed wrinkle

Upstream's `internal/dockerfile_hub` builds only the Go binary. The web UI is built **outside** the Dockerfile by upstream CI (`bun run build` → `internal/site/dist`) and embedded via `//go:embed all:dist`. Because `dist/` is git-ignored and never committed, a naive clone-and-build of the upstream Dockerfile fails to compile. Our `Dockerfile.override` is a self-contained 3-stage build (bun → Go → scratch) that produces the frontend bundle itself, so the pipeline needs no host-side build step.

## Differences from `ghcr.io/henrygd/beszel`

- **Frontend built in-image** (pinned bun), not on the CI runner.
- **Go toolchain pinned** to match `go.mod` (Go 1.26.3) — controls stdlib CVEs.
- **All base images digest-pinned** (Renovate keeps them current).
- `-trimpath` + `-buildvcs=false` for reproducibility.
- Single arch (linux/amd64); upstream also ships arm64/armv7.
- OCI labels populated.

The application code itself is byte-equivalent to upstream — no source patches.

## Optional hardening: non-root

The image runs as root (UID 0) by default, matching upstream, so it's drop-in. To run non-root, add a numeric `USER` to `Dockerfile.override` and ensure the mounted `/beszel_data` volume is writable by that UID (PocketBase needs to write there). This is left opt-in because changing the UID breaks existing deployments whose volumes are root-owned.

## No SLA

This is THEKROLL's own internal build, made public as a reference. There is no service-level agreement, no support commitment, no compatibility guarantee.

**For production-critical use**, fork the template and run your own pipeline: [THEKROLL-LTD/oss-mirror-build](https://github.com/THEKROLL-LTD/oss-mirror-build).

## License

The build system in this repository — workflow YAML, Dockerfile override, documentation — is licensed under Apache-2.0. See [`LICENSE`](LICENSE).

The container images produced here contain the Beszel hub, which is licensed under MIT. The images inherit MIT. See [`NOTICE.md`](NOTICE.md).

## Related

- **Upstream:** [github.com/henrygd/beszel](https://github.com/henrygd/beszel) (MIT)
- **Template this repo was forked from:** [THEKROLL-LTD/oss-mirror-build](https://github.com/THEKROLL-LTD/oss-mirror-build) (Apache-2.0)
- **Sister mirrors:** [mirror-Gokapi](https://github.com/THEKROLL-LTD/mirror-Gokapi), [mirror-Plausible](https://github.com/THEKROLL-LTD/mirror-Plausible), [mirror-uptime-kuma](https://github.com/THEKROLL-LTD/mirror-uptime-kuma)

## Maintained by

[THEKROLL](https://thekroll.ltd) — DevOps consultancy from Cyprus. For production-critical use, don't depend on this mirror; fork the template and run your own pipeline at [`THEKROLL-LTD/oss-mirror-build`](https://github.com/THEKROLL-LTD/oss-mirror-build).
