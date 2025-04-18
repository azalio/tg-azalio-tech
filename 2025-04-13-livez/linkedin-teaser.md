https://www.linkedin.com/feed/update/urn:li:activity:7317463276898930688/

🔒 **Securing Kubernetes API Server Health Checks: Can You Disable Anonymous Access Without Breaking Things?**

By default, Kubernetes allows anonymous access to health check endpoints like `/livez` and `/readyz`. While convenient, this can pose a security risk. But what if you want to disable anonymous access globally for better security, yet still allow these crucial checks?

It turns out there's a way using `AuthenticationConfiguration` introduced via KEP-4633! This allows fine-grained control over which paths permit anonymous requests, even when `--anonymous-auth=false`.

Want to learn how to implement this for a more secure cluster?

➡️ **Read the full deep-dive (English):** https://medium.com/@azalio_16174/securing-kubernetes-api-server-health-checks-without-anonymous-access-0be907fbf5e8
➡️ **Читайте подробный разбор (на русском):** https://t.me/azalio_tech/48

#kubernetes #k8s #security #devops #infosec #authentication #apiserver #linkedinpost