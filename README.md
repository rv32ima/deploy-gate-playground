# deploy-gate-playground

A dummy repo for end-to-end testing of the bidder deploy gate (tvScientific/ds
`python/src/deploy_gate`, see `docs/plans/bidder-deploy-gate.md` there).

The gate keys on only two things:

- the **workflow path** of the run asking to deploy to the gated environment
- the **environment** that run targets

So this repo has workflows at the same paths as the real ones, and their jobs just sleep:

| Workflow | Environment | Gate behaviour |
|---|---|---|
| `ec2-bidding-agent-dev.yaml` | `bidder-devprod` | records a deployment (pass `fail=true` to record a failed one) |
| `ec2-load-balancer-dev.yaml` | `lb-devprod` | same, for LB |
| `ec2-bidding-agent-prod-fc.yaml` | `bidder-prod` | gated: needs this exact sha on `bidder-devprod` |
| `ec2-load-balancer-prod-fc.yaml` | `bidder-prod` | gated: needs this exact sha on `lb-devprod` |
| `unpaired-prod.yaml` | `bidder-prod` | not in the pairing table, always rejected |

## Setup

1. Push this repo to your personal account as a **public** repo. Custom deployment
   protection rules only work in private repos on GitHub Enterprise.
2. Register a GitHub App on your personal account:
   - Repository permissions: Deployments read & write, Actions read, Metadata read.
   - Subscribe to the **Deployment protection rule** event.
   - Set a webhook secret.
   - Set the webhook URL to `https://<wherever the gate runs>/webhook`.
   - Generate a private key.
3. Install the app on this repo only.
4. Run the gate somewhere GitHub can reach over HTTPS. Not from the work laptop. From a ds checkout:

   ```bash
   export DEPLOY_GATE_GITHUB_APP_ID=<app id>
   export DEPLOY_GATE_GITHUB_APP_PRIVATE_KEY="$(cat <app>.private-key.pem)"
   export DEPLOY_GATE_WEBHOOK_SECRET=<webhook secret>
   uv run python -m deploy_gate.main --port 8000
   ```

   Avoid relays that re-serialize the JSON body (e.g. smee). The HMAC is computed over the
   exact bytes GitHub sent, so a re-encoded body fails with 401.
5. Create the `bidder-prod` environment (Settings → Environments). Then, under **Deployment
   protection rules**, enable the app. The dev environments are created automatically on
   their first run.
6. Check the app's **Advanced → Recent Deliveries**. GitHub sends a `ping` on registration,
   and the gate should have answered it with 204.

## Test plan

Run these in order. Each prod run waits on the gate. Its approve/reject comment shows on the
run page and in the gate's logs.

```bash
gh workflow run ec2-bidding-agent-prod-fc.yaml   # 1. nothing deployed to dev yet -> rejected

gh workflow run ec2-bidding-agent-dev.yaml       # 2. dev deploy succeeds...
gh workflow run ec2-bidding-agent-prod-fc.yaml   #    ...then prod on the same sha -> approved

gh workflow run ec2-load-balancer-prod-fc.yaml   # 3. bidder dev doesn't count for LB -> rejected (lb-devprod)

gh workflow run unpaired-prod.yaml               # 4. unpaired workflow -> rejected
```

5. A failed dev deploy doesn't count. Move main to a fresh commit, then:

   ```bash
   jj new main -m "bump" && jj bookmark set main -r @ && jj git push --bookmark main
   gh workflow run ec2-bidding-agent-dev.yaml -f fail=true
   gh workflow run ec2-bidding-agent-prod-fc.yaml       # -> rejected
   ```

6. Exact sha, with promotion via a tag:

   ```bash
   gh workflow run ec2-bidding-agent-dev.yaml           # dev deploys main's current sha
   git tag promote-test main && git push origin promote-test
   jj new main -m "trunk moves on" && jj bookmark set main -r @ && jj git push --bookmark main
   gh workflow run ec2-bidding-agent-prod-fc.yaml       # main's new sha -> rejected
   gh workflow run ec2-bidding-agent-prod-fc.yaml --ref promote-test   # the dev sha -> approved
   ```

7. Bad signature: change `DEPLOY_GATE_WEBHOOK_SECRET`, restart the gate and redeliver any
   delivery from Recent Deliveries. It should get 401, and no review is posted.

Wait for each dev run to finish before dispatching prod. The gate only counts a deployment
whose newest status is `success`.
