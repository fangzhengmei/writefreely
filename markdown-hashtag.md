# WriteFreely 中 Hashtag 的处理流程分析

本文档详细分析 WriteFreely 项目中 hashtag（标签）在 Markdown 渲染管道、链接生成、标签索引和协作处理的完整处理流程。

---

## 一、整体架构概览

Hashtag 的处理涉及四个主要层面：

1. **Markdown 渲染层**：将 `#tag` 文本渲染为可点击的 HTML 链接
2. **数据提取层**：从文章内容中提取所有 hashtag
3. **数据库索引层**：通过正则表达式在 content 字段中进行基于 hashtag 的查询
4. **页面展示与 API 层**：标签页面、Reader 时间线、RSS/Atom Feed、ActivityPub 联邦协议输出

---

## 二、Markdown 渲染管道：#tag → HTML 链接

### 2.1 核心渲染函数

主要渲染逻辑位于 [postrender.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/postrender.go) 中的 `applyMarkdownSpecial` 函数。

#### 关键代码路径：

```go
// L146-L183
func applyMarkdownSpecial(data []byte, baseURL string, cfg *config.Config, skipNoFollow bool) string {
    // 1. 启用 blackfriday 的 HTML_HASHTAGS 扩展（当 baseURL 非空时）
    htmlFlags := 0 | blackfriday.HTML_USE_SMARTYPANTS | ...
    if baseURL != "" {
        htmlFlags |= blackfriday.HTML_HASHTAGS  // L159
    }

    // 2. 调用 blackfriday Markdown 解析器
    md := blackfriday.Markdown([]byte(data), blackfriday.HtmlRenderer(htmlFlags, "", ""), mdExtensions)

    // 3. 用正则替换特殊占位符为真正的 HTML 链接
    if baseURL != "" {
        tagPrefix := baseURL + "tag:"
        if cfg.App.Chorus {
            tagPrefix = "/read/t/"
        }
        // L170: 关键替换逻辑
        md = []byte(hashtagReg.ReplaceAll(md, []byte("<a href=\""+tagPrefix+"$1\" class=\"hashtag\"><span>#</span><span class=\"p-category\">$1</span></a>")))
    }
    // ... HTML 清理和后续处理
}
```

### 2.2 两段式渲染机制（关键理解点）

**Hashtag 的渲染分为两个阶段：**

**阶段 1：Blackfriday 标记**
- 当启用 `HTML_HASHTAGS` flag 时，`blackfriday`（WriteFreely 使用的 fork 版本 `saturday`）不会直接输出 `<a>` 链接
- 它输出一个特殊的占位符格式：`{{[[||tagname||]]}}`
- 这个占位符由正则 `hashtagReg` 定义（L42）：

```go
// L42
hashtagReg = regexp.MustCompile(`{{\[\[\|\|([^|]+)\|\|\]\]}}`)
```

**阶段 2：正则替换生成最终链接**
- Markdown 解析完成后，通过 `hashtagReg.ReplaceAll` 将占位符替换为真实 HTML
- 链接格式使用了微格式 `p-category`（符合 IndieWeb / ActivityPub 规范）

最终输出 HTML 结构示例：
```html
<a href="/blog/tag:golang" class="hashtag">
  <span>#</span>
  <span class="p-category">golang</span>
</a>
```

### 2.3 链接前缀策略

不同模式下链接前缀不同（[postrender.go L166-L169](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/postrender.go#L166-L169)）：

| 模式 | tagPrefix 示例 | 最终 URL |
|------|---------------|----------|
| 默认 Collection | `/blog/tag:` | `/blog/tag:golang` |
| Single User 模式 | `/tag:` | `/tag:golang` |
| Chorus (Reader) 模式 | `/read/t/` | `/read/t/golang` |

### 2.4 入口函数调用链

Markdown 渲染通过以下入口触发：

1. **普通文章渲染**：`Post.formatContent()` → [postrender.go L77-L92](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/postrender.go#L77-L92)
   ```go
   func (p *Post) formatContent(cfg *config.Config, c *Collection, isOwner bool, isPostPage bool) {
       p.HTMLContent = template.HTML(applyMarkdown([]byte(p.Content), baseURL, cfg))
   }
   ```

2. **API 实时渲染**：`handleRenderMarkdown()` → [postrender.go L342-L372](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/postrender.go#L342-L372)
   - 前端 ProseMirror 编辑器可调用 `/api/RenderMarkdown` 获取预览 HTML

3. **标题渲染**：`applyBasicMarkdown()` → [postrender.go L185-L217](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/postrender.go#L185-L217)
   - **注意**：标题渲染**不启用** `HTML_HASHTAGS`，标题中的 hashtag 不会被链接化

---

## 三、标签提取层：从内容中提取 Tags 列表

### 3.1 核心提取函数

标签提取使用外部库 `github.com/writeas/web-core/tags`，位于 [posts.go L1714-L1717](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/posts.go#L1714-L1717)：

```go
func (p *Post) extractData() {
    p.Tags = tags.Extract(p.Content)  // 提取所有 hashtag
    p.extractImages()                  // 顺带提取图片
}
```

`tags.Extract()` 会：
- 从原始 Markdown 文本（非渲染后 HTML）中提取所有 `#tagname`
- 自动去除前导 `#`
- 返回规范化后的字符串数组

### 3.2 Post 结构体中的 Tags 字段

位于 [posts.go L102-L127](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/posts.go#L102-L127)：

```go
type Post struct {
    // ...
    Tags    []string `json:"tags"`  // 存储在内存中，不直接持久化到独立表
    // ...
}
```

**关键点**：Tags 字段**不作为独立数据库列存储**，也没有独立的 `post_tags` 关联表。所有 tag 查询都是通过对 `content` 字段做正则匹配实现的。

### 3.3 HasTag 辅助方法

用于快速判断单篇文章是否包含某个 tag，位于 [posts.go L290-L296](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/posts.go#L290-L296)：

```go
func (p *Post) HasTag(tag string) bool {
    hasTag, _ := regexp.MatchString("#"+tag+`(?:[[:punct:]]|\s|\z)`, p.Content)
    return hasTag
}
```

该方法使用正则确保匹配完整词边界（标点、空白或行尾），避免 `#go` 错误匹配到 `#golang`。

### 3.4 extractData() 的调用时机

`extractData()` 在以下场景被调用（从数据库读取 Post 后）：

- `GetPostsTagged()` 中 → [database.go L1475](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/database.go#L1475)
- `GetPosts()` 中 → [database.go L1351](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/database.go#L1351)
- 其他获取文章的方法（共 7 处调用）

---

## 四、数据库索引层：基于正则的内容查询

### 4.1 核心查询方法

#### GetAllPostsTaggedIDs - 获取所有带 tag 的文章 ID

位于 [database.go L1371-L1414](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/database.go#L1371-L1414)：

```go
func (db *datastore) GetAllPostsTaggedIDs(c *Collection, tag string, includeFuture bool) ([]string, error) {
    if db.driverName == driverSQLite {
        // SQLite 使用 \b 词边界
        rows, err = db.Query(
            "SELECT id FROM posts WHERE collection_id = ? AND LOWER(content) regexp ? ...",
            collID, `.*#`+strings.ToLower(tag)+`\b.*`)
    } else {
        // MySQL 使用 [[:>:]] 或 \b 词边界（取决于 MySQL 版本）
        rows, err = db.Query(
            "SELECT id FROM posts WHERE collection_id = ? AND LOWER(content) RLIKE ? ...",
            collID, "#"+strings.ToLower(tag)+"[[:>:]]")
    }
}
```

#### GetPostsTagged - 分页获取带 tag 的文章

位于 [database.go L1420-L1487](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/database.go#L1420-L1487)：

```go
func (db *datastore) GetPostsTagged(cfg *config.Config, c *Collection, tag string, page int, includeFuture bool) (*[]PublicPost, error) {
    // 分页逻辑
    pagePosts := cf.PostsPerPage()
    start := page*pagePosts - pagePosts
    limitStr := fmt.Sprintf(" LIMIT %d, %d", start, pagePosts)

    // 与 GetAllPostsTaggedIDs 相同的正则查询模式，加 LIMIT
    // 查询后对每行调用 p.extractData() 填充 Tags 字段
}
```

### 4.2 数据库设计特点

**无独立 tags 表**：WriteFreely 没有传统的 `tags` / `post_tags` 表结构。全部通过对 `posts.content` 列做正则表达式（RLIKE / REGEXP）匹配来实现标签过滤。

**优点**：
- 实现简单，无需额外表和关联维护
- tag 可以实时出现在内容中，立即生效

**缺点**：
- 无法使用传统 B-Tree 索引，查询性能随数据量增长而下降
- 依赖数据库的正则匹配能力
- 无法做 tag 聚合统计（如"最热门 tag"）

### 4.3 跨数据库兼容处理

代码中处理了以下差异：

| 数据库 | 正则语法 | 词边界 |
|--------|---------|--------|
| SQLite | `REGEXP` | `\b` |
| MySQL < 8.0.4 | `RLIKE` | `[[:>:]]` (Spencer 实现) |
| MySQL >= 8.0.4 | `RLIKE` | `\b` (ICU 实现) |

---

## 五、页面展示与路由层

### 5.1 路由定义

位于 [routes.go L229-L247](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/routes.go#L229-L247)：

```go
// Collection 标签页面
r.HandleFunc("/tag:{tag}", handler.Web(handleViewCollectionTag, UserLevelReader))
r.HandleFunc("/tag:{tag}/page/{page:[0-9]+}", handler.Web(handleViewCollectionTag, UserLevelReader))
r.HandleFunc("/tag:{tag}/feed/", handler.Web(ViewFeed, UserLevelReader))

// Reader (Chorus模式) 标签时间线
r.HandleFunc("/t/{tag}", handler.Web(viewLocalTimeline, readPerm))
```

URL 模式说明：
- `/blog/tag:golang` - 单 Collection 下的标签页
- `/blog/tag:golang/page/2` - 标签分页
- `/blog/tag:golang/feed/` - 标签 RSS Feed
- `/t/golang` - Chorus/Reader 全局标签时间线

### 5.2 Collection 标签页处理

位于 [collections.go L1024-L1120](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/collections.go#L1024-L1120) 中的 `handleViewCollectionTag`：

处理流程：
1. 解析 URL 获取 `tag` 参数
2. 权限检查（Collection 私有/保护状态）
3. 调用 `GetAllPostsTaggedIDs` 获取总数用于分页计算
4. 调用 `GetPostsTagged` 获取当前页文章
5. 构造 `TagCollectionPage` 数据结构
6. 渲染 `collection-tags` 模板

#### TagCollectionPage 结构体

位于 [collections.go L654-L676](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/collections.go#L654-L676)：

```go
type TagCollectionPage struct {
    CollectionPage  // 嵌入通用 CollectionPage
    Tag string      // 当前标签名
}

// 分页 URL 生成
func (tcp TagCollectionPage) PrevPageURL(prefix string, n int, tl bool) string
func (tcp TagCollectionPage) NextPageURL(prefix string, n int, tl bool) string
```

### 5.3 模板渲染

标签页模板位于 [templates/collection-tags.tmpl](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/templates/collection-tags.tmpl)：

- 标题：`{{.Tag}} — {{.Collection.DisplayTitle}}`
- RSS Feed 链接：`{{.CanonicalURL}}tag:{{.Tag}}/feed/`
- 文章列表通过 `{{template "posts" .}}` 复用通用 posts 模板
- 标签 hashtag 本身是在文章内容 HTML 渲染阶段生成的可点击链接

### 5.4 Reader (Chorus) 全局标签时间线

位于 [read.go L185-L249](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/read.go#L185-L249) 中的 `showLocalTimeline`：

```go
} else if tag != "" {
    posts = []PublicPost{}
    for _, p := range *app.timeline.posts {
        if p.HasTag(tag) {  // 使用内存中的 HasTag() 方法过滤
            posts = append(posts, p)
        }
    }
}
```

**注意**：Reader 时间线使用内存缓存 + 应用层过滤（`HasTag`），而非数据库查询。适用于小数据量场景。

### 5.5 RSS Feed 标签过滤

位于 [feed.go L66-L88](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/feed.go#L66-L88)：

```go
tag := mux.Vars(req)["tag"]
if tag != "" {
    coll.Posts, _ = app.db.GetPostsTagged(app.cfg, c, tag, 1, false)
    collectionTitle = tag + " &mdash; " + collectionTitle
    siteURL += "tag:" + tag
}
```

---

## 六、ActivityPub 联邦协议层输出

### 6.1 Hashtag 作为 Tag 对象

当文章通过 ActivityPub 联邦到 Fediverse（如 Mastodon、Pleroma 等）时，hashtag 被转换为标准 ActivityStreams `Tag` 对象。

核心逻辑位于 [posts.go L1255-L1275](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/posts.go#L1255-L1275)：

```go
func (p *PublicPost) ActivityObject(app *App) *activitystreams.Object {
    // ...
    if len(p.Tags) == 0 {
        o.Tag = []activitystreams.Tag{}
    } else {
        // 根据模式生成 tagBaseURL
        if isSingleUser {
            tagBaseURL = p.Collection.CanonicalURL() + "tag:"
        } else if cfg.App.Chorus {
            tagBaseURL = fmt.Sprintf("%s/read/t/", p.Collection.hostName)
        } else {
            tagBaseURL = fmt.Sprintf("%s/%s/tag:", p.Collection.hostName, p.Collection.Alias)
        }
        // 每个 tag 生成一个 Tag 对象
        for _, t := range p.Tags {
            o.Tag = append(o.Tag, activitystreams.Tag{
                Type: activitystreams.TagHashtag,  // "Hashtag"
                HRef: tagBaseURL + t,              // 完整 URL
                Name: "#" + t,                     // 显示名带 #
            })
        }
    }
}
```

### 6.2 ActivityStreams 输出示例

```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Article",
  "tag": [
    {
      "type": "Hashtag",
      "href": "https://example.com/blog/tag:golang",
      "name": "#golang"
    }
  ]
}
```

---

## 七、完整调用链与协作时序

### 7.1 文章发布流程中的 hashtag 处理

```
用户提交 Post (newPost)
    ↓
db.CreatePost()  [database.go L675]
    ├─ 将原始 Content（含 #tag）直接写入 posts.content 列
    └─ 返回 Post 对象（此时 Tags 字段为空）
    ↓
p.extractData()  [posts.go L1714]
    └─ tags.Extract(p.Content) → 填充 p.Tags = ["golang", ...]
    ↓
（可选）federatePost()
    └─ p.ActivityObject() → 将 Tags 转为 ActivityStreams Tag 对象
```

### 7.2 文章展示流程中的 hashtag 处理

```
用户访问 /blog/tag:golang
    ↓
handleViewCollectionTag()  [collections.go L1024]
    ↓
db.GetPostsTagged()  [database.go L1420]
    ├─ SQL: SELECT ... FROM posts WHERE content RLIKE '#golang\b'
    ├─ 对每一行:
    │   ├─ p.extractData() → 填充 Tags
    │   ├─ p.augmentContent() → 添加签名等
    │   └─ p.formatContent() → applyMarkdown() → #tag → <a>链接
    └─ 返回 []PublicPost
    ↓
模板渲染 collection-tags.tmpl
    └─ 输出带可点击 hashtag 链接的 HTML
```

### 7.3 Markdown 渲染管道详细步骤

```
原始 Markdown 文本:  "Hello #golang world"
    ↓
[1] blackfriday.Markdown(HTML_HASHTAGS 启用)
    └─ 输出:  "<p>Hello {{[[||golang||]]}} world</p>"
    ↓
[2] hashtagReg.ReplaceAll()
    └─ 输出:  "<p>Hello <a href="/blog/tag:golang" class="hashtag">
                     <span>#</span><span class="p-category">golang</span>
                   </a> world</p>"
    ↓
[3] bluemonday Sanitize (HTML 白名单过滤)
    └─ 保留 <a href class>、<span class> 等安全元素
    ↓
[4] 后处理 (换行清理、Youtube autoplay 禁用等)
    ↓
最终 HTML
```

---

## 八、各模块文件参考

| 文件 | 职责 | 关键行号 |
|------|------|---------|
| [postrender.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/postrender.go) | Markdown 渲染、hashtag 占位符替换 | L42, L123-L183 |
| [posts.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/posts.go) | Post 结构、Tags 提取、HasTag、ActivityPub 输出 | L102-L127, L290-L296, L1223-L1309, L1714-L1717 |
| [database.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/database.go) | 基于正则的 tag 查询、CreatePost | L675-L772, L1371-L1487 |
| [collections.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/collections.go) | 标签页处理、TagCollectionPage | L654-L676, L1024-L1120 |
| [routes.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/routes.go) | 标签路由定义 | L229-L247 |
| [read.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/read.go) | Reader 标签时间线过滤 | L185-L249 |
| [feed.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/feed.go) | 标签 RSS Feed | L66-L88 |
| [templates/collection-tags.tmpl](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/templates/collection-tags.tmpl) | 标签页模板 | L1-L211 |
| [postrender_test.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-writefreely/postrender_test.go) | Markdown 渲染测试 | L15-L43 |

---

## 九、设计总结与要点

1. **两段式渲染**：hashtag 通过 blackfriday 占位符 + Go 正则替换实现，避免了 Markdown 解析器内部处理复杂的 URL 前缀逻辑
2. **无物化索引**：tags 不存储在独立表中，全部依赖 content 字段的正则匹配，简单但扩展性受限
3. **多协议支持**：同一套 Tags 数据同时支撑 HTML 微格式（`p-category`）、RSS Feed、ActivityPub 联邦协议
4. **多层过滤**：数据库层（RLIKE）→ 应用层（extractData）→ 展示层（HTML 链接），每层有独立职责
5. **URL 策略**：不同部署模式（Single User / Multi User / Chorus）使用不同的标签 URL 前缀，但通过统一的 `tagPrefix` 逻辑管理
