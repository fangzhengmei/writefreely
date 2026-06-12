# WriteFreely 数据库抽象与迁移路径评估

## 1. 架构总览

WriteFreely 的数据库系统分为三层抽象，自底向上依次为：

| 层级 | 目录/文件 | 职责 |
|------|-----------|------|
| SQL Builder 层 | [db/](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/db) | 方言无关的 SQL 语句构建（建表、改表、索引、事务） |
| 迁移引擎层 | [migrations/](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations) | 版本管理、驱动适配、迁移执行 |
| 业务数据层 | [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database.go) | 业务 CRUD 操作（用户、文章、集合等） |

三层之间的调用关系：
```
业务代码 (App)
   ↓
datastore (database.go) → SQL Builder (db/)
   ↓
migrations.Migrate()     ↗
   ↓
底层 *sql.DB (database/sql)
```

---

## 2. 驱动差异分析

WriteFreely 支持两种数据库驱动：**MySQL** (`mysql`) 和 **SQLite3** (`sqlite3`)。驱动差异通过两个维度处理。

### 2.1 编译期差异（Build Tags）

通过 Go Build Tags 分离驱动特定的错误处理代码：

| 文件 | Build Constraint | 适用场景 |
|------|------------------|----------|
| [database-sqlite.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database-sqlite.go) | `sqlite && !wflib` | 含 SQLite 支持的完整编译 |
| [database-no-sqlite.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database-no-sqlite.go) | `!sqlite && !wflib` | 仅 MySQL 的标准编译 |
| [database-lib.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database-lib.go) | `wflib` | 作为库嵌入的场景（空实现） |

驱动差异的核心函数：

- **`isDuplicateKeyErr(err)`** — 检测唯一键冲突
  - MySQL: 检查 `*mysql.MySQLError.Number == 1062`
  - SQLite: 检查 `sqlite3.Error.Code == sqlite3.ErrConstraint`

- **`isHighLoadError(err)`** — 检测高负载/连接耗尽错误
  - MySQL: 错误号 `1040` (Too many connections) 或 `1203` (Max user connections)
  - SQLite: 始终返回 `false`（文件锁机制不同）

- **`isIgnorableError(err)`** — 可忽略的错误
  - MySQL: `1267` (Collation mix)
  - SQLite: 始终返回 `false`

### 2.2 运行期差异（Driver Name 分支）

通过 `datastore.driverName` 字段在运行时动态切换 SQL 方言。

#### 2.2.1 数据类型映射

在 [migrations/drivers.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/drivers.go) 和 [db/create.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/db/create.go#L68-L130) 中定义了统一的类型映射规则：

| 逻辑类型 | MySQL | SQLite |
|----------|-------|--------|
| `typeInt()` / ColumnTypeInteger | `INT` 或 `INT(size)` | `INTEGER` |
| `typeSmallInt()` | `SMALLINT` | `INTEGER` |
| `typeTinyInt()` | `TINYINT` | `INTEGER` |
| `typeBool()` | `TINYINT(1)` | `INTEGER` |
| `typeChar(l)` | `CHAR(l)` | `TEXT` |
| `typeVarChar(l)` | `VARCHAR(l)` | `TEXT` |
| `typeVarBinary(l)` | `VARBINARY(l)` | `BLOB` |
| `typeIntPrimaryKey()` | `INT AUTO_INCREMENT PRIMARY KEY` | `INTEGER PRIMARY KEY` (即 ROWID 别名) |
| `typeText()` | `TEXT` | `TEXT` |
| `typeDateTime()` | `DATETIME` | `DATETIME` |

#### 2.2.2 SQL 函数与语法差异

| 功能 | MySQL | SQLite | 代码位置 |
|------|-------|--------|----------|
| **当前时间** | `NOW()` | `strftime('%Y-%m-%d %H:%M:%S','now')` | [now()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/drivers.go#L18-L23) |
| **默认时间戳** | `NOW()` | `CURRENT_TIMESTAMP` | [SetDefaultCurrentTimestamp()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/db/create.go#L152-L159) |
| **字符串截取** | `LEFT(field, l)` | `SUBSTR(field, 0, l)` | [clip()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database.go#L167-L172) |
| **Upsert** | `ON DUPLICATE KEY UPDATE` | `ON CONFLICT(cols) DO UPDATE SET` | [upsert()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database.go#L174-L182) |
| **日期加法** | `DATE_ADD(NOW(), INTERVAL n SECOND)` | `DATETIME('now', 'n SECOND')` | [dateAdd()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database.go#L184-L189) |
| **日期减法** | `DATE_SUB(NOW(), INTERVAL n HOUR)` | `DATETIME('now', '-n HOUR')` | [dateSub()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database.go#L191-L196) |
| **表存在性检查** | `SHOW TABLES LIKE 't'` | `SELECT name FROM sqlite_master WHERE ...` | [tableExists()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/migrations.go#L135-L152) |
| **正则匹配** | `RLIKE pattern` (含 `[[:>:]]` / `\b` 两种词边界) | `regexp` 自定义函数 | [GetPostsTagged()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database.go#L1447-L1458) |
| **多字节排序** | `COLLATE utf8_bin` | （空字符串） | [collateMultiByte()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/drivers.go#L91-L96) |
| **存储引擎** | `ENGINE = InnoDB` | （空字符串） | [engine()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/drivers.go#L98-L103) |
| **列位置指定** | `AFTER col_name` | （空字符串） | [after()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/drivers.go#L105-L110) |
| **Upsert (写死)** | `INSERT ... ON DUPLICATE KEY UPDATE` | `INSERT OR REPLACE` | [UpdateCollection()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database.go#L965-L970), [1109-1113](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database.go#L1109-L1113) |

#### 2.2.3 数据库连接参数差异

在 [connectToDatabase()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/app.go#L847-L875) 中：

- **MySQL**：
  - DSN: `user:pass@tcp(host:port)/db?charset=utf8mb4&parseTime=true&loc=Local&tls=false`
  - `SetMaxOpenConns(50)`
- **SQLite**：
  - DSN: `filename?parseTime=true&cached=shared`，使用自定义驱动 `sqlite3_with_regex`（注册了 `regexp` Go 函数）
  - `SetMaxOpenConns(2)`（SQLite 并发写入能力有限）

#### 2.2.4 驱动专属迁移

部分迁移只针对特定驱动执行，例如 [v17/fixPostSignatureCharset()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/v17.go#L13-L37)：

```go
func fixPostSignatureCharset(db *datastore) error {
    // 仅 MySQL 需要修复字符集，SQLite 的 TEXT 无编码问题
    if db.driverName != driverMySQL {
        return nil
    }
    // ... ALTER TABLE MODIFY ... CHARACTER SET utf8mb4 ...
}
```

---

## 3. 事务边界分析

### 3.1 迁移执行的事务边界

**每个迁移版本（Vx → Vx+1）内部是一个独立事务**，由各个迁移函数自行管理。

典型模式（如 [v1/supportUserInvites()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/v1.go#L13-L49)）：

```
┌─ supportUserInvites() ──────────────────────────────┐
│  t, err := db.Begin()                               │
│    ├─ t.Exec(CREATE TABLE userinvites ...)          │ ← 失败 → t.Rollback() + return err
│    ├─ t.Exec(CREATE TABLE usersinvited ...)         │ ← 失败 → t.Rollback() + return err
│    └─ t.Commit()                                    │ ← 失败 → t.Rollback() + return err
└─────────────────────────────────────────────────────┘
```

该模式的特点：
1. **原子性良好**：单个迁移中的多个 DDL 要么全部成功，要么全部回滚（注意：MySQL 部分 DDL 会隐式提交事务）
2. **失败即返回**：任何步骤失败立即中断整个迁移流程
3. **无外层总事务**：`Migrate()` 主函数本身不开启事务

### 3.2 迁移主循环的事务间隙

[Migrate()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/migrations.go#L92-L133) 主循环**不包裹在全局事务中**：

```
版本状态: N
    ↓
[迁移 N+1 内部事务]  ← 独立 tx1
    ↓ 成功
INSERT appmigrations(version=N+1)  ← 独立自动提交
    ↓
[迁移 N+2 内部事务]  ← 独立 tx2
    ↓ 成功
INSERT appmigrations(version=N+2)  ← 独立自动提交
    ↓
...
```

关键观察：
- **迁移逻辑与版本记录是两步独立操作**：`m.Migrate(db)` 提交后，才执行 `INSERT INTO appmigrations`
- 存在**潜在不一致窗口**：若迁移成功但 `INSERT appmigrations` 失败（或进程崩溃），下次重启会**重复执行**该迁移
- 因此**迁移函数必须具备幂等性**（见 §5 中断恢复）

### 3.3 业务层事务边界

以 [CreateUser()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database.go#L213-L267) 为代表的业务操作：

```
┌─ CreateUser() ─────────────────────────────────┐
│  t := db.Begin()                                │
│    ├─ INSERT INTO users ...                     │ ← 获取 LastInsertId → u.ID
│    ├─ INSERT INTO collections ...               │ ← 使用 u.ID 作为 owner_id
│    ├─ DELETE FROM collectionredirects ...       │ ← 清理重定向
│    └─ t.Commit()                                │
└─────────────────────────────────────────────────┘
```

事务粒度：**一个业务用例 = 一个事务**。跨表一致性通过事务保障（用户表与文集表必须同时创建）。

### 3.4 通用事务执行器

[db/tx.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/db/tx.go) 提供了更安全的闭包式事务执行器 `RunTransactionWithOptions()`：

```go
func RunTransactionWithOptions(ctx context.Context, db *sql.DB, txOpts *sql.TxOptions, txWork TransactionScopedWork) error {
    tx, err := db.BeginTx(ctx, txOpts)
    if err != nil { return err }
    if err = txWork(ctx, tx); err != nil {
        if txErr := tx.Rollback(); txErr != nil { return txErr }
        return err
    }
    return tx.Commit()
}
```

此函数目前仅存在于 `db/` 包中，业务代码和迁移代码**尚未广泛采用**，仍以手写 `Begin/Rollback/Commit` 为主。

---

## 4. 版本推进机制

### 4.1 版本号设计

版本号采用**线性递增整数**，直接对应 `migrations` 数组的索引 + 1。

定义在 [migrations/migrations.go#L58-L81](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/migrations.go#L58-L81)：

```go
var migrations = []Migration{
    New("support user invites", supportUserInvites),                  // V1 (v0.8.0)
    New("support dynamic instance pages", supportInstancePages),      // V2 (v0.9.0)
    // ... 共 17 个 ...
    New("fix post signature character set", fixPostSignatureCharset), // V17 (v0.17.0)
}

func CurrentVer() int {
    return len(migrations)  // 当前 = 17
}
```

版本清单：

| 版本 | 描述 | 对应应用版本 | 迁移函数 |
|------|------|-------------|----------|
| V1 | 用户邀请 | v0.8.0 | `supportUserInvites` |
| V2 | 动态实例页面 | v0.9.0 | `supportInstancePages` |
| V3 | 用户封禁/状态 | v0.11.0 | `supportUserStatus` |
| V4 | OAuth 基础支持 | - | `oauth` |
| V5 | Slack OAuth | - | `oauthSlack` |
| V6 | ActivityPub @提及 | - | `supportActivityPubMentions` |
| V7 | OAuth 账号绑定 | - | `oauthAttach` |
| V8 | 邀请链接 OAuth | v0.12.0 | `oauthInvites` |
| V9 | 草稿查询优化 | - | `optimizeDrafts` |
| V10 | 文章签名 | v0.13.0 | `supportPostSignatures` |
| V11 | OAuth Token 字段加宽 | - | `widenOauthAcceesToken` |
| V12 | Fediverse 身份验证 | v0.14.0 | `fediverseVerifyProfile` |
| V13 | 邮件订阅（Newsletter） | - | `supportLetters` |
| V14 | 密码重置 | - | `supportPassReset` |
| V15 | 文章查询索引加速 | - | `addPostRetrievalIndex` |
| V16 | ActivityPub 点赞 | v0.16.0 | `supportRemoteLikes` |
| V17 | 文章签名字符集修复 | v0.17.0 | `fixPostSignatureCharset` |

### 4.2 版本记录表 `appmigrations`

结构定义在 [migrations.go#L103-L107](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/migrations.go#L103-L107)：

| 字段 | 类型 | 含义 |
|------|------|------|
| `version` | INT / INTEGER | 迁移版本号（从 1 开始） |
| `migrated` | DATETIME | 执行完成时间 |
| `result` | TEXT | 执行结果（目前始终为空串 `""`） |

当前版本查询：`SELECT MAX(version) FROM appmigrations`

### 4.3 初始化路径（db init）

[adminInitDatabase()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/app.go#L967-L1011) 流程：

```
1. 选择 schema 文件
   ├─ SQLite → sqlite.sql
   └─ MySQL  → schema.sql
   
2. 按 ";\n" 切分 SQL 语句，逐条 Exec
   └─ 这些 schema 已包含 V1 之前的所有基础表
   
3. 调用 SetInitialMigrations()
   └─ INSERT INTO appmigrations(version=1, migrated=NOW, result="")
      （标记：静态 schema 等价于 V1）
   
4. 调用 migrations.Migrate()
   └─ 从版本 1 开始，继续执行 V2, V3, ... V17
```

> **设计意图**：静态 schema 文件维护的是「V0 → V1」的快照，减少新安装时需要执行的迁移数量。

### 4.4 增量迁移路径（db migrate）

[Migrate()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/migrations.go#L92-L133) 的算法：

```
1. 检查 appmigrations 表是否存在
   ├─ 不存在 → CREATE TABLE + 设 version=0
   └─ 存在   → SELECT MAX(version) → version = N

2. 切片 migrations[N:] 得到待执行的迁移列表
   └─ 若为空 → 输出 "Database up-to-date"

3. 循环遍历待执行列表：
   for i, m := range migrations[N:] {
       curVer := N + i + 1
       log "Migrating to VcurVer: ..."
       err = m.Migrate(db)         // 1) 执行迁移（内部事务）
       if err != nil { return err }
       INSERT INTO appmigrations   // 2) 记录版本（自动提交）
   }
```

---

## 5. 中断恢复机制

### 5.1 幂等性保障手段

由于迁移的「执行」与「版本记录」是两步操作，中间可能崩溃，因此各迁移通过以下手段保障重复执行的安全性：

#### 5.1.1 条件化 DDL

- **建表**：新安装的 schema 使用 `CREATE TABLE IF NOT EXISTS`（见 [sqlite.sql](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/sqlite.sql#L11)）
- **索引**：通过 `CREATE INDEX` 时如果索引已存在会报错 —— 但实际迁移中 `addPostRetrievalIndex`（V15）**无幂等保护**，重复执行会失败

#### 5.1.2 驱动内部分支

部分迁移通过驱动检测减少无意义执行：
- V17（fixPostSignatureCharset）：仅 MySQL 执行，SQLite 直接 `return nil`

#### 5.1.3 表存在检测

`appmigrations` 表不存在时视为全新安装，避免在已有数据上误操作：

```go
if db.tableExists("appmigrations") {
    // 正常迁移流程
} else {
    // 全新初始化：CREATE TABLE + version = 0
}
```

### 5.2 崩溃场景与恢复行为

假设执行到 V5 时发生中断：

| 崩溃时机 | 结果 | 恢复行为 |
|----------|------|----------|
| `m.Migrate(db)` 执行中（tx 未提交） | 事务自动回滚，`appmigrations` 仍停留在 V4 | 重启后从 V5 开始正常执行 ✅ |
| `m.Migrate(db)` 已返回成功，但 `INSERT appmigrations` 之前 | 迁移 DDL 已提交，但版本未记录 | 重启后会**再次执行 V5** ⚠️ |
| `INSERT appmigrations` 之后 | 版本已推进到 V5 | 重启后从 V6 开始正常执行 ✅ |

### 5.3 已知风险点

#### 风险 1：非幂等迁移的重复执行

以下迁移在重复执行时会报错（无 IF NOT EXISTS 或错误吞掉逻辑）：

| 版本 | 操作 | 重复执行结果 |
|------|------|-------------|
| V3 | `ALTER TABLE users ADD COLUMN status` | MySQL: `Duplicate column name`；SQLite: 相同错误 |
| V6 | `ALTER TABLE remoteusers ADD COLUMN handle` | 同上 |
| V15 | `CREATE INDEX posts_get_collection_index` | `Duplicate key name` 或 `index already exists` |
| V17 | `ALTER TABLE ... MODIFY post_signature` | MySQL 无报错（可重复 MODIFY） |

**建议**：关键迁移应先检查列/索引是否存在，或使用 `db/` 包中的 SQL Builder（目前迁移未使用，全部是手写原生 SQL）。

#### 风险 2：MySQL DDL 的隐式提交

MySQL 中 `ALTER TABLE`、`CREATE INDEX` 等 DDL 语句会**隐式提交当前事务**。这意味着：

- 即使 `supportUserStatus()`（V3）中用了 `Begin()` / `Rollback()`，`ALTER TABLE` 执行成功后已无法回滚
- 若后续操作（如 `Commit()`）失败，表结构已变更，但版本未记录 —— 下次会重试同样的 `ALTER TABLE` 并报错

#### 风险 3：事务回滚后的版本号未校验

迁移成功的标志是**同时满足**：
1. 数据库结构已变更
2. `appmigrations` 表有对应版本记录

当前代码仅以 `appmigrations.MAX(version)` 为准，不校验实际结构是否匹配。若人工修复数据库后跳过某些迁移，可能导致版本号与实际结构不一致。

#### 风险 4：无 result 字段利用

`appmigrations.result` 字段始终写入空字符串 `""`，未用于存储：
- 迁移错误日志
- 迁移耗时
- 跳过/部分执行的标记

这不利于事后排查。

---

## 6. SQL Builder 抽象层评估

[db/](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/db) 包提供了一套类型安全的 SQL 构建 API：

| 类型 | 用途 | 示例 |
|------|------|------|
| `DialectType` | 方言枚举，工厂入口 | `DialectMySQL.Table("users")` |
| `CreateTableSqlBuilder` | 建表 | `.Column(...).UniqueConstraint(...).ToSQL()` |
| `AlterTableSqlBuilder` | 改表 | `.AddColumn(...).ChangeColumn(...).ToSQL()` |
| `CreateIndexSqlBuilder` | 建索引 | `.CreateUniqueIndex(...)` |
| `DropIndexSqlBuilder` | 删索引 | `.DropIndex(name, table)` |
| `RawSqlBuilder` | 原生 SQL 透传 | |
| `RunTransactionWithOptions` | 闭包事务 | 支持 `context.Context` 和 `sql.TxOptions` |

**现状评估**：
- 这套 Builder 层设计良好，分离了方言差异
- 但实际 `migrations/` 包**完全未使用**该层，全部采用字符串拼接 + 驱动分支函数
- 两套方言适配逻辑并存（`migrations/drivers.go` vs `db/create.go` 的 `ColumnType.Format()`），存在重复维护成本
- 业务层 `database.go` 也未使用该 Builder，全部手写 SQL

**建议方向**：迁移代码逐步向 `db/` Builder 迁移，消除重复的类型映射逻辑。

---

## 7. 命令行入口

通过 [cmd/writefreely/db.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/cmd/writefreely/db.go) 暴露两个子命令：

```bash
writefreely db init      # 新安装：加载 schema.sql/sqlite.sql + SetInitialMigrations + Migrate
writefreely db migrate   # 升级：仅调用 Migrate()
```

---

## 8. 总结与改进建议

### 核心优势
1. **清晰的三层架构**：SQL Builder / 迁移引擎 / 业务数据层各司其职
2. **双驱动支持完备**：编译期 + 运行期双层差异处理，覆盖了类型、函数、错误、连接参数等维度
3. **版本模型简洁**：整数版本 + 线性数组，`CurrentVer() = len(migrations)` 设计优雅
4. **迁移粒度合理**：每个版本独立事务，单个失败不影响已完成版本

### 改进建议

| 优先级 | 建议 | 对应风险 |
|--------|------|----------|
| 🔴 高 | 将迁移的 `m.Migrate()` + `INSERT appmigrations` 合并到同一事务中（需注意 MySQL DDL 隐式提交问题） | §5.3 风险 1、2 |
| 🔴 高 | 为 `ALTER TABLE ADD COLUMN` 和 `CREATE INDEX` 类迁移增加「先检查后执行」逻辑，确保幂等 | §5.3 风险 1 |
| 🟡 中 | 统一使用 `db/` 包的 SQL Builder，消除 `migrations/drivers.go` 重复逻辑 | §6 现状评估 |
| 🟡 中 | 业务代码统一使用 `db.RunTransactionWithOptions()`，减少手写 Begin/Rollback 的遗漏 | §3.4 |
| 🟢 低 | 利用 `appmigrations.result` 字段记录迁移详情（耗时、错误信息等） | §5.3 风险 4 |
| 🟢 低 | 在 `Migrate()` 完成后增加结构一致性校验（可选） | §5.3 风险 3 |
