# Runtime hardening guide

Use this guide when building and deploying a service from a gold runtime image. The [README](../README.md) explains how to find the images. The [Security Contract](SecurityContract.md) lists the requirements that the service team owns.

## 1. Build the service image

- Pin the builder and runtime bases to their published digests.
- Install and compile dependencies in a builder stage. Copy only required application files into the runtime stage.
- Do not add a shell, package manager, compiler, debugger, credentials, or build caches to the final image.
- Keep application files owned by root and not writable by the runtime user. Give executables the permissions needed to run.
- Keep a non-root runtime user. Launch the service directly with an exec-form `ENTRYPOINT` or `CMD`. Turn off hot reload and features that expect a shell.

> **Check:** Inspect and run the **final service image**. Starting from a hardened base does not guarantee that your Dockerfile kept its settings.

## 2. Check dependencies and CA trust

Scan the final image, including dependencies added by your build. Test it on `amd64` and `arm64` if the service will run on both. Native modules and compiled libraries need separate checks on each architecture.

Check the selected image and the trust store used by your runtime. Add the certificates your service needs, including approved internal CAs, in your service build. Keep them read-only. Java uses a JDK trust store, so verify that store and add internal CAs with the appropriate Java tooling. Test TLS against the endpoints your service uses. Do not disable certificate validation.

## 3. Make the root filesystem read-only

Set `readOnlyRootFilesystem: true` or the equivalent setting on your platform. If the application needs temporary files, provide a separate, size-limited `/tmp` mount. Review every other writable mount. Keep persistent application data outside `/tmp`.

Where supported, mount `/tmp` with `noexec`, `nosuid`, and `nodev`. These flags limit direct execution, SUID use, and device files on that mount. They do not stop an interpreter from reading a script.

For a local Docker check, replace `IMAGE_REFERENCE` with your final service image:

```sh
docker run --rm --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,nodev,size=128m,mode=1777 \
  --cap-drop=ALL --security-opt=no-new-privileges \
  IMAGE_REFERENCE
```

Docker normally applies its default seccomp profile. Verify that your host has not disabled it. Adjust the `/tmp` size to the needs of your service.

> **Kubernetes warning:** `emptyDir` with `medium: Memory` provides an in-memory volume, but it does not set `noexec`, `nosuid`, or `nodev` in a Pod spec. Use an approved platform method that enforces the flags and inspect the effective mount. If none is available, record the gap for security review.

**About `/home/nonroot`:** On distroless `:nonroot` images, this home directory may be owned by the non-root user. With `readOnlyRootFilesystem: true` (or Docker `--read-only`), an ordinary home directory in the image's root filesystem is **not writable**. A separate writable mount over it would change that. Check the actual mounts; do not add a home-directory overlay just because the directory is owned by the runtime user. An additional overlay may be a defense-in-depth choice, not a universal requirement.

## 4. Set process controls

On Kubernetes, apply these settings to the **service container**. The fragment below uses the distroless example identity (`65532`); the documented .NET chiseled runtime uses `1654`. Verify the published image's identity before deploying:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65532
  runAsGroup: 65532
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

This is a container `securityContext` fragment, not a full Pod manifest. Configure a writable `/tmp` volume separately if the service needs one, then handle its mount flags as described above. On ECS or another platform, use the supported equivalent of each control. Record controls the platform cannot provide.

Limit IAM to required actions and resources. Limit network access to required destinations. Supply secrets through your deployment platform, not the image.

## Avoid runtime development tools and shell hooks

Disable development hot reload and file-watching modes in production. Review build scripts, package lifecycle hooks, pre-start steps, and commands launched by the application: they must not assume a shell or builder-only tool exists at runtime.

For debugging, do not add a shell to the production image. Use platform logs and inspection tools or a separate approved debug environment. When checking file ownership or the effective user, **run the final production image**; a development image started with the same numeric UID is not an equivalent test.

## 5. Verify before release

1. Confirm the builder and runtime digest references. Check the final image for unexpected tools and writable application files.
2. Start the service as a non-root user with a read-only root filesystem. Test a normal request.
3. Confirm that temporary writes work only on intended mounts. Inspect the effective `/tmp` mount options.
4. Check the effective UID, capabilities, privilege escalation setting, and seccomp profile in the running workload.
5. Test TLS, IAM, and required network access. Scan the final image and test each architecture used in production.
6. Record the final image digest, test results, and approved exceptions.

Keep production images free of a shell. Use a separate debug environment or platform inspection tools when investigating a problem.
