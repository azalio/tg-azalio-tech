# НЕ анонимный доступ к проверкам здоровья kube-apiserver.

## Разговор в чате
Пришел ко мне недавно безопасник, который реально понимает в кубере и задал вопрос - а можно ли сделать так чтобы анонимный доступ был отключен, но лайв и реди пробы в kube-apiserver работали?
Сходу ответа я не знал, поэтому посмотрел в исходники кубера. А потом решил рассказать и вам.

## Дефолтный конфиг
Как вы конечно знаете по-умолчанию в [kube-apiserver](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/) разрешен анонимный доступ
```
--anonymous-auth     Default: true
```
> Enables anonymous requests to the secure port of the API server. Requests that are not rejected by another authentication method are treated as anonymous requests. Anonymous requests have a username of system:anonymous, and a group name of system:unauthenticated.

Ну и реально, когда поднимаем по дефолту кубер через kubeadm получаем анонимный доступ
```bash
# curl -k https://192.168.56.20:6443/livez ; echo
ok
```

```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Metadata",
  "auditID": "2caca66d-ccec-4b1f-8116-86741193cdce",
  "stage": "ResponseComplete",
  "requestURI": "/livez",
  "verb": "get",
  "user": {
    "username": "system:anonymous",
    "groups": [
      "system:unauthenticated"
    ]
  },
  "sourceIPs": [
    "192.168.56.20"
  ],
  "userAgent": "curl/7.88.1",
  "responseStatus": {
    "metadata": {},
    "code": 200
  },
  "requestReceivedTimestamp": "2025-04-13T14:41:33.630618Z",
  "stageTimestamp": "2025-04-13T14:41:33.630908Z",
  "annotations": {
    "authorization.k8s.io/decision": "allow",
    "authorization.k8s.io/reason": "RBAC: allowed by ClusterRoleBinding \"system:public-info-viewer\" of ClusterRole \"system:public-info-viewer\" to Group \"system:unauthenticated\""
  }
}
```

Но до списка апишек достучаться не можем, конечно.

```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Metadata",
  "auditID": "883357ee-8e25-4a06-8ebf-57df36a1b96e",
  "stage": "ResponseComplete",
  "requestURI": "/apis",
  "verb": "get",
  "user": {
    "username": "system:anonymous",
    "groups": [
      "system:unauthenticated"
    ]
  },
  "sourceIPs": [
    "192.168.56.20"
  ],
  "userAgent": "curl/7.88.1",
  "responseStatus": {
    "metadata": {},
    "status": "Failure",
    "message": "forbidden: User \"system:anonymous\" cannot get path \"/apis\"",
    "reason": "Forbidden",
    "details": {},
    "code": 403
  },
  "requestReceivedTimestamp": "2025-04-13T15:11:51.270418Z",
  "stageTimestamp": "2025-04-13T15:11:51.270948Z",
  "annotations": {
    "authorization.k8s.io/decision": "forbid",
    "authorization.k8s.io/reason": ""
  }
}
```

Но проблема все же есть. Кто его знает до чего сможет получить доступ взломщик если не правильно будет настроен рбак или будет какая-нибудь уязвимость в кубере.

## Исходники кубера

А исходники кубера меня навели на интересный [KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-auth/4633-anonymous-auth-configurable-endpoints/README.md) в котором как раз обсуждалось что же делать тем тревожным людям, которые хотят выключить анонимный доступ, но TCP проверка порта их не устраивает.

И в итоге было предложено [решение](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#anonymous-authenticator-configuration)

```bash
# cat /etc/kubernetes/auth-config.yaml
apiVersion: apiserver.config.k8s.io/v1beta1
kind: AuthenticationConfiguration
anonymous:
  enabled: true
  conditions:
  - path: /livez
  - path: /readyz
  - path: /healthz
```

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    ...
    - --anonymous-auth=false
    - --authentication-config=/etc/kubernetes/auth-config.yaml

    volumeMounts:
    ...
    - mountPath: /etc/kubernetes/auth-config.yaml
      name: auth
      readOnly: true

  volumes:
  ...
  - name: auth
    hostPath:
      path: /etc/kubernetes/auth-config.yaml
      type: File
```

```bash
curl -k https://192.168.56.20:6443/apis ; echo
```

После внедрения которого можно разрешать анонимный доступ до определенных урлов, а для всех остальных запрещать. 
Судя по [PR](https://github.com/kubernetes/enhancements/issues/4633) эта функциональность доступна в альфе в 1.31 и в бете 1.32.

```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Metadata",
  "auditID": "dfe74de7-15ff-4863-89e0-cf22f6779ea3",
  "stage": "ResponseComplete",
  "requestURI": "/livez",
  "verb": "get",
  "user": {
    "username": "system:anonymous",
    "groups": [
      "system:unauthenticated"
    ]
  },
  "sourceIPs": [
    "192.168.56.20"
  ],
  "userAgent": "curl/7.88.1",
  "responseStatus": {
    "metadata": {},
    "code": 200
  },
  "requestReceivedTimestamp": "2025-04-13T15:04:00.675020Z",
  "stageTimestamp": "2025-04-13T15:04:00.675433Z",
  "annotations": {
    "authorization.k8s.io/decision": "allow",
    "authorization.k8s.io/reason": "RBAC: allowed by ClusterRoleBinding \"system:public-info-viewer\" of ClusterRole \"system:public-info-viewer\" to Group \"system:unauthenticated\""
  }
}
```


```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Metadata",
  "auditID": "139b8cd8-c674-4f42-b5b0-34fae8c65901",
  "stage": "ResponseStarted",
  "requestURI": "/apis",
  "verb": "get",
  "user": {},
  "sourceIPs": [
    "192.168.56.20"
  ],
  "userAgent": "curl/7.88.1",
  "responseStatus": {
    "metadata": {},
    "status": "Failure",
    "message": "Unauthorized",
    "reason": "Unauthorized",
    "code": 401
  },
  "requestReceivedTimestamp": "2025-04-13T15:05:11.486607Z",
  "stageTimestamp": "2025-04-13T15:05:11.486838Z"
}
```