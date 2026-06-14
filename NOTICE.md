# NOTICE

## Repository license

The build system in this repository — `Dockerfile.override`, the GitHub Actions
workflow, configuration files, and documentation — is licensed under
Apache-2.0. See `LICENSE` for the full text.

No Beszel source code is contained in this repository. Upstream source is
cloned fresh at build time and discarded after the image is produced.

## Image license

The container images produced by this repository and published to
`ghcr.io/thekroll-ltd/beszel` contain a compiled build of the Beszel **hub**
(the dashboard server), which is licensed under MIT. The images are therefore
distributed under the MIT license, inheriting upstream's terms.

- Upstream: https://github.com/henrygd/beszel
- Upstream license: https://github.com/henrygd/beszel/blob/main/LICENSE

The mirror build adds no new code to the application. `Dockerfile.override`
only:

- builds the frontend bundle and the Go binary from upstream source on
  pinned, current toolchains (bun + Go), and
- rebases the build onto digest-pinned base images.

No Beszel source is patched.

## Scope: hub only

This mirror builds **only** the Beszel hub image. It does not build or publish
the Beszel agent (`beszel-agent`) or its GPU variants (`-nvidia`, `-intel`).
Run the upstream agent images, or a separate mirror, for those.

## Architecture

Images are **linux/amd64 only**, matching the oss-mirror-build pipeline's
single-architecture build. Upstream publishes arm64/armv7 hub images; this
mirror does not.

## No SLA

This mirror is THEKROLL's internal build, published publicly as reference.
No service-level agreement, no support commitment, no compatibility guarantee.
For production-critical deployments, fork the template at
`github.com/THEKROLL-LTD/oss-mirror-build` and run your own pipeline.
