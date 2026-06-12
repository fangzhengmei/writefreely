# WriteFreely 文章编辑与发布流程分析

## 一、核心数据模型

### 1.1 Post 结构定义

文章（Post）的状态不通过单独的 `status` 字段管理，而是通过以下关键字段的组合判断：

| 字段 | 类型 | 含义 | 状态判断 |
|------|------|------|----------|
| `owner_id` | `null.Int` | 文章所有者用户 ID | NULL = 匿名文章；有值 = 用户所有 |
| `collection_id` | `null.Int` | 所属博客集合 ID | NULL = 草稿/独立文章；有值 = 已发布到博客 |
| `slug` | `null.String` | URL 友好的文章别名 | 有值 = 集合内文章；NULL = 独立草稿 |
| `title` + `content` | 字符串 | 标题与正文 | 两者都为空 = 已取消发布 |
| `created` | `time.Time` | 创建/发布时间 | 大于当前时间 = 定时发布文章 |

数据结构定义在 [posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/posts.go#L102-L127) 的 `Post` 结构体中。

### 1.2 Collection 可见性级别

博客集合（Collection）有四种可见性（位掩码），定义在 [collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/collections.go#L155-L159)：

```go
const CollUnlisted collVisibility = 0       // 0: 未列出
const (
    CollPublic    collVisibility = 1 << iota  // 1: 公开
    CollPrivate                               // 2: 私有（仅作者可见）
    CollProtected                             // 4: 密码保护
)
```

判断方法位于 [collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/collections.go#L218-L228)：
- `IsPrivate()` → 检查 `CollPrivate` 位
- `IsProtected()` → 检查 `CollProtected` 位
- `IsPublic()` → 检查 `CollPublic` 位

---

## 二、草稿保存机制

草稿保存分为**浏览器本地**和**服务端数据库**两个层级。

### 2.1 浏览器本地草稿 (localStorage)

在编辑器模板 [pad.tmpl](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/templates/pad.tmpl#L149-L160) 中定义：

- **新文章**：`draftDoc = 'lastDoc'`，保存到 localStorage key `lastDoc`
- **编辑已有文章**：`draftDoc = 'draft{PostId}'`，保存到 `draft{id}`

相关 JS 逻辑：
- 自动保存：通过 `typingTimer` 在输入 200ms 后调用 `H.save($writer, draftDoc)`
- 加载草稿：`H.load($writer, draftDoc, true, updated)`，并对比服务器 `updated` 时间判断是否有别处修改
- 字体偏好：`draft{id}font` 或 `padFont`
- 冲突检测：若本地 `draft{id}-published` 时间早于服务器 `updated`，显示 "edited elsewhere" 警告

### 2.2 服务端草稿（数据库）

服务端"草稿"的判定条件：**文章有 `owner_id` 但 `collection_id` 为 NULL**。

草稿创建流程位于 [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/database.go#L675-L772) 的 `CreatePost()`：
- `collID <= 0` → `ownerCollID.Valid = false`，即 `collection_id` 为 NULL
- 仅当指定了集合时才生成 `slug` 字段

用户草稿列表查询条件见 [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/database.go#L2030)：
```sql
SELECT COUNT(*) FROM posts WHERE owner_id = ? AND collection_id IS NULL
```

---

## 三、发布状态转换

### 3.1 状态转换总图

```
创建新文章 ──▶ [草稿: owner_id有值, collection_id=NULL]
                    │
                    │ ClaimPosts (设置 collection_id + slug)
                    ▼
           [已发布: owner_id有值, collection_id有值]
                    │
                    │ DispersePosts (collection_id 设为 NULL)
                    ▼
           [退回草稿状态]
                    │
                    │ 将 title + content 清空
                    ▼
           [已取消发布 (Gone)]
```

### 3.2 草稿 → 发布到集合（ClaimPosts）

API 端点：`POST /api/collections/{alias}/collect` 或 `POST /api/posts/claim`

处理函数：
- HTTP 层：[posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/posts.go#L950-L1014) `addPost()`
- DB 层：[database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/database.go#L1698-L1896) `ClaimPosts()`

核心 SQL（已有所有权的文章）见 [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/database.go#L1804)：
```sql
UPDATE posts SET collection_id = ?, slug = ? WHERE id = ? AND owner_id = ?
```

slug 生成逻辑：
1. 用户指定 `post.Slug` → 使用用户值
2. 否则：优先从 `Title` 生成，其次从 `Content` 生成（`getSlugFromPost`）
3. 最后 fallback：使用文章 ID

slug 重复处理：`AttemptClaim()` 递归调用 `id.GenSafeUniqueSlug()` 添加随机后缀

### 3.3 已发布 → 草稿（DispersePosts）

API 端点：`POST /api/posts/disperse`

处理函数：
- HTTP 层：[posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/posts.go#L1016-L1049) `dispersePost()`
- DB 层：[database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/database.go#L1621-L1696) `DispersePosts()`

核心 SQL 见 [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/database.go#L1671)：
```sql
UPDATE posts SET collection_id = NULL WHERE id = ? AND owner_id = ?
```

### 3.4 前端触发：文章移动操作

前端 JS 位于 [static/js/postactions.js](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/static/js/postactions.js)：

- 移动到集合：`He.postJSON("/api/collections/" + collAlias + "/collect", ...)`
- 移为草稿（特殊别名 `|anonymous|`）：`He.postJSON("/api/posts/disperse", ...)`

### 3.5 取消发布 (Unpublished/Gone)

判定条件：`title == "" && content == ""`

检查位置：
- 独立文章：[posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/posts.go#L454-L467) `handleViewPost()`
- 集合文章：[posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/posts.go#L1594-L1597) `viewCollectionPost()`
- DB 查询：[database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/database.go#L1142-L1144) `GetEditablePost()`

---

## 四、编辑器页面流程

### 4.1 路由定义

编辑器路由在 [routes.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/routes.go#L196-L216)：

| 模式 | 单用户模式前缀 | 路径 | 说明 |
|------|----------------|------|------|
| 新建 | `/me/new` | `/new` | `handleViewPad` |
| 编辑草稿 | `/d/{id}/edit` | `/{id}/edit` | `handleViewPad` |
| 编辑集合文章 | `/{slug}/edit` | `/{slug}/edit` | `handleViewPad` |
| 编辑元数据 | `/d/{id}/meta` + `/{slug}/edit/meta` | 同上 | `handleViewMeta` |

单用户模式下，草稿路径加前缀 `/d` 避免与集合 slug 冲突。

### 4.2 编辑器加载逻辑

`handleViewPad()` 在 [pad.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/pad.go#L23-L121)：

1. 无 `action`/`slug` → 新建文章，`Editing = false`
2. 有 `slug` → 集合文章编辑：`getRawCollectionPost()`，校验 `OwnerID`
3. 有 `action`（ID）→ 独立草稿编辑：`getRawPost()`
4. 权限校验：非作者 302 跳回文章页面
5. 反缓存头：`Cache-Control: no-cache, no-store, must-revalidate`

### 4.3 编辑器模板

主模板 [pad.tmpl](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/templates/pad.tmpl)：

- 目标选择菜单：列出所有可发布博客 + "Draft" 选项
- 工具区：元数据编辑、主题切换、预览、发布按钮
- 发布按钮禁用条件：内容为空 或 编辑时内容未改变（`$writer.el.value == origDoc`）
- 异步发布：调用 `newPost()` 或 `existingPost()`，成功后清除对应 localStorage 草稿

---

## 五、文章保存/更新流程

### 5.1 创建新文章 (newPost)

HTTP 处理：[posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/posts.go#L553-L699) `newPost()`

数据流：
1. 鉴权：`Authorization` header → `accessToken`；否则取 Cookie Session
2. 参数解析：JSON Body 或 Form Data
3. 内容校验：标题+正文均为空 → `ErrNoPublishableContent`
4. 字体校验：无效值 fallback 为 `"norm"`
5. DB 调用：
   - 有 token → `CreateOwnedPost()`（内部调 `CreatePost`）
   - 有 session → 直接 `CreatePost(userID, collID, p)`
6. 后置操作：
   - 非私有集合且开启联邦 → `go federatePost()`
   - 开启邮件订阅 → 插入 `PostJob{Action: "email"}`

### 5.2 更新已有文章 (existingPost)

HTTP 处理：[posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/posts.go#L701-L832) `existingPost()`

DB 更新：[database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/database.go#L776-L854) `UpdateOwnedPost()`

特点：**增量更新**，仅更新传入字段（通过指针 nil 判断）：
- `slug`、`content`、`title`、`language`、`rtl`、`font`、`created`
- 始终更新 `updated = NOW()`
- 授权条件：`WHERE id = ? AND owner_id = ?`

Web 表单更新成功后 302 跳转：
- 集合文章 → `/{collAlias}/{slug}/edit/meta`
- 独立草稿 → `/d/{id}/meta`（单用户）或 `/{id}/meta`

---

## 六、页面渲染与发布状态关系

### 6.1 三种渲染入口

#### A. 独立文章页面 (handleViewPost)

路由：`/{post}` → [posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/posts.go#L314-L544)

权限与保护逻辑（按顺序）：
1. 独立文章直接查询 `posts` 表，不通过集合
2. 若所属集合是 `Private`/`Protected` → 非作者返回 404（`protectDraft`）
3. `title == "" && content == ""` → 已取消发布，返回 `ErrPostUnpublished` (410)
4. 作者被 silenced（禁言）且访问者不是作者 → 返回 404
5. 非作者 + 受保护集合 → 返回 404

模板：`templates/post.tmpl`，传入结构含 `IsOwner` 字段控制编辑按钮。

#### B. 集合文章页面 (viewCollectionPost)

路由：`/{collection}/{slug}` → [posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/posts.go#L1465-L1702)

集合权限检查（先于文章查询）见 [collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/collections.go#L725-L825) `processCollectionPermissions()`：
- `IsPrivate()` + 非作者 → `ErrCollectionNotFound`（彻底隐藏存在）
- `IsProtected()` → 检查 Session Cookie 中的授权，未授权则渲染密码输入页

文章状态检查：
- `content == "" && title == ""` → 410 Gone
- silenced 作者 + 非作者/非管理员 → 404
- 定时发布文章（`created > NOW()`）：仅作者可见（通过 `GetPosts` 的 `includeFuture` 参数控制）

#### C. 集合文章列表 (handleViewCollection)

文章列表查询：[database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/database.go#L1303-L1369) `GetPosts()`

```sql
WHERE collection_id = ? 
  AND pinned_position IS NULL
  AND created <= NOW()   -- 非作者时加此条件（includeFuture=false）
ORDER BY created DESC    -- blog 格式；novel 格式为 ASC
```

关键：`includeFuture` 传入 `isCollOwner`，因此作者能看到未到时间的定时文章。

### 6.2 Markdown 渲染管道

核心函数：[postrender.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/postrender.go)

渲染时机：
1. **列表视图**（博客首页）：`formatContent(cfg, c, isOwner, false)`
   - 遇到 `<!--more-->` → 生成 `HTMLExcerpt` 截断
   - 遇到 `<!--paid-->` + 非所有者 → 截断并显示 Coil 会员提示
2. **详情页**：`formatContent(cfg, c, isOwner, true)`
   - 显示全文，但付费内容仍按权限截断
3. **签名追加**：`augmentContent()` → 若集合设置 `Signature` 且无 `<!--nosig-->`，则在正文末尾追加

Markdown → HTML 过程见 [postrender.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/postrender.go#L146-L183) `applyMarkdownSpecial()`：
- blackfriday 解析（表格、围栏代码、自动链接、删除线、标题 ID）
- 标签链接替换：`#tag` → `<a href="/{coll}/tag:tag">`
- @提及替换：`@user@host` → `/@/user@host` 链接
- bluemonday 白名单过滤 XSS
- YouTube 自动播放禁用

### 6.3 渲染控制字段

| 字段/标记 | 效果 |
|-----------|------|
| `Font` | `norm`=衬线, `sans`=无衬线, `wrap`=等号宽, `code`=原始代码 |
| `Language` | 影响 `slug` 生语言、日期本地化 |
| `RTL` | True 时设置 `dir="rtl"` |
| `<!--more-->` | 列表页截断点，详情页中移除 |
| `<!--paid-->` | 付费内容截断点（需集合设置 Monetization） |
| `<!--nosig-->` | 不追加集合签名 |
| `<!--emailsub-->` | 渲染为邮件订阅表单 |

---

## 七、关键文件索引

| 文件 | 主要职责 |
|------|----------|
| [posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/posts.go) | 文章 HTTP 处理、查看、创建、更新、删除 |
| [pad.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/pad.go) | 编辑器页面、元数据编辑页面 |
| [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/database.go) | 所有 DB 操作：CreatePost/UpdateOwnedPost/ClaimPosts/DispersePosts/GetPosts |
| [collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/collections.go) | 集合可见性常量、权限校验、集合页面 |
| [postrender.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/postrender.go) | Markdown 渲染、内容增强、摘要生成 |
| [routes.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/routes.go) | 所有 URL 路由定义 |
| [handle.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/handle.go) | HTTP Handler 包装、错误处理、用户鉴权 |
| [static/js/postactions.js](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/static/js/postactions.js) | 前端：文章移动（发布/取消发布）AJAX 操作 |
| [templates/pad.tmpl](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/templates/pad.tmpl) | 编辑器模板含本地草稿保存 JS |
| [templates/include/posts.tmpl](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/templates/include/posts.tmpl) | 文章列表渲染、"移动到草稿" UI |
| [schema.sql](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/schema.sql) | 数据库表结构：posts 表字段定义 |
