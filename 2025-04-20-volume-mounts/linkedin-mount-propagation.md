🤔 **Kubernetes Agent Can't See New Mounts? Let's Talk `mountPropagation`!**

Ever deployed a Kubernetes agent or operator only to find it blind to PVs mounted *after* it started? Restarting works, but it's a pain. This common issue stems from Linux mount namespace isolation.

The fix? Kubernetes' `mountPropagation: HostToContainer`! This setting allows mount events from the host (like new PVs) to propagate into your container's namespace without compromising security by allowing container mounts back onto the host.

It leverages the Linux `rslave` propagation mode under the hood.

Want the full technical breakdown and YAML recipe?

➡️ **Read the deep-dive (English):** [Medium](https://medium.com/@azalio_16174/kubernetes-agent-blind-to-new-mounts-demystifying-mount-propagation-42176595ca6a)
➡️ **Читайте подробный разбор (на русском):** [tg](https://t.me/azalio_tech/49)

#kubernetes #k8s #devops #linux #storage #mountpropagation #namespaces #linkedinpost