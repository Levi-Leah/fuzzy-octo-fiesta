# Optional: Register a gold-image deployment

Use this step **after a successful deployment** if your team uses the shared gold-image deployment inventory. It is separate from building the application or publishing a gold base image.

The example below was supplied by an application team. Confirm the reusable workflow's current inputs, access permissions, and supported version with the [deployment tracking workflow](https://github.com/wcmisec/gold-base-images-tracking/blob/main/.github/workflows/register-deployment.yml) before copying it.

## GitHub Actions example

Add a registration job in the service repository's workflow:

```yaml
jobs:
  deploy:
    # Existing deployment job goes here.
    # It must expose app_digest and gold_digest as job outputs.

  register-deployment:
    needs: deploy
    if: ${{ needs.deploy.result == 'success' }}
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

Replace the sample team, environment, cluster, namespace, workload, container, and language with your own values.

- `application_digest`: digest of the **final deployed service image**, not its base.
- `gold_digest`: digest of the **gold base image** used by that service build.
- `needs: deploy`: ensures the registration job waits for a successful deploy; verify that `deploy` actually exposes both outputs.
- `@v1`: the example's reusable-workflow version. Confirm that it is maintained before adopting it.
- `secrets: inherit` and `contents: write`: security-sensitive permissions. Only grant the permissions the called workflow actually requires; `contents: write` applies to the caller's repository token and **does not automatically grant write access to another repository**. Check how inventory updates authenticate before enabling it.

> [!IMPORTANT]
> This is a **sample integration**, not a verified drop-in workflow. Do not add an empty `deploy` job: reuse your real deployment job and its actual output names. Verify that inventory registration succeeded and that the recorded digests match the images deployed.

> [!WARNING]
> Never put long-lived credentials in the workflow file. Review the called workflow's permissions and secret requirements with the owners before enabling `secrets: inherit`.

## Verify the registration

1. Confirm that the deployment job succeeded.
2. Check the registration job logs.
3. Check the inventory entry for the intended team, environment, workload, and container.
4. Compare the recorded application and gold-image digests with the deployed release.

If your team does not use this inventory workflow, keep the digest and release evidence in your existing deployment records. Registration does not replace deployment testing or the [Security Contract](../SecurityContract.md).
