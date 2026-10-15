# Lab 42 – Verification evidence

## Image imported into k3s
```
docker.io/library/quote-app:local   ...   86.4 MiB   linux/amd64
```

## Service
```
NAME        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
quote-app   ClusterIP   10.43.181.30   <none>        80/TCP    4s
```

## Health checks
Port-forward (`kubectl port-forward svc/quote-app 8080:80`, then `curl http://127.0.0.1:8080/health`):
```
{"ok":true}
```
In-cluster through the Service (`kubectl exec service-check -- wget -qO- http://quote-app/health`):
```
{"ok":true}
```

## Three Ready replicas
```
NAME                         READY   STATUS    RESTARTS   AGE     IP
quote-app-69fbbcff49-hr8nn   1/1     Running   0          7m48s   10.42.0.9
quote-app-69fbbcff49-j4pnw   1/1     Running   0          11s     10.42.0.12
quote-app-69fbbcff49-v7fmh   1/1     Running   0          11s     10.42.0.11
```

## Pod replacement (reconciliation)
- Deleted: `quote-app-69fbbcff49-hr8nn`
- Replacement: `quote-app-69fbbcff49-n6clz` (new name, created by the ReplicaSet)
```
quote-app-69fbbcff49-hr8nn   1/1     Terminating         0   9m12s
quote-app-69fbbcff49-n6clz   0/1     Pending             0   0s
quote-app-69fbbcff49-n6clz   0/1     ContainerCreating   0   0s
quote-app-69fbbcff49-n6clz   0/1     Running             0   0s
quote-app-69fbbcff49-n6clz   1/1     Running             0   5s
```

## Deliberate readiness failure
Readiness probe path changed from `/health` to `/does-not-exist`.
Rollout:
```
error: timed out waiting for the condition
```
Pods – new Pod Running but not Ready, no restart; old Pods stay Ready:
```
quote-app-57b88c776b-n5flc   0/1     Running   0          103s
quote-app-69fbbcff49-j4pnw   1/1     Running   0          4m32s
quote-app-69fbbcff49-n6clz   1/1     Running   0          2m57s
quote-app-69fbbcff49-v7fmh   1/1     Running   0          4m32s
```
ReplicaSets – rollout stalled:
```
quote-app-57b88c776b   1   1   0
quote-app-69fbbcff49   3   3   3
```
Probe failure event:
```
Warning  Unhealthy  13s (x25 over 2m9s)  kubelet  Readiness probe failed: HTTP probe failed with statuscode: 404
```
EndpointSlice – the unready Pod is excluded from Service traffic:
```
quote-app-69fbbcff49-v7fmh      ready=true
quote-app-69fbbcff49-j4pnw      ready=true
quote-app-69fbbcff49-n6clz      ready=true
quote-app-57b88c776b-n5flc      ready=false
```
The Service still answered `{"ok":true}` during the failure, because only Ready Pods receive traffic.

## Recovery
```
deployment "quote-app" successfully rolled out

quote-app-69fbbcff49-j4pnw   1/1     Running   0          8m23s
quote-app-69fbbcff49-n6clz   1/1     Running   0          6m48s
quote-app-69fbbcff49-v7fmh   1/1     Running   0          8m23s

quote-app-57b88c776b   0   0   0
quote-app-69fbbcff49   3   3   3
```
Health after recovery: port-forward `{"ok":true}`, in-cluster `{"ok":true}`.

## Summary
- **Symptom:** after the change, the rollout timed out and the new Pod stayed `0/1 Running` with 0 restarts.
- **Cause:** the readiness probe requested `/does-not-exist`; the app returned HTTP 404, so the kubelet reported `Readiness probe failed ... statuscode: 404` and the Pod was never marked Ready.
- **Fix:** restored the probe path to `/health`, which returns 200 with `{"ok": true}`. The Pod template matched the previous one again, so the Deployment scaled the original ReplicaSet back to 3 Ready Pods. The probe was kept, not removed.

## What this baseline leaves untested
The app runs without PostgreSQL and `/health` does not query the database, so quote retrieval and insertion are not validated. All replicas run on one node, so losing the VM would stop the whole application.

## Extension – Lane B: liveness probe
Added a `livenessProbe` (HTTP GET `/health`, period 5s, failureThreshold 3) next to the readiness probe.

Healthy baseline, 0 restarts:
```
quote-app-5474798cbf-7p57q   1/1     Running   0          2m14s
quote-app-5474798cbf-d8n8f   1/1     Running   0          2m20s
quote-app-5474798cbf-wr6lz   1/1     Running   0          2m8s
```

Deliberate failure – only the liveness path changed to `/does-not-exist` (readiness kept `/health`):
```
quote-app-85b8c4c94b-28tgk   0/1     CrashLoopBackOff   9 (2m53s ago)   15m
quote-app-85b8c4c94b-f8pxs   0/1     CrashLoopBackOff   9 (2m9s ago)    15m
quote-app-85b8c4c94b-m9tv9   0/1     CrashLoopBackOff   9 (2m12s ago)   14m
```
```
Restart Count:  9
Liveness:       http-get http://:3000/does-not-exist delay=5s timeout=1s period=5s #success=1 #failure=3
Warning  Unhealthy  Liveness probe failed: HTTP probe failed with statuscode: 404
Normal   Killing    Container quote-app failed liveness probe, will be restarted
```
EndpointSlice during the failure – no Pod could receive traffic:
```
quote-app-85b8c4c94b-f8pxs      ready=false
quote-app-85b8c4c94b-28tgk      ready=false
quote-app-85b8c4c94b-m9tv9      ready=false
```

Recovery – liveness path restored to `/health`; after 45 s the new Pods still had 0 restarts:
```
deployment "quote-app" successfully rolled out

quote-app-5474798cbf-6fpqd   1/1     Running   0          113s
quote-app-5474798cbf-6qb4n   1/1     Running   0          107s
quote-app-5474798cbf-bzcjw   1/1     Running   0          119s
```

What this demonstrates:
- **Liveness** answers "is the container alive?". After 3 consecutive failures the kubelet kills and restarts the container; repeated restarts lead to `CrashLoopBackOff` with growing back-off delays.
- **Readiness** answers "can this Pod receive traffic?". In the core exercise the failing readiness probe only removed the Pod from the Service endpoints (`ready=false`); restart count stayed 0, because readiness never restarts a container.
- Here readiness still passed, so the rollout completed and replaced the old Pods before liveness started killing the new ones, which caused a full outage. A wrong liveness probe is more dangerous than a wrong readiness probe.
- The bad URL models a failed check, not a real application deadlock.
