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
datastore (database.go) ── 部分调用 ──→ wf_db.RunTransactionWithOptions (db/tx.go)
   ↓                                            ↑
migrations.Migrate()                            │
   ├─ 13 个迁移：手写 SQL + drivers.go 适配     │
   └─  4 个迁移：wf_db.Builder + wf_db.RunTransactionWithOptions
   ↓
底层 *sql.DB (database/sql)
```

> **关键修正**：SQL Builder 层（`db/` 包）与迁移层并非完全脱节。实际有 **4 个迁移版本（V4/V5/V7/V8）** 完整采用了 Builder + 事务闭包的模式，业务层的 `ValidateOAuthState()` 也复用了事务执行器。详见 §6。

---

## 2. 驱动差异分析

WriteFreely 支持两种数据库驱动：**MySQL** (`mysql`) 和 **SQLite3** (`sqlite3`)。驱动差异通过 **两套并存的适配机制** + 编译期 Build Tags 三个维度处理。

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

### 2.2 运行期差异（两套方言适配机制并存）

项目中同时存在 **两种独立的方言适配机制**，分别服务于不同的迁移版本和代码路径。

#### 机制 A：`migrations/drivers.go` 手写方法分支

适用范围：**13 个迁移版本（V1/V2/V3/V6/V9-V17）** + 业务层 `database.go`

通过 `datastore.driverName` 字段在运行时动态切换 SQL 方言，每种差异对应一个独立方法。

**数据类型映射方法**（见 [migrations/drivers.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/drivers.go)）：

| 逻辑方法 | MySQL | SQLite |
|----------|-------|--------|
| `typeInt()` | `INT` | `INTEGER` |
| `typeSmallInt()` | `SMALLINT` | `INTEGER` |
| `typeTinyInt()` | `TINYINT` | `INTEGER` |
| `typeBool()` | `TINYINT(1)` | `INTEGER` |
| `typeChar(l)` | `CHAR(l)` | `TEXT` |
| `typeVarChar(l)` | `VARCHAR(l)` | `TEXT` |
| `typeVarBinary(l)` | `VARBINARY(l)` | `BLOB` |
| `typeIntPrimaryKey()` | `INT AUTO_INCREMENT PRIMARY KEY` | `INTEGER PRIMARY KEY` (ROWID 别名) |
| `typeText()` | `TEXT` | `TEXT` |
| `typeDateTime()` | `DATETIME` | `DATETIME` |

**SQL 函数与语法差异方法**：

| 方法 | MySQL | SQLite | 代码位置 |
|------|-------|--------|----------|
| `now()` | `NOW()` | `strftime('%Y-%m-%d %H:%M:%S','now')` | [drivers.go#L18-L23](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/drivers.go#L18-L23) |
| `clip()` | `LEFT(field, l)` | `SUBSTR(field, 0, l)` | [database.go#L167-L172](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database.go#L167-L172) |
| `upsert()` | `ON DUPLICATE KEY UPDATE` | `ON CONFLICT(cols) DO UPDATE SET` | [database.go#L174-L182](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database.go#L174-L182) |
| `dateAdd()` | `DATE_ADD(NOW(), INTERVAL n SECOND)` | `DATETIME('now', 'n SECOND')` | [database.go#L184-L189](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database.go#L184-L189) |
| `dateSub()` | `DATE_SUB(NOW(), INTERVAL n HOUR)` | `DATETIME('now', '-n HOUR')` | [database.go#L191-L196](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database.go#L191-L196) |
| `tableExists()` | `SHOW TABLES LIKE 't'` | `SELECT name FROM sqlite_master` | [migrations.go#L135-L152](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/migrations.go#L135-L152) |
| `collateMultiByte()` | `COLLATE utf8_bin` | 空字符串 | [drivers.go#L91-L96](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/drivers.go#L91-L96) |
| `engine()` | `ENGINE = InnoDB` | 空字符串 | [drivers.go#L98-L103](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/drivers.go#L98-L103) |
| `after()` | `AFTER col_name` | 空字符串 | [drivers.go#L105-L110](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/drivers.go#L105-L110) |

**调用方式示例（来自 V1）**：
```go
t.Exec(`CREATE TABLE userinvites (
    id ` + db.typeChar(6) + ` NOT NULL,
    ...
) ` + db.engine() + `;`)
```

#### 机制 B：`db/` 包的 Builder + DialectType 枚举

适用范围：**4 个迁移版本（V4/V5/V7/V8）** + 业务层 `ValidateOAuthState()`（仅事务执行器）

通过 `DialectType` 枚举 + 强类型 Builder 链式 API 实现类型安全的方言适配。

**类型映射**由 `ColumnType.Format(dialect, size)` 统一处理（见 [db/create.go#L68-L130](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/db/create.go#L68-L130)）：

| ColumnType 枚举 | MySQL | SQLite |
|-----------------|-------|--------|
| `ColumnTypeSmallInt` | `SMALLINT` / `SMALLINT(n)` | `INTEGER` |
| `ColumnTypeInteger` | `INT` / `INT(n)` | `INTEGER` |
| `ColumnTypeChar` | `CHAR(n)` | `TEXT` |
| `ColumnTypeVarChar` | `VARCHAR(n)` | `TEXT` |
| `ColumnTypeBool` | `TINYINT(1)` | `INTEGER` |
| `ColumnTypeDateTime` | `DATETIME` | `DATETIME` |
| `ColumnTypeText` | `TEXT` | `TEXT` |

**方言工厂入口**见 [db/dialect.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/db/dialect.go)：
- `DialectMySQL.Table(name)` → 返回绑定 MySQL 方言的 `CreateTableSqlBuilder`
- `DialectSQLite.AlterTable(name)` → 返回绑定 SQLite 方言的 `AlterTableSqlBuilder`
- `DialectMySQL.CreateUniqueIndex(...)` → 返回绑定 MySQL 方言的 `CreateIndexSqlBuilder`

**调用方式示例（来自 V4）**：
```go
dialect := wf_db.DialectMySQL
if db.driverName == driverSQLite {
    dialect = wf_db.DialectSQLite
}
// 链式 Builder 生成 SQL
sql, _ := dialect.Table("oauth_users").
    Column(dialect.Column("user_id", wf_db.ColumnTypeInteger, wf_db.UnsetSize)).
    Column(dialect.Column("remote_user_id", wf_db.ColumnTypeInteger, wf_db.UnsetSize)).
    ToSQL()
```

#### 两套机制的能力边界对比

| 差异维度 | 机制 A (drivers.go) | 机制 B (db/ Builder) |
|----------|---------------------|----------------------|
| 数据类型映射 | ✅ 完整 | ✅ 完整 |
| 时间函数 now/dateAdd | ✅ 提供 | ❌ 不提供（仅 `SetDefaultCurrentTimestamp`） |
| Upsert 语法 | ✅ 提供 | ❌ 不提供 |
| 字符串截取 clip | ✅ 提供 | ❌ 不提供 |
| 表存在性检查 | ✅ 提供 | ❌ 不提供 |
| 建表 CREATE TABLE | ✅ 字符串拼接 | ✅ 类型安全 Builder |
| 改表 ALTER TABLE | ✅ 手写字符串 | ✅ Builder (AddColumn/ChangeColumn) |
| 索引 CREATE/DROP INDEX | ✅ 手写字符串 | ✅ Builder |
| 存储引擎 ENGINE/字符集 | ✅ 提供方法 | ❌ 不提供（Builder 无对应 API） |
| 列位置 AFTER | ✅ 提供方法 | ❌ 不提供 |
| 事务管理 | ✅ 手写 Begin/Commit | ✅ 闭包 RunTransactionWithOptions |
| 类型安全 | ❌ 字符串拼接 | ✅ 枚举 + 方法链 |
| 编译期错误检查 | ❌ SQL 拼写错误运行时才发现 | ✅ Builder 方法不存在即编译错误 |

### 2.3 数据库连接参数差异

在 [connectToDatabase()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/app.go#L847-L875) 中：

- **MySQL**：
  - DSN: `user:pass@tcp(host:port)/db?charset=utf8mb4&parseTime=true&loc=Local&tls=false`
  - `SetMaxOpenConns(50)`
- **SQLite**：
  - DSN: `filename?parseTime=true&cached=shared`，使用自定义驱动 `sqlite3_with_regex`（注册了 `regexp` Go 函数）
  - `SetMaxOpenConns(2)`（SQLite 并发写入能力有限）

驱动注册见 [database-sqlite.go#L25-L36](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database-sqlite.go#L25-L36)：
```go
sql.Register("sqlite3_with_regex", &sqlite3.SQLiteDriver{
    ConnectHook: func(conn *sqlite3.SQLiteConn) error {
        return conn.RegisterFunc("regexp", regexp.MatchString, true)
    },
})
```

### 2.4 驱动专属迁移

部分迁移只针对特定驱动执行，例如 [v17/fixPostSignatureCharset()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/v17.go#L13-L37)：
```go
func fixPostSignatureCharset(db *datastore) error {
    if db.driverName != driverMySQL {
        return nil  // SQLite 的 TEXT 无编码问题，直接跳过
    }
    // ALTER TABLE MODIFY post_signature TEXT CHARACTER SET utf8mb4 ...
}
```

同理，[v5/oauthSlack()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/v5.go#L65-L75) 中修改 `remote_user_id` 列长度的操作**仅在 MySQL 上执行**（SQLite 的 VARBINARY/INTEGER 动态类型无需调整宽度）。

---

## 3. 事务边界分析

### 3.1 迁移执行的事务边界（两种模式）

迁移事务管理同样分为两种风格，与方言适配机制的选择一一对应。

#### 模式 1：手写 Begin/Commit/Rollback（13 个迁移）

适用 V1/V2/V3/V6/V9-V17。典型模式（以 [v1/supportUserInvites()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/v1.go#L13-L49) 为例）：

```
┌─ supportUserInvites() ──────────────────────────────┐
│  t, err := db.Begin()                               │
│    ├─ t.Exec(CREATE TABLE userinvites ...)          │ ← 失败 → t.Rollback() + return err
│    ├─ t.Exec(CREATE TABLE usersinvited ...)         │ ← 失败 → t.Rollback() + return err
│    └─ t.Commit()                                    │ ← 失败 → t.Rollback() + return err
└─────────────────────────────────────────────────────┘
```

#### 模式 2：闭包式事务执行器（4 个迁移：V4/V5/V7/V8）

适用 OAuth 系列迁移。以 [v4/oauth()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/v4.go#L20-L54) 为例：

```
┌─ oauth() ─────────────────────────────────────────────────────┐
│ wf_db.RunTransactionWithOptions(ctx, db.DB, &sql.TxOptions{}, │
│   func(ctx context.Context, tx *sql.Tx) error {               │
│     ├─ builder1.ToSQL() → tx.ExecContext()                    │ ← 错误直接 return，自动回滚
│     ├─ builder2.ToSQL() → tx.ExecContext()                    │ ← 错误直接 return，自动回滚
│     └─ return nil                                             │ ← 自动 Commit
│   })                                                          │
└───────────────────────────────────────────────────────────────┘
```

两种模式的对比：

| 特性 | 手写模式 | 闭包模式 |
|------|---------|---------|
| 回滚触发时机 | 每步手动 `if err != nil { t.Rollback() }` | 闭包返回非 nil error 自动回滚 |
| 遗漏回滚风险 | 高（忘记写 Rollback 即泄露事务） | 低（框架保证） |
| 上下文支持 | ❌ 用 `Exec()` 无 context | ✅ 用 `ExecContext(ctx, ...)` |
| 代码行数 | 较多（每个操作 3 行） | 简洁（闭包内专注业务） |

### 3.2 迁移主循环的事务间隙

[Migrate()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/migrations.go#L92-L133) 主循环**不包裹在全局事务中**：

```
版本状态: N
    ↓
[迁移 N+1 内部事务]  ← 独立 tx1（模式 1 或 2）
    ↓ 成功
INSERT appmigrations(version=N+1)  ← 独立自动提交
    ↓
[迁移 N+2 内部事务]  ← 独立 tx2（模式 1 或 2）
    ↓ 成功
INSERT appmigrations(version=N+2)  ← 独立自动提交
    ↓
...
```

关键观察：
- **迁移逻辑与版本记录是两步独立操作**：`m.Migrate(db)` 返回成功后，才执行 `INSERT INTO appmigrations`
- 存在**潜在不一致窗口**：若迁移成功但 `INSERT appmigrations` 失败（或进程崩溃），下次重启会**重复执行**该迁移
- 因此**迁移函数必须具备幂等性**（见 §5 中断恢复）

### 3.3 业务层事务边界

以 [CreateUser()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database.go#L213-L267) 为代表的业务操作（手写模式）：

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

### 3.4 通用事务执行器的实际使用

[db/tx.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/db/tx.go) 定义的 `RunTransactionWithOptions()` 实际被以下位置调用：

| 位置 | 用途 | 模式 |
|------|------|------|
| [migrations/v4.go#L25](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/v4.go#L25) | OAuth 表迁移（oauth_users, oauth_client_states） | 迁移 Builder 模式 |
| [migrations/v5.go#L25](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/v5.go#L25) | Slack OAuth 字段扩充 | 迁移 Builder 模式 |
| [migrations/v7.go#L25](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/v7.go#L25) | OAuth 绑定账号字段 | 迁移 Builder 模式 |
| [migrations/v8.go#L25](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/v8.go#L25) | OAuth 邀请码字段 | 迁移 Builder 模式 |
| [database.go#L2960](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database.go#L2960) | `ValidateOAuthState()` — 校验并消费 OAuth state | 业务场景 |

**业务层使用示例（ValidateOAuthState）**：
```go
err := wf_db.RunTransactionWithOptions(ctx, db.DB, &sql.TxOptions{},
    func(ctx context.Context, tx *sql.Tx) error {
        // 1. 查询 state 记录（SELECT ... FOR UPDATE 语义由事务隔离级别保证）
        err := tx.QueryRowContext(ctx,
            "SELECT provider, client_id, ... FROM oauth_client_states WHERE state = ? AND used = FALSE",
            state).Scan(&provider, ...)
        if err != nil { return err } // 自动回滚
        // 2. 标记为已使用
        res, err := tx.ExecContext(ctx,
            "UPDATE oauth_client_states SET used = TRUE WHERE state = ?", state)
        if err != nil { return err } // 自动回滚
        // 3. 验证行数影响
        rowsAffected, _ := res.RowsAffected()
        if rowsAffected != 1 { return fmt.Errorf("state not found") } // 自动回滚
        return nil // 自动提交
    })
```

这是典型的「读-改-校验」原子操作场景，使用闭包事务确保 state 不会被并发消费两次。

---

## 4. 版本推进机制

### 4.1 版本号设计

版本号采用**线性递增整数**，直接对应 `migrations` 数组的索引 + 1。

定义在 [migrations/migrations.go#L58-L81](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/migrations.go#L58-L81)：

```go
var migrations = []Migration{
    New("support user invites", supportUserInvites),                  // V1 (v0.8.0) — 手写模式
    New("support dynamic instance pages", supportInstancePages),      // V2 (v0.9.0) — 手写模式
    New("support users suspension", supportUserStatus),               // V3 (v0.11.0) — 手写模式
    New("support oauth", oauth),                                      // V4 — Builder 模式 ★
    New("support slack oauth", oauthSlack),                           // V5 — Builder 模式 ★
    New("support ActivityPub mentions", supportActivityPubMentions),  // V6 — 手写模式
    New("support oauth attach", oauthAttach),                         // V7 — Builder 模式 ★
    New("support oauth via invite", oauthInvites),                    // V8 (v0.12.0) — Builder 模式 ★
    New("optimize drafts retrieval", optimizeDrafts),                 // V9 — 手写模式
    New("support post signatures", supportPostSignatures),            // V10 (v0.13.0) — 手写模式
    New("Widen oauth_users.access_token", widenOauthAcceesToken),     // V11 — 手写模式
    New("support verifying fedi profile", fediverseVerifyProfile),    // V12 (v0.14.0) — 手写模式
    New("support newsletters", supportLetters),                       // V13 — 手写模式
    New("support password resetting", supportPassReset),              // V14 — 手写模式
    New("speed up blog post retrieval", addPostRetrievalIndex),       // V15 — 手写模式
    New("support ActivityPub likes", supportRemoteLikes),             // V16 (v0.16.0) — 手写模式
    New("fix post signature character set", fixPostSignatureCharset), // V17 (v0.17.0) — 手写模式
}

func CurrentVer() int {
    return len(migrations)  // 当前 = 17
}
```

版本清单（含所采用的适配机制）：

| 版本 | 描述 | 应用版本 | 迁移函数 | 方言机制 | 事务模式 |
|------|------|---------|----------|---------|---------|
| V1 | 用户邀请 | v0.8.0 | `supportUserInvites` | A (drivers.go) | 手写 Begin |
| V2 | 动态实例页面 | v0.9.0 | `supportInstancePages` | A (drivers.go) | 手写 Begin |
| V3 | 用户封禁/状态 | v0.11.0 | `supportUserStatus` | A (drivers.go) | 手写 Begin |
| **V4** | OAuth 基础支持 | - | `oauth` | **B (Builder)** | **闭包事务** |
| **V5** | Slack OAuth | - | `oauthSlack` | **B (Builder)** | **闭包事务** |
| V6 | ActivityPub @提及 | - | `supportActivityPubMentions` | A (drivers.go) | 手写 Begin |
| **V7** | OAuth 账号绑定 | - | `oauthAttach` | **B (Builder)** | **闭包事务** |
| **V8** | 邀请链接 OAuth | v0.12.0 | `oauthInvites` | **B (Builder)** | **闭包事务** |
| V9 | 草稿查询优化 | - | `optimizeDrafts` | A (drivers.go) | 手写 Begin |
| V10 | 文章签名 | v0.13.0 | `supportPostSignatures` | A (drivers.go) | 手写 Begin |
| V11 | OAuth Token 加宽 | - | `widenOauthAcceesToken` | A (drivers.go) | 手写 Begin |
| V12 | Fediverse 验证 | v0.14.0 | `fediverseVerifyProfile` | A (drivers.go) | 手写 Begin |
| V13 | 邮件订阅 | - | `supportLetters` | A (drivers.go) | 手写 Begin |
| V14 | 密码重置 | - | `supportPassReset` | A (drivers.go) | 手写 Begin |
| V15 | 文章查询索引 | - | `addPostRetrievalIndex` | A (drivers.go) | 手写 Begin |
| V16 | ActivityPub 点赞 | v0.16.0 | `supportRemoteLikes` | A (drivers.go) | 手写 Begin |
| V17 | 签名字符集修复 | v0.17.0 | `fixPostSignatureCharset` | A (drivers.go) + 驱动判断 | 手写 Begin |

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
       err = m.Migrate(db)         // 1) 执行迁移（内部事务，模式 1 或 2）
       if err != nil { return err }
       INSERT INTO appmigrations   // 2) 记录版本（自动提交）
   }
```

---

## 5. 中断恢复机制

### 5.1 幂等性保障手段

由于迁移的「执行」与「版本记录」是两步操作，中间可能崩溃，因此各迁移通过以下手段保障重复执行的安全性：

#### 5.1.1 条件化 DDL

- **建表**：新安装的 schema 使用 `CREATE TABLE IF NOT EXISTS`
- **V4（OAuth）**：使用 Builder 的 `SetIfNotExists(false)` — 注意此处显式设为 `false`，不具备幂等性
- **索引**：通过 `CREATE INDEX` 时如果索引已存在会报错 —— V15 `addPostRetrievalIndex` **无幂等保护**，重复执行会失败

#### 5.1.2 驱动内部分支

部分迁移通过驱动检测减少无意义执行：
- V17（fixPostSignatureCharset）：仅 MySQL 执行，SQLite 直接 `return nil`
- V5（oauthSlack）修改 remote_user_id 列：仅 MySQL 执行 `ChangeColumn`

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

假设执行到 V5（采用 Builder 闭包事务模式）时发生中断：

| 崩溃时机 | 结果 | 恢复行为 |
|----------|------|----------|
| 闭包执行中（tx 未提交） | RunTransactionWithOptions 自动回滚，`appmigrations` 仍停留在 V4 | 重启后从 V5 开始正常执行 ✅ |
| 闭包已 return nil，框架正在 Commit | 取决于 Commit 是否成功：若 Commit 失败则自动回滚 | 重启后从 V5 开始正常执行 ✅ |
| `m.Migrate(db)` 已返回成功，但 `INSERT appmigrations` 之前 | DDL 已提交，但版本未记录 | 重启后会**再次执行 V5** ⚠️ |
| `INSERT appmigrations` 之后 | 版本已推进到 V5 | 重启后从 V6 开始正常执行 ✅ |

注意：**Builder 模式的闭包事务虽然降低了内部回滚遗漏的风险，但并未消除「迁移成功 + 版本记录失败」的不一致窗口**，因为 INSERT appmigrations 在事务外部。

### 5.3 已知风险点

#### 风险 1：非幂等迁移的重复执行

以下迁移在重复执行时会报错（无 IF NOT EXISTS 或错误吞掉逻辑）：

| 版本 | 操作 | 重复执行结果 | 采用机制 |
|------|------|-------------|---------|
| V3 | `ALTER TABLE users ADD COLUMN status` | MySQL: `Duplicate column name`；SQLite: 相同错误 | 手写模式 |
| V4 | `CREATE TABLE oauth_users` / `oauth_client_states` | `Table already exists`（Builder 显式关闭了 IfNotExists） | Builder 模式 |
| V5 | `ALTER TABLE ... ADD COLUMN provider/client_id/...` + `CREATE UNIQUE INDEX` | `Duplicate column name` / `Duplicate key name` | Builder 模式 |
| V6 | `ALTER TABLE remoteusers ADD COLUMN handle` | `Duplicate column name` | 手写模式 |
| V7 | `ALTER TABLE oauth_client_states ADD COLUMN attach_user_id` | `Duplicate column name` | Builder 模式 |
| V8 | `ALTER TABLE oauth_client_states ADD COLUMN invite_code` | `Duplicate column name` | Builder 模式 |
| V15 | `CREATE INDEX posts_get_collection_index` | `Duplicate key name` 或 `index already exists` | 手写模式 |
| V17 | `ALTER TABLE ... MODIFY post_signature` | MySQL 无报错（可重复 MODIFY） | 手写模式 + 驱动判断 |

> **重要发现**：采用 Builder 模式的 V4/V5/V7/V8 **同样缺少幂等性保护**。Builder 模式虽然提供了类型安全的 SQL 生成，但其 `SetIfNotExists` 在 V4 中被显式设为 `false`，而 `AlterTableSqlBuilder.AddColumn()` 也未提供「先检查列是否存在」的辅助方法。

#### 风险 2：MySQL DDL 的隐式提交

MySQL 中 `ALTER TABLE`、`CREATE INDEX` 等 DDL 语句会**隐式提交当前事务**。这意味着：

- 即使 `supportUserStatus()`（V3）中用了 `Begin()` / `Rollback()`，`ALTER TABLE` 执行成功后已无法回滚
- 即使 V4/V5/V7/V8 使用了 `RunTransactionWithOptions` 闭包，内部的 `ALTER TABLE` 也会提前结束事务
- 若后续操作（如 `Commit()`）失败，表结构已变更，但版本未记录 —— 下次会重试同样的操作并报错

#### 风险 3：事务回滚后的版本号未校验

迁移成功的标志是**同时满足**：
1. 数据库结构已变更
2. `appmigrations` 表有对应版本记录

当前代码仅以 `appmigrations.MAX(version)` 为准，不校验实际结构是否匹配。若人工修复数据库后跳过某些迁移，可能导致版本号与实际结构不一致。

#### 风险 4：result 字段未利用

`appmigrations.result` 字段始终写入空字符串 `""`，未用于存储：
- 迁移错误日志
- 迁移耗时
- 跳过/部分执行的标记

这不利于事后排查。

---

## 6. 抽象层（db/）与迁移层（migrations/）的实际复用关系

本节详细说明两套机制的交织关系，纠正「迁移层完全未使用 Builder」的不准确描述。

### 6.1 复用全景：4 个迁移 + 1 个业务函数

`db/` 包（SQL Builder 抽象层）提供了 6 类核心组件，其实际被上层复用的情况如下：

| 组件 | 定义位置 | 实际复用点 |
|------|---------|-----------|
| `DialectType` + `DialectMySQL/SQLite` | [db/dialect.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/db/dialect.go) | migrations/v4, v5, v7, v8 |
| `CreateTableSqlBuilder`（Table/Column/UniqueConstraint） | [db/create.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/db/create.go) | migrations/v4 |
| `AlterTableSqlBuilder`（AddColumn/ChangeColumn） | [db/alter.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/db/alter.go) | migrations/v5, v7, v8 |
| `CreateIndexSqlBuilder`（CreateUniqueIndex） | [db/index.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/db/index.go) | migrations/v5 |
| `RunTransactionWithOptions`（事务闭包） | [db/tx.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/db/tx.go) | migrations/v4, v5, v7, v8; business/ValidateOAuthState |
| `RawSqlBuilder` | [db/raw.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/db/raw.go) | 未被任何非测试代码使用 |
| `DropIndexSqlBuilder`（DropIndex） | [db/index.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/db/index.go) | 未被任何非测试代码使用 |

### 6.2 V4：首次完整采用 Builder 模式（建表场景）

[migrations/v4.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/v4.go) 是第一个引入 Builder 模式的迁移。

**方言选择桥接**（机制 A 与机制 B 的交汇点）：
```go
dialect := wf_db.DialectMySQL
if db.driverName == driverSQLite {   // ← 仍然使用机制 A 的 driverName 做判断
    dialect = wf_db.DialectSQLite    // ← 映射到机制 B 的 DialectType 枚举
}
```

**Builder 链式调用构建 SQL**：
```go
createTableUsersOauth, err := dialect.
    Table("oauth_users").
    SetIfNotExists(false).
    Column(dialect.Column("user_id", wf_db.ColumnTypeInteger, wf_db.UnsetSize)).
    Column(dialect.Column("remote_user_id", wf_db.ColumnTypeInteger, wf_db.UnsetSize)).
    ToSQL()
```

对比**手写模式（V1 建表）**的等价写法：
```go
_, err = t.Exec(`CREATE TABLE userinvites (
    id ` + db.typeChar(6) + ` NOT NULL,
    owner_id ` + db.typeInt() + ` NOT NULL,
    ...
) ` + db.engine() + `;`)
```

**关键差异**：
- Builder 模式自动完成类型映射（`ColumnTypeInteger` → MySQL `INT` / SQLite `INTEGER`）
- Builder 模式未生成 `ENGINE = InnoDB` 后缀（机制 A 的 `db.engine()` 覆盖了机制 B 未实现的部分）
- Builder 模式的编译期类型检查可以防止 `typeIntt()` 这类拼写错误

### 6.3 V5：最复杂的 Builder 复用场景（改表 + 索引 + 驱动分支）

[migrations/v5.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/v5.go) 展示了 Builder 模式的全部能力。

**批量构建 SQL**：使用 `[]wf_db.SQLBuilder` 切片收集多个语句，统一遍历执行：
```go
builders := []wf_db.SQLBuilder{
    // 1. 改表：ALTER TABLE oauth_client_states ADD COLUMN provider VARCHAR(24) DEFAULT ''
    dialect.AlterTable("oauth_client_states").
        AddColumn(dialect.Column("provider", wf_db.ColumnTypeVarChar,
            wf_db.OptionalInt{Set: true, Value: 24}).SetDefault("")),

    // ... 4 个更多的 AddColumn ...

    // 5. 创建唯一索引
    dialect.CreateUniqueIndex("oauth_users_uk", "oauth_users",
        "user_id", "provider", "client_id"),
}
```

**驱动内部分支 + Builder 混合使用**（针对 MySQL 独有的字段宽度调整）：
```go
if dialect != wf_db.DialectSQLite {
    builders = append(builders, dialect.
        AlterTable("oauth_users").
        ChangeColumn("remote_user_id",
            dialect.Column("remote_user_id", wf_db.ColumnTypeVarChar,
                wf_db.OptionalInt{Set: true, Value: 128})))
}
```

**统一执行循环**：
```go
for _, builder := range builders {
    query, err := builder.ToSQL()
    if err != nil { return err }
    if _, err := tx.ExecContext(ctx, query); err != nil { return err }
}
```

这种模式的优势在于：迁移逻辑的「结构描述」与「执行机制」完全分离，若后续需要新增执行钩子（如日志、耗时统计），仅需修改循环部分。

### 6.4 V7 / V8：简单改表场景的 Builder 复用

V7 和 V8 是单条 ALTER TABLE 的简单场景，同样采用 Builder 模式：

- [v7/oauthAttach()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/v7.go#L20-L46)：为 `oauth_client_states` 增加 `attach_user_id` 列（可空 INT）
- [v8/oauthInvites()](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/migrations/v8.go#L20-L45)：为 `oauth_client_states` 增加 `invite_code` 列（CHAR(6)，可空）

### 6.5 复用模式的演进趋势

从 V1 到 V17，可以观察到明显的演进轨迹：

```
V1-V3 (2019)  →  纯手写模式（drivers.go + Begin/Commit）
   ↓  转折点：OAuth 功能引入
V4-V8 (2019-2021) →  全面转向 Builder 模式（wf_db.Builder + RunTransactionWithOptions）
   ↓  未延续
V9-V17 (2021-2026) →  回归手写模式（drivers.go + Begin/Commit）
```

**V8 之后为何放弃 Builder？** 从代码中无法直接找到原因，但可以推测可能的因素：
1. 两套方言适配机制并存增加了学习成本，开发者需要同时理解 `db.typeInt()` 和 `wf_db.ColumnTypeInteger`
2. Builder 模式在处理驱动专属 SQL（如 MySQL 的 ENGINE、AFTER 子句）时需要额外补充字符串拼接
3. 手写模式对于简单的单列 ALTER / CREATE INDEX 场景代码量差异不明显
4. 缺少架构约束或 Code Review 时的规范检查

### 6.6 业务层的复用：仅事务执行器

业务代码（[database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database.go)）整体上对手写 SQL 的依赖非常强，**仅复用了 `db/` 包的 `RunTransactionWithOptions` 事务执行器**。

使用场景集中在 OAuth 相关的 `ValidateOAuthState()`（[database.go#L2955-L2985](file:///d:/fz/0601-1/solo-dogfeeding/code/36-writefreely/database.go#L2955-L2985)），对应读-改-校验的原子需求。而：
- 建表 / 改表场景为 0（业务层不做 DDL）
- Builder 的 Create/Alter/Index 组件未被业务层使用
- 业务层的方言差异仍然通过 `database.go` 内的 `now()/upsert()/clip()/dateAdd()/dateSub()` 等方法处理

### 6.7 重复维护成本分析

两套方言适配机制并存带来的实际维护成本：

| 维护点 | 机制 A (drivers.go) | 机制 B (db/create.go) | 是否重复 |
|--------|---------------------|----------------------|---------|
| INT 类型 | `typeInt() → INT` | `ColumnTypeInteger.Format() → INT` | ✅ 重复 |
| VARCHAR(l) | `typeVarChar(l)` | `ColumnTypeVarChar.Format()` | ✅ 重复 |
| BOOL 类型 | `typeBool() → TINYINT(1)` | `ColumnTypeBool.Format()` | ✅ 重复 |
| CHAR(l) | `typeChar(l)` | `ColumnTypeChar.Format()` | ✅ 重复 |
| SMALLINT | `typeSmallInt()` | `ColumnTypeSmallInt.Format()` | ✅ 重复 |
| DATETIME | `typeDateTime()` | `ColumnTypeDateTime.Format()` | ✅ 重复 |
| TEXT | `typeText()` | `ColumnTypeText.Format()` | ✅ 重复 |
| 默认时间戳 | （无对应方法） | `SetDefaultCurrentTimestamp()` | 互补 |
| now() 函数 | ✅ 提供 | ❌ 无 | 互补 |
| Upsert | ✅ 提供 | ❌ 无 | 互补 |
| ENGINE 子句 | ✅ 提供 | ❌ 无 | 互补 |

目前 **7 类数据类型映射完全重复定义**，而函数级差异为互补关系。理想的重构方向是让机制 A 的方法内部委托给机制 B 的 `ColumnType.Format()`，消除重复。

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
2. **双驱动支持完备**：编译期 + 运行期双层差异处理，覆盖类型、函数、错误、连接参数等维度
3. **版本模型简洁**：整数版本 + 线性数组，`CurrentVer() = len(migrations)` 设计优雅
4. **迁移粒度合理**：每个版本独立事务，单个失败不影响已完成版本
5. **Builder 模式已有实践**：V4/V5/V7/V8 证明了类型安全 Builder + 闭包事务的可行性，尤其是 V5 的批量 SQL 构建模式值得推广

### 改进建议

| 优先级 | 建议 | 对应风险/问题 |
|--------|------|--------------|
| 🔴 高 | 将迁移的 `m.Migrate()` + `INSERT appmigrations` 合并到同一事务中（需注意 MySQL DDL 隐式提交问题） | §5.3 风险 1、2 |
| 🔴 高 | 为 `ALTER TABLE ADD COLUMN` 和 `CREATE INDEX` 类迁移增加「先检查后执行」逻辑（不论是手写模式还是 Builder 模式），确保幂等 | §5.3 风险 1 |
| 🔴 高 | 消除方言适配的重复定义：让 `migrations/drivers.go` 的 `typeInt()/typeVarChar()` 等方法内部委托给 `wf_db.ColumnType.Format()` | §6.7 重复维护 |
| 🟡 中 | 统一事务管理：手写模式的 13 个迁移逐步迁移到 `RunTransactionWithOptions`，减少 `if err { t.Rollback() }` 的重复样板代码 | §3.4 |
| 🟡 中 | 明确新迁移的规范：后续迁移统一采用 Builder + 闭包事务模式，并在 `AlterTableSqlBuilder` 中补充 `IfNotExists` 辅助方法 | §6.5 演进趋势 |
| 🟡 中 | 业务代码的复杂事务场景（如 `CreateUser` 多表操作）统一使用 `RunTransactionWithOptions`，减少手写 Begin/Rollback 的遗漏 | §3.3 |
| 🟢 低 | 利用 `appmigrations.result` 字段记录迁移详情（耗时、错误信息、Builder/手写 模式标记） | §5.3 风险 4 |
| 🟢 低 | 在 `Migrate()` 完成后增加结构一致性校验（可选） | §5.3 风险 3 |
| 🟢 低 | 补充 `db/` Builder 的 Engine/Collate/After 子句 API，使其能完全替代 `migrations/drivers.go` 的字符串拼接能力 | §6.7 互补关系 |
