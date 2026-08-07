# GSP053, Managing Deployments Using Kubernetes Engine

Five tasks, three checkpoints, twenty five minutes stated and closer to forty five. All Cloud Shell. Carries the "Do not deviate from instructions" warning.

Checkpoints: **cluster and deployments (fortune-app)**, **Canary Deployment**, **Blue-green deployment**.

## The thing that actually decides how this run goes: cpu

The deployment manifests set container `limits` but **no `requests`**, so Kubernetes sets requests equal to limits: **200m cpu per pod, Guaranteed QoS**. The cluster is three `e2-small` nodes, roughly 940m allocatable each, with about 600m per node already taken by GKE system pods.

That leaves room for **one or two app pods per node**, about five or six in total. Blue's 3 plus canary's 1 fills it. Then:

```
0/3 nodes are available: 3 Insufficient cpu.
```

Everything that went wrong in our run traces back to this. Three separate places it bites:

1. **The rolling update `undo`** needs to start a replacement pod before it can drop an old one, and with canary running there is nowhere to put it. The undo sits at `2 out of 3 new replicas have been updated` indefinitely.
2. **Green wants 3 replicas.** With blue at 3, only one green pod schedules and two sit `Pending`.
3. Nothing reports this as a capacity problem unless you go looking. `kubectl get deployments` shows `3/3` and healthy while a fourth pod is stuck.

The diagnostic:

```
kubectl get pods -l app=fortune-app
kubectl describe pod PENDING_POD_NAME | tail -15
```

The `FailedScheduling` event names the reason directly.

The fixes, both inside the lab's own scope since it has you scale blue in task 2:

```
kubectl delete deployment fortune-app-canary     # after its checkpoint is green
kubectl scale deployment fortune-app-blue --replicas=1   # before creating green
```

Checkpoints latch, so removing the canary once it has scored is safe. Blue on one pod still serves `1.0.0` and still satisfies the blue-green rollback.

## rollout status blocks forever on a paused deployment

The lab claims `kubectl rollout status` returns immediately after a pause, noting it "might immediately report successfully rolled out". **It does not.** It sits on `Waiting for deployment ... 1 out of 3 new replicas have been updated` and never returns.

Always give it a timeout:

```
kubectl rollout status deployment/fortune-app-blue --timeout=180s
```

Without one it hangs, and Ctrl+C out of a pasted block is genuinely dangerous. In our run the remaining buffered commands kept executing after each Ctrl+C, so `resume` and `undo` raced each other and the deployment ended up on the wrong version with no clear indication of what had run.

## Replacing kubectl edit

Task 3 has you `kubectl edit deployment fortune-app-blue` and change two values in vi. Scripting it is worth doing, because `rollout pause` needs to fire while the rollout is still in flight and with three replicas it can finish before you quit the editor.

```
kubectl get deployment fortune-app-blue -o yaml > blue-v2.yaml
sed -i 's/fortune-service:1.0.0/fortune-service:2.0.0/' blue-v2.yaml
sed -i "s/value: 1.0.0/value: 2.0.0/" blue-v2.yaml
grep -n '2\.0\.0' blue-v2.yaml
kubectl apply -f blue-v2.yaml
kubectl rollout pause deployment/fortune-app-blue
```

**`kubectl get -o yaml` emits the env value unquoted** as `value: 1.0.0`, not `value: "1.0.0"` as it appears in the source manifest. A sed written with quotes silently matches nothing, and `grep` then returns only the image line. Check for **two** matches before applying.

`kubectl apply` on a resource made with `kubectl create` warns about a missing `last-applied-configuration` annotation and patches it automatically. Harmless.

## Checking the paused mid rollout state

The lab's step here curls each pod's **pod ip** from Cloud Shell, which is not routable from there. It hangs rather than failing. This shows the same thing and works:

```
kubectl get pods -l app=fortune-app -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image
```

A mix of `:1.0.0` and `:2.0.0` is the paused state.

## Order that avoids all of the above

1. cluster, blue deployment, service, wait for external ip, `curl /version` gives `1.0.0` → **checkpoint 1**
2. scale to 5, back to 3
3. scripted image bump, `pause`, inspect images, `resume`, `undo`, all with `--timeout`
4. create canary, `curl` in a loop, mostly `1.0.0` with occasional `2.0.0` → **checkpoint 2**
5. **delete the canary**
6. apply blue service, create green, **scale blue to 1** so green can schedule
7. switch service to green, five `2.0.0`; switch back to blue, five `1.0.0` → **checkpoint 3**

Useful one liner for the external ip, since it takes a couple of minutes:

```
until [ -n "$(kubectl get svc fortune-app -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')" ]; do echo "waiting for external ip"; sleep 10; done; echo "IP READY"
```

Then `export APP_IP=$(kubectl get svc fortune-app -o=jsonpath="{.status.loadBalancer.ingress[0].ip}")` and use `$APP_IP` throughout. Re-export it after each service apply and in any new tab.

## What the three strategies actually demonstrate

- **Rolling update**: one deployment, replicas replaced gradually. `pause` freezes it mid way with both versions live, `resume` finishes, `undo` reverts to the previous ReplicaSet.
- **Canary**: a second deployment sharing the `app: fortune-app` label, so the same service load balances across both. Traffic split is just the pod ratio, one canary pod against three prod pods.
- **Blue-green**: two full deployments, and the service selector decides which one gets traffic. Green runs fully and receives nothing until the selector moves, and rollback is re-applying the old service manifest. Instant, at the cost of running both versions at once, which is exactly why the cpu ran out.
