# RSS Feed 与订阅输出分析

## 1. Feed 路由与入口点

### 1.1 Collection 级 Feed

| 路由 | 处理函数 | 说明 |
|------|----------|------|
| `/{collection}/feed/` | `ViewFeed` | Collection 主 RSS 订阅源 |
| `/{collection}/tag:{tag}/feed/` | `ViewFeed` | 按标签过滤的 RSS 订阅源 |

**代码位置:** [routes.go L231-L233](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/routes.go#L231-L233)

```go
r.HandleFunc("/tag:{tag}/feed/", handler.Web(ViewFeed, UserLevelReader))
r.HandleFunc("/feed/", handler.AllReader(ViewFeed))
```

### 1.2 实例级 Reader Feed

| 路由 | 处理函数 | 说明 |
|------|----------|------|
| `/read/feed/` | `viewLocalTimelineFeed` | 全站公开文章聚合 RSS |

**代码位置:** [routes.go L245-L246](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/routes.go#L245-L246)

---

## 2. 核心处理函数

### 2.1 Collection 级: `ViewFeed`

**文件:** [feed.go](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/feed.go)

**处理流程:**

1. **Collection 定位:**
   - 单用户模式: `GetCollectionByID(1)` [feed.go L31](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/feed.go#L31-L31)
   - 多用户模式: `GetCollection(alias)` [feed.go L33](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/feed.go#L33-L33)

2. **权限检查链:**
   ```go
   // 1. 检查用户是否被静默
   silenced, err := app.db.IsUserSilenced(c.OwnerID)
   if silenced {
       return ErrCollectionNotFound  // 返回 404
   }
   
   // 2. 检查 Collection 隐私状态
   if c.IsPrivate() || c.IsProtected() {
       return ErrCollectionNotFound  // 返回 404
   }
   ```
   **代码位置:** [feed.go L39-L51](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/feed.go#L39-L51)

3. **数据获取:**
   ```go
   if tag != "" {
       // 按标签获取文章
       coll.Posts, _ = app.db.GetPostsTagged(app.cfg, c, tag, 1, false)
   } else {
       // 获取 Collection 文章
       // 参数: cfg, collection, page=1, includeFuture=false, 
       //       forceRecentFirst=true, includePinned=false, contentType=""
       coll.Posts, _ = app.db.GetPosts(app.cfg, c, 1, false, true, false, "")
   }
   ```
   **代码位置:** [feed.go L66-L71](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/feed.go#L66-L71)

4. **Feed 元数据构建:**
   ```go
   feed := &feeds.Feed{
       Title:       collectionTitle,
       Link:        &feeds.Link{Href: siteURL},
       Description: coll.Description,
       Author:      &feeds.Author{author, ""},
       Created:     time.Now(),
   }
   ```
   **代码位置:** [feed.go L90-L96](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/feed.go#L90-L96)

5. **条目构建:**
   ```go
   feed.Items = append(feed.Items, &feeds.Item{
       Id:          fmt.Sprintf("%s%s", basePermalinkUrl, p.Slug.String),
       Title:       title,
       Link:        &feeds.Link{Href: permalink},
       Description: "<![CDATA[" + stripmd.Strip(p.Content) + "]]>",
       Content:     string(p.HTMLContent),
       Author:      &feeds.Author{author, ""},
       Created:     p.Created,
       Updated:     p.Updated,
   })
   ```
   **代码位置:** [feed.go L106-L115](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/feed.go#L106-L115)

### 2.2 实例级: `viewLocalTimelineFeed`

**文件:** [read.go L292-L344](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/read.go#L292-L344)

**处理流程:**

1. **功能开关检查:**
   ```go
   if !app.cfg.App.LocalTimeline {
       return impart.HTTPError{http.StatusNotFound, "Page doesn't exist."}
   }
   ```

2. **缓存更新:**
   ```go
   updateTimelineCache(app.timeline, false)
   ```

3. **Feed 元数据:**
   ```go
   feed := &Feed{
       Title:       app.cfg.App.SiteName + " Reader",
       Link:        &Link{Href: app.cfg.App.Host},
       Description: "Read the latest posts from " + app.cfg.App.SiteName + ".",
       Created:     time.Now(),
   }
   ```

4. **条目数量限制:**
   ```go
   const tlFeedLimit = 100  // 最多 100 篇
   for _, p := range *app.timeline.posts {
       if c == tlFeedLimit {
           break
       }
       // ... 构建条目
   }
   ```
   **代码位置:** [read.go L312-L314](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/read.go#L312-L314)

---

## 3. 数据库查询与条目选择逻辑

### 3.1 `GetPosts` - Collection 文章查询

**文件:** [database.go L1303-L1369](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/database.go#L1303-L1369)

**函数签名:**
```go
func (db *datastore) GetPosts(
    cfg *config.Config, 
    c *Collection, 
    page int, 
    includeFuture bool, 
    forceRecentFirst bool, 
    includePinned bool, 
    contentType PostType
) (*[]PublicPost, error)
```

**SQL 查询条件:**
```sql
SELECT postCols FROM posts 
WHERE collection_id = ? 
  AND pinned_position IS NULL  -- 排除置顶文章 (includePinned=false)
  AND created <= NOW()         -- 排除未来发布的文章 (includeFuture=false)
ORDER BY created DESC          -- 降序排列 (forceRecentFirst=true)
LIMIT 0, {pagePosts}           -- 分页
```

**分页逻辑:**
- `page = 0`: 无分页限制，最多返回 1000 篇
  ```go
  if page == 0 {
      start = 0
      pagePosts = 1000
  }
  ```
  **代码位置:** [database.go L1314-L1316](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/database.go#L1314-L1316)

- `page > 0`: 按 Collection 设置的 `PostsPerPage()` 分页

**排序逻辑:**
```go
order := "DESC"
if cf.Ascending() && !forceRecentFirst {
    order = "ASC"
}
```
**代码位置:** [database.go L1307-L1310](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/database.go#L1307-L1310)

### 3.2 `GetPostsTagged` - 标签过滤查询

**文件:** [database.go L1420-L1477](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/database.go#L1420-L1477)

**标签匹配正则:**
- SQLite: `.*#tag\b.*`
- MySQL (旧版): `#tag[[:>:]]`
- MySQL (8.0.4+): `#tag\b`

**代码位置:** [database.go L1447-L1458](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/database.go#L1447-L1458)

### 3.3 `FetchPublicPosts` - 实例级 Reader 查询

**文件:** [read.go L71-L131](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/read.go#L71-L131)

**SQL 查询:**
```sql
SELECT p.id, c.id, alias, c.title, p.slug, p.title, p.content, ...
FROM collections c
LEFT JOIN posts p ON p.collection_id = c.id
LEFT JOIN users u ON u.id = p.owner_id
WHERE c.privacy = 1                     -- 仅公开 Collection (CollPublic)
  AND (p.created <= NOW() 
       AND pinned_position IS NULL)     -- 非置顶、已发布
  AND u.status = 0                      -- 用户状态正常 (未封禁)
ORDER BY p.created DESC
LIMIT 250                               -- 最多缓存 250 篇
```
**代码位置:** [read.go L78-L84](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/read.go#L78-L84)

**每作者文章限制:**
```go
const tlMaxAuthorPosts = 5  // 每个作者最多 5 篇
if c.Alias != "" && ap[c.Alias] == tlMaxAuthorPosts {
    continue  // 跳过该作者的后续文章
}
```
**代码位置:** [read.go L108-L111](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/read.go#L108-L111)

**缓存机制:**
```go
func initLocalTimeline(app *App) {
    app.timeline = &localTimeline{
        postsPerPage: tlPostsPerPage,
        m:            memo.New(app.FetchPublicPosts, tlCacheDur),
    }
}
const tlCacheDur = 10 * time.Minute  // 缓存有效期 10 分钟
```
**代码位置:** [read.go L63-L68](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/read.go#L63-L68)

---

## 4. 内容裁剪与输出格式

### 4.1 标题处理

**`PlainDisplayTitle()`** - [posts.go L225-L230](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/posts.go#L225-L230)

```go
func (p *Post) PlainDisplayTitle() string {
    if t := stripmd.Strip(p.DisplayTitle()); t != "" {
        return t
    }
    return p.ID  // 如果标题为空，返回文章 ID
}
```

**`DisplayTitle()`** - [posts.go L214-L220](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/posts.go#L214-L220)
- 有显式标题: 返回 `p.Title.String`
- 无显式标题: 调用 `friendlyPostTitle()` 从内容生成
  - 取第一段内容（`\n\n` 之前）
  - 超过 80 字符截断并添加 `...`

### 4.2 Description 字段 (纯文本摘要)

**Collection Feed:** [feed.go L110](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/feed.go#L110-L110)
```go
Description: "<![CDATA[" + stripmd.Strip(p.Content) + "]]>"
```

**Reader Feed:** [read.go L327](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/read.go#L327-L327)
```go
Description: "<![CDATA[" + stripmd.Strip(p.Content) + "]]>"
```

**处理逻辑:**
1. 使用 `github.com/writeas/go-strip-markdown/v2` 库的 `Strip()` 函数
2. 移除所有 Markdown 格式（标题、链接、图片、代码块等）
3. 用 CDATA 包装，避免 XML 解析器处理特殊字符

### 4.3 Content 字段 (完整 HTML 内容)

**Collection Feed:** [feed.go L111](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/feed.go#L111-L111)
```go
Content: string(p.HTMLContent)
```
- 使用数据库查询时预渲染的 HTML 内容
- 通过 `formatContent()` 处理生成

**Reader Feed:** [read.go L328](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/read.go#L328-L328)
```go
Content: applyMarkdown([]byte(p.Content), "", app.cfg)
```
- 实时将 Markdown 转换为 HTML

### 4.4 内容裁剪标签机制

#### `<!--more-->` - 摘要分割标签

**代码位置:** [postrender.go L89-L91](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/postrender.go#L89-L91)

```go
if exc := strings.Index(string(p.Content), "<!--more-->"); exc > -1 {
    p.HTMLExcerpt = template.HTML(applyMarkdown([]byte(p.Content[:exc]), baseURL, cfg))
}
```

**注意:** 在 RSS Feed 输出中，`Description` 使用纯文本，`Content` 使用完整 HTML，`<!--more-->` 标签对 RSS 输出**没有直接影响**，因为 RSS 输出的是完整内容。

#### `<!--paid-->` - 付费内容标签

**代码位置:** [postrender.go L47-L75](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/postrender.go#L47-L75)

```go
func (p *Post) handlePremiumContent(c *Collection, isOwner, postPage bool, cfg *config.Config) {
    if c.Monetization != "" {
        spl := strings.Index(p.Content, shortCodePaid)
        p.IsPaid = spl > -1
        if !postPage {
            // 列表页/Feed: 只显示付费标签之前的内容
            if spl > -1 {
                p.Content = p.Content[:spl+len(shortCodePaid)]
                p.HTMLExcerpt = template.HTML(applyMarkdown([]byte(p.Content[:spl]), baseURL, cfg))
            }
        }
    }
}
```

**对 RSS 的影响:**
- 如果 Collection 启用了 Web Monetization 且文章包含 `<!--paid-->` 标签
- Feed 输出中**只包含付费标签之前**的内容
- 付费内容之后的部分会被截断

#### `<!--nosig-->` - 禁用签名标签

**代码位置:** [postrender.go L103-L106](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/postrender.go#L103-L106)

```go
if strings.Index(p.Content, shortCodeNoSig) > -1 {
    return  // 不添加 Collection 签名
}
```

### 4.5 内容增强 (`augmentContent`)

**代码位置:** [postrender.go L98-L111](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/postrender.go#L98-L111)

```go
func (p *Post) augmentContent(c *Collection) {
    if p.PinnedPosition.Valid {
        return  // 置顶文章不增强
    }
    if strings.Index(p.Content, shortCodeNoSig) > -1 {
        return  // 有 nosig 标签不增强
    }
    // 添加 Collection 签名
    if c.Signature != "" {
        p.Content += "\n\n" + c.Signature
    }
}
```

**调用时机:** 在 `GetPosts` 查询结果处理中调用 [database.go L1352](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/database.go#L1352-L1352)

### 4.6 Markdown 渲染 (`applyMarkdown`)

**代码位置:** [postrender.go L123-L183](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/postrender.go#L123-L183)

**启用的 Markdown 扩展:**
- 表格 (`EXTENSION_TABLES`)
- 围栏代码块 (`EXTENSION_FENCED_CODE`)
- 自动链接 (`EXTENSION_AUTOLINK`)
- 删除线 (`EXTENSION_STRIKETHROUGH`)
- 空格标题 (`EXTENSION_SPACE_HEADERS`)
- 自动标题 ID (`EXTENSION_AUTO_HEADER_IDS`)

**HTML 净化:**
- 使用 `bluemonday.UGCPolicy()` 作为基础
- 允许 `iframe`, `video`, `audio` 等媒体元素
- 允许自定义 `style`, `class`, `id` 属性
- 支持的 URL 协议: `http`, `https`, `mailto`, `xmpp`, `gopher`, `gophers`, `gemini`, `spartan`

---

## 5. 隐私与权限控制

### 5.1 Collection 可见性级别

**代码位置:** [collections.go L155-L160](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/collections.go#L155-L160)

```go
const CollUnlisted collVisibility = 0
const (
    CollPublic    collVisibility = 1 << iota  // 1 - 公开
    CollPrivate                               // 2 - 私有
    CollProtected                             // 4 - 密码保护
)
```

**检查方法:** [collections.go L218-L228](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/collections.go#L218-L228)
```go
func (c *Collection) IsPrivate() bool   { return c.Visibility&CollPrivate != 0 }
func (c *Collection) IsProtected() bool { return c.Visibility&CollProtected != 0 }
func (c *Collection) IsPublic() bool    { return c.Visibility&CollPublic != 0 }
```

**对 RSS 的影响:**
- `CollPrivate` 或 `CollProtected`: `ViewFeed` 返回 `ErrCollectionNotFound` (404)
- `CollPublic` 或 `CollUnlisted`: 正常提供 RSS

### 5.2 用户静默状态

**检查函数:** [database.go L360](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/database.go#L360-L360)
```go
func (db *datastore) IsUserSilenced(id int64) (bool, error)
```

**对 RSS 的影响:**
- 静默用户的 Collection: `ViewFeed` 返回 `ErrCollectionNotFound` (404)
- **代码位置:** [feed.go L44-L46](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/feed.go#L44-L46)

### 5.3 Reader Feed 过滤条件

**只包含:**
- 公开 Collection (`c.privacy = 1`)
- 用户状态正常 (`u.status = 0`)
- 非置顶文章 (`pinned_position IS NULL`)
- 已发布文章 (`created <= NOW()`)

**代码位置:** [read.go L80-L83](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/read.go#L80-L83)

---

## 6. 输出字段对照表

| RSS 字段 | Collection Feed | Reader Feed | 处理逻辑 |
|---------|-----------------|-------------|----------|
| `title` | `p.PlainDisplayTitle()` | `p.PlainDisplayTitle()` | 移除 Markdown 的标题 |
| `link` | `baseUrl + p.Slug.String` | `p.CanonicalURL()` | 文章永久链接 |
| `description` | `stripmd.Strip(p.Content)` + CDATA | `stripmd.Strip(p.Content)` + CDATA | 纯文本摘要 |
| `content:encoded` | `string(p.HTMLContent)` | `applyMarkdown(p.Content)` | 完整 HTML 内容 |
| `author` | Collection 所有者用户名 | Collection 标题 / "Anonymous" | 作者信息 |
| `pubDate` | `p.Created` | `p.Created` | 创建时间 |
| `updated` | `p.Updated` | `p.Updated` | 更新时间 |
| `guid` | `basePermalinkUrl + p.Slug.String` | `host + /read/a/ + p.ID` | 唯一标识符 |

---

## 7. 关键常量汇总

| 常量 | 值 | 说明 | 位置 |
|------|----|------|------|
| `tlFeedLimit` | 100 | Reader Feed 最大条目数 | [read.go L33](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/read.go#L33-L33) |
| `tlMaxAuthorPosts` | 5 | Reader 中每作者最大文章数 | [read.go L35](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/read.go#L35-L35) |
| `tlMaxPostCache` | 250 | Reader 最大缓存文章数 | [read.go L37](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/read.go#L37-L37) |
| `tlCacheDur` | 10 分钟 | Reader 缓存有效期 | [read.go L38](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/read.go#L38-L38) |
| `shortCodeMore` | `<!--more-->` | 摘要分割标签 | [posts.go L58](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/posts.go#L58-L58) |
| `shortCodePaid` | `<!--paid-->` | 付费内容标签 | [posts.go L59](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/posts.go#L59-L59) |
| `shortCodeNoSig` | `<!--nosig-->` | 禁用签名标签 | [posts.go L60](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/posts.go#L60-L60) |

---

## 8. 用户、Collection、实例三种层级的选条目方式辨析

### 8.1 概念澄清：用户 vs Collection

在 WriteFreely 中，**"用户页面"本质上就是 Collection 页面**，两者通过 `alias = 用户名` 的约定建立关联：

- **用户 (User)**：系统账户实体，存储在 `users` 表，拥有 `username`、`password`、`status` 等账户属性
- **Collection (博客)**：内容容器实体，存储在 `collections` 表，拥有 `alias`、`title`、`owner_id` 等内容属性
- **关联关系**：每个用户创建账户时，系统自动创建一个**主 Collection**，其 `alias` 等于用户的 `username`
- **访问方式**：在多用户实例中访问 `/{username}/` 时，系统用 `username` 作为 `alias` 查找并渲染该 Collection

**数据库关系:**
```
users 表                     collections 表
┌──────────────┐ 1:N      ┌──────────────────────┐
│ id (PK)      │─────────▶│ id (PK)              │
│ username     │          │ alias (= username)    │ ← 用户页面访问的核心
│ ...          │          │ owner_id (FK→users.id)│
└──────────────┘          └──────────────────────┘
```

**全局开关定义:** [app.go L65, L451](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/app.go#L65-L65)
```go
isSingleUser bool  // 全局变量，在 Serve() 中根据配置初始化
isSingleUser = app.cfg.App.SingleUser
```

---

### 8.2 公开用户页面（多用户模式下的 Collection 页面）

**页面路由:**
- `/{prefix}{collection}/` → `handleViewCollection`
- `/{prefix}{collection}/page/{page}` → `handleViewCollection`

**代码位置:** [collections.go L857-L1005](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/collections.go#L857-L1005)

**处理流程:**

1. **请求解析与 Collection 定位:**
   ```go
   err := processCollectionRequest(cr, vars, w, r)
   c, err := processCollectionPermissions(app, cr, u, w, r)
   ```

2. **数据获取调用:**
   ```go
   // 关键参数解析:
   // page: 页码（从 URL 获取，默认 1）
   // includeFuture: cr.isCollOwner（只有所有者能看未来文章）
   // forceRecentFirst: false（可按 Collection 配置的升降序排列）
   // includePinned: false（排除置顶文章，置顶文章单独获取）
   coll.Posts, _ = app.db.GetPosts(app.cfg, c, page, cr.isCollOwner, false, false, "")
   ```
   **代码位置:** [collections.go L919](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/collections.go#L919-L919)

3. **置顶文章单独处理:**
   ```go
   displayPage.PinnedPosts, _ = app.db.GetPinnedPosts(coll.CollectionObj, isOwner)
   ```
   置顶文章不参与分页，始终在页面顶部展示。

**排序逻辑差异:**
- Collection 页面 (`forceRecentFirst=false`): 尊重 Collection 的 `Format.Ascending()` 设置
  ```go
  order := "DESC"
  if cf.Ascending() && !forceRecentFirst {
      order = "ASC"  // 如果 Collection 配置为升序且不强制最新在前，则升序
  }
  ```
  **代码位置:** [database.go L1307-L1310](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/database.go#L1307-L1310)

- RSS Feed (`forceRecentFirst=true`): **强制降序**，不考虑 Collection 设置
  ```go
  // feed.go 调用:
  coll.Posts, _ = app.db.GetPosts(app.cfg, c, 1, false, true, false, "")
  //                                   forceRecentFirst ↑ = true
  ```
  **代码位置:** [feed.go L70](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/feed.go#L70-L70)

**`includeFuture` 参数对比:**
| 场景 | 调用者 | `includeFuture` 值 | 效果 |
|------|--------|-------------------|------|
| Collection 页面（访客） | `handleViewCollection` | `false` | 不显示未来文章 |
| Collection 页面（所有者） | `handleViewCollection` | `true` (`cr.isCollOwner`) | 显示未来定时文章 |
| Collection RSS Feed | `ViewFeed` | `false` | 不显示未来文章 |
| Reader 页面/Feed | `viewLocalTimeline` | `false` | 不显示未来文章 |

---

### 8.3 单用户顶层 Feed

当配置为 `SingleUser=true` 时，整个实例只服务一个用户，路由和处理方式发生重大变化。

**首页路由:** [app.go L226-L230](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/app.go#L226-L230)
```go
func handleViewHome(app *App, w http.ResponseWriter, r *http.Request) error {
    if app.cfg.App.SingleUser {
        // 首页直接渲染 Collection 索引页
        return handleViewCollection(app, w, r)
    }
    // ... 多用户模式的其他逻辑
}
```

**Collection 级路由注册差异:** [routes.go L208-L214](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/routes.go#L208-L214)
```go
if apper.App().cfg.App.SingleUser {
    // 单用户模式: Collection 路由挂载在根路径
    RouteCollections(handler, write.PathPrefix("/").Subrouter())
} else {
    // 多用户模式: Collection 路由需要前缀，如 /username/
    write.HandleFunc("/{prefix:[@~$!\\-+]}{collection}", ...)
    RouteCollections(handler, write.PathPrefix("/{prefix:[@~$!\\-+]?}{collection}").Subrouter())
}
```

**Feed 路由含义变化:**
- **单用户模式**: `/feed/` → 顶层根路径的 Feed，即默认 Collection 的 Feed
- **多用户模式**: `/{collection}/feed/` → 指定 Collection 的 Feed

**Collection 定位差异:** [feed.go L30-L34](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/feed.go#L30-L34)
```go
if app.cfg.App.SingleUser {
    // 单用户模式: 固定取 ID=1 的 Collection
    c, err = app.db.GetCollectionByID(1)
} else {
    // 多用户模式: 从 URL 路径解析 alias
    c, err = app.db.GetCollection(alias)
}
```

**Canonical URL 差异:** [collections.go L275-L280](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/collections.go#L275-L280)
```go
func (c *Collection) RedirectingCanonicalURL(isRedir bool) string {
    if isSingleUser {
        return c.hostName + "/"  // 单用户: https://example.com/
    }
    return fmt.Sprintf("%s/%s/", c.hostName, c.Alias)  // 多用户: https://example.com/username/
}
```

**Feed 中文章链接的差异:**
| 模式 | 文章链接格式 | 示例 |
|------|-------------|------|
| 单用户 | `hostName + "/" + slug` | `https://blog.example.com/my-post` |
| 多用户 | `hostName + "/" + alias + "/" + slug` | `https://write.example.com/alice/my-post` |

---

### 8.4 实例级 Reader Feed

实例级 Reader Feed 是从全站聚合公开文章，与 Collection 级 Feed 的选条目逻辑有本质不同。

**路由:** `/read/feed/` → `viewLocalTimelineFeed`

**代码位置:** [read.go L292-L344](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/read.go#L292-L344)

#### 底层查询：`FetchPublicPosts`

**SQL 中的 JOIN 结构:** [read.go L78-L84](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/read.go#L78-L84)
```sql
FROM collections c
LEFT JOIN posts p ON p.collection_id = c.id   -- 只关联有 Collection 的文章
LEFT JOIN users u ON u.id = p.owner_id         -- 关联用户表检查状态
```

**关键特征：**
1. **从 collections 表出发**：意味着 `collection_id = NULL` 的**匿名文章不会出现在 Reader Feed 中**
2. **必须是公开 Collection**：`c.privacy = 1` (`CollPublic`)
3. **用户状态正常**：`u.status = 0`

#### Reader 页面的作者过滤

**代码位置:** [read.go L185-L224](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/read.go#L185-L224)

```go
func showLocalTimeline(app *App, ..., page int, author, tag string) error {
    // ...
    if author != "" {
        posts = []PublicPost{}
        for _, p := range *app.timeline.posts {
            if author == "anonymous" {
                // 筛选 "匿名" → p.Collection == nil 的文章
                // 注意: 由于 SQL 从 collections 出发，实际不会有 p.Collection == nil 的文章
                if p.Collection == nil {
                    posts = append(posts, p)
                }
            } else if p.Collection != nil && p.Collection.Alias == author {
                // 按 Collection 别名（即用户名）筛选
                posts = append(posts, p)
            }
        }
    }
}
```

**路由中的 author 参数来源:** [routes.go L249](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/routes.go#L249-L249)
```go
r.HandleFunc("/{author}", handler.Web(viewLocalTimeline, readPerm))  // /read/{author}
```

#### 按作者过滤的三种形式

| URL 模式 | 筛选条件 | 说明 |
|---------|---------|------|
| `/read/` | 无过滤 | 所有公开文章聚合 |
| `/read/{author}` | `p.Collection.Alias == author` | 只显示指定 Collection（用户名）的文章 |
| `/read/anonymous` | `p.Collection == nil` | 理论上筛选匿名文章，但实际结果为空（因 SQL JOIN 排除） |
| `/read/t/{tag}` | 按标签内容筛选 | 文章内容包含指定 #标签 |

---

## 9. 匿名文章 vs 具名 Collection 文章

### 9.1 数据库层面的区别

**posts 表字段:** [sqlite.sql L125-L126](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/sqlite.sql#L125-L126)
```sql
owner_id INTEGER DEFAULT NULL,       -- NULL = 未认领/无主
collection_id INTEGER DEFAULT NULL,  -- NULL = 匿名/草稿
```

| 文章类型 | `collection_id` | `owner_id` | `slug` |
|---------|-----------------|-----------|--------|
| 匿名/草稿文章 | `NULL` | 可为 NULL 或有值 | 通常 NULL，使用 `id` 作为 URL 标识 |
| 具名 Collection 文章 | 有值（对应 collections.id） | 有值（对应用户） | 有语义化的 slug 字符串 |

### 9.2 链接 (URL) 差异

#### 文章 Canonical URL 生成逻辑

**代码位置:** [posts.go L1207-L1212](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/posts.go#L1207-L1212)
```go
func (p *PublicPost) CanonicalURL(hostName string) string {
    if p.Collection == nil || p.Collection.Alias == "" {
        // 匿名文章: 没有 Collection，使用 ID 访问
        return hostName + "/" + p.ID + ".md"
    }
    // Collection 文章: 使用语义化 slug
    return p.Collection.CanonicalURL() + p.Slug.String
}
```

**对比表:**

| 方面 | 匿名文章 | 具名 Collection 文章 |
|------|---------|---------------------|
| **Go 结构判断** | `p.Collection == nil` | `p.Collection != nil` |
| **URL 标识** | 10 字符 `ID` | 语义化 `slug` |
| **链接格式 (单用户)** | `/{id}.md` 或 `/d/{id}` (草稿) | `/{slug}` |
| **链接格式 (多用户)** | `/{id}.md` | `/{alias}/{slug}` |
| **示例** | `https://example.com/abc123xyz.md` | `https://example.com/alice/my-first-post` |

**草稿 URL 前缀 (单用户模式):** [routes.go L197-L199](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/routes.go#L197-L199)
```go
if apper.App().cfg.App.SingleUser {
    draftEditPrefix = "/d"  // 单用户模式下草稿带 /d 前缀
}
```

#### Reader 模板中的链接差异

**代码位置:** [read.tmpl L102, L108](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/templates/read.tmpl#L102-L102)
```go
// 日期链接:
{{if .Collection}}
    <a href="{{.Collection.CanonicalURL}}{{.Slug.String}}">...</a>
{{else}}
    <a href="{{.CanonicalURL .Host}}.md">...</a>
{{end}}

// "阅读更多" 链接:
{{if .Collection}}
    <a href="{{.Collection.CanonicalURL}}{{.Slug.String}}">Read more...</a>
{{else}}
    <a href="{{.CanonicalURL .Host}}.md">Read more...</a>
{{end}}
```

### 9.3 作者信息差异

#### Reader Feed 中的作者字段

**代码位置:** [read.go L318-L322](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/read.go#L318-L322)
```go
if p.Collection != nil {
    author = p.Collection.Title    // 用 Collection 标题作为作者
} else {
    author = "Anonymous"           // 匿名显示为 "Anonymous"
}
```

#### Collection Feed 中的作者字段

**代码位置:** [feed.go L73-L76, L112](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/feed.go#L73-L76)
```go
author := ""
if coll.Owner != nil {
    author = coll.Owner.Username   // Collection Feed: 直接显示所有者用户名
}
```

**对比表:**

| 方面 | 匿名文章 | 具名 Collection 文章 |
|------|---------|---------------------|
| **Reader Feed author** | `"Anonymous"` | `p.Collection.Title` (Collection 显示标题) |
| **Collection Feed author** | N/A (不在此 Feed 出现) | `coll.Owner.Username` (所有者用户名) |
| **Reader 页面来源显示** | `<em>Anonymous</em>` | `from <a href="...">Collection Title</a>` |
| **GUID** | `host + /read/a/ + p.ID` | `basePermalinkUrl + p.Slug.String` |

**Reader 模板中的来源信息:** [read.tmpl L105](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/templates/read.tmpl#L105-L105)
```html
<p class="source">
    {{if .Collection}}
        from <a href="{{.Collection.CanonicalURL}}">{{.Collection.DisplayTitle}}</a>
    {{else}}
        <em>Anonymous</em>
    {{end}}
</p>
```

### 9.4 出现在哪些 Feed 中的对比

| Feed 类型 | 匿名文章 (`collection_id=NULL`) | 具名 Collection 文章 |
|----------|--------------------------------|---------------------|
| **Collection RSS Feed** (`/{alias}/feed/`) | ❌ 不出现 (WHERE `collection_id = ?`) | ✅ 出现 |
| **Collection 标签 Feed** | ❌ 不出现 | ✅ 出现 (若含对应标签) |
| **实例 Reader Feed** (`/read/feed/`) | ❌ 不出现 (SQL 从 `collections` JOIN) | ✅ 仅公开 Collection 的文章 |
| **Reader 作者筛选 `/read/anonymous`** | ❌ 理论出现但实际为空 | ❌ 不出现 |
| **Reader 作者筛选 `/read/{alias}`** | ❌ 不出现 | ✅ 出现 |
| **单篇文章匿名访问** (`/{id}.md`) | ✅ 可直接访问 | ❌ 用 slug 访问 |

### 9.5 总结关系图

```
WriteFreely 文章发布体系
│
├─ 用户 (User)
│   └── 对应 Account，拥有 1 个或多个 Collection
│
├─ Collection (博客/用户页面)
│   ├── 类型: Unlisted(0) / Public(1) / Private(2) / Protected(4)
│   ├── 文章来源: owner_id = User.id
│   ├── 文章特征: collection_id ≠ NULL, 有语义化 slug
│   ├── 可出现在: Collection页面 + Collection RSS + 公开时可出现在 Reader
│   └── 作者显示: Collection 标题 / 用户名
│
├─ 匿名文章 (Anonymous Post / Draft)
│   ├── 特征: collection_id = NULL, 用 10 位 ID 标识
│   ├── 可被认领 (Claim) 为具名文章
│   ├── 可直接通过 /{id}.md 或 /{id} 访问
│   ├── 不可出现在任何 RSS Feed
│   └── 作者显示: "Anonymous" (若出现在 Reader，实际不会)
│
└─ Reader (实例聚合阅读器)
    ├── 数据源: 所有 c.privacy=1 的公开 Collection
    ├── 排序: 创建时间 DESC
    ├── 限制: 每作者最多 5 篇，缓存最多 250 篇，Feed 最多 100 篇
    ├── 按作者筛选: /read/{alias} (按 Collection 别名)
    └── 按标签筛选: /read/t/{tag}
```

---

## 10. 用户、Collection alias、作者信息的概念辨析

### 10.1 三个概念的区分依据

在 WriteFreely 代码体系中，用户（User）、Collection alias、作者信息（Author）是三个不同层次的概念，分别对应不同的代码实体和数据库字段。

#### 用户 (User)

**代码实体:** `User` 结构体，对应 `users` 表

**核心字段:**
- `ID int64` - 用户唯一标识
- `Username string` - 登录用户名
- `Password []byte` - 密码哈希
- `Email zero.String` - 邮箱
- `Status int64` - 用户状态（0=正常，非0=封禁/静默）

**数据库查询函数:**
```go
func (db *datastore) GetUserByID(userID int64) (*User, error)
func (db *datastore) GetUserByName(username string) (*User, error)
```

**在 RSS 中的使用:**
- 仅当 Collection 的 `PublicOwner = true` 时，才会查询用户表获取 `Username`
- 查询后赋值给 `coll.Owner` 字段

#### Collection Alias

**代码实体:** `Collection.Alias` 字段，对应 `collections.alias` 字段

**定义:**
```go
type Collection struct {
    ID      int64
    Alias   string  // URL 友好的别名，用于路径路由
    Title   string  // 显示标题
    OwnerID int64   // 外键，关联 users.id
    // ...
}
```

**用途:**
1. **URL 路由解析**: 从 `/alice/feed/` 中提取 `alice` 作为 alias 查找 Collection
2. **子域名解析**: 从 `alice.writeas.com` 中提取 `alice` 作为 alias
3. **Canonical URL 生成**: 生成 `https://example.com/alice/my-post` 形式的链接
4. **Reader 作者筛选**: `/read/alice` 按 alias 过滤文章

#### 作者信息 (Author)

**在 RSS Feed 中的表现:**
- **Collection RSS Feed**: `coll.Owner.Username`（用户的登录名）
- **Reader RSS Feed**: `p.Collection.Title`（Collection 的显示标题）或 `"Anonymous"`

**三者关系图:**
```
users 表
├─ id (PK)
├─ username
└─ status
    │
    │ 1:N 关系
    │
    ▼
collections 表
├─ id (PK)
├─ alias       ← URL 路径/子域名解析用
├─ title       ← Reader Feed 作者显示用
└─ owner_id (FK → users.id)
    │
    │ 1:N 关系
    │
    ▼
posts 表
├─ id (PK)
├─ collection_id (FK → collections.id)  ← NULL 表示匿名文章
├─ owner_id (FK → users.id)
└─ slug         ← 语义化 URL 片段（匿名文章为 NULL）
```

### 10.2 Collection Feed 中作者信息为空的具体情形

**核心代码:** [feed.go L56-L76](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/feed.go#L56-L76)

```go
// 56行: 仅当 PublicOwner 为 true 时才查询用户信息
if c.PublicOwner {
    u, err := app.db.GetUserByID(coll.OwnerID)
    if err != nil {
        log.Error("Error getting user for collection: %v", err)
    } else {
        coll.Owner = u  // 查询成功才赋值
    }
}
// ...
// 74行: 仅当 Owner 不为 nil 时才设置 author
author := ""
if coll.Owner != nil {
    author = coll.Owner.Username
}
```

**作者信息为空（`author = ""`）的三种情形:**

| 情形 | 原因 | 代码位置 |
|------|------|----------|
| **情形 1: `PublicOwner = false`（默认）** | 创建 Collection 时默认 `PublicOwner = false`，此时不会调用 `GetUserByID`，`coll.Owner` 始终为 `nil` | [database.go L331](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/database.go#L331-L331) |
| **情形 2: 用户查询失败** | 即使 `PublicOwner = true`，如果 `GetUserByID` 返回错误（如用户已删除、数据库错误），则 `coll.Owner` 仍为 `nil` | [feed.go L58-L60](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/feed.go#L58-L60) |
| **情形 3: 用户查询成功但 Username 为空** | 理论上不会发生，但如果用户表中 `username` 字段为空，也会导致 author 为空 | `users.username` 字段 |

**数据库层面的 `PublicOwner` 字段:**

注意：`GetCollectionBy` 的 SELECT 语句中**并没有包含 `public_owner` 字段**：
```sql
SELECT id, alias, title, description, style_sheet, script, post_signature, 
       format, owner_id, privacy, view_count 
FROM collections WHERE ...
```
**代码位置:** [database.go L861](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/database.go#L861-L861)

这意味着：
- `c.PublicOwner` 始终为 Go 零值 `false`
- **实际上 Collection Feed 的作者信息**总是空的**
- 这是一个已知的代码缺陷（FIXME 注释也提到了 "change Collection to reflect database values"）

---

### 10.3 Reader 页面匿名分支无法实际访问的根本原因

**代码路径:** [read.go L206-L209](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/read.go#L206-L209)

```go
if author == "anonymous" {
    if p.Collection == nil {
        posts = append(posts, p)  // 理论上筛选匿名文章
    }
}
```

**为什么这个分支永远不会匹配到文章？**

#### 原因 1: 数据源层的排除

**`FetchPublicPosts` 的 SQL 结构:** [read.go L78-L84](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/read.go#L78-L84)

```sql
FROM collections c              -- 从 collections 表开始
LEFT JOIN posts p ON p.collection_id = c.id  -- 只 JOIN 有 collection_id 的文章
```

这个 LEFT JOIN 的方向决定了：
- 只有 `posts.collection_id = collections.id` 的文章才会被选中
- `posts.collection_id = NULL` 的**匿名文章根本不会出现在结果集中**
- 因此 `app.timeline.posts` 中的所有文章都有 `p.Collection != nil`

#### 原因 2: 数据结构层面的保证

在 `FetchPublicPosts` 的结果处理中，每篇文章都显式设置了 Collection：
```go
// read.go L93-L96
c := &Collection{
    ID:    collID.Int64,
    Alias: collAlias.String,
    Title: title.String,
}
p.Collection = c  // 确保每篇文章都有 Collection
```
**代码位置:** [read.go L93-L96](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/read.go#L93-L96)

#### 原因 3: 每作者文章限制的影响

```go
// read.go L108-L111
if c.Alias != "" && ap[c.Alias] == tlMaxAuthorPosts {
    continue
}
```
这个判断也假设了 `c.Alias` 存在，进一步确认所有处理的文章都有 Collection。

**结论:** `/read/anonymous` 路径存在于代码中，但**由于数据源层的 SQL JOIN 结构，永远不会返回任何文章**。这是一个"死代码"分支。

---

### 10.4 Feed Alias 从子域名或路径解析的完整逻辑

#### 核心解析函数: `collectionAliasFromReq`

**代码位置:** [collections.go L1337-L1346](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/collections.go#L1337-L1346)

```go
func collectionAliasFromReq(r *http.Request) string {
    vars := mux.Vars(r)
    alias := vars["subdomain"]   // 优先尝试子域名
    isSubdomain := alias != ""
    if !isSubdomain {
        // 子域名为空时，回退到路径参数
        alias = vars["collection"]
    }
    return alias
}
```

#### 解析优先级

1. **第一优先级: 子域名 (`subdomain`)**
   - 来源: `mux.Vars(r)["subdomain"]`
   - 例如: `alice.example.com` → `alice`

2. **第二优先级: URL 路径 (`collection`)**
   - 来源: `mux.Vars(r)["collection"]`
   - 例如: `example.com/alice/feed/` → `alice`

#### 路由层面的参数配置

**多用户模式下的 Collection 路由前缀:** [routes.go L211-L213](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/routes.go#L211-L213)

```go
write.HandleFunc("/{prefix:[@~$!\\-+]}{collection}", handler.Web(...))
write.HandleFunc("/{collection}/", handler.Web(...))
RouteCollections(handler, write.PathPrefix("/{prefix:[@~$!\\-+]?}{collection}").Subrouter())
```

**路径参数匹配规则:**
- `{prefix:[@~$!\\-+]}`: 可选前缀字符（`@`, `~`, `$`, `!`, `-`, `+`）
- `{collection}`: 匹配 Collection alias

**支持的 URL 形式:**
| URL 形式 | 提取的 alias | 说明 |
|---------|-------------|------|
| `/alice/feed/` | `alice` | 无前缀 |
| `/@alice/feed/` | `alice` | `@` 前缀 |
| `/~alice/feed/` | `alice` | `~` 前缀 |
| `alice.example.com/feed/` | `alice` | 子域名方式 |

#### 子域名方式的数据库查询

当 alias 通过子域名方式获取后，最终会调用 `GetCollection(alias)`：
```go
// feed.go L33
c, err = app.db.GetCollection(alias)
// → SELECT ... FROM collections WHERE alias = ?
```

**代码位置:** [feed.go L33](file:///d:/fz/0601-1/solo-dogfeeding/code/34-writefreely/feed.go#L33-L33)

#### 单用户模式的特殊处理

在单用户模式下，**alias 解析被完全跳过**：
```go
// feed.go L30-L31
if app.cfg.App.SingleUser {
    c, err = app.db.GetCollectionByID(1)  // 固定取 ID=1 的 Collection
} else {
    c, err = app.db.GetCollection(alias)  // 用解析的 alias 查询
}
```

#### 查询调用链汇总

```
HTTP 请求
    │
    ▼
┌──────────────────────────┐
│ collectionAliasFromReq() │
│ 1. vars["subdomain"]?    │  → 子域名方式（如 alice.example.com）
│ 2. vars["collection"]?   │  → 路径方式（如 /alice/feed/）
└──────────────────────────┘
    │
    ▼
┌──────────────────────┐
│ feed.go ViewFeed()   │
│ 单用户? ──┐          │
│   │       │          │
│   ▼       ▼          │
│ GetCollectionByID(1) │  → 固定 ID=1
│ GetCollection(alias) │  → WHERE alias = ?
└──────────────────────┘
    │
    ▼
SELECT * FROM collections WHERE ...
```

### 10.5 字段使用位置对照表

| 数据来源 | 字段 | 使用位置 | 说明 |
|---------|------|---------|------|
| **users 表** | `username` | Collection Feed `author` 字段 | 仅当 `PublicOwner=true` 时使用 |
| **users 表** | `status` | `IsUserSilenced()` 检查 | 非 0 则返回 404 |
| **collections 表** | `alias` | URL 解析、Canonical URL、Reader 筛选 | 核心标识字段 |
| **collections 表** | `title` | Reader Feed `author` 字段 | 显示为作者名 |
| **collections 表** | `owner_id` | `PublicOwner` 检查时查询用户 | 关联外键 |
| **collections 表** | `privacy` | 权限检查 | `1=公开` 才可出现在 RSS |
| **collectionattributes 表** | `monetization_pointer` | 付费内容处理 | 通过 `GetCollectionAttribute` 获取 |
| **posts 表** | `collection_id` | 区分匿名/具名文章 | `NULL=匿名，有值=具名` |
| **posts 表** | `slug` | 文章 URL 生成 | 匿名文章为 NULL，用 ID |


