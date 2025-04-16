🔥 DevOps-реальность: DiskPressure, Talos и запуск Pod’а любой ценой

На днях столкнулся с реальной проблемой на Kubernetes-кластере под управлением Talos: тестировал local-path-provisioner и накосячил в конфиге для `fio` все worker ноды забились до состояния DiskPressure. Нужно было срочно почистить директорию на самой ноде.

Удаление PV не помогало, под с `local-path-provisioner` тоже вышибло с ноды :)

Ну я же умный! Подумал я и выкатил DaemonSet с toleration для node.kubernetes.io/disk-pressure — но ничего не вышло: kubelet сразу эвакуирует ("Evicted") поды. Обычные tolerations тут не спасают — DiskPressure в K8s работает беспощадно, особенно на иммутабельных дистрибутивах вроде Talos, где напрямую на ноду не попасть ни ssh, ни bash’ем (только через talosctl).

Что реально помогло?  
Ключ оказалась в приоритетах:  
Прописал у Pod’а
`priorityClassName: "system-cluster-critical"`
и только с ним Kubernetes разрешил моему DaemonSet’у стартовать даже под DiskPressure! Это помогло заскочить в контейнер, смонтировать /opt через hostPath, быстренько почистить мусор — и только после этого ушло DiskPressure.

Лайфхак:
- На Talos (и не только) при DiskPressure стандартный набор tolerations _не спасает_ от немедленного Eviction.
- Используйте критический PriorityClass (system-cluster-critical) — и у вас есть шанс стартовать POD даже при полном диске и доделать нужную работу без болезненно-мучительных последующий операций.
