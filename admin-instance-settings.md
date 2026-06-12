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

## 五、关键设计总结

### 5.1 权限边界

- **单管理员模型**：只有 ID=1 的用户是管理员，无多管理员、角色分组或权限矩阵
- **信息隐藏**：未授权访问 /admin 返回 404 而非 403，避免暴露管理入口
- **防御纵深**：删除用户等危险操作有 Handler 层 + 业务层双重管理员校验
- **无 CSRF 保护的 Admin 路由**：Admin Handler 未使用 `csrf.Protect()` 中间件包裹，而用户设置页面 `/me/settings` 和 `/me/delete` 使用了 CSRF 保护。这可能是管理后台的安全隐患

### 5.2 配置保存

- **双层存储**：内存（`app.cfg` 指针）+ 磁盘（INI 文件），Web 修改后同时更新两者
- **部分即时生效**：`LocalTimeline` 等少数配置有即时生效逻辑，其他依赖后续请求自然读取
- **配置丢失风险**：`config.Save()` 使用 `ini.Empty()` + 反射重建，手动添加的注释和额外字段会丢失
- **Server/DB 配置不暴露**：网络、数据库、OAuth 等敏感配置不在 Web 管理界面中暴露

### 5.3 缓存影响

- **缓存一致性不完整**：取消静音不刷新时间线缓存，删除用户不触发任何缓存清理
- **用户文章缓存极短**：4 秒过期，对性能帮助有限
- **时间线缓存主动失效**：仅静音用户时触发，使用 `memo.Memo` 自动过期机制
- **页面内容无缓存**：实例页面（about/contact/privacy 等）直接查数据库

### 5.4 运行状态

- **按需采集**：系统监控数据仅在访问 /admin/monitor 时采集，无后台定时采集
- **无历史数据**：运行状态全部来自 Go runtime 实时快照，无持久化或趋势图
- **版本硬编码**：`softwareVer` 在编译时通过 `-ldflags` 注入，运行时无法更新
