# Migration: Move an existing service to gold images

Use this guide to move an existing service onto gold images. For the standard two-stage Dockerfile pattern, see [Onboarding](OnboardingGuide.md#2-create-your-dockerfile).

## 1. Prepare a rollback

Before editing the service:

1. Save the current image and deployment configuration.
2. Check startup commands, writable paths, certificates, native dependencies, and external services.
3. Identify database migrations or other changes that an image rollback would not reverse.

> [!WARNING]
> Reverting the application image does not undo a database schema migration or another external change.

## 2. Replace the base images

1. Select published builder/runtime digests from [TODO: gold image catalog]; check target architectures.
2. Build dependencies and application artifacts with the gold builder.
3. Copy only runtime artifacts into the gold runtime, keeping files non-writable by its non-root user.

```sh
docker build --pull -t gold-service:migration-test .
```

> [!WARNING]
> Do not restore compatibility by adding a shell, package manager, build tools, or extra privileges to the production image.

## 3. Adapt startup behavior

Check Dockerfile `CMD`/`ENTRYPOINT`, Kubernetes `command`/`args`, and any startup hooks.

An old command like this requires tools that are absent from a distroless runtime:

```yaml
command: ["/bin/sh", "-c"]
args: ["npx prisma migrate deploy && node index.js"]
```

Invoke the interpreter or binary directly. For multi-step startup, use an application entrypoint or separate task (for example, a database migration Job).

**Check startup timing:** Removing a shell wrapper can change when the first database, cache, or API connection happens. Verify readiness, retries, and timeouts instead of assuming the external service will already be available.

See [TODO: language READMEs] for supported executable paths.

## 4. Check runtime compatibility

Check for incompatibilities introduced by the new runtime:

- **Filesystem:** Root-owned application files, writable temporary paths, and effective UID (`65532` for distroless, `1654` for .NET chiseled).
- **TLS:** Required CA certificates (Java uses its JDK trust store).
- **Architecture:** Compiled binaries and native dependencies on every target architecture.
- **Networking:** DNS, IAM, secrets, and egress permissions.

## 5. Apply deployment controls

Apply the [Security Contract](../SecurityContract.md) and [Hardening Guide](../BaseRuntimeHardeningGuide.md), including read-only root filesystem, process restrictions, and `/tmp` mount requirements.

## 6. Test and roll out

Test the **final image** locally with the hardened settings:

```sh
docker run --rm --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,nodev,size=128m,mode=1777 \
  --cap-drop=ALL --security-opt=no-new-privileges \
  gold-service:migration-test
```

Scan the final image; test startup, external dependencies, and deployment security settings in staging. Roll out gradually where supported.

For Kubernetes:

```sh
kubectl rollout status deployment/DEPLOYMENT -n NAMESPACE
kubectl logs deployment/DEPLOYMENT -n NAMESPACE --all-containers=true --tail=100
```

Check readiness, restarts, and connection errors before proceeding.

## 7. Register the deployment

After a successful rollout, add the [deployment registration job](OnboardingGuide.md#6-register-the-deployment) to the service's workflow. Register the application and gold-image digests for each deployed workload and verify the inventory entry.

## 8. Roll back if the rollout fails

Restore the previously saved image **and matching deployment configuration** through your release process. For a Kubernetes Deployment with usable rollout history:

```sh
kubectl rollout undo deployment/DEPLOYMENT -n NAMESPACE
kubectl rollout status deployment/DEPLOYMENT -n NAMESPACE
```

> [!WARNING]
> GitOps may reapply the declared state. Check database compatibility before rolling back.

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
