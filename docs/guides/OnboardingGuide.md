# Onboarding guide: Use gold base images

Use this guide when creating a **new application** with the shared gold container images. If you already have a running service and need to replace its image, use the [migration guide](MigrationGuide.md).

## What are gold images?

A **builder image** contains tools to install dependencies and build your application. A **runtime image** contains only what is needed to run it in production.

Your team creates the final application image. Gold images do not contain your code, create your deployment, or automatically make the deployed service secure.

The runtime images are minimal: most use *distroless* bases, while .NET uses a *chiseled* base. These images do not provide a shell such as `/bin/sh` or normal package managers. You must prepare the application before the runtime stage.

## 1. Check the prerequisites

Before you start, make sure you have:

- An application repository and a Dockerfile you can edit.
- Access to the organization's published gold image catalog and container registry.
- A supported language/version with **published builder and runtime images**.
- A way to build, scan, and test your final application image.
- Access to the deployment configuration (for example, Kubernetes or ECS).

> [!IMPORTANT]
> A version listed in a README may not have a published image. Confirm the image in the current catalog before proceeding.

## 2. Choose and pin the images

1. Find your language and version in the published catalog.
2. Copy the **full image reference and digest** for both the builder and runtime. A *digest* (`sha256:...`) identifies one specific image.
3. Check that the image supports the CPU architecture used by your cluster (for example, `amd64` or `arm64`).
4. Save the chosen digests with your release information.

You can inspect a published image with Docker (replace the example reference):

```sh
docker manifest inspect REGISTRY/IMAGE@sha256:DIGEST
```

> [!IMPORTANT]
> `REGISTRY/IMAGE@sha256:DIGEST` is a placeholder, not a working image. Follow your registry's authentication instructions: [TODO: link to registry access guide].

## 3. Create and build the service image

Complete these steps in your **application repository**, not in the gold-image repository.

### 3.1 Build your application in a builder stage

Start the Dockerfile with the **published builder image**. Install dependencies, compile or package the service, and keep its dependency lockfile in use.

```dockerfile
# Illustration only — use your published image reference.
FROM REGISTRY/BUILDER@sha256:BUILDER_DIGEST AS build
WORKDIR /app
# Copy source and lockfiles, then use your language's documented build commands.
```

Use the [language build instructions](#language-notes) for the actual commands.

### 3.2 Copy the result into the runtime stage

Add the production runtime stage. Copy only the output needed to run the service. The location below is illustrative: adjust it to your application's real build output.

```dockerfile
FROM REGISTRY/RUNTIME@sha256:RUNTIME_DIGEST
WORKDIR /app
COPY --from=build --chown=0:0 --chmod=0555 /app/OUTPUT_DIRECTORY /app/OUTPUT_DIRECTORY
# Set an exec-form CMD or ENTRYPOINT for your actual executable.
```

Use file and directory permissions appropriate to the artifact: executable binaries need execute permission; non-executable files normally only need read permission. Preserve the image's default non-root user, unless the published image guidance requires an override.

> [!WARNING]
> Do **not** copy the entire builder stage into production or add a shell, package manager, compiler, credentials, or build caches. Do not assume `RUN chmod` works in a shell-free final stage: set permissions in the builder or use `COPY --chown` / `COPY --chmod`.

### 3.3 Build and inspect the final image

Run from the directory containing your service Dockerfile:

```sh
docker build --pull -t gold-service:test .
docker image inspect gold-service:test --format '{{.Config.User}}'
docker image inspect gold-service:test --format '{{json .Config.Entrypoint}} {{json .Config.Cmd}}'
```

Confirm the runtime user and startup command match the selected gold image and your application. These commands inspect metadata; they do **not** verify every file permission or deployment setting. Follow the [Hardening Guide](../BaseRuntimeHardeningGuide.md) for full checks.

### Language notes

| Language | Build stage | Check in the runtime stage |
| --- | --- | --- |
| Python | Install/build with `uv` | Direct interpreter startup, dependency paths, CA certificates |
| Node.js | Install/build with the supported package tooling | Do not expect `npm`, `npx`, `pnpm`, Yarn, or `/bin/sh` |
| Java | Build the artifact with Maven | Java entrypoint and JDK trust store |
| Go | Compile the target binary; use `CGO_ENABLED=0` for the documented static approach | Binary architecture, static linking, CA certificates |
| Rust | Build for the appropriate musl target | Binary architecture, static linking, CA certificates |
| .NET | Publish with the .NET SDK | Chiseled runtime compatibility and non-root UID |

The latest supplied source README includes **.NET 10** with a chiseled runtime. The language notes identify UID/GID **65532** for the distroless `:nonroot` families and **1654** for .NET. **Check the published image**, rather than applying either value to every service. Available versions and published images must also be checked in the catalog or Artifactory.

For language-specific build commands, use the relevant language README. This guide does not assume every application has the same project layout or entrypoint.

## 4. Configure startup and certificates

> [!WARNING]
> The runtime has no shell. Do not use shell-based startup commands such as:

```yaml
# This requires /bin/sh and will not work in a shell-free runtime.
command: ["/bin/sh", "-c"]
args: ["prepare && start"]
```

Use a direct executable instead, or keep the image's default startup command if it is correct. For example, Python can run a *module* with `python3 -m ...` when the interpreter and module exist in the image. A virtualenv console script can also work if its shebang and interpreter are valid; **test it** rather than assuming.

**Startup dependencies:** If the application calls a database, cache, or external API before becoming ready, make sure it handles a temporary connection failure. Use retries and readiness checks rather than depending on an incidental startup delay. This applies with or without a service mesh.

Before deployment, confirm that TLS connections work:

- **Python, Node.js, Go, Rust:** Provide the CA certificates needed by your application, using the path and settings expected by your runtime.
- **Java:** Check the built-in JDK trust store; import internal CAs there if required.
- **.NET:** Verify the selected chiseled image's trust-store behavior.

> [!WARNING]
> Do not disable certificate verification to work around a TLS error.

## 5. Secure the deployment

The base image alone does **not** configure the deployed container. Apply the requirements in the [Security Contract](../SecurityContract.md), using the [Hardening Guide](../BaseRuntimeHardeningGuide.md) for examples.

For Kubernetes, the **container** security context should include:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

This is a fragment, not a full Pod manifest. Set `runAsUser` and `runAsGroup` only after confirming the selected runtime's UID/GID.

Also:

- Provide a writable temporary location only if the application needs one. Keep the root filesystem read-only.
- Follow the Security Contract for `/tmp` mount restrictions. A Kubernetes memory-backed `emptyDir` alone does not guarantee `noexec`, `nosuid`, or `nodev`; verify effective mount options.
- Restrict network access and IAM permissions to what the application needs.
- Provide secrets at deployment, not in the image.

## 6. Test before release

Run the final application image with a read-only root filesystem. The example below provides a restricted temporary directory and assumes the image's configured entrypoint is runnable:

```sh
docker run --rm --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,nodev,size=128m,mode=1777 \
  --cap-drop=ALL --security-opt=no-new-privileges \
  gold-service:test
```

If your service needs ports, environment variables, secrets, or external dependencies, configure them through your approved local test setup. Never put real secrets directly in the shell command.



1. Build and scan the **final application image**, not only the gold base.
2. Run it with its real startup command, non-root user, and read-only root filesystem.
3. Check application startup, a normal request, and shutdown.
4. Check writes to temporary storage and confirm that required certificates, secrets, DNS, and external connections work.
5. Verify the deployed security settings and any required mount options.
6. Test the CPU architectures actually used in production, including native dependencies.
7. Record the final image digest, test results, and approved exceptions.

If the application makes network requests **during startup** in an Istio-injected namespace, also read [Istio startup checks](IstioStartup.md). Other workloads can skip that guide.

After a successful deployment, teams using the shared inventory can follow [Register a deployment](DeploymentTracking.md). This is optional and does not replace validation.

## Ready-to-release checklist

- [ ] Published builder and runtime digests selected and recorded.
- [ ] Application builds in a separate stage; runtime contains only needed files.
- [ ] Startup does not depend on missing shell or build tools.
- [ ] Non-root identity, file permissions, and certificate trust tested.
- [ ] Deployment follows the Security Contract.
- [ ] Final image scanned and tested in the target environment.

## References

- [Repository overview](../../README.md)
- [Image publishing and supply-chain boundaries](../ImagePublishing.md)
- [Migration guide](MigrationGuide.md)
- [Security Contract](../SecurityContract.md)
- [Runtime Hardening Guide](../BaseRuntimeHardeningGuide.md)
- [Optional deployment tracking](DeploymentTracking.md)
- [TODO: language-specific README links and published image catalog]
