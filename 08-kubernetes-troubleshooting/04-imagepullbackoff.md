# 🔴 ImagePullBackOff / ErrImagePull

Kubernetes can't pull your container image. Always check the exact error message — the cause is almost always one of four things.

```bash
kubectl describe pod myapp-xxx -n production | grep -A10 "Events:"
# Events:
#   Warning  Failed     2m    kubelet  Failed to pull image "myregistry.io/myapp:v2.0.0":
#            rpc error: code = Unknown desc = failed to pull and unpack image:
#            unexpected status code 401 Unauthorized
```

## The four causes and fixes

```bash
# Cause 1: Wrong image tag or image doesn't exist
kubectl describe pod myapp-xxx -n production | grep "Image:"
# Image: myregistry.io/myapp:v2.0.0-typo   ← Tag doesn't exist

# Verify the image exists in your registry
aws ecr describe-images --repository-name myapp \
    --image-ids imageTag=v2.0.0
```

```bash
# Cause 2: Registry authentication failure (401)

# Check if imagePullSecret is configured
kubectl get pod myapp-xxx -n production -o jsonpath='{.spec.imagePullSecrets}'

# Create the pull secret if missing
kubectl create secret docker-registry regcred \
    --docker-server=myregistry.io \
    --docker-username=myuser \
    --docker-password=mypassword \
    -n production
# Add it to deployment
# spec:
#   imagePullSecrets:
#     - name: regcred
```

```bash
# Cause 3: Node can't reach the registry (network/firewall)
# Test from within the cluster:
kubectl run registry-test --rm -it --image=curlimages/curl -- \
    curl -I https://myregistry.io/v2/
```

```bash
# Cause 4: Image too large, node disk full
kubectl describe node <node-name> | grep -A5 "Conditions:"
# DiskPressure = True → Node is out of disk space
```

---

## Quick Recap

| # | Cause | How to confirm | Fix |
|---|-------|-----------------|-----|
| 1 | Wrong tag / image doesn't exist | `describe pod` → `Image:` field, check registry | Correct the tag/push image |
| 2 | Registry auth failure (401) | Check `imagePullSecrets` | Create `docker-registry` secret, attach to spec |
| 3 | Node can't reach registry | `curl -I` from a test pod | Fix network/firewall rules |
| 4 | Node disk full | `describe node` → `DiskPressure` condition | Free up disk space on node |

**Rule of thumb:** always read the exact `Events:` error message first — it tells you which of the four causes you're dealing with (401 = auth, timeout = network, "not found" = tag, disk-related = node capacity).

**Previous:** [← OOMKilled](./03-oomkilled.md) | **Next:** [Pending Pods →](./05-pending-pods.md)
