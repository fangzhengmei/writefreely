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
