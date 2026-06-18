# Webmention 入站处理代码链路分析

> 目标：沿「来源验证 → 内容抓取 → 归档入库 → 回显」四阶段，理清 WriteFreely 中提及/交互的入站处理链路。

## 0. 背景与重要前提

在动手前需要先澄清一个容易踩坑的事实，否则会沿着错误的协议找代码：

- **WriteFreely 没有实现 W3C Webmention 协议。** 全仓库检索 `webmention`（不区分大小写）零命中，不存在 `/webmention` 端点，也没有 `source`/`target` 表单字段、microformats 解析（`h-entry` 解析侧）等 Webmention 接收器特征。
- **联邦通信基于 ActivityPub。** 所谓「提及/交互的入站处理」实际发生在 ActivityPub 的 **inbox** 端点。模板里出现的 `h-entry`/`h-card` 等 microformats2 标记（见 [collection-post.tmpl](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/templates/collection-post.tmpl#L68)）是给外部爬虫/解析器用的出站标记，并非入站接收逻辑。
- 因此本文将用户关心的四个阶段映射到**实际的 ActivityPub inbox 入站链路**，并如实标注与「典型 Webmention 接收器」的差异。

依赖佐证：[go.mod](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/go.mod) 中联邦相关依赖为 `github.com/writeas/activity/streams`、`github.com/writeas/activityserve`、`github.com/writeas/httpsig`、`github.com/writeas/web-core/activitypub`，以及间接依赖 `github.com/go-fed/httpsig`——均为 ActivityPub 栈，无 webmention 库。

## 1. 入口与路由

入站请求落在 collection 的 inbox 路由上：

- 路由注册：[routes.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/routes.go#L155)
  ```
  apiColls.HandleFunc("/{alias}/inbox", handler.All(handleFetchCollectionInbox)).Methods("POST")
  ```
- 处理函数本体：[handleFetchCollectionInbox](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L317-L738)（`activitypub.go`）。
- 包装器 `handler.All`：[handle.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/handle.go#L550-L581)。它只做 panic 恢复与请求日志，**第 567 行留有 `// TODO: do any needed authentication` 注释**——即入站侧没有任何鉴权中间件。这是后续「来源验证」阶段的关键前提。

另有一条「查看提及对象」的页面路由（非协议入站，仅解析 `@handle` 跳转到远端 profile）：[routes.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/routes.go#L77) → [handleViewMention](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/collections.go#L1011)，不要与 inbox 混淆。

## 2. 来源验证（Source Validation）

这是整条链路里最不直白、也最需要警惕的一段。

**关键结论：入站 ActivityPub 请求不做 HTTP 签名验证。** 仓库里 `httpsig` 仅用于「出站签名」（见 [makeActivityPost](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L764) 与 [resolveIRI](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L816) 调用 `httpsig.NewSigner`），全仓库检索 `httpsig.Verify` / `VerifySignature` 零命中。也就是说，任何能 POST 到 `/{alias}/inbox` 的请求都会进入业务逻辑。

实际的「来源验证」退化为「拉取并确认 actor 存在」：

1. [handleFetchCollectionInbox](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L317-L738) 先按 `alias` 取出本地 collection，并做静默检查 [IsUserSilenced](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L333)（被静默的博客直接返回 404，相当于拒绝入站）。
2. 各回调内通过 [getActor](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L1053-L1096) 处理来源：
   - 先 [getRemoteUser](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L1001-L1017) 查本地 `remoteusers` 表；命中即复用。
   - 未命中（404）则真正去远端拉取 actor：[resolveIRI](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L799-L849) 用实例私钥签名后 `GET` actor IRI，再用 [unmarshalActor](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L1166-L1204) 规范化（兼容各实现的 `@context` 字段差异）。
   - 注意 `getActor` 还会二次 `resolveIRI` `baseActor.PublicKey.Owner`（[activitypub.go#L1077](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L1077)）拿「真正 actor」，存在两段式抓取。
3. 这一阶段只是「能拉到 actor 就算来源可信」，**并未用 actor 公钥校验本次请求签名**——这是与标准 ActivityPub 安全模型的最大偏差，也是与 Webmention「回源验证 source 链接 target」在语义上最接近却又不等价的环节。

## 3. 内容抓取（Content Fetching）

入站 body 的解析与分发：

1. 读 body 并 `json.Decode` 到 `map[string]any`：[activitypub.go#L357-L365](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L357-L365)。
2. 用 `streams.Resolver` 按活动类型分发回调：[activitypub.go#L378-L548](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L378-L548)，目前仅注册：
   - `LikeCallback`（[L379](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L379)）
   - `FollowCallback`（[L428](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L428)）
   - `UndoCallback`（处理 Undo:Like / Undo:Follow，[L477](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L477)）
   - `DeleteCallback`（[L539](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L539)）
3. **没有 `CreateCallback`。** 解析失败/未知类型会落到 [L549-L559](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L549-L559) 仅记录日志并返回 200。这意味着远端发来的「带 Mention 标签的 Create 活动」（即真正意义上的「别人提及了我」）**不会被解析、不会被入库**——只是被「已读回执」。
4. 对 Like/Unlike，进一步从 `object` IRI 抽取本地文章 ID：[parsePostIDFromURL](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L1206-L1232)，用 `apCollectionPostIRIRegex` / `apDraftPostIRIRegex`（[L50-L51](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L50-L51)）匹配，必要时再 `GetCollection`+`GetPost` 反查 slug→postID。
5. actor 侧的「内容抓取」复用第 2 阶段的 `getActor`/`resolveIRI`。

> 对比：典型 Webmention 接收器在此阶段会抓取 source 页面 HTML 并解析 `h-entry` 取摘要/作者；这里抓取的是 **actor 的 ActivityStreams JSON**，抓的是「谁」，而非「说了什么内容」。所以「回显」阶段也只能回显计数，无法回显提及正文（见第 5 节）。

## 4. 归档入库（Archiving / Storage）

按活动类型分别落库（事务包裹），表结构见 [schema.sql](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/schema.sql) 与迁移：

- **Like** → `INSERT INTO remote_likes`：[activitypub.go#L578](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L578)。表由迁移 [v16.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/migrations/v16.go#L20-L25)（`supportRemoteLikes`，注册于 [migrations.go#L74](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/migrations/migrations.go#L74)）创建，主键 `(post_id, remote_user_id)`。
- **Unlike（Undo:Like）** → `DELETE FROM remote_likes`：[activitypub.go#L618](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L618)。
- **Follow** → 三张表联动写入：
  - `remoteusers`（actor/inbox/shared_inbox/url）：[activitypub.go#L678](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L678)，或统一封装 [apAddRemoteUser](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/database_activitypub.go#L21-L49)（Like 分支会走这里补 actor）。
  - `remoteuserkeys`（actor 公钥）：[activitypub.go#L695](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L695)（[database_activitypub.go#L36](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/database_activitypub.go#L36)）。
  - `remotefollows`（collection_id, remote_user_id）：[activitypub.go#L706](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L706)。
- **Unfollow（Undo:Follow）** → `DELETE FROM remotefollows`：[activitypub.go#L723](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L723)。
- **Delete** → 仅 [回 200](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L539-L547)，**不真正删除**本地数据。

Follow 的落库在 [L639-L728](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L639-L728) 的 `go func()` 异步执行（先睡 2 秒、发 Accept、再写库），是链路里最容易看漏的「异步分支」。

表结构参考：`remotefollows`（[schema.sql#L152](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/schema.sql#L152)）、`remoteuserkeys`（[schema.sql#L165](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/schema.sql#L165)）、`remoteusers`（[schema.sql#L179](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/schema.sql#L179)）。

## 5. 回显（Display / Echo）

入站 Like 最终以「计数」形式回显，链路如下：

1. 计数查询：[GetPostLikeCounts](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/database.go#L1264-L1274) `SELECT COUNT(*) FROM remote_likes WHERE post_id = ?`。
2. 装配到 Post：
   - 单篇：[getPost](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/database.go#L1197) 写入 `p.LikeCount`。
   - 列表：[GetPosts 循环](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/database.go#L2107-L2112) 逐篇写入。
3. 字段映射：[processPost](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/posts.go#L1196-L1205) `res.Likes = p.LikeCount`。结构体字段定义见 [Post.LikeCount](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/posts.go#L116) 与 [PublicPost.Likes](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/posts.go#L139)。
4. 模板渲染：
   - 文章页：[collection-post.tmpl#L58](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/templates/collection-post.tmpl#L58) `{{if .Likes}}<strong>{{largeNumFmt .Likes}}</strong> {{pluralize "like" "likes" .Likes}}{{end}}`——注意外层 `{{ if and .IsOwner .IsFound }}`，**仅博主本人可见**。
   - 统计页：[stats.tmpl#L61](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/templates/user/stats.tmpl#L61) `{{.LikeCount}}`（受 `{{if $.Federation}}` 开关控制）。

> 回显仅是「点赞计数」，没有「谁提及了我 / 提及了什么」的明细列表——因为第 3 节并未抓取并归档提及正文。Follower 侧的回显走 `remotefollows` 计数，不在本次「mention 入站」范围内。

## 6. 链路总览

```
POST /{alias}/inbox
  └─ handler.All(handle.go:550)  [无鉴权中间件，仅 panic 恢复 + 日志]
       └─ handleFetchCollectionInbox (activitypub.go:317)
            ├─ 取 collection + IsUserSilenced 静默拦截
            ├─ json.Decode body → map
            ├─ streams.Resolver 分发
            │    ├─ Like   → parsePostIDFromURL + getActor(来源验证/抓取)
            │    ├─ Follow → getActor
            │    ├─ Undo   → (Like | Follow)
            │    └─ Delete → 仅 200
            └─ 处理结果：
                 · Like   → INSERT remote_likes (L578)
                 · Unlike → DELETE remote_likes (L618)
                 · Follow → go func(): 发 Accept → INSERT remoteusers/remoteuserkeys/remotefollows (L678/695/706)
                 · Unfollow→ DELETE remotefollows (L723)
                 · 未注册类型(含 Create/Mention) → 200，不入库

回显：remote_likes → GetPostLikeCounts(database.go:1264)
        → getPost/GetPosts 装填 LikeCount (L1197/L2107)
        → processPost: res.Likes = p.LikeCount (posts.go:1199)
        → collection-post.tmpl:58 / stats.tmpl:61
```

来源验证与内容抓取共享底层：[getActor](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L1053-L1096) → [getRemoteUser](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L1001-L1017)（本地）→ [resolveIRI](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L799-L849)（远端 GET + 出站签名）→ [unmarshalActor](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L1166-L1204)。

## 7. 关键观察与风险点

1. **无入站签名校验**：[handle.go#L567](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/handle.go#L567) 的 `TODO: do any needed authentication` 至今未实现，任何人可伪造 Like/Follow 写入 `remote_likes`/`remotefollows`。与 Webmention「回源验证 source 真链向 target」的安全意图形成对照。
2. **入站提及（Create+Mention）不被处理**：缺 `CreateCallback`，[L549-L559](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L549-L559) 仅回 200。真正「提及了我」的语义在入站侧缺失；出站侧的 Mention 标签构建见 [posts.go#L1281-L1299](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/posts.go#L1281-L1299) 与发送逻辑 [activitypub.go#L971-L994](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L971-L994)（出站，非本主题）。
3. **Delete 是「假删除」**：[L539-L547](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L539-L547) 不清理本地 like/follow 记录。
4. **Follow 落库在异步 goroutine**：[L639](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/activitypub.go#L639) `go func()`，且先 `time.Sleep(2s)` 再发 Accept 后写库，排查「收到 Follow 但 follower 未即时入库」时需注意这段延迟与异步特性。
5. **回显仅计数、仅博主可见**：[collection-post.tmpl#L58](file:///d:/fz/0601-2/solo-dogfeeding/code/30-writefreely/templates/collection-post.tmpl#L58) 受 `.IsOwner` 包裹，访客看不到点赞数。
