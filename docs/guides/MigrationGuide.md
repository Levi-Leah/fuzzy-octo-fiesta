# Migration: Move an existing service to gold images

Use this guide when an application already runs on another container image. The [onboarding guide](OnboardingGuide.md) explains gold builder/runtime images and the standard Dockerfile pattern.

## 1. Prepare a rollback

Before editing the service:

1. Save the current image digest, Dockerfile, and deployment configuration.
2. Record the existing startup command, writable paths, native dependencies, CA certificates, and external services.
3. Identify startup jobs such as database migrations and determine whether the previous release can run after they complete.
4. Confirm your team's rollback method.

> [!WARNING]
> Reverting the application image does not undo a database schema migration or another external change.

## 2. Replace the base images

1. Select published builder and runtime digests from [TODO: gold image catalog].
2. Confirm the target `amd64`/`arm64` architecture and application dependency compatibility.
3. Replace the Dockerfile's build stage with the gold builder. Install dependencies and compile the application there.
4. Replace the final stage with the gold runtime. Copy only the required output, keep it root-owned/non-writable, and preserve the runtime's non-root user.

```sh
# Replace the placeholders with the published reference.
docker manifest inspect REGISTRY/RUNTIME@sha256:RUNTIME_DIGEST

# Run from the service repository after editing its Dockerfile.
docker build --pull -t gold-service:migration-test .
docker image inspect gold-service:migration-test --format '{{.Config.User}}'
docker image inspect gold-service:migration-test --format '{{json .Config.Entrypoint}} {{json .Config.Cmd}}'
```

For the Dockerfile layout, see [Onboarding: Create your Dockerfile](OnboardingGuide.md#2-create-your-dockerfile). For language-specific instructions, see [TODO: language READMEs].

> [!WARNING]
> Do not restore compatibility by adding a shell, package manager, build tools, or extra privileges to the production image.

## 3. Adapt startup behavior

Check Dockerfile `CMD`/`ENTRYPOINT`, Kubernetes `command`/`args`, and any startup hooks.

An old command like this requires tools that are absent from a distroless runtime:

```yaml
command: ["/bin/sh", "-c"]
args: ["npx prisma migrate deploy && node index.js"]
```

Start the installed executable or interpreter directly. If the service needs multiple startup actions, use an application entrypoint that handles their failures or an appropriate separate deployment task. For Python, `python3 -m uvicorn ...` may be suitable when that interpreter and module exist; a virtualenv console script is not automatically incompatible.

**Check startup timing:** Removing a shell wrapper can change when the first database, cache, or API connection happens. Verify readiness, retries, and timeouts instead of assuming the external service will already be available.

> [!IMPORTANT]
> Use the executable paths supplied by the **selected image**; do not assume every language uses the same paths. [TODO: runtime command reference]

## 4. Check runtime compatibility

Review the following only for changes introduced by the new runtime:

- **File writes and identity:** Check the effective runtime UID, application permissions, caches, and temporary directories. The documented distroless UID/GID is `65532`; .NET chiseled uses `1654`.
- **TLS:** Test real connections with the required CA certificates. Java uses its JDK trust store. Never disable certificate verification to resolve a failure.
- **Native libraries:** Check compiled modules and binaries on each architecture used by the service.
- **Networking:** Confirm DNS, IAM permissions, secrets, egress policy, and access to external dependencies. Normal DNS configuration belongs to the container platform, not a baked-in `/etc/resolv.conf`.

## 5. Apply deployment controls

Use the [Security Contract](../SecurityContract.md) and [Hardening Guide](../BaseRuntimeHardeningGuide.md) to set the required non-root identity, read-only filesystem, dropped capabilities, no privilege escalation, and seccomp profile.

Confirm that any `/tmp` mount has the required restrictions; an ordinary Kubernetes memory-backed `emptyDir` does not establish all mount flags.

## 6. Test and roll out

Test the **final image** locally with the hardened settings:

```sh
docker run --rm --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,nodev,size=128m,mode=1777 \
  --cap-drop=ALL --security-opt=no-new-privileges \
  gold-service:migration-test
```

In staging, test startup and restarts, application requests, external dependencies, and the effective security settings. Scan the final image. Then release gradually if your deployment system supports it.

For Kubernetes, replace the example names with your Deployment and namespace:

```sh
kubectl rollout status deployment/DEPLOYMENT -n NAMESPACE
kubectl get pods -n NAMESPACE
kubectl logs deployment/DEPLOYMENT -n NAMESPACE --all-containers=true --tail=100
```

Watch health checks, restarts, connection errors, and application logs before completing the rollout.

After a successful deployment, you may [register the deployment](DeploymentTracking.md) if your team uses the shared inventory.

## 7. Roll back if the rollout fails

Restore the previously saved image **and matching deployment configuration** through your release process. For a Kubernetes Deployment with usable rollout history:

```sh
kubectl rollout undo deployment/DEPLOYMENT -n NAMESPACE
kubectl rollout status deployment/DEPLOYMENT -n NAMESPACE
```

> [!WARNING]
> A GitOps controller may restore the currently declared state. Confirm database/schema compatibility and follow the team's rollback procedure before using this command.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| `/bin/sh` or `npx` not found | Old shell/package-manager startup command |
| Permission denied / read-only filesystem | Effective UID, file modes, intended writable mounts |
| Unknown CA / TLS failure | Runtime-specific trust store and supplied certificates |
| Works on AMD64, fails on ARM64 | Binary architecture and native dependencies |
| DNS or connection failure | DNS configuration, network rules, destination health |
| Connection fails only at startup | Dependency readiness and retry behavior; [Istio-specific checks](IstioStartup.md) **only if the service uses Istio** |

## References

- [Onboarding guide](OnboardingGuide.md)
- [Security Contract](../SecurityContract.md)
- [Hardening Guide](../BaseRuntimeHardeningGuide.md)
- [TODO: gold image catalog and language READMEs]
