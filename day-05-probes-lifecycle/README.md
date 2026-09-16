# Day 5 — Probes & Application Lifecycle

## Checklist
- [ ] Liveness probe
- [ ] Readiness probe
- [ ] Startup probe
- [ ] Understand probe failure scenarios
- [ ] Configure probes for an ASP.NET Core app
- [ ] Graceful shutdown and SIGTERM
- [ ] terminationGracePeriodSeconds
- [ ] preStop hook
- [ ] Explain why readiness matters during deployments

---

## 1. The Three Probes

| Probe | Question it answers | Failure action |
|-------|---------------------|-----------------|
| **Startup** | "Has the app finished starting up yet?" | Blocks liveness/readiness checks until it succeeds; if it never succeeds within its budget, container is killed and restarted |
| **Liveness** | "Is the app still alive/healthy, or is it stuck/deadlocked?" | kubelet **kills and restarts** the container |
| **Readiness** | "Is the app ready to receive traffic right now?" | Pod is **removed from Service Endpoints** — no restart, just stops receiving traffic |

```yaml
containers:
- name: aspnet-app
  image: mcr.microsoft.com/dotnet/samples:aspnetapp
  ports:
  - containerPort: 8080
  startupProbe:
    httpGet:
      path: /healthz/startup
      port: 8080
    failureThreshold: 30      # 30 x periodSeconds = 60s to start up
    periodSeconds: 2
  livenessProbe:
    httpGet:
      path: /healthz/live
      port: 8080
    initialDelaySeconds: 0    # startupProbe already covers startup time
    periodSeconds: 10
    failureThreshold: 3       # 3 consecutive failures = restart
    timeoutSeconds: 2
  readinessProbe:
    httpGet:
      path: /healthz/ready
      port: 8080
    periodSeconds: 5
    failureThreshold: 3
    timeoutSeconds: 2
```

## 2. Why Three Separate Probes?

- Without **startupProbe**, a slow-starting app (JIT warm-up, EF migrations, cache priming) would fail liveness checks before it's even ready and get killed in an infinite restart loop.
- Without **readinessProbe**, traffic would be routed to a Pod that's still warming up or temporarily can't reach its database — causing user-facing errors instead of just skipping that Pod.
- Without **livenessProbe**, a deadlocked/hung process (still "running" but unresponsive) would never be restarted automatically.

## 3. Probe Failure Scenarios

| Scenario | What happens |
|----------|--------------|
| Liveness fails repeatedly (transient DB hiccup) | Pod restarts even though the real problem is external — **misconfigured liveness probes are a top cause of cascading outages** (restarting won't fix a downstream DB issue, and repeated restarts can worsen load) |
| Readiness fails during deploy | New Pod never receives traffic until truly ready → prevents 502s during rollout |
| Readiness fails permanently (bad config) | Deployment rollout **hangs** — `maxUnavailable`/`maxSurge` budget is never satisfied, `kubectl rollout status` blocks |
| Startup probe threshold too short | App killed repeatedly before finishing startup → looks like CrashLoopBackOff but is really just slow boot |

**Rule of thumb:** liveness probes should check "is this process fundamentally broken" (e.g., can serve a static 200), NOT "can I reach my database" — that belongs in readiness.

## 4. Graceful Shutdown

```
1. kubectl delete pod  (or rollout, or node drain)
2. Pod set to "Terminating"; simultaneously:
   a. Removed from Service Endpoints (stops new traffic)
   b. preStop hook executes (if defined)
3. SIGTERM sent to container's main process
4. App has up to `terminationGracePeriodSeconds` to shut down cleanly
   (finish in-flight requests, close DB connections, flush logs)
5. If still running when grace period expires → SIGKILL (forceful)
```

```yaml
spec:
  terminationGracePeriodSeconds: 30
  containers:
  - name: aspnet-app
    image: mcr.microsoft.com/dotnet/samples:aspnetapp
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "sleep 5"]
          # gives in-flight LB connections time to drain
          # BEFORE SIGTERM is even sent
```

**Why preStop + sleep matters:** Endpoint removal (step 2a) and SIGTERM (step 3) happen concurrently, but kube-proxy/LB propagation across the cluster is not instant. A short `sleep` in `preStop` delays SIGTERM just enough that in-flight connections routed in that split-second window aren't abruptly cut.

### ASP.NET Core specifics
- ASP.NET Core listens for `SIGTERM` automatically via `IHostApplicationLifetime` and begins graceful shutdown (stops accepting new requests, waits for in-flight ones, respects `ASPNETCORE_SHUTDOWNTIMEOUTSECONDS` / `HostOptions.ShutdownTimeout`).
- Make sure `HostOptions.ShutdownTimeout` (app-level) is **less than** `terminationGracePeriodSeconds` (Pod-level), or Kubernetes will SIGKILL before the app finishes its own graceful shutdown logic.

## 5. Why Readiness Matters During Deployments

During a rolling update:
1. New Pod (new ReplicaSet) is created.
2. It is **not added to the Service's Endpoints** until its readiness probe passes.
3. Only once ready does the Deployment controller consider it "available" and proceed to terminate an old Pod (respecting `maxUnavailable`/`maxSurge`).
4. If readiness is missing or always returns success immediately, traffic gets routed to a Pod that isn't actually ready to serve (still connecting to DB, loading cache) → user-visible errors during every deploy.

**Interview soundbite:** "Readiness probes are what make rolling updates actually zero-downtime — without them, Kubernetes has no way to know a new Pod is truly ready to serve traffic, and it will happily route requests to a Pod that returns 500s while it's still starting up."

## 6. Hands-On Lab

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probe-demo
spec:
  replicas: 2
  selector:
    matchLabels: {app: probe-demo}
  template:
    metadata:
      labels: {app: probe-demo}
    spec:
      terminationGracePeriodSeconds: 20
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        ports: [{containerPort: 80}]
        readinessProbe:
          httpGet: {path: /, port: 80}
          periodSeconds: 3
        livenessProbe:
          httpGet: {path: /, port: 80}
          periodSeconds: 10
        lifecycle:
          preStop:
            exec:
              command: ["sh", "-c", "sleep 5"]
EOF

kubectl get pods -l app=probe-demo -w &
kubectl delete pod -l app=probe-demo --wait=false
# Observe: Terminating state held for a few seconds due to preStop before container exits
```
