# Onboarding: Build a new service with gold images

Use this guide to create a new service image. For an existing service, follow the [migration guide](MigrationGuide.md).

A **builder image** has tools to install and compile your application. A **runtime image** runs the finished application in production without those build tools. Your team creates the final service image and configures its deployment.

## 1. Find your images

You need an application repository, Docker access, registry access, and permission to edit the service deployment.

1. Find your supported language and version in [TODO: gold image catalog].
2. Copy the **full registry references and digests** of its builder and runtime images. A digest (`sha256:...`) identifies an image version.
3. Check that the images support your target CPU architecture (`amd64` or `arm64`).

```sh
# Replace this reference with your published runtime image.
docker manifest inspect REGISTRY/RUNTIME@sha256:RUNTIME_DIGEST
```

> [!IMPORTANT]
> Do not copy the placeholder references. Confirm that both images are **published** and follow [TODO: registry login instructions]. A version in a README is not proof that an image is available.

## 2. Create your Dockerfile

Make these changes in your **service repository**, not in the gold-image repository.

### Build stage

Use the published **builder** image to install dependencies and produce application artifacts. Use your project's dependency lockfile.

```dockerfile
FROM REGISTRY/BUILDER@sha256:BUILDER_DIGEST AS build
WORKDIR /app
# Copy source files and lockfiles.
# Run the build commands for your language.
```

### Runtime stage

Use the published **runtime** image and copy only the files needed to start the application.

```dockerfile
FROM REGISTRY/RUNTIME@sha256:RUNTIME_DIGEST
WORKDIR /app
# Replace these paths with your actual build output.
COPY --from=build --chown=0:0 /app/BUILD_OUTPUT /app/BUILD_OUTPUT
# Set an exec-form CMD or ENTRYPOINT for the actual executable.
```

Give executable files execute permission and keep application files non-writable by the runtime user. Set permissions in the builder or with `COPY --chmod` as appropriate. The examples show the two-stage structure; they are **not complete runnable Dockerfiles**.

> [!WARNING]
> Do not copy the full builder filesystem or add shells, package managers, compilers, build caches, or credentials to the runtime image. Do not rely on `RUN chmod` in a shell-free final stage.

### Language-specific differences

| Language | Build tools | Runtime consideration |
| --- | --- | --- |
| Python | `uv` | Check interpreter, installed modules, and CA bundle |
| Node.js | `pnpm` or other approved builder tooling | No `npx` or package manager at runtime |
| Java | Maven | Use the JDK trust store |
| Go | Go toolchain | Build a compatible static binary as required |
| Rust | Cargo and musl target | Build a compatible static binary |
| .NET 10 | .NET SDK | Chiseled runtime; check supported libraries |

The supplied image descriptions use UID/GID **65532** for distroless `:nonroot` images and **1654** for the .NET chiseled runtime. Check the published image's actual user before overriding it.

For actual build commands and runtime paths, see [TODO: language READMEs].

## 3. Configure application startup

The final image must start the application **directly**, using exec-form `CMD` or `ENTRYPOINT` (or the runtime image's suitable default). Check your Dockerfile and deployment `command`/`args`.

> [!WARNING]
> Shell commands such as `/bin/sh -c "prepare && start"` fail without a shell. Call the installed interpreter or binary directly. If startup has multiple steps, implement and test that sequence in the application or a separate deployment task.

If startup contacts a database, cache, or API, handle temporary connection failures with appropriate retries, timeouts, and readiness checks. Do not depend on an incidental startup delay.

Provide the required CA certificates for TLS. Java uses a JDK trust store; other runtimes may need a supplied CA bundle. Check the language-specific instructions.

> [!WARNING]
> Do not disable TLS certificate verification to make a connection work.

## 4. Apply deployment security settings

Follow the [Security Contract](../SecurityContract.md) and [Hardening Guide](../BaseRuntimeHardeningGuide.md). For Kubernetes, set this **container** security context:

```yaml
securityContext:
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

Provide an approved writable `/tmp` mount if needed, with required restrictions. A Kubernetes memory-backed `emptyDir` does not by itself guarantee `noexec`, `nosuid`, and `nodev`. Limit IAM and network permissions, and inject secrets through the deployment system.

## 5. Build and verify the service

From the directory containing the service Dockerfile:

```sh
docker build --pull -t gold-service:test .
docker image inspect gold-service:test --format '{{.Config.User}}'
docker image inspect gold-service:test --format '{{json .Config.Entrypoint}} {{json .Config.Cmd}}'
docker run --rm --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,nodev,size=128m,mode=1777 \
  --cap-drop=ALL --security-opt=no-new-privileges \
  gold-service:test
```

Configure necessary ports, secrets, and external dependencies through your approved test setup. These commands are a **local smoke test**, not proof that the deployment meets every requirement.

Before release, scan the **final application image**, test normal requests and shutdown, confirm required external connections, verify the deployed security controls, and test each CPU architecture you use. Keep the final image digest and test evidence with the release.

## Optional next steps

- **Istio sidecar:** If the service uses Istio and connects externally during startup, follow [Istio startup checks](IstioStartup.md).
- **Deployment inventory:** After a successful deployment, teams using the shared registry can [register their deployment](DeploymentTracking.md).

## References

- [Gold images overview](../../README.md)
- [Security Contract](../SecurityContract.md)
- [Hardening Guide](../BaseRuntimeHardeningGuide.md)
- [TODO: image catalog, language READMEs, and registry access guide]
