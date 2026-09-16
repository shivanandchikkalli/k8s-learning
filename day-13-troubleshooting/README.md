# Day 13 — Production Troubleshooting

## Checklist
- [ ] kubectl get pods
- [ ] kubectl describe
- [ ] kubectl logs / --previous
- [ ] kubectl exec
- [ ] kubectl get events
- [ ] kubectl top
- [ ] Pending
- [ ] CrashLoopBackOff
- [ ] ImagePullBackOff / ErrImagePull
- [ ] OOMKilled
- [ ] Diagnose Service/Ingress connectivity problems
- [ ] Intentionally break an app and troubleshoot it

---

## 1. Core Diagnostic Commands

```bash
kubectl get pods -A -o wide                          # quick status + node + IP
kubectl describe pod <pod> -n <ns>                    # events, conditions, resource info
kubectl logs <pod> -n <ns> -c <container>              # current container logs
kubectl logs <pod> -n <ns> -c <container> --previous   # logs from the CRASHED instance
kubectl exec -it <pod> -n <ns> -- sh                   # shell into a running container
kubectl get events -n <ns> --sort-by='.lastTimestamp'  # cluster/pod-level events, chronological
kubectl top pods -n <ns>; kubectl top nodes            # live CPU/memory (needs metrics-server)
```

**Debugging order of operations:** `get` (what's the status?) → `describe` (why — events/conditions) → `logs` (what did the app say?) → `exec` (interactive investigation) → `events` (cluster-wide correlation) → `top` (resource pressure).

## 2. Pending Pods

**Meaning:** Pod accepted by API Server but **not yet scheduled** to any node.

```bash
kubectl describe pod <pod>   # check the Events section at the bottom
```

Common causes:
| Cause | Event message hint |
|-------|---------------------|
| Insufficient CPU/memory on all nodes | `Insufficient cpu` / `Insufficient memory` |
| No node matches nodeSelector/affinity | `didn't match Pod's node affinity/selector` |
| Taints without matching tolerations | `node(s) had taint {...}, that the pod didn't tolerate` |
| PVC not bound (waiting on storage) | `pod has unbound immediate PersistentVolumeClaims` |
| Not enough nodes (cluster autoscaler still scaling up) | Pending resolves itself after a minute or two |

## 3. CrashLoopBackOff

**Meaning:** container starts, exits (crashes) repeatedly; kubelet is backing off restart attempts exponentially (10s, 20s, 40s... up to 5 min).

```bash
kubectl logs <pod> --previous       # THE most important command — see why it crashed
kubectl describe pod <pod>          # check "Last State: Terminated, Reason, Exit Code"
```

| Exit Code | Meaning |
|-----------|---------|
| 0 | Clean exit (shouldn't loop unless restartPolicy forces it and something re-triggers exit) |
| 1 | General application error — check logs |
| 137 | `128 + 9 (SIGKILL)` — usually **OOMKilled** or forcefully killed after grace period |
| 143 | `128 + 15 (SIGTERM)` — graceful termination signal received |

Common causes: app crashes on startup (bad config/missing env var), failing liveness probe restarting it faster than it can start, missing dependency (DB not reachable and app doesn't retry gracefully).

## 4. ImagePullBackOff / ErrImagePull

**Meaning:** kubelet cannot pull the container image.

```bash
kubectl describe pod <pod>   # Events will show the exact pull error
```

Common causes:
- Typo in image name/tag, or tag doesn't exist.
- Private registry without an `imagePullSecrets` configured on the Pod/ServiceAccount.
- Registry auth expired or wrong credentials.
- Network policy or firewall blocking egress to the registry.
- Rate limiting from the registry (e.g., Docker Hub anonymous pull limits).

```yaml
spec:
  imagePullSecrets:
  - name: regcred
```

## 5. OOMKilled

**Meaning:** container exceeded its memory **limit**; the kernel OOM killer terminated it (exit code 137, `Reason: OOMKilled`).

```bash
kubectl describe pod <pod> | grep -A5 "Last State"
kubectl top pod <pod>          # compare live usage vs configured limits
```

Fix options:
- Raise memory `limits` if the workload legitimately needs more.
- Investigate a memory leak if usage grows unbounded over time.
- Check for a missing/too-low `requests` causing bad scheduling density (many pods packed tightly, all pressuring node memory).

## 6. Service / Ingress Connectivity Problems

**Systematic diagnosis path:**

```bash
# 1. Does the Service have any endpoints at all?
kubectl get endpoints <service> -n <ns>
# Empty? -> selector doesn't match any pod labels, or pods aren't Ready (readiness probe failing)

# 2. Do the pod labels actually match the Service selector?
kubectl get pods -n <ns> --show-labels
kubectl get svc <service> -n <ns> -o yaml | grep -A3 selector

# 3. Are the target pods actually Ready?
kubectl get pods -n <ns> -o wide

# 4. Test connectivity FROM another pod directly to a pod IP (bypass Service)
kubectl run debug --image=nicolaka/netshoot -it --rm -- curl <pod-ip>:<port>

# 5. Test connectivity to the Service ClusterIP
kubectl run debug --image=nicolaka/netshoot -it --rm -- curl <service-name>.<ns>.svc.cluster.local

# 6. If DNS resolution itself fails
kubectl run debug --image=nicolaka/netshoot -it --rm -- nslookup <service>.<ns>.svc.cluster.local
kubectl get pods -n kube-system -l k8s-app=kube-dns   # is CoreDNS healthy?

# 7. Ingress-specific: check the Ingress Controller's own logs
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
kubectl describe ingress <ingress-name> -n <ns>   # check backend resolution, TLS secret issues
```

**Narrowing logic:** Pod-to-Pod direct IP works but Service doesn't → Service/Endpoints/kube-proxy issue. Service works internally but Ingress doesn't → Ingress rule/controller/DNS/TLS issue. Nothing works even Pod-to-Pod → CNI/NetworkPolicy issue.

## 7. Hands-On: Intentionally Break Things

```bash
# 1. CrashLoopBackOff
kubectl run crash-demo --image=busybox --restart=Always -- sh -c "exit 1"
kubectl logs crash-demo --previous
kubectl describe pod crash-demo

# 2. ImagePullBackOff
kubectl run badimage --image=nonexistent/madeupimage:v99
kubectl describe pod badimage | tail -10

# 3. OOMKilled
kubectl run oom-demo --image=polinux/stress --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"oom-demo","image":"polinux/stress","resources":{"limits":{"memory":"50Mi"}},"command":["stress"],"args":["--vm","1","--vm-bytes","100M","--vm-hang","1"]}]}}'
kubectl describe pod oom-demo | grep -A5 "Last State"

# 4. Pending (impossible resource request)
kubectl run pending-demo --image=nginx --overrides='{"spec":{"containers":[{"name":"pending-demo","image":"nginx","resources":{"requests":{"cpu":"999"}}}]}}'
kubectl describe pod pending-demo | tail -5

# Clean up
kubectl delete pod crash-demo badimage oom-demo pending-demo --ignore-not-found
```

## 8. Interview Points
- "My first three commands on any incident are always `get pods -o wide`, `describe pod`, and `logs --previous` — that resolves 80% of issues in minutes."
- "CrashLoopBackOff means the container IS starting and then dying — the fix is always in `logs --previous`, not in restarting harder."
- "For connectivity issues I work outside-in or inside-out systematically: pod IP direct → Service ClusterIP → DNS name → Ingress — each layer isolates where the break is."
- "OOMKilled is a memory limit problem (exit 137), not a CPU problem — I check `top` vs configured limits before blindly raising limits."
