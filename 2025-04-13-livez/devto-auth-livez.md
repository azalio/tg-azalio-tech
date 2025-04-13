---
title: Securing Kubernetes API Server Health Checks Without Anonymous Access
tags: [kubernetes, k8s, security, devops, infosec]
published: false # Set to true when ready to publish
---

## How to disable anonymous auth globally but keep /livez, /readyz, and /healthz accessible

Recently, a security-savvy colleague posed an interesting question: "Is it possible to disable anonymous access to the Kubernetes API server entirely, but still allow the `/livez`, `/readyz`, and `/healthz` endpoints to work?" 🤔

Honestly, I didn't know the answer offhand, so I dove into the Kubernetes source code and relevant KEPs (Kubernetes Enhancement Proposals). Here's what I found and how you can implement it.

### The Default Scenario: Anonymous Access Enabled

As you probably know, by default, `kube-apiserver` allows anonymous authentication:

```
--anonymous-auth     Default: true
```

> Enables anonymous requests to the secure port of the API server. Requests that are not rejected by another authentication method are treated as anonymous requests. Anonymous requests have a username of `system:anonymous`, and a group name of `system:unauthenticated`.

When you set up a cluster using `kubeadm` with default settings, anonymous requests to health endpoints (and potentially others like `/version`, depending on default bindings) succeed:

```bash
# curl -k https://<YOUR_API_SERVER_IP>:6443/livez ; echo
ok
```

Looking at the audit logs, we can confirm the request is handled by `system:anonymous`:

```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Metadata",
  "auditID": "2caca66d-ccec-4b1f-8116-86741193cdce", // Example ID
  "stage": "ResponseComplete",
  "requestURI": "/livez",
  "verb": "get",
  "user": {
    "username": "system:anonymous", // <--- Anonymous user
    "groups": [
      "system:unauthenticated"
    ]
  },
  "sourceIPs": [ "<SOURCE_IP>" ],
  "userAgent": "curl/7.88.1",
  "responseStatus": { "code": 200 },
  "requestReceivedTimestamp": "2025-04-13T14:41:33.630618Z", // Example timestamp
  "stageTimestamp": "2025-04-13T14:41:33.630908Z", // Example timestamp
  "annotations": {
    "authorization.k8s.io/decision": "allow",
    "authorization.k8s.io/reason": "RBAC: allowed by ClusterRoleBinding \"system:public-info-viewer\"..."
  }
}
```

However, anonymous users can't access other API paths like `/apis`:

```json
{
  // ... audit metadata ...
  "requestURI": "/apis",
  "verb": "get",
  "user": {
    "username": "system:anonymous",
    "groups": [ "system:unauthenticated" ]
  },
  "responseStatus": {
    "status": "Failure",
    "message": "forbidden: User \"system:anonymous\" cannot get path \"/apis\"",
    "reason": "Forbidden",
    "code": 403 // <--- Forbidden
  },
  // ... timestamps and annotations ...
  "annotations": {
    "authorization.k8s.io/decision": "forbid",
    "authorization.k8s.io/reason": ""
  }
}
```

While this default setup restricts access, leaving anonymous authentication enabled at all, even for specific paths, can be a security concern. Misconfigured RBAC or potential vulnerabilities could expose more than intended. 😟

### The Solution: KEP-4633 and AuthenticationConfiguration

Digging into the Kubernetes source code led me to [KEP-4633: Make anonymous authentication configuration endpoints configurable](https://github.com/kubernetes/enhancements/blob/master/keps/sig-auth/4633-anonymous-auth-configurable-endpoints/README.md). This KEP addresses the exact concern of wanting to disable anonymous access globally while still allowing essential health checks (which don't necessarily need full TCP checks to be useful).

The solution, documented [here](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#anonymous-authenticator-configuration), involves using an `AuthenticationConfiguration` object.

**How to Implement It:**

1.  **Disable Global Anonymous Auth:**
    Modify your `kube-apiserver` manifest (usually a static pod in `/etc/kubernetes/manifests/`) to include these flags:
    ```yaml
    spec:
      containers:
      - command:
        - kube-apiserver
        # ... other flags ...
        - --anonymous-auth=false # <--- Disable global anonymous auth
        - --authentication-config=/etc/kubernetes/auth-config.yaml # <--- Point to the config file
        volumeMounts:
        # ... other volume mounts ...
        - mountPath: /etc/kubernetes/auth-config.yaml
          name: auth-config-volume
          readOnly: true
      volumes:
      # ... other volumes ...
      - name: auth-config-volume
        hostPath:
          path: /etc/kubernetes/auth-config.yaml
          type: File
    ```

2.  **Create the Authentication Configuration File:**
    Create the following file on your control plane node(s) at `/etc/kubernetes/auth-config.yaml`:
    ```yaml
    # /etc/kubernetes/auth-config.yaml
    apiVersion: apiserver.config.k8s.io/v1beta1
    kind: AuthenticationConfiguration
    anonymous:
      # Enable anonymous access ONLY for the paths listed below
      enabled: true
      conditions:
      - path: /livez
      - path: /readyz
      - path: /healthz
    ```
    *Ensure the `kube-apiserver` pod restarts to pick up the changes.*

### The Result: Granular Anonymous Access

With this configuration:

✅ Requests to `/livez`, `/readyz`, and `/healthz` are still allowed for `system:anonymous`. The audit log looks similar to the default case for these paths.

```json
{
  // ... audit metadata ...
  "requestURI": "/livez",
  "verb": "get",
  "user": {
    "username": "system:anonymous", // <--- Still anonymous
    "groups": [ "system:unauthenticated" ]
  },
  "responseStatus": { "code": 200 }, // <--- Still OK
  // ... timestamps and annotations ...
}
```

❌ Requests to any *other* path (like `/apis`) without valid credentials will now receive a `401 Unauthorized` instead of a `403 Forbidden`, because the request isn't even treated as anonymous anymore.

```bash
# curl -k https://<YOUR_API_SERVER_IP>:6443/apis ; echo
Unauthorized
```

The audit log confirms this:

```json
{
  // ... audit metadata ...
  "requestURI": "/apis",
  "verb": "get",
  "user": {}, // <--- No user identified (not even anonymous)
  "responseStatus": {
    "status": "Failure",
    "message": "Unauthorized",
    "reason": "Unauthorized",
    "code": 401 // <--- Unauthorized
  },
  // ... timestamps ...
}
```

### Feature Availability

According to the [KEP issue tracker](https://github.com/kubernetes/enhancements/issues/4633), this feature graduated to Alpha in Kubernetes v1.31 and Beta in v1.32.

This approach provides a more secure posture by default, explicitly allowing anonymous access only where strictly necessary for health checks, without relying solely on RBAC for protection against anonymous requests to other endpoints. Now you can tighten security without breaking essential cluster monitoring! 👍

---

Connect with me on [LinkedIn](https://www.linkedin.com/in/azalio/)!