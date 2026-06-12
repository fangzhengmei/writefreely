# WriteFreely 文章编辑与发布流程

## 一、用户视角下的两条内容路径

当你在 WriteFreely 中写作时，实际上是在两条独立的轨道上切换。它们共享同一个数据库，但权限逻辑和用户体验完全不同。

```
                     你写了一篇文章
                         │
           ┌─────────────┴─────────────┐
           ▼                           ▼
    保存为"草稿"                   发布到"博客"
（独立存在，不归属任何博客）      （归属到某个 Collection）
           │                           │
   URL: /{文章ID}               URL: /{博客别名}/{文章别名}
   无 slug，无 collection_id     有 slug，有 collection_id
           │                           │
   只有你自己能在"我的草稿"        博客的读者能在博客首页
   列表里看到它                   列表里看到它（取决于博客可见性）
```

**状态没有专门的字段**：一篇文章是"草稿"还是"已发布到博客"，只看 `collection_id` 字段有没有值。标题和正文同时为空 = 已彻底删除。

---

## 二、三部分权限隔离的设计逻辑

整个系统的权限分布在三个环节上，每个环节承担不同的隔离职责。理解这三个环节，就能理解为什么独立草稿和集合文章的处理方式不一样。

### 2.1 第一环：列表查询的权限隔离

这是最外层的筛选。用户看到什么文章列表，在 SQL 查询阶段就已经决定了。

#### 独立草稿列表（/me/posts）

**用户行为**：你登录后点击"我的草稿"，想看自己所有未发布的文章。

**设计结果**：

```
GET /me/posts
    │
    ▼
调用 GetAnonymousPosts(u, 1)
    │
    ▼
执行 SQL：
SELECT ... FROM posts
WHERE owner_id = ?          -- 关键：天然只返回当前用户的
  AND collection_id IS NULL -- 只返回草稿
ORDER BY created DESC
```

**为什么这样设计？**
- `owner_id = ?` 这个条件意味着：**数据库层面就确保了用户只能看到自己的草稿**
- 不需要额外的权限检查逻辑，SQL 查询本身就是权限隔离
- 也没有时间过滤——你当然可以看到自己设置了未来发布时间的草稿

**结果**：你在"我的草稿"列表里看到的每一篇，100% 都是你自己写的。

#### 集合文章列表（/{博客别名}/）

**用户行为**：访客访问某个博客的首页，想看这个博客发布的所有文章。

**设计结果**：

```
GET /{博客别名}/
    │
    ▼
第1步：processCollectionPermissions() 先检查博客本身的权限
    ├─ Private（私有博客）+ 非作者 → 直接 404（不告诉你这个博客存在）
    ├─ Protected（密码博客）+ 未输入密码 → 渲染密码输入页，中断流程
    └─ 可见性正常 → 继续
    │
    ▼
第2步：调用 GetPosts(..., includeFuture=isCollOwner, ...)
    │
    ▼
执行 SQL：
SELECT ... FROM posts
WHERE collection_id = ?          -- 只返回这个博客的文章
  AND pinned_position IS NULL
  AND created <= NOW()           -- 非作者时，过滤掉未来发布的
ORDER BY created DESC
```

**为什么这样设计？**
- `collection_id = ?` 只按博客筛选，**不过滤 owner_id**——因为博客的文章本来就是要给别人看的
- 但是博客本身可能是 Private 或 Protected 的，所以必须先过集合可见性这一关
- `includeFuture=isCollOwner`：作者能看到未到时间的定时文章（显示"Scheduled"标记），访客看不到

**结果**：访客看到的列表 = 这个博客的、已到发布时间的、公开可见的文章。

---

### 2.2 第二环：进入编辑器的权限检查

列表展示只是"看"，点"edit"进入编辑器就是"改"了。这一环的设计差异最大，也是之前理解有误的地方。

#### 独立草稿编辑器（/d/{文章ID}/edit）

**用户行为**：你在"我的草稿"列表里点了某篇的"edit"按钮。

**设计结果**：

```
GET /d/{id}/edit
    │
    ▼
handleViewPad() 接收请求
    │
    ├─ 从 URL 取出 action = 文章ID
    │
    ├─ getRawPost(id) 按 ID 查文章
    │      │
    │      └─ SQL: SELECT ... FROM posts WHERE id = ?
    │         （不检查 owner_id，不检查时间，不检查集合）
    │
    ├─ 检查是否已取消发布（Gone）
    │
    └─ 直接渲染编辑器
       （注意：这里没有 OwnerID 对比！）
```

**为什么不检查 OwnerID？**
- 因为你只能从"我的草稿"列表进入这个编辑器，而那个列表的 SQL 已经保证了只显示你的文章
- 如果有人构造 URL 尝试编辑别人的草稿（比如猜 ID），理论上是可能的，但实际上：
  - 草稿 ID 是随机字符串（不是自增数字），难猜
  - 即使猜中，getRawPost 会把文章内容返回给你，这是一个潜在漏洞

**结果**：进入独立草稿编辑器的门槛很低，主要靠上一环（列表查询）来筛选。

#### 集合文章编辑器（/{博客别名}/{文章别名}/edit）

**用户行为**：你在自己博客的文章列表里点了某篇的"edit"按钮。

**设计结果**：

```
GET /{collAlias}/{slug}/edit
    │
    ▼
第1层：processCollectionPermissions() 检查博客权限
    ├─ Private + 非作者 → 404
    ├─ Protected + 未授权 → 密码输入页
    └─ 正常 → 继续
    │
    ▼
第2层：handleViewPad() 内部检查
    │
    ├─ getRawCollectionPost(slug, collAlias)
    │      │
    │      └─ SQL: SELECT ... FROM posts
    │                WHERE slug = ? AND collection_id = (SELECT id FROM collections WHERE alias = ?)
    │
    └─ ★ 必须做 OwnerID 对比 ★
           if appData.Post.OwnerID != appData.User.ID {
               302 重定向到文章公开页（不让编辑）
           }
```

**为什么必须检查 OwnerID？**
- 因为博客的文章列表是公开的，**任何人都能看到文章 URL 和 edit 按钮的结构**
- 如果不做 OwnerID 检查，任何登录用户都可以构造 `/{blog}/{slug}/edit` 来编辑别人的文章
- 这是硬要求，和独立草稿的情况完全不同

**结果**：即使你通过了博客权限检查，也必须是文章的作者才能进入编辑器。

---

### 2.3 第三环：集合文章编辑的双重权限

集合文章的编辑有两道门，这是它和独立草稿最大的区别。

#### 第一道门：集合可见性

在做任何文章操作之前，先检查这个博客本身能不能被访问：

| 博客类型 | 非作者访问 | 结果 |
|---------|-----------|------|
| Public（公开） | 是 | 正常访问 |
| Unlisted（未列出） | 是 | 正常访问（只是不对外推荐） |
| Protected（密码保护） | 是 | 需要输入密码，存在 Cookie 中，每个博客独立 |
| Private（私有） | 是 | 404（假装不存在） |

这一步发生在 `processCollectionPermissions()` 中，是所有集合相关操作的总入口。

#### 第二道门：作者验证

即使博客是公开的，也只有文章的所有者才能编辑它。这一步发生在 `handleViewPad()` 中：

```go
if appData.Post.OwnerID != appData.User.ID {
    // 不是作者，重定向到文章公开页
    return impart.HTTPError{http.StatusFound, r.URL.Path[:strings.LastIndex(r.URL.Path, "/edit")]}
}
```

**为什么要双重检查？**

独立草稿只有一道门（列表查询的 `owner_id = ?`），因为草稿 URL 是 `/d/{id}/edit`，ID 难猜，且列表只显示自己的。

集合文章有两道门，因为：
1. 博客可能本身就不可见（Private/Protected）
2. 即使博客可见，文章也只能由作者编辑

---

## 三、定时发布的可见性：为什么列表和详情不一样

这是最容易让人困惑的设计。

### 3.1 现象

假设你设置了一篇文章 `created = 明天上午10点`：

- **博客列表页**：访客看不到这篇文章；你作为作者能看到，且标题旁边有"Scheduled"标签
- **文章详情页**：如果有人知道完整 URL（`/{blog}/{slug}`），**即使是访客也能直接打开看全文**

### 3.2 根因：两个查询函数的设计差异

| 函数 | 用途 | 有无时间过滤 |
|------|------|-------------|
| `GetPosts(includeFuture)` | 列表查询（博客首页、归档、标签） | 有。非作者时加 `AND created <= NOW()` |
| `GetPost(id, collectionID)` | 单篇详情查询 | **没有**。直接按 slug 返回，不检查时间 |

列表查询时，`includeFuture` 参数永远传入 `isCollOwner`（你是不是作者），所以：
- 作者 = `true` → 不过滤，看得到未来文章
- 非作者 = `false` → 加时间过滤，看不到未来文章

但是单篇查询 `GetPost` 根本没有这个参数，也没有时间判断。

### 3.3 设计意图

这不是 bug，是有意为之的**"通过不公开实现保密"**：

1. 文章的 slug 通常由标题自动生成（比如"我的假期计划" → `my-vacation-plan`），如果标题是私密的，外人很难猜到
2. 列表页是主要的发现入口，过滤掉定时文章就达到了"未发布不被发现"的目的
3. 如果你作为作者主动把 URL 分享给别人预览，那是你授权的

代码里自己也承认了参数命名有问题（注释 TODO）：
```go
// TODO: change includeFuture to isOwner, since that's how it's used
```

### 3.4 独立草稿列表的特殊情况

在"我的草稿"列表中，即使某篇草稿设置了未来发布时间，你也看不到"Scheduled"标签。原因很简单：草稿本身就是"未发布"状态，Scheduled 标记只对"已发布到博客"的文章有意义——表示"已发布但还没到公开时间"。

---

## 四、页面渲染：四层权限层叠模型

当一篇文章被最终渲染成 HTML 时，权限检查是逐层生效的。任何一层不通过，流程就中断了。

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
          │                      │  集合可见性检查    │
          │                      │ Private → 404     │
          │                      │ Protected → 密码页 │
          │                      └────────┬──────────┘
          │                               │
          │                      ┌────────▼─────────┐
          │                      │ 第 2 层：文章级    │
          │                      │  作者验证 + 状态   │
          │                      │ 非作者 → 重定向    │
          │                      │ Gone → 410        │
          │                      │ silenced → 404    │
          │                      └────────┬──────────┘
          └───────────────┬───────────────┘
                          ▼
                 ┌─────────────────┐
                 │ 第 3 层：内容级    │
                 │  protectDraft     │
                 │ 曾属私有集合的独立文章  │
                 │ 非作者 → 404     │
                 └────────┬────────┘
                          ▼
                 ┌─────────────────┐
                 │ 第 4 层：模板级    │
                 │  展示差异         │
                 │ IsOwner 控制按钮  │
                 │ Scheduled 徽章    │
                 │ paid/more 截断    │
                 └─────────────────┘
```

### 第 1 层：集合级（仅集合文章有）

- Private 博客：非作者一律 404，假装不存在
- Protected 博客：非作者需要输密码，授权存在 Cookie 里
- 被禁言（silenced）作者的 Protected 博客：对外也 404

### 第 2 层：文章级

- 集合文章在编辑器入口检查 OwnerID，非作者不让进
- `title == "" && content == ""`（已取消发布）→ 410 Gone
- 作者被禁言且访问者不是作者/管理员 → 404

### 第 3 层：内容级（protectDraft 机制）

独立文章的 URL `/{id}` 本身是公开的。但如果这篇文章的 `collection_id` 曾经指向一个 Private 或 Protected 博客（比如你曾经把它发布到私有博客，后来又撤回为草稿），那么非作者访问这个独立 URL 也会被 404。

这是为了防止一个漏洞：你把文章发布到私有博客 → 记下它的独立 ID → 撤回为草稿 → 把独立 ID 分享给别人绕过私有博客权限。protectDraft 机制堵上了这个口子。

### 第 4 层：模板级展示差异

即使通过了所有权限检查，最终看到的页面也会因身份而异：

| 展示元素 | 作者 | 非作者 |
|---------|------|--------|
| edit/delete/pin 按钮 | ✅ 显示 | ❌ 隐藏 |
| Scheduled 徽章（集合内定时文章） | ✅ 显示 | ✅ 也显示（如果能进来） |
| `<!--more-->` 截断 | 列表页截断，详情页全文 | 同左 |
| `<!--paid-->` 付费内容 | 全文可见，有"订阅内容开始"提示 | 截断，显示 Coil 会员提示 |
| 邮件订阅表单 | 显示"你已订阅 / 退订" | 显示订阅输入框 |

---

## 五、未发布内容（Gone）的处理差异

"已取消发布" = `title` 和 `content` 同时为空。此时的返回状态码取决于访问路径：

| 请求格式 | 独立文章（/{id}） | 集合文章（/{coll}/{slug}） |
|---------|------------------|------------------------|
| 浏览器 HTML | 410 Gone 页面 | 410 Gone |
| JSON API | HTTP 200，返回 `{"error": "Post was unpublished."}` | 410 Gone |
| 纯文本 / Markdown | HTTP 200，返回 "Post was unpublished." | 410 Gone |

集合文章的处理更干脆——不管什么格式统一 410。独立文章对机器可读格式做了妥协，返回 200 带错误信息，这是早期 API 兼容的历史遗留。

---

## 六、设计取舍总览

理解整个系统的关键是看**每个环节在为什么场景做优化**：

| 设计决策 | 适用场景 | 背后逻辑 |
|---------|---------|---------|
| 列表查询靠 `owner_id = ?` 天然隔离 | 独立草稿 | 草稿是私密的，SQL 层过滤最简单可靠 |
| 编辑器不做 OwnerID 检查 | 独立草稿 | 列表已过滤，且 URL 难猜 |
| 前置集合可见性检查 | 集合文章 | 博客本身有 Public/Private/Protected 之分 |
| 编辑器必须做 OwnerID 检查 | 集合文章 | 博客 URL 是公开的，任何人都可能构造编辑链接 |
| 列表过滤 `created <= NOW()`，详情不过滤 | 定时发布 | 通过不公开（列表中看不到）实现保密，不防知道 URL 的人 |
| protectDraft 检查 | 独立文章 | 防止用独立 URL 绕过曾经归属的私有博客权限 |

---

## 七、关键文件索引

| 文件 | 在流程中的角色 |
|------|---------------|
| [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/database.go#L2129-L2166) | `GetAnonymousPosts(u, page)`：独立草稿列表查询，`WHERE owner_id = ?` 天然隔离 |
| [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/database.go#L1303-L1369) | `GetPosts(includeFuture)`：集合文章列表查询，时间过滤逻辑在这里 |
| [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/database.go#L1160-L1208) | `GetPost(id, collectionID)`：单篇查询，**没有**时间过滤，是列表/详情可见性不一致的根因 |
| [pad.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/pad.go#L23-L121) | `handleViewPad()`：编辑器入口，**独立草稿无 OwnerID 检查**，**集合文章有 OwnerID 检查** |
| [posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/posts.go#L1382-L1411) | `getRawPost()`：按 ID 查独立草稿，不检查 owner |
| [posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/posts.go#L1414-L1451) | `getRawCollectionPost()`：按 slug+集合查文章，供后续 OwnerID 对比 |
| [collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/collections.go#L725-L825) | `processCollectionPermissions()`：集合可见性总入口，Private/Protected 检查 |
| [collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/collections.go#L857-L919) | `handleViewCollection()`：博客首页渲染，先过集合权限，再查文章列表 |
| [account.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/account.go#L774-L818) | `viewArticles()`：独立草稿列表页，直接调 `GetAnonymousPosts(u, 1)` |
| [posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/posts.go#L286-L288) | `IsScheduled()`：定时发布判定 `p.Created.After(time.Now())` |
| [postrender.go](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/postrender.go#L47-L96) | `formatContent()`：`<!--paid-->` / `<!--more-->` 截断逻辑 |
| [templates/user/articles.tmpl](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/templates/user/articles.tmpl) | 独立草稿列表模板，**没有** `IsScheduled` 判断 |
| [templates/include/posts.tmpl](file:///d:/fz/0601-1/solo-dogfeeding/code/31-writefreely/templates/include/posts.tmpl#L3) | 集合文章列表模板，**有** `{{if .IsScheduled}}<p class="badge">Scheduled</p>{{end}}` |
