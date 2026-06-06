
# ConfigMap

## 是什么

ConfigMap 是存储**非敏感配置数据**的 Kubernetes 资源，将配置从容器镜像中解耦。修改配置不需要重新构建镜像。

存储形式是键值对，每个键可以是一个环境变量的值，也可以是一个文件的完整内容。

## 如何使用

ConfigMap 本身只定义数据，使用方式由 Pod 决定。同一个 ConfigMap 可以被不同 Pod 以不同方式引用。

### 方式一：注入为环境变量

```yaml
spec:
  containers:
  - envFrom:                        # 将整个 ConfigMap 注入为环境变量
    - configMapRef:
        name: app-config            # 每个键名 → 环境变量名，键值 → 环境变量值
```

也可以只注入单个键：
```yaml
spec:
  containers:
  - env:
    - name: DB_HOST                 # 自定义环境变量名
      valueFrom:
        configMapKeyRef:
          name: app-config          # ConfigMap 名称
          key: database_host        # 取出 database_host 键的值
```

环境变量在容器启动时注入一次，**ConfigMap 后续更新不会自动同步到已运行容器的环境变量中**，需要重启 Pod 才生效。

### 方式二：挂载为文件

```yaml
spec:
  containers:
  - volumeMounts:
    - name: config-vol
      mountPath: /etc/config        # 每个键变成一个文件
  volumes:
  - name: config-vol
    configMap:
      name: app-config              # ConfigMap 名称
```

容器内 `/etc/config/` 目录下，每个键成为一个文件，文件内容是键的值。

文件挂载的优势：kubelet 会定期同步 ConfigMap 更新到挂载的文件（有几十秒延迟），Pod 无需重启。

## ConfigMap 中的数据格式

```yaml
kind: ConfigMap
metadata:
  name: app-config
data:
  database_host: "mysql.svc.local"  # 单行键值对
  nginx.conf: |                     # | 是 YAML 多行字面量，保留换行
    server {
        listen 80;
        location / { proxy_pass http://backend; }
    }
```

## 典型用途

- 应用配置文件（nginx.conf、app.properties）
- 环境变量（数据库地址、服务名）
- 命令行参数

# Secret

## 是什么

Secret 与 ConfigMap 结构和用法几乎一样，区别在于：Secret 用于存储**敏感数据**（密码、token、TLS 私钥、证书）。

## 与 ConfigMap 的关键差异

| | ConfigMap | Secret |
|---|---|---|
| 存储 | 明文存在 etcd 中 | Base64 编码后存 etcd（非加密） |
| 挂载到容器 | 直接写入文件 | 写入 tmpfs（内存文件系统），不落磁盘 |
| kubectl 查看 | `kubectl describe` 直接显示 | `kubectl describe` 隐藏内容 |

**注意**：etcd 中的 Secret 仅是 Base64 编码，不是真正加密。生产环境需启用 etcd 静态加密（Encryption at Rest）来保护 Secret。

## 使用方式

与 ConfigMap 完全相同：

```yaml
# 注入为环境变量（Secret 专用字段）
spec:
  containers:
  - env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password

# 挂载为文件
spec:
  containers:
  - volumeMounts:
    - name: secret-vol
      mountPath: /etc/secrets
  volumes:
  - name: secret-vol
    secret:
      secretName: db-secret
```

## 典型用途

- 数据库密码、API 密钥
- TLS 证书与私钥（[Ingress](../服务与网络/Ingress.md) 的 HTTPS 终结）
- Docker Registry 拉取凭证（imagePullSecret）
- ServiceAccount 的认证 Token（详见 [ServiceAccount](../安全与权限/ServiceAccount.md)）
