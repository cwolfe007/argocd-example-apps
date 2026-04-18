# ARC Controller: "too many open files" — Root Cause & Fix

## Symptom

The `actions-runner-controller` pod crashes on startup with:

```
ERROR   problem running manager {"error": "too many open files"}
```

Preceding the crash, all informers fail to sync:

```
ERROR   controller-runtime.source   failed to get informer from cache
        {"error": "Timeout: failed waiting for *v1alpha1.Runner Informer to sync"}
```

The controller never acquires the leader lease and exits.

## Misleading dead ends

### Node-level `fs.file-max`

```bash
kubectl debug node/kind-control-plane -it --image=busybox -- cat /proc/sys/fs/file-max
# 9223372036854775807  (essentially unlimited)
```

The host kernel allows an astronomically large number of open files. Not the cause.

### Container / containerd `nofile` ulimit

```bash
podman exec kind-control-plane sh -c \
  "cat /proc/$(pgrep containerd | head -1)/limits" | grep "open files"
# Max open files   1048576   1048576   files
```

Containerd and the kind node container both inherit a 1 048 576 (`nofile`) hard limit. Not the cause.

## Root cause: inotify instance exhaustion

`controller-runtime` (the framework ARC is built on) opens one **inotify instance** per informer watch. Inotify instances are file descriptors; when the per-user instance cap is hit, the kernel returns `EMFILE` — which surfaces as "too many open files".

```bash
podman exec kind-control-plane \
  sysctl fs.inotify.max_user_instances fs.inotify.max_user_watches fs.inotify.max_queued_events
```

```
fs.inotify.max_user_instances = 128      ← default, far too low
fs.inotify.max_user_watches   = 121559
fs.inotify.max_queued_events  = 16384
```

ARC registers controllers for Runner, RunnerDeployment, RunnerReplicaSet, RunnerSet, HorizontalRunnerAutoscaler, Pod, PersistentVolumeClaim, PersistentVolume, and StatefulSet — easily exceeding 128 instances once the API server, kubelet, and other system controllers also hold instances.

## Debug checklist (in order)

1. **Read the last line of the crash log** — the actual error is always at the bottom.
2. **Check node `fs.file-max`** — rules out kernel-level exhaustion.
3. **Check container `nofile` ulimit** — rules out runtime-level FD cap.
4. **Check inotify limits** — the likely culprit on kind/podman clusters with default kernel params.

```bash
# Step 1 — confirm inotify is the bottleneck
podman exec kind-control-plane \
  sysctl fs.inotify.max_user_instances fs.inotify.max_user_watches

# Step 2 — count current inotify usage on the node (optional, informational)
podman exec kind-control-plane \
  sh -c "find /proc/*/fd -lname 'anon_inode:inotify' 2>/dev/null | wc -l"
```

## Fix

> **Reference:** This is a [known kind issue](https://kind.sigs.k8s.io/docs/user/known-issues/#pod-errors-due-to-too-many-open-files). The fix must be applied on the **host machine**, not inside the kind node — kind node containers share the host kernel's inotify namespace, so host sysctls are what actually matter.

### Immediate (ephemeral — lost on host reboot)

```bash
sudo sysctl fs.inotify.max_user_instances=512
sudo sysctl fs.inotify.max_user_watches=524288

kubectl rollout restart deployment/actions-runner-controller -n actions-runner-system
```

### Permanent — host sysctl

```bash
echo "fs.inotify.max_user_watches = 524288" | sudo tee -a /etc/sysctl.conf
echo "fs.inotify.max_user_instances = 512"  | sudo tee -a /etc/sysctl.conf
sudo sysctl -p /etc/sysctl.conf
```

Or drop a dedicated file in `/etc/sysctl.d/`:

```bash
sudo tee /etc/sysctl.d/99-inotify.conf <<EOF
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 512
EOF
sudo sysctl --system
```

This survives reboots and applies automatically to all future kind clusters on this host.

## Why the informer timeouts appear first

The controller starts all informers concurrently. Each informer attempts to open an inotify instance. Once the 128-instance cap is hit, later informers block waiting for a descriptor that never arrives, and `controller-runtime` times them out after its sync deadline. The manager then exits, producing the "too many open files" error as the final summary.
