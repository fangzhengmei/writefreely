# Collection 与站点路由

本文档基于 WriteFreely 源码，深入分析 Collection（集合/博客）与站点路由的实现机制，特别关注根路径分流、前缀匹配、用户数据关联和别名冲突处理的多层设计。

## 目录

1. [Collection 数据结构](#collection-数据结构)
2. [路由系统概述](#路由系统概述)
3. [站点根路径分流机制](#站点根路径分流机制)
4. [集合入口：带前缀与不带前缀的命中机制](#集合入口带前缀与不带前缀的命中机制)
5. [公开路径详解](#公开路径详解)
6. [用户集合管理](#用户集合管理)
7. [/me/posts/ 与 /me/c/ 的数据关联](#meposts-与-mec-的数据关联)
8. [文章列表](#文章列表)
9. [别名冲突处理：两层设计](#别名冲突处理两层设计)
10. [总结](#总结)

---

## Collection 数据结构

### 核心结构体

Collection（集合/博客）是 WriteFreely 的核心概念，代表一个用户的博客空间。其定义位于 [collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/collections.go#L49-L75)。

```go
type Collection struct {
    ID          int64          // 集合ID
    Alias       string         // 别名（URL路径）
    Title       string         // 标题
    Description string         // 描述
    Visibility  collVisibility // 可见性（位掩码）
    Format      string         // 格式：blog/novel/notebook
    OwnerID     int64          // 所有者用户ID
    PublicOwner bool           // 是否公开所有者
    Public      bool           // 是否公开
    hostName    string         // 运行时填充，用于构造 CanonicalURL
}
```

### 可见性级别

可见性使用位掩码实现，定义在 [collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/collections.go#L149-L167)：

| 级别 | 值 | 说明 |
|------|----|------|
| CollUnlisted | 0 | 未列出（默认） |
| CollPublic | 1 | 公开，可在公共时间线显示 |
| CollPrivate | 2 | 私有，只有所有者能访问 |
| CollProtected | 4 | 密码保护，需要密码访问 |

### 集合格式

集合有三种格式，影响文章展示方式：

- **blog**：博客格式，倒序排列（DESC），显示日期
- **novel**：小说格式，正序排列（ASC）
- **notebook**：笔记本格式

---

## 路由系统概述

路由定义位于 [routes.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/routes.go) 的 `InitRoutes` 函数中。

### 单用户 vs 多用户模式

WriteFreely 支持两种运行模式，通过 `App.SingleUser` 配置切换，全局变量 `isSingleUser` 在 `Serve()` 启动时赋值，见 [app.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/app.go#L451)。

**单用户模式** (`SingleUser = true`)：
- 整个站点就是一个集合（固定 ID = 1）
- 集合路由直接挂在根路径：`RouteCollections(handler, write.PathPrefix("/").Subrouter())`
- 草稿编辑路径前缀为 `/d`，与根路径下的文章 slug 避免冲突
- 新建文章路径：`/me/new`

**多用户模式** (`SingleUser = false`)：
- 每个用户可以有多个集合
- 通过三组独立的路由入口匹配集合（详见下节）
- 草稿无前缀 `/d`
- 新建文章路径：`/new`

### 用户级别控制

路由通过 `UserLevelFunc` 控制访问权限，定义在 [handle.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/handle.go#L31-L64)：

| 级别函数 | 枚举值 | 说明 |
|----------|--------|------|
| UserLevelNone | NoneType | 忽略用户状态，完全不检查 session |
| UserLevelOptional | OptionalType | 用户可选，有用户时尝试解析 session 获取用户 |
| UserLevelNoneRequired | NoneRequiredType | 必须未登录，已登录用户会被重定向 |
| UserLevelUser | UserType | 必须已登录，否则返回 ErrNotLoggedIn |
| UserLevelReader | — | 动态：站点私有则要求 User，否则为 Optional |

---

## 站点根路径分流机制

### 路由注册顺序（关键）

根路径 `/` 的路由注册在 **所有其他路由之后**，见 [routes.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/routes.go#L208-L217)：

```
注册顺序（自上而下）：
  1. /d/{action}/edit            (单用户草稿编辑)
  2. /d/{action}/meta
  3. {集合相关路由，详见下节}
  4. /d/{post}                   (单用户草稿查看)
  5. /                           ← 根路径最后注册！
```

由于 gorilla/mux 使用 **先注册先匹配** 的策略，所有更具体的路由都会优先于根路径被匹配。只有当所有其他路由都不匹配时，才会命中 `/` → `handleViewHome`。

### handleViewHome 的分层决策逻辑

`handleViewHome` 定义于 [app.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/app.go#L226-L261)，按以下优先级分流：

```
请求到达 /
│
├─► 单用户模式？ ──是──► 直接调用 handleViewCollection() 渲染博客首页
│
│  多用户模式 ↓
│
├─► forceLanding=1 参数？ ──是──► handleViewLanding() 登录注册页
│
├─► Chorus 模式 且 (站点非私有 或 用户已登录)？
│      └──是──► viewLocalTimeline() 本地读者时间线
│
├─► 用户已登录？ ──是──► handleViewPad() 写作编辑板
│
├─► 站点私有？ ──是──► viewLogin() 登录页
│
├─► 配置了自定义 LandingPath 且不是 "/"？
│      └──是──► 302 重定向到配置的路径
│
└──► 其他所有情况：handleViewLanding() 登录注册页
```

**关键修正**：之前误以为多用户模式下根路径总是展示 Landing 页面。实际上登录用户会直接进入写作板、Chorus 模式站点进入读者时间线。

---

## 集合入口：四层路由与命中机制

多用户模式下，集合相关的 URL 实际上分布在 **四个不同的路由层次** 上，而非三条"并列"的入口。理解它们的关键是：路由是按注册顺序尝试的，先注册先匹配；`/{post}` 是根级别的 catch-all 路由，承担了很多 fallback 职责。

### 完整路由注册顺序（多用户模式）

路由注册位于 [routes.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/routes.go#L205-L217)，按从上到下的顺序：

```
注册顺序（自上而下，先注册先匹配）：
  ① /{prefix:[@~$!\-+]}{collection}      → handleViewCollection
  ② /{collection}/                       → handleViewCollection
  ③ PathPrefix /{prefix?}{collection} + 子路由（RouteCollections）
       ├── /logout                       → handleLogOutCollection
       ├── /page/{page}                  → handleViewCollection
       ├── /archive/                     → handleViewCollection
       ├── /tag:{tag}                    → handleViewCollectionTag
       ├── /feed/                        → ViewFeed
       ├── /sitemap.xml                  → handleViewSitemap
       ├── /{slug}                       → CollectionPostOrStatic → viewCollectionPost
       └── /{slug}/edit 等
  ④ /{post}                              → handleViewPost    ← 根级 catch-all！
  ⑤ /                                    → handleViewHome
```

### 四个层次的定位与职责

| 层次 | 路由模式 | 处理函数 | 定位 |
|------|----------|----------|------|
| ① | `/{prefix}{collection}` | handleViewCollection | **前缀风格集合首页**：@/~/$ 等前缀 + 别名，直接进入集合视图 |
| ② | `/{collection}/` | handleViewCollection | **规范集合首页**：带末尾斜杠的标准集合 URL |
| ③ | `PathPrefix /{prefix?}{collection}` + 子路由 | 多个 | **集合子资源**：分页、归档、标签、文章、Feed 等所有带层级路径 |
| ④ | `/{post}` | handleViewPost | **根级 catch-all**：帖子直链 + 集合别名 fallback |
| ⑤ | `/` | handleViewHome | **根路径**：根据模式和用户状态分流 |

### 前缀字符说明

路由模式 `[@~$!\\-+]` 支持 6 种前缀字符：

| 前缀 | 字符 | 常见用途 |
|------|------|----------|
| @ | 艾特 | 类似微博/Twitter 的用户名风格 |
| ~ | 波浪号 | 传统 Unix 用户目录风格 |
| $ | 美元符 | 付费/专属内容标识 |
| ! | 感叹号 | 特殊/突出内容标识 |
| \- | 连字符 | 需注意与 slug 中的连字符区分 |
| + | 加号 | 增强/扩展内容标识 |

前缀存储在 `collectionReq.prefix` 中（仅在层次①和③中存在），后续用于构造分页、标签、归档等 URL，保证用户的自定义前缀风格被保留。

---

### `/myblog` 的完整命中链路（无前缀、无斜杠）

**这是最容易被误解的一条路径**。它不匹配层次①②③，而是匹配层次④的根级 catch-all 路由 `/{post}`，然后在函数内部通过 fallback 识别为集合别名。

完整链路见 [posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/posts.go#L314-L341) 的 `handleViewPost` 函数：

```
请求 /myblog
  │
  ├─► 路由层匹配：/{post} → friendlyID = "myblog"
  │
  ├─► handleViewPost 内部执行：
  │     ① 检查是否是保留页面（pages map）→ 否
  │     ② 检查是否是静态文件（含 "." 且非 raw）→ 否
  │     ③ GetCollection("myblog") → 查到了！
  │          └─► 301 MovedPermanently → "/myblog/"
  │
  └─► 如果不是集合，才继续按帖子 ID 处理
```

关键代码：
```go
// handleViewPost (posts.go L338-L341)
c, _ := app.db.GetCollection(friendlyID)
if c != nil {
    return impart.HTTPError{http.StatusMovedPermanently, 
        fmt.Sprintf("/%s/", friendlyID)}
}
```

**为什么不在路由层直接匹配集合别名？**
- 集合别名和帖子 ID 都是任意字符串，路由层面无法区分
- 只能在业务逻辑层通过数据库查询来判定：先查集合表，命中则重定向到规范 URL；否则按帖子处理
- 这是一种 **数据库驱动的 URL 路由** 设计

---

### 各种 URL 形式的命中路径对照

| 请求 URL | 路由层次 | 处理函数 | 最终结果 |
|----------|----------|----------|----------|
| `/@myblog` | ① | handleViewCollection | 直接渲染集合首页（带前缀风格） |
| `/myblog/` | ② | handleViewCollection | 直接渲染集合首页（规范 URL） |
| `/myblog` | ④ → fallback | handleViewPost → 301 | 重定向到 `/myblog/`（集合别名识别） |
| `/~oldblog/feed/` | ③ 子路由 | ViewFeed | 集合 RSS Feed |
| `/tech-notes/tag:go` | ③ 子路由 | handleViewCollectionTag | 标签过滤页 |
| `/myblog/hello-world` | ③ 子路由 `/` + `/{slug}` | viewCollectionPost | 集合内文章页 |
| `/abcdefghij` | ④ → 帖子逻辑 | handleViewPost | 帖子直链（10 字符 ID） |
| `/about` | ④ → 保留页 | handleViewPost → 模板 | 静态关于页 |

---

### 层次④与层次③的关系：两条独立的命名空间

层次④（根级 `/{post}`）和层次③（PathPrefix 子路由的 `/{slug}`）看起来很像，都是单段路径参数，但它们属于 **完全不同的命名空间**：

| 维度 | 层次④ `/{post}` | 层次③ 子路由 `/{slug}` |
|------|-----------------|----------------------|
| 位置 | 根路由 write 上 | PathPrefix 子路由上 |
| 参数含义 | 帖子 ID / 集合别名 fallback | 集合内的文章 slug |
| 命名空间 | 全局唯一（posts.id + collections.alias） | 单个集合内唯一（posts.slug） |
| 长度约束 | 帖子 ID 固定 10 字符 | slug 任意长度 |
| 集合上下文 | 无（posts 可能无集合） | 有（collection_id 非 NULL） |

**关系总结**：
- 层次①②③是**集合域**的路由：都以集合别名为前缀，后续路径在集合内部
- 层次④是**全局域**的路由：帖子直链、保留页面、集合别名 fallback
- 两者通过 `handleViewPost` 中的 `GetCollection` 检查桥接起来
- 前缀入口（层次①）和规范入口（层次②）直接命中集合，不需要经过 fallback

---

## 公开路径详解

### RouteCollections 子路由

所有集合的子路径（分页、归档、标签、文章等）通过 `RouteCollections` 函数统一注册，见 [routes.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/routes.go#L222-L240)：

| 子路径 | 处理函数 | 说明 |
|--------|----------|------|
| `/logout` | `handleLogOutCollection` | 集合密码保护的"登出"（清除授权 session） |
| `/page/{page:[0-9]+}` | `handleViewCollection` | 分页 |
| `/archive/` | `handleViewCollection` | 归档视图（通过 `isArchiveView()` 检测） |
| `/archive/page/{page}` | `handleViewCollection` | 归档分页 |
| `/lang:{lang:[a-z]{2}}` | `handleViewCollectionLang` | 语言过滤 |
| `/tag:{tag}` | `handleViewCollectionTag` | 标签过滤 |
| `/tag:{tag}/feed/` | `ViewFeed` | 标签 RSS/Atom Feed |
| `/sitemap.xml` | `handleViewSitemap` | 集合站点地图 |
| `/feed/` | `ViewFeed` | 集合 RSS/Atom Feed |
| `/email/confirm/{subscriber}` | `handleConfirmEmailSubscription` | 邮件订阅确认 |
| `/email/unsubscribe/{subscriber}` | `handleDeleteEmailSubscription` | 邮件退订 |
| `/{slug}` | `CollectionPostOrStatic` | **文章页或静态文件**（见下） |
| `/{slug}/edit` | `handleViewPad` | 编辑文章（UserLevelUser） |
| `/{slug}/edit/meta` | `handleViewMeta` | 编辑文章元数据 |
| `/{slug}/` | `handleCollectionPostRedirect` | 带末尾斜杠 → 302 重定向到无斜杠版本 |

### CollectionPostOrStatic：文章 vs 静态文件的分流

`/{slug}` 路由既可能匹配文章 slug，也可能匹配静态资源路径（如 `style.css`、`favicon.ico`）。`CollectionPostOrStatic`（[handle.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/handle.go#L469-L483)）的分流逻辑：

```
URL 包含 "." ？
├─► 是，且不是 raw 请求（.json/.xml/.md/.txt）→ 走静态文件服务器 shttp.ServeHTTP
└─► 否 → 调用 viewCollectionPost 渲染文章
```

### handleViewCollection 处理流程

集合首页/分页/归档视图的处理流程，见 [collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/collections.go#L856-L1005)：

```
① processCollectionRequest  → 解析 prefix/alias，大写→小写重定向
② checkUserForCollection    → 从 session 获取当前用户
③ getCollectionPage         → 从 vars 中解析页码，默认 1
④ processCollectionPermissions → 核心权限+fallback 层（见别名冲突章节）
⑤ newDisplayCollection      → 计算 TotalPosts、TotalPages，页码越界则重定向到最后一页
⑥ GetPosts                  → 从 DB 查询当前页的文章列表
⑦ 模板渲染 + 视图统计（非所有者、非 bot 才计数）
```

---

## 用户集合管理

### 管理后台路由

用户登录后可以管理自己的集合，路由位于 `/me/c/` 下，见 [routes.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/routes.go#L98-L102)：

| 路径 | 处理函数 | UserLevel | 说明 |
|------|----------|-----------|------|
| `/me/c/` | `viewCollections` | User | 我的博客列表页 |
| `/me/c/{collection}` | `viewEditCollection` | User | 编辑博客设置 |
| `/me/c/{collection}/stats` | `viewStats` | User | 统计数据 |
| `/me/c/{collection}/subscribers` | `handleViewSubscribers` | User | 订阅者管理 |

### viewCollections：博客列表

`viewCollections` 定义于 [account.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/account.go#L820-L859)：

```go
// 核心数据查询：
c, err := app.db.GetCollections(u, app.cfg.App.Host)     // 所有集合
uc, _ := app.db.GetUserCollectionCount(u.ID)              // 集合数量（用于显示上限）
```

页面展示内容：
- 现有集合卡片：标题、描述、可见性、访问量
- 集合数量配额：已用 vs 总量，超限则禁用新建
- 创建新博客表单：别名、标题输入框
- 每个集合的入口链接：编辑 / 统计 / 订阅者

### GetCollections 与 GetPublishableCollections

两者定义于 [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/database.go#L1935-L1983)：

```go
// GetCollections：按 owner_id 查询所有集合，ID 升序
func (db *datastore) GetCollections(u *User, hostName string) (*[]Collection, error)

// GetPublishableCollections：封装 GetCollections，确保至少有 1 个集合，否则报错
// 用于文章页的"发布到博客"下拉功能
func (db *datastore) GetPublishableCollections(u *User, hostName string) (*[]Collection, error)
```

### 创建集合：newCollection

`newCollection` 定义于 [collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/collections.go#L427-L513)：

```
① 解析 alias + title（支持 JSON/表单两种提交方式）
② alias 为空则从 title 通过 getSlug 生成
③ 认证：JSON 请求检查 Authorization header，Web 请求检查 session
④ 检查用户是否被禁言（silenced）
⑤ 调用 author.IsValidUsername 验证别名合法性：
   - 长度 ≥ MinUsernameLen
   - 不与保留字冲突（about、admin、api、read 等）
   - 不与 pages/ 目录下的静态页面文件名冲突
   - 正则匹配（字母/数字/连字符）
⑥ 调用 db.CreateCollection → 触发帖子 ID 冲突检查 + DB 唯一约束
⑦ 成功则 302 到 /me/c/
```

---

## /me/posts/ 与 /me/c/ 的数据关联

### viewArticles：草稿 + 匿名文章列表

`viewArticles` 定义于 [account.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/account.go#L774-L818)，页面标题为 "Drafts"。

核心数据查询：
```go
// 查询 ①：匿名/草稿文章（collection_id 为 NULL 的帖子）
p, err := app.db.GetAnonymousPosts(u, 1)

// 查询 ②：可发布的集合（用于"移动到博客"功能）
c, err := app.db.GetPublishableCollections(u, app.cfg.App.Host)
```

### GetAnonymousPosts 的 SQL 条件

`GetAnonymousPosts` 定义于 [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/database.go#L2129-L2166)：

```sql
SELECT ... FROM posts 
WHERE owner_id = ? 
  AND collection_id IS NULL   -- ← 关键：未分配到任何集合
ORDER BY created DESC
```

### posts 表的 collection_id 字段是关联的核心

posts 表通过 `collection_id` 字段与 collections 表关联，这是整个系统的枢纽：

| collection_id 值 | 含义 | 显示位置 |
|------------------|------|----------|
| NULL | 匿名/草稿文章，未分配给任何集合 | `/me/posts/` Drafts 页面 |
| 非 NULL 且等于某个集合 ID | 已发布到该集合的文章 | 对应集合首页 `/alias/` |

### 模板中的关联交互：move to 功能

模板 [articles.tmpl](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/templates/user/articles.tmpl) 将两组数据整合在一起：

```html
<!-- 每篇匿名文章下方都有"移动到"操作 -->
{{ if $.Collections }}
  {{if gt (len $.Collections) 1}}
    <!-- 有多个集合 → 下拉选择 -->
    <select onchange="postActions.multiMove(this, '{{.ID}}', ...)">
      {{range $.Collections}}
        <option value="{{.Alias}}">{{.DisplayTitle}}</option>
      {{end}}
    </select>
  {{else}}
    <!-- 只有 1 个集合 → 直接按钮 -->
    <a onclick="postActions.move(this, '{{$el.ID}}', '{{.Alias}}', ...)">
      move to {{.DisplayTitle}}
    </a>
  {{end}}
{{ end }}
```

**数据流动总结**：

```
/me/posts/ (viewArticles)
  │
  ├──► GetAnonymousPosts(owner_id, collection_id IS NULL)
  │       ↓
  │     草稿文章列表
  │
  ├──► GetPublishableCollections(owner_id)
  │       ↓
  │     用户所有集合
  │
  └──► 模板渲染：草稿列表 + 每篇草稿的"发布到集合"下拉
               ↓ 用户点击"move to"
             POST /api/posts/claim 或 /api/collections/{alias}/collect
               ↓
             UPDATE posts SET collection_id = ?, slug = ? WHERE id = ?
               ↓
             文章从 /me/posts/ 消失，出现在 /me/c/{alias} 对应的集合首页
```

---

## 文章列表

### GetPosts：核心查询

文章列表通过 `GetPosts` 函数获取，定义于 [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/database.go#L1303-L1369)。

函数签名：
```go
func (db *datastore) GetPosts(cfg *config.Config, c *Collection, page int, 
    includeFuture, forceRecentFirst, includePinned bool, contentType PostType) 
    (*[]PublicPost, error)
```

核心参数说明：
- `page`：页码，**为 0 时返回最多 1000 篇**（用于导出等全量场景）
- `includeFuture`：是否包含未来发布的文章（所有者视图可见）
- `forceRecentFirst`：强制按最新排序（覆盖 novel 格式的正序）
- `includePinned`：是否包含置顶文章（默认排除，置顶用单独的 `GetPinnedPosts` 查询）
- `contentType`：postArch 归档视图时使用更大的每页数量 `postsPerArchPage`

### SQL 构造逻辑

```sql
SELECT postCols 
FROM posts 
WHERE collection_id = ?
  AND pinned_position IS NULL        -- includePinned=false 时追加
  AND created <= NOW()               -- includeFuture=false 时追加
ORDER BY created DESC/ASC            -- blog→DESC, novel→ASC
LIMIT start, pagePosts
```

### 分页与越界处理

每页文章数：`postsPerPage`（blog 和 novel 格式当前相同）。

总页数计算（[collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/collections.go#L910)）：
```go
coll.TotalPages = int(math.Ceil(float64(coll.TotalPosts) / float64(ppp)))
```

**页码越界**：如果请求页码超过总页数，自动 302 重定向到最后一页（[collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/collections.go#L911-L916)）。

### 文章计数：GetPostsCount

`GetPostsCount` 定义于 [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/database.go#L1279)：

```sql
SELECT COUNT(*) 
FROM posts 
WHERE collection_id = ? 
  AND pinned_position IS NULL 
  AND created <= NOW()  -- includeFuture=false 时追加
```

注意：置顶文章不计入总数（因为在列表顶部单独展示）。

---

## 别名冲突处理：两层设计

WriteFreely 的别名系统面临多个命名空间的共享问题：
- 集合别名 vs 帖子 ID（都是 URL 路径段）
- 集合别名 vs 保留字/静态页面文件名
- 旧别名 vs 新别名（用户改名后）

这些冲突通过 **创建时防御** 和 **访问时 fallback** 两层机制来处理。

### 第一层：创建时防御（事前检查）

#### 1a. 别名合法性校验（IsValidUsername）

`author.IsValidUsername` 定义于 [author/author.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/author/author.go#L110-L137)：

```
检查项：
  ① 长度 ≥ MinUsernameLen（配置项，防止单字符别名与路由前缀冲突）
  ② 遍历 pages/ 目录，所有静态页面文件名加入保留字集合
  ③ 硬编码保留字检查（reservedUsernames）：
     about, admin, api, app, apps, auth, blog, blogs, contact,
     dashboard, dev, developer, developers, faq, feed, feeds,
     help, index, invite, invites, log, login, logout, logs,
     me, new, news, oauth, page, pages, privacy, read, reader,
     register, registration, settings, signup, tag, tags, tos,
     update, updates, user, users, yourname, ...
  ④ 正则：^[a-zA-Z0-9][a-zA-Z0-9_-]{0,}$
```

#### 1b. 帖子 ID 冲突检查（PostIDExists）

帖子 ID 长度固定为 10 字符（`minIDLen = maxIDLen = 10`，见 [posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/posts.go#L45-L46)）。

`CreateCollection` 中检查，见 [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/database.go#L312-L315)：

```go
func (db *datastore) CreateCollection(...) (*Collection, error) {
    if db.PostIDExists(alias) {
        return nil, impart.HTTPError{http.StatusConflict, "Invalid collection name."}
    }
    // INSERT ...
}
```

`PostIDExists` 实现（[database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/database.go#L1154-L1158)）：
```sql
SELECT 1 FROM posts WHERE id = ?
```

**为什么需要这个检查**：因为 `/abcdefghij` 这种 10 字符路径既可能匹配集合路由也可能匹配帖子路由。如果允许创建与帖子 ID 同名的集合，旧的帖子链接将永远无法被访问！

#### 1c. 数据库唯一约束（最后防线）

`collections` 表的 `alias` 列有 UNIQUE 约束。插入时触发重复键错误，见 [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/database.go#L320-L322)：

```go
if db.isDuplicateKeyErr(err) {
    return nil, impart.HTTPError{http.StatusConflict, "Collection already exists."}
}
```

### 第二层：访问时 Fallback（事后补救）

即使创建时做了完整检查，仍然有两种场景需要在访问时处理：
1. 先有了帖子 ID，后来才加了"禁止与帖子 ID 同名"的限制 → 旧帖子链接需要保留可访问性
2. 用户后来修改了集合别名 → 旧别名的所有链接不应失效

这层逻辑位于 `processCollectionPermissions`（集合视图）和 `viewCollectionPost`（文章视图）两个函数中。

#### 2a. 集合视图的 Fallback 链

`processCollectionPermissions` 定义于 [collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/collections.go#L725-L825)：

```
DB 查询 GetCollection(alias) 失败（404）？
│
├─► 场景 A：自定义域名但无集合 → 静默返回 nil（后续展示 404）
│
├─► 场景 B：别名长度在帖子 ID 范围（10字符）？
│      └─► 检查 PostIDExists(alias)
│           └─► 是 → 301 MovedPermanently 重定向到 /{alias} 帖子直链
│
└─► 场景 C：检查 collectionredirects 表
       └─► 存在重定向记录 → 302 Found 重定向到 /{newAlias}/
```

代码片段（[collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/collections.go#L746-L757)）：
```go
if len(cr.alias) >= minIDLen && len(cr.alias) <= maxIDLen {
    if app.db.PostIDExists(cr.alias) {
        return nil, impart.HTTPError{http.StatusMovedPermanently, "/" + cr.alias}
    }
}
newAlias := app.db.GetCollectionRedirect(cr.alias)
if newAlias != "" {
    return nil, impart.HTTPError{http.StatusFound, "/" + newAlias + "/"}
}
```

#### 2b. 文章视图的 Fallback 链

`viewCollectionPost` 定义于 [posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/posts.go#L1464-L1516)，同样有类似的 Fallback：

```
查询集合 GetCollection(cr.alias) 失败（404）？
└─► 检查 collectionredirects 表
    └─► 存在重定向记录 → 302 到 /{newAlias}/{slug}（保留文章路径！）
```

**为什么分两处处理**：因为集合首页和集合内文章页的重定向目标不同：
- `/oldalias/` → `/newalias/`
- `/oldalias/my-post` → `/newalias/my-post`（slug 部分需要保留）

两层分别构造不同的重定向 URL。

### 集合重定向的写入：用户名/别名变更时

当用户修改用户名（单用户模式下等同于集合别名）时，在 `UpdateUserSettings` 事务中写入重定向记录，见 [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/database.go#L2283-L2302)：

```sql
-- ① 更新集合主表中的 alias
UPDATE collections SET alias = ? WHERE alias = ? AND owner_id = ?

-- ② 删除已有的反向链（新别名→其他），避免循环
DELETE FROM collectionredirects WHERE prev_alias = ?  -- newUsername

-- ③ 更新链式重定向（old→mid 变为 old→new）
UPDATE collectionredirects SET new_alias = ? WHERE new_alias = ?

-- ④ 添加本次的 old→new 记录
INSERT INTO collectionredirects (prev_alias, new_alias) VALUES (?, ?)
```

### 集合删除时的清理

删除集合时，会清理所有指向该集合的重定向记录，见 [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/database.go#L2599-L2607)：

```sql
DELETE FROM collectionredirects WHERE new_alias = ?
```

### 两层处理的设计原因

| 层次 | 处理时机 | 目标场景 | 无法替代对方的原因 |
|------|----------|----------|-------------------|
| 第一层：创建时防御 | CreateCollection 阶段 | 阻止明显不合法的别名被创建 | 无法处理"先有帖子后有限制"的历史遗留；无法处理用户改名的旧链接 |
| 第二层：访问时 Fallback | 每次访问 404 时 | 历史帖子链接、旧别名重定向、slug/alias 模糊匹配 | 性能开销大（每次 404 都要多查 1-2 张表）；无法给出友好的创建时错误提示 |

两层缺一不可：第一层减少运行时的 Fallback 次数（性能），第二层保证 URL 长期稳定（SEO 和用户体验）。

---

## 总结

WriteFreely 的 Collection 路由系统设计精巧，核心架构要点：

1. **路由注册顺序至关重要**：集合路由 → 草稿路由 → 根路径，确保具体路由优先于模糊路由。
2. **三条入口覆盖所有集合访问方式**：带前缀无斜杠、无前缀有斜杠、前缀可选子路由，再加上"文章路由找不到就重定向到集合"的 fallback，构成完整的 URL 设计。
3. **根路径智能分流**：根据模式（单用户/多用户）、配置（Chorus/私有站点）、用户状态（登录/未登录）动态决定首页内容。
4. **`collection_id` 字段是数据枢纽**：NULL 值表示草稿（`/me/posts/`），非 NULL 值表示已发布（对应集合首页）。
5. **`/me/posts/` 与 `/me/c/` 通过 GetPublishableCollections 桥接**：草稿页面加载用户的集合列表，实现"move to"发布功能。
6. **别名冲突的两层防御**：创建时的三道防线（保留字→帖子ID→DB唯一约束）+ 访问时的两条 Fallback（帖子ID重定向→旧别名重定向），既保障性能又保障 URL 稳定性。
7. **旧别名重定向分两个入口实现**：集合视图和文章视图分别处理，因为重定向 URL 的构造方式不同（集合首页追加 `/`，文章页追加 slug）。
