# WriteFreely OAuth 登录与账号绑定代码理解

> 本文档所有结论均来自精确代码分析，每一项断言都有对应的代码引用。

---

## 一、核心数据模型与数据库约束

首先从数据库迁移代码出发，明确客观存在的表结构和约束——这是所有上层逻辑的基础。

### 1.1 表结构定义

两张核心表的结构由多次数据库迁移累积定义：

| 表名 | 迁移版本 | 关键新增内容 |
|------|---------|-------------|
| `oauth_client_states` | V4 ([v4.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/migrations/v4.go#L35-L42)) | 初始表：`state`, `used`, `created_at` |
| ↳ 扩展字段 | V5 ([v5.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/migrations/v5.go#L28-L40)) | 新增 `provider`, `client_id` |
| ↳ 扩展字段 | V7 ([v7.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/migrations/v7.go#L28-L33)) | 新增 `attach_user_id` |
| ↳ 扩展字段 | V8 ([v8.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/migrations/v8.go)) | 新增 `invite_code` |
| `oauth_users` | V4 ([v4.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/migrations/v4.go#L26-L31)) | 初始表：`user_id`, `remote_user_id`（INTEGER） |
| ↳ 扩展字段 | V5 ([v5.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/migrations/v5.go#L42-L61)) | 新增 `provider`, `client_id`, `access_token`；`remote_user_id` 改为 VARCHAR(128) |
| ↳ 扩宽字段 | V11 ([v11.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/migrations/v11.go#L24)) | `access_token` 从 VARCHAR(512) 扩为 TEXT |

### 1.2 数据库层唯一索引约束（客观事实）

**唯一索引 `oauth_users_uk`** 在 V5 迁移中创建 [v5.go:62](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/migrations/v5.go#L62)：

```go
dialect.CreateUniqueIndex("oauth_users_uk", "oauth_users", "user_id", "provider", "client_id")
```

生成的 SQL：
```sql
CREATE UNIQUE INDEX oauth_users_uk ON oauth_users (user_id, provider, client_id)
```

**索引 `oauth_users_uk` 的精确含义：**

✅ **有** 数据库级约束：同一个 `user_id` + 同一个 `provider` + 同一个 `client_id` → 只能有 **一条** 绑定记录

❌ **没有** 数据库级约束：
- 同一个 `remote_user_id` 不能对应多个 `user_id`
- 同一个 `(remote_user_id, provider, client_id)` 不能出现多次
- `(user_id, remote_user_id)` 组合唯一

**`oauth_client_states` 表唯一索引** 在 V4 迁移中创建 [v4.go:41](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/migrations/v4.go#L41)：
```go
UniqueConstraint("state")
```
→  `state` 字段全局唯一，确保同一个 state 不会被创建两次。

---

## 二、State 校验机制

### 2.1 State 生成 [GenerateOAuthState](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/database.go#L2944-L2953)

```go
func (db *datastore) GenerateOAuthState(ctx context.Context, 
    provider string, clientID string, 
    attachUser int64, inviteCode string) (string, error) {
    state := id.Generate62RandomString(24)
    attachUserVal := sql.NullInt64{Valid: attachUser > 0, Int64: attachUser}
    inviteCodeVal := sql.NullString{Valid: inviteCode != "", String: inviteCode}
    _, err := db.ExecContext(ctx, 
        "INSERT INTO oauth_client_states (state, provider, client_id, used, created_at, attach_user_id, invite_code) VALUES (?, ?, ?, FALSE, "+db.now()+", ?, ?)",
        state, provider, clientID, attachUserVal, inviteCodeVal)
    // ...
}
```

**State 生成时机与参数携带：**
- `attachUser > 0`：当前用户希望将 OAuth 账号**绑定**到自己（通过 `?attach=t` URL 参数触发）
- `inviteCode != ""`：通过邀请码注册场景（通过 `?invite_code=XXX` URL 参数传入）
- 这两个参数存入数据库，**不是** 放在 state 字符串中编码

### 2.2 State 校验 [ValidateOAuthState](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/database.go#L2955-L2985)

```go
func (db *datastore) ValidateOAuthState(ctx context.Context, state string) (string, string, int64, string, error) {
    var provider, clientID string
    var attachUserID sql.NullInt64
    var inviteCode sql.NullString
    
    err := wf_db.RunTransactionWithOptions(ctx, db.DB, &sql.TxOptions{}, func(ctx context.Context, tx *sql.Tx) error {
        // 步骤1: 查询未使用的 state
        err := tx.QueryRowContext(ctx, 
            "SELECT provider, client_id, attach_user_id, invite_code FROM oauth_client_states WHERE state = ? AND used = FALSE", 
            state).Scan(&provider, &clientID, &attachUserID, &inviteCode)
        if err != nil {
            return err
        }
        // 步骤2: 标记为已使用
        res, err := tx.ExecContext(ctx, "UPDATE oauth_client_states SET used = TRUE WHERE state = ?", state)
        rowsAffected, _ := res.RowsAffected()
        if rowsAffected != 1 {
            return fmt.Errorf("state not found")
        }
        return nil
    })
    if err != nil {
        return "", "", 0, "", nil   // ⚠️ 缺陷：错误被丢弃，返回 nil error
    }
    return provider, clientID, attachUserID.Int64, inviteCode.String, nil
}
```

**安全特性：**
1. **事务内查询+更新**：避免竞态条件
2. **一次性消费**：`used` 字段从 FALSE → TRUE，通过 `rowsAffected != 1` 检测重放
3. **参数回传**：从数据库中恢复 `attachUserID` 和 `inviteCode`

**缺陷（代码第 2982 行）：**
校验失败时返回 `("", "", 0, "", nil)`，错误被丢弃。上层得到 nil error 但拿到空字符串，可能引发后续逻辑异常。

---

## 三、Provider 分派机制

### 3.1 oauthClient 接口抽象 [oauth.go:109-L116](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L109-L116)

```go
type oauthClient interface {
    GetProvider() string
    GetClientID() string
    GetCallbackLocation() string
    buildLoginURL(state string) (string, error)
    exchangeOauthCode(ctx context.Context, code string) (*TokenResponse, error)
    inspectOauthAccessToken(ctx context.Context, accessToken string) (*InspectResponse, error)
}
```

### 3.2 五种 Provider 实现

| Provider | GetProvider() | 文件 | 特有机制 |
|----------|---------------|------|---------|
| GitLab | `"gitlab"` | [oauth_gitlab.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_gitlab.go) | 硬编码 GitLab 端点 |
| Gitea | `"gitea"` | [oauth_gitea.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_gitea.go) | 字段映射硬编码：`sub`→UserID, `login`→Username, `full_name`→DisplayName |
| Slack | `"slack"` | [oauth_slack.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_slack.go) | Slack OAuth v2 流程 |
| Write.as | `"write.as"` | [oauth_writeas.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_writeas.go) | Write.as 自有 OAuth |
| Generic | `"generic"` | [oauth_generic.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_generic.go) | 配置驱动的字段映射（`map_user_id`、`map_username` 等） |

### 3.3 路由注册

初始化流程在 [routes.go:79-L83](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/routes.go#L79-L83) 按固定顺序调用 5 个配置函数（只在对应 Provider 的 `ClientID != ""` 时真正生效）：

```go
configureSlackOauth(handler, write, apper.App())    // 第 1 个
configureWriteAsOauth(handler, write, apper.App())  // 第 2 个
configureGitlabOauth(handler, write, apper.App())   // 第 3 个
configureGenericOauth(handler, write, apper.App())  // 第 4 个
configureGiteaOauth(handler, write, apper.App())    // 第 5 个
```

每个配置函数内部调用 [configureOauthRoutes](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L309-L321) 注册路由：

```go
func configureOauthRoutes(parentHandler *Handler, r *mux.Router, 
    app *App, oauthClient oauthClient, callbackProxy *callbackProxyClient) {
    handler := &oauthHandler{
        Config:        app.Config(),
        DB:            app.DB(),
        Store:         app.SessionStore(),
        oauthClient:   oauthClient,
        callbackProxy: callbackProxy,
        EmailKey:      app.keys.EmailKey,
    }
    // 路由 1：登录入口（每个 provider 路径不同，无冲突）
    r.HandleFunc("/oauth/"+oauthClient.GetProvider(), 
        parentHandler.OAuth(handler.viewOauthInit)).Methods("GET")
    // 路由 2：回调地址（每个 provider 路径不同，无冲突）
    r.HandleFunc("/oauth/callback/"+oauthClient.GetProvider(), 
        parentHandler.OAuth(handler.viewOauthCallback)).Methods("GET")
    // 路由 3：注册提交（⚠️ 所有 provider 路径相同 → 重复注册）
    r.HandleFunc("/oauth/signup", 
        parentHandler.OAuth(handler.viewOauthSignup)).Methods("POST")
}
```

**路由汇总表：**

| 路由 | 路径 | 注册次数 | 说明 |
|------|------|---------|------|
| 登录入口 | `/oauth/slack`、`/oauth/write.as`、`/oauth/gitlab`、`/oauth/generic`、`/oauth/gitea` | 各 1 次 | 路径中包含 `GetProvider()`，互不冲突 |
| OAuth 回调 | `/oauth/callback/slack`、`/oauth/callback/write.as` 等 | 各 1 次 | 路径中包含 `GetProvider()`，互不冲突 |
| 注册提交 | **`POST /oauth/signup`** | **每个启用的 Provider 各注册 1 次** | 所有 Provider 使用**同一路径**，存在**多次注册** |

**`POST /oauth/signup` 多次注册的行为：**

由于 `configureOauthRoutes` 在每个启用的 Provider 配置中都会执行 `r.HandleFunc("/oauth/signup", ...)`，同一路径被重复注册给 gorilla/mux。其行为是：
- mux 不会报错，允许同一路径+方法的多次注册
- 匹配请求时按 **注册顺序** 找到第一个匹配的 handler
- 也就是：**最先启用的 Provider（按 Slack → Write.as → GitLab → Generic → Gitea 顺序）的 `oauthHandler` 实例会处理所有的 `/oauth/signup` 请求**

但由于 `viewOauthSignup` 处理函数**不使用** `oauthClient` 字段（所有 provider 信息都从表单隐藏字段 `provider` / `client_id` 回传），所以无论由哪个 handler 实例处理，结果都一致——这是代码能正常工作的原因。

---

## 四、远端账号归属约束：数据库层 vs 应用层

> **核心澄清**：`oauth_users` 表的归属约束通过**数据库唯一索引**和**应用层先查后插**的**双层机制**实现，两层保护的维度**不同**。

### 4.1 数据库层负责的约束

数据库唯一索引 `oauth_users_uk(user_id, provider, client_id)` 保证：

```
一个本地用户 ←(唯一关联)→ 一个 Provider 的一个 Client ID
```

即：用户 A 不能把同一个 GitLab 应用（app1）绑定两次，但可以绑定 GitLab app1 和 GitLab app2（不同 client_id），也可以绑定 GitLab app1 和 Gitea app1（不同 provider）。

### 4.2 应用层负责的约束

由于数据库层**不保证** `(remote_user_id, provider, client_id)` 唯一，这个约束完全由应用层 [viewOauthCallback](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L323-L425) 中的逻辑保证。

**应用层约束的核心代码（第 355-389 行）：**

```go
// 第一步：用 remote_user_id 查询是否已绑定
localUserID, err := h.DB.GetIDForRemoteUser(ctx, 
    tokenInfo.UserID,    // OAuth Provider 返回的远程用户唯一标识
    provider, 
    clientID)

// 第二步：根据查询结果分支处理
if localUserID != -1 && attachUserID > 0 {
    // 分支1：远程账号已绑定到其他用户 → 返回冲突
    if localUserID != attachUserID {
        addSessionFlash(app, w, r, "This OAuth account is already attached to another user.", nil)
        return impart.HTTPError{http.StatusFound, "/me/settings"}
    }
    // 子情况：localUserID == attachUserID（重复绑定，继续执行会更新 access_token）
}

if localUserID != -1 {
    // 分支2：远程账号已绑定 → 直接以该本地用户身份登录
    user, _ := h.DB.GetUserByID(localUserID)
    loginOrFail(h.Store, w, r, user)
    return nil
}

if attachUserID > 0 {
    // 分支3：远程账号未绑定 + 当前已登录 → 绑定到当前用户
    err = h.DB.RecordRemoteUserID(ctx, attachUserID, tokenInfo.UserID, provider, clientID, ...)
    return impart.HTTPError{http.StatusFound, "/me/settings"}
}

// 分支4：远程账号未绑定 + 未登录 → 进入注册流程
```

**[GetIDForRemoteUser](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/database.go#L3001-L3011) 的实现：**

```go
func (db *datastore) GetIDForRemoteUser(ctx context.Context, 
    remoteUserID, provider, clientID string) (int64, error) {
    var userID int64 = -1
    err := db.QueryRowContext(ctx, 
        "SELECT user_id FROM oauth_users "+
        "WHERE remote_user_id = ? AND provider = ? AND client_id = ?",
        remoteUserID, provider, clientID).Scan(&userID)
    if err != nil && err != sql.ErrNoRows {
        return -1, err
    }
    return userID, nil
}
```

**应用层约束的精确语义：**

同一个 `(remote_user_id, provider, client_id)` → 只能关联到 **一个** `user_id`

如果数据库中已存在该三元组 → 后续所有使用该远程账号的 OAuth 操作（登录或绑定）都会被路由到这个已关联的本地用户。

### 4.3 约束组合表

| 场景 | 数据库层 | 应用层 | 最终结果 |
|------|---------|--------|---------|
| 用户A 绑定 GitLab(app1) 账号X | 插入 `(A, gitlab, app1, X)` ✓ | 查询 X → 未绑定过 ✓ | 成功 |
| 用户A 再次绑定 GitLab(app1) 账号X | `oauth_users_uk` 冲突 → upsert 更新 `access_token` | 查询 X → 返回 A；`localUserID==attachUserID` → 正常继续 | 刷新 token 成功 |
| 用户B 绑定 GitLab(app1) 账号X | 若先插入 `(B, gitlab, app1, X)` → **不冲突**（`oauth_users_uk` 只看 `(B, gitlab, app1)`）⚠️ | **但插入前**查询 X → 返回 A → 返回冲突错误 ✓ | 被应用层挡住 |
| 用户A 绑定 GitLab(app2) 账号X | 插入 `(A, gitlab, app2, X)` ✓ | 查询 X+app2 → 未绑定过 ✓ | 成功（不同 client_id 视为不同身份） |
| 用户A 绑定 Gitea(app1) 账号X | 插入 `(A, gitea, app1, X)` ✓ | 查询 X+gitea → 未绑定过 ✓ | 成功（不同 provider 视为不同身份） |

### 4.4 [RecordRemoteUserID](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/database.go#L2987-L2998) 的 upsert 行为

```go
func (db *datastore) RecordRemoteUserID(ctx context.Context, 
    localUserID int64, remoteUserID, provider, clientID, accessToken string) error {
    if db.driverName == driverSQLite {
        _, err = db.ExecContext(ctx, 
            "INSERT OR REPLACE INTO oauth_users "+
            "(user_id, remote_user_id, provider, client_id, access_token) "+
            "VALUES (?, ?, ?, ?, ?)",
            localUserID, remoteUserID, provider, clientID, accessToken)
    } else {
        _, err = db.ExecContext(ctx, 
            "INSERT INTO oauth_users "+
            "(user_id, remote_user_id, provider, client_id, access_token) "+
            "VALUES (?, ?, ?, ?, ?) "+
            db.upsert("user")+" access_token = ?",
            localUserID, remoteUserID, provider, clientID, accessToken, accessToken)
    }
}
```

**[upsert](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/database.go#L174-L182) 的实现：**

```go
func (db *datastore) upsert(indexedCols ...string) string {
    if db.driverName == driverSQLite {
        cc := strings.Join(indexedCols, ", ")
        return "ON CONFLICT(" + cc + ") DO UPDATE SET"
    }
    return "ON DUPLICATE KEY UPDATE"
}
```

**upsert 行为的精确分析：**

| 数据库 | 语句 | 冲突检测依据 |
|--------|------|-------------|
| **SQLite** | `INSERT OR REPLACE` | 遇到任意 UNIQUE 约束冲突（包括 `oauth_users_uk`）→ 删除旧行插入新行 |
| **MySQL** | `ON DUPLICATE KEY UPDATE` | 遇到任意唯一索引冲突 → 更新 `access_token`。`db.upsert("user")` 传入的 `"user"` 参数在 MySQL 分支**完全未使用**，直接返回通用的 `ON DUPLICATE KEY UPDATE` |

注意：代码中调用 `db.upsert("user")` 期望按 `user_id` 列冲突时 upsert，但 MySQL 的 `ON DUPLICATE KEY UPDATE` 不支持指定冲突列——它检测**所有**唯一索引的冲突。

---

## 五、OAuth 回调全流程分析

### 5.1 回调处理主逻辑 [viewOauthCallback](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L323-L425)

完整处理步骤：

```
viewOauthCallback()
  │
  ├─ ① ValidateOAuthState(state) → 一次性消费 state
  │     恢复出 provider, clientID, attachUserID, inviteCode
  │
  ├─ ② exchangeOauthCode(code) → OAuth code → access_token
  │
  ├─ ③ inspectOauthAccessToken(access_token) → 获取远程用户信息
  │     返回 tokenInfo { UserID, Username, DisplayName, Email }
  │
  ├─ ④ GetIDForRemoteUser(tokenInfo.UserID, provider, clientID)
  │     查询本地绑定关系 → localUserID
  │
  ├─ ⑤ 归属约束检查与分派（参见 4.2 节）
  │     ├─ 分支1：已绑定 + 正在绑定其他用户 → 冲突错误
  │     ├─ 分支2：已绑定 + 新登录 → 直接登录
  │     ├─ 分支3：未绑定 + 正在绑定 → 建立绑定
  │     └─ 分支4：未绑定 + 新用户 → 进入注册流程
  │
  └─ ⑥ 注册准备（分支4）
        ├─ 邀请码校验（如有）
        ├─ 构造 oauthSignupPageParams
        ├─ HashTokenParams 签名
        └─ 渲染注册确认页
```

### 5.2 分支4：注册确认页参数传递

在 [oauth.go:412-L422](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L412-L422) 构造注册参数：

```go
tp := &oauthSignupPageParams{
    AccessToken:     tokenResponse.AccessToken,
    TokenUsername:   tokenInfo.Username,
    TokenAlias:      tokenInfo.DisplayName,
    TokenEmail:      tokenInfo.Email,
    TokenRemoteUser: tokenInfo.UserID,
    Provider:        provider,
    ClientID:        clientID,
    InviteCode:      inviteCode,    // 从 state 中恢复的邀请码
}
tp.TokenHash = tp.HashTokenParams(h.Config.Server.HashSeed)
```

模板 [signup-oauth.tmpl:66-L75](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/pages/signup-oauth.tmpl#L66-L75) 将这些参数渲染为隐藏表单字段：

```html
<input type="hidden" name="access_token" value="{{ .AccessToken }}" />
<input type="hidden" name="token_username" value="{{ .TokenUsername }}" />
<input type="hidden" name="token_alias" value="{{ .TokenAlias }}" />
<input type="hidden" name="token_email" value="{{ .TokenEmail }}" />
<input type="hidden" name="token_remote_user" value="{{ .TokenRemoteUser }}" />
<input type="hidden" name="provider" value="{{ .Provider }}" />
<input type="hidden" name="client_id" value="{{ .ClientID }}" />
<input type="hidden" name="signature" value="{{ .TokenHash }}" />
{{if .InviteCode}}<input type="hidden" name="invite_code" value="{{ .InviteCode }}" />{{end}}
```

### 5.3 注册提交处理 [viewOauthSignup](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_signup.go#L90-L153)

```go
func (h oauthHandler) viewOauthSignup(app *App, w http.ResponseWriter, r *http.Request) error {
    tp := &oauthSignupPageParams{
        AccessToken:     r.FormValue("access_token"),
        TokenUsername:   r.FormValue("token_username"),
        TokenAlias:      r.FormValue("token_alias"),
        TokenEmail:      r.FormValue("token_email"),
        TokenRemoteUser: r.FormValue("token_remote_user"),
        ClientID:        r.FormValue("client_id"),
        Provider:        r.FormValue("provider"),
        InviteCode:      r.FormValue("invite_code"),   // 从表单读取
    }
    // 签名校验
    if tp.HashTokenParams(h.Config.Server.HashSeed) != r.FormValue("signature") {
        return impart.HTTPError{http.StatusBadRequest, "Request has been tampered with."}
    }
    // ... 校验用户输入 ...
    // ... 创建用户 ...
    // ... 记录邀请使用（tp.InviteCode != "" 时）...
    // ... 关联远程用户 ID ...
    loginOrFail(h.Store, w, r, newUser)
}
```

---

## 六、签名机制与邀请码回传分析

### 6.1 [HashTokenParams](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_signup.go#L77-L88) 的精确实现

```go
func (p oauthSignupPageParams) HashTokenParams(key string) string {
    hasher := sha256.New()
    hasher.Write([]byte(key))              // 1. 服务端密钥 HashSeed
    hasher.Write([]byte(p.AccessToken))    // 2. OAuth Access Token
    hasher.Write([]byte(p.TokenUsername))  // 3. OAuth 返回的用户名
    hasher.Write([]byte(p.TokenAlias))     // 4. OAuth 返回的显示名
    hasher.Write([]byte(p.TokenEmail))     // 5. OAuth 返回的邮箱
    hasher.Write([]byte(p.TokenRemoteUser))// 6. OAuth 返回的远程用户 ID
    hasher.Write([]byte(p.ClientID))       // 7. OAuth 应用 Client ID
    hasher.Write([]byte(p.Provider))       // 8. Provider 名称
    // ⚠️  hasher.Write([]byte(p.InviteCode))  —— 缺失！
    return hex.EncodeToString(hasher.Sum(nil))
}
```

### 6.2 签名覆盖范围清单

| 字段 | 参与签名 | 表单回传 | 说明 |
|------|---------|---------|------|
| AccessToken | ✅ 是 | ✅ 隐藏字段 | OAuth 凭证，必须保护 |
| TokenUsername | ✅ 是 | ✅ 隐藏字段 | OAuth 用户名，必须保护 |
| TokenAlias | ✅ 是 | ✅ 隐藏字段 | OAuth 显示名，必须保护 |
| TokenEmail | ✅ 是 | ✅ 隐藏字段 | OAuth 邮箱，必须保护 |
| TokenRemoteUser | ✅ 是 | ✅ 隐藏字段 | 远程用户唯一标识，核心关联键，必须保护 |
| ClientID | ✅ 是 | ✅ 隐藏字段 | OAuth 应用标识，必须保护 |
| Provider | ✅ 是 | ✅ 隐藏字段 | Provider 标识，必须保护 |
| **InviteCode** | **❌ 否** | ✅ 条件隐藏字段 | 邀请码，**不在签名范围内** |
| username | ❌ 否 | ❌ 用户输入 | 设计上允许用户修改 |
| alias | ❌ 否 | ❌ 用户输入 | 设计上允许用户修改 |
| email | ❌ 否 | ❌ 用户输入 | 设计上允许用户修改 |
| password | ❌ 否 | ❌ 用户输入 | 设计上允许用户设置密码 |

### 6.3 InviteCode 未被签名保护的具体影响

由于 `InviteCode` 不在签名计算范围内，**签名值与 `InviteCode` 无关**——修改、移除或替换 `InviteCode` 都不会导致签名校验失败。

**攻击场景 1：移除邀请码绕过邀请制**

前提：管理员关闭开放注册（`open_registration = false`），仅允许邀请码注册。

1. 攻击者通过某个 OAuth 链接获取了合法邀请码 `INVITE_A`
2. 到达注册确认页后，通过浏览器 DevTools **删除** `<input type="hidden" name="invite_code" value="INVITE_A">`
3. 提交表单：
   - `tp.InviteCode = ""`（从表单读取为空）
   - `tp.HashTokenParams(HashSeed)` 重新计算签名
   - 由于 `InviteCode` 不写入哈希，**新签名与表单中的旧签名完全相同**
   - 签名校验通过 ✅
   - `CreateInvitedUser` 因 `tp.InviteCode == ""` 被跳过
   - **结果**：攻击者无需有效邀请码即可注册账号

**攻击场景 2：替换邀请码消耗他人配额**

攻击者将表单中的邀请码替换为任意已知的有效邀请码（如获取到的他人高价邀请码），由于签名不检测，替换成功后该邀请码会被消耗。

**攻击场景 3：邀请码过期后替换**

邀请码校验仅发生在回调入口 [oauth.go:393-L401](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L393-L401)：
```go
if inviteCode != "" {
    i, err := app.db.GetUserInvite(inviteCode)
    if !i.Active(app.db) {
        return "Invite link has expired."
    }
}
```

从回调返回到用户提交表单可能间隔很久。如果期间邀请码过期，攻击者可以将表单中的邀请码替换为新获取的有效邀请码，无需重新走 OAuth 流程。

### 6.4 邀请码的完整传递链路

```
用户访问 /oauth/gitlab?invite_code=ABC123
    ↓
viewOauthInit() 读取 r.FormValue("invite_code")
    ↓
GenerateOAuthState(..., invite_code="ABC123")
  → 写入 oauth_client_states.invite_code
    ↓ (用户授权后回调)
viewOauthCallback()
  ├─ ValidateOAuthState() → 从数据库恢复 inviteCode="ABC123"
  ├─ 校验邀请码有效性（GetUserInvite + Active）
  ├─ 构造 oauthSignupPageParams{ InviteCode: "ABC123" }
  └─ 签名（不包含 InviteCode）
    ↓
signup-oauth.tmpl 渲染：
  {{if .InviteCode}}<input type="hidden" name="invite_code" value="ABC123">{{end}}
    ↓ (用户提交 POST /oauth/signup)
viewOauthSignup()
  ├─ 从表单读取 invite_code（可被篡改）
  ├─ 签名校验（不检测 invite_code 变化）
  └─ CreateInvitedUser(invite_code, newUser.ID)
```

---

## 七、风险分析（每项均有代码依据）

### 7.1 数据模型层风险

**风险 1：缺少 `UNIQUE(remote_user_id, provider, client_id)` 索引 → TOCTOU 竞态漏洞**

- **代码依据**：数据库唯一索引是 `oauth_users_uk(user_id, provider, client_id)` [v5.go:62](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/migrations/v5.go#L62)，不包含 `remote_user_id`
- **为什么成立**：应用层约束通过"先查 `GetIDForRemoteUser` → 后插 `RecordRemoteUserID`"实现，但查询和插入不在同一个事务中，也没有行锁保护
- **攻击场景**：用户 A 和 B 同时发起同一个远程账号 X 的绑定流程
  1. 两者并发执行 `GetIDForRemoteUser(X)` → 都返回 -1（X 尚未绑定）
  2. 两者都通过应用层检查
  3. 两者并发执行 `RecordRemoteUserID`：
     - 用户 A 插入 `(user=A, remote=X, gitlab, app1)` → `oauth_users_uk` 不冲突 ✓
     - 用户 B 插入 `(user=B, remote=X, gitlab, app1)` → `oauth_users_uk` 不冲突 ✓
  4. **结果**：同一个远程 OAuth 账号 X 同时绑定到两个不同的本地用户
- **后续影响**：后续 `GetIDForRemoteUser(X)` 查询不带 `LIMIT 1`，返回哪条记录取决于存储引擎，登录身份不确定

**风险 2：`ValidateOAuthState` 错误被丢弃**

- **代码依据**：[database.go:2981-L2982](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/database.go#L2981-L2982)
  ```go
  if err != nil {
      return "", "", 0, "", nil   // 返回 nil error，错误被吞
  }
  ```
- **为什么成立**：state 校验失败（如 state 不存在、已被使用、或数据库错误）时，调用方得到 nil error 但拿到空字符串
- **后果**：上层 `viewOauthCallback` 使用空字符串的 `provider` 和 `clientID` 调用 `GetIDForRemoteUser`，可能产生意外的查询结果

**风险 3：Access Token 明文存储**

- **代码依据**：[database.go:2987-L2998](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/database.go#L2987-L2998)，`access_token` 字段直接写入数据库，无加密
- **为什么成立**：没有任何加密逻辑，直接以明文存储
- **后果**：数据库泄露时，所有用户的第三方 OAuth Access Token 全部泄露

### 7.2 注册流程风险

**风险 4：InviteCode 不在签名覆盖范围内 → 可绕过邀请制**

- **代码依据**：[HashTokenParams](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_signup.go#L77-L88) 中没有写入 `p.InviteCode`
- **为什么成立**：签名计算的输入中不包含 `InviteCode`，修改该字段不会改变签名值，因此签名校验无法检测
- **具体攻击**：参见 6.3 节

**风险 5：签名无过期机制 → 可永久使用**

- **代码依据**：`HashTokenParams` 不包含时间戳或过期字段
- **为什么成立**：只要参数值不变，哈希值永远不变
- **后果**：获取到的签名 HTML 可以永久使用，没有 15 分钟/1 小时等时效性保护

### 7.3 绑定流程风险

**风险 6：绑定冲突信息泄露 → 可探测 OAuth 账号是否已注册**

- **代码依据**：[oauth.go:361-L366](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L361-L366)
  ```go
  if localUserID != -1 && attachUserID > 0 {
      addSessionFlash(app, w, r, "This OAuth account is already attached to another user.", nil)
      return impart.HTTPError{http.StatusFound, "/me/settings"}
  }
  ```
- **为什么成立**：错误消息明确告知"该 OAuth 账号已被其他用户绑定"，与其他错误场景返回的消息不同
- **后果**：攻击者可以枚举 `?attach=t` 参数，通过返回的 flash 消息判断某个 OAuth 账号是否已在系统中注册

**风险 7：解绑后用户可能无法登录**

- **代码依据**：[removeOauth](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/account.go#L1525-L1536) 解绑前不检查用户是否还有其他登录方式
- **为什么成立**：如果管理员开启 `disable_password_auth = true`，且用户唯一登录方式是 OAuth，解绑所有 OAuth 账号后用户将无法登录
- **后果**：用户自我锁定，需要管理员干预

### 7.4 架构层风险

**风险 8：Callback Proxy 无身份认证**

- **代码依据**：[callbackProxyClient.register](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L427-L448) 仅做 HTTP POST，无签名或认证
- **为什么成立**：请求中没有包含任何只有代理服务器和 WriteFreely 知道的共享密钥
- **后果**：攻击者篡改代理服务器地址或进行中间人攻击，可将 state 关联到恶意回调地址窃取 OAuth code

---

## 八、改进建议（按优先级）

| 优先级 | 风险点 | 修复建议 | 代码参考 |
|--------|--------|---------|---------|
| **Critical** | InviteCode 不在签名覆盖范围内 | 在 `HashTokenParams` 中加入 `hasher.Write([]byte(p.InviteCode))` | [oauth_signup.go:77-L88](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_signup.go#L77-L88) |
| **Critical** | 缺少 `UNIQUE(remote_user_id, provider, client_id)` 索引 | 新增 V18 迁移创建该唯一索引；`GetIDForRemoteUser` 查询加 `LIMIT 1` | [migrations/v5.go:62](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/migrations/v5.go#L62) |
| **High** | TOCTOU 竞态漏洞 | 将 `GetIDForRemoteUser` + `RecordRemoteUserID` 包裹在事务中，使用 `SELECT ... FOR UPDATE`（MySQL）或 `SERIALIZABLE` 隔离级别（SQLite） | [oauth.go:355-L389](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L355-L389) |
| **High** | `ValidateOAuthState` 错误丢弃 | 将 `return "", "", 0, "", nil` 改为 `return "", "", 0, "", err`，错误透传 | [database.go:2981-L2982](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/database.go#L2981-L2982) |
| **High** | Access Token 明文存储 | 使用应用级密钥（如 `HashSeed` 派生的 AES 密钥）对 `access_token` 加密后存储 | [database.go:2987-L2998](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/database.go#L2987-L2998) |
| **Medium** | MySQL upsert 参数被忽略 | `db.upsert("user")` 中的 `"user"` 对 MySQL 无效，清理为 `db.upsert()` 或注释说明 | [database.go:2992](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/database.go#L2992) |
| **Medium** | 签名无过期机制 | 在 `oauthSignupPageParams` 中增加 `Timestamp int64` 字段，加入签名计算；校验时拒绝超过 15 分钟的请求 | [oauth_signup.go:65-L88](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_signup.go#L65-L88) |
| **Medium** | Callback Proxy 无认证 | 配置 Proxy 共享密钥，请求时加入 `X-Signature: HMAC-SHA256(secret, state+location)` | [oauth.go:427-L448](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L427-L448) |
| **Medium** | 解绑后用户锁定 | `removeOauth` 前检查：若 `disable_password_auth=true` 且用户仅剩最后一个 OAuth 绑定则拒绝 | [account.go:1525-L1536](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/account.go#L1525-L1536) |
| **Low** | 绑定冲突信息泄露 | 统一返回模糊错误消息，如"无法完成绑定"，不区分具体冲突原因 | [oauth.go:361-L366](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L361-L366) |

---

## 九、关键调用链总结

### 9.1 新用户 OAuth 注册流程

```
用户带 invite_code 访问 /oauth/gitlab?invite_code=ABC123
    ↓
viewOauthInit() [oauth.go:133]
  ├─ getUserAndSession()（attach=t 时）
  ├─ GenerateOAuthState(provider, clientID, attachUser, inviteCode)
  │   → 写入 oauth_client_states
  └─ 302 重定向到 GitLab 授权页
    ↓ (用户授权)
GitLab 回调 /oauth/callback/gitlab?code=xxx&state=xxx
    ↓
viewOauthCallback() [oauth.go:323]
  ├─ ValidateOAuthState(state)
  │   → 事务内 SELECT + UPDATE used=TRUE
  │   → 恢复 provider, clientID, attachUserID, inviteCode
  ├─ exchangeOauthCode(code) → access_token
  ├─ inspectOauthAccessToken(access_token) → tokenInfo{UserID, ...}
  ├─ GetIDForRemoteUser(UserID, provider, clientID) → localUserID=-1（新用户）
  ├─ 归属约束检查（分支4：新用户注册）
  ├─ 邀请码有效性校验（GetUserInvite + Active）
  ├─ 构造 oauthSignupPageParams + HashTokenParams 签名
  │   ⚠️  签名不包含 InviteCode
  └─ 渲染 signup-oauth.tmpl
    ↓ (用户填写用户名，提交 POST /oauth/signup)
viewOauthSignup() [oauth_signup.go:90]
  ├─ 从表单读取所有隐藏字段 + 用户输入
  ├─ HashTokenParams 校验（不检测 InviteCode 变化）
  ├─ validateOauthSignup() 校验用户名/邮箱
  ├─ CreateUser() → 写入 users + collections
  ├─ CreateInvitedUser(InviteCode, newUser.ID)（InviteCode != "" 时）
  ├─ RecordRemoteUserID(newUser.ID, UserID, provider, clientID, access_token)
  │   → 写入 oauth_users 表
  └─ loginOrFail() → 设置 Session Cookie → 302 到首页
```

### 9.2 已登录用户绑定 OAuth 账号流程

```
已登录用户访问 /oauth/gitlab?attach=t
    ↓
viewOauthInit() [oauth.go:133]
  ├─ getUserAndSession() → 当前用户 ID = 123
  ├─ GenerateOAuthState(..., attachUser=123)
  └─ 302 重定向到 GitLab 授权页
    ↓
viewOauthCallback() [oauth.go:323]
  ├─ ValidateOAuthState() → attachUserID=123
  ├─ ... → 获取 tokenInfo.UserID = X
  ├─ GetIDForRemoteUser(X) → localUserID=-1（未绑定过）
  ├─ 归属约束检查（分支3：绑定到当前用户）
  └─ RecordRemoteUserID(123, X, ...) → 建立绑定
```

### 9.3 已有用户 OAuth 登录流程

```
用户访问 /oauth/gitlab
    ↓
viewOauthCallback() [oauth.go:323]
  ├─ ... → 获取 tokenInfo.UserID = X
  ├─ GetIDForRemoteUser(X) → localUserID=456（已绑定到用户 456）
  ├─ 归属约束检查（分支2：已有用户登录）
  ├─ GetUserByID(456) → 用户对象
  └─ loginOrFail() → 以用户 456 身份登录
```
