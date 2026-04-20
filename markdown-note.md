# RFC-046: 自研 Repo 存储系统

## TLDR

> 当前用 Gitea 管理 workspace repo，难以水平扩展。本方案以 **对象存储（S3）+ 元数据 DB** 替代 Gitea，用文件 UUID 作为稳定标识，通过 snapshot 机制支持 agent 本地同步与增量轮询，同时满足前端单文件读写需求。

## 元信息

- **作者**：@Kai & Moxt

- **创建日期**：2026-03-18

- **最后更新**：2026-03-19

## 背景与问题

Moxt workspace 目前使用 Gitea 作为 repo 后端。随着 workspace 数量增长，暴露出以下问题：

1. **伸缩瓶颈**：Gitea 单实例承载大量 repo 时，高频读写压力集中，难以水平扩展。
2. **功能过载**：Git 的分支、全局 snapshot、merge、DAG 等高级功能对 Moxt 场景完全用不到，只增加复杂度。

## 需求

### 核心需求[^NgVF2GOos-wsYbHY3YggT]

- **R1 本地同步（Agent）**：支持 Agent 将整个 repo 下载到本地；本地的增删改能识别变更并回写；可通过轮询感知远端变更

- **R2 单文件读写（前端）**：用户在前端直接创建、编辑、删除单个文件，实时写入；通过轮询感知远端变更并刷新视图

- **R3 文件版本管理**：每次写入保留历史版本，支持查看版本列表、回滚到指定版本

- **R4 海量 repo 支持**：支撑数十万级 repo，单 repo 文件数不超过 10 万，内容均为文本

### 非需求（明确排除）

- 不需要分支、merge 等 Git 高级操作

- 不需要跨文件的全局版本快照

- 并发写冲突：last-write-wins，不做锁或合并。写入时不检查 path 唯一性，冲突在读取时解决（取 `mfs_version` 最大的，自动清理旧的）

- 不需要权限控制

- 不需要离线写入支持

- 不需要文件权限位（Git executable bit `100755`）和 submodule（`160000`）——从 Git 迁移时丢弃这些属性，所有普通文件视为 mode `100644`

- 需要支持符号链接（symlink）[^aBXlXszWTO29cpqrgdMH-]

## 方案设计

### 存储结构

系统使用三种存储：**PostgreSQL**（元数据）、**S3**（文件内容与 snapshot 包）、**Redis**（缓存）。

#### 现有 Project 表（不动）

现有 `project` 表保持不变，灰度期间继续服务 Gitea 模式的 repo。迁移完成后整张表可以下掉。

```sql
project (
  id                  BIGSERIAL PRIMARY KEY,
  project_id          VARCHAR(255) UNIQUE,       -- 即旧 repo_id
  title               VARCHAR(255),
  workspace_id        VARCHAR(255),
  space_type          SMALLINT,                  -- 1=team, 2=personal, 3=ai_teammate
  head_commit_sha     CHAR(40),
  gitea_instance_id   VARCHAR(32) DEFAULT 'default',
  is_github_ready     BOOLEAN DEFAULT FALSE,
  cover_image_meta    TEXT,
  public_access_role  INT DEFAULT 0,
  deleted_ts          BIGINT DEFAULT -1,
  created_ts          BIGINT DEFAULT -1,
  updated_ts          BIGINT DEFAULT -1,
  dbctime             TIMESTAMPTZ,
  dbutime             TIMESTAMPTZ
)
```

#### 新建 Repo 表

新建 `mfs_repo` 表替代 `project` 中与 repo 相关的职责。灰度期间两张表共存，`mfs_repo.storage_backend` 决定走哪条路径。

```sql
mfs_repo (
  repo_id             VARCHAR(255) PRIMARY KEY NOT NULL,  -- 与 project.project_id 保持一致，迁移时直接复用
  workspace_id        VARCHAR(255) NOT NULL,
  space_type          SMALLINT NOT NULL DEFAULT 1,       -- 1=team, 2=personal, 3=ai_teammate
  storage_backend     SMALLINT NOT NULL DEFAULT 2,       -- 1=gitea（迁移中）, 2=mfs
  mfs_shard_name      VARCHAR(32),                       -- MFS shard schema 名称（如 mfs_shard_1），仅 storage_backend=2 时有值
  deleted_ts          BIGINT NOT NULL DEFAULT -1,        -- -1=未删除，>0=删除时间戳（与 project 表一致）
  created_ts          BIGINT NOT NULL DEFAULT -1,
  updated_ts          BIGINT NOT NULL DEFAULT -1,
  dbctime             TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
  dbutime             TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP  -- Drizzle $onUpdate(() => new Date())
)
-- 索引：
--   (workspace_id)              → 查找 workspace 下的 repo
```

`repo_id` 直接复用 `project.project_id`，迁移时无需 ID 映射，前端持有的 project_id 在切换后仍然有效。

**Repo 与用户的归属关系**：`mfs_repo` 本身不存 owner。归属通过现有 `user_role` 表关联：

```sql
user_role (
  id            BIGSERIAL PRIMARY KEY,
  userid        BIGINT NOT NULL,
  role          INT NOT NULL,              -- ADMIN 表示归属
  resourcetype  SMALLINT NOT NULL,         -- PROJECT 类型
  resourceid    VARCHAR(64) NOT NULL,      -- 即 repo_id / project_id
  jointime      TIMESTAMPTZ NOT NULL
)
-- 约束：UNIQUE (resourceid, resourcetype, userid)
-- 索引：(userid, resourcetype)
```

查找某用户的 personal repo：`JOIN user_role ON repo_id = resourceid WHERE userid=? AND resourcetype=PROJECT AND role=ADMIN AND space_type IN (2,3)`。Team repo（`space_type=1`）每个 workspace 只有一个，直接按 `workspace_id` 查。

与 `project` 表的主要差异：

- **去掉 Gitea 专属字段**：`head_commit_sha`、`gitea_instance_id`、`is_github_ready`、`title`、`cover_image_meta`、`public_access_role`

- **时间戳沿用旧模式**（与 `project` 表一致）：`deleted_ts`/`created_ts`/`updated_ts`（BIGINT）+ `dbctime`/`dbutime`（TIMESTAMPTZ）

#### Sharding

每个 shard 对应一个 PostgreSQL schema（如 `mfs_shard_1`），拥有独立的表和 sequence。路由方式：

```
shard_schema = mfs_repo.mfs_shard_name   -- 如 mfs_shard_1
```

#### PostgreSQL 表结构（每个 shard 相同）

```sql
-- 变更提交（类似 Git commit，一个 commit 包含一或多个文件变更）
mfs_shard_1.mfs_commits (
  id            UUID PRIMARY KEY NOT NULL,  -- 调用方生成，无默认值
  repo_id       VARCHAR(255) NOT NULL,
  user_id       BIGINT NOT NULL,          -- 操作者（人类用户或 agent 对应的 user_id）
  pipeline_id   VARCHAR(20),              -- agent 修改时关联的 pipeline，人类直接操作时为 NULL
  commit_message TEXT NOT NULL,           -- 业务元数据，沿用 Gitea 的 [email][ACTION][summary] 格式
  dbctime       TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
  dbutime       TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
)
-- 索引：
--   (repo_id)                 → 按 repo 查 commit

-- 文件版本历史（append-only，每行是一个 commit 内的单个文件变更）
mfs_shard_1.mfs_file_changes (
  id            BIGSERIAL PRIMARY KEY,
  commit_id     UUID NOT NULL,            -- 关联 mfs_commits.id
  file_id       UUID NOT NULL,
  repo_id       VARCHAR(255) NOT NULL,    -- 冗余，方便按 repo 查询
  change_type   SMALLINT NOT NULL,        -- 1=created, 2=updated, 3=renamed, 4=deleted
  file_type     SMALLINT NOT NULL DEFAULT 1, -- 1=regular file, 2=symlink
  path          TEXT NOT NULL,            -- 该版本时的路径
  content_hash  TEXT,                     -- deleted 时为 NULL；symlink 时内容为目标路径字符串，此为其 SHA-256
  size          INT,                      -- deleted 时为 NULL
  dbctime       TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
  dbutime       TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
)
-- 索引：
--   (commit_id)               → 查一个 commit 内的所有文件变更
--   (file_id)                 → 单文件版本历史 WHERE file_id=? ORDER BY id
--   (repo_id, id)             → 增量轮询 WHERE repo_id=? AND id > V

-- 文件当前状态（mfs_file_changes 的物化视图，best-effort 更新）
mfs_shard_1.mfs_files (
  id                 UUID PRIMARY KEY NOT NULL,  -- 调用方生成，无默认值
  repo_id            VARCHAR(255) NOT NULL,
  file_type          SMALLINT NOT NULL DEFAULT 1, -- 1=regular file, 2=symlink
  path               TEXT NOT NULL,               -- 当前路径，如 "team/dev/readme.md"
  mfs_version BIGINT NOT NULL,              -- 指向 mfs_file_changes.id
  content_hash       TEXT,                         -- SHA-256；symlink 时内容为目标路径字符串，此为其 SHA-256
  deleted            BOOLEAN NOT NULL DEFAULT FALSE,
  dbctime            TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
  dbutime            TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
)
-- 索引：
--   PK (id)                   → 单文件读取 WHERE id=?，精确匹配，O(1)
--                                文件修改 UPDATE WHERE id=?，同上
--   (repo_id, mfs_version) → 按 repo 查文件变更 WHERE repo_id=? AND mfs_version > V
--   (repo_id, path)           → 按路径查找 WHERE repo_id=? AND path=?，可能返回多行（path 冲突），取 mfs_version 最大的
--                                写入时不做 path 唯一性检查，冲突在读取时解决

-- Repo 全量快照（由 Compact Cronjob 写入）
mfs_shard_1.mfs_repo_snapshots (
  id            BIGSERIAL PRIMARY KEY,
  repo_id       VARCHAR(255) NOT NULL,
  mfs_version   BIGINT NOT NULL,          -- 生成时的版本号
  full_s3_key   TEXT NOT NULL,            -- tar.gz，含全部文件内容
  tree_s3_key   TEXT NOT NULL,            -- JSON，仅含目录树
  dbctime       TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
  dbutime       TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
)
-- 索引：
--   (repo_id)                 → Compact 查/删 snapshot WHERE repo_id=?，正常 0~1 行，O(1)
--                                文件树合并 WHERE repo_id=?，同上

-- Cronjob 扫描进度（单行表，CHECK 约束确保只有一行）
mfs_shard_1.mfs_compact_cursor (
  id              INT PRIMARY KEY DEFAULT 1 NOT NULL,
  scanned_version BIGINT NOT NULL DEFAULT 0,  -- 上次扫描到的 mfs_file_changes.id（对应「当前时间 - 10 分钟」的位置）
  dbctime         TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
  dbutime         TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT mfs_compact_cursor_single_row CHECK (id = 1)
)
```

每个文件用 **UUID** 作为稳定标识（`mfs_files.id`），path 是可变元数据。`mfs_file_changes.id`（BIGSERIAL）在 shard 内单调递增，充当版本号（下文称 `change_id`），单个 repo 内不连续。

#### S3 对象

```
-- 文件内容（永不删除，按 content_hash 去重）
blobs/{content_hash}                          -- SHA-256，相同内容只存一份
-- 写入者：文件修改流程
-- 上传：Conditional PUT（If-None-Match: *），key 已存在时 S3 返回 412 自动跳过

-- 全量 snapshot
snapshots/{repo_id}/{mfs_version}.tar.gz      -- 含全部文件内容
-- 写入者：Compact Cronjob

-- 目录树 snapshot
snapshots/{repo_id}/{mfs_version}.json        -- 仅含目录树（file_id/path/size/content_hash）
-- 写入者：Compact Cronjob
```

#### Redis 缓存

```
Key:   mfs:repo_snapshot:{repo_id}
Value: JSON { id, mfs_version, full_s3_key, tree_s3_key, dbctime }
TTL:   不设过期，由业务流程主动管理
```

缓存存在时，表示该 repo 自上次 snapshot 以来没有新的文件变更，可直接使用。缓存不存在时，退化为 DB 查询。

## 业务流程[^4YGVkMQh1qgomsFkOvDqg]

十个核心流程：

1. **文件修改** — 前端/Agent 写入单文件
2. **删除文件** — 删除指定路径的文件（含 path 冲突清理）
3. **删除目录** — 删除指定路径前缀下的所有文件
4. **移动目录** — 批量重命名指定路径前缀下的所有文件
5. **读取文件** — 按路径获取文件内容
6. **增量轮询** — 客户端检查是否有新变更
7. **Snapshot Compact Cronjob** — 后台定期压缩增量为完整快照
8. **获取文件树** — 获取 repo 的完整目录结构（依赖 Compact 生成的 snapshot）
9. **Agent 拉取** — Agent 启动时下载全量 snapshot
10. **Agent 提交** — Agent 结束时拉取远端最新状态，合并冲突后回写

### 1. 文件修改

文件写入有两条路径，取决于调用方使用的 API：

**路径 A：兼容 API**（服务端中转内容）

```
1. 路由到 shard：shard = mfs_repo.mfs_shard_name
2. 前置检查：
   - change_type='created' → SELECT 1 FROM mfs_files WHERE repo_id=? AND path=? AND deleted=false LIMIT 1
     已存在 → 报错（文件已存在）
   - change_type='updated' → SELECT 1 FROM mfs_files WHERE id=? AND deleted=false
     不存在或 deleted=true → 报错（文件已删除或不存在）
3. DEL mfs:repo_snapshot:{repo_id}              -- 写前删缓存
4. 计算文件内容的 content_hash（SHA-256）
5. S3 Conditional PUT {content_hash}（If-None-Match: *），已存在则自动跳过
6. INSERT INTO mfs_commits (id, repo_id, user_id, pipeline_id, commit_message)
   -- 返回 commit_id
7. INSERT INTO mfs_file_changes (commit_id, file_id, repo_id, change_type, path, content_hash, size)
   -- id（BIGSERIAL）自动分配，作为版本号
8. 后置检查：查 mfs_file_changes 中步骤 2 检查到步骤 7 INSERT 之间（即 id 区间内），是否有该 file_id 的 deleted 记录
   - 有 → 本次修改无效，返回错误（文件在写入期间被删除）。已写入的 commit 和 file_change 记录成为无效记录，由 Compact Cronjob 处理
   - 无 → 继续
9. UPSERT mfs_files (INSERT ON CONFLICT id DO UPDATE) SET mfs_version=..., content_hash=..., path=..., deleted=...
   -- best-effort，失败不影响数据正确性；created 时为 INSERT，其余为 UPDATE
```

**路径 B：Content-Addressed API**（客户端直传 S3）

```
客户端：
  1. 计算 content_hash
  2. POST /mfs/presign-upload → 获取 presigned URL（如果 alreadyExists=true 则跳过上传）
  3. PUT 文件内容到 S3 presigned URL
  4. POST /mfs/write { fileId, path, contentHash, size, changeType, commitMessage }

服务端（/mfs/write 处理）：
  1. 路由到 shard
  2. 前置检查（同路径 A 步骤 2）
  3. DEL mfs:repo_snapshot:{repo_id}
  4. INSERT INTO mfs_commits ...（同路径 A 步骤 6）
  5. INSERT INTO mfs_file_changes ...（contentHash 由客户端传入，不再计算和上传）
  6. 后置检查（同路径 A 步骤 8）：检查 INSERT 期间是否有删除，有则返回错误
  7. UPSERT mfs_files ...（同路径 A 步骤 9）
```

路径 B 中 S3 上传由客户端完成，服务端只做元数据写入，减少服务端带宽和延迟。

**不需要数据库事务**。步骤 6-7（INSERT mfs_commits + mfs_file_changes）是写入点[^XMP-ZRS9nVkJBCs_RH-9c]，步骤 8 做后置[^sSoIdFwLVra3O_OwgZKC6]检查确认写入有效。步骤 9 的 `mfs_files` 更新是为了加速单文件查询的快捷路径，即使中断导致 `mfs_files` 状态过期，Compact Cronjob 会基于 `mfs_file_changes` 重建正确的 snapshot，最终一致。

**缓存失效**：只在写前做一次 DEL（步骤 3）。写后不再做第二次删除，接受小概率缓存脏——如果步骤 3 DEL 之后、INSERT 之前恰好 Cronjob 运行并写入了缓存，该缓存会指向一个不含本次修改的 snapshot。这个窗口极短，且下次 Cronjob 运行时会生成包含本次修改的新 snapshot 并覆盖缓存，最终一致。

**中断恢复**：

- **S3 上传前崩溃**：缓存已删，无副作用 → 下次 Cronjob 重建缓存

- **S3 上传成功、INSERT 前崩溃**：S3 多了一份内容（按 hash 寻址，无害） → 无需处理

- **INSERT 成功（mfs_commits + mfs_file_changes）、UPDATE mfs_files 失败**：`mfs_files` 状态过期，单文件查询可能返回旧版本 → Compact Cronjob 基于 `mfs_file_changes` 生成正确 snapshot

### 2. 删除文件

```
1. 路由到 shard：shard = mfs_repo.mfs_shard_name
2. DEL mfs:repo_snapshot:{repo_id}
3. 查询同路径的所有文件：SELECT id FROM mfs_files WHERE repo_id=? AND path=? AND deleted=false
4. INSERT INTO mfs_commits (id, repo_id, user_id, pipeline_id, commit_message)
   -- 一次删除操作生成一个 commit
5. 对查询结果中的每个文件，INSERT INTO mfs_file_changes (commit_id, change_type='deleted', file_id, repo_id, path)
6. UPDATE mfs_files SET deleted=true WHERE id IN (上述所有 file_id)
```

步骤 3 确保同一 path 下所有 file_id 都被标记为 deleted——包括因 path 冲突而存在的 `mfs_version` 较小的文件。如果只删除一个 file_id，残留的旧文件会在按 `deleted=false` 过滤时浮出来，被错误地当作当前文件。

### 3. 删除目录

```
1. 路由到 shard：shard = mfs_repo.mfs_shard_name
2. DEL mfs:repo_snapshot:{repo_id}
3. 查询目录下所有文件：SELECT id, path FROM mfs_files WHERE repo_id=? AND path LIKE '{dirPath}/%' AND deleted=false
4. INSERT INTO mfs_commits (id, repo_id, user_id, pipeline_id, commit_message)
   -- 一次目录删除生成一个 commit
5. 对查询结果中的每个文件，INSERT INTO mfs_file_changes (commit_id, change_type='deleted', file_id, repo_id, path)
6. UPDATE mfs_files SET deleted=true WHERE id IN (上述所有 file_id)
```

逻辑与删除文件相同，区别在于步骤 3 按路径前缀匹配而非精确匹配。同样需要确保同一 path 下所有 path 冲突的文件都被标记为 deleted。

### 4. 移动目录

```
1. 路由到 shard：shard = mfs_repo.mfs_shard_name
2. DEL mfs:repo_snapshot:{repo_id}
3. 查询目录下所有文件：SELECT id, path, content_hash, size FROM mfs_files WHERE repo_id=? AND path LIKE '{oldDirPath}/%' AND deleted=false
4. INSERT INTO mfs_commits (id, repo_id, user_id, pipeline_id, commit_message)
   -- 一次目录移动生成一个 commit
5. 对查询结果中的每个文件，计算新路径 newPath = replace(path, oldDirPath, newDirPath)，
   INSERT INTO mfs_file_changes (commit_id, change_type='renamed', file_id, repo_id, path=newPath, content_hash, size)
6. UPDATE mfs_files SET path=newPath WHERE id IN (上述所有 file_id)
```

移动目录是批量 rename 操作。每个文件的 `file_id` 不变，只更新 `path`。所有文件变更归属同一个 commit。

### 5. 读取文件

**兼容 API**：`GET /{projectId}/file-content?filepath=...`

```
1. 查 DB: SELECT * FROM mfs_files WHERE repo_id=? AND path=? AND deleted=false ORDER BY mfs_version DESC LIMIT 1
2. 如果返回多行（path 冲突），取 mfs_version 最大的
3. 用 content_hash 从 S3 GET 文件内容
4. 返回 { content, blobSha: content_hash, encoding }
```

**Content-Addressed API**：`GET /{projectId}/mfs/file-content?filepath=...`

```
1. 查 DB: SELECT * FROM mfs_files WHERE repo_id=? AND path=? AND deleted=false ORDER BY mfs_version DESC LIMIT 1
2. 如果返回多行（path 冲突），取 mfs_version 最大的
3. 生成 S3 presigned GET URL
4. 返回 { contentHash, downloadUrl }，客户端直接从 S3 下载
```

`mfs_files` 是快捷路径，如果其状态过期（文件修改中断导致），返回的可能是旧版本，Compact Cronjob 会最终修正。

**Path 冲突处理**：写入时不检查 path 唯一性（不同 mfs_file_id 可能写入相同 path）。读取时如果同一 path 对应多个文件，取 `mfs_version` 最大的，忽略其余。冲突的清理由 Compact Cronjob 负责——构建 snapshot 时检测同 path 多文件，为 `mfs_version` 较小的文件生成 `deleted` change 并更新 `mfs_files`。文件树构建也按此规则——同一 path 多个文件只保留最新版本。

### 6. 增量轮询

客户端（前端/Agent）持有上次同步的版本号 V，定期查询：

**兼容 API**：前端先 `GET /{projectId}/head-commit-sha`（返回 `{ headCommitSha: string }`，MFS 下映射为 `mfs_file_changes` 最大 `id` 字符串化），与本地缓存的 sha 比较；不同则 `GET /{projectId}/tree` 重新拉取完整文件树。这与现有前端轮询逻辑完全一致，无需改动前端代码。

**Content-Addressed API**：`GET /{projectId}/mfs/has-changes?since=V`

```
1. GET mfs:repo_snapshot:{repo_id}
2. 命中且 snapshot.mfs_version == V → 返回 { changed: false }
3. 命中且 snapshot.mfs_version > V → 返回 { changed: true, latestVersion }
4. 未命中 → 查 DB 获取 mfs_file_changes 最大 id，与 V 比较
```

只返回是否有变更，不返回变更列表。有变更时客户端再调 `GET /mfs/tree` 获取完整文件树。

### 7. Snapshot Compact Cronjob

每 **1 分钟**执行一次，将增量变更压缩为完整 snapshot。通过 `mfs_compact_cursor` 表记录扫描进度，避免每次扫描范围越来越大。

**扫描逻辑**：

```
1. 读取 mfs_compact_cursor.scanned_version（记为 cursor_v）
2. 查询 mfs_file_changes 中 id > cursor_v 的记录，按 repo_id 聚合变更数
3. 对每个有变更的 repo，判断是否需要刷新 snapshot：
   - 该 repo 上次 snapshot 距今 > 10 分钟 → 刷新
   - 该 repo 自上次 snapshot 以来累计 ≥ 10 个文件变更 → 提前刷新
   - 否则跳过，等下次 cronjob
4. 对需要刷新的 repo 执行 compact（见下方）
5. 移动 offset：cursor_v = 当前时间 - 10 分钟 对应的 mfs_file_changes.id
```

**Offset 移动策略**：每次扫描结束都移动 cursor，但 cursor 不是移到最新的 `mfs_file_changes.id`，而是移到「当前时间 - 10 分钟」对应的 `mfs_file_changes.id`。这样：最近 10 分钟内的变更会被下次扫描重新覆盖，保证还没到 10 分钟过期条件的 repo 不会被跳过；超过 10 分钟的变更如果已经 compact 过则幂等无副作用，如果因为变更数不足被跳过则这次会因为满足 10 分钟条件而被刷新。

**单个 repo 的 compact 流程**：

```
1. 尝试获取分布式锁：SET mfs:compact_lock:{repo_id} NX EX 60（60 秒超时）
   - 获取失败 → 说明另一个实例正在 compact 该 repo，跳过
2. 读取该 repo 上一个 snapshot（从 mfs_repo_snapshots 或 Redis 缓存）
   - 有 snapshot → 查 mfs_file_changes 中 snapshot 之后的增量（JOIN mfs_commits 获取 commit 元数据），在内存中合并为完整文件树
   - 无 snapshot（新 repo）→ 查该 repo 全部 mfs_file_changes（JOIN mfs_commits），从零构建文件树
   合并时处理无效记录：如果一个 file_id 先有 deleted 记录，之后又有 created/updated 记录（写入时后置检查失败但 INSERT 已落盘），
   则忽略 deleted 之后的记录（该文件已被删除，后续写入无效）
3. 打包 full snapshot（tar.gz）和 tree snapshot（JSON），上传到 S3
4. 写入 DB：INSERT INTO mfs_repo_snapshots (repo_id, mfs_version=当前最大版本号, full_s3_key, tree_s3_key)
5. 删除旧 snapshot：DELETE FROM mfs_repo_snapshots WHERE repo_id=? AND id != 新id
6. 写入缓存：SET mfs:repo_snapshot:{repo_id} = 新 snapshot JSON
7. 释放锁：DEL mfs:compact_lock:{repo_id}
```

正常情况下每个 repo 只保留一行 snapshot 记录。读取时如果遇到多行（compact 步骤 5 中断导致），取 `mfs_version` 最大的。

**并发控制**：通过 Redis 分布式锁保证同一时刻只有一个实例 compact 同一个 repo。锁的 60 秒超时兜底实例崩溃场景——即使持锁实例挂掉，锁也会自动过期，下次 Cronjob 可以重新获取。

**中断恢复**：

- **S3 上传完成、DB 未写入**：S3 有多余 snapshot 文件 → 无害，下次 Cronjob 重新生成

- **DB 写入完成、Redis 未写入**：缓存缺失，读走 DB → 下次 Cronjob 会补上

- **锁未释放**：60 秒后自动过期，下次 Cronjob 重新 compact → 幂等

- **Offset 未移动**：下次扫描重复覆盖同一范围 → 幂等，重复 compact 无副作用

**与文件修改的并发**：如果 Cronjob 写入缓存后恰好有文件修改，文件修改的写前删会立即清除缓存，保证不会读到过期数据。[^sKX6pwICcTEBw3cvPPeJJ]

### 8. 获取文件树

**兼容 API**：`GET /{projectId}/tree`

**Content-Addressed API**：`GET /{projectId}/mfs/tree`

```
1. GET mfs:repo_snapshot:{repo_id}
2. 命中 → 缓存中的 snapshot 即为最新状态（没有后续变更），直接用 S3 key 下载返回
3. 未命中 → 需要合并 snapshot + 增量：
   a. 查 DB 获取 snapshot：SELECT * FROM mfs_repo_snapshots WHERE repo_id=?（正常只有一行；如有多行取 mfs_version 最大的）
   b. 有 snapshot → 查增量：SELECT * FROM mfs_file_changes WHERE repo_id=? AND id > snapshot.mfs_version ORDER BY id，合并到 snapshot 文件树
   c. 无 snapshot（新 repo，尚未被 compact）→ 查全部：SELECT * FROM mfs_file_changes WHERE repo_id=? ORDER BY id，从零构建文件树
   d. 返回合并后的结果（不回填缓存）
```

缓存命中时无需合并——缓存存在意味着 snapshot 之后没有新的文件变更。缓存未命中时，说明有增量修改尚未被 Compact Cronjob 压缩，必须在内存中合并。不回填缓存，因为只有 Cronjob 生成新 snapshot 后才能写入缓存。

### 9. Agent 拉取

Agent 全程使用 Content-Addressed API。启动时获取全量，运行期间不做增量同步。

```
1. GET /mfs/tree → 获取文件树和所有文件内容
2. 在本地 git init，将所有文件 commit 为初始状态
3. 记录 mfsVersion 作为 V0
4. Agent 在本地 git 仓库中工作，所有修改通过 git diff 可追踪
```

### 8. Agent 提交[^XN-RNIh3guX-HVTY_8jeL]

Agent 运行结束后将本地变更回写，使用 Content-Addressed API。[^hFU0JMryriuL4I3zjDLJB]提交前通过 git stash + 远端拉取 + stash pop 来合并冲突，循环直到无冲突。

```
1. git stash — 暂存本地所有修改
2. GET /mfs/has-changes?since=V0 → 检查远端是否有变更
3. 无变更 → git stash pop，跳到步骤 7
4. 有变更 → GET /mfs/tree 获取远端最新文件树，将远端变更写入本地工作区并 git commit
5. git stash pop — 恢复本地修改
   - 无冲突 → 跳到步骤 7
   - 有冲突 → Agent 解决冲突，git add + git commit，更新 V0，回到步骤 1
6.（步骤 5 回到步骤 1 是为了确保 Agent 解决冲突期间没有新的远端变更）
7. 通过 git diff 计算最终变更集（相对于远端最新状态）
8. 对每个变更文件计算 content_hash
9. 批量 POST /mfs/presign-upload，过滤掉 alreadyExists 的文件
10. 并发 PUT 文件内容到 S3（presigned URL）
11. POST /mfs/batch-write 批量提交元数据
```

## API 设计

提供两套 API：**兼容 API** 服务前端页面，保持与现有 `ProjectController` 一致的接口签名，前端无需改动；**Content-Addressed API** 服务 Agent，客户端直接与 S3 交互（通过 presigned URL），请求中只传 `content_hash` 而非文件内容。

### 兼容 API（与现有 ProjectController 对齐）

基础路径：`/web/api/projects/{projectId}`

- **GET** **`/{projectId}/tree`** — 获取文件树，返回 `{ sha, entries[], truncated }`。`sha` 映射为 `mfs_file_changes` 最大 `id`，`entries[].sha` 映射为 `content_hash`

- **GET** **`/{projectId}/file-content?filepath&ref`** — 读取单文件内容，返回 `{ content, blobSha, encoding }`。`ref` 是 Git 分支名或 commit SHA，MFS 下忽略（始终读最新）。`blobSha` 映射为 `content_hash`

- **GET** **`/{projectId}/file-blob-sha?filepath`** — 获取文件 SHA → 返回 `mfs_files.content_hash`

- **GET** **`/{projectId}/head-commit-sha`** — 获取最新 commit SHA → 返回当前最大 `mfs_file_changes.id`（字符串化）

- **POST** **`/{projectId}/files`** — 创建文件（body 含 `filepath`, `content`, `message`） → 服务端计算 hash → 走流程 1

- **PUT** **`/{projectId}/files`** — 更新文件（body 含 `filepath`, `content`, `message`, `safeSave`） → 同上，`safeSave.baseSha` 对应旧 `content_hash`

- **POST** **`/{projectId}/files/delete-file-v2`** — 删除文件（body 含 `path`, `sha`） → 走流程 1（change_type=deleted）

- **POST** **`/{projectId}/files/rename-file-v2`** — 重命名文件 → 走流程 1（change_type=renamed）

- **POST** **`/{projectId}/files/batch-rename-files`** — 批量重命名 → 批量走流程 1

- **POST** **`/{projectId}/files/batch-delete-files`** — 批量删除 → 批量走流程 1

- **GET** **`/{projectId}/files/commits?filepath`** — 单文件版本历史 → 查 `mfs_file_changes`

- **POST** **`/{projectId}/commits/:sha/diff`** — 版本 diff → 对比两个版本的内容

- **GET** **`/{projectId}/content-search?q`** — 全文搜索 → 从最新 snapshot 中搜索

- **POST** **`/{projectId}/folders`** — 创建文件夹 → 写入 `.gitkeep` 文件

- **GET** **`public/{projectId}/tree`** — 公开访问文件树 → 同 tree，跳过鉴权

- **GET** **`public/{projectId}/file-content`** — 公开访问文件内容 → 同 file-content，跳过鉴权

兼容 API 仅供前端页面使用。文件内容由服务端中转（请求 body 含 `content` 字段），服务端负责计算 `content_hash` 并上传 S3。

### Content-Addressed API（新增）

基础路径：`/web/api/projects/{projectId}/mfs`

Agent 专用 API。文件内容由 Agent 直接上传到 S3，API 请求只传 `content_hash`，服务端不中转文件内容。

**S3 Presigned URL**

```
POST /{projectId}/mfs/presign-upload
  Body: { contentHash: string, size: number }
  Response: { uploadUrl: string, alreadyExists: boolean }
```

如果 `alreadyExists=true`，客户端跳过上传。否则用 `uploadUrl`（S3 presigned PUT URL）直接上传文件内容。

**Presigned 下载**

```
GET /{projectId}/mfs/presign-download?contentHash={hash}
  Response: { downloadUrl: string }
```

通过 `contentHash` 获取 S3 presigned GET URL，客户端直接从 S3 下载文件内容。

**文件写入**

```
POST /{projectId}/mfs/write
  Body: {
    fileId: string,           // UUID
    path: string,
    contentHash: string,      // 客户端已上传到 S3 的内容 hash
    size: number,
    changeType: 'created' | 'updated' | 'renamed' | 'deleted',
    commitMessage: string   // 写入 mfs_commits，NOT NULL，沿用 [email][ACTION][summary] 格式
  }
  Response: { mfsVersion: number }
```

服务端只做元数据写入（流程 1 的步骤 2/3/6/7/8），不涉及 S3 上传。

**批量写入**

```
POST /{projectId}/mfs/batch-write
  Body: {
    items: Array<{ fileId, path, contentHash, size, changeType, commitMessage }>
  }
  Response: { mfsVersion: number }
```

**获取文件树**

```
GET /{projectId}/mfs/tree
  Response: {
    mfsVersion: number,
    entries: Array<{ fileId, path, size, contentHash }>
  }
```

**是否有变更**

```
GET /{projectId}/mfs/has-changes?since={mfsVersion}
  Response: { changed: boolean, latestVersion: number }
```

只检查是否有新变更，不返回变更列表。有变更时客户端再调 `GET /mfs/tree` 获取完整文件树。

**读取文件**

```
GET /{projectId}/mfs/file-content?filepath={path}
  Response: { contentHash: string, downloadUrl: string }
```

按路径查找文件，返回 `contentHash` 和 S3 presigned GET URL，客户端直接从 S3 下载。

## 性能特征

- **获取文件树（缓存命中）**：Redis GET → < 1ms

- **获取文件树（缓存未命中）**：DB 查询 → \~5ms

- **增量轮询**：DB 索引查询 `(repo_id, id)` → \~5-10ms

- **文件写入（兼容 API）**：S3 PUT + DB INSERT/UPDATE + Redis DEL → \~50-100ms

- **文件写入（Content-Addressed）**：DB INSERT/UPDATE + Redis DEL（S3 由客户端直传） → \~10-20ms

- **单文件读取**：DB 查 content_hash + S3 GET → \~20-50ms

活跃 repo 的 snapshot 缓存命中率高——文件修改频率远低于文件树读取频率（Agent 启动、前端加载）。缓存未命中只发生在刚修改完文件、尚未运行 Cronjob 的窗口期内。

## 双写、灰度与回滚方案

待回答的问题：

1. 双写阶段，`ProjectService` 如何同时写 Gitea 和 MFS？写入顺序是先 Gitea 再 MFS 还是反过来？一方失败时另一方如何处理？

   A：先写 Gitea，再写 MFS。MFS 写入失败时静默记录日志，不阻断业务；Gitea 写入失败时直接报错，不写 MFS。
   Gitea 是现有的 source of truth，灰度阶段保证 Gitea 数据完整性。
   可以考虑切换到 MFS 之前，运行一次全量对账脚本（diff Gitea tree vs MFS files），补齐缺失的变更。[^w8Ela67cSt7O0VMt3UvLz]

2. `.moxt/.registry/` 中的 UUID↔path 映射如何迁移到 `mfs_files` 表？全量扫描所有 repo 的 registry 目录一次性导入，还是按需懒迁移？

   A：全量批量导入，不做懒迁移。按需懒迁移的方式每次读取需判断是否已迁移；可能长期处于半迁移状态；增加读路径复杂度。[^2J526YcgJtyPA9lUraHaI]

3. 迁移后 `mfs_files.id` 是否复用现有 registry 中的 UUID？如何保证前端持有的 file UUID 在切换后仍然有效？

   A：复用。 文件名格式：`UUID#sha256hex(filePath)`，直接将 registry 中的 UUID 作为 `mfs_files.id`。
   优点: 前端/Agent/评论零改动；无需 ID 映射表

4. 现有前端通过 `headCommitSha` 轮询变更（5 秒间隔），切到 MFS 后改为 `mfs_file_changes.id` 轮询。灰度期间同一个 workspace 内的前端如何判断该走哪条路径？

   A：前端只关心"值是否变了"，不关心值的语义。因此前端可以不需要判断。兼容 API `GET /{projectId}/head-commit-sha` 对两种后端返回统一格式，服务端内部路由。

5. Sandbox/Agent 当前通过 Git clone + git push 同步文件。灰度期间是否需要同时支持 Git 方式和 Content-Addressed API 方式？还是 Agent 侧一刀切？

   A：Agent 侧按 repo 切换，不需要同时支持两种方式。同一个 Agent 实例只用一种方式。

6. `GiteaService.changeFiles()` 支持在单次 commit 中原子地操作多个文件 + registry。MFS 的 `batch-write` 不保证原子性（逐条 INSERT）。跨文件操作（批量重命名、跨 repo 移动）在灰度期间如何保证一致性？

   A：

7. 现有 safe save 机制基于 Gitea blob SHA 做 3-way merge。MFS 用 `content_hash` 做 last-write-wins。[^knww-D-QeP58MUcLA2Km0]前端的 safe save 逻辑在灰度期间是否需要同时支持两种冲突检测？

   A：前端调用服务端，保持 3-way-merge。

8. Gitea webhook 触发 `GiteaBackupService` 做备份和 `headCommitSha` 同步。MFS 不再有 webhook。灰度期间 Gitea 侧的 webhook 是否继续运行？切换后如何替代备份功能？

   A：灰度期间 Gitea webhook 继续运行，目前 webhook 的作用是：
   `headCommitSha` 同步、文件树缓存失效、git 数据备份任务，Cron 配置同步，前三个在切换后可以不管了？Cron 考虑用新方案。

9. `RepoTreeCacheService` 缓存了 Gitea 的文件树。MFS 用 Redis 缓存 snapshot。灰度期间两套缓存如何共存？切换时如何保证缓存不残留旧数据？

   A：两套缓存隔离运行，互不干扰。切换后清除缓存就行？

10. 回滚场景：如果 MFS 上线后发现数据问题，需要切回 Gitea。期间在 MFS 上产生的写入如何回写到 Gitea？是否需要一个反向同步工具？

    A：保持双写到完全确认，灰度阶段采用 Q1 的"先 Gitea 再 MFS"方案，Gitea 始终有完整数据，回滚只需切换 `storage_backend` 回 1，**无需反向同步**。[^nofN8yeWON5TQOqkQ6CDf]

11. `content-search`（全文搜索）当前依赖 Gitea 的 `git grep`。MFS 没有 Git 索引，全文搜索如何实现？灰度期间是否降级为不可用？

    A：短期对 MFS repo 降级为不可用；中长期引入搜索引擎。

12. `getCommitHistory` 和 `getCommitDiff` 依赖 Gitea 的 Git commit 历史。MFS 的 `mfs_file_changes` 只有扁平的版本记录。兼容 API 如何模拟 commit 历史的语义？

    A：

13. 多 Gitea 实例路由（`GiteaRoutingService`，按用户邮箱域分 default/external）。迁移时不同 Gitea 实例上的 repo 如何分配 `mfs_shard_name`？是否需要按 Gitea 实例对应到不同 shard？

    A：Shard 分配与 Gitea 实例无关。按 repo_id hash 均匀分配 shard，不按 Gitea 实例映射。

14. `project.giteaInstanceId` 字段在迁移后是否还需要保留？`storage_backend=1`（gitea）的 repo 仍需要 `giteaInstanceId` 做 Gitea 实例路由，迁移完成后是否可以清理？

    A：迁移完成后可以清理。

15. 灰度策略的粒度是什么？按 workspace、按 project、还是按用户？新创建的 project 默认走 MFS，存量 project 手动迁移？

    A：按 project（repo）粒度。新创建的 project 默认走 MFS，存量 project 分批迁移。

16. **Commit message 迁移**：`mfs_commits.commit_message` 沿用 Gitea 的 `[email][ACTION][summary]` 格式（由 `buildCommitMessage()` 生成，见 `web/src/signals/workspace/commit-message.ts`）。前端代码无需改动，写入目标从 Gitea commit 变为 `mfs_commits.commit_message` 字段。`change_type` 只保留文件系统操作（created/updated/renamed/deleted），业务语义（COPY/RESET/COMMENT 等）由 `commit_message` 承载。

## 开放问题

1. **版本清理策略**：每次写入保留版本，长期运行后存储成本如何控制？
2. **轮询频率**：agent 和前端各自的合理轮询间隔？
