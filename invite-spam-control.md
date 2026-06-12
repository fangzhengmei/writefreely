# WriteFreely 邀请注册风控机制代码分析

## 一、整体架构概览

WriteFreely 的邀请注册风控系统由以下四个核心模块协作构成：

| 模块 | 核心职责 | 主要文件 |
|------|----------|----------|
| 邀请额度控制 | 限制邀请链接的使用次数和有效期 | [invites.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/invites.go) |
| 实例配置 | 控制谁可以创建邀请（全局开关） | [config/config.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/config/config.go)、[admin.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/admin.go) |
| 邮箱风控 | 邮箱标准化、蜜罐反垃圾 | [spam/email.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/spam/email.go) |
| IP 风控 | IP 获取、登录频率限制 | [spam/ip.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/spam/ip.go)、[account.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/account.go) |

辅助风控：用户静默状态（User Silenced）贯穿多个入口。

---

## 二、邀请额度控制（Invite Limit）

### 2.1 数据结构

定义在 [invites.go#L27-L35](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/invites.go#L27-L35)：

```go
type Invite struct {
    ID       string
    MaxUses  sql.NullInt64  // 最大使用次数，NULL 表示无限制
    Created  time.Time      // 创建时间
    Expires  *time.Time     // 过期时间，nil 表示永不过期
    Inactive bool           // 是否手动停用

    uses int64              // 已使用次数（非持久化字段，运行时计算）
}
```

数据库表：
- `userinvites`：存储邀请基本信息（id, owner_id, max_uses, created, expires, inactive）
- `usersinvited`：存储邀请码与被邀请用户的关联关系（invite_id, user_id）

### 2.2 有效性判断

定义在 [invites.go#L41-L55](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/invites.go#L41-L55)：

```go
func (i Invite) Expired() bool {
    return i.Expires != nil && i.Expires.Before(time.Now())
}

func (i Invite) Active(db *datastore) bool {
    if i.Expired() {          // 1. 检查是否过期
        return false
    }
    if i.MaxUses.Valid && i.MaxUses.Int64 > 0 {
        // 2. 检查是否超过最大使用次数
        if c := db.GetUsersInvitedCount(i.ID); c >= i.MaxUses.Int64 {
            return false
        }
    }
    return true
}
```

有效性判定的双重条件：
1. **时间维度**：`Expires` 字段非 nil 且早于当前时间 → 已过期
2. **数量维度**：`MaxUses` 有值且 `已使用次数 >= 最大次数` → 已用完

### 2.3 创建邀请

定义在 [invites.go#L99-L134](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/invites.go#L99-L134)：

```go
func handleCreateUserInvite(app *App, u *User, w http.ResponseWriter, r *http.Request) error {
    muVal := r.FormValue("uses")      // 使用次数参数
    expVal := r.FormValue("expires")  // 过期时间参数（分钟）

    if u.IsSilenced() {               // 风控检查1：静默用户禁止创建邀请
        return ErrUserSilenced
    }
    // ... 解析 maxUses 和 expDate ...
    inviteID := id.GenerateRandomString("0123456789BCDFGHJKLMNPQRSTVWXYZbcdfghjklmnpqrstvwxyz", 6)
    err = app.db.CreateUserInvite(inviteID, u.ID, maxUses, expDate)
    // ...
}
```

创建邀请前的风控检查：
- **静默状态检查**：调用 `u.IsSilenced()`，被静默用户无法创建邀请

### 2.4 邀请码使用（注册流程关联）

在 [account.go#L178-L184](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/account.go#L178-L184) 中，用户注册成功后记录邀请关系：

```go
if signup.InviteCode != "" {
    err = app.db.CreateInvitedUser(signup.InviteCode, u.ID)
    if err != nil {
        return nil, err
    }
}
```

OAuth 注册同样处理，见 [oauth_signup.go#L136-L142](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/oauth_signup.go#L136-L142)。

---

## 三、实例配置（全局邀请权限控制）

### 3.1 配置项定义

定义在 [config/config.go#L161](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/config/config.go#L161)：

```go
type AppCfg struct {
    // ...
    UserInvites string `ini:"user_invites"`  // 邀请权限配置
}
```

### 3.2 权限判断逻辑

定义在 [account.go#L74-L77](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/account.go#L74-L77)：

```go
func canUserInvite(cfg *config.Config, isAdmin bool) bool {
    return cfg.App.UserInvites != "" &&
        (isAdmin || cfg.App.UserInvites != "admin")
}
```

三档配置说明：

| 配置值 | 含义 | 判定结果 |
|--------|------|----------|
| `""`（或 "none"） | 无人可创建邀请 | `canUserInvite` 返回 `false` |
| `"admin"` | 仅管理员可创建 | 仅 `isAdmin=true` 时返回 `true` |
| `"user"` | 所有用户可创建 | 所有登录用户返回 `true` |

### 3.3 配置持久化

管理员在后台修改配置时，处理逻辑在 [admin.go#L571-L606](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/admin.go#L571-L606)：

```go
apper.App().cfg.App.UserInvites = r.FormValue("user_invites")
if apper.App().cfg.App.UserInvites == "none" {
    apper.App().cfg.App.UserInvites = ""  // "none" 归一化为空字符串
}
```

### 3.4 页面访问控制

在 [invites.go#L61-L65](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/invites.go#L61-L65) 中，邀请管理页面前置检查：

```go
func handleViewUserInvites(app *App, u *User, w http.ResponseWriter, r *http.Request) error {
    if !(app.cfg.App.UserInvites != "" && (u.IsAdmin() || app.cfg.App.UserInvites != "admin")) {
        return impart.HTTPError{http.StatusNotFound, ""}
    }
    // ...
}
```

---

## 四、邮箱风控

### 4.1 邮箱标准化（CleanEmail）

定义在 [spam/email.go#L28-L44](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/spam/email.go#L28-L44)：

```go
func CleanEmail(email string) string {
    emailParts := strings.Split(strings.ToLower(email), "@")
    if len(emailParts) < 2 {
        return ""
    }
    u := emailParts[0]
    d := emailParts[1]
    // 1. 去除 + 号及后面内容（如 user+spam@gmail.com → user@gmail.com）
    plusIdx := strings.IndexRune(u, '+')
    if plusIdx > -1 {
        u = u[:plusIdx]
    }
    // 2. 去除用户名中的所有点号（如 u.s.e.r@gmail.com → user@gmail.com）
    u = strings.ReplaceAll(u, ".", "")
    return u + "@" + d
}
```

该函数用于将各种变体邮箱归一化为唯一标识，防止通过邮箱变体绕过黑名单或批量注册。

### 4.2 蜜罐反垃圾（Honeypot）

定义在 [spam/email.go#L19-L26](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/spam/email.go#L19-L26)：

```go
var honeypotField string

func HoneypotFieldName() string {
    if honeypotField == "" {
        honeypotField = id.Generate62RandomString(39)  // 生成39位随机字段名
    }
    return honeypotField
}
```

**工作原理**：
1. 服务启动后生成一个随机、不可预测的字段名
2. 在表单中插入一个隐藏（`position: absolute; left: -5000px`）的 input 元素
3. 普通用户看不到此字段不会填写，而自动化机器人通常会填充所有字段
4. 后端检查：如果该字段有值，则判定为机器人请求，直接拒绝

实际使用见 [email.go#L134-L136](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/email.go#L134-L136)：

```go
if r.FormValue(spam.HoneypotFieldName()) != "" || r.FormValue("fake_password") != "" {
    log.Info("Honeypot field was filled out! Not subscribing.")
    return impart.HTTPError{http.StatusFound, from}
}
```

### 4.3 用户注册结构体中的 Honeypot 字段

定义在 [users.go#L40-L50](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/users.go#L40-L50)：

```go
type userRegistration struct {
    userCredentials
    InviteCode string `json:"invite_code" schema:"invite_code"`
    Honeypot   string `json:"fullname" schema:"fullname"`  // 蜜罐字段，伪装成 "fullname"
    // ...
}
```

注册表单预留了 `Honeypot` 字段（映射为表单字段 `fullname`），为后续注册反垃圾预留扩展点。

### 4.4 邮箱加密存储

定义在 [account.go#L1538-L1550](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/account.go#L1538-L1550)：

```go
func prepareUserEmail(input string, emailKey []byte) zero.String {
    email := zero.NewString("", input != "")
    if len(input) > 0 {
        encEmail, err := data.Encrypt(emailKey, input)
        if err != nil {
            log.Error("Unable to encrypt email: %s\n", err)
        } else {
            email.String = string(encEmail)
        }
    }
    return email
}
```

用户邮箱使用 AES 加密后存储在数据库（`varbinary(255)`），即使数据库泄露也无法直接获取用户邮箱。

---

## 五、IP 风控

### 5.1 真实 IP 获取

定义在 [spam/ip.go#L18-L25](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/spam/ip.go#L18-L25)：

```go
func GetIP(r *http.Request) string {
    h := r.Header.Get("X-Forwarded-For")
    if h == "" {
        return ""
    }
    ips := strings.Split(h, ",")
    return strings.TrimSpace(ips[0])  // 取第一个 IP（最原始的客户端 IP）
}
```

应用场景：反向代理 / CDN 环境下，从 `X-Forwarded-For` 头部提取真实客户端 IP，用于日志记录和风控判断。

### 5.2 登录频率限制

定义在 [account.go#L394-L496](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/account.go#L394-L496)：

```go
var loginAttemptUsers = sync.Map{}  // 内存中记录各用户的登录尝试过期时间
const loginAttemptExpiration = 3 * time.Second

func login(app *App, w http.ResponseWriter, r *http.Request) error {
    // ...
    if !app.cfg.Server.Dev {  // 开发环境跳过频率限制
        now := time.Now()
        attemptExp, att := loginAttemptUsers.LoadOrStore(signin.Alias, now.Add(loginAttemptExpiration))
        if att {
            if attemptExpTime, ok := attemptExp.(time.Time); ok {
                if attemptExpTime.After(now) {
                    // 冷却期内，返回 429 Too Many Requests
                    return impart.HTTPError{http.StatusTooManyRequests, "You're doing that too much."}
                } else {
                    // 冷却期已过，清除记录
                    loginAttemptUsers.Delete(signin.Alias)
                }
            }
        }
    }
    // ... 继续密码校验
}
```

限制策略：
- **粒度**：按用户名（`signin.Alias`）
- **窗口**：3 秒
- **效果**：同一用户名 3 秒内只能发起一次登录请求，防止暴力破解密码

### 5.3 管理员密码重置 IP 告警

在 [account.go#L1332-L1348](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/account.go#L1332-L1348) 中：

```go
ip := spam.GetIP(r)
// ...
if u.IsAdmin() {
    // 管理员账号禁止通过邮件重置密码，并记录告警日志
    log.Error("Admin reset attempt", `Someone just tried to reset the password for an admin (ID %d - %s). IP address: %s`, u.ID, u.Username, ip)
    return returnLoc
}
```

管理员账户的密码重置请求会被直接拦截并记录 IP，防止针对管理员的社会工程学攻击。

---

## 六、用户静默状态（跨模块风控）

### 6.1 状态定义

定义在 [users.go#L22-L27](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/users.go#L22-L27)：

```go
type UserStatus int

const (
    UserActive   = iota  // 0: 正常
    UserSilenced         // 1: 已静默
)
```

判断逻辑在 [users.go#L134-L136](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/users.go#L134-L136)：

```go
func (u *User) IsSilenced() bool {
    return u.Status&UserSilenced != 0  // 位运算判断，支持多状态位扩展
}
```

### 6.2 静默状态影响的功能

| 功能 | 检查位置 | 行为 |
|------|----------|------|
| 创建邀请 | [invites.go#L103-L105](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/invites.go#L103-L105) | 返回 `ErrUserSilenced`，禁止创建邀请 |
| 浏览邀请页 | [invites.go#L79-L85](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/invites.go#L79-L85) | 标记 `Silenced`，前端可能限制显示 |
| 发布文章/操作 | 多个页面（articles, collections, stats 等） | 标记 `Silenced` 状态，前端展示提示 |

### 6.3 管理员切换用户状态

定义在 [admin.go#L352-L377](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/admin.go#L352-L377)：

```go
func handleAdminToggleUserStatus(app *App, u *User, w http.ResponseWriter, r *http.Request) error {
    // ...
    if user.IsSilenced() {
        err = app.db.SetUserStatus(user.ID, UserActive)     // 解除静默
    } else {
        err = app.db.SetUserStatus(user.ID, UserSilenced)   // 设为静默
        updateTimelineCache(app.timeline, true)             // 同时清空时间线缓存
    }
    // ...
}
```

---

## 七、全流程协作关系

### 7.1 邀请创建流程

```
用户点击创建邀请
    │
    ▼
路由权限检查 handler.User(handleCreateUserInvite)
    │  └─ 需已登录
    ▼
实例配置检查 canUserInvite()
    │  ├─ UserInvites == "" → 404 拒绝
    │  ├─ UserInvites == "admin" 且非管理员 → 404 拒绝
    │  └─ 其他 → 继续
    ▼
用户静默检查 u.IsSilenced()
    │  └─ 已静默 → 返回 ErrUserSilenced
    ▼
解析参数 uses / expires
    │
    ▼
生成6位随机邀请码
    │
    ▼
写入 userinvites 表
    │  └─ id, owner_id, max_uses, created, expires
    ▼
重定向至 /me/invites
```

### 7.2 邀请注册流程

```
访客访问 /invite/{code}
    │
    ▼
查询 userinvites 表获取邀请记录
    │  └─ 不存在 → 404
    ▼
有效性检查 i.Active()
    ├─ Expired() → 邀请过期
    └─ uses >= MaxUses → 邀请已用完
    │
    ▼
渲染 signup.tmpl（携带 invite_code）
    │
    ▼
用户提交注册表单 /auth/signup
    │
    ├─► 用户名合法性检查 author.IsValidUsername()
    │
    ├─► 用户名重复性检查 db.PostIDExists() / users 表
    │
    ├─► 密码哈希 auth.HashPass()
    │
    ├─► 邮箱加密 prepareUserEmail()
    │
    ▼
事务写入 users + collections 表
    │
    ▼
若有 InviteCode → 写入 usersinvited 表建立关联
    │  └─ invite_id, user_id
    │
    ▼
登录态建立（Session / AccessToken）
    │
    ▼
重定向（有邀请码则回到 /invite/{code}）
```

### 7.3 登录风控流程

```
用户提交登录 /auth/login
    │
    ▼
非 Dev 环境 → 登录频率限制
    │  └─ 同用户名 3 秒内重复请求 → 429 Too Many Requests
    │
    ▼
获取用户信息 db.GetUserForAuth()
    │
    ▼
密码校验 auth.Authenticated()
    │  └─ 失败 → 401 Unauthorized
    │
    ▼
管理员密码重置 → IP 告警 + 直接拒绝
    │
    ▼
登录成功 → 发放 Session / Token
```

### 7.4 模块协作总览

```
                    ┌────────────────────────┐
                    │   config.AppCfg        │
                    │   - UserInvites        │
                    │  (""/"admin"/"user")   │
                    └──────────┬─────────────┘
                               │ 全局开关
                               ▼
        ┌───────────────────────────────────────────────┐
        │              邀请模块 invites.go              │
        │  - Invite{MaxUses, Expires, Inactive}         │
        │  - Active() 双条件判定(时间+数量)              │
        │  - 创建前检查 u.IsSilenced()                   │
        └───────────────────┬───────────────────────────┘
                            │ 邀请码有效性
                            ▼
        ┌───────────────────────────────────────────────┐
        │           注册模块 account.go                  │
        │  - signupWithRegistration()                   │
        │  - 用户名/密码/邮箱校验                         │
        │  - 成功后写入 usersinvited 关联                 │
        └────┬─────────────────────┬────────────────────┘
             │                     │
             ▼                     ▼
   ┌─────────────────┐   ┌─────────────────────┐
   │ spam/email.go   │   │    spam/ip.go        │
   │ - CleanEmail()  │   │  - GetIP()           │
   │ - HoneypotField │   │  登录频率限制 3s      │
   │  (邮箱标准化)    │   │  (防暴力破解)        │
   └─────────────────┘   └─────────────────────┘
                            │
                            ▼
                   ┌─────────────────────┐
                   │   users.UserStatus  │
                   │  - UserActive(0)    │
                   │  - UserSilenced(1)  │
                   │  (跨模块风控标记)    │
                   └─────────────────────┘
```

---

## 八、数据库表结构

### 8.1 userinvites（邀请码表）

```sql
CREATE TABLE `userinvites` (
  `id` char(6) NOT NULL,              -- 邀请码（6位随机字符串）
  `owner_id` int(11) NOT NULL,        -- 创建者用户 ID
  `max_uses` smallint(6) DEFAULT NULL,-- 最大使用次数（NULL=无限）
  `created` datetime NOT NULL,        -- 创建时间
  `expires` datetime DEFAULT NULL,    -- 过期时间（NULL=永不过期）
  `inactive` tinyint(1) DEFAULT NULL, -- 是否停用
  PRIMARY KEY (`id`)
);
```

### 8.2 usersinvited（邀请关联表）

```sql
CREATE TABLE `usersinvited` (
  `invite_id` char(6) NOT NULL,  -- 邀请码
  `user_id` int(11) NOT NULL     -- 被邀请用户 ID
);
```

### 8.3 users（用户表）

```sql
CREATE TABLE IF NOT EXISTS `users` (
  `id` int(6) NOT NULL AUTO_INCREMENT,
  `username` varchar(100) NOT NULL,
  `password` char(60) NOT NULL,      -- bcrypt 哈希
  `email` varbinary(255) DEFAULT NULL,-- AES 加密邮箱
  `created` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `status` int(11) NOT NULL DEFAULT 0,-- 0=正常, 1=静默
  PRIMARY KEY (`id`)
);
```

---

## 九、关键代码索引

| 功能模块 | 文件 | 关键行 |
|----------|------|--------|
| Invite 结构体 | [invites.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/invites.go) | L27-L55 |
| 创建邀请 | [invites.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/invites.go) | L99-L134 |
| 邀请有效性检查 | [invites.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/invites.go) | L41-L55, L136-L150 |
| UserInvites 配置 | [config/config.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/config/config.go) | L161 |
| 邀请权限判断 | [account.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/account.go) | L74-L77 |
| 管理员配置更新 | [admin.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/admin.go) | L571-L606 |
| CleanEmail 标准化 | [spam/email.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/spam/email.go) | L28-L44 |
| Honeypot 蜜罐 | [spam/email.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/spam/email.go) | L19-L26 |
| GetIP 获取真实 IP | [spam/ip.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/spam/ip.go) | L18-L25 |
| 登录频率限制 | [account.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/account.go) | L394-L496 |
| 用户静默状态 | [users.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/users.go) | L22-L27, L134-L136 |
| 静默用户禁止发邀请 | [invites.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/invites.go) | L103-L105 |
| 注册核心流程 | [account.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/account.go) | L134-L249 |
| 邀请关联记录 | [account.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/account.go) | L178-L184 |
| OAuth 邀请注册 | [oauth_signup.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/oauth_signup.go) | L136-L142 |
| DB: CreateUser | [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/database.go) | L213-L267 |
| DB: CreateUserInvite | [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/database.go) | L2737-L2740 |
| DB: CreateInvitedUser | [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/database.go) | L2800-L2803 |
| DB: SetUserStatus | [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/38-writefreely/database.go) | L2922-L2928 |
