https://t.me/azalio_tech/48

🔒 **Отключаем анонимный доступ к kube-apiserver, но оставляем health checks!**

Привет! Недавно ко мне пришел коллега-безопасник (Дима привет!) с интересным вопросом: как полностью отключить анонимный доступ к API-серверу Kubernetes, но оставить рабочими проверки `/livez`, `/readyz` и `/healthz`? 🤔 Сходу не ответил, полез копаться в исходниках и KEPах.

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
    spec:
      containers:
      - command:
        - kube-apiserver
        # ... другие флаги ...
        - --anonymous-auth=false # <--- Выключаем!
        - --authentication-config=/etc/kubernetes/auth-config.yaml # <--- Указываем конфиг
    ```

Затем создаем файл конфигурации `/etc/kubernetes/auth-config.yaml` на control plane нодах и монтируем его в под `kube-apiserver`:
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
    *(Не забудьте добавить `volume` и `volumeMount` в манифест `kube-apiserver` для этого файла)*

В итоге получаем:
- Запросы к `/livez`, `/readyz`, `/healthz` проходят как `system:anonymous`.
- Запросы к другим путям (например, `/apis`) без аутентификации получают `401 Unauthorized`.

Кстати, эта функциональность появилась как Alpha в Kubernetes 1.31 и стала Beta в 1.32.

Теперь можно спать спокойнее, зная, что анонимный доступ под контролем! 👍

#kubernetes #k8s #security #authentication #kubeadm #api #devops #infosec