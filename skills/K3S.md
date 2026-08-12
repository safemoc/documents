> `K3s` 是 `K8s` 的精简版,与`K8s` 的适用领域不同,更加适合与 轻量级的 边缘计算,而不是 特别繁重的集群工作.

# 安装
## 官方脚本

> 默认使用 `sqlite`  ,案例是指定了 `mysql` 

**server**
```sh
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn sh -s - server \
  --token='Spider-Man' \
  --datastore-endpoint='mysql://k3s:npiFNOX5UqyySu@tcp(192.168.1.50:3306)/k3s'
  
```

**Agent**
```sh
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  K3S_URL={{Server URL}} \
  K3S_TOKEN='Spider-Man' \
  sh -
```


# 配置加速 代理

> 加速应该在 `server` 与 `agent`中都配置.

```sh
sudo mkdir -p /etc/rancher/k3s

sudo tee /etc/rancher/k3s/registries.yaml >/dev/null <<'EOF'
mirrors:
  docker.io:
    endpoint:
      - "https://docker.m.daocloud.io"
      - "https://docker.1ms.run"
EOF
```

- [[Linux#`tee`的作用|`tee`的使用]]



# 安装Web页面
## 安装 Rancher


# 运行逻辑
**硬件**
分为 server 和 agent ,server作为操控机 ,调配 agent
## 执行逻辑
### `NameSpace` 空间
 逻辑空间划分

### 查看
```sh
kubectl get ns                 # 列出所有命名空间（ns = namespaces）
kubectl get namespaces         # 同上，完整写法
kubectl get ns -o wide         # 多一点信息
kubectl describe ns default    # 看某个 ns 详情（配额、标签等）
```

### 操作
```sh
kubectl create ns demo         # 创建
kubectl delete ns demo         # 删除（会删掉里面的资源，慎用）
```

**也可以用 `YAML`**
```sh
kubectl apply -f ns.yaml
```

`ns.yaml` 示例：

``` yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
```

### 在指定的命名空间中操作
几乎所有资源命令都带 `-n`：
``` sh
kubectl get pods -n demo
kubectl get all -n demo
kubectl get deploy,svc,cm,secret -n demo

kubectl apply -f app.yaml -n demo
kubectl delete pod <pod名> -n demo
kubectl logs <pod名> -n demo
kubectl exec -it <pod名> -n demo -- sh
```

看所有命名空间：
```sh
kubectl get pods -A # -A = --all-namespaces

kubectl get pods --all-namespaces
```





# `agent` 节点使用 `kubectl`
在 `/etc/rancher/k3s/k3s.yaml` 中写入
server端中的 同目录文件中的内容且调整`IP` 为 server端

``` yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data:  .............
    server: https://{Server IP地址}:6443
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
users:
- name: default
  user:
    client-certificate-data:..........
    client-key-data:..........

```






# 部署 UI 页面并配置 管理员账户



**Dashboard 本身没有“账号密码登录”模式。**

Kubernetes Dashboard 官方采用的是 **Kubernetes ServiceAccount + Token 认证**。

也就是说：

```
浏览器
  |
  | 输入 Token
  |
Dashboard
  |
  |
Kubernetes API Server
  |
ServiceAccount 权限
```

所以你不能通过 Deployment YAML 设置账号密码。

---

不过你可以重新部署，并且自己定义一个管理员名称，例如：

```
账号名：admin
密码：xxxx
```

只是这个“密码”实际上是 Token。

结构如下：

```
Namespace
 └── kubernetes-dashboard

Deployment
 └── kubernetes-dashboard

ServiceAccount
 └── admin-user   <--- 你的账号名

ClusterRoleBinding
 └── admin-user -> cluster-admin

Secret(Token)
 └── 登录密码
```

---

## 重新部署流程


### 1. 安装 Dashboard Deployment

例如官方 YAML：

```bash
sudo kubectl apply -f \
https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

这会创建：

```
Deployment
Service
ConfigMap
ServiceAccount
```

---

### 2. 创建你的管理员账号

创建：

```bash
sudo kubectl create serviceaccount admin \
-n kubernetes-dashboard
```

这里：

```
admin
```

就是你的用户名。

---

### 3. 给 admin 权限

创建：

```bash
sudo kubectl create clusterrolebinding admin \
--clusterrole=cluster-admin \
--serviceaccount=kubernetes-dashboard:admin
```

意思：

```
admin
       |
       |
       v

cluster-admin

拥有整个集群权限
```

---

### 4. 生成登录密码(Token)

K3s / Kubernetes 新版本：

```bash
sudo kubectl create token admin \
-n kubernetes-dashboard
```

输出：

```
eyJhbGciOiJSUzI1NiIsInR5cCI6...
```

这个就是：

```
用户名：admin

密码:
eyJhbGciOi...
```

---

## 如果你想固定 Token

现在：

```bash
kubectl create token
```

生成的是临时 Token。

如果你希望类似：

```
admin
123456
```

这种固定密码，需要创建 Secret：

例如：

`admin-token.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: admin-token
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/service-account.name: admin
type: kubernetes.io/service-account-token
```

应用：

```bash
kubectl apply -f admin-token.yaml
```

查看：

```bash
kubectl describe secret admin-token \
-n kubernetes-dashboard
```

里面：

```
token:
eyJhbGc...
```

就是固定 Token。

---

## 访问方式

你现在集群：

```
safe
192.168.110.230
(control-plane)
```

我建议直接 NodePort：

修改 Service：

```bash
kubectl edit svc kubernetes-dashboard \
-n kubernetes-dashboard
```

改：

```yaml
spec:
  type: NodePort
```

然后：

```bash
kubectl get svc \
-n kubernetes-dashboard
```

例如：

```
443:30443/TCP
```

访问：

```
https://192.168.110.230:30443
```

登录：

```
Token:
eyJhbGc...
```
----
Kubernetes 的理念就是：

**Deployment 管应用生命周期，ServiceAccount 管身份，RBAC 管权限。**

账号和密码不属于 Deployment。你现在正好碰到了 Kubernetes 和传统 Web 应用最大的区别。





1 自重启 {node k3s-agent 可以自启动了,pod 需要验证{可以使用 doplyment 创建pod,设定参数可以直接多节点重启 pod 或者恢复pod中容器进程}}
2  管理容器 {现在是 管理pod ,pod中是多个 容器}
3  根据资源使用情况 动态扩容 {暂未实现}
4 web页面管理 {k8s 的 dashboard UI}
5 平滑过渡至 K8s


