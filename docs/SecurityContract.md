# Security contract for gold base container images

## Scope and use

This contract applies when a team builds a service image from a runtime image published by this project. It sets out what the shared base is intended to provide and what the consuming team must implement. Builder images are build-time tools and must not be used as production runtimes. No images or build code are present in this documentation repository yet; these requirements apply when the images become available. See the [project README](../README.md) for status and the [Hardening Guide](BaseRuntimeHardeningGuide.md) for implementation and checks.

> **Important:** This repository does not enforce deployment settings or inspect every final service image. Starting from a gold base does not establish compliance. Teams must verify their final image and deployment.

## Responsibility split

| Control | Base image / pipeline | Consuming team |
| --- | --- | --- |
| Minimal runtime | Produces a language runtime base without builder tools, package manager, or normal shell | Keep the final image free of these tools; do not use a builder image in production |
| User and files | Runtime base is designed to default to non-root UID/GID `65532:65532`; copied runtime files are intended to be root-owned and read-only | Keep a non-root user; make application files non-writable by that user; test the **final** image |
| Image identity | Publishes a digest and build evidence, including a CycloneDX SBOM | Pin the full published digest, review refreshes, and scan the complete service image and its dependencies |
| Root filesystem | Cannot set a deployment's root filesystem read-only | Enable a read-only root filesystem and explicitly mount only the needed writable paths |
| Temporary files | Does not create a secure runtime mount | Provide a size-limited writable `/tmp` with `noexec`, `nosuid`, and `nodev` where the platform can enforce them |
| Process isolation | Cannot set deployment security context | Disable privilege escalation, drop all capabilities, and use RuntimeDefault seccomp or stricter |
| TLS trust | Language runtimes differ; shared and internal CA coverage is not established by these sources | Verify trust for the selected language; provide required CA certificates and test TLS, including internal endpoints |
| Service access | Does not control service permissions or traffic | Restrict IAM, secrets, inbound access, and outbound destinations |

The intended base-image properties must be checked against the actual published image and its language-specific instructions once available. A service Dockerfile can override the user, add tools, change permissions, or copy unsafe files; the project cannot prevent it.

## Required conditions for production use

The consuming team **must**:

1. Use a published runtime image by digest and retain the matching build record. Update the digest through the team's normal change process.
2. Build the application in a separate stage. Copy only the executable, application files, runtime dependencies, and trust material that it needs. Do not add a shell, package manager, compiler, or debugger to the production stage. Do not install dependencies at container startup.
3. Run as a non-root user. Keep executable and application paths owned by root and not writable by the runtime user. Launch the process directly with an exec-form `ENTRYPOINT` or `CMD`, without a shell or production auto-reload mode.
4. Configure `readOnlyRootFilesystem: true` (or the platform equivalent). Give the application only explicitly required writable mounts. Use a bounded temporary mount for `/tmp` with `noexec`, `nosuid`, and `nodev` **if the platform supports and enforces those mount flags**. Do not claim these flags are active solely because `/tmp` is an in-memory volume.
5. Set `allowPrivilegeEscalation: false`, drop `ALL` Linux capabilities, and apply `RuntimeDefault` seccomp or a stricter approved profile. Use the equivalent controls on non-Kubernetes platforms and verify them in the deployed workload.
6. Grant only the IAM permissions the service needs. Scope resources and actions; document and approve any exception requiring broad permissions. Limit egress to required destinations and protect secrets outside the image.
7. Check application-specific risks, including unsafe deserialization, dynamic code execution, untrusted templates, exposed endpoints, and data access. Test both `amd64` and `arm64` if the service will run on both; base-image support alone does not validate application dependencies.
8. Scan and test the **final service image**, including added packages and copied application dependencies. Record any exceptions and their owner under the organization's normal review process.

> **Kubernetes note:** `emptyDir.medium: Memory` provides an in-memory volume but does not by itself set `noexec`, `nosuid`, or `nodev`. Standard Pod volume specs do not expose arbitrary mount options for `emptyDir`. Use an approved platform mechanism that enforces the flags, or document the gap and obtain a security review. See the [mount guidance](BaseRuntimeHardeningGuide.md#4-provide-writable-tmp).

The exact configuration depends on the orchestrator. The [Hardening Guide](BaseRuntimeHardeningGuide.md#5-set-deployment-process-controls) includes a Kubernetes security context and verification steps. On ECS or another platform, map each requirement to its supported setting rather than copying Kubernetes YAML.

## Boundaries and exceptions

- A read-only application tree is not a read-only root filesystem. The latter is a deployment setting. Writable paths introduced by the service image or by mounts must be reviewed separately.
- `noexec` limits direct execution from a mount; it does not stop an interpreter from reading a script. Distroless packaging does not eliminate application-level code execution flaws.
- A runtime image cannot guarantee that application TLS succeeds. Trust stores vary by language and service; see the [trust-store step](BaseRuntimeHardeningGuide.md#3-add-ca-trust-when-needed).
- The described pipeline scans an AMD64 pull of the base. Its result is not a scan of the ARM64 variant or of your final application image. A 90-day workflow artifact retention period is not a long-term evidence-retention promise.
- If a service needs extra system libraries or a different runtime behavior, request a change to the shared base or obtain an approved exception. Do not silently weaken the final image or deployment.

Before production release, the service owner should be able to show the pinned digest, final-image scan, deployment settings, application architecture tests where relevant, and approved exceptions. This project supplies guidance and build evidence; enforcement and approval remain with the organization's deployment and review processes.
