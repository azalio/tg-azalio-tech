😠 **Агент не видит новые маунты? Разбираемся с `mountPropagation` в Kubernetes!**

Писал я недавно IO лимитер для LVM томов кубера и столкнулся со следующей ситуацией: запускаю агент лимитера, он видит все маунты подов (маунты мне были нужны для определения MAJOR:MINOR значений для блочного устростройста которое я хочу лимитировать) и все в принципе работает. Потом на хост приезжает новый под и я не вижу его маунта.

Перезапуск агента помогал, но это же костыль!

**В чем корень зла?**

Все дело в **Linux mount namespaces**. По умолчанию каждый контейнер получает свой изолированный "мир" маунтов (это как `private` propagation в Linux, или `mountPropagation: None` в Kubernetes). Изменения на хосте (новые PV) в этот изолированный мир просто так не попадают.

**Как починить? `mountPropagation` спешит на помощь!**

В Kubernetes есть параметр `mountPropagation` у `volumeMounts`, который позволяет управлять этим поведением. Всего три режима:

1. `None` (по умолчанию): Полная изоляция (`private`).
2. `HostToContainer`: Маунты с хоста **пробрасываются** в контейнер, но не наоборот (`rslave` в Linux). То, что нам нужно!
3. `Bidirectional`: Маунты ходят в обе стороны - с хоста в контейнер И из контейнера на хост (`rshared` в Linux). **Опасно!** Используйте с большой осторожностью.

**Решение для агента:**

Чтобы мой агент видел новые PV, монтируемые kubelet'ом (обычно в `/var/lib/kubelet/pods`), нужно использовать `HostToContainer`:

```yaml
spec:
  containers:
  - name: my-agent
    image: my-agent-image
    volumeMounts:
    - name: kubelet-pods-dir
      mountPath: /var/lib/kubelet/pods # Куда монтируем в контейнере
      mountPropagation: HostToContainer # <--- Магия здесь!
  volumes:
  - name: kubelet-pods-dir
    hostPath:
      path: /var/lib/kubelet/pods # Что монтируем с хоста
      type: DirectoryOrCreate
```

**Под капотом:**

Если кратко: Kubelet видит `HostToContainer`, говорит Container Runtime Interface (CRI) использовать `PROPAGATION_HOST_TO_CONTAINER`. Контейнерный рантайм (containerd, CRI-O) уже через системные вызовы Linux настраивает mount namespace контейнера как `rslave` по отношению к хостовому маунту.

**Итог:**

С `HostToContainer` ваш агент будет видеть все новые маунты, появляющиеся на хосте в указанной директории, без необходимости перезапуска. Проблема решена!

# kubernetes #k8s #mountpropagation #linux #namespaces #devops #kubelet #internals #storage #volumes

**Ссылки**

- [Код кубера](https://github.com/kubernetes/cri-api/blob/v0.32.4/pkg/apis/runtime/v1/api.proto#L210)
- [Дока на сайте кубера](https://kubernetes.io/docs/concepts/storage/volumes/#mount-propagation)
- [Соотвествующий man](https://man7.org/linux/man-pages/man8/mount.8.html)
