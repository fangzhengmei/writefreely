# WriteFreely OAuth 登录与账号绑定代码理解

## 一、整体架构概览

WriteFreely 的 OAuth 系统采用 **Provider 接口抽象 + 数据库持久化状态** 的设计模式。核心文件分布如下：

| 文件 | 职责 |
|------|------|
| [oauth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go) | 主流程控制：初始化、回调、路由注册、通用登录逻辑 |
| [oauth_signup.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_signup.go) | OAuth 新用户注册与表单校验 |
| [oauth/state.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth/state.go) | State 管理接口定义 |
| [oauth_generic.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_generic.go) | 通用 OAuth Provider 实现（可配置字段映射） |
| [oauth_gitlab.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_gitlab.go) | GitLab Provider |
| [oauth_gitea.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_gitea.go) | Gitea Provider |
| [oauth_slack.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_slack.go) | Slack Provider |
| [oauth_writeas.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_writeas.go) | Write.as Provider |
| [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/database.go#L2944-L3067) | OAuth State 生成/校验、远程用户 ID 关联的数据库实现 |
| [config/config.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/config/config.go#L68-L120) | 各 Provider 的配置结构定义 |

---

## 二、State 校验机制

### 2.1 State 生成流程

State 的生成发生在 OAuth 登录初始化阶段 [viewOauthInit](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L133-L164)：

```go
func (h oauthHandler) viewOauthInit(app *App, w http.ResponseWriter, r *http.Request) error {
    ctx := r.Context()

    var attachUser int64
    if attach := r.URL.Query().Get("attach"); attach == "t" {
        user, _ := getUserAndSession(app, r)
        if user == nil {
            return impart.HTTPError{http.StatusInternalServerError, "..."}
        }
        attachUser = user.ID
    }

    state, err := h.DB.GenerateOAuthState(ctx, 
        h.oauthClient.GetProvider(), 
        h.oauthClient.GetClientID(), 
        attachUser, 
        r.FormValue("invite_code"))
    // ...
}
```

**关键点：**
- 当 URL 参数 `attach=t` 时，表示当前用户希望将 OAuth 账号绑定到已登录的本地账号，此时 `attachUser` 存储当前用户 ID
- `invite_code` 支持邀请码注册场景
- State 通过数据库持久化，而非仅存在 Session 中

### 2.2 State 数据库存储

数据库表 `oauth_client_states` 的写入逻辑在 [GenerateOAuthState](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/database.go#L2944-L2953)：

```go
func (db *datastore) GenerateOAuthState(ctx context.Context, provider string, clientID string, attachUser int64, inviteCode string) (string, error) {
    state := id.Generate62RandomString(24)
    attachUserVal := sql.NullInt64{Valid: attachUser > 0, Int64: attachUser}
    inviteCodeVal := sql.NullString{Valid: inviteCode != "", String: inviteCode}
    _, err := db.ExecContext(ctx, 
        "INSERT INTO oauth_client_states (state, provider, client_id, used, created_at, attach_user_id, invite_code) VALUES (?, ?, ?, FALSE, "+db.now()+", ?, ?)",
        state, provider, clientID, attachUserVal, inviteCodeVal)
    // ...
}
```

**State 表字段：**
- `state`: 24 位随机字符串（62 进制字符集）
- `provider`: OAuth 提供商标识（如 `gitlab`、`generic`）
- `client_id`: OAuth 应用的 Client ID
- `used`: 布尔标记，防止 State 被重放使用
- `created_at`: 创建时间
- `attach_user_id`: 关联绑定的本地用户 ID（可空）
- `invite_code`: 邀请码（可空，v8 迁移新增）

### 2.3 State 校验与一次性消费

在回调处理 [viewOauthCallback](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L323-L425) 中，首先进行 State 校验：

```go
provider, clientID, attachUserID, inviteCode, err := h.DB.ValidateOAuthState(ctx, state)
```

[ValidateOAuthState](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/database.go#L2955-L2985) 的实现采用 **事务 + 行级更新** 来保证一次性消费：

```go
func (db *datastore) ValidateOAuthState(ctx context.Context, state string) (string, string, int64, string, error) {
    var provider, clientID string
    var attachUserID sql.NullInt64
    var inviteCode sql.NullString
    
    err := wf_db.RunTransactionWithOptions(ctx, db.DB, &sql.TxOptions{}, func(ctx context.Context, tx *sql.Tx) error {
        // 1. 查询未使用的 state
        err := tx.QueryRowContext(ctx, 
            "SELECT provider, client_id, attach_user_id, invite_code FROM oauth_client_states WHERE state = ? AND used = FALSE", state).
            Scan(&provider, &clientID, &attachUserID, &inviteCode)
        if err != nil {
            return err
        }

        // 2. 标记 state 为已使用
        res, err := tx.ExecContext(ctx, "UPDATE oauth_client_states SET used = TRUE WHERE state = ?", state)
        // ...
        rowsAffected, _ := res.RowsAffected()
        if rowsAffected != 1 {
            return fmt.Errorf("state not found")
        }
        return nil
    })
    // ...
}
```

**安全特性：**
1. **一次性消费**：`used` 字段从 FALSE 更新为 TRUE，通过 `rowsAffected != 1` 检测并发重放
2. **事务隔离**：整个查询+更新在事务内完成，防止竞态条件
3. **上下文透传**：使用 `ExecContext` / `QueryRowContext` 支持请求取消

**潜在问题：**
- `err != nil` 时返回空字符串和 nil error（第 2982 行），这会导致上层无法区分"state 无效"和"校验成功但值为空"的情况，属于错误处理缺陷

---

## 三、用户创建逻辑

### 3.1 OAuth 回调中的用户决策流程

在 [viewOauthCallback](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L323-L425) 中，完成 State 校验后，通过远程用户 ID 查询本地关联关系：

```go
// 步骤1: 用 code 换 access_token
tokenResponse, err := h.oauthClient.exchangeOauthCode(ctx, code)

// 步骤2: 用 access_token 获取用户信息（含 remote UserID）
tokenInfo, err := h.oauthClient.inspectOauthAccessToken(ctx, tokenResponse.AccessToken)

// 步骤3: 查询该远程用户是否已绑定本地账号
localUserID, err := h.DB.GetIDForRemoteUser(ctx, tokenInfo.UserID, provider, clientID)
```

[GetIDForRemoteUser](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/database.go#L3001-L3011) 从 `oauth_users` 表查询：

```go
func (db *datastore) GetIDForRemoteUser(ctx context.Context, remoteUserID, provider, clientID string) (int64, error) {
    var userID int64 = -1
    err := db.QueryRowContext(ctx, 
        "SELECT user_id FROM oauth_users WHERE remote_user_id = ? AND provider = ? AND client_id = ?", 
        remoteUserID, provider, clientID).Scan(&userID)
    if err != nil && err != sql.ErrNoRows {
        return -1, err
    }
    return userID, nil  // 未找到返回 -1
}
```

### 3.2 三种分支场景

根据 `localUserID` 和 `attachUserID` 的组合，回调处理分为三条路径：

```
                      ┌─────────────────────────┐
                      │  localUserID != -1 ?    │
                      └───────────┬─────────────┘
                                  │
                     ┌────────────┴─────────────┐
                     │ 是                       │ 否
                     ▼                          ▼
          ┌────────────────────┐      ┌─────────────────────────┐
          │ attachUserID > 0 ? │      │ attachUserID > 0 ?      │
          └───────────┬────────┘      └────────────┬────────────┘
                      │                              │
             ┌────────┴─────────┐           ┌───────┴──────────┐
             │ 是               │ 否        │ 是               │ 否
             ▼                  ▼           ▼                  ▼
    ┌────────────────┐  ┌──────────────┐  ┌───────────┐  ┌────────────┐
    │ 已绑定其他用户 │  │ 直接登录     │  │ 绑定到    │  │ 跳转到注册 │
    │ 返回错误      │  │ 已有用户     │  │ 当前用户  │  │ 页面       │
    └────────────────┘  └──────────────┘  └───────────┘  └────────────┘
```

**分支1：远程用户已绑定本地账号**（第 361-380 行）
- 如果同时 `attachUserID > 0`（即用户想绑定到另一个账号）：返回错误 "This OAuth account is already attached to another user."
- 否则：查询本地用户信息，直接调用 `loginOrFail` 设置 Session 登录

**分支2：远程用户未绑定，但当前在绑定模式**（第 381-389 行）
```go
if attachUserID > 0 {
    err = h.DB.RecordRemoteUserID(r.Context(), attachUserID, tokenInfo.UserID, provider, clientID, tokenResponse.AccessToken)
    // 跳转到 /me/settings
}
```

**分支3：全新用户注册**（第 391-424 行）
- 先检查邀请码有效性或开放注册配置
- 构造 `oauthSignupPageParams`，用 HashSeed 对参数签名，防止篡改
- 展示注册页面 `signup-oauth.tmpl`，让用户确认/修改用户名、显示名、邮箱

### 3.3 OAuth 用户创建实现

注册表单提交到 [viewOauthSignup](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_signup.go#L90-L153)：

```go
func (h oauthHandler) viewOauthSignup(app *App, w http.ResponseWriter, r *http.Request) error {
    tp := &oauthSignupPageParams{...}
    
    // 1. 校验签名，防止注册参数被篡改
    if tp.HashTokenParams(h.Config.Server.HashSeed) != r.FormValue(oauthParamHash) {
        return impart.HTTPError{Status: http.StatusBadRequest, Message: "Request has been tampered with."}
    }
    
    // 2. 校验用户输入（用户名长度、邮箱格式）
    if err := h.validateOauthSignup(r); err != nil { ... }
    
    // 3. 可选：密码哈希（OAuth 用户可不设密码）
    var hashedPass []byte
    clearPass := r.FormValue(oauthParamPassword)
    if clearPass != "" {
        hashedPass, err = auth.HashPass([]byte(clearPass))
    }
    
    // 4. 创建本地用户（users + collections 表）
    newUser := &User{...}
    err = h.DB.CreateUser(h.Config, newUser, displayName, "")
    
    // 5. 记录邀请使用（如有）
    if tp.InviteCode != "" {
        err = app.db.CreateInvitedUser(tp.InviteCode, newUser.ID)
    }
    
    // 6. 关联远程用户 ID 到本地用户
    err = h.DB.RecordRemoteUserID(r.Context(), newUser.ID, 
        r.FormValue(oauthParamTokenRemoteUserID), 
        r.FormValue(oauthParamProvider), 
        r.FormValue(oauthParamClientID), 
        r.FormValue(oauthParamAccessToken))
    
    // 7. 自动登录
    loginOrFail(h.Store, w, r, newUser)
}
```

**参数防篡改签名** [HashTokenParams](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_signup.go#L77-L88)：
```go
func (p oauthSignupPageParams) HashTokenParams(key string) string {
    hasher := sha256.New()
    hasher.Write([]byte(key))
    hasher.Write([]byte(p.AccessToken))
    hasher.Write([]byte(p.TokenUsername))
    hasher.Write([]byte(p.TokenAlias))
    hasher.Write([]byte(p.TokenEmail))
    hasher.Write([]byte(p.TokenRemoteUser))
    hasher.Write([]byte(p.ClientID))
    hasher.Write([]byte(p.Provider))
    return hex.EncodeToString(hasher.Sum(nil))
}
```

使用服务端 `HashSeed` 对 OAuth 返回的核心参数做 SHA-256 哈希，确保注册表单提交时这些参数未被前端篡改。

### 3.4 远程用户 ID 记录

[RecordRemoteUserID](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/database.go#L2987-L2998) 写入 `oauth_users` 表：

```go
func (db *datastore) RecordRemoteUserID(ctx context.Context, localUserID int64, remoteUserID, provider, clientID, accessToken string) error {
    if db.driverName == driverSQLite {
        _, err = db.ExecContext(ctx, 
            "INSERT OR REPLACE INTO oauth_users (user_id, remote_user_id, provider, client_id, access_token) VALUES (?, ?, ?, ?, ?)",
            localUserID, remoteUserID, provider, clientID, accessToken)
    } else {
        _, err = db.ExecContext(ctx, 
            "INSERT INTO oauth_users (user_id, remote_user_id, provider, client_id, access_token) VALUES (?, ?, ?, ?, ?) "+
            db.upsert("user")+" access_token = ?",
            localUserID, remoteUserID, provider, clientID, accessToken, accessToken)
    }
}
```

**`oauth_users` 表结构（由 [V4](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/migrations/v4.go#L20-L53) + [V5](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/migrations/v5.go#L20-L87) 迁移定义）：**
- `user_id`: 本地用户 ID（INTEGER）
- `remote_user_id`: OAuth Provider 返回的用户唯一标识（VARCHAR(128)，V4 初始为 INTEGER，V5 改为 VARCHAR）
- `provider`: Provider 名称（VARCHAR(24)，V5 新增）
- `client_id`: OAuth 应用 Client ID（VARCHAR(128)，V5 新增）
- `access_token`: OAuth Access Token（TEXT，V11 从 VARCHAR(512) 扩宽）

**数据库唯一索引约束（V5 迁移建立）：**
在 [v5.go:62](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/migrations/v5.go#L62) 创建了唯一索引：

```go
dialect.CreateUniqueIndex("oauth_users_uk", "oauth_users", "user_id", "provider", "client_id")
```

生成的 SQL 为：
```sql
CREATE UNIQUE INDEX oauth_users_uk ON oauth_users (user_id, provider, client_id)
```

⚠️ **关键发现：唯一索引不包含 `remote_user_id`**

这意味着：
- **一个本地用户** + **一个 provider** + **一个 client_id** → 只能有 **一条** 绑定记录（正确，防止同一本地用户重复绑定同一 Provider 同一应用）
- 但数据库层面 **不保证** 同一个 `remote_user_id` 只能绑定到一个本地用户——这完全依赖应用层逻辑（[viewOauthCallback](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L355-L366) 中的检查）

**Upsert 行为分析：**

[upsert](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/database.go#L174-L182) 函数定义：
```go
func (db *datastore) upsert(indexedCols ...string) string {
    if db.driverName == driverSQLite {
        cc := strings.Join(indexedCols, ", ")
        return "ON CONFLICT(" + cc + ") DO UPDATE SET"
    }
    return "ON DUPLICATE KEY UPDATE"
}
```

- **SQLite 分支**：使用 `INSERT OR REPLACE`，由 `INSERT OR REPLACE` 的语义决定（遇到任何 UNIQUE 约束冲突即替换整行）
- **MySQL 分支**：调用 `db.upsert("user")` 传入 `"user"`，生成 `ON DUPLICATE KEY UPDATE`——注意 `"user"` 参数在 MySQL 分支中 **完全未使用**！MySQL 的 `ON DUPLICATE KEY UPDATE` 会检测任意唯一索引冲突，而不仅限于某列

### 3.5 邀请码在注册确认页的回传与签名覆盖分析

#### 3.5.1 邀请码的传递链路

邀请码的完整流转路径：

```
用户带 invite_code 访问登录入口 /oauth/gitlab?invite_code=ABC123
    ↓
viewOauthInit() 取 r.FormValue("invite_code")
    ↓
GenerateOAuthState(..., invite_code="ABC123")  → 存入 oauth_client_states.invite_code
    ↓ (用户授权后回调)
viewOauthCallback()
  ├─ ValidateOAuthState() → 从 state 中恢复出 inviteCode
  ├─ 校验邀请码有效性（GetUserInvite + i.Active）
  ├─ 构造 oauthSignupPageParams{ InviteCode: "ABC123" }
  └─ showOauthSignupPage() 渲染模板
    ↓
signup-oauth.tmpl 渲染隐藏字段
  {{if .InviteCode}}<input type="hidden" name="invite_code" value="{{ .InviteCode }}" />{{end}}
    ↓ (用户提交表单 POST /oauth/signup)
viewOauthSignup() 读取 r.FormValue("invite_code")
  ├─ 签名校验（但 InviteCode 不在签名范围内 → 见下文）
  └─ 调用 CreateInvitedUser(tp.InviteCode, newUser.ID) 记录使用
```

#### 3.5.2 签名覆盖范围的精确分析

[HashTokenParams](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_signup.go#L77-L88) 的实现：

```go
type oauthSignupPageParams struct {
    AccessToken     string   // ✅ 参与签名
    TokenUsername   string   // ✅ 参与签名
    TokenAlias      string   // ✅ 参与签名
    TokenEmail      string   // ✅ 参与签名
    TokenRemoteUser string   // ✅ 参与签名
    ClientID        string   // ✅ 参与签名
    Provider        string   // ✅ 参与签名
    TokenHash       string   // ❌ 签名结果本身
    InviteCode      string   // ❌ 未参与签名 —— 关键漏洞
}

func (p oauthSignupPageParams) HashTokenParams(key string) string {
    hasher := sha256.New()
    hasher.Write([]byte(key))                // HashSeed
    hasher.Write([]byte(p.AccessToken))      // 写入
    hasher.Write([]byte(p.TokenUsername))    // 写入
    hasher.Write([]byte(p.TokenAlias))       // 写入
    hasher.Write([]byte(p.TokenEmail))       // 写入
    hasher.Write([]byte(p.TokenRemoteUser))  // 写入
    hasher.Write([]byte(p.ClientID))         // 写入
    hasher.Write([]byte(p.Provider))         // 写入
    // ⚠️  hasher.Write([]byte(p.InviteCode))  —— 缺失！
    return hex.EncodeToString(hasher.Sum(nil))
}
```

**签名覆盖字段清单：**

| 字段 | 参与签名 | 表单回传 | 篡改风险 |
|------|---------|---------|---------|
| AccessToken | ✅ 是 | ✅ `<input type="hidden" name="access_token">` | 被签名保护 |
| TokenUsername | ✅ 是 | ✅ `token_username` | 被签名保护 |
| TokenAlias | ✅ 是 | ✅ `token_alias` | 被签名保护 |
| TokenEmail | ✅ 是 | ✅ `token_email` | 被签名保护 |
| TokenRemoteUser | ✅ 是 | ✅ `token_remote_user` | 被签名保护 |
| ClientID | ✅ 是 | ✅ `client_id` | 被签名保护 |
| Provider | ✅ 是 | ✅ `provider` | 被签名保护 |
| **InviteCode** | **❌ 否** | ✅ `invite_code`（条件渲染） | **可被篡改** |
| username（用户输入） | ❌ 否 | ✅ 用户填写 | 设计上允许修改 |
| alias（显示名） | ❌ 否 | ✅ 用户填写 | 设计上允许修改 |
| email（邮箱） | ❌ 否 | ✅ 用户填写 | 设计上允许修改 |
| password（密码） | ❌ 否 | ✅ 用户填写 | 设计上允许修改 |

#### 3.5.3 邀请码篡改的具体影响

由于 `InviteCode` **不在签名计算范围内**，攻击者可以：

**攻击场景（1）—— 移除邀请码绕过邀请制限制：**

1. 管理员开启邀请码注册（`open_registration = false`），攻击者通过某个 OAuth 链接获取了合法邀请码 `INVITE_VALID`
2. 到达注册确认页后，攻击者通过浏览器 DevTools **删除** 表单中 `<input type="hidden" name="invite_code" value="INVITE_VALID">`
3. 提交时：
   - `viewOauthSignup` 重新计算签名 → `InviteCode=""`，但签名是用 `InviteCode="INVITE_VALID"` 生成的？
   - **不，等等**：`viewOauthSignup` 从表单读取 `InviteCode`（现在为空），然后用**当前**的参数（包括空邀请码）计算签名
   - 但表单中的 `signature` 值是回调时生成的，当时 `InviteCode="INVITE_VALID"`

   **关键问题反转了**：由于签名 **不包含** `InviteCode`，不管 `InviteCode` 是 `INVITE_VALID` 还是空串，计算出的哈希值 **完全相同**！

4. 所以攻击可行：
   - 移除邀请码 → `tp.InviteCode = ""`
   - 重新计算哈希：由于 `InviteCode` 不写入 hasher，哈希值与原始签名 **匹配**
   - 签名校验通过！
   - `CreateInvitedUser` 因 `tp.InviteCode == ""` 跳过
   - **结果**：在 `open_registration = false` 的邀请码注册模式下，攻击者无需有效邀请码即可注册账号

**攻击场景（2）—— 替换他人邀请码：**

攻击者可以将邀请码替换为任意已知邀请码（例如一个还未使用的高价邀请码），以此消耗他人的邀请配额，因为签名不会检测到邀请码的改变。

**攻击场景（3）—— 在回调阶段之后篡改：**

即使 `viewOauthCallback` 中对邀请码做了有效性校验（[oauth.go:393-401](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L393-L401)），但该检查仅发生在回调入口。从回调返回到用户提交注册表单之间可能间隔数小时/数天，期间：
- 邀请码可能已被使用/过期
- 但因为表单中的邀请码 **可以被自由替换为另一个有效邀请码**（签名不检测），所以攻击者可以等待某个邀请码失效后，将其换为新获取的邀请码，而无需重新走 OAuth 流程

#### 3.5.4 其他字段可被篡改的影响

用户名、显示名、邮箱是设计上允许用户修改的，不在签名保护范围内是正确的。但需注意：

- `password` 字段同样在签名外，但 OAuth 注册流程中密码是可选的（用户可选择不设置密码，后续仅通过 OAuth 登录）
- 如果管理员开启了 `disable_password_auth = true`，密码字段无意义
- 但如果没有开启，攻击者理论上可以通过篡改表单给自己设置密码，获得密码登录能力（即便 OAuth 解绑也能登录）

---

## 四、Provider 分派机制

### 4.1 oauthClient 接口抽象

所有 Provider 实现统一的 [oauthClient](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L109-L116) 接口：

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

五种实现：
| Provider | GetProvider() 返回值 | 文件 |
|----------|---------------------|------|
| Slack | `"slack"` | [oauth_slack.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_slack.go) |
| Write.as | `"write.as"` | [oauth_writeas.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_writeas.go) |
| GitLab | `"gitlab"` | [oauth_gitlab.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_gitlab.go) |
| Gitea | `"gitea"` | [oauth_gitea.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_gitea.go) |
| Generic | `"generic"` | [oauth_generic.go](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_generic.go) |

### 4.2 路由注册与分派

在 [configureOauthRoutes](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L309-L321) 中，每个 Provider 注册独立的路由：

```go
func configureOauthRoutes(parentHandler *Handler, r *mux.Router, app *App, oauthClient oauthClient, callbackProxy *callbackProxyClient) {
    handler := &oauthHandler{
        Config:        app.Config(),
        DB:            app.DB(),
        Store:         app.SessionStore(),
        oauthClient:   oauthClient,  // 每个 Provider 独享自己的 oauthClient 实例
        callbackProxy: callbackProxy,
    }
    r.HandleFunc("/oauth/"+oauthClient.GetProvider(), parentHandler.OAuth(handler.viewOauthInit)).Methods("GET")
    r.HandleFunc("/oauth/callback/"+oauthClient.GetProvider(), parentHandler.OAuth(handler.viewOauthCallback)).Methods("GET")
    r.HandleFunc("/oauth/signup", parentHandler.OAuth(handler.viewOauthSignup)).Methods("POST")
}
```

路由示例：
- `GET /oauth/gitlab` → GitLab 登录初始化
- `GET /oauth/callback/gitlab` → GitLab 回调
- `GET /oauth/slack` → Slack 登录初始化
- `GET /oauth/callback/slack` → Slack 回调

每个 `configure*Oauth` 函数（如 [configureGitlabOauth](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L217-L243)）根据配置是否启用（`ClientID != ""`）来决定是否注册路由。

### 4.3 Generic Provider 的字段映射

[genericOauthClient](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_generic.go#L24-L37) 支持通过配置自定义 JSON 字段映射：

```go
type genericOauthCfg struct {
    MapUserID        string // 默认 "user_id"
    MapUsername      string // 默认 "username"
    MapDisplayName   string // 默认 "-" （表示忽略）
    MapEmail         string // 默认 "email"
}
```

在 [inspectOauthAccessToken](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth_generic.go#L108-L145) 中通过反射从 JSON 响应中取值：

```go
var genericInterface map[string]interface{}
limitedJsonUnmarshal(resp.Body, infoRequestMaxLen, &genericInterface)

var inspectResponse InspectResponse
inspectResponse.UserID, _ = genericInterface[c.MapUserID].(string)
inspectResponse.Username, _ = genericInterface[c.MapUsername].(string)
inspectResponse.DisplayName, _ = genericInterface[c.MapDisplayName].(string)
inspectResponse.Email, _ = genericInterface[c.MapEmail].(string)
```

Gitea Provider 也采用相同的字段映射模式，在 [configureGiteaOauth](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L277-L307) 中硬编码了映射值：

```go
oauthClient := giteaOauthClient{
    Scope:         "openid profile email",
    MapUserID:     "sub",
    MapUsername:   "login",
    MapDisplayName:"full_name",
    MapEmail:      "email",
}
```

---

## 五、账号合并风险分析

### 5.1 数据模型层面的风险

`oauth_users` 表通过 `(remote_user_id, provider, client_id)` 三元组唯一标识一个远程 OAuth 用户，关联到单个本地 `user_id`。

**风险点 1：同一本地用户可绑定多个不同 Provider 的账号**
- 这是设计上允许的功能，但如果不同 Provider 返回的 `remote_user_id` 意外相同（例如自托管 Generic OAuth 配置错误），会导致用户 A 的 OAuth 登录被路由到用户 B 的账号

**风险点 2：MySQL upsert 的唯一索引问题**
- 在 [RecordRemoteUserID](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/database.go#L2987-L2998) 中，MySQL 分支使用 `db.upsert("user")`，该函数展开为 `ON DUPLICATE KEY UPDATE`
- 如果 `oauth_users` 表的唯一索引仅建立在 `user_id` 上而非 `(remote_user_id, provider, client_id)` 组合上，则：
  - 同一本地用户尝试绑定同一远程账号时，只更新 access_token（预期行为）
  - 但无法防止不同本地用户绑定到同一个远程账号（应返回冲突错误）

**风险点 3：ValidateOAuthState 的错误处理缺陷**
- 在第 2982 行，`err != nil` 时返回 `("", "", 0, "", nil)`，丢弃了原始错误
- 上层 [viewOauthCallback](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L329-L333) 会得到 nil error，但 `provider` 和 `clientID` 为空字符串
- 后续 `GetIDForRemoteUser` 使用空字符串查询，可能导致匹配到意外记录

### 5.2 绑定流程中的风险

**场景：用户尝试将已被绑定的 OAuth 账号绑定到自己**

在 [viewOauthCallback 第 361-366 行](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L361-L366)：

```go
if localUserID != -1 && attachUserID > 0 {
    // localUserID 是已绑定的用户，attachUserID 是当前登录想绑定的用户
    if localUserID != attachUserID {
        addSessionFlash(app, w, r, "This OAuth account is already attached to another user.", nil)
        return impart.HTTPError{http.StatusFound, "/me/settings"}
    }
    // 如果 localUserID == attachUserID，代码继续执行到第 381 行，会再次 RecordRemoteUserID（即更新 token）
}
```

这里逻辑正确，但存在一个 **信息泄露风险**：攻击者可以通过枚举 `attach=t` 参数绑定已知的 OAuth 账号，根据返回的 flash 消息判断该 OAuth 账号是否已在系统中注册。

### 5.3 注册流程防篡改风险

OAuth 注册流程中，AccessToken、RemoteUserID 等敏感参数通过 HTML 表单隐藏字段传递，依赖 `HashTokenParams` 签名保护。

**风险分析：**
- 签名使用 SHA-256(key + 所有字段拼接)，无随机盐，属于确定性 MAC
- 如果 `HashSeed` 泄露，攻击者可以伪造任意注册参数
- 未包含时间戳或过期机制，签名永不过期，获取到签名后的 URL 可永久使用

### 5.4 Callback Proxy 的安全考量

系统支持通过 Callback Proxy 接收 OAuth 回调（适用于内网部署场景）。在 [viewOauthInit](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L151-L156)：

```go
if h.callbackProxy != nil {
    if err := h.callbackProxy.register(ctx, state); err != nil {
        // ...
    }
}
```

[callbackProxyClient.register](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/oauth.go#L427-L448) 将 state 和回调地址发送给代理服务器：

```go
func (r *callbackProxyClient) register(ctx context.Context, state string) error {
    form := url.Values{}
    form.Add("state", state)
    form.Add("location", r.callbackLocation)
    // POST 到代理服务器
    resp, err := r.httpClient.Do(req)
    if resp.StatusCode != http.StatusCreated {
        return fmt.Errorf("unable register state location: %d", resp.StatusCode)
    }
}
```

**风险：** 代理服务器与 WriteFreely 之间的通信没有身份认证机制。如果代理服务器地址被篡改或中间人攻击，攻击者可将 state 关联到恶意回调地址，进而窃取 OAuth code。

### 5.5 Access Token 存储风险

[RecordRemoteUserID](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/database.go#L2987-L2998) 将 OAuth Access Token **明文存储** 在 `oauth_users.access_token` 字段中：

```go
// access_token 直接存入数据库，无加密
_, err = db.ExecContext(ctx, "INSERT ... access_token = ?", ..., accessToken)
```

如果数据库被拖库，所有用户的第三方 OAuth Access Token 将直接泄露，攻击者可利用这些 token 访问用户在对应 Provider 上的资源（取决于 token scope）。

### 5.6 账号解绑与锁定风险

[removeOauth](file:///d:/fz/0601-1/solo-dogfeeding/code/35-writefreely/account.go#L1525-L1536) 允许用户解除 OAuth 绑定：

```go
func removeOauth(app *App, u *User, w http.ResponseWriter, r *http.Request) error {
    provider := r.FormValue("provider")
    clientID := r.FormValue("client_id")
    remoteUserID := r.FormValue("remote_user_id")
    err := app.db.RemoveOauth(r.Context(), u.ID, provider, clientID, remoteUserID)
}
```

结合 `DisablePasswordAuth` 配置场景：
- 如果管理员开启了 `[app] disable_password_auth = true`，且用户唯一的登录方式是 OAuth
- 用户误操作解绑所有 OAuth 账号后，将 **无法再登录** 系统
- 虽然 `viewLogout` 中有类似保护逻辑（检查 email/password），但解绑流程中无此防护

---

## 六、关键调用链总结

### 6.1 新用户 OAuth 注册完整流程

```
用户点击 /oauth/gitlab
    ↓
viewOauthInit() [oauth.go:133]
  ├─ GenerateOAuthState() → 写入 oauth_client_states
  └─ 302 重定向到 GitLab 授权页
    ↓ (用户在 GitLab 授权后)
GitLab 回调 /oauth/callback/gitlab?code=xxx&state=xxx
    ↓
viewOauthCallback() [oauth.go:323]
  ├─ ValidateOAuthState() → 标记 state 已使用，获取 attachUserID / inviteCode
  ├─ exchangeOauthCode() → code → access_token
  ├─ inspectOauthAccessToken() → 获取 remote_user_id, username, email
  ├─ GetIDForRemoteUser() → 返回 -1（新用户）
  ├─ attachUserID == 0，inviteCode 校验通过
  ├─ 构造 oauthSignupPageParams + HashTokenParams 签名
  └─ 渲染 signup-oauth.tmpl 页面（带隐藏表单字段）
    ↓ (用户填写用户名后提交 POST /oauth/signup)
viewOauthSignup() [oauth_signup.go:90]
  ├─ HashTokenParams 校验 → 防篡改
  ├─ validateOauthSignup() → 用户名/邮箱格式校验
  ├─ CreateUser() → 写入 users + collections 表
  ├─ RecordRemoteUserID() → 写入 oauth_users 表
  └─ loginOrFail() → 设置 Session Cookie → 302 到首页
```

### 6.2 已登录用户绑定 OAuth 账号流程

```
已登录用户访问 /oauth/gitlab?attach=t
    ↓
viewOauthInit() [oauth.go:133]
  ├─ getUserAndSession() → 获取当前用户 ID = 123
  ├─ GenerateOAuthState(attachUser=123) → state 中记录待绑定用户
  └─ 302 重定向到 GitLab 授权页
    ↓
viewOauthCallback() [oauth.go:323]
  ├─ ValidateOAuthState() → 返回 attachUserID=123
  ├─ ... 获取 remote_user_id ...
  ├─ GetIDForRemoteUser() → 返回 -1（该 GitLab 账号未绑定过）
  └─ RecordRemoteUserID(localUserID=123, remoteUserID=...) → 建立关联
```

---

## 七、改进建议

| 风险点 | 建议 |
|--------|------|
| ValidateOAuthState 错误丢弃 | 返回错误而非 nil，确保上层能正确处理 state 失效 |
| oauth_users 唯一索引 | 确保数据库存在 `UNIQUE(remote_user_id, provider, client_id)` 约束 |
| Access Token 明文存储 | 使用应用级别密钥加密存储，或仅存储 refresh_token |
| HashTokenParams 无过期 | 在签名中加入时间戳，设置注册链接有效期（如 15 分钟） |
| Callback Proxy 无认证 | 配置 Proxy 共享密钥，请求时加入 HMAC 签名 |
| 解绑后无法登录 | removeOauth 前检查 `DoesUserNeedAuth`，防止用户失去所有登录方式 |
| 绑定冲突信息泄露 | 统一返回模糊错误信息，不暴露 "该 OAuth 已被绑定" |
