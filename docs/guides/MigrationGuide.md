# Migration guide: Move an existing service to gold images

Use this guide when your service **already runs** and you want to replace its current builder or runtime images with the shared gold images.

A gold **builder** supplies build tools. A gold **runtime** is a small, hardened image used to run the finished application. The runtime usually has no shell or package manager. Your service must not depend on those tools when it starts.

For a new service, use the [onboarding guide](OnboardingGuide.md).

## 1. Record the current working configuration

Before changing the image:

1. Record the current application image digest and deployment configuration.
2. Save the existing Dockerfile and startup command.
3. List all runtime dependencies, including native libraries, certificates, writable directories, and external services.
4. Note any commands run at startup, such as database migrations or cache warm-up. Record which external services must be reachable before the application is ready.
5. Confirm how to restore the existing release.

> [!IMPORTANT]
> **General startup lesson:** Replacing a shell wrapper or package-manager command can change how quickly the application reaches its first network request. A successful deployment may therefore expose an existing dependency race. Check readiness and retries even when you do **not** use Istio.

> [!WARNING]
> Do not skip rollback preparation. Rolling back the image may not undo a database schema change.

## 2. Choose the replacement images

1. Open the published gold image catalog.
2. Confirm that your language/version has published builder and runtime images.
3. Copy both full image references with their `sha256` digests.
4. Check the cluster CPU architecture and any native dependencies.

Use the **published image catalog or Artifactory**, not a README version list alone, to confirm availability. Digest pinning identifies the base image used in a given build. [TODO: link to image catalog and registry access instructions]

Inspect the target image before changing the Dockerfile:

```sh
docker manifest inspect REGISTRY/RUNTIME@sha256:RUNTIME_DIGEST
```

Replace the reference with a real published image digest.

## 3. Update the Dockerfile

1. Replace the old build stage with the appropriate gold **builder** image.
2. Install dependencies and build the application in that stage.
3. Use the matching gold **runtime** image for the final stage.
4. Copy only the required application files and runtime dependencies.
5. Set permissions before the shell-free runtime stage, or use `COPY --chown` / `COPY --chmod`.
6. Keep the runtime's default non-root identity unless the approved image requirements say otherwise.

> [!WARNING]
> Do not copy the entire builder filesystem into production or add a shell or package manager back to the final image.

Build and inspect the modified service image from its application repository:

```sh
docker build --pull -t gold-service:migration-test .
docker image inspect gold-service:migration-test --format '{{.Config.User}}'
docker image inspect gold-service:migration-test --format '{{json .Config.Entrypoint}} {{json .Config.Cmd}}'
```

These commands check the built image and its configured startup metadata, not full runtime security. For the two-stage Dockerfile pattern, see [Onboarding: Create and build the service image](OnboardingGuide.md#3-create-and-build-the-service-image).

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

> [!IMPORTANT]
> This command requires both `/bin/sh` and `npx`. Move dependency installation to the builder, and run the installed application through its interpreter or binary.

If startup requires several steps, use a deliberately written application entrypoint or a suitable separate deployment task. Test errors and exit codes. Do not silently remove database migrations or other required work.

For Python, a direct command such as `python3 -m uvicorn ...` may be appropriate **if** the interpreter and module exist. A virtualenv console script is not automatically broken; verify its shebang, permissions, and dependencies.

> [!IMPORTANT]
> Check the actual image: command paths differ across languages and images. [TODO: links to language-specific runtime READMEs]

## 5. Check changes that can break the application

### Files and runtime identity

- The root filesystem should be read-only at deployment.
- Application files must be root-owned and not writable by the runtime user.
- Move legitimate temporary writes to approved writable mounts; disable unnecessary caches.
- Verify the selected runtime UID/GID. The supplied `:nonroot` images use **65532**, while the .NET chiseled image description uses **1654**.

### Startup dependencies and external access

- Check whether the application contacts databases, caches, or external APIs during startup.
- Confirm when those destinations become available. Do not use a delay caused by a shell wrapper or package manager as a readiness mechanism.
- Use appropriate dependency retries, timeouts, and readiness checks. A reachable database is not necessarily ready at the instant the app starts.
- Consider moving one-time database migrations into a separate controlled deployment step, where appropriate. Rollback and schema compatibility still need planning.

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

For a local test with a read-only root filesystem, run:

```sh
docker run --rm --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,nodev,size=128m,mode=1777 \
  --cap-drop=ALL --security-opt=no-new-privileges \
  gold-service:migration-test
```

Add approved runtime configuration and dependencies as needed. For a Kubernetes deployment, replace `NAMESPACE` and `DEPLOYMENT` below with your real values:

```sh
kubectl rollout status deployment/DEPLOYMENT -n NAMESPACE
kubectl get pods -n NAMESPACE
kubectl logs deployment/DEPLOYMENT -n NAMESPACE --all-containers=true --tail=100
```

These are basic health and log checks; they do not replace application tests or security validation.



1. **Build:** Confirm that all required runtime files exist and the final image is scanned.
2. **Start locally:** Run the final image as non-root with a read-only root filesystem. Confirm the real entrypoint works without a shell.
3. **Test in staging:** Verify startup, health checks, requests, shutdown, restarts, certificates, DNS, secrets, and external connections.
4. **Test the deployment controls:** Inspect the effective security context and writable mounts.
5. **Check startup tasks:** Confirm migrations, cache initialization, and other startup operations work correctly.
6. **Release gradually:** Use a canary or small rollout if supported. Watch restarts, readiness, connection errors, and application logs.
7. **Record the outcome:** Save the image digest, test evidence, configuration, and exceptions.

**Optional — Istio:** If the workload uses an Istio sidecar and makes outbound calls at startup, follow [Istio startup checks](IstioStartup.md). This is not a required step for services without Istio.

After the successful rollout, teams using the shared inventory can follow [Register a deployment](DeploymentTracking.md). It is an optional post-deployment step.

## 8. Roll back if needed

1. Restore the last known good **application image and matching deployment configuration**.
2. Check application health and the external connections it needs.
3. Review database schema compatibility separately; an image rollback may not reverse migrations.
4. Record the failed check and fix it before retrying.

For a Kubernetes Deployment using rollout history, the following may restore its previous Pod template:

```sh
kubectl rollout undo deployment/DEPLOYMENT -n NAMESPACE
kubectl rollout status deployment/DEPLOYMENT -n NAMESPACE
```

> [!WARNING]
> `rollout undo` does **not** reverse external changes such as database migrations. Confirm that the previous image and configuration remain compatible. Follow your team's release/rollback procedure; GitOps controllers may also reapply the desired state.

> [!WARNING]
> Do not work around failures by running as root, adding a shell, or disabling required security controls.

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
| External connection fails during startup | Check dependency readiness, network policies, and retry/timeout handling; see [Istio startup checks](IstioStartup.md) **only if using Istio** |

## Migration checklist

- [ ] Previous release and rollback configuration saved.
- [ ] Published gold images selected by digest.
- [ ] Build tools and dependencies kept out of the runtime image.
- [ ] Startup commands tested without a shell.
- [ ] Permissions, UID, writable paths, and CA trust tested.
- [ ] IAM, network access, DNS, secrets, and CPU architecture checked.
- [ ] Security Contract requirements verified.
- [ ] Startup dependencies and readiness/retry behavior tested; Istio checks completed **only if used**.
- [ ] Final image scanned, staged rollout monitored, and evidence recorded.

## References

- [Onboarding guide](OnboardingGuide.md)
- [Istio startup checks — only for Istio workloads](IstioStartup.md)
- [Security Contract](../SecurityContract.md)
- [Runtime Hardening Guide](../BaseRuntimeHardeningGuide.md)
- [Repository overview](../../README.md)
- [Image publishing and supply-chain boundaries](../ImagePublishing.md)
- [Optional deployment tracking](DeploymentTracking.md)
- [TODO: language-specific README links and published image catalog]
