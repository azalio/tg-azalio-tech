😠 **Агент-лимитер “слепнет” после появления новых PV? Раз и навсегда разбираемся с `mountPropagation` в Kubernetes!**

Недавно писал I/O-лимитер для LVM-томов: агент читает маунты подов, чтобы вычислить `MAJOR:MINOR` для добавления в `io.max`, и всё ок… пока на узел не приедет новый Pod. Его маунта агент уже не видит. Перезапуск помогает, но это же костыль!

---

## Почему так происходит

* Каждый контейнер стартует в своём *privаte* mount-namespace → изменения на хосте туда не пролетают.
* В Kubernetes это равно `mountPropagation: None`.

---

## Как починить

Параметр `mountPropagation` у `volumeMounts` имеет 3 режима:

* **None** - полная изоляция (`rprivate`), дефолт.  
* **HostToContainer** - маунты летят *с хоста → в контейнер* (`rslave`). **Нам нужен именно он.**  
* **Bidirectional** - маунты ходят в обе стороны (`rshared`). Работает *только* в `privileged`-контейнере, иначе Pod не стартует.

---

## Рецепт

```yaml
spec:
  containers:
  - name: my-agent
    image: my-agent-image
    volumeMounts:
    - name: kubelet-pods
      mountPath: /var/lib/kubelet/pods
      mountPropagation: HostToContainer
  volumes:
  - name: kubelet-pods
    hostPath:
      path: /var/lib/kubelet/pods
      type: Directory
```

Теперь каждое *новое* bind-mount событие внутри `/var/lib/kubelet/pods` мгновенно видно агенту - без рестартов.

---

## Что происходит под капотом

1. Kubelet видит `HostToContainer` и пишет в CRI `PROPAGATION_HOST_TO_CONTAINER`.  
2. CRI-shim (containerd/CRI-O) делает каталог `rslave`.  
3. Все будущие маунты, созданные kubelet’ом на хосте, автоматически “проливаются” в контейнер.

---

### На заметку

* `Bidirectional` опасен в мульти-тенант окружениях: контейнер может пролить маунты *на хост*.  
* В Kubernetes 1.10 было краткое время, когда [дефолтом неожиданно стал](https://github.com/kubernetes/kubernetes/pull/62462) `HostToContainer`. Если админите древний кластер - проверьте (хотя боюсь вам уже ничего не поможет...).  
* Cilium тоже использует этот трюк чтобы [следить](https://github.com/cilium/cilium/blob/v1.17.3/install/kubernetes/cilium/templates/cilium-envoy/daemonset.yaml#L195) за bpf мапами.

---

Полезные ссылки  
• Документация: <https://kubernetes.io/docs/concepts/storage/volumes/#mount-propagation>  
• CRI интерфейс: <https://github.com/kubernetes/cri-api/blob/release-1.33/pkg/apis/runtime/v1/api.proto#L226>  
• man mount: <https://man7.org/linux/man-pages/man8/mount.8.html>

#kubernetes #k8s #mountPropagation #linux #devops #storage
