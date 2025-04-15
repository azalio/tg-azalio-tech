---
title: "Отключаем анонимный доступ к kube-apiserver, но оставляем health checks"
date: 2025-04-13T00:00:00+03:00 # Using date from directory name
draft: true
tags: ["kubernetes", "k8s", "security", "authentication", "kube-apiserver", "devops", "infosec"]
description: "Узнайте, как тонко настроить анонимный доступ к API-серверу Kubernetes, отключив его глобально, но сохранив доступ к эндпоинтам /livez, /readyz и /healthz с помощью AuthenticationConfiguration."
---

Привет! Недавно ко мне пришел коллега-безопасник с интересным вопросом: как полностью отключить анонимный доступ к API-серверу Kubernetes, но оставить рабочими проверки `/livez`, `/readyz` и `/healthz`? 🤔 Сходу не ответил, полез копаться в исходниках и KEPах.

Проблема в том, что по умолчанию (`--anonymous-auth=true`) любой может дернуть эндпоинты health-чеков и не только health-чеков:

```bash
curl -k https://<API_SERVER_IP>:6443/livez
# Output: ok
```

Это удобно, но создает потенциальный вектор атаки, если RBAC настроен не идеально или найдется уязвимость. Безопасники такое не любят. 😟

К счастью, в KEP-4633 сообщество Kubernetes предложило решение! Теперь можно тонко настроить, к каким путям разрешен анонимный доступ, даже если глобально он выключен.

Сделать это можно так:

Сначала выключаем глобальный анонимный доступ в манифесте `kube-apiserver`:

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    # ... другие флаги ...
    - --anonymous-auth=false # <--- Выключаем!
    - --authentication-config=/etc/kubernetes/auth-config.yaml # <--- Указываем конфиг
    volumeMounts: # <--- Не забываем про volumeMount
    - name: auth-config
      mountPath: /etc/kubernetes/auth-config.yaml
      readOnly: true
    # ... другие volumeMounts ...
  volumes: # <--- И про volume
  - name: auth-config
    hostPath:
      path: /etc/kubernetes/auth-config.yaml
      type: File
  # ... другие volumes ...
```

Затем создаем файл конфигурации `/etc/kubernetes/auth-config.yaml` на control plane нодах:

```yaml
# /etc/kubernetes/auth-config.yaml
apiVersion: apiserver.config.k8s.io/v1beta1
kind: AuthenticationConfiguration
anonymous:
  enabled: true # Включаем анонимный доступ *только* для указанных путей
  conditions:
  - path: /livez
  - path: /readyz
  - path: /healthz
```

*(Не забудьте добавить `volume` и `volumeMount`, как показано в примере манифеста `kube-apiserver` выше, чтобы подхватить этот файл)*

В итоге получаем:
- Запросы к `/livez`, `/readyz`, `/healthz` проходят как `system:anonymous`.
- Запросы к другим путям (например, `/apis`) без аутентификации получают `401 Unauthorized`.

Кстати, эта функциональность появилась как Alpha в Kubernetes 1.31 и стала Beta в 1.32.

Теперь можно спать спокойнее, зная, что анонимный доступ под контролем! 👍

---

**Хотите глубже разобраться в Kubernetes и Cilium?**

Эта статья затрагивает лишь один аспект безопасности и конфигурации Kubernetes. Если вы хотите освоить Kubernetes на профессиональном уровне, понимать его внутреннее устройство, научиться эффективно использовать Cilium для сетевой безопасности и observability, приглашаю вас на мои курсы:

*   **[Название курса по Kubernetes]**: [Краткое описание и ссылка]
*   **[Название курса по Cilium]**: [Краткое описание и ссылка]

Присоединяйтесь, чтобы стать экспертом в мире облачных технологий!