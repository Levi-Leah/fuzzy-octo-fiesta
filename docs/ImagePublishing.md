# Image publishing and supply-chain boundaries

This page explains the **gold base-image pipeline**, not how to build a service. Application teams should start with the [onboarding guide](guides/OnboardingGuide.md) or [migration guide](guides/MigrationGuide.md).

This is a summary of the pipeline documentation in `ollololololo/README-v2.md`. **The workflow implementation is not included in this documentation-only repository.** Confirm current behavior and published results in the live pipeline and image catalog; do not treat this page as proof that a control ran for your service.

## Builder and runtime images

Each language has a **builder** image for creating application artifacts and a **runtime** image for production. Language-specific builds are coordinated through a shared GitHub Actions workflow. Each language/version/image type can be built independently.

The pipeline described in the source documentation:

1. Validates build inputs and selects the Dockerfile and language version.
2. Authenticates to Artifactory with a short-lived OIDC-based token.
3. Builds and publishes multi-architecture base images for `linux/amd64` and `linux/arm64`.
4. Resolves a published digest, then scans the **AMD64** image using Wiz.
5. Generates a CycloneDX software bill of materials (SBOM).
6. Records image digest metadata and, where available, attaches build, SBOM, and scan evidence.

The source describes runtime-image scanning as stricter than builder-image scanning. The **latest** source README lists language refresh workflows scheduled on **Monday at 02:00 UTC**. The previous source version listed Monday, Wednesday, and Friday; use the live workflow configuration to confirm the effective schedule.

**Check the current pipeline before relying on any of these details.** The supplied language READMEs describe earlier phases in which some publishing, scanning, and signing features were planned rather than complete.

## What the published base does not verify

- The base-image scan does **not** scan application code or application dependencies added by the consuming team.
- The described scan pulls **AMD64**, not a separate ARM64 image for security analysis.
- A multi-architecture base does not guarantee that application dependencies or native binaries work on ARM64.
- The published digest identifies the base-image artifact. It is not proof that your final application image was scanned or hardened.

Application teams must still pin the published base digest, review their own dependency lockfiles, scan the **final service image**, test the required architectures, and verify deployment security settings.

## Why multi-architecture support matters

A multi-architecture image reference may resolve to different images for AMD64 and ARM64 nodes. This helps with mixed-node Kubernetes or ECS environments and future ARM adoption.

Your service must still support each target architecture. For example, compiled binaries, Python wheels, Node native add-ons, and architecture-specific system libraries can differ between nodes. Test the **final service image**, not only the base.

## Responsibility split

| Base-image pipeline | Consuming application team |
| --- | --- |
| Builds and publishes shared builder/runtime images | Builds the final service image |
| Publishes base-image digest and pipeline evidence | Pins the chosen base digest and records the final digest |
| Scans a base-image artifact as described above | Scans application dependencies and the final image |
| Provides a minimal runtime starting point | Applies non-root, read-only filesystem, mount, process, IAM, and network controls at deployment |

See the [Security Contract](SecurityContract.md) and [Hardening Guide](BaseRuntimeHardeningGuide.md) for requirements.

## Source and verification

- Sources: `Levi-Leah/ollololololo/README-v2.md` (detailed publishing description) and the later README screenshots supplied for this review (version and schedule changes).
- The language README screenshots provided for this documentation also describe planned states and historical workflows.
- For operational decisions, use the actual workflow configuration, image catalog, and current run evidence.

