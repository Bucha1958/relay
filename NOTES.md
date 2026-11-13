SHIP WITH CONDITIONS

total: sent=44997 status[201:42632 409:2354] errors[ECONNRESET:11] p50=33ms p95=328ms p99=852ms

## Would I ship Friday?

Not as things stand. I'd ship only if we get a few things owned before launch — mainly an app fix for idempotent retries, a deploy path that doesn't spray local manifests into prod, and confirmation that HPA can actually see CPU. The service runs, the load test looks acceptable after probe work, but I wouldn't bet launch day on what's still open.

## What I fixed first (and why)

`make up` did not give me a working service. I chased blockers in the order they stopped me.

The image wouldn't build: Dockerfile ran `npm ci` after copying only `package.json`. Lockfile was in the repo; it just never made it into the build context. Copied `package-lock.json` too.

Kind already had the image, but the pod sat in ImagePullBackOff because `relay:latest` defaults to Always. Set `imagePullPolicy: IfNotPresent`.

Then CreateContainerConfigError — Deployment asked for secret key `database-url`, Secret had `DATABASE_URL`. One character case, pod never starts.

After that, pods looked Ready but `curl localhost:8080/health` reset the connection. ConfigMap (and the handoff) say listen on 8080; the Service and probes still pointed at 3000. Lined those up.

Once it answered HTTP I tightened capacity. Node heap was set to 512MB while the container limit was 256Mi — that's an OOM waiting to happen — so I raised memory request/limit and added a small CPU request. Pool vs DB math was worse: 25 connections × 10 HPA max = 250 against a hard cap of 100. I set `PG_POOL_MAX` and `maxReplicas` both to 9 (81 max), which leaves room for migrations and anything else on the instance.

First full load test at 150 r/s for 5 minutes was ugly: hundreds of 500s, over a thousand ECONNRESETs, multi-second p99. Describe showed the liveness probe timing out on `/health` (that endpoint hits the DB) with a 1s timeout, then Kubernetes killing the container. Classic death spiral under load. I loosened liveness, added readiness, and re-ran the same test. Result is the `total:` line above — no 500s in that window, 11 resets, p99 under a second, and the relay pod finished with 0 restarts. That change mattered more than anything else for stability.

## What I left on purpose

I did not change application code. Under load and after restarts we still have a real bug: clients retry with the same Idempotency-Key (expected), the in-memory dedupe map is empty after a restart, Postgres unique index fires, and the handler turns that into a 500 instead of a 409. I saw it locally with `test-1` after a rollout — log line was the unique constraint on `events_idempotency_key`. I'd file that to a developer with a clear acceptance test: same key after restart → 409 with the existing id, not 500. That's my main "assign to eng" item for the follow-up.

HPA never moved off 1 replica during the test (`cpu: <unknown>/70%`). On kind that's probably missing metrics-server; in prod we'd still need to prove metrics are flowing or autoscaling is theatre. I also didn't rewrite CI or apply Terraform — both need a hard look before Friday but fixing them properly is a bigger change than this window.

`/health` still depends on the database. Probe tuning stopped it from killing us; longer term someone should split "process alive" from "can take traffic." I left that as a follow-up rather than inventing a new endpoint.

Local `postgres.yaml` is out of scope per the brief. I didn't touch it.

## CI and Terraform (read, didn't rewrite)

The GitHub Action uses long-lived AWS keys, pushes `:latest`, then `kubectl apply -f k8s/` — which would also apply the laptop Postgres and local Secret into the cluster. I would not run that pipeline as-is for a production cutover.

Terraform looks like a first pass that never landed: public ALB on HTTP only, bastion SSH open to the world, AMI hardcoded. Fine as a sketch; not something I'd apply the night before launch without a security pass.

## Conditions

- Eng owns the 409-on-duplicate behaviour before we lean on client retries at peak.
- Platform confirms metrics so HPA can scale, and that the connection budget still fits the real managed DB.
- Deploy path for Friday does not apply the local k8s bundle; no static AWS keys in CI if we can help it.
- Someone accepts TLS / bastion lockdown on the Terraform side before it's the prod footprint.
- Bare-minimum alerts: 5xx, restarts, DB connections getting tight.

## Questions

Is the launch target 150 r/s (handoff) or ~400 (README)? I'd like one number.

Who owns Postgres `max_connections` if we need a carve-out for non-app clients?

Is NodePort + ALB really the prod shape, or should this be ClusterIP behind something else?

Does archive-to-S3 have to be on for day one?

Is there already a metrics/alerting stack I should plug into, or are we inventing that on Friday too?
