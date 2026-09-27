# Hardening a service built from a gold runtime image

Use this guide when preparing a service image and its deployment. The [project README](../README.md) gives the project status and explains how to select an image once published. The [Security Contract](SecurityContract.md) defines the required controls and ownership. Confirm language-specific details and the published digest record when they become available.

## 1. Build a small final image

- Pin the builder and runtime images to published digests. Use a builder stage for dependency installation and compilation; keep the final stage on the matching runtime image.
- Copy only the files required to run the service. Do not copy package managers, shell tools, compiler output caches, credentials, or the full build workspace.
- Make application code and dependencies root-owned and non-writable by the runtime UID. Check file modes for your actual executable: a binary needs execute permission, while ordinary data files do not.
- Run the process directly with an exec-form `ENTRYPOINT` or `CMD`. Turn off reload modes, shell hooks, and write-on-start caches. Precompute assets where practical.
- Retain the runtime image's non-root default, or use another approved non-root UID/GID. Test the user and file access in the **final service image**.

> **Note:** The project plans to provide base images, not a universal application Dockerfile. The right `COPY`, ownership, and startup command depend on the language and service. A final-stage `USER root` or a copied shell can undo base hardening.

## 2. Check architecture and dependencies

The base is described as publishing `linux/amd64` and `linux/arm64` variants. Build your service for each architecture you deploy, especially if it uses native modules or compiled libraries. Start each final image on the matching architecture and run service tests. Pin dependency versions and scan the final image; the base-image scan covers neither your added files nor every architecture.

## 3. Add CA trust when needed

Check what trust material is present in your selected language image. The supplied project documents disagree about a shared CA bundle, so do not assume one exists. If your service needs public or internal TLS trust, assemble the necessary certificates in a build stage and copy the resulting trust material into the final image at the location the runtime actually reads. Make it read-only. For Java, inspect the JDK trust store and import approved internal CAs using the supported Java tooling. Verify a TLS connection to each required endpoint in a test deployment.

Do not disable certificate validation to make an image work.

## 4. Provide writable `/tmp`

Run with a read-only root filesystem. Mount `/tmp` as a bounded writable temporary volume if the application needs temporary files. Aim to make it the only writable path, but explicitly review any additional mounts such as logs, sockets, or application data. Do not store data that must survive restarts in `/tmp`.

For a local Docker check, an example is:

```sh
docker run --rm --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,nodev,size=128m,mode=1777 \
  --cap-drop=ALL --security-opt=no-new-privileges \
  IMAGE_REFERENCE
```

Replace `IMAGE_REFERENCE` with the final service image digest. Docker normally applies its default seccomp profile; verify that your host has not disabled it, or select an approved stricter profile. A writable `/tmp` may need a different size or mode for your service.

> **Kubernetes warning:** An `emptyDir` with `medium: Memory` does not provide a Pod-level field for `noexec`, `nosuid`, and `nodev`. Use an approved platform volume or node configuration that actually sets these mount flags and verify the mounted result. If that is unavailable, record the deviation for security review; do not label a plain `emptyDir` as meeting the full mount requirement.

With `readOnlyRootFilesystem: true`, a native base-image directory such as `/home/nonroot` is read-only **unless a separate writable volume is mounted there**. Do not add a writable home mount without review. The earlier source material proposed mounting a read-only tmpfs over that path, but it is unnecessary for an ordinary base-layer path under a read-only root and can hide files. Check actual mounts instead.

## 5. Set deployment process controls

For Kubernetes, put these settings on the service container. Set the UID/GID to match the selected runtime image; `65532` is the UID/GID described in the project material, and should be checked against the published image.

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

This fragment belongs under `containers[].securityContext`; it is not a complete Pod manifest. If `/tmp` is needed, add a bounded volume and mount, then handle its mount flags as described in step 4. For ECS and other runtimes, use their equivalent fields for readonly root, non-root identity, capabilities, no-new-privileges where supported, seccomp where supported, and mount controls. Document controls that the platform cannot provide.

Restrict the task or workload IAM role to required actions and resources. Limit outbound traffic to required endpoints. Supply secrets through the platform's secret mechanism, not by baking them into the image.

## 6. Verify the final service

Before release, check:

1. The base reference uses the published digest; the final image has no unexpected tools, files, SUID binaries, or added writable application paths.
2. The service starts without root or a shell and serves a normal request; it also works with the root filesystem read-only.
3. Temporary writes succeed only on intended mounts. Inspect effective `/tmp` mount flags on the running platform, rather than relying on manifest text alone.
4. The running UID, dropped capabilities, privilege escalation setting, and seccomp profile match the contract. Inspect the effective deployment, not only the source manifest.
5. TLS works with the required public and internal endpoints; the service's IAM and egress rules cover only intended access.
6. The final image has been scanned and tested on every architecture used in production. Record the digest, result, and approved exceptions.

Production images need not gain a shell for debugging. Reproduce issues in a separate debug environment or use platform-supported inspection while keeping the shipped image unchanged.
