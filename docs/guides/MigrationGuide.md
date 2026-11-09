# Migration guide: Move an existing service to gold images

Use this guide when your service **already runs** and you want to replace its current builder or runtime images with the shared gold images.

A gold **builder** supplies build tools. A gold **runtime** is a small, hardened image used to run the finished application. The runtime usually has no shell or package manager. Your service must not depend on those tools when it starts.

For a new service, use the [onboarding guide](OnboardingGuide.md).

## 1. Record the current working configuration

Before changing the image:

1. Record the current application image digest and deployment configuration.
2. Save the existing Dockerfile and startup command.
3. List all runtime dependencies, including native libraries, certificates, writable directories, and external services.
4. Note any commands run at startup, such as database migrations or cache warm-up.
5. Confirm how to restore the existing release.

**Do not skip rollback preparation.** Rolling back the image may not undo a database schema change.

## 2. Choose the replacement images

1. Open the published gold image catalog.
2. Confirm that your language/version has published builder and runtime images.
3. Copy both full image references with their `sha256` digests.
4. Check the cluster CPU architecture and any native dependencies.

Use the **published image catalog**, not a README version list, to confirm availability. Digest pinning identifies the base image used in a given build.

## 3. Update the Dockerfile

1. Replace the old build stage with the appropriate gold **builder** image.
2. Install dependencies and build the application in that stage.
3. Use the matching gold **runtime** image for the final stage.
4. Copy only the required application files and runtime dependencies.
5. Set permissions before the shell-free runtime stage, or use `COPY --chown` / `COPY --chmod`.
6. Keep the runtime's default non-root identity unless the approved image requirements say otherwise.

Do not copy the entire builder filesystem. Do not add a shell or package manager back to the final image.

The [onboarding guide](OnboardingGuide.md) explains the general two-stage pattern and language-specific differences.

## 4. Replace shell-dependent startup commands

Review **all** of these locations:

- Dockerfile `CMD` and `ENTRYPOINT`.
- Kubernetes `command` and `args`.
- Entrypoint scripts, package-manager commands, and application startup hooks.

For example, this existing Kubernetes command is **not compatible** with a shell-free runtime:

```yaml
command: ["/bin/sh", "-c"]
args: ["npx prisma migrate deploy && node index.js"]
```

It requires both `/bin/sh` and `npx`. Move dependency installation to the builder, and run the installed application through its interpreter or binary.

If startup requires several steps, use a deliberately written application entrypoint or a suitable separate deployment task. Test errors and exit codes. Do not silently remove database migrations or other required work.

For Python, a direct command such as `python3 -m uvicorn ...` may be appropriate **if** the interpreter and module exist. A virtualenv console script is not automatically broken; verify its shebang, permissions, and dependencies.

**Check the actual image:** command paths differ across languages and images.

## 5. Check changes that can break the application

### Files and runtime identity

- The root filesystem should be read-only at deployment.
- Application files must be root-owned and not writable by the runtime user.
- Move legitimate temporary writes to approved writable mounts; disable unnecessary caches.
- Verify the selected runtime UID/GID. The supplied `:nonroot` images use **65532**, while the .NET chiseled image description uses **1654**.

### Certificates and external access

- Verify TLS against required endpoints. Supply CA certificates where needed.
- Java uses its JDK trust store; internal CAs may need to be imported into it.
- Check DNS, IAM permissions, secrets, and network policies.
- Do **not** disable TLS verification to make a test pass.

### Architecture and libraries

- Test every target architecture (such as `amd64` and `arm64`).
- Check native modules and compiled binaries, particularly Go/Rust static linking and .NET chiseled compatibility.
- DNS settings normally come from the container platform. Do not bake `/etc/resolv.conf` into the image as a routine workaround.

## 6. Apply the required deployment settings

Review the [Security Contract](../SecurityContract.md) and [Hardening Guide](../BaseRuntimeHardeningGuide.md). The deployment must use a read-only root filesystem, non-root execution, dropped capabilities, no privilege escalation, and the approved seccomp profile.

Check all writable mounts. In Kubernetes, a memory-backed `emptyDir` does not automatically enforce the required `noexec`, `nosuid`, and `nodev` flags. Verify the effective settings and use the platform-approved mechanism.

## 7. Test the migration in stages

1. **Build:** Confirm that all required runtime files exist and the final image is scanned.
2. **Start locally:** Run the final image as non-root with a read-only root filesystem. Confirm the real entrypoint works without a shell.
3. **Test in staging:** Verify startup, health checks, requests, shutdown, restarts, certificates, DNS, secrets, and external connections.
4. **Test the deployment controls:** Inspect the effective security context and writable mounts.
5. **Check startup tasks:** Confirm migrations, cache initialization, and other startup operations work correctly.
6. **Release gradually:** Use a canary or small rollout if supported. Watch restarts, readiness, connection errors, and application logs.
7. **Record the outcome:** Save the image digest, test evidence, configuration, and exceptions.

**Optional — Istio:** If the workload uses an Istio sidecar and makes outbound calls at startup, follow [Istio startup checks](IstioStartup.md). This is not a required step for services without Istio.

## 8. Roll back if needed

1. Restore the last known good **application image and matching deployment configuration**.
2. Check application health and the external connections it needs.
3. Review database schema compatibility separately; an image rollback may not reverse migrations.
4. Record the failed check and fix it before retrying.

Do not work around failures by permanently running as root, adding a shell, or disabling required security controls.

## Troubleshooting

| Symptom | First thing to check |
| --- | --- |
| `/bin/sh` not found | Shell-based Dockerfile or Kubernetes startup command |
| `npx` or another CLI not found | Build-only tool used at runtime; wrong executable path |
| Permission denied | Actual runtime UID, file ownership, execute permissions |
| Read-only filesystem error | Write to a path without an approved writable mount |
| TLS / unknown CA error | Correct certificates and language-specific trust store |
| Works on AMD64 but fails on ARM64 | Native dependencies or binary architecture |
| DNS failure | Runtime DNS configuration and service discovery |
| External connection fails only at startup in Istio | Follow [Istio startup checks](IstioStartup.md) |

## Migration checklist

- [ ] Previous release and rollback configuration saved.
- [ ] Published gold images selected by digest.
- [ ] Build tools and dependencies kept out of the runtime image.
- [ ] Startup commands tested without a shell.
- [ ] Permissions, UID, writable paths, and CA trust tested.
- [ ] IAM, network access, DNS, secrets, and CPU architecture checked.
- [ ] Security Contract requirements verified.
- [ ] Startup integrations tested, including Istio **only if used**.
- [ ] Final image scanned, staged rollout monitored, and evidence recorded.

## References

- [Onboarding guide](OnboardingGuide.md)
- [Istio startup checks — only for Istio workloads](IstioStartup.md)
- [Security Contract](../SecurityContract.md)
- [Runtime Hardening Guide](../BaseRuntimeHardeningGuide.md)
- [Repository overview](../../README.md)
- Relevant language README and published image catalog
