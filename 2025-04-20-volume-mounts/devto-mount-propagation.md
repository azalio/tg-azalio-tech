---
title: Kubernetes Agent Blind to New Mounts? Demystifying Mount Propagation
tags: [kubernetes, k8s, linux, devops, storage, mountpropagation, namespaces]
published: false # Set to true when ready
---

## How `mountPropagation: HostToContainer` leverages Linux namespaces to solve a common agent problem

Ever run into this frustrating scenario? You deploy a Kubernetes agent (like an I/O limiter, monitoring tool, or operator) that needs to interact with PersistentVolumes mounted by Kubelet for other pods. It works fine initially, seeing all existing mounts. But then, a *new* pod gets scheduled, its PV gets mounted by Kubelet... and your agent is completely blind to it! 😠 Restarting the agent fixes it, but that's just a workaround, not a solution.

I recently wrestled with this exact issue while building an I/O limiter that needed device `MAJOR:MINOR` numbers from pod mounts. Let's dive into why this happens and how to fix it properly using Kubernetes `mountPropagation`.

### The Root Cause: Linux Mount Namespaces

The core of the issue lies in **Linux mount namespaces**. By default, each container in Kubernetes gets its own isolated mount namespace. Think of it as a private view of the system's mount points. When a container starts, it gets a *copy* of the host's mount table at that moment.

Crucially, subsequent mount events happening on the host (like Kubelet mounting a new PV under `/var/lib/kubelet/pods/`) are *not* automatically reflected inside the container's isolated namespace. This default behavior corresponds to `private` propagation in Linux terminology.

In Kubernetes, this default isolation is represented by:

```yaml
mountPropagation: None # (Default if not specified)
```

This isolation is great for security and preventing containers from interfering with each other or the host, but it causes the "blind agent" problem.

### The Solution: Kubernetes `mountPropagation`

Kubernetes provides the `mountPropagation` field within a `volumeMount` definition to control how mount events are shared between the host and the container's namespace. It bridges the isolation gap when needed. There are three modes:

1. **`None` (Default):** Complete isolation. No mount events propagate in either direction. (Linux: `private`)
2. **`HostToContainer`:** One-way street! Mount events from the host **propagate into** the container. Mounts created *inside* the container do **not** propagate back to the host. (Linux: `rslave` - recursive slave). **This is exactly what we need for our agent!** 👍
3. **`Bidirectional`:** Two-way street. Mounts propagate from host to container AND from container back to the host (and other containers in the pod using `Bidirectional`). (Linux: `rshared` - recursive shared). **Use with extreme caution!** This can affect the host system and requires the container to be privileged or have `CAP_SYS_ADMIN`.

### The Recipe: Fixing the Blind Agent

To allow our agent container to see new PVs mounted by Kubelet under `/var/lib/kubelet/pods` *after* the agent has started, we need to mount that host directory into the agent and set `mountPropagation: HostToContainer`:

```yaml
# Example Pod spec for the agent
spec:
  containers:
  - name: my-observing-agent
    image: my-agent-image:latest
    # Optional: If your agent needs to *do* something with mounts,
    # it might need privileges beyond just seeing them.
    # Bidirectional *requires* privileged or CAP_SYS_ADMIN.
    # HostToContainer itself doesn't require extra privileges just to see mounts.
    # securityContext:
    #   privileged: false
    #   capabilities:
    #     add: ["SYS_ADMIN"] # Example if needed for other operations
    volumeMounts:
    - name: kubelet-pods-dir
      # The path inside the container where the host dir will be mounted
      mountPath: /mnt/kubelet-pods
      # The magic setting!
      mountPropagation: HostToContainer
  volumes:
  - name: kubelet-pods-dir
    hostPath:
      # The directory on the host we want to observe
      path: /var/lib/kubelet/pods
      # Ensures the path exists on the host
      type: DirectoryOrCreate
```

With this configuration, any new directory or mount point created by Kubelet under `/var/lib/kubelet/pods` on the host will automatically appear under `/mnt/kubelet-pods` inside the `my-observing-agent` container, without requiring a restart. Problem solved! ✅

### Under the Hood: How It Works

What happens when you set `HostToContainer`?

1. **Kubelet:** Sees `mountPropagation: HostToContainer` in the Pod spec.
2. **CRI Request:** Kubelet translates this to the corresponding enum (`PROPAGATION_HOST_TO_CONTAINER`) in its request to the Container Runtime Interface (CRI).
3. **Container Runtime (containerd, CRI-O):** Receives the request and uses Linux system calls (specifically the `mount` syscall with flags like `MS_SLAVE | MS_REC`) to configure the container's mount point (`/mnt/kubelet-pods` in our example) as a **recursive slave** (`rslave`) of the corresponding host mount point (`/var/lib/kubelet/pods`).
4. **Propagation:** Because the container's mount is now a slave of the host's mount, any mount events occurring under the host path are automatically propagated by the Linux kernel into the container's mount namespace at the slave mount point.

### Important Considerations

* **Security (`Bidirectional`):** Seriously, be careful with `Bidirectional`. A container mount propagating back to the host can have significant security implications, especially in multi-tenant clusters. It generally requires the container to run as privileged or have `CAP_SYS_ADMIN`.
* **Privileges (`HostToContainer`):** Simply *observing* mounts with `HostToContainer` doesn't inherently require extra privileges. However, if your agent needs to *perform actions* within those propagated mounts (like modifying files owned by root, mounting things itself), it might need appropriate capabilities (`CAP_SYS_ADMIN`) or run as privileged, independent of the `mountPropagation` setting itself.
* **Debugging:** If things aren't working, check the mount table inside the container (`findmnt` or `cat /proc/self/mountinfo`) and compare it to the host's. Look for the `master:` entries or propagation flags (`shared`, `slave`) associated with your mount point.
* **Historical Note:** For a brief period around Kubernetes 1.10, the default was accidentally changed to `HostToContainer` before being reverted. If you're managing a very old cluster, be aware of this possibility ([#62462](https://github.com/kubernetes/kubernetes/pull/62462)).

### Conclusion

Kubernetes `mountPropagation` is a powerful mechanism rooted in Linux mount namespaces that allows you to selectively break container isolation for specific use cases. Understanding the `HostToContainer` mode (`rslave`) is key to solving the common problem of agents needing visibility into dynamically created host mounts, like those managed by Kubelet. By applying it correctly, you can build more robust and reliable agents and operators without resorting to restarting them.

---

Connect with me on [LinkedIn](https://www.linkedin.com/in/azalio/)!
