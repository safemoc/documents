
# `tee`的作用
使用实例:
```sh
sudo tee /etc/rancher/k3s/registries.yaml >/dev/null <<'EOF'
mirrors:
  docker.io:
    endpoint:
      - "https://docker.m.daocloud.io"
      - "https://docker.1ms.run"
EOF
```
---

普通情况下：

```
echo hello > file.txt
```

可以写文件。

但是：

```
sudo echo hello > file.txt
```

经常失败。

原因：

`>` 重定向是在当前用户权限下执行的，不受 `sudo` 控制。

例如：

```
sudo echo test > /etc/test.txt
```

实际：

```
echo 使用root权限
>
写文件使用普通用户权限
```

所以可能：

```
Permission denied
```

---

`tee` 可以解决：

```
echo hello | sudo tee file.txt
```

流程：

```
echo
 |
 v
tee (root权限)
 |
 v
file.txt
```

所以：

```
sudo tee /etc/rancher/k3s/registries.yaml
```

就是：
用 root 权限写入文件。

---
` >/dev/null` 是将 tee的返回值 丢弃
