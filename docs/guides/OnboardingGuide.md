# Onboarding: Build a new service with gold images

Use this guide to create a new service image. For an existing service, follow the [migration guide](MigrationGuide.md).

Gold **builder** images contain build tools; gold **runtime** images are minimal, shell-free production bases. Your service's Dockerfile combines the two.

## 1. Find your images

1. Find your supported language and version in [TODO: gold image catalog].
2. Copy the builder and runtime references with their published `sha256` digests.
3. Confirm support for your target architectures (`amd64`/`arm64`).

## 2. Create your Dockerfile

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

Set file permissions during build or with `COPY --chmod`. Replace the example paths and add your executable's `CMD`/`ENTRYPOINT`.

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

Distroless images use UID `65532`; .NET chiseled uses `1654`. See [TODO: language READMEs] for actual commands and image settings.

## 3. Configure application startup

Use direct exec-form `CMD`/`ENTRYPOINT`. Check Kubernetes `command`/`args` overrides too.

> [!WARNING]
> Shell commands such as `/bin/sh -c "prepare && start"` fail without a shell. Call the installed interpreter or binary directly. If startup has multiple steps, implement and test that sequence in the application or a separate deployment task.

For startup-time external calls, test readiness and retry behavior.

Supply the required CA certificates (Java uses its JDK trust store); test TLS without disabling verification.

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

Configure restricted writable `/tmp` storage where needed, and scope IAM, egress, and secrets as required by the contract. Kubernetes `emptyDir` alone does not enforce all mount flags.

## 5. Build and verify the service

From the directory containing the service Dockerfile:

```sh
docker build --pull -t gold-service:test .
docker run --rm --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,nodev,size=128m,mode=1777 \
  --cap-drop=ALL --security-opt=no-new-privileges \
  gold-service:test
```

Scan the final service image. Verify startup, external connections, deployment controls, and each target architecture in staging.

## 6. Register the deployment

Add this job to your service's GitHub Actions workflow after its deployment job succeeds:

```yaml
register-deployment:
  needs: deploy
  uses: wcmisec/gold-base-images-tracking/.github/workflows/register-deployment.yml@v1
  secrets: inherit
  permissions:
    contents: write
  with:
    team: payments
    environment: prod
    cluster: us-east-1-prod
    namespace: payments
    workload: checkout-api
    container: app
    application_digest: ${{ needs.deploy.outputs.app_digest }}
    gold_image: python
    gold_digest: ${{ needs.deploy.outputs.gold_digest }}
```

Replace the sample values and ensure `deploy` exports the two digests. Review the reusable [tracking workflow](https://github.com/wcmisec/gold-base-images-tracking/blob/main/.github/workflows/register-deployment.yml) before granting `secrets: inherit` or `contents: write`; those permissions depend on how your repositories authenticate. Verify the registration in the inventory. [TODO: inventory link]

**Istio users:** For sidecar readiness when the application calls external services at startup, see [Istio startup checks](IstioStartup.md).

## References

- [Gold images overview](../../README.md)
- [Security Contract](../SecurityContract.md)
- [Hardening Guide](../BaseRuntimeHardeningGuide.md)
- [TODO: image catalog, language READMEs, and registry access guide]
