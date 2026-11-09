# Security contract for gold base container images

This contract applies to application teams that use a gold runtime image. It separates what the base image provides from what the service team must provide. The [README](../README.md) explains image selection. The [Hardening Guide](BaseRuntimeHardeningGuide.md) covers implementation and verification.

## Provided by the runtime base

- A minimal runtime without the builder's compiler, package manager, and normal shell tools (distroless for most languages; chiseled for .NET)
- A non-root default user (typically `65532:65532` for the described distroless images; `1654:1654` for the described .NET chiseled image)
- Read-only, root-owned files that the base build copies into the runtime image
- A published image digest and build evidence, including a CycloneDX SBOM

These protections cover the base image as published. They do not make the whole container filesystem read-only. A service Dockerfile can also remove them by changing the user, file permissions, or image contents.

## Required from the service team

| Area | Requirement |
| --- | --- |
| Image | Use the published runtime image by digest. Build dependencies separately and keep shells, package managers, compilers, and debuggers out of the final image. |
| User and files | Run as non-root. Keep application files and executables root-owned and not writable by the runtime user. Start the application directly, without a shell. |
| Filesystem | Set a read-only root filesystem. Provide only the writable mounts the service needs. Use a size-limited `/tmp` with `noexec`, `nosuid`, and `nodev` where the platform can enforce those flags. |
| Process | Disable privilege escalation, drop all Linux capabilities, and use RuntimeDefault seccomp or an approved stricter profile. |
| Trust | Provide and test the CA trust your application needs, including internal CAs. Check the selected language's trust store. |
| Access | Limit IAM permissions and outbound traffic to what the service needs. Keep secrets out of the image. |
| Application | Review application security risks. Scan and test the complete service image. Test on every CPU architecture used in production. |

The runtime image cannot set deployment controls on its own. The published image's actual user and trust store must be checked before deployment. Configure them in your orchestrator and check the effective workload. See the [Hardening Guide](BaseRuntimeHardeningGuide.md) for an example security context and a release checklist.

> **Kubernetes `/tmp` note:** A memory-backed `emptyDir` alone does not guarantee the `noexec`, `nosuid`, or `nodev` mount flags. Verify that your platform enforces them. If it cannot, document the gap and seek a security review.

## Application security and network access

A minimal runtime reduces the tools available to a compromised process, but **does not fix application vulnerabilities**. Service teams must review unsafe deserialization, dynamic code execution, template injection, and other language-level remote code execution paths.

Allow only required outbound connections. Restrict cloud IAM to the actions and resources the service uses; avoid broad wildcard permissions. A compromised application inherits its workload credentials and permitted network reach.

## Limits and release evidence

The base cannot fix unsafe application code, broad IAM permissions, exposed services, or secrets handling. The workflow scans an AMD64 pull of the base image. This does not cover the ARM64 variant or your final service image.

Before release, the service owner should be able to show the pinned base digest, final image scan, application tests, deployment settings, and approved exceptions. This repository does not enforce deployment settings. The service team remains responsible for its final image and workload.
