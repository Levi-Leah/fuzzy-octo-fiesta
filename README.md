# Gold base container images

This project is intended to provide shared container base images for Python, Go, Node.js, Java, and Rust. Each language has a builder image for compiling or installing dependencies and a smaller runtime image for running the service. Teams can build their own service images on top of these bases.

The benefit is a consistent starting point. Build tools stay out of production images, and teams can use a published digest to choose an exact base image version.

> **Current status:** This repository contains documentation only. It has no image build files, published digests, or images ready for use. The steps below apply after images and access instructions are published.

## Use an image

1. Choose a published language and version. Check its language-specific instructions for build and startup details.
2. Get the builder and runtime image references from the published digest record. Pin each image using its full registry address and digest, for example `registry/path@sha256:...`. The registry address and real digests have not been provided yet.
3. Build your service and its dependencies in the builder stage. Copy only the files needed to run it into a final stage based on the runtime image.
4. Scan and test your **final service image**. Deploy it with the settings required by the [Security Contract](docs/SecurityContract.md).

The planned runtime images use a small distroless base and start as a non-root user. Your service image and deployment can change those properties. Follow the [Hardening Guide](docs/BaseRuntimeHardeningGuide.md) to keep them and verify the result.

## Planned languages

| Language | Versions in the project notes |
| --- | --- |
| Python | 3.13 |
| Go | 1.26 and floating 1 |
| Node.js | 20 and 24 |
| Java | 21 and 25 |
| Rust | 1.90 and floating 1 |

These are planned versions, not a list of available images. .NET is mentioned in the project notes, but its versions and base image have not been decided. The floating Go and Rust versions may change when images are refreshed.

## What your team still owns

- Your application code, dependencies, final image, and tests
- The CA certificates your service needs, including internal CAs where applicable
- A read-only root filesystem, restricted temporary storage, non-root execution, and other deployment controls
- IAM permissions, secrets, network access, and application security
- Compatibility of your application and dependencies with each CPU architecture you deploy

The planned pipeline builds `amd64` and `arm64` bases, scans an AMD64 pull with Wiz, and produces an SBOM and digest record. That scan does not cover the complete service image or establish ARM64 scan coverage. The [Security Contract](docs/SecurityContract.md) explains the requirements and limits. The [Hardening Guide](docs/BaseRuntimeHardeningGuide.md) shows how to apply and check them.
