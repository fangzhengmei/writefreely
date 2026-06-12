# WriteFreely 文章编辑与发布流程

## 一、两条发布路径总览

WriteFreely 的内容生产有两条完全独立的路径，它们共享同一个 `posts` 数据表，但通过字段组合区分身份：

```
┌─────────────────────────────────────────────────────────────┐
│                        作者创建内容                           │
└──────────────────────────┬──────────────────────────────────┘
                           │
            ┌──────────────┴──────────────┐
            ▼                              ▼
   ┌─────────────────┐            ┌─────────────────┐
   │  路径 A：独立草稿  │            │  路径 B：集合文章  │
   │ (Anonymous Post) │            │(Collection Post)│
   └────────┬────────┘            └────────┬────────┘
            │                              │
  collection_id = NULL           collection_id = 有值
  slug = NULL                    slug = 有值(URL别名)
            │                              │
            └──────────────┬───────────────┘
                           ▼
                  可互相转换：
        ClaimPosts(草稿→集合) / DispersePosts(集合→草稿)
```

**状态的本质**：没有独立的 `status` 字段。一篇文章是"草稿"还是"已发布"，完全由 `collection_id` 是否为空决定。标题+正文同时为空则表示"已取消发布（Gone）"。

---

## 二、进入编辑：两种入口的权限约束节点

### 2.1 路径 A：独立草稿编辑入口

**用户访问路径**：`/me/posts`（我的草稿列表）→ 点击某篇草稿的"edit" → `/d/{id}/edit`

**权限约束发生在进入编辑器时**（不是查询列表时）：

```
GET /d/{id}/edit
    │
    ▼
handleViewPad() 接收请求
    │
    ├─ 从 URL 取出文章 ID（action 参数）
    │
    ├─ getRawPost(id) 直接查 posts 表
    │      │
    │      └─ 仅按 id 查询，不检查用户，不检查时间
    │
    └─ 权限校验（约束节点）：
           对比当前登录用户 ID ≟ 文章的 owner_id
           ├─ 相等 → 渲染编辑器
           └─ 不等 → 302 重定向到文章公开页（不让编辑）
```

**草稿列表查询本身不过滤权限**：`GetAnonymousPosts(u)` 的 SQL 条件是 `WHERE owner_id = u.ID`，天然只返回当前用户的草稿，所以列表页不需要额外权限检查。也**不做时间过滤**——作者可以看到自己所有草稿，包括定时发布在未来的。

### 2.2 路径 B：集合文章编辑入口

**用户访问路径**：进入某个博客（Collection）→ 在文章列表点击"edit" → `/{collAlias}/{slug}/edit`

**权限约束有两层，先查集合、再查文章**：

```
GET /{collAlias}/{slug}/edit
    │
    ▼
第1层：集合级权限约束（processCollectionPermissions）
    │
    ├─ 查集合是否存在
    │
    ├─ 集合是 Private（私有）？
    │   └─ 非作者 → 404（连集合存在都不暴露）
    │
    ├─ 集合是 Protected（密码保护）？
    │   ├─ 作者被禁言(silenced) → 404
    │   ├─ 已输入过密码(Cookie授权) → 放行
    │   └─ 未授权 → 渲染密码输入页，中断流程
    │
    └─ Public / Unlisted → 放行
    │
    ▼
第2层：文章级权限约束（handleViewPad 内）
    │
    ├─ getRawCollectionPost(slug, collAlias) 按 slug+集合查文章
    │
    └─ 对比当前登录用户 ID ≟ 文章 owner_id
           ├─ 相等 → 渲染编辑器
           └─ 不等 → 302 重定向
```

### 2.3 两条路径权限差异对照

| 对比维度 | 独立草稿编辑 | 集合文章编辑 |
|---------|------------|------------|
| 列表查询时权限 | SQL 自带 `owner_id = ?`，天然隔离 | 列表本身受集合权限约束（Private=404 等） |
| 进入编辑器前检查 | 仅检查 `owner_id` | 先检查集合可见性 + 密码授权，再检查 `owner_id` |
| 定时文章可见性 | 列表中全部可见，无 Scheduled 标记 | 非作者列表中被 SQL 过滤掉；作者可见并带 Scheduled badge |
| URL 特征 | `/d/{id}/edit`（单用户模式加 `/d` 前缀） | `/{coll}/{slug}/edit` |

---

## 三、发布与状态转换

### 3.1 发布的本质 = 关联集合

"发布一篇草稿到博客"在数据层面只做一件事：给 `posts` 表的那一行填上 `collection_id` 和 `slug`。

| 操作 | 数据变化 | 触发点 |
|------|---------|--------|
| 草稿 → 已发布 | `collection_id` 从 NULL → 有值；`slug` 从 NULL → 生成别名 | 前端 `postActions.move()` 调 `/api/collections/{alias}/collect` |
| 已发布 → 草稿 | `collection_id` 从有值 → NULL；`slug` 保留但不再被使用 | 前端选"Draft"调 `/api/posts/disperse` |
| 彻底取消发布 | `title` 和 `content` 同时被清空 | 作者在编辑器删除内容 |

### 3.2 slug 的生成逻辑

发布到集合时需要一个 URL 友好的别名（slug）：
1. 如果作者在元数据页指定了 slug → 用指定值
2. 否则从标题生成；没标题就从正文前几个字生成
3. 都不行就用文章 ID 本身
4. 如果 slug 和集合内已有文章冲突，自动加随机后缀重试

---

## 四、定时发布的可见性问题：为什么列表和详情不一致

### 4.1 现象描述

假设一篇集合文章设置了 `created = 未来某个时间`（即定时发布）：

- **在博客列表页**：非作者看不到这篇文章；作者能看到，且标题旁有 "Scheduled" 标记
- **在文章详情页**：只要有人知道完整 URL（`/{coll}/{slug}`），**任何人都可以直接访问并看到全文**，即使发布时间还没到

### 4.2 根因：两个查询函数的设计不对称

| 查询函数 | 用途 | 有无 `created <= NOW()` 过滤 |
|---------|------|----------------------------|
| `GetPosts(includeFuture)` | 列表查询（博客首页、归档、标签页） | 有。`includeFuture=false` 时加 `AND created <= NOW()` |
| `GetPost(id, collectionID)` | 单篇详情查询 | **没有**。直接按 slug/id 返回，不检查时间 |

列表查询的 `includeFuture` 参数在所有调用点都传入 `isCollOwner`（是否是集合所有者），所以作者能看到未来文章、其他人看不到。

但是单篇查询 `GetPost` 没有对应的参数，也没有时间判断。只要 slug 和 collection_id 匹配就返回。

### 4.3 设计意图推测

这是一种**"通过不公开实现保密"**的设计：
- slug 通常由标题自动生成，如果标题是私密的，外人难以猜到完整 URL
- 列表页是主要的发现入口，过滤掉定时文章就达到了"未发布不被发现"的目的
- 如果作者主动把 URL 分享给别人，则视为授权提前查看

代码本身承认了这个参数命名有误导性（注释 TODO：`change includeFuture to isOwner, since that's how it's used`）。

### 4.4 独立草稿列表的特殊情况

独立草稿列表（`/me/posts`）的模板中**没有 Scheduled badge 的判断**。即使草稿设置了未来发布时间，作者在草稿列表里也看不到任何"Scheduled"标记。原因是：草稿本身就等于"未发布"，Scheduled 标记只对集合内的文章有意义。

---

## 五、页面渲染：四层权限层叠模型

当一篇文章被渲染成 HTML 时，权限检查不是一个单点判断，而是从外到内共四层检查逐层生效。任何一层未通过都会中断渲染。

```
                     HTTP 请求到达
                          │
          ┌───────────────┴───────────────┐
          ▼                               ▼
   独立文章 URL                    集合文章 URL
   (/{id})                         (/{coll}/{slug})
          │                               │
          │                      ┌────────▼─────────┐
          │                      │ 第 1 层：集合级    │
          │                      │ processCollection  │
          │                      │  Permissions      │
          │                      │  Private → 404    │
          │                      │  Protected→密码页  │
          │                      └────────┬──────────┘
          │                               │
          │                      ┌────────▼─────────┐
          │                      │ 第 2 层：文章级    │
          │                      │ viewCollectionPost│
          │                      │  Private二次校验   │
          │                      │  Gone→410         │
          │                      │  silenced→404     │
          │                      └────────┬──────────┘
          └───────────────┬───────────────┘
                          ▼
                 ┌─────────────────┐
                 │ 第 3 层：内容级    │
                 │ handleViewPost   │
                 │ (独立文章走这里)  │
                 │ protectDraft检查  │
                 │ Gone→410         │
                 │ silenced→404     │
                 └────────┬────────┘
                          ▼
                 ┌─────────────────┐
                 │ 第 4 层：模板级    │
                 │ 渲染 + 展示差异   │
                 │ IsOwner 控制按钮  │
                 │ IsScheduled徽章   │
                 │ paid/more 截断    │
                 └─────────────────┘
```

### 第 1 层：集合级权限（仅集合文章有）

- **Private 集合**：非作者一律 404，甚至不告诉你这个博客存在
- **Protected 集合**：非作者需要输入密码，授权存在 Cookie 中，每个集合独立授权
- **silenced（被禁言）的作者**：其 Protected 集合对外也表现为 404

### 第 2 层：文章级权限

- 集合文章在 `viewCollectionPost` 中**又做了一次 Private/Protected 检查**（双重保险）
- `title == "" && content == ""`（已取消发布）→ 410 Gone
- 作者被 silenced 且访问者不是作者/管理员 → 404

### 第 3 层：内容级权限（独立文章的 protectDraft）

独立文章的 URL `/{id}` 本身是公开的。但是如果这篇文章的 `collection_id` 指向了一个 Private 或 Protected 集合，那么非作者访问这篇独立 URL 也会被 404。这是为了防止有人通过"先发布到私有集合、再记住独立 ID"的方式绕过集合权限。

### 第 4 层：模板级展示差异

即使通过了前面所有检查，最终渲染的 HTML 仍会根据访问者身份有差异：

| 展示元素 | 作者 | 非作者 |
|---------|------|--------|
| edit / delete / pin 操作按钮 | ✅ 显示 | ❌ 隐藏 |
| Scheduled badge（集合内定时文章） | ✅ 显示 | ✅ 也显示（如果能进来） |
| `<!--more-->` 截断 | 列表页截断，详情页全文 | 同左 |
| `<!--paid-->` 付费内容截断 | 全文可见，标有"订阅内容开始"提示 | 截断，显示 Coil 会员提示 |
| 集合签名追加 | 受 `<!--nosig-->` 控制 | 同左 |
| 邮件订阅表单 | 作者看到"你已订阅 / 退订" | 看到订阅输入框 |

---

## 六、未发布内容（Gone）的渲染差异

"已取消发布"状态 = `title` 和 `content` 同时为空字符串。此时渲染行为取决于访问的是独立文章还是集合文章：

| 请求方式 | 独立文章（/{id}） | 集合文章（/{coll}/{slug}） |
|---------|------------------|------------------------|
| 浏览器 HTML | 410 Gone 页面 | 410 Gone |
| JSON API | HTTP 200，返回 `{"error": "Post was unpublished."}` | 410 Gone |
| 纯文本 /md | HTTP 200，返回 "Post was unpublished." | 410 Gone |

集合文章的处理更"干脆"——不管什么格式统一 410。独立文章则对机器可读格式做了妥协，返回 200 带错误信息，可能是为了兼容早期 API 客户端。

---

## 七、关键文件索引

| 文件 | 核心作用 |
|------|---------|
| [posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/posts.go) | 文章 HTTP 层：查看、创建、更新、Claim、Disperse；`IsScheduled()` 判定；`getRawPost` / `getRawCollectionPost` |
| [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/database.go) | DB 层：`CreatePost` / `UpdateOwnedPost` / `ClaimPosts` / `DispersePosts` / `GetPosts`（有 includeFuture）/ `GetPost`（无时间过滤）/ `GetAnonymousPosts` |
| [collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/collections.go) | 集合可见性常量、`processCollectionPermissions`（集合级权限）、`isAuthorizedForCollection`（密码授权） |
| [pad.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/pad.go) | `handleViewPad`：编辑器加载，两种编辑入口的 owner 校验都在这里 |
| [account.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/account.go) | `viewArticles`（独立草稿列表页）、`viewMyPostsAPI`（草稿列表 API） |
| [postrender.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/postrender.go) | Markdown 渲染管道、`formatContent`（paid/more 截断、列表 vs 详情差异）、`augmentContent`（签名追加） |
| [routes.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/routes.go) | 路由定义：草稿 `/d/` 前缀区分、集合文章与独立文章 URL 映射 |
| [templates/user/articles.tmpl](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/templates/user/articles.tmpl) | 独立草稿列表模板（无 Scheduled 判断） |
| [templates/include/posts.tmpl](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/templates/include/posts.tmpl) | 集合文章列表模板（有 Scheduled badge） |
| [templates/collection-post.tmpl](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/templates/collection-post.tmpl) | 集合文章详情页（有 Scheduled badge） |
| [templates/collection-archive.tmpl](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/templates/collection-archive.tmpl) | 集合归档页（`[Scheduled]` 文本标记） |
| [templates/pad.tmpl](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/templates/pad.tmpl) | 编辑器模板（本地草稿 localStorage、发布按钮逻辑） |
| [static/js/postactions.js](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/static/js/postactions.js) | 前端发布/取消发布：`move()` / `multiMove()` 调 collect / disperse 接口 |
