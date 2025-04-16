🔥 DevOps-реальность: DiskPressure, Talos и запуск Pod’а любой ценой

На днях словил реальную боль на K8s кластере под Talos: накосячил с `fio` конфигом для `local-path-provisioner` и забил все worker ноды до состояния DiskPressure.

Удаление PV не помогало, под provisioner'а тоже вылетел.

Думал, самый умный: выкатил DaemonSet с toleration для `node.kubernetes.io/disk-pressure`... но нет! Kubelet все равно немедленно эвакуирует такие поды. Обычные tolerations тут бессильны, особенно на иммутабельном Talos, куда просто так не зайти.

Что реально спасло?

Ключ в приоритете! Прописал у Pod'а:
`priorityClassName: "system-cluster-critical"`

Только с этим классом Kubernetes разрешил моему DaemonSet'у стартануть даже под DiskPressure! Это позволило заскочить в контейнер, смонтировать `/opt` через `hostPath`, быстро почистить мусор — и вуаля, DiskPressure ушел.

**Лайфхак:**
*   На Talos (и не только) при DiskPressure стандартные tolerations **не спасают** от немедленного Eviction.
*   Используйте критический PriorityClass (`system-cluster-critical`) — это ваш шанс запустить Pod даже при полном диске и спасти ситуацию без лишней боли. 💪

#kubernetes #devops #talos #diskpressure #lifehack