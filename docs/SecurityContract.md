# Security contract for gold base container images

This contract applies to services built from this project's runtime images once they are published. It states what the base is designed to provide and what each service team must do before production use. The [README](../README.md) shows project status. The [Hardening Guide](BaseRuntimeHardeningGuide.md) has implementation steps.

> **Status:** This repository currently contains documentation only. The image properties below describe the intended design. Verify them against the published image when it becomes available.

## What the base provides

The runtime image is designed to contain the language runtime without a normal shell, package manager, compiler, or other builder tools. Its default user is non-root (`65532:65532` in the project notes). Files that the base copies into the runtime stage are intended to be root-owned and not writable by that user.

The planned pipeline publishes a digest and a CycloneDX SBOM. It builds for `amd64` and `arm64`. The workflow notes describe a Wiz scan of the AMD64 image. They do not establish a separate ARM64 scan or a scan of your final service image.

These properties apply to the published base as built. Your Dockerfile can override the user, add tools, or change file permissions. Verify the final service image before relying on them.

## What the service team must do

| Area | Requirement |
| --- | --- |
| Image selection | Use the published **runtime** image by its full digest. Do not run a builder image in production. Review new digests before updating. |
| Final image | Build dependencies in a separate stage. Copy only what the service needs. Do not add a shell, package manager, compiler, debugger, or startup package installation. |
| User and files | Run as non-root. Keep application files and executables owned by root and not writable by the runtime user. Start the process directly without a shell. |
| Filesystem | Set a read-only root filesystem. Provide only the writable mounts the service needs. Use a size-limited `/tmp` with `noexec`, `nosuid`, and `nodev` where the platform can enforce those flags. |
| Process | Disable privilege escalation, drop all Linux capabilities, and apply `RuntimeDefault` seccomp or a stricter approved profile. |
| Trust | Verify the selected language's CA trust store. Add the public or internal CAs your service needs, and test TLS. |
| Access | Give the service only the IAM permissions and network access it needs. Keep secrets out of the image. |
| Application | Test and scan the complete service image. Review application vulnerabilities. Test on every CPU architecture used in production. |

Use equivalent settings on platforms other than Kubernetes. If a platform cannot enforce a required setting, record the gap and get it reviewed under your normal security exception process. The [Hardening Guide](BaseRuntimeHardeningGuide.md) gives examples and checks.

> **Kubernetes `/tmp` warning:** A memory-backed `emptyDir` alone does not guarantee `noexec`, `nosuid`, or `nodev`. Standard Pod specs have no field to set these mount options on `emptyDir`. Verify an approved platform mechanism before claiming the flags are active.

## Before production release

The service owner must be able to show the pinned base digest, a scan and tests of the final image, the effective deployment settings, and any approved exceptions. Check ARM64 application compatibility if the service runs on ARM64.

The base cannot protect against vulnerable application code, overly broad IAM, exposed services, or unsafe secrets handling. This repository does not enforce the deployment requirements. Ownership of the final service and its deployment stays with the service team.
