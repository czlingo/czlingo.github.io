---
title: "RBAC"

categories: [
    "kubernetes"
]
---

在 Kubernetes 中，RBAC 是负责完成授权工作的机制(基于角色的访问控制(Role-Based Access Control))

在 RBAC 体系中有三个核心的概念：

- Role： 角色， 它其实是一组规则，定义了一组对 Kubernetes API 对象的操作权限。
- Subject： 被作用者， 既可以是“人”， 也可以是“机器”， 也可以是你在 Kubernetes 里定义的 “用户”。
- RoleBinding： 定义了 “被作用者” 和 “角色” 的绑定关系。

### Role

Role 本身是一个 Kubernetes 中的 API 对象：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
    namespace: mynamespace
    name: pod-reader
rules: 
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["get", "watch", ""list]
```

这个例子中描述了一个角色，它作用于 mynamespace 的 Namespace 中， 允许 “被作用者” 对 mynamespace 中的 Pod 对象， 进行 GET、 WATCH 和 LIST 操作。

### RoleBinding

在 Role 中提到，当定义了一个 Role 对象之后，它允许 “被作用者” 通过 Role 中所定义的规则去操作 Kubernetes 中的资源， 其中 “被作用者” 则是通过 RoleBinding 这个 Kubernetes 的 API 对象所实现。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
    namespace: mynamespace
    name: reader
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: example-user
roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: pod-reader
```

在这个 RoleBinding 对象中定义了一个 subjects 字段，即上文中描述的 “被作用者”。 其类型为 User，即 Kubernetes 里的用户，这个用户的名字是 example-user。

这里的 User 只是授权系统里的一个逻辑概念，在 Kubernetes 中并没有一个叫做 “User” 的 API 对象。在大多数时候，都是直接使用 Kubernetes 里的 “内置用户”： 即 ServiceAccount。

RoleRef 字段则是指向前面所定义的 Role 对象，从而定义了 “被作用者” 与 “角色” 之间的绑定关系。

### ServiceAccount

一个最简单的 ServiceAccount 对象只需要 Name 和 Namespace 这两个最基本的字段。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
    namespace: mynamespace
    name: example-sa
```

然后，通过 RoleBinding 的 YAML 文件，来为这个 ServiceAccount 分配权限：

```yaml
apiVersion： rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
    name: example-rolebinding
    namespace: mynamespace
subjects:
- kind: ServiceAccount
  name: example-sa
  namespace: mynamespace
roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: example-role
```

这里的 RoleBinding 对象里，subjects 字段的类型(kind)，不再是一个User，而是一个名叫 example-sa 的 ServiceAccount。

当创建 ServiceAccount 时， Kubernetes 会自动为 SA 创建一个 Secret 对象，这个 Secret，就是用来跟 APIServer 进行交互的授权文件 Token。

这时，用户的 Pod，就可以申明使用这个 ServiceAccount：

```yaml
apiVersion: v1
kind: Pod
metadata:
    namespace: mynamespace
    name: sa-token-test
spec:
    containers:
    - name: nginx
      image: nginx:1.7.9
    serviceAccountName: example-sa
```

当 Pod 运行起来之后， 该 ServiceAccount 的 token，就会被 Kubernetes 挂在到容器的 `/var/run/secrets/kubernetes.io/serviceaccount` 目录下

如果一个 Pod 没有声明 ServiceAccount， Kubernetes 会自动在它的 Namespace 下创建一个名叫 default 的默认 ServiceAccount， 然后分配给这个 Pod。

但在这种情况下， 默认的 ServiceAccount 并没有关联任何 Role。也就是说，此时它有访问 APIServer 的绝大多数权限。所以在生产环境下建议为所有 Namespace 下的默认 ServiceAccount 绑定一个只读权限 Role。

### Group

除了 “User”， Kubernetes 还拥有 “Group” (用户组) 的概念。实际上一个 ServiceAccount， 在 Kubernetes 里对应的 “用户” 名字为：

```
system:serviceaccount:<ServiceAccount Name>
```

而它对应的内置 “用户组” 的名字为：

```
system:serviceaccounts:<Namespace>
```

例如，在 RoleBinding 里定义如下的 subjects：

```ymal
subjects:
- kind: Group
  apiGroup: rbac.authorization.k8s.io
  name: system:serviceaccounts:mynamespace
```

这就意味着这个 Role 的权限规则，作用于 mynamespace 里所有的 SA。

而下面这个例子：

```yaml
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
```

意味着这个 Role 的权限规则，作用于整个系统里的所有 ServiceAccount。