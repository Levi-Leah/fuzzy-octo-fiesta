# Optional: Istio startup checks

**Use this guide only if your workload has an Istio sidecar** and makes outbound requests as it starts. You can skip it for workloads that do not use Istio.

## Why startup can fail

The application may try to connect to a database, cache, or API **before the Istio proxy is ready**. If the connection must pass through the proxy, the request can fail even when the destination is healthy.

This sometimes becomes visible after a move to gold runtime images. A previous shell wrapper or `npx` command may have delayed application startup. Removing that work changes timing and can expose a race that already existed. **Gold images do not necessarily start faster in every application.**

With `outboundTrafficPolicy: REGISTRY_ONLY`, also verify that the destination is permitted by the mesh. Waiting for the proxy does not fix a destination that the mesh blocks.

## What the error may look like

A Node application previously started with:

```yaml
command: ["/bin/sh", "-c"]
args: ["npx prisma migrate deploy && node index.js"]
```

After migration, a direct Node entrypoint runs a Prisma database migration at startup. It fails to reach PostgreSQL on port `5432`, although the database is healthy.

**Do not assume this error proves an Istio race.** DNS, credentials, network policies, service registration, and real database outages can cause similar failures.

## Check the cause

1. Confirm that the Pod has an `istio-proxy` sidecar.
2. Compare the application's first failed connection with the proxy's startup/readiness timeline.
3. Check that the destination is allowed by the current mesh configuration.
4. Check database/API health, DNS, credentials, and network policies.
5. Test whether waiting for proxy readiness removes the startup-only failure.

## Apply the workload-level mitigation

If the mesh owners approve it, add this annotation to the **Pod template** of the Deployment:

```yaml
spec:
  template:
    metadata:
      annotations:
        proxy.istio.io/config: '{ "holdApplicationUntilProxyStarts": true }'
```

This tells Istio to wait for the proxy before starting application containers. In the reported migration incident, it resolved the issue without changing the image.

It **does not** make the database or API available, fix denied egress, or replace application-level retries and timeouts.

### Mesh-wide configuration

Mesh owners can evaluate `meshConfig.defaultConfig.holdApplicationUntilProxyStarts: true` as a default. This is a platform decision: verify the supported Istio version and test any effect on workload startup before changing it globally.

## Verify the fix

1. Deploy the change in staging with sidecar injection enabled.
2. Restart the Pod to exercise a fresh startup.
3. Confirm that the app starts after proxy readiness and its first required outbound request succeeds.
4. Repeat in the production rollout process, watching startup failures and restarts.

## References

- [Migration guide](MigrationGuide.md)
- [Onboarding guide](OnboardingGuide.md)
- [Istio proxy configuration](https://istio.io/latest/docs/reference/config/istio.mesh.v1alpha1/#ProxyConfig)
