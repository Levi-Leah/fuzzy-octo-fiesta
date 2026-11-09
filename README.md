# Gold base container images

This project builds shared builder and runtime images for application teams. A builder image has the tools needed to prepare an application. A runtime image contains what the application needs to run, without those build tools. Teams use the images from their own service repositories.

The shared images give services a consistent, smaller production base. They are built for both `linux/amd64` and `linux/arm64`, so the base can run in mixed architecture environments. Your application and its dependencies must also support the architecture you deploy.

## Available image families

| Language | Versions | Runtime base | Build notes |
| --- | --- | --- | --- |
| Python | 3.13 | Distroless | See the Python README for runtime details |
| Go | 1.26, 1 | Distroless static | `1` tracks the current Go 1 release |
| Node.js | 20, 24 | Distroless | `pnpm` is in the builder only |
| Java | 21, 25 | Distroless | Maven 3 is in the builder. Java uses the JDK trust store |
| Rust | 1.90, 1 | Distroless static | The builder targets `musl`. `1` tracks the current Rust 1 release |
| .NET | 10 | Chiseled Ubuntu .NET runtime | Non-root UID/GID 1654; check published image details |

These are the versions listed in the latest supplied source README. Confirm actual availability in the published image catalog or Artifactory before using one. Java's upstream base is described as not digest-pinned.

## Guides for application teams

- [Onboarding guide](docs/guides/OnboardingGuide.md) — use gold images for a new service.
- [Migration guide](docs/guides/MigrationGuide.md) — move an existing service to gold images.
- [Istio startup checks](docs/guides/IstioStartup.md) — optional checks for services using Istio.

## Use an image

1. Choose a language and version. Read that language's `README.md` for its build and startup details.
2. Find the builder and runtime image digests in the published Gold Base Image Digests page or Artifactory. Use the full published registry reference and digest in your Dockerfile. A digest fixes the exact image version you build from.
3. Build your application and dependencies with the builder image. Copy only the required application files into a final stage based on the runtime image.
4. Scan and test the final service image. Apply the deployment controls in the [Security Contract](docs/SecurityContract.md). The [Hardening Guide](docs/BaseRuntimeHardeningGuide.md) shows how to implement and verify them.

> **Important:** The runtime base does not add your application, configure your deployment, or grant access to other services. Your team owns the final image, CA trust needed by the application, IAM permissions, network access, and tests.

For pipeline scope, publishing evidence, and the difference between base-image scans and application scans, see [Image publishing and supply-chain boundaries](docs/ImagePublishing.md).

## How images are refreshed

Each language has its own refresh workflow, scheduled for **Monday at 02:00 UTC** according to the latest source README. The shared workflow builds and pushes multi-architecture images to Artifactory, records their digests, and generates a CycloneDX SBOM. A successful run updates the published digest records.

The workflow scans an AMD64 pull with Wiz. Runtime image findings can fail the build. Builder image findings are informative. A base image scan does not cover your application dependencies or provide a separate ARM64 scan. Scan the final service image and test it on each architecture you use.

See the [Security Contract](docs/SecurityContract.md) for the split between image protections and team responsibilities. Use the [Hardening Guide](docs/BaseRuntimeHardeningGuide.md) when building and deploying a service.
