<!--
  Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->

# StreamPark PR 改动总结报告

> **作者：** shangeyao  
> **目标分支：** 均为 `dev`  
> **文档生成：** 2026-06-27  
> **PR 状态：** 以下 PR 均为 OPEN（未合并），Fork PR 的 CI 通常需 upstream maintainer 审批 workflow（`action_required`）。

---

## 一、总览

| 类别 | PR 数量 | 编号 |
|------|---------|------|
| 大特性（Flink 2.x / Spark / 文档） | 4 | #4368, #4372, #4373, #4383 |
| CI / 质量门禁 | 2 | #4377, #4401 |
| 安全 / 权限 | 4 | #4381, #4389, #4393, #4395 |
| 可靠性 / 并发 | 3 | #4385, #4387, #4397 |
| Common / 引擎环境 | 4 | #4379, #4383, #4399, #4405 |
| Console 性能 / 运维 | 2 | #4391, #4397 |
| K8s | 1 | #4403 |
| 前端 | 1 | #4407 |
| **合计** | **21** | （#4373 与 #4383 内容重叠，见「重叠说明」） |

---

## 二、大特性与基础设施

### [#4368](https://github.com/apache/streampark/pull/4368) — Flink 2.x 支持

| 项 | 内容 |
|----|------|
| **分支** | `feature/flink-2.x-support` |
| **主题** | 在 Console 仍用 JDK 8 的前提下支持 Flink 2.0 / 2.1 / 2.2 |

**主要改动：**

- 新增 Flink 2.x shims 模块（`streampark-flink-shims_flink-2.{0,1,2}`）
- `FlinkVersion`：jar 名优先解析版本，fallback `flink --version`
- 新增 `FlinkEnvUtils`：从 `flink-env.sh` 读取 Flink 侧 `JAVA_HOME`
- `FlinkClientTrait`：提交任务时传递 `env.java.home`
- `ClassLoaderUtils`：JDK 9+ `ucp` 反射兼容（dev 上另见 #4399 独立补丁）
- 测试、文档（`FLINK_JDK_GUIDE`）、UI 文案

**意义：** 改动面最大的战略 PR，与其余小步 fix PR 相对独立。

---

### [#4372](https://github.com/apache/streampark/pull/4372) — AGENTS.md

| 项 | 内容 |
|----|------|
| **分支** | `feature/add-agents-md-clean` |
| **主题** | 为 AI Agent / 贡献者提供项目规范文档 |

**主要改动：**

- 新增根目录 `AGENTS.md`：模块边界、设计模式、编码规范、构建命令、PR 约定等
- 替代已关闭的 #4371（更精简的版本）

---

### [#4373](https://github.com/apache/streampark/pull/4373) / [#4383](https://github.com/apache/streampark/pull/4383) — Spark 版本解析（jar-first）

| 项 | 内容 |
|----|------|
| **分支** | `feature/spark-4-version-parse` / `fix/spark-version-jar-first` |
| **主题** | Console JDK 8 下注册 Spark 4.x 环境，无需升级 Console JDK |

**主要改动：**

- 新增 `SparkEnvUtils`：从 `spark-env.sh` 解析 Spark `JAVA_HOME`
- `SparkVersion`：优先从 `spark-core_*.jar` / `RELEASE` 读版本，`spark-submit --version` 作 fallback
- `YarnClient`：`setJavaHome` 传递 Spark JDK
- 测试 + `SPARK_JDK_GUIDE` 文档 + UI 提示

**说明：** #4383 是从 #4373 的 commit cherry-pick 到 dev 的等价实现；**合并时二选一即可**。

---

## 三、CI 与质量门禁

### [#4377](https://github.com/apache/streampark/pull/4377) — E2E 失败应阻断 PR

| 项 | 内容 |
|----|------|
| **文件** | `.github/workflows/e2e.yml` |
| **改动** | E2E 失败时 `exit 0` → `exit 1` |
| **意义** | 避免测试挂了 PR 仍能 merge，立刻提升 CI 门禁有效性 |

---

### [#4401](https://github.com/apache/streampark/pull/4401) — Backend CI 提速

| 项 | 内容 |
|----|------|
| **文件** | `.github/workflows/backend.yml` |

**改动：**

- 去掉 Backend workflow 内重复的 license / checkstyle（Unit-Test workflow 已覆盖）
- **JDK 8**：`-Pshaded`，跳过 webapp 模块
- **JDK 11**：完整 `-Pshaded,webapp,dist` + 依赖 license 检查

**意义：** 缩短纯后端改动的 CI 时间，减少重复检查。

---

## 四、安全与权限（RBAC）

### [#4381](https://github.com/apache/streampark/pull/4381) — Cluster / Variable RBAC

| 项 | 内容 |
|----|------|
| **文件** | `FlinkClusterController.java`、`VariableController.java` |
| **改动** | 为集群分页/列表/启停/删除、变量列表/校验等接口补 `@RequiresPermissions` |

---

### [#4389](https://github.com/apache/streampark/pull/4389) — Actuator / Metrics / H2 鉴权

| 项 | 内容 |
|----|------|
| **文件** | `ShiroConfig.java`、`SecurityStartupRunner.java`（新） |

**改动：**

- `/actuator/health`、`/actuator/info` 仍匿名（探针可用）
- 其余 `/actuator/**`、`/metrics/**`、`/h2-console/**` 需 JWT
- H2 嵌入式库启动时打 WARN，提示修改默认密码

---

### [#4393](https://github.com/apache/streampark/pull/4393) — SettingController 权限补全

| 项 | 内容 |
|----|------|
| **文件** | `SettingController.java` |
| **改动** | `get`、`checkHadoop` 补 `@RequiresPermissions("setting:view")` |

---

### [#4395](https://github.com/apache/streampark/pull/4395) — Spark 团队级权限

| 项 | 内容 |
|----|------|
| **文件** | `PermissionAspect.java`、`SparkApplicationController.java` |

**改动：**

- `PermissionAspect` 在 Flink 查不到 app 时继续查 Spark 应用
- Spark 应用 CRUD/启停/备份/日志等接口补 `@Permission`，与 Flink 团队隔离一致

**意义：** 此前 Spark 仅有 Shiro 角色权限，缺少团队级隔离。

---

## 五、可靠性与并发

### [#4385](https://github.com/apache/streampark/pull/4385) — 启停并发锁

| 项 | 内容 |
|----|------|
| **文件** | `FlinkApplicationActionServiceImpl.java`、`SparkApplicationActionServiceImpl.java` |

**改动：**

- 内存级 `pendingStarts` / `pendingCancels` 防重复提交
- 条件 `lambdaUpdate` 原子将状态改为 STARTING / CANCELLING
- 异步完成或失败时清理 pending 标记

---

### [#4387](https://github.com/apache/streampark/pull/4387) — Watcher 单应用去重

| 项 | 内容 |
|----|------|
| **文件** | `FlinkAppHttpWatcher.java`、`SparkAppHttpWatcher.java` |
| **改动** | 每 app 用 `AtomicBoolean` single-flight，避免重叠 REST 轮询入队 |

---

### [#4397](https://github.com/apache/streampark/pull/4397) — Watcher 降频 + 去重（增强版）

| 项 | 内容 |
|----|------|
| **文件** | 同上 + `FlinkClusterWatcher.java` |

**改动：**

- **稳态 5s** 轮询 + **启停期间 1s** 快路径（双 `@Scheduled`）
- 保留 single-flight 去重
- Cluster watcher 调度对齐 **30s** 间隔

**说明：** 与 #4387 有重叠；建议 **只保留 #4397，关闭 #4387**。

---

## 六、Common / 引擎与 Hadoop

### [#4379](https://github.com/apache/streampark/pull/4379) — Maven 本地仓库目录初始化 bug

| 项 | 内容 |
|----|------|
| **文件** | `EnvInitializer.java` |
| **改动** | `createMvnLocalRepoDir()` 从「目录存在才 mkdir」改为 `mkdirsIfNotExists` |

---

### [#4399](https://github.com/apache/streampark/pull/4399) — ClassLoaderUtils JDK 9+ 修复

| 项 | 内容 |
|----|------|
| **文件** | `ClassLoaderUtils.scala` |

**改动：**

- 依次尝试 TCCL 和 System ClassLoader
- 沿类加载器层次查找 `ucp` 字段（修复 JDK 17 下 `NoSuchFieldException: ucp`）
- 解决 Hadoop conf 目录动态加载失败

---

### [#4405](https://github.com/apache/streampark/pull/4405) — HadoopUtils 客户端同步

| 项 | 内容 |
|----|------|
| **文件** | `HadoopUtils.scala` |

**改动：**

- 新增 `hadoopLock`，统一保护 UGI、HDFS、YarnClient、配置访问
- Kerberos 票据刷新在锁内 `closeHadoop()`，避免 watcher 与 submit 并发踩客户端

---

## 七、Console 性能与运维

### [#4391](https://github.com/apache/streampark/pull/4391) — 备份清理批量删除

| 项 | 内容 |
|----|------|
| **文件** | `ApplicationBackUpCleanTask.java` |
| **改动** | 由 per-app N+1 查询 + 逐条 `removeById` 改为一次查全量 → 内存分组 → `removeByIds` 批量删 |

---

## 八、Kubernetes

### [#4403](https://github.com/apache/streampark/pull/4403) — MetricCache TTL

| 项 | 内容 |
|----|------|
| **文件** | `FlinkK8sWatchController.scala`（`MetricCache`） |
| **改动** | Caffeine 增加 `expireAfterWrite(6h)` + `maximumSize(10_000)`，防止 Job 结束后指标长期残留 |

---

## 九、前端（Web UI）

### [#4407](https://github.com/apache/streampark/pull/4407) — Monaco 拆包 + 自适应轮询 + axiosCancel

| 项 | 内容 |
|----|------|
| **文件** | `vite.config.ts`、`pollingSetting.ts`（新）、Flink/Spark 应用列表 `View.vue`、`axiosCancel.ts` |

**改动：**

- `monaco-editor` 独立 Rollup chunk，减小首屏体积
- Flink/Spark 应用列表：**空闲 5s / 操作中 2s** 自适应轮询
- axios 重复请求 cancel key 加入 query params，避免不同分页/筛选互相 cancel

---

## 十、建议合并顺序

按依赖关系和风险，建议 maintainer 按以下顺序 review / merge：

```
1. 质量门禁（低风险、立刻受益）
   #4377 → #4379 → #4401

2. 安全（独立、应优先）
   #4381 → #4389 → #4393 → #4395

3. Common / 运行时稳定性
   #4399 → #4405 → #4383（或 #4373 二选一）

4. Console 可靠性 / 性能
   #4385 → #4397（关 #4387）→ #4391

5. K8s + 前端
   #4403 → #4407

6. 大特性（单独评审、周期最长）
   #4372 → #4368
```

---

## 十一、重叠与冲突说明

| 情况 | 建议 |
|------|------|
| **#4373 vs #4383** | 内容等价，合并其一 |
| **#4387 vs #4397** | #4397 包含并增强 #4387，关 #4387、合 #4397 |
| **#4368 vs #4399** | ClassLoaderUtils 两处都有修；#4368 合入后可能需 rebase #4399 |
| **#4389 与探针** | health/info 仍匿名；若 K8s 探针走其他 actuator 路径需再确认 |
| **Fork CI** | 全部 PR 需 maintainer 点 approve 才能跑完整 CI |

---

## 十二、PR 完整索引

| PR | 标题 | 分支 |
|----|------|------|
| [#4368](https://github.com/apache/streampark/pull/4368) | [Feature] Support Apache Flink 2.x (2.0, 2.1, 2.2) | `feature/flink-2.x-support` |
| [#4372](https://github.com/apache/streampark/pull/4372) | [Build] Add AGENTS.md for AI agent coding conventions | `feature/add-agents-md-clean` |
| [#4373](https://github.com/apache/streampark/pull/4373) | [Common][Spark] Support Spark 4.x env registration without upgrading service JDK | `feature/spark-4-version-parse` |
| [#4377](https://github.com/apache/streampark/pull/4377) | [CI] Fail workflow when E2E tests fail | `fix/ci-e2e-result-exit-code` |
| [#4379](https://github.com/apache/streampark/pull/4379) | [Console] Fix Maven local repo directory initialization on startup | `fix/env-initializer-maven-repo-dir` |
| [#4381](https://github.com/apache/streampark/pull/4381) | [Console] Add missing RBAC checks on cluster and variable endpoints | `fix/cluster-variable-rbac` |
| [#4383](https://github.com/apache/streampark/pull/4383) | [Common][Spark] Parse Spark version from JAR/RELEASE before spark-submit fallback | `fix/spark-version-jar-first` |
| [#4385](https://github.com/apache/streampark/pull/4385) | [Console] Prevent concurrent duplicate start/cancel for Flink and Spark apps | `fix/app-start-stop-concurrency-lock` |
| [#4387](https://github.com/apache/streampark/pull/4387) | [Console] Deduplicate Flink/Spark HTTP watcher polling per application | `fix/watcher-single-flight-dedup` |
| [#4389](https://github.com/apache/streampark/pull/4389) | [Console] Require authentication for actuator/metrics and warn on H2 default credentials | `fix/actuator-security-h2-warn` |
| [#4391](https://github.com/apache/streampark/pull/4391) | [Console] Batch delete expired application backups in cleanup task | `fix/backup-clean-batch-delete` |
| [#4393](https://github.com/apache/streampark/pull/4393) | [Console] Add missing RBAC on SettingController get and checkHadoop | `fix/setting-controller-rbac` |
| [#4395](https://github.com/apache/streampark/pull/4395) | [Console] Add Spark application team-level permission checks | `fix/spark-app-team-permission` |
| [#4397](https://github.com/apache/streampark/pull/4397) | [Console] Reduce watcher poll frequency and deduplicate in-flight polls | `fix/watcher-poll-interval-dedup` |
| [#4399](https://github.com/apache/streampark/pull/4399) | [Common] Fix ClassLoaderUtils dynamic classpath on JDK 9+ | `fix/classloader-utils-jdk17-ucp` |
| [#4401](https://github.com/apache/streampark/pull/4401) | [CI] Speed up backend workflow and avoid duplicate style checks | `fix/ci-backend-build-speed` |
| [#4403](https://github.com/apache/streampark/pull/4403) | [K8s] Add TTL and size limit to Flink K8s MetricCache | `fix/k8s-metric-cache-ttl` |
| [#4405](https://github.com/apache/streampark/pull/4405) | [Common] Synchronize HadoopUtils YarnClient and HDFS access | `fix/hadoop-utils-yarn-client-lock` |
| [#4407](https://github.com/apache/streampark/pull/4407) | [Web UI] Split Monaco chunk, reduce idle list polling, fix axios cancel key | `fix/webapp-polling-and-monaco-chunk` |

---

## 十三、整体评价

本次 PR 群覆盖 StreamPark **dev 主线**上扫描出的主要短板：

- **质量：** E2E 门禁、EnvInitializer bug、CI 提速
- **安全：** RBAC 补全（Cluster / Variable / Setting / Spark 团队隔离）、Actuator 鉴权
- **稳定性：** 启停并发锁、Watcher 优化、Hadoop 客户端同步
- **性能：** 备份清理 N+1、前端轮询与拆包、K8s 缓存 TTL
- **前瞻能力：** Flink 2.x、Spark 4.x jar-first 环境注册

共 **21 个 PR**（去重后约 **19 个有效独立改动**），均为小步、可独立 review 的拆分方式。
