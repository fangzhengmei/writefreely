# Collection 与站点路由

本文档基于 WriteFreely 源码，深入分析 Collection（集合/博客）与站点路由的实现机制，包括公开路径、用户集合、文章列表和别名冲突处理。

## 目录

1. [Collection 数据结构](#collection-数据结构)
2. [路由系统概述](#路由系统概述)
3. [公开路径](#公开路径)
4. [用户集合管理](#用户集合管理)
5. [文章列表](#文章列表)
6. [别名冲突处理](#别名冲突处理)

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
    // ... 其他字段
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

- **blog**：博客格式，倒序排列，显示日期
- **novel**：小说格式，正序排列
- **notebook**：笔记本格式

---

## 路由系统概述

路由定义位于 [routes.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/routes.go) 的 `InitRoutes` 函数中。

### 单用户 vs 多用户模式

WriteFreely 支持两种运行模式，路由结构差异很大：

**单用户模式** (`SingleUser = true`)：
- 整个站点就是一个集合
- 集合直接在根路径 `/` 下
- 草稿编辑路径前缀为 `/d`

**多用户模式** (`SingleUser = false`)：
- 每个用户可以有多个集合
- 集合通过别名访问：`/{collection}/`
- 支持前缀字符：`/{prefix:[@~$!\-+]}{collection}`

全局变量 `isSingleUser` 在应用启动时设置，见 [app.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/app.go#L451)。

### 用户级别控制

路由通过 `UserLevelFunc` 控制访问权限，定义在 [handle.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/handle.go#L31-L64)：

| 级别 | 说明 |
|------|------|
| UserLevelNone | 忽略用户状态 |
| UserLevelOptional | 用户可选，有用户时获取用户信息 |
| UserLevelNoneRequired | 必须是未登录用户 |
| UserLevelUser | 必须是已登录用户 |
| UserLevelReader | 读者级别，站点私有则需登录 |

---

## 公开路径

### 集合首页与分页

集合相关路由通过 `RouteCollections` 函数注册，见 [routes.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/routes.go#L222-L240)。

| 路径 | 处理函数 | 说明 |
|------|----------|------|
| `/` | `handleViewCollection` | 集合首页 |
| `/page/{page}` | `handleViewCollection` | 分页 |
| `/archive/` | `handleViewCollection` | 归档视图 |
| `/archive/page/{page}` | `handleViewCollection` | 归档分页 |

集合视图处理流程在 [collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/collections.go#L856-L1005) 的 `handleViewCollection` 函数中：

1. 调用 `processCollectionRequest` 解析请求参数
2. 调用 `checkUserForCollection` 获取当前用户
3. 调用 `processCollectionPermissions` 检查权限
4. 调用 `newDisplayCollection` 创建显示用集合对象
5. 调用 `GetPosts` 获取文章列表
6. 渲染模板

### 标签与语言

| 路径 | 处理函数 | 说明 |
|------|----------|------|
| `/tag:{tag}` | `handleViewCollectionTag` | 标签过滤 |
| `/tag:{tag}/page/{page}` | `handleViewCollectionTag` | 标签分页 |
| `/tag:{tag}/feed/` | `ViewFeed` | 标签 Feed |
| `/lang:{lang}` | `handleViewCollectionLang` | 语言过滤 |

### Feed 与 Sitemap

| 路径 | 处理函数 | 说明 |
|------|----------|------|
| `/feed/` | `ViewFeed` | RSS/Atom Feed |
| `/sitemap.xml` | `handleViewSitemap` | 站点地图 |

### 文章页面

| 路径 | 处理函数 | 说明 |
|------|----------|------|
| `/{slug}` | `CollectionPostOrStatic` | 文章或静态文件 |
| `/{slug}/` | `handleCollectionPostRedirect` | 带斜杠重定向 |
| `/{slug}/edit` | `handleViewPad` | 编辑文章（需登录） |
| `/{slug}/edit/meta` | `handleViewMeta` | 编辑元数据 |

**静态文件判断**：`CollectionPostOrStatic`（见 [handle.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/handle.go#L469-L483)）会先检查 URL 是否包含 `.`，如果是且不是 raw 请求，则作为静态文件处理。

### 前缀字符

多用户模式下，集合别名前可以带有前缀字符，路由模式为 `/{prefix:[@~$!\-+]?}{collection}`，见 [routes.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/routes.go#L213)。

支持的前缀字符：`@`、`~`、`$`、`!`、`-`、`+`

前缀存储在 `collectionReq.prefix` 中，用于构造 URL。

---

## 用户集合管理

### 管理后台路由

用户登录后可以管理自己的集合，路由位于 `/me/c/` 下，见 [routes.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/routes.go#L98-L102)：

| 路径 | 处理函数 | 说明 |
|------|----------|------|
| `/me/c/` | `viewCollections` | 我的博客列表 |
| `/me/c/{collection}` | `viewEditCollection` | 编辑博客设置 |
| `/me/c/{collection}/stats` | `viewStats` | 统计数据 |
| `/me/c/{collection}/subscribers` | `handleViewSubscribers` | 订阅者管理 |

### 用户文章列表

用户的所有文章（包括匿名文章和集合文章）在 `/me/posts/` 查看，见 [account.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/account.go#L774-L818) 的 `viewArticles` 函数：

```go
func viewArticles(app *App, u *User, w http.ResponseWriter, r *http.Request) error {
    // 获取匿名文章
    p, err := app.db.GetAnonymousPosts(u, 1)
    // 获取可发布的集合
    c, err := app.db.GetPublishableCollections(u, app.cfg.App.Host)
    // ...
}
```

### API 接口

| 路径 | 方法 | 说明 |
|------|------|------|
| `/api/collections` | POST | 创建集合 |
| `/api/collections/{alias}` | GET | 获取集合信息 |
| `/api/collections/{alias}` | POST/DELETE | 更新/删除集合 |
| `/api/collections/{alias}/posts` | GET | 获取集合文章 |
| `/api/me/collections` | GET | 获取我的集合 |

创建集合的入口函数是 `newCollection`，见 [collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/collections.go#L427)。

---

## 文章列表

### 获取文章列表

文章列表通过 `GetPosts` 函数获取，定义在 [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/database.go#L1303-L1369)。

函数签名：
```go
func (db *datastore) GetPosts(cfg *config.Config, c *Collection, page int, 
    includeFuture, forceRecentFirst, includePinned bool, contentType PostType) 
    (*[]PublicPost, error)
```

参数说明：
- `page`：页码，为 0 时返回最多 1000 篇
- `includeFuture`：是否包含未来发布的文章（所有者可见）
- `forceRecentFirst`：强制按最新排序
- `includePinned`：是否包含置顶文章
- `contentType`：内容类型（普通/归档）

### 分页逻辑

每页文章数量由集合格式决定（当前均为 `postsPerPage`），见 [collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/collections.go#L183-L188)。

总页数计算：
```go
coll.TotalPages = int(math.Ceil(float64(coll.TotalPosts) / float64(ppp)))
```

如果请求页码超过总页数，会重定向到最后一页，见 [collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/collections.go#L911-L916)。

### 排序方式

- **blog 格式**：按创建时间倒序（DESC）
- **novel 格式**：按创建时间正序（ASC）

排序逻辑在 `GetPosts` 函数中通过 `cf.Ascending()` 判断。

### 文章数量统计

文章总数通过 `GetPostsCount` 函数获取，定义在 [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/database.go#L1279)。

---

## 别名冲突处理

### 别名规范

1. **小写化**：别名统一使用小写。`processCollectionRequest` 函数会检查并将大写别名重定向到小写版本，见 [collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/collections.go#L708-L717)。

```go
func processCollectionRequest(cr *collectionReq, vars map[string]string, 
    w http.ResponseWriter, r *http.Request) error {
    cr.prefix = vars["prefix"]
    cr.alias = vars["collection"]
    if cr.alias != strings.ToLower(cr.alias) {
        return impart.HTTPError{
            http.StatusMovedPermanently, 
            fmt.Sprintf("/%s/", strings.ToLower(cr.alias)),
        }
    }
    return nil
}
```

2. **slug 生成**：别名由标题通过 `getSlug` 函数生成，见 [posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/posts.go#L1339-L1366)。

### 与帖子 ID 的冲突

帖子 ID 长度固定为 10 个字符（`minIDLen = maxIDLen = 10`），见 [posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/posts.go#L45-L46)。

**创建时检查**：创建集合时会检查别名是否与现有帖子 ID 冲突，见 [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/database.go#L312-L315)：

```go
func (db *datastore) CreateCollection(...) (*Collection, error) {
    if db.PostIDExists(alias) {
        return nil, impart.HTTPError{http.StatusConflict, "Invalid collection name."}
    }
    // ...
}
```

**访问时处理**：访问集合时，如果未找到集合且别名长度在帖子 ID 范围内，会检查是否是帖子，如果是则重定向到帖子页面，见 [collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/collections.go#L746-L751)：

```go
if len(cr.alias) >= minIDLen && len(cr.alias) <= maxIDLen {
    if app.db.PostIDExists(cr.alias) {
        return nil, impart.HTTPError{http.StatusMovedPermanently, "/" + cr.alias}
    }
}
```

### 重复键错误

数据库层面通过唯一约束保证别名唯一。插入时如果遇到重复键错误，返回 "Collection already exists."，见 [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/database.go#L320-L322)：

```go
if db.isDuplicateKeyErr(err) {
    return nil, impart.HTTPError{http.StatusConflict, "Collection already exists."}
}
```

### 集合重定向

当集合别名更改时，旧别名会保留在 `collectionredirects` 表中，实现无缝重定向。

**重定向查询**：`GetCollectionRedirect` 函数，见 [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/database.go#L2425-L2432)：

```go
func (db *datastore) GetCollectionRedirect(alias string) (new string) {
    row := db.QueryRow("SELECT new_alias FROM collectionredirects WHERE prev_alias = ?", alias)
    // ...
}
```

**使用场景**：
- 集合访问时：`processCollectionPermissions` 中检查并重定向
- 文章访问时：`viewCollectionPost` 中检查并重定向

### 用户名与集合别名

用户注册时也会进行类似的冲突检查。用户名同样需要避免与帖子 ID 冲突，见 [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/32-writefreely/database.go#L214)。

### 文章 slug 冲突

同一集合内的文章 slug 也是唯一的。创建/更新文章时会处理 slug 冲突，通过在 slug 后追加数字后缀解决。

---

## 总结

WriteFreely 的 Collection 路由系统设计精巧，主要特点：

1. **灵活的模式切换**：单用户/多用户模式通过配置切换，路由结构差异明显
2. **多级权限控制**：通过 UserLevel 机制实现细粒度的访问控制
3. **完善的冲突处理**：从创建时的检查到访问时的重定向，全方位处理别名冲突
4. **优雅的 URL 设计**：支持前缀字符、标签、语言等多种 URL 形式
5. **良好的用户体验**：URL 规范化（小写化）、旧别名重定向等细节处理
