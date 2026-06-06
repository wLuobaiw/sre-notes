
# Job

## 是什么

Job 管理**一次性任务**：创建一个或多个 Pod，等待它们成功退出（exit code 0），然后结束。与 Deployment 的本质区别在于目标 —— Deployment 追求"永远运行"，Job 追求"成功退出"。

## Job vs Deployment 对比

| | Deployment | Job |
|---|---|---|
| 目标 | 持续运行 | 成功完成 |
| Pod 退出后 | 重启，保持 Running | 标记完成，不重启 |
| 副本含义 | 同时运行 N 个 | 总共成功 N 次 |
| 失败处理 | 无限重启 | 重试指定次数后标记失败 |

## 关键配置

**并行度控制**：
- `completions: 10` —— 总共需要成功完成 10 个 Pod
- `parallelism: 3` —— 但最多同时运行 3 个

典型场景：处理 1000 条数据，开 10 个 worker 并行处理，每个处理 100 条。

**失败重试**：
- `backoffLimit: 4` —— 单个 Pod 失败后最多重试 4 次
- 超过重试次数后，整个 Job 标记为 Failed
- Deployment 是无限重启，Job 有上限

**TTL 自动清理**：
- `ttlSecondsAfterFinished: 3600` —— 完成 1 小时后自动删除 Job 对象及完成的 Pod
- 清理的是 Kubernetes API 中保留的**记录**（Job 资源和已退出的 Pod 资源），不是运行中的进程
- 保留期间可以查看日志、确认执行结果
- 设为 0 则完成即清理

**Pod 重启策略**：
- Job 的 Pod 必须使用 `restartPolicy: Never` 或 `OnFailure`（restartPolicy 的完整说明详见 [Pod](Pod.md)）
- 不能使用 `Always`（否则永远不会"完成"）

## 典型场景

- 数据库迁移
- 批量数据处理
- 一次性初始化任务（如创建索引、导入种子数据）

# CronJob

## 是什么

CronJob 在指定的时间点**自动创建 Job**。语法与 Linux crontab 完全一致（5 位：分 时 日 月 周）。

## 与 Linux crontab 的区别

Linux crontab 在单机上执行命令。CronJob 在集群中创建一个 Job，Job 创建 Pod，Pod 由调度器决定在哪个节点运行。任务在集群层面执行，不绑定任何特定节点。

## 并发策略

`concurrencyPolicy` 控制上一个任务还没跑完时，下一次定时触发如何处理：

- **Allow**（默认）：不管，新任务照常启动
- **Forbid**：跳过本次触发
- **Replace**：杀掉正在跑的任务，启动新的

## 与上层调度平台的定位

CronJob 是基础设施层的能力，极其简单。它不具备：
- 任务分片与分片路由
- 任务依赖编排
- 分布式锁与故障转移
- 执行器集群管理

这些能力由上层调度平台（xxl-job、Airflow、Argo Workflows）提供。这些平台本身通常部署在 Kubernetes 中，底层任务的 Pod 运行依赖的可能仍然是 Job。类比：K8s Job 像是操作系统级的 `crontab` + 进程，而 xxl-job 是用 crontab 能力搭建出来的分布式调度平台。

## 典型场景

- 定时数据库备份
- 每小时生成报表
- 定时清理过期数据
