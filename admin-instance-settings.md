# WriteFreely 管理后台与实例设置分析

## 一、管理后台权限边界

### 1.1 管理员身份判定

管理员判定逻辑位于 [users.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/users.go#L129-L132)：

```go
func (u *User) IsAdmin() bool {
    return u.ID == 1
}
```

**关键要点**：管理员身份硬编码为 `ID == 1` 的用户，即数据库中第一个被创建的用户。这不是一个可配置的角色或权限字段，而是与用户 ID 绑定。这意味着：

- 只有 ID 为 1 的用户才是管理员，没有多管理员机制
- 注释 `// TODO: get this from database` 表明作者也意识到这一设计的局限性
- 在 CLI 创建用户时（[app.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/app.go#L898-L959)），`--create-admin` 仅允许创建第一个用户为管理员，后续不能再创建管理员

### 1.2 权限中间件层

管理后台通过 Handler 封装实现权限控制，定义在 [handle.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/handle.go#L176-L249)：

| Handler 方法 | 权限要求 | 认证方式 | 失败响应 |
|---|---|---|---|
| `Handler.Admin()` | 登录 + `IsAdmin()` | Cookie Session | 404 Not Found |
| `Handler.AdminApper()` | 登录 + `IsAdmin()` | Cookie Session | 404 Not Found |
| `Handler.User()` | 仅登录 | Cookie Session | 401 → 重定向 /login |
| `Handler.AllReader()` | 私有实例需登录 | API Token / Cookie | 对应错误码 |

**安全设计**：Admin 路由未授权时返回 404 而非 403，这是一种信息隐藏策略——不让攻击者确认 /admin 路由的存在。

### 1.3 Admin 路由一览

所有管理路由定义在 [routes.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/routes.go#L173-L185)：

| 路由 | Handler | 功能 | 方法 |
|---|---|---|---|
| `/admin` | `Admin()` | 仪表盘概览 | GET |
| `/admin/monitor` | `Admin()` | 系统监控面板 | GET |
| `/admin/settings` | `Admin()` | 实例配置页面 | GET |
| `/admin/users` | `Admin()` | 用户列表 | GET |
| `/admin/user/{username}` | `Admin()` | 查看单个用户 | GET |
| `/admin/user/{username}/delete` | `Admin()` | 删除用户 | POST |
| `/admin/user/{username}/status` | `Admin()` | 切换用户状态（静音/激活） | POST |
| `/admin/user/{username}/passphrase` | `Admin()` | 重置用户密码 | POST |
| `/admin/pages` | `Admin()` | 实例页面管理 | GET |
| `/admin/page/{slug}` | `Admin()` | 编辑单个页面 | GET |
| `/admin/update/config` | `AdminApper()` | 保存实例配置 | POST |
| `/admin/update/{page}` | `Admin()` | 保存页面内容 | POST |
| `/admin/updates` | `Admin()` | 软件更新检查 | GET |

注意 `/admin/update/config` 使用 `AdminApper()` 而非 `Admin()`，因为它需要访问 `Apper` 接口来调用 `SaveConfig()` 方法持久化配置到磁盘。

### 1.4 额外的权限校验

删除用户操作在 [admin.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/admin.go#L321-L350) 中有双重校验：

```go
func handleAdminDeleteUser(app *App, u *User, w http.ResponseWriter, r *http.Request) error {
    if !u.IsAdmin() {
        return impart.HTTPError{http.StatusForbidden, "Administrator privileges required for this action"}
    }
    // ...
}
```

这是在 `Admin()` Handler 已经验证过 `IsAdmin()` 之后的二次校验，形成了防御纵深。

---

## 二、配置保存机制

### 2.1 配置文件结构

配置使用 INI 格式存储在 `config.ini` 文件中，结构定义在 [config/config.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/config/config.go#L188-L198)：

```
Config
├── Server       ServerCfg       [server]
├── Database     DatabaseCfg     [database]
├── App          AppCfg          [app]
├── Email        EmailCfg        [email]
├── SlackOauth   SlackOauthCfg   [oauth.slack]
├── WriteAsOauth WriteAsOauthCfg [oauth.writeas]
├── GitlabOauth  GitlabOauthCfg  [oauth.gitlab]
├── GiteaOauth   GiteaOauthCfg   [oauth.gitea]
└── GenericOauth GenericOauthCfg [oauth.generic]
```

### 2.2 AppCfg — 管理后台可修改的配置

`AppCfg` 是管理后台 `/admin/settings` 页面可以修改的配置子集，定义在 [config/config.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/config/config.go#L123-L171)：

| 字段 | INI 键 | 说明 |
|---|---|---|
| `SiteName` | `site_name` | 站点名称 |
| `SiteDesc` | `site_description` | 站点描述 |
| `Landing` | `landing` | 着陆页路径 |
| `OpenRegistration` | `open_registration` | 开放注册 |
| `OpenDeletion` | `open_deletion` | 开放账号删除 |
| `MinUsernameLen` | `min_username_len` | 最短用户名长度 |
| `MaxBlogs` | `max_blogs` | 每用户最大博客数 |
| `Federation` | `federation` | 联邦（ActivityPub） |
| `PublicStats` | `public_stats` | 公开统计 |
| `Monetization` | `monetization` | Web Monetization |
| `Private` | `private` | 私有实例 |
| `LocalTimeline` | `local_timeline` | 本地时间线/Reader |
| `UserInvites` | `user_invites` | 用户邀请码机制 |
| `DefaultVisibility` | `default_visibility` | 默认文章可见性 |

### 2.3 配置保存流程

1. **Web 界面保存**：`handleAdminUpdateConfig()` 在 [admin.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/admin.go#L571-L606) 中处理 POST 请求：
   - 从表单读取所有字段值，写入内存中的 `apper.App().cfg.App`
   - 对于 checkbox 类字段（如 `OpenRegistration`），通过 `r.FormValue("xxx") == "on"` 判断
   - 调用 `apper.SaveConfig(apper.App().cfg)` 将完整 Config 写入 INI 文件
   - 成功后重定向到 `/admin/settings?cm=Configuration+saved.#config`

2. **INI 文件持久化**：`config.Save()` 在 [config/config.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/config/config.go#L304-L315) 中使用 `go-ini` 库：
   ```go
   func Save(uc *Config, fname string) error {
       cfg := ini.Empty()
       err := ini.ReflectFrom(cfg, uc)  // 将结构体反射为 INI
       return cfg.SaveTo(fname)          // 写入文件
   }
   ```
   **注意**：`ini.Empty()` 创建空 INI 对象后再从结构体反射，这意味着手动添加到 config.ini 中的注释和额外字段会在保存时丢失。

3. **配置加载**：`config.Load()` 在 [config/config.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/config/config.go#L269-L301) 中从 INI 文件读取：
   - 使用 `ini.Load()` 解析文件
   - 使用 `cfg.MapTo(uc)` 映射到结构体
   - 对 Host 字段做 IDNA 规范化处理

4. **CLI 交互式配置**：`config.Configure()` 在 [config/setup.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/config/setup.go#L30-L396) 中提供分步向导，支持 server / db / app 三个配置段。

### 2.4 运行时配置变更的即时生效

`handleAdminUpdateConfig()` 修改配置后直接写入内存中的 `cfg.App` 对象，部分变更即时生效：

| 配置项 | 即时生效行为 | 代码位置 |
|---|---|---|
| `LocalTimeline` | 如果从关闭切换到开启，立即调用 `initLocalTimeline()` 初始化时间线 | [admin.go#L590-L593](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/admin.go#L590-L593) |
| `UserInvites` | 值为 `"none"` 时清空为 `""` | [admin.go#L595-L597](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/admin.go#L595-L597) |
| 其他配置 | 写入内存 + 文件，后续请求自然读取新值 | 全局 `app.cfg` 引用 |

**重要**：`ServerCfg`（端口、绑定地址、TLS 等）和 `DatabaseCfg` 不在 Web 管理界面中暴露，修改这些需要手动编辑 config.ini 后重启服务。

---

## 三、缓存影响

### 3.1 缓存体系总览

WriteFreely 有三层独立的缓存机制：

| 缓存类型 | 实现位置 | 用途 | 过期时间 |
|---|---|---|---|
| 用户文章缓存 `userPostsCache` | [cache.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/cache.go) | 缓存用户文章列表 | 4 秒 |
| 本地时间线缓存 `localTimeline` | [read.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/read.go#L41-L47) | 缓存 Reader 页面的公开文章 | 由 `memo` 库控制 |
| 更新检查缓存 `updatesCache` | [updates.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/updates.go#L29-L36) | 缓存远程版本信息 | 12 小时 |

### 3.2 用户文章缓存

[cache.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/cache.go#L22-L69)：

```go
postsCacheTime = 4 * time.Second

type postsCacheItem struct {
    Expire time.Time
    Posts  *[]PublicPost
    ready  chan struct{}
}

var userPostsCache = struct {
    sync.RWMutex
    users map[int64]postsCacheItem
}{}
```

- 以 `userID` 为键，`RWMutex` 保护并发安全
- 缓存命中时直接返回；过期返回 `nil`，触发重新查询数据库
- `ready` channel 用于防缓存击穿（阻止并发请求同时回源）

### 3.3 本地时间线缓存（localTimeline）

[read.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/read.go#L41-L47)：

```go
type localTimeline struct {
    m     *memo.Memo
    posts *[]PublicPost
    postsPerPage int
}
```

- 使用 `memo.Memo` 库实现自动过期刷新
- `initLocalTimeline()` 在 [read.go#L63-L68](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/read.go#L63-L68) 中初始化
- `updateTimelineCache()` 在 [read.go#L162-L178](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/read.go#L162-L178) 中支持强制刷新

**与管理操作的联动**：

当管理员静音（silence）用户时，时间线缓存会被强制刷新以移除该用户的文章：

```go
// admin.go#L364-L371
if user.IsSilenced() {
    err = app.db.SetUserStatus(user.ID, UserActive)
} else {
    err = app.db.SetUserStatus(user.ID, UserSilenced)
    updateTimelineCache(app.timeline, true)  // 强制刷新缓存
}
```

反之，取消静音时**不会**主动刷新缓存，需要等待缓存自然过期。

### 3.4 更新检查缓存

[updates.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/updates.go#L29-L36)：

```go
type updatesCache struct {
    mu             sync.Mutex
    frequency      time.Duration
    lastCheck      time.Time
    latestVersion  string
    currentVersion string
    checkError     error
}
```

- 默认 12 小时检查一次（`defaultUpdatesCacheTime`）
- 管理员可在 `/admin/updates` 页面通过 `?check=now` 参数强制即时检查
- 仅在 `App.UpdateChecks = true` 时启用
- `AdminPage` 在每个管理页面都会显示更新可用状态（`UpdateAvailable` 标记）

### 3.5 缓存与管理操作的交互关系

| 管理操作 | 缓存影响 |
|---|---|
| 修改实例配置 | 配置写入内存 + 文件，`LocalTimeline` 开启时初始化新缓存 |
| 静音用户 | 强制刷新 `localTimeline` 缓存（`reset=true`） |
| 取消静音用户 | 不主动刷新缓存，等待自然过期 |
| 删除用户 | 无主动缓存操作 |
| 修改实例页面内容 | 写入数据库 `appcontent` 表，无缓存层 |
| 检查更新 | 刷新 `updatesCache` |

---

## 四、运行状态展示

### 4.1 管理仪表盘（/admin）

[admin.go#L118-L145](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/admin.go#L118-L145) `handleViewAdminDash()`：

展示内容：
- 用户总数 `UsersCount`
- 博客（Collection）总数 `CollectionsCount`
- 文章总数 `PostsCount`
- 是否有可用更新（`AdminPage.UpdateAvailable`）

### 4.2 系统监控面板（/admin/monitor）

[admin.go#L147-L168](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/admin.go#L147-L168) `handleViewAdminMonitor()`：

展示内容：
- 系统运行状态 `systemStatus`
- 当前 `AppCfg` 配置（只读展示）

### 4.3 systemStatus 详细字段

[admin.go#L39-L78](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/admin.go#L39-L78) 定义了完整的系统状态结构：

| 分类 | 字段 | 说明 |
|---|---|---|
| 运行时间 | `Uptime` | 自 `appStartTime` 以来的友好格式时间 |
| 协程 | `NumGoroutine` | 当前 Goroutine 数量 |
| 内存总览 | `MemAllocated` | 已分配且仍在使用的字节 |
| | `MemTotal` | 累计分配字节（含已释放） |
| | `MemSys` | 从系统获取的总字节 |
| 堆内存 | `HeapAlloc` / `HeapSys` / `HeapIdle` / `HeapInuse` / `HeapReleased` / `HeapObjects` | 堆分配详细指标 |
| 栈/低级分配 | `StackInuse` / `StackSys` / `MSpanInuse` / `MSpanSys` / `MCacheInuse` / `MCacheSys` / `BuckHashSys` / `GCSys` / `OtherSys` | 栈和底层分配器指标 |
| GC | `NextGC` / `LastGC` / `PauseTotalNs` / `PauseNs` / `NumGC` | 垃圾回收统计 |

`updateAppStats()` 在 [admin.go#L608-L644](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/admin.go#L608-L644) 中通过 `runtime.ReadMemStats()` 采集 Go 运行时内存数据，通过 `appstats.TimeSincePro()` 格式化运行时间，通过 `appstats.FileSize()` 将字节数转为人类可读格式（KB/MB/GB）。

**注意**：`updateAppStats()` 只在访问 `/admin/monitor` 页面时调用，是按需采集而非持续监控。

### 4.4 版本更新页面（/admin/updates）

[admin.go#L659-L693](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/admin.go#L659-L693)：

展示内容：
- 当前版本号（`softwareVer`，硬编码为 `"0.16.0"`）
- 最近检查时间
- 最新可用版本
- 最新版本发布 URL 和 Release Notes URL
- 更新是否可用
- 检查是否失败

### 4.5 NodeInfo 公开统计

[nodeinfo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/nodeinfo.go) 通过 NodeInfo 协议对外暴露实例统计：

- 当 `PublicStats = true` 时，显示半年度/月度活跃用户数和文章总数
- 当 `PublicStats = false` 时，仅返回博客总数和文章总数，不计算活跃用户指标
- 元数据中包含 `Private`、`MaxBlogs`、`PublicReader`、`Invites` 等信息

### 4.6 用户管理页面状态

管理员查看单个用户时（`/admin/user/{username}`），展示：

- 用户基本信息（用户名、创建时间、状态）
- 总文章数 `TotalPosts`
- 最近发文时间 `LastPost`
- 所属博客列表（含粉丝数和最近发文时间，联邦开启时）
- 密码重置（生成随机密码 `passgen.NewWordish()`，通过 session flash 传递）

---

---

## 六、Session 认证机制深度分析

### 6.1 Session 初始化与存储

Session 系统基于 `gorilla/sessions` 的 CookieStore 实现，定义在 [session.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/session.go#L37-L53)：

```go
func (app *App) InitSession() {
    gob.Register(&User{})
    store := sessions.NewCookieStore(app.keys.CookieAuthKey, app.keys.CookieKey)
    store.Options = &sessions.Options{
        Path:     "/",
        MaxAge:   sessionLength,  // 180 天
        HttpOnly: true,
        Secure:   strings.HasPrefix(app.cfg.App.Host, "https://"),
    }
    if store.Options.Secure {
        store.Options.SameSite = http.SameSiteNoneMode
    }
    app.sessionStore = store
}
```

**关键安全属性**：

| 属性 | 值 | 说明 |
|---|---|---|
| `HttpOnly` | `true` | 防止 JavaScript 读取 Cookie，缓解 XSS 攻击 |
| `Secure` | 视 `Host` 协议而定 | HTTPS 站点自动启用，防止 HTTP 明文传输 |
| `SameSite` | HTTPS 时为 `None` | 允许跨站请求携带 Cookie（联邦场景） |
| `MaxAge` | 180 天（`sessionLength = 180 * day`） | 长效会话 |
| 签名密钥 | `CookieAuthKey` | 验证 Cookie 完整性，防篡改 |
| 加密密钥 | `CookieKey` | 加密 Cookie 内容 |

**注意**：当站点为 HTTPS 时 `SameSite=None`，这意味着 Cookie 会被第三方网站的请求携带，增加了 CSRF 攻击面。

### 6.2 Cookie 中存储的用户信息

`User.Cookie()` 方法在 [users.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/users.go#L123-L127) 中定义：

```go
func (u User) Cookie() *User {
    u.HashedPass = []byte{}  // 清除密码哈希
    return &u
}
```

Cookie 中存储的 `User` 对象包含：`ID`、`Username`、`HasPass`、`Email`（加密形式）、`Created`、`Status`。

**密码哈希已被清除**，但用户 ID 和用户名以明文形式存储在加密 Cookie 中。每次保存 Session 时都会调用 `Cookie()` 方法精简数据（[session.go#L128](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/session.go#L128)）。

### 6.3 登录流程与认证机制

登录流程在 `login()` 函数中实现（[account.go#L396-L566](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/account.go#L396-L566)）：

**Web 登录流程**：
1. 解析 POST 表单（用户名 + 密码）
2. **登录速率限制**：非开发环境下，同一用户名 3 秒内只能尝试一次
3. 从数据库获取用户（`GetUserForAuth`）
4. 使用 `auth.Authenticated()` 验证密码（bcrypt）
5. 成功后将用户对象存入 Session Cookie
6. 重定向到目标 URL

**登录速率限制实现**（[account.go#L478-L496](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/account.go#L478-L496)）：

```go
var loginAttemptUsers = sync.Map{}

if !app.cfg.Server.Dev {
    now := time.Now()
    attemptExp, att := loginAttemptUsers.LoadOrStore(signin.Alias, now.Add(loginAttemptExpiration))
    if att && attemptExpTime.After(now) {
        return ErrTooManyRequests
    }
}
```

**重要限制**：
- 仅按**用户名**限流，不按 IP 限流——攻击者可以针对不同用户名暴力破解
- 限流窗口只有 3 秒，防护作用有限
- 使用 `sync.Map` 存储在内存中，多实例部署时无效
- 开发环境（`Server.Dev = true`）完全跳过敏率限制

### 6.4 Admin 认证流程

Admin 路由的认证在 `Handler.Admin()` 和 `Handler.AdminApper()` 中（[handle.go#L176-L249](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/handle.go#L176-L249)）：

```go
u := getUserSession(h.app.App(), r)
if u == nil || !u.IsAdmin() {
    err := impart.HTTPError{http.StatusNotFound, ""}
    status = err.Status
    return err
}
```

**两步验证**：
1. `getUserSession()` 从 Cookie 中解析用户（验证签名 + 解密）
2. `u.IsAdmin()` 检查用户 ID 是否为 1

未通过时返回 404 而非 401/403，隐藏管理入口存在性。

### 6.5 会话管理漏洞点

1. **无会话失效机制**：用户修改密码后，现有 Session Cookie 仍然有效（因为 Cookie 中存的是用户对象，不是 token）
2. **多设备登录无记录**：无法查看或撤销其他设备的登录会话
3. **180 天超长有效期**：Cookie 被盗用后可长期使用
4. **无登录日志**：无法审计登录行为
5. **Admin 无特殊会话策略**：管理员会话与普通用户会话安全级别相同

---

## 七、CSRF 防护差异深度分析

### 7.1 CSRF 保护的路由分布

通过 `csrf.Protect()` 中间件保护的路由（[routes.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/routes.go)）：

| 路由 | 方法 | Handler | CSRF 保护 |
|---|---|---|---|
| `/me/delete` | POST | `handler.User(handleUserDelete)` | **有** |
| `/me/settings` | GET | `handler.User(viewSettings)` | **有** |
| `/reset` | GET + POST | `handler.Web(viewResetPassword, UserLevelNoneRequired)` | **有** |

**所有 Admin POST 路由均无 CSRF 保护**：
- `/admin/user/{username}/delete` — 删除用户
- `/admin/user/{username}/status` — 切换用户状态
- `/admin/user/{username}/passphrase` — 重置用户密码
- `/admin/update/config` — 修改实例配置
- `/admin/update/{page}` — 修改页面内容

### 7.2 CSRF Token 的生成与校验

CSRF 保护使用 `gorilla/csrf` 库，密钥为 `app.keys.CSRFKey`（32 字节）。

**Token 生成与模板渲染**：在 `viewSettings()` 中（[account.go#L1230](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/account.go#L1230)）：

```go
CSRFField: csrf.TemplateField(r),
```

这会在模板中渲染一个隐藏的表单字段，例如：
```html
<input type="hidden" name="gorilla.csrf.Token" value="...token..." />
```

### 7.3 Admin 路由缺少 CSRF 保护的风险

**攻击场景**：

1. **诱导管理员点击恶意链接**：攻击者构造一个自动提交的表单页面，诱导已登录的管理员访问
2. **自动执行敏感操作**：如删除用户、修改实例配置、禁用注册、开启私有模式等
3. **隐蔽性强**：由于 Admin 路由返回 404 隐藏入口，攻击成功后管理员可能不易察觉

**为什么风险较高**：
- `SameSite=None`（HTTPS 站点）意味着 Cookie 会被跨站请求携带
- 管理员功能具有最高权限，可以删除用户、修改实例配置
- 无 Referer 校验等替代防护
- Session 有效期长达 180 天

### 7.4 普通用户路由的 CSRF 覆盖

普通用户设置页面有 CSRF 保护，但更新设置的 API 端点 `/api/me/self`（POST）使用 Token 认证而非 Session Cookie，天然免疫 CSRF。

**用户删除操作**（`/me/delete`）同时使用了 `csrf.Protect()` 和 `handler.User()`，形成双重保护。

### 7.5 密码重置页面的 CSRF

`/reset` 页面使用了 `csrf.Protect()`，这是合理的：
- 防止 CSRF 触发密码重置邮件发送（邮件轰炸）
- 防止 CSRF 提交新密码

---

## 八、密码重置流程安全性分析

### 8.1 密码重置流程总览

完整的密码重置流程涉及三个函数（[account.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/account.go)）：

1. **发起重置**：`handleResetPasswordInit()` （第 1324-1379 行）
2. **邮箱验证与新密码设置页面**：`viewResetPassword()` （第 1247-1307 行）
3. **执行密码修改**：`doAutomatedPasswordChange()` （第 1309-1322 行）

### 8.2 发起重置的安全措施

`handleResetPasswordInit()` 中的安全设计：

| 安全措施 | 实现 | 说明 |
|---|---|---|
| **管理员账户禁止重置** | `if u.IsAdmin()` 直接返回 | 防止通过重置获取管理员权限 |
| **无邮箱用户拒绝重置** | `if u.Email.String == ""` | 无验证渠道则无法重置 |
| **用户枚举防护** | 无论用户是否存在，都返回相同提示 | 通过 `addSessionFlash` 统一提示，不暴露用户存在性 |
| **记录攻击日志** | 管理员重置尝试时记录 IP | `log.Error("Admin reset attempt...")` |

**管理员重置拦截**（[account.go#L1344-L1348](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/account.go#L1344-L1348)）：

```go
if u.IsAdmin() {
    log.Error("Admin reset attempt", `Someone just tried to reset the password for an admin (ID %d - %s). IP address: %s`, u.ID, u.Username, ip)
    return returnLoc
}
```

### 8.3 重置 Token 的生成与验证

数据库表 `password_resets` 结构相关操作（[database.go#L610-L639](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/database.go#L610-L639)）：

```go
func (db *datastore) CreatePasswordResetToken(userID int64) (string, error) {
    t := id.Generate62RandomString(32)  // 32 字符，62 进制
    _, err := db.Exec("INSERT INTO password_resets (user_id, token, used, created) VALUES (?, ?, 0, "+db.now()+")", userID, t)
    return t, nil
}
```

**Token 验证查询**：
```sql
SELECT user_id FROM password_resets 
WHERE token = ? AND used = 0 AND created > DATE_SUB(NOW(), INTERVAL 3 HOUR)
```

**Token 安全属性**：
- **长度**：32 个字符，62 进制 → 熵约 190 位，暴力破解不可行
- **有效期**：3 小时
- **一次性**：使用后 `used` 设为 1（`ConsumePasswordResetToken`）
- **存储方式**：明文存储在数据库中

**潜在问题**：
1. **Token 明文存储**：数据库泄露则所有重置 Token 可用
2. **可多次生成**：没有限制同一用户生成重置 Token 的频率
3. **旧 Token 不失效**：生成新 Token 不会使旧 Token 失效，3 小时内都有效

### 8.4 密码重置的执行

`viewResetPassword()` 中的重置执行逻辑（[account.go#L1260-L1278](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/account.go#L1260-L1278)）：

```go
if r.Method == http.MethodPost {
    newPass := r.FormValue("new-pass")
    if newPass == "" {
        return handleResetPasswordInit(app, w, r)
    }
    err := doAutomatedPasswordChange(app, userID, newPass)
    err = app.db.ConsumePasswordResetToken(token)
    addSessionFlash(app, w, r, "Your password was reset.", nil)
    return impart.HTTPError{http.StatusFound, "/login"}
}
```

**注意**：
- `doAutomatedPasswordChange` 中使用 `auth.HashPass()`（bcrypt）哈希新密码
- Token 消耗（`ConsumePasswordResetToken`）失败不影响密码修改成功——即使 Token 标记失败，密码已被修改
- 修改成功后不会使现有 Session 失效

### 8.5 管理员重置用户密码

管理员在 `/admin/user/{username}/passphrase` 重置密码（[admin.go#L379-L409](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/admin.go#L379-L409)）：

```go
func handleAdminResetUserPass(app *App, u *User, w http.ResponseWriter, r *http.Request) error {
    pass := passgen.NewWordish()  // 生成易读随机密码
    hashedPass, err := auth.HashPass([]byte(pass))
    err = app.db.ChangePassphrase(int64(id), true, "", hashedPass)
    addSessionFlash(app, w, r, fmt.Sprintf("SUCCESS: %s", pass), nil)
    return impart.HTTPError{http.StatusFound, fmt.Sprintf("/admin/user/%s", username)}
}
```

**特点**：
- 使用 `passgen.NewWordish()` 生成可读随机密码
- 通过 `ChangePassphrase(userID, sudo=true, "", hashedPass)` 绕过旧密码验证
- 新密码通过 session flash 返回给管理员，仅显示一次
- **不会通知用户**：用户不会收到密码被管理员重置的邮件

### 8.6 一次性登录 Token（邮件登录）

除了密码重置，还有邮件登录机制（`loginViaEmail()`，[account.go#L1409-L1459](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/account.go#L1409-L1459)）：

- 生成 15 分钟有效的一次性登录 Token（`GetTemporaryOneTimeAccessToken(userID, 60*15, true)`）
- 点击链接后直接登录（`oneTimeToken` 参数）
- 适用于用户没有设置密码的场景

---

## 九、更新检查的外部请求行为分析

### 9.1 更新检查触发机制

更新检查系统由 `updatesCache` 实现（[updates.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/updates.go)）。

**触发条件**：
1. **应用启动时**：`InitUpdates()` 创建缓存并立即发起一次检查（go 协程）
2. **定期检查**：每次调用 `AreAvailable()` 时，若距上次检查超过 12 小时则自动检查
3. **管理员手动触发**：访问 `/admin/updates?check=now` 时调用 `CheckNow()`

**启动初始化**（[updates.go#L108-L114](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/updates.go#L108-L114)）：

```go
func (app *App) InitUpdates() {
    if app.cfg.App.UpdateChecks {
        app.updates = newUpdatesCache(defaultUpdatesCacheTime) // 12 小时
    }
}

func newUpdatesCache(expiry time.Duration) *updatesCache {
    // ...
    go cache.CheckNow()  // 启动时异步检查一次
    return cache
}
```

### 9.2 外部请求详情

`newVersionCheck()` 函数（[updates.go#L116-L132](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/updates.go#L116-L132)）：

```go
func newVersionCheck() (string, error) {
    res, err := http.Get("https://version.writefreely.org")
    if err == nil && res.StatusCode == http.StatusOK {
        defer res.Body.Close()
        body, err := io.ReadAll(res.Body)
        return string(body), nil
    }
    return "", err
}
```

**请求特点**：

| 特性 | 值 | 说明 |
|---|---|---|
| **目标 URL** | `https://version.writefreely.org` | 官方版本检查端点 |
| **请求方法** | `GET` | 简单 GET 请求 |
| **User-Agent** | Go 默认 `Go-http-client/1.1` | **未设置自定义 UA** |
| **超时** | 无超时设置 | 可能导致挂起 |
| **响应体** | 纯文本版本号（如 `v0.16.0`） |  |

**与其他外部请求对比**：

OAuth 请求和 ActivityPub 请求都使用了自定义 User-Agent（[app.go#L1013-L1021](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/app.go#L1013-L1021)）：

```go
func ServerUserAgent(hostName string) string {
    hostUAStr := ""
    if hostName != "" {
        hostUAStr = "; +" + hostName
    }
    return "Go (" + serverSoftware + "/" + softwareVer + hostUAStr + ")"
}
```

但版本检查请求**没有**使用这个 UA，直接使用 Go 默认 UA，这可能会被某些防火墙拦截。

### 9.3 更新检查的隐私影响

**泄露的信息**：
- 实例 IP 地址（请求来源 IP）
- 请求时间（服务端可记录）
- 通过请求频率可推断实例活跃度

**未泄露的信息**：
- 实例域名（没有在请求中携带）
- 用户数据
- 数据库信息

### 9.4 版本比较与安全

版本比较使用 `CompareSemver()`（[semver.go](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/semver.go)），使用语义化版本号比较。

**检查失败的处理**：
- 检查失败时 `checkError` 不为 nil
- 在管理页面显示"检查失败"状态
- 不会影响实例正常运行
- 下次检查会重试

### 9.5 管理后台更新展示

每个管理页面的 `AdminPage` 结构都包含 `UpdateAvailable` 字段（[admin.go#L94-L104](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/admin.go#L94-L104)）：

```go
type AdminPage struct {
    UpdateAvailable bool
}

func NewAdminPage(app *App) *AdminPage {
    ap := &AdminPage{}
    if app.updates != nil {
        ap.UpdateAvailable = app.updates.AreAvailableNoCheck()
    }
    return ap
}
```

**注意**：使用 `AreAvailableNoCheck()` 而非 `AreAvailable()`，意味着在普通管理页面浏览时不会触发更新检查，只使用缓存结果。只有 `/admin/updates` 页面才会检查是否过期并刷新。

---

## 十、实例页面的存储更新机制深度分析

### 10.1 数据存储结构

实例页面内容存储在数据库 `appcontent` 表中，数据结构为 `instanceContent`（[admin.go#L86-L92](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/admin.go#L86-L92)）：

```go
type instanceContent struct {
    ID      string
    Type    string
    Title   sql.NullString
    Content string
    Updated time.Time
}
```

**字段说明**：

| 字段 | 类型 | 说明 |
|---|---|---|
| `ID` | `string` | 内容标识符，如 "about"、"contact"、"landing-banner" |
| `Type` | `string` | 内容类型：`"page"`（页面）或 `"section"`（区块） |
| `Title` | `sql.NullString` | 页面标题，可为 NULL |
| `Content` | `string` | Markdown 格式的内容 |
| `Updated` | `time.Time` | 最后更新时间 |

### 10.2 内置页面与区块类型

系统内置的可编辑内容（[admin.go#L411-L537](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/admin.go#L411-L537)）：

| ID | 类型 | 默认值来源 | 说明 |
|---|---|---|---|
| `about` | `page` | `defaultAboutPage()` | 关于页面 |
| `contact` | `page` | `defaultContactPage()` | 联系页面 |
| `privacy` | `page` | `defaultPrivacyPolicy()` | 隐私政策 |
| `landing-banner` | `section` | `defaultLandingBanner()` | 着陆页横幅 |
| `landing-body` | `section` | `defaultLandingBody()` | 着陆页主体 |
| `reader` | `section` | `defaultReaderBanner()` | Reader 区块标题 |

**特殊处理**：`landing` 页面实际上由 `landing-banner` 和 `landing-body` 两部分组成，在 `handleAdminUpdateSite` 中分别更新。

### 10.3 读取逻辑：回退到默认值

所有内置页面都遵循"数据库优先，缺省回退"的模式。以 `getAboutPage()` 为例（[pages.go#L22-L38](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/pages.go#L22-L38)）：

```go
func getAboutPage(app *App) (*instanceContent, error) {
    c, err := app.db.GetDynamicContent("about")
    if err != nil {
        return nil, err
    }
    if c == nil {
        c = &instanceContent{
            ID:      "about",
            Type:    "page",
            Content: defaultAboutPage(app.cfg),
        }
    }
    if !c.Title.Valid {
        c.Title = defaultAboutTitle(app.cfg)
    }
    return c, nil
}
```

**层级回退逻辑**：
1. 首先从数据库 `appcontent` 表查询
2. 数据库中没有记录 → 使用代码中的默认内容
3. 有记录但 Title 为 NULL → 使用默认标题
4. 有记录且 Title 有效 → 使用数据库中的值

这意味着管理员删除（清空）页面后，系统会自动回退到默认内容。

### 10.4 更新机制

更新操作通过 `UpdateDynamicContent()` 实现（[database.go#L2856-L2867](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/database.go#L2856-L2867)）：

```go
func (db *datastore) UpdateDynamicContent(id, title, content, contentType string) error {
    if db.driverName == driverSQLite {
        _, err = db.Exec("INSERT OR REPLACE INTO appcontent (...) VALUES (?, ?, ?, "+db.now()+", ?)", ...)
    } else {
        _, err = db.Exec("INSERT INTO appcontent (...) VALUES (...) "+db.upsert("id")+" title = ?, content = ?, updated = "+db.now(), ...)
    }
}
```

**更新特点**：
- **UPSERT 语义**：MySQL 使用 `ON DUPLICATE KEY UPDATE`，SQLite 使用 `INSERT OR REPLACE`
- **自动更新时间戳**：`updated` 字段设为当前时间
- **无版本历史**：每次更新直接覆盖，无法回退到历史版本
- **无审核机制**：保存即生效

### 10.5 管理后台的更新流程

`handleAdminUpdateSite()` 处理页面内容保存（[admin.go#L539-L569](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/admin.go#L539-L569)）：

```go
func handleAdminUpdateSite(app *App, u *User, w http.ResponseWriter, r *http.Request) error {
    vars := mux.Vars(r)
    id := vars["page"]

    // 白名单校验
    if id != "about" && id != "contact" && id != "privacy" && id != "landing" && id != "reader" {
        return impart.HTTPError{http.StatusNotFound, "No such page."}
    }

    if id == "landing" {
        err = app.db.UpdateDynamicContent("landing-banner", "", r.FormValue("banner"), "section")
        err = app.db.UpdateDynamicContent("landing-body", "", r.FormValue("content"), "section")
    } else if id == "reader" {
        err = app.db.UpdateDynamicContent(id, r.FormValue("title"), r.FormValue("content"), "section")
    } else {
        err = app.db.UpdateDynamicContent(id, r.FormValue("title"), r.FormValue("content"), "page")
    }
    // ...
}
```

**安全设计**：
- **ID 白名单**：只允许修改 5 个预定义页面，防止 SQL 注入或创建任意 ID 的内容
- Landing 页面特殊处理，拆分为两个内容块分别保存
- 保存后通过 302 重定向回编辑页面，避免重复提交

### 10.6 页面列表的展示逻辑

管理后台的 Pages 列表页面（`handleViewAdminPages()`，[admin.go#L411-L481](file:///d:/fz/0601-1/solo-dogfeeding/code/40-writefreely/admin.go#L411-L481)）展示逻辑较复杂：

1. 先从数据库获取所有 `content_type = "page"` 的记录
2. 检查是否包含 about/contact/privacy 三个默认页面
3. 如果缺少某个默认页面，用默认内容补全到列表中
4. 补全时同时设置默认标题

这确保了管理员始终能看到所有可编辑的页面，即使从未保存过。

### 10.7 无缓存的直读模式

实例页面内容**没有缓存层**，每次页面请求都会直接查询数据库 `appcontent` 表。对于访问量较大的 about/landing 等页面，可能成为数据库压力点。

**读取路径**：
- 首页 → `handleViewHome()` → `handleViewLanding()` → `getLandingBanner()` + `getLandingBody()` → 数据库
- `/about` → `handleTemplatedPage()` → `getAboutPage()` → 数据库
- `/contact` → `handleTemplatedPage()` → `getContactPage()` → 数据库
- `/privacy` → `handleTemplatedPage()` → `getPrivacyPage()` → 数据库

---

## 十一、关键设计总结

### 11.1 权限边界

- **单管理员模型**：只有 ID=1 的用户是管理员，无多管理员、角色分组或权限矩阵
- **信息隐藏**：未授权访问 /admin 返回 404 而非 403，避免暴露管理入口
- **防御纵深**：删除用户等危险操作有 Handler 层 + 业务层双重管理员校验
- **无 CSRF 保护的 Admin 路由**：所有 Admin POST 路由均缺少 `csrf.Protect()` 中间件，而用户设置页面 `/me/settings` 和 `/me/delete` 以及密码重置页 `/reset` 均有 CSRF 保护。HTTPS 站点 `SameSite=None` 进一步增加了 CSRF 风险

### 11.2 配置保存

- **双层存储**：内存（`app.cfg` 指针）+ 磁盘（INI 文件），Web 修改后同时更新两者
- **部分即时生效**：`LocalTimeline` 等少数配置有即时生效逻辑，其他依赖后续请求自然读取
- **配置丢失风险**：`config.Save()` 使用 `ini.Empty()` + 反射重建，手动添加的注释和额外字段会丢失
- **Server/DB 配置不暴露**：网络、数据库、OAuth 等敏感配置不在 Web 管理界面中暴露

### 11.3 缓存影响

- **缓存一致性不完整**：取消静音不刷新时间线缓存，删除用户不触发任何缓存清理
- **用户文章缓存极短**：4 秒过期，对性能帮助有限
- **时间线缓存主动失效**：仅静音用户时触发，使用 `memo.Memo` 自动过期机制
- **页面内容无缓存**：实例页面（about/contact/privacy 等）直接查数据库，高访问量时可能成为瓶颈
- **更新检查缓存**：12 小时过期，后台异步刷新，避免频繁外部请求

### 11.4 运行状态

- **按需采集**：系统监控数据仅在访问 /admin/monitor 时采集，无后台定时采集
- **无历史数据**：运行状态全部来自 Go runtime 实时快照，无持久化或趋势图
- **版本硬编码**：`softwareVer` 在编译时通过 `-ldflags` 注入，运行时无法更新
- **更新检查**：启动时异步检查一次，后续 12 小时缓存；管理员可手动强制检查

### 11.5 Session 与认证安全

- **双层密钥**：Cookie 使用独立的签名密钥（Auth）和加密密钥（Encrypt），保证完整性和机密性
- **登录限流**：3 秒内同一用户名只能登录一次，但仅按用户名限流且窗口短，防护有限
- **180 天超长会话**：无会话失效机制，密码修改后现有会话仍然有效
- **管理员禁止邮件重置**：密码重置功能对管理员账户禁用，防止通过邮件渠道提权

### 11.6 密码重置安全

- **Token 高强度**：32 字符 62 进制随机串，3 小时有效期，一次性使用
- **用户枚举防护**：无论用户是否存在，返回相同结果
- **明文存储隐患**：重置 Token 以明文存储在数据库中，数据库泄露则可用
- **旧 Token 不失效**：生成新 Token 不会使旧 Token 提前失效
- **管理员重置无通知**：管理员重置用户密码后，用户不会收到通知邮件

### 11.7 实例页面存储

- **UPSERT 模式**：使用 INSERT OR REPLACE / ON DUPLICATE KEY UPDATE 语义
- **三层回退**：数据库 → 默认标题 → 默认内容，确保页面始终可访问
- **无版本历史**：每次更新直接覆盖，无法回滚
- **白名单机制**：只允许修改预定义的 5 个页面 ID，防止越权操作
- **无缓存层**：所有页面内容直接从数据库读取
