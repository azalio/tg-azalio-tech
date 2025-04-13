https://www.reddit.com/r/kubernetes/comments/1jyi8jo/how_to_disable_kubeapi_server_anonymous_auth/

**Title:** How to Disable Kube-API Server Anonymous Auth Globally BUT Keep /livez &amp; /readyz Working (KEP-4633 Deep Dive)

---

Hey r/kubernetes! 👋

Ever wanted to tighten security by setting `--anonymous-auth=false` on your `kube-apiserver` but worried about breaking essential health checks like `/livez`, `/readyz`, and `/healthz`? 🤔

By default, disabling anonymous auth blocks *everything*, including those crucial endpoints used by load balancers and monitoring. But leaving it enabled, even with RBAC, might feel like an unnecessary risk.

Turns out, there's a cleaner way thanks to [KEP-4633](https://github.com/kubernetes/enhancements/blob/master/keps/sig-auth/4633-anonymous-auth-configurable-endpoints/README.md) and the `AuthenticationConfiguration` object (Alpha in v1.31, Beta in v1.32).

This lets you:
1. Set `--anonymous-auth=false` globally.
2. Explicitly allow anonymous access *only* for specific paths like `/livez`, `/readyz`, `/healthz` via a configuration file.

Now, unauthenticated requests to `/apis` (or anything else) get a proper `401 Unauthorized`, while your health checks keep working perfectly. ✅

I did a deep dive into how this works, including the necessary `kube-apiserver` flags, the `AuthenticationConfiguration` YAML structure, and example audit logs showing the difference.

**Check out the full guide on Medium:**
[Securing Kubernetes API Server Health Checks Without Anonymous Access](https://medium.com/@azalio_16174/securing-kubernetes-api-server-health-checks-without-anonymous-access-0be907fbf5e8)

Hope this helps someone else looking to secure their clusters without compromise! 👍

#kubernetes #k8s #security #devops #infosec