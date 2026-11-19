# 配置中心系统（基础版）后端技术方案设计

> 本文档为「后端技术方案设计」，承接 `requirement.md`（需求定型：D1–D5、FR-1~10、AC-1~11）与 `design.md`（产品方案：M1–M7、BR、RBAC 矩阵），聚焦后端实现：技术选型、系统架构、物理数据模型、接口契约、关键流程、安全与测试。
> **合规声明**：文中所有「验收标准 / 判定条件」均为**待后续实现与测试阶段执行的验证条件**，不代表已实现或已通过；所有未给出的参数一律标注为**定稿决策或后续版本事项**；**不包含任何真实凭据或密钥**。
> 本文档为纯文档产物，不改动任何业务代码。

---

## 0. 文档元信息

| 项目 | 内容 |
| --- | --- |
| 需求名称 | 配置中心系统（基础版） |
| 平台工作项 | issue-1795061266659410559 |
| 上游产物 | `requirement.md`、`design.md`（产品方案设计） |
| 本文档范围 | 后端服务 + 物理数据模型 + API 契约 + SDK 消费契约（不含控制台 M7 的详细前端设计） |
| 文档状态 | 后端方案设计（已按市面通用方式定稿待评审） |
| 仓库上下文 | loopforge-demo（github:jkstg01/loopforge-demo） |

---

## 1. 设计目标与边界

### 1.1 设计目标

把产品方案中的「可管、可发、可追、可退、可隔离、可审计、可控权」闭环，落到可实现的后端架构与接口上：

1. **草稿 / 已发布强隔离**：编辑只触达草稿表，消费端只读已发布快照（BR-04）。
2. **发布原子化**：以 `(namespace, env, group)` 为原子单位，快照全有或全无（BR-05）。
3. **版本不可变**：版本只增不改不删；回滚 = 复制历史快照为新版本（BR-06/07）。
4. **环境硬隔离**：所有读写强制携带 `env`，服务端作为过滤与校验条件（BR-08）。
5. **安全默认**：服务端鉴权、敏感值字段级加密 + 全链路脱敏、审计 append-only（BR-09/11/13）。
6. **客户端最终一致**：长轮询 + 版本号比对，REST 兜底（BR-14、D2）。

### 1.2 边界（本版不做）

继承 `requirement.md` 3.2 节：文件型配置、结构化 JSON/YAML 编辑与校验、灰度/标签分发、模板与继承、跨环境晋升、服务端主动推送、多集群/多地域、发布审批工作流、完整密钥自动轮换、多语言 SDK。

---

## 2. 技术选型

### 2.1 选型总览

| 层面 | 推荐方案 | 备选方案 | 选择理由（MVP 定位：中小规模、简化版、快速迭代） |
| --- | --- | --- | --- |
| 后端语言/框架 | **Go 1.22+**（`net/http` + 轻量路由，如 Gin；`database/sql` + sqlx/GORM） | Java 17 + Spring Boot 3 | 单二进制部署简单；goroutine 天然适合长轮询的高并发挂起；内存/资源占用低；标准库加密（crypto/aes、crypto/cipher）够用 |
| 数据库 | **PostgreSQL 15+** | MySQL 8 | ACID 事务保证发布原子性；JSONB 存版本快照 + 索引；行锁（`FOR UPDATE`）支持并发发布串行化 |
| 缓存/通知 | **MVP 单实例内存通知**（watch 等待器注册表） | Redis pub/sub（多实例演进） | 中小规模单实例内存即可；多实例时再引入 Redis 广播版本变更 |
| 敏感值加密 | **AES-256-GCM**（Go `crypto/aes` + `crypto/cipher`） | 硬件 KMS 透明加密 | 字段级、自带认证（防篡改）、无填充；密钥外部注入 |
| 认证 | **JWT（访问令牌）** + Consumer API Key | 对接组织 SSO/OIDC | 无状态、易落地；认证接口抽象，后续可换 SSO |
| 迁移 | **golang-migrate**（SQL 迁移脚本） | Flyway | 版本化 DDL，随部署执行 |
| 可观测 | 结构化日志（slog）+ 指标（Prometheus 暴露） | — | 审计之外的操作日志与健康指标 |

> **说明**：D4 结论为首版官方 SDK 建议 Java；服务端与 SDK 通过 HTTP/REST 通信，语言无关，故服务端可用 Go、SDK 用 Java，二者不冲突。若团队以 Java 为主栈，可整体切到 Spring Boot 3，本设计的数据模型与接口契约不变。

### 2.2 关键选型理由（补充）

- **Go 而非 Java（服务端）**：长轮询意味着大量请求被挂起，Go 的 goroutine 每连接开销远小于线程池模型；单二进制部署契合「基础版」运维成本；标准库即可完成 AES-GCM、JWT、HTTP，依赖面小。
- **PostgreSQL 而非 MySQL**：版本快照天然是半结构化数据，JSONB 可「一列存快照 + 建 GIN 索引 + 事务内原子写入」，MySQL 的 JSON 在部分查询/索引能力上略弱；两者 ACID 均满足，PostgreSQL 的 `SELECT ... FOR UPDATE` 与 `JSONB` 让并发发布实现更简单。
- **内存通知而非直接上 Redis**：MVP 单实例，发布后同进程内通知 waiter 即可；保留 `Notifier` 接口，多实例部署时替换为 Redis pub/sub 实现，无需改业务代码。

---

## 3. 系统架构

### 3.1 分层架构

```
HTTP 客户端（控制台 M7 / SDK / 脚本）
        │  JSON over HTTPS
        ▼
┌─────────────────────────────────────────────┐
│ API 层（router + handler）                   │
│  - 认证中间件（JWT/APIKey）                   │
│  - 授权中间件（RBAC + 环境/空间作用域）        │
│  - 审计中间件（写操作/拒绝操作留痕）           │
│  - 脱敏输出层（敏感值一律脱敏）                │
├─────────────────────────────────────────────┤
│ Service 层（业务用例）                        │
│  Namespace / Group / Draft / Publish /        │
│  Version / Rollback / Watch / Audit / RBAC    │
├─────────────────────────────────────────────┤
│ 领域支撑组件                                  │
│  Crypto（AES-GCM）· Auth（JWT/RBAC）·         │
│  WatchHub（长轮询通知）· Notifier（可替换）    │
├─────────────────────────────────────────────┤
│ Repository 层（数据访问）                     │
│  SQL + 事务边界；乐观/悲观锁封装               │
└─────────────────────────────────────────────┘
        │  SQL
        ▼
   PostgreSQL（元数据 + 草稿 + 版本快照 + 审计）
        │
   外部密钥源（KMS / 环境注入，仅引用，不落库）
```

### 3.2 模块划分（对应产品方案 M1–M7）

| 后端模块 | 职责 | 对应 M/FR |
| --- | --- | --- |
| `namespace` / `group` | 命名空间、分组的 CRUD 与归档/删除前置校验 | M1、FR-1/2 |
| `config`（draft） | 草稿 CRUD、敏感标记、加密落库 | M2、FR-3 |
| `publish` / `version` | 原子发布、版本快照、历史查询、回滚 | M3、FR-4/5 |
| `distribute` | REST 读已发布、长轮询 watch、版本号比对 | M4、FR-7 |
| `audit` | 写操作（含拒绝）全量记录、条件查询 | M5、FR-8 |
| `security` | RBAC 服务端鉴权、敏感值加密与脱敏 | M6、FR-9/10 |

### 3.3 部署拓扑（MVP）

- **单实例**：API 服务 + PostgreSQL（单机或云 RDS）+ 外部密钥源；无额外组件。
- 长轮询 watch 依赖单进程内存通知；若水平扩展，需引入 Redis pub/sub 作为 `Notifier` 实现（本版不做，预留接口）。
- 规模假设：中小量级（继承假设 8），单实例可支撑，无分片设计。

---

## 4. 数据模型（物理设计）

### 4.1 设计原则

- 逻辑唯一键 `(namespace_id, env, group_id, key)` 落到草稿表主键。
- 草稿与版本分离：`config_draft` 是可编辑工作副本；`config_version` 是不可变快照；`published_pointer` 指向当前生效版本。
- 版本快照以 `JSONB` 整组存储，发布时在**同一事务**内写入新版本并移动指针，保证原子性。
- 审计 append-only：`audit_log` 只插不改不删。
- 敏感值在 `value` 列以密文落库（带 `v1:` 前缀的密文格式，见 7.3），列表/详情/审计一律脱敏后输出。

### 4.2 表结构（DDL 级草案）

```sql
CREATE TABLE namespace (
    id          BIGSERIAL PRIMARY KEY,
    code        VARCHAR(64)  NOT NULL UNIQUE,       -- 全局唯一（BR-01）
    name        VARCHAR(128) NOT NULL,
    description TEXT,
    status      VARCHAR(16)  NOT NULL DEFAULT 'active', -- active | archived
    created_by  BIGINT,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE TABLE grp (                                     -- group 为保留字，表名用 grp
    id           BIGSERIAL PRIMARY KEY,
    namespace_id BIGINT       NOT NULL REFERENCES namespace(id),
    code         VARCHAR(64)  NOT NULL,               -- 空间内唯一（BR-01）
    name         VARCHAR(128),
    description  TEXT,
    status       VARCHAR(16)  NOT NULL DEFAULT 'active', -- active | deleted
    created_at   TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ  NOT NULL DEFAULT now(),
    UNIQUE (namespace_id, code)
);

CREATE TABLE config_draft (
    namespace_id BIGINT      NOT NULL,
    env          VARCHAR(16) NOT NULL,               -- dev | test | prod（BR-02）
    group_id     BIGINT      NOT NULL,
    key          VARCHAR(255) NOT NULL,
    value        TEXT,                                -- 敏感值加密后存储
    sensitive    BOOLEAN     NOT NULL DEFAULT false,
    description  TEXT,
    updated_by   BIGINT,
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (namespace_id, env, group_id, key)    -- 逻辑唯一键（BR-03）
);

CREATE TABLE config_version (
    id             BIGSERIAL PRIMARY KEY,
    namespace_id   BIGINT      NOT NULL,
    env            VARCHAR(16) NOT NULL,
    group_id       BIGINT      NOT NULL,
    version_no     BIGINT      NOT NULL,             -- 组内单调递增 v(n)（BR-06）
    snapshot       JSONB       NOT NULL,             -- {key: {value, sensitive}}
    change_type    VARCHAR(16) NOT NULL,             -- publish | rollback
    source_version BIGINT,                           -- 回滚来源版本（BR-07）
    published_by   BIGINT,
    published_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (namespace_id, env, group_id, version_no)
);

CREATE TABLE published_pointer (
    namespace_id       BIGINT      NOT NULL,
    env                VARCHAR(16) NOT NULL,
    group_id           BIGINT      NOT NULL,
    current_version_no BIGINT,                       -- 指向当前生效版本
    updated_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (namespace_id, env, group_id)
);

CREATE TABLE audit_log (
    id             BIGSERIAL PRIMARY KEY,
    ts             TIMESTAMPTZ NOT NULL DEFAULT now(),
    actor_id       BIGINT,
    actor_name     VARCHAR(64),
    actor_role     VARCHAR(32),
    action         VARCHAR(64) NOT NULL,             -- create/update/publish/rollback/deny...
    target_type    VARCHAR(32) NOT NULL,             -- namespace/group/config/version...
    target_id      VARCHAR(64),
    namespace_id   BIGINT,
    env            VARCHAR(16),
    group_id       BIGINT,
    key            VARCHAR(255),
    before_summary TEXT,                             -- 脱敏摘要
    after_summary  TEXT,                             -- 脱敏摘要
    result         VARCHAR(16) NOT NULL,             -- success | denied
    detail         JSONB,
    request_id     VARCHAR(64)
);

-- RBAC（对接组织账号体系时，user 可替换为外部主体引用）
CREATE TABLE user_account (
    id        BIGSERIAL PRIMARY KEY,
    username  VARCHAR(64) NOT NULL UNIQUE,
    display_name VARCHAR(128),
    status    VARCHAR(16) NOT NULL DEFAULT 'active'
);

CREATE TABLE grant (
    namespace_id BIGINT NOT NULL,
    user_id      BIGINT NOT NULL,
    role         VARCHAR(32) NOT NULL,               -- ADMIN | EDITOR | VIEWER | CONSUMER
    granted_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (namespace_id, user_id)
);
```

> 说明：`env` 用 `VARCHAR(16)` + 应用层枚举校验（dev/test/prod），避免枚举列的迁移负担（BR-02）。RBAC 采用「角色（role）内置 + grant 按命名空间绑定」的轻量模型，不单独建 permission 表，权限由角色代码 + 服务端判定逻辑决定，够 MVP 用。

### 4.3 索引设计

| 表 | 索引 | 用途 |
| --- | --- | --- |
| `grp` | `(namespace_id)` | 按空间列分组 |
| `config_draft` | 主键 `(namespace_id, env, group_id, key)` | 草稿精确读 |
| `config_version` | `(namespace_id, env, group_id, version_no DESC)` | 版本历史、最新版本 |
| `audit_log` | `(namespace_id, env, ts DESC)`；`(key)`；`(actor_id, ts DESC)` | 审计多条件查询 |
| `config_version(snapshot)` | 可选 GIN 索引（JSONB） | 若未来需要按 key 检索历史值（非 MVP 必选） |

### 4.4 版本快照存储策略

- 快照 JSON 结构：`{"db.url": {"value": "jdbc:...", "sensitive": false}, "db.password": {"value": "v1:...", "sensitive": true}}`。
- 发布时从 `config_draft` 读出该 `(ns, env, group)` 全部草稿构建快照，整组写入 `config_version.snapshot`，与 `published_pointer` 更新同一事务提交（BR-05）。
- 回滚时读取目标版本 `snapshot`，作为新版本内容再次写入（`change_type=rollback`、`source_version` 指向源版本），历史不删改（BR-07）。
- 消费端读已发布始终走 `published_pointer → config_version.snapshot`，不读草稿（BR-04）。

---

## 5. 接口契约

### 5.1 通用约定

- **前缀**：`/api/v1`。
- **认证**：人类用户（Admin/Editor/Viewer）用 `Authorization: Bearer <JWT>`；消费端用 `X-Api-Key`（Consumer 凭据），两者鉴权角色不同。
- **响应包裹**：`{"code": 0, "message": "ok", "requestId": "...", "data": ...}`；`code != 0` 表示业务/系统错误。
- **错误码**：
  | HTTP | 语义 | 场景 |
  | --- | --- | --- |
  | 400 | 参数校验失败 | env 非法、key 非法（BR-02/03） |
  | 401 | 未认证 | 缺/无效 token、apikey |
  | 403 | 越权 | Viewer 发布、跨环境/空间访问（BR-11/12） |
  | 404 | 不存在 | ns/group/version 不存在 |
  | 409 | 冲突 | 同名 ns/group、并发发布冲突（BR-01/17） |
  | 422 | 业务规则不满足 | 删除存在已发布版本的分组（BR-15） |
- **脱敏**：所有响应中的敏感值一律返回脱敏串，明文不出现在响应/日志/错误（BR-09）。

### 5.2 API 清单

#### 元数据（M1，FR-1/2）
| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/namespaces` | 创建命名空间 |
| GET | `/namespaces` | 命名空间列表 |
| GET | `/namespaces/{nsId}` | 命名空间详情 |
| POST | `/namespaces/{nsId}/archive` | 归档（停止编辑/发布，历史保留） |
| POST | `/namespaces/{nsId}/groups` | 创建分组 |
| GET | `/namespaces/{nsId}/groups` | 分组列表 |
| GET | `/groups/{groupId}` | 分组详情 |
| PUT | `/groups/{groupId}` | 修改分组 |
| DELETE | `/groups/{groupId}` | 删除分组（前置校验 BR-15） |

#### 配置草稿（M2，FR-3）
| 方法 | 路径 | 说明 |
| --- | --- | --- |
| PUT | `/namespaces/{nsId}/envs/{env}/groups/{groupId}/draft` | 新增/更新草稿 key（body：`{key, value, sensitive, description}`） |
| GET | `/namespaces/{nsId}/envs/{env}/groups/{groupId}/draft` | 读取该实例全部草稿 |
| DELETE | `/namespaces/{nsId}/envs/{env}/groups/{groupId}/draft/{key}` | 删除草稿 key |

#### 发布与版本（M3，FR-4/5）
| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/namespaces/{nsId}/envs/{env}/groups/{groupId}/publish` | 原子发布，返回 `versionNo` |
| GET | `/namespaces/{nsId}/envs/{env}/groups/{groupId}/publish/preview` | 发布前 diff 预览（草稿 vs 当前已发布） |
| GET | `/namespaces/{nsId}/envs/{env}/groups/{groupId}/versions` | 版本历史列表 |
| GET | `/versions/{versionId}` | 版本详情（快照内容） |
| POST | `/versions/{versionId}/rollback` | 回滚到该版本 |

#### 配置分发（M4，FR-7，D2）
| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/namespaces/{nsId}/envs/{env}/groups/{groupId}/published` | 读当前已发布快照（REST 兜底，Consumer 权限） |
| GET | `/namespaces/{nsId}/envs/{env}/groups/{groupId}/published?version=v(n)` | 读指定版本快照 |
| GET | `/namespaces/{nsId}/envs/{env}/groups/{groupId}/watch?version=<local>` | 长轮询变更检测（挂起至变更/超时） |

#### 审计与权限（M5/M6，FR-8/9）
| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/audit-logs?nsId=&env=&group=&key=&actor=&action=&from=&to=&result=` | 审计查询 |
| POST | `/namespaces/{nsId}/members/grant` | 授权（body：`{userId, role}`） |
| DELETE | `/namespaces/{nsId}/members/{userId}/revoke` | 撤销授权 |
| GET | `/namespaces/{nsId}/members` | 成员列表 |

### 5.3 长轮询 watch 机制（D2）

- 客户端带本地版本号 `version` 发起 `watch`；服务端读取 `published_pointer.current_version_no`：
  - 若当前版本 ≠ 本地版本 → 立即返回 `{changed: true, version: v(n)}`；
  - 若相同 → 在 `WatchHub` 注册 waiter，挂起请求，直到发布通知或超时（默认 30s，可配置）→ 超时返回 `{changed: false}`。
- 发布提交后，服务端通知该 `(ns, env, group)` 的所有 waiter，唤醒后返回新版本号；客户端再 `GET published` 拉快照。
- 资源控制：单 waiter 超时、服务端并发连接上限、心跳/清理，避免连接资源耗尽（R-11）。

### 5.4 关键接口示例

**发布**（POST `/namespaces/{ns}/envs/dev/groups/{g}/publish`）
```json
// 请求（无 body 或 {comment: "..."}）
// 响应 200
{ "code": 0, "data": { "versionNo": 3, "publishedAt": "2026-11-19T12:00:00+08:00" } }
```

**读已发布**（GET `.../published`，Consumer 凭据）
```json
{ "code": 0, "data": {
    "version": 3,
    "items": [
      { "key": "db.url", "value": "jdbc:mysql://...", "sensitive": false },
      { "key": "db.password", "value": "s3cret!", "sensitive": true }
    ]
} }
```
> 说明（已定稿）：敏感值返回形态按市面通用方式确定——**服务端解密、通过 TLS 对已授权 Consumer 返回可用明文**；控制台/审计/日志一律返回脱敏串。两者由「鉴权角色 + `sensitive` 标记」区分（见第 11 节 Q1 定稿）。

---

## 6. 关键流程实现

### 6.1 发布流程（事务 + 悲观锁）

```
1. 鉴权：主体须具备 publish 权限（Admin/Editor），作用域含 (ns, env, group)
2. 开启事务
3. SELECT ... FOR UPDATE 锁 published_pointer 行（按 (ns, env, group)）
   — 串行化同实例并发发布，避免版本冲突（BR-17；悲观锁够 MVP，预留乐观锁切换）
4. 读该实例全部草稿 → 构建 snapshot JSONB（敏感 key 值已在草稿层加密）
5. 计算新版本号 = max(version_no)+1
6. INSERT config_version（change_type=publish）
7. UPDATE published_pointer SET current_version_no = 新版本
8. 写审计（action=publish，前后摘要脱敏）
9. 提交事务
10. 提交后通知 WatchHub 唤醒该实例所有 waiter（发布失败则整体回滚，无半版本 BR-05）
```

### 6.2 回滚流程

```
1. 鉴权（publish/rollback 权限）
2. 校验目标版本存在且不可变（BR-07）
3. 事务：读目标版本 snapshot → 新版本号=max+1 → INSERT config_version
   （change_type=rollback, source_version=目标版本号）→ 更新 pointer → 写审计
4. 提交；通知 waiter
5. 校验：新版本内容 == 目标版本内容（AC-4），历史版本不变
```

### 6.3 长轮询 watch 流程

```
1. 鉴权（Consumer 或人类只读角色）
2. 读 published_pointer 当前版本号
3. 版本变化 → 立即返回新版本号
4. 版本未变 → WatchHub.Register(ns, env, group, waiter)；用 context 超时挂起
5. 发布/回滚提交后 → WatchHub.Notify(ns, env, group) → waiter 收到信号 → 返回新版本号
6. 超时 → 返回 changed=false（客户端稍后重试）
```

### 6.4 敏感值加密 / 解密流程（D3）

- **写入**：`sensitive=true` 时，`crypto.Encrypt(plaintext, key)` → 得 `v1:<base64(nonce)>:<base64(ciphertext)>` 存 `value`。
- **读取**：按角色分两种返回——控制台/审计返回脱敏串；已授权 Consumer 读已发布时由服务端解密、经 TLS 返回明文。
- **密钥**：从外部密钥源（环境变量 `CONFIG_ENC_KEY` 或 KMS 引用）加载，**不落库、不写文档、不打印**；密钥加载失败则拒绝启动。
- **审计/日志**：任何环节不记录明文敏感值（BR-09、AC-11）。

---

## 7. 安全设计

### 7.1 认证

- 人类用户：登录换取 JWT（含 `userId`、`username`），中间件校验签名与过期。
- Consumer：签发独立 API Key（随机高强度串，只读权限），与人类账号分离（BR-12、产品方案 7.2 注）。
- 认证方式抽象为 `Authenticator` 接口，后续可对接组织 SSO/OIDC（Q5 已定稿为「预留对接」）。

### 7.2 授权（RBAC）

- 角色：`ADMIN / EDITOR / VIEWER / CONSUMER`，权限矩阵继承 `design.md` 7.2。
- 授权中间件：认证 → 取角色 → 校验动作权限 + 作用域（`namespace_id` 及可选 `env/group`），越环境/越空间一律拒绝（BR-08/11）。
- 拒绝同样写审计（`result=denied`），支撑 AC-8。

### 7.3 敏感值加密

- 算法 AES-256-GCM（认证加密，防篡改）；密文带版本前缀 `v1:` 便于未来密钥轮换。
- 密钥仅引用注入；MVP 单密钥 + 密文版本前缀 `v1:`，多密钥轮换留后续版本。
- 全链路脱敏：列表、详情、审计摘要、日志、错误、diff 预览均不出现明文。

### 7.4 审计

- 所有写操作 + 越权拒绝写 `audit_log`（append-only，无更新/删除接口）。
- 摘要只存脱敏值，不存明文；审计查询需 Viewer/Admin 权限。

---

## 8. 项目结构（建议，Go）

```
backend/
├── cmd/config-server/main.go        # 入口：装配依赖、启动 HTTP
├── internal/
│   ├── config/                      # 配置加载（端口、DB、密钥引用、超时）
│   ├── api/
│   │   ├── router.go                # 路由注册
│   │   ├── middleware/              # auth / rbac / audit / mask / recover
│   │   └── handler/                 # namespace / group / draft / publish /
│   │                                # version / distribute / audit / member
│   ├── service/                     # 业务用例（发布、回滚、watch 编排）
│   ├── repository/                  # SQL 访问 + 事务
│   ├── model/                       # 领域模型与 DTO
│   ├── crypto/                      # AES-GCM 加密/解密
│   ├── auth/                        # JWT 签发/校验、RBAC 判定
│   ├── watch/                       # WatchHub、Notifier 接口
│   └── audit/                       # 审计记录封装
├── migrations/                      # golang-migrate SQL 脚本
├── go.mod
└── README.md
```

---

## 9. 非功能设计

| 类别 | 设计要点 | 关联 |
| --- | --- | --- |
| 一致性 | 最终一致；发布后客户端在 `T_effective`（≤5s，秒级）内收敛，SDK 长轮询即时通知 + 本地缓存兜底 | BR-14、D2 |
| 可靠性 | 发布/回滚与指针更新同一事务，全有或全无；版本历史不可丢失 | BR-05、NFR |
| 并发 | 同实例发布用悲观锁串行化；预留乐观锁；拒绝冲突返回 409 | BR-17 |
| 安全 | 敏感值加密落库、全链路脱敏、服务端强制鉴权、审计 append-only | BR-09/11/13 |
| 可观测 | 结构化日志 + 指标（请求量、发布延迟、watch 连接数、错误率） | NFR |
| 规模 | 中小量级，单实例 + 单库；不做分片/水平扩展（假设 8） | 假设 8 |

---

## 10. 测试策略

| 层次 | 覆盖点 | 对应 AC |
| --- | --- | --- |
| 单元测试 | 加密/解密往返、版本号递增、脱敏函数、RBAC 判定、草稿/发布隔离逻辑 | AC-9/10 支撑 |
| 集成测试（真实 DB） | 命名冲突 409、发布原子性、回滚内容等价、环境隔离、审计字段完整、越权拒绝留痕 | AC-1/3/4/5/7/8 |
| 长轮询测试 | watch 挂起/变更唤醒/超时；发布后 Consumer 在 T_effective（≤5s）内读到新值 | AC-6 |
| 存储层扫描 | 敏感值落库为密文、无明文；响应/日志无凭据 | AC-9/11 |

> 集成测试需命中真实 PostgreSQL（不用 mock 数据库），避免 mock 与生产存储语义分叉导致漏测（版本快照、事务、行锁等）。

---

## 11. 决策记录（已按市面通用方式定稿）

> 用户确认按「市面通用方式」（对标 Apollo / Nacos）处理待确认项。以下为定稿决策；标注「后续版本」的项不阻断 MVP。

| 项 | 定稿决策 | 对标依据 |
| --- | --- | --- |
| 敏感信息加密（Q1/D3） | 服务端字段级 AES-256-GCM 加密落库；密钥由 KMS/环境变量注入（不落库、不写文档）；传输全程 TLS；已授权消费端经 TLS 拿到服务端解密后的明文；控制台/审计/日志一律脱敏；MVP 单密钥 + 密文版本前缀 `v1:`，多密钥轮换留后续版本 | Nacos 配置加密 + Spring Cloud Config 对称加密 |
| 生效时延（Q2/D2） | 秒级生效：发布后长轮询即时通知，`T_effective ≤ 5s`；长轮询超时 30s；SDK 本地缓存 + 定时兜底刷新（默认 5 分钟） | Apollo / Nacos 长轮询 |
| SDK 语言（Q3/D4） | Java，本地缓存降级必备 | Apollo / Nacos 生态 |
| 认证（Q5） | 用户名密码 + JWT 访问令牌；预留 OAuth2/OIDC（组织 SSO）对接点 | 通用后台做法 |
| JSON/YAML 校验（Q6） | 作为不透明字符串，MVP 不校验 | 基础版简化 |
| 命名/配额（Q7） | code 允许字母/数字/下划线/中划线，namespace/group 1~64，key 1~255；配额上限可后续调整 | 通用命名约束 |
| 发布审批（Q4） | MVP 不做（BR-16），留后续版本 | 基础版简化 |

> 上述为定稿决策（不再阻塞实现）；仅「密钥轮换细节」「组织 SSO 具体接口」「命名配额上限」需在联调时最终确认，均不影响本轮设计定稿。

---

## 12. 实现顺序建议（里程碑）

1. **M1 骨架**：项目结构、DB 迁移、认证/授权/审计/脱敏中间件、namespace/group CRUD。
2. **M2 草稿**：草稿 CRUD、敏感标记与 AES-GCM 加密落库。
3. **M3 发布与版本**：原子发布、版本快照、历史、回滚（事务 + 锁 + 审计）。
4. **M4 分发**：读已发布、长轮询 watch、WatchHub、版本号比对。
5. **M5/M6 收尾**：审计查询、成员/角色管理、全链路脱敏校验、指标。
6. **测试**：单测 + 集成测试（真实 DB）+ 存储层明文扫描，逐条对齐 AC-1~11。
7. **SDK（D4）**：Java 轻量 SDK（首次拉取 → 本地缓存 → 长轮询 → 版本比对 → 缓存更新）。

---

> **节点结论**：本文档把需求定型与产品方案完整落到后端技术实现层——技术选型、分层架构、物理数据模型、接口契约、关键流程、安全与测试均已给出，待确认项已按「市面通用方式」定稿（第 11 节）。文中不含任何真实凭据或密钥；所有验收条件均为待实现/待测试的判定条件，未声称任何实现或验证已完成。
