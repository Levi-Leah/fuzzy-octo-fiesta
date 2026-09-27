# Gold base container images

This project defines shared builder and runtime container images for Python, Go, Node.js, Java, and Rust services. Application teams will use published images as the base for their service images. This repository currently contains the project documentation; no image build code, digest records, or published image references have been added here yet.

The planned builder image contains the tools needed to build a service. The runtime image is a smaller, distroless base intended for production. Shared bases would give teams a consistent starting point, reduce tooling shipped to production, and provide centrally refreshed image digests and build evidence. They do not secure application code or deployment settings on their own.

> **Status:** No image in this repository is currently ready for adoption. The source material lists .NET as planned, with versions and base image undecided. Check actual registry records and build status before adopting any language or version.

## Choose and use an image

1. When images are published, pick an available language and version, then check its language-specific instructions for build and runtime details.
2. Find the published builder and runtime references in the digest records or registry instructions provided by the image maintainers. Use the **full registry reference with its published digest** (`registry/path@sha256:...`), not an unqualified tag. A digest identifies a particular published image; update it deliberately when a refreshed image is released.
3. In your service repository, build application code and dependencies in a builder stage. Copy only what the application needs into a final stage based on the matching runtime image. The base image contains the language runtime, not your application.
4. Check the resulting service image and deploy it using the settings in the [Security Contract](docs/SecurityContract.md). Follow the [Hardening Guide](docs/BaseRuntimeHardeningGuide.md) for a practical build and deployment checklist.

> **Note:** Registry paths, access permissions, and exact image references are not specified in the available project material. Take them from the published records and your organization's registry access instructions; do not paste a sample path into production.

| Language | Versions listed in the project material | Runtime base | Build notes |
| --- | --- | --- | --- |
| Python | 3.13 | Distroless | Check the language README for dependencies and startup |
| Go | 1.26 pinned; 1 floating | Distroless static | Verify the service binary targets the required architecture |
| Node.js | 20, 24 | Distroless | `pnpm` is available in the builder only |
| Java | 21, 25 | Distroless | Maven 3 in builder; Java uses its JDK trust store |
| Rust | 1.90 pinned; 1 floating | Distroless static | Builder targets `musl` |

These versions come from the supplied project description; they are not proof of availability. Floating `1` versions can change on refresh. Java's upstream base is reported as not digest pinned; review that limitation before relying on a fixed upstream input.

## What the project publishes

The supplied design calls for each language directory to contain `pipeline/Dockerfile.builder`, `pipeline/Dockerfile.runtime`, `buildargs.conf`, `records/gold-digests.json`, and a language `README.md`. It names `.github/workflows/gold-base-image.yml` as the shared workflow, with language workflows requesting a version and image type. These files are not present in this documentation repository yet.

The described pipeline would build and push `linux/amd64` and `linux/arm64` images to Artifactory, resolve the published digest, scan the AMD64 image with Wiz, create a CycloneDX SBOM, and write digest metadata for the record update. The runtime scan is specified to use a policy that can fail the workflow; the builder scan is informative. The SBOM is specified as a workflow artifact with 90-day retention. Attestations are designed to attach available build, SBOM, and scan evidence when the Wiz report is available.

Language refresh workflows are described as running Monday, Wednesday, and Friday at 02:00 UTC once implemented. A new digest would become available after a successful publish and record update; a schedule is not a promise that every build succeeds. Check the record and workflow result before updating your service. The workflow summary describes an AMD64 vulnerability scan, so do not infer that the ARM64 variant received a separate Wiz scan.

The multi-architecture base allows a service image to run on either architecture **if** its application code and dependencies also support both. Teams must verify their own native modules, libraries, and compiled binaries before scheduling on ARM64.

## Your part of the security model

The runtime base is designed to default to a non-root user and omit a shell, package manager, and build tools. Your final service image can change those properties. Once images are available, keep the final image minimal and non-root; supply application dependencies and the appropriate CA trust material; then set a read-only root filesystem, a restricted writable `/tmp`, and process protections at deployment. Restrict IAM and network access and scan/test your final service image.

The [Security Contract](docs/SecurityContract.md) states the required controls and who owns each one. The [Hardening Guide](docs/BaseRuntimeHardeningGuide.md) explains how to apply and verify them. Read both before production deployment.

> **CA trust:** The supplied documents conflict on whether a CA bundle is included. Confirm the selected language image's trust store and your application's TLS behavior. Add organization-specific trust in your service image or deployment where needed; Java uses a JDK trust store.
