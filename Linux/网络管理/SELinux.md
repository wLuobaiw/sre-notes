
# 是什么

SELinux（Security-Enhanced Linux）是 Linux 内核的**强制访问控制**（MAC）系统。它给每个进程和文件打上安全标签，由内核策略控制谁能访问谁——即使是 root 进程也受策略约束。

# 为什么需要 SELinux

传统 Linux 权限是**自主访问控制**（DAC）——用户拥有自己的文件，可以随意授权。一旦进程被攻破（如通过漏洞拿到 root 权限），攻击者就能访问任意文件。

SELinux 的**强制访问控制**（MAC）是第二道防线：即使进程以 root 运行，只要策略不允许，它就无法读取、写入、执行特定文件。这就是"最小权限原则"。

# 三种模式

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| Enforcing | 强制执行策略，拒绝违规操作并记录日志 | 生产环境 |
| Permissive | 不拒绝违规操作，但记录警告日志 | 调试策略、排障 |
| Disabled | 完全关闭 SELinux | **不推荐** |

## 查看与切换

```shell
# 查看当前模式
getenforce

# 查看详细状态
sestatus

# 临时切换（重启失效）
setenforce 0    # 切换为 Permissive
setenforce 1    # 切换为 Enforcing

# 永久配置：编辑 /etc/selinux/config
SELINUX=enforcing
```

# 安全上下文（Context）

SELinux 的核心是**安全上下文**——每个主体（进程）和客体（文件/端口/设备）都携带一个标签：

```
system_u:object_r:httpd_sys_content_t:s0
  用户   :  角色  :        类型        : 级别
```

- **用户（user）**：SELinux 用户身份，与 Linux 用户不同
- **角色（role）**：定义用户可以进入的域
- **类型（type）**：**核心——**定义访问规则。策略主要基于类型
- **级别（level）**：MLS/MCS 安全级别（多用于机密性场景）

## 查看上下文

```shell
# 文件上下文
ls -Z /var/www/html/index.html

# 进程上下文
ps -Z

# 当前用户上下文
id -Z

# 端口上下文
semanage port -l | grep http
```

# 常见操作

## 修改文件上下文

```shell
# 临时修改文件上下文（仅本次）
chcon -t httpd_sys_content_t /web/index.html

# 递归修改
chcon -R -t httpd_sys_content_t /web/

# 恢复为默认上下文（推荐——使用策略中定义的规则）
restorecon -Rv /web/index.html
```

`chcon` 的修改在文件系统重标记时会丢失，**推荐使用 `semanage` 加 `restorecon`**：

```shell
# 添加默认上下文规则（持久化）
semanage fcontext -a -t httpd_sys_content_t '/web(/.*)?'

# 应用规则
restorecon -Rv /web
```

## 管理布尔值（策略开关）

```shell
# 查看所有布尔值
getsebool -a

# 查看特定布尔值
getsebool httpd_enable_cgi

# 临时设置
setsebool httpd_enable_cgi on

# 永久设置
setsebool -P httpd_enable_cgi on
```

布尔值是不需要写策略就能调整的安全开关。如 `httpd_can_network_connect` 控制 Apache 是否能发起外部网络连接。

# 故障排查

## 判断是不是 SELinux 导致的问题

```shell
# 1. 临时设为 Permissive 验证
setenforce 0
# 重试操作——如果问题消失，就是 SELinux 导致的
# 排查完恢复：
setenforce 1
```

## 查看审计日志

```shell
# 查看 SELinux 拒绝记录
ausearch -m avc -ts recent

# 或直接查看审计日志
grep "denied" /var/log/audit/audit.log
```

## 生成允许策略

```shell
# 从审计日志生成策略模块（自动分析需要哪些权限）
grep "denied" /var/log/audit/audit.log | audit2allow -M my_fix

# 查看生成的策略（需要手动审查）
cat my_fix.te

# 安装策略（审核后确认安全再执行）
semodule -i my_fix.pp
```

> **重要**：不要盲目允许所有拒绝——先理解被拒绝的操作是否合理。也可能是文件上下文错误，先 `restorecon` 试试。

## 常见典型问题

| 现象 | 常见原因 | 修复 |
|------|---------|------|
| Nginx 无法访问自定义目录 | 目录没有 `httpd_sys_content_t` 上下文 | `semanage fcontext` + `restorecon` |
| 服务无法绑定非标准端口 | 端口没有对应类型 | `semanage port -a -t http_port_t -p tcp 8080` |
| 文件上传后 403 | 上传文件上下文不对 | `restorecon -Rv /upload/dir` |

# 容器/K8s 中的 SELinux

- Docker/Podman 使用 SELinux 标签（如 `container_file_t`）隔离容器间文件访问
- Kubernetes 中，Pod 的 SELinux 选项（`seLinuxOptions`）在 PodSecurityContext 中配置
- 挂载 hostPath 卷时，目录需要 `container_file_t` 上下文或使用 `:Z` / `:z` 重新标记

# 注意

- **不要轻易关闭 SELinux**——它是深度防御的重要一环
- 排障时先设为 Permissive 验证是 SELinux 问题，但**不要永久设为 Permissive**
- Rocky/Alma/RHEL 9 默认 Enforcing 模式
- audit2allow 生成的策略需要人工审查，它只是机械翻译拒绝日志

> 返回 [网络管理基础](./网络管理基础.md)
