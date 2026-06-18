# Webmention 入站处理代码链路分析

> 目标：沿「来源验证 → 内容抓取 → 归档入库 → 回显」四阶段，理清 WriteFreely 中提及/交互的入站处理链路。

## 0. 背景与重要前提

在动手前需要先澄清一个容易踩坑的事实，否则会沿着错误的协议找代码：

- **WriteFreely 没有实现 W3C Webmention 协议。** 全仓库检索 `webmention`（不区分大小写）零命中，不存在 `/webmention` 端点，也没有 `source`/`target` 表单字段、microformats 解析（`h-entry` 解析侧）等 Webmention 接收器特征。
- **联邦通信基于 ActivityPub。** 所谓「提及/交互的入站处理」实际发生在 ActivityPub 的 **inbox** 端点。模板里出现的 `h-entry`/`h-card` 等 microformats2 标记（见 [`collection-post.tmpl`](./templates/collection-post.tmpl#L68)）是给外部爬虫/解析器用的出站标记，并非入站接收逻辑。
- 因此本文将「提及入站」四阶段映射到**实际的 ActivityPub inbox 入站链路**，并在第 8 节给出与标准 Webmention 接收器的逐阶段对照表。
- 第 7 节单独展开 Like / Follow / Undo / Delete 四个分支的来源校验、响应写回、入库方式、失败行为差异。

依赖佐证：[`go.mod`](./go.mod) 中联邦相关依赖为 `github.com/writeas/activity/streams`、`github.com/writeas/activityserve`、`github.com/writeas/httpsig`、`github.com/writeas/web-core/activitypub`，以及间接依赖 `github.com/go-fed/httpsig`——均为 ActivityPub 栈，无 webmention 库。

## 1. 入口与路由

入站请求落在 collection 的 inbox 路由上：

- 路由注册：[`routes.go`](./routes.go#L155-L155)
  ```
  apiColls.HandleFunc("/{alias}/inbox", handler.All(handleFetchCollectionInbox)).Methods("POST")
  ```
- 处理函数本体：[`handleFetchCollectionInbox`](./activitypub.go#L317-L738)（`activitypub.go`）。
- 包装器 `handler.All`：[`handle.go`](./handle.go#L550-L581)。它只做 panic 恢复与请求日志，**第 567 行留有 `// TODO: do any needed authentication` 注释**——即入站侧没有任何鉴权中间件。这是后续「来源验证」阶段的关键前提。

另有一条「查看提及对象」的页面路由（非协议入站，仅解析 `@handle` 跳转到远端 profile）：[`routes.go`](./routes.go#L77-L77) → [`handleViewMention`](./collections.go#L1011-L1011)，不要与 inbox 混淆。

## 2. 来源验证（Source Validation）

这是整条链路里最不直白、也最需要警惕的一段。

**关键结论：入站 ActivityPub 请求不做 HTTP 签名验证。** 仓库里 `httpsig` 仅用于「出站签名」（见 [`makeActivityPost`](./activitypub.go#L764-L764) 与 [`resolveIRI`](./activitypub.go#L816-L816) 调用 `httpsig.NewSigner`），全仓库检索 `httpsig.Verify` / `VerifySignature` 零命中。也就是说，任何能 POST 到 `/{alias}/inbox` 的请求都会进入业务逻辑。

实际的「来源验证」退化为「拉取并确认 actor 存在」：

1. [`handleFetchCollectionInbox`](./activitypub.go#L317-L738) 先按 `alias` 取出本地 collection，并做静默检查 [`IsUserSilenced`](./activitypub.go#L333-L333)（被静默的博客直接返回 404，相当于拒绝入站）。
2. 各回调内通过 [`getActor`](./activitypub.go#L1053-L1096) 处理来源：
   - 先 [`getRemoteUser`](./activitypub.go#L1001-L1017) 查本地 `remoteusers` 表；命中即复用。
   - 未命中（404）则真正去远端拉取 actor：[`resolveIRI`](./activitypub.go#L799-L849) 用实例私钥签名后 `GET` actor IRI，再用 [`unmarshalActor`](./activitypub.go#L1166-L1204) 规范化（兼容各实现的 `@context` 字段差异）。
   - 注意 `getActor` 还会二次 `resolveIRI` `baseActor.PublicKey.Owner`（[`activitypub.go#L1077`](./activitypub.go#L1077-L1077)）拿「真正 actor」，存在两段式抓取。
3. 这一阶段只是「能拉到 actor 就算来源可信」，**并未用 actor 公钥校验本次请求签名**——这是与标准 ActivityPub 安全模型的最大偏差，也是与 Webmention「回源验证 source 链接 target」在语义上最接近却又不等价的环节。

## 3. 内容抓取（Content Fetching）

入站 body 的解析与分发：

1. 读 body 并 `json.Decode` 到 `map[string]any`：[`activitypub.go#L357-L365`](./activitypub.go#L357-L365)。
2. 用 `streams.Resolver` 按活动类型分发回调：[`activitypub.go#L378-L548`](./activitypub.go#L378-L548)，目前仅注册：
   - `LikeCallback`（[`L379`](./activitypub.go#L379-L379)）
   - `FollowCallback`（[`L428`](./activitypub.go#L428-L428)）
   - `UndoCallback`（处理 Undo:Like / Undo:Follow，[`L477`](./activitypub.go#L477-L477)）
   - `DeleteCallback`（[`L539`](./activitypub.go#L539-L539)）
3. **没有 `CreateCallback`。** 解析失败/未知类型会落到 [`L549-L559`](./activitypub.go#L549-L559) 仅记录日志并返回 200。这意味着远端发来的「带 Mention 标签的 Create 活动」（即真正意义上的「别人提及了我」）**不会被解析、不会被入库**——只是被「已读回执」。
4. 对 Like/Unlike，进一步从 `object` IRI 抽取本地文章 ID：[`parsePostIDFromURL`](./activitypub.go#L1206-L1232)，用 `apCollectionPostIRIRegex` / `apDraftPostIRIRegex`（[`L50-L51`](./activitypub.go#L50-L51)）匹配，必要时再 `GetCollection`+`GetPost` 反查 slug→postID。
5. actor 侧的「内容抓取」复用第 2 阶段的 `getActor`/`resolveIRI`。

> 对比：典型 Webmention 接收器在此阶段会抓取 source 页面 HTML 并解析 `h-entry` 取摘要/作者；这里抓取的是 **actor 的 ActivityStreams JSON**，抓的是「谁」，而非「说了什么内容」。所以「回显」阶段也只能回显计数，无法回显提及正文（见第 5 节）。

## 4. 归档入库（Archiving / Storage）

按活动类型分别落库（事务包裹），表结构见 [`schema.sql`](./schema.sql) 与迁移：

- **Like** → `INSERT INTO remote_likes`：[`activitypub.go#L578`](./activitypub.go#L578-L578)。表由迁移 [`v16.go`](./migrations/v16.go#L20-L25)（`supportRemoteLikes`，注册于 [`migrations.go#L74`](./migrations/migrations.go#L74-L74)）创建，主键 `(post_id, remote_user_id)`。
- **Unlike（Undo:Like）** → `DELETE FROM remote_likes`：[`activitypub.go#L618`](./activitypub.go#L618-L618)。
- **Follow** → 三张表联动写入：
  - `remoteusers`（actor/inbox/shared_inbox/url）：[`activitypub.go#L678`](./activitypub.go#L678-L678)，或统一封装 [`apAddRemoteUser`](./database_activitypub.go#L21-L49)（Like 分支会走这里补 actor）。
  - `remoteuserkeys`（actor 公钥）：[`activitypub.go#L695`](./activitypub.go#L695-L695)（[`database_activitypub.go#L36`](./database_activitypub.go#L36-L36)）。
  - `remotefollows`（collection_id, remote_user_id）：[`activitypub.go#L706`](./activitypub.go#L706-L706)。
- **Unfollow（Undo:Follow）** → `DELETE FROM remotefollows`：[`activitypub.go#L723`](./activitypub.go#L723-L723)。
- **Delete** → 仅 [回 200](./activitypub.go#L539-L547)，**不真正删除**本地数据。

Follow 的落库在 [`L639-L728`](./activitypub.go#L639-L728) 的 `go func()` 异步执行（先睡 2 秒、发 Accept、再写库），是链路里最容易看漏的「异步分支」。

表结构参考：`remotefollows`（[`schema.sql#L152`](./schema.sql#L152-L152)）、`remoteuserkeys`（[`schema.sql#L165`](./schema.sql#L165-L165)）、`remoteusers`（[`schema.sql#L179`](./schema.sql#L179-L179)）。

## 5. 回显（Display / Echo）

入站 Like 最终以「计数」形式回显，链路如下：

1. 计数查询：[`GetPostLikeCounts`](./database.go#L1264-L1274) `SELECT COUNT(*) FROM remote_likes WHERE post_id = ?`。
2. 装配到 Post：
   - 单篇：[`getPost`](./database.go#L1197-L1197) 写入 `p.LikeCount`。
   - 列表：[`GetPosts 循环`](./database.go#L2107-L2112) 逐篇写入。
3. 字段映射：[`processPost`](./posts.go#L1196-L1205) `res.Likes = p.LikeCount`。结构体字段定义见 [`Post.LikeCount`](./posts.go#L116-L116) 与 [`PublicPost.Likes`](./posts.go#L139-L139)。
4. 模板渲染：
   - 文章页：[`collection-post.tmpl#L58`](./templates/collection-post.tmpl#L58-L58) `{{if .Likes}}<strong>{{largeNumFmt .Likes}}</strong> {{pluralize "like" "likes" .Likes}}{{end}}`——注意外层 `{{ if and .IsOwner .IsFound }}`，**仅博主本人可见**。
   - 统计页：[`stats.tmpl#L61`](./templates/user/stats.tmpl#L61-L61) `{{.LikeCount}}`（受 `{{if $.Federation}}` 开关控制）。

> 回显仅是「点赞计数」，没有「谁提及了我 / 提及了什么」的明细列表——因为第 3 节并未抓取并归档提及正文。Follower 侧的回显走 `remotefollows` 计数，不在本次「mention 入站」范围内。

## 6. 链路总览

```
POST /{alias}/inbox
  └─ handler.All (handle.go:550)  [无鉴权中间件，仅 panic 恢复 + 日志]
       └─ handleFetchCollectionInbox (activitypub.go:317)
            ├─ 取 collection + IsUserSilenced 静默拦截
            ├─ json.Decode body → map
            ├─ streams.Resolver 分发
            │    ├─ Like   → parsePostIDFromURL + getActor (来源验证/抓取)
            │    ├─ Follow → getActor
            │    ├─ Undo   → (Like | Follow)
            │    └─ Delete → 仅 200
            └─ 处理结果：
                 · Like   → INSERT remote_likes (L578)
                 · Unlike → DELETE remote_likes (L618)
                 · Follow → go func(): 发 Accept → INSERT remoteusers/remoteuserkeys/remotefollows (L678/695/706)
                 · Unfollow→ DELETE remotefollows (L723)
                 · 未注册类型(含 Create/Mention) → 200，不入库

回显：remote_likes → GetPostLikeCounts (database.go:1264)
        → getPost/GetPosts 装填 LikeCount (L1197/L2107)
        → processPost: res.Likes = p.LikeCount (posts.go:1199)
        → collection-post.tmpl:58 / stats.tmpl:61
```

来源验证与内容抓取共享底层：[`getActor`](./activitypub.go#L1053-L1096) → [`getRemoteUser`](./activitypub.go#L1001-L1017)（本地）→ [`resolveIRI`](./activitypub.go#L799-L849)（远端 GET + 出站签名）→ [`unmarshalActor`](./activitypub.go#L1166-L1204)。

## 7. 四分支逐分支详解

上一节的链路总览偏宏观，本节按 `streams.Resolver` 注册的四个回调（Like / Follow / Undo / Delete）逐支拆解，重点对比**来源校验强度、响应写回时机、入库方式、失败行为**四个维度。

四个分支共用同一套前置流程（取 collection、静默检查、json.Decode），在 [`activitypub.go#L378`](./activitypub.go#L378-L378) 的 `streams.Resolver` 处开始分叉。回调的返回值会影响 [`res.Deserialize(m)`](./activitypub.go#L549-L549) 的结果：
- 回调返回 `nil` → Deserialize 成功，继续走后续同步/异步处理
- 回调返回 `error` → Deserialize 失败，落到 [`L549-L560`](./activitypub.go#L549-L560) 的错误兜底：记日志、回 200 空 body、`return nil`

> **重要结论**：**回调阶段的所有错误最终都是「静默 200」**，不会向调用方返回 4xx/5xx。只有回调成功后、进入同步处理块（Like/Unlike）时发生的 DB 错误，才会以 `return err` 的形式冒泡到外层 handler（最终变成 500）。异步 goroutine 里的错误只打 log，不影响响应。

### 7.1 Like 分支

**入口**：[`LikeCallback`](./activitypub.go#L379-L426)

**来源校验**：强度中等。
1. 先从 Like 活动取 `object` IRI（即被点赞的帖子），用 [`parsePostIDFromURL`](./activitypub.go#L1206-L1232) 校验并解析为本地 `post_id`——这相当于「target 归属校验」。若 `object` 缺失或不是本站 IRI，直接返回 error（最终静默 200）。
2. 再从 Like 活动取 `actor`（点赞者），调 [`getActor`](./activitypub.go#L1053-L1096) 完整两段式拉取：本地 `remoteusers` 查不到就去远端 GET actor IRI，再 GET `publicKey.owner` 拿「真正 actor」。
3. **不校验签名**：只确认 actor 存在，不用 actor 公钥校验本次请求。

**响应写回**：容易误读的一段逻辑。
- 回调末尾设 `responseWritten = true`（[L425](./activitypub.go#L425-L425)），但**回调本身并不写响应**，只 `return nil`。
- 真正写响应在同步处理块末尾 [`L601`](./activitypub.go#L601-L601)：`impart.RenderActivityJSON(w, "", http.StatusOK)`——**空 body**。
- `responseWritten = true` 的作用只是阻止函数末尾 [`L730-L735`](./activitypub.go#L730-L735) 的 fallback 再写一次（写两次会触发 http 库 "superfluous WriteHeader" 告警）。

**入库**：同步事务，单表单条写入。
- 位置：[`L563-L602`](./activitypub.go#L563-L602)（`if isLike { ... }` 块）
- 过程：`db.Begin()` → 若 `remoteUser` 为 nil 则先 [`apAddRemoteUser`](./database_activitypub.go#L21-L49) 补 actor → `INSERT INTO remote_likes` → `Commit`
- 表：`remote_likes(post_id, remote_user_id, created)`，主键 `(post_id, remote_user_id)`

**失败条件与行为**：

| 失败点 | 触发条件 | 行为 |
| --- | --- | --- |
| `HasObject(0) == NoPresence` | Like 活动没有 object | 回调返回 error → Deserialize 失败 → 200 空 body |
| `GetObjectIRI(0) == nil` | object 不是 IRI 形式 | 同上 |
| `parsePostIDFromURL` 失败 | object IRI 不匹配本站格式 / 文章不存在 | 同上 |
| `GetActor` 失败 | actor 拉不到 | 同上 |
| `db.Begin()` 失败 | DB 连不上 | 同步块 `return err` → 外层 500 |
| `INSERT remote_likes` 失败（非重复键） | DB 写入异常 | 回滚 + `return err` → 外层 500 |
| `INSERT remote_likes` 失败（重复键） | 重复点赞 | **也回滚 + 报错**（[`L584-L587`](./activitypub.go#L584-L587) 两个分支逻辑相同，都是 `return fmt.Errorf(...)`）→ 外层 500 |
| `Commit` 失败 | 提交失败 | 回滚 + `return err` → 外层 500 |

> 小瑕疵：重复键的分支跟普通错误分支行为完全一样，都回滚都返回 500，代码里写了 `if !app.db.isDuplicateKeyErr(err)` 但 else 分支并没有做幂等处理。

### 7.2 Follow 分支

**入口**：[`FollowCallback`](./activitypub.go#L428-L476)

**来源校验**：强度中等，跟 Like 基本一致但校验对象不同。
1. 从 Follow 活动取 `actor`（关注者），调 [`getActor`](./activitypub.go#L1053-L1096) 完整两段式拉取。
2. `object` 端（被关注的 collection）做了双重兜底：先 `GetObjectIRI`，拿不到就 `GetObject(0).GetId()`（[`L452-L462`](./activitypub.go#L452-L462)），再拿不到就只打 log 不报错。**没有校验 object 是否是本地 collection**——理论上外部可以用任意 object IRI 触发 Follow 入库。
3. 同样不校验签名。

**响应写回**：回调内即时写，body 为原始 payload。
- 位置：回调末尾 [`L475`](./activitypub.go#L475-L475) `return impart.RenderActivityJSON(w, m, http.StatusOK)`
- body：**完整的入站 JSON `m`**（补了 `@context`），不是空 body。
- 同时设 `responseWritten = true`（[`L474`](./activitypub.go#L474-L474)）。

**入库**：异步 goroutine，三表联动，有 2 秒延迟。
- 位置：[`L639-L728`](./activitypub.go#L639-L728) 的 `go func()` 内、`if isFollow { ... }` 块
- 时序：`time.Sleep(2 * time.Second)` → 序列化 Accept 活动 → `makeActivityPost` 发到远端 inbox → **之后才落库**（[`L664-L720`](./activitypub.go#L664-L720)）
- 三表：
  - `remoteusers`（actor_id, inbox, shared_inbox, url）：本地查不到才 INSERT
  - `remoteuserkeys`（id, remote_user_id, public_key）：同上
  - `remotefollows`（collection_id, remote_user_id, created）：必 INSERT

**失败条件与行为**：

| 失败点 | 触发条件 | 行为 |
| --- | --- | --- |
| `GetActor(0) == nil` | Follow 没有 actor | 回调返回 error → Deserialize 失败 → 200 空 body |
| `getActor` 失败 | actor 拉不到 | 同上 |
| `to == nil` | `GetActor(0)` 返回 nil | 回调返回 `fmt.Errorf("No valid 'to' string")` → 同上 |
| **Accept 序列化失败** | `a.Serialize()` 报错 | goroutine 内 log + return，**不落库** |
| **Accept 发送失败** | `makeActivityPost` 报错 | goroutine 内 log + return，**不落库** |
| `INSERT remoteusers` 失败 | DB 写入异常（非重复键） | goroutine 内回滚 + log + return |
| `LastInsertId` 失败 | 拿不到自增 ID | goroutine 内回滚 + log + return |
| `INSERT remoteuserkeys` 失败（非重复键） | 公钥写入异常 | goroutine 内回滚 + log + return |
| `INSERT remoteuserkeys` 失败（重复键） | 公钥已存在 | **不回滚，继续走**（[`L696-L702`](./activitypub.go#L696-L702) 只打 log，不 return） |
| `INSERT remotefollows` 失败（非重复键） | DB 写入异常 | goroutine 内回滚 + log + return |
| `INSERT remotefollows` 失败（重复键） | 重复关注 | **不回滚，不报错，静默继续**（[`L707-L712`](./activitypub.go#L707-L712) 只有 `if !isDuplicateKeyErr` 才回滚） |
| `Commit` 失败 | 提交失败 | goroutine 内回滚 + log + return |

> 关键差异：Follow 的入库发生在「Accept 发出去之后」，如果 Accept 发送失败，**库也不落**——相当于「先通知对方我接受了，再自己记账」。另外重复键的处理是幂等放过，跟 Like 分支的「重复键也报错」策略不同。

### 7.3 Undo 分支

Undo 是四个分支里最复杂的，因为它内部又分 **Undo:Like** 和 **Undo:Follow** 两种子情况，两种子情况的来源校验强度、响应时机、入库方式都不一样。

**入口**：[`UndoCallback`](./activitypub.go#L477-L538)，内部通过 [`u.ResolveObject`](./activitypub.go#L487-L507) 二次分发。

#### 7.3.1 Undo:Like（取消点赞）

**来源校验**：与 Like 分支**完全相同**——`getActor` 两段式拉取 + `parsePostIDFromURL` 校验 object。
- 位置：[`UndoCallback` 内的 `LikeCallback`](./activitypub.go#L488-L504)

**响应写回**：**回调里不写，也不设 `responseWritten`**。
- `isUnlike = true` 后 `return nil`，外层 `UndoCallback` 也随之 `return nil`（[`L511-L513`](./activitypub.go#L511-L513)）。
- `responseWritten` 保持 `false`。
- 真正写响应在同步处理块末尾 [`L635`](./activitypub.go#L635-L635)：空 body 200。

**入库**：同步事务，单条 DELETE。
- 位置：[`L603-L637`](./activitypub.go#L603-L637)（`else if isUnlike { ... }` 块）
- 过程：`db.Begin()` → 若 `remoteUser` 为 nil 则先 `apAddRemoteUser` 补 actor → `DELETE FROM remote_likes WHERE post_id = ? AND remote_user_id = ?` → `Commit`
- 注意：**即使 actor 在本地不存在，也会先插入再删除**——有点反直觉，但代码逻辑就是如此（`remoteUser` 为 nil 时调 `apAddRemoteUser` 新增）。

**失败条件**：与 Like 对称。`DELETE` 删不到记录不算错误（SQL 不报错），只会静默成功。DB 层错误（连不上等）会 `return err` → 外层 500。

#### 7.3.2 Undo:Follow（取消关注）

**来源校验**：**比 Follow 分支弱**——只查本地，不回源拉取。
- 位置：[`L515-L535`](./activitypub.go#L515-L535)
- 只调 [`getRemoteUser`](./activitypub.go#L1001-L1017)（纯本地 `remoteusers` 表查询），**不调 `getActor`**。
- 如果本地查不到（`ErrNoRows`），直接返回 error → Deserialize 失败 → 200 空 body。**不会去远端确认 actor 是否真的存在过**。
- 注释也说明了这是简化实现：[`L517`](./activitypub.go#L517-L517) `// TODO: get actor from object.object, not object`

**响应写回**：回调内即时写，body 为原始 payload。
- 位置：[`L537`](./activitypub.go#L537-L537) `return impart.RenderActivityJSON(w, m, http.StatusOK)`
- 同时设 `responseWritten = true`（[`L536`](./activitypub.go#L536-L536)）。
- 行为与 Follow 分支一致。

**入库**：异步 goroutine，单条 DELETE，**非事务**。
- 位置：[`L721-L726`](./activitypub.go#L721-L726) 的 `else if isUnfollow { ... }` 块
- SQL：`DELETE FROM remotefollows WHERE collection_id = ? AND remote_user_id = (SELECT id FROM remoteusers WHERE actor_id = ?)`
- **没有事务包裹**，直接 `app.db.Exec`。
- **失败只打 log**（[`L724-L726`](./activitypub.go#L724-L726)），不回滚不报错。
- 跟 Follow 的「先发 Accept 再落库」对应：Undo 也是「先回 200、异步里发 Accept、然后删库」——注意发 Accept 的逻辑在 `go func()` 开头 [`L647-L662`](./activitypub.go#L647-L662) 就做了，`isUnfollow` 分支在更后面。

> 两个不对称：(1) Follow 用 `getActor`（会回源），Undo:Follow 用 `getRemoteUser`（仅本地）；(2) Follow 入库是事务，Undo:Follow 入库是非事务。这是代码演进过程中产生的不一致，不是设计出来的。

### 7.4 Delete 分支

**入口**：[`DeleteCallback`](./activitypub.go#L539-L547)

**来源校验**：**完全没有**。
- 不读 actor，不调 `getActor`/`getRemoteUser`。
- 不校验 object 是什么。
- 只要 `streams.Resolver` 能识别出这是个 Delete 活动，就进入回调。

**响应写回**：回调内即时写，body 为原始 payload。
- 位置：[`L544`](./activitypub.go#L544-L544) `impart.RenderActivityJSON(w, m, http.StatusOK)`
- 设 `responseWritten = true`（[`L545`](./activitypub.go#L545-L545)）。

**入库**：**完全没有**。
- 回调之后 `isFollow/isUnfollow/isLike/isUnlike` 四个 flag 全为 false。
- 同步处理块（Like/Unlike）不命中。
- 异步 goroutine 里 `to == nil`（因为没设），直接 return（[`L640-L645`](./activitypub.go#L640-L645)）。
- 所以 Delete 就是「收到了，回个 200，什么都不删」。

**失败条件**：理论上没有——只要 JSON 能解、类型是 Delete，就成功。DB/网络都不涉及。

### 7.5 四分支对比总表

| 维度 | Like | Follow | Undo:Like | Undo:Follow | Delete |
| --- | --- | --- | --- | --- | --- |
| **来源校验强度** | 中等：`getActor` 两段式 + `parsePostIDFromURL` | 中等：`getActor` 两段式；object 不校验归属 | 中等：与 Like 完全相同 | **弱**：仅 `getRemoteUser` 本地查，不回源 | **无** |
| **签名校验** | 无 | 无 | 无 | 无 | 无 |
| **响应写回时机** | 同步处理块末尾 | 回调内即时 | 同步处理块末尾 | 回调内即时 | 回调内即时 |
| **响应 body** | 空 `""` | 原始 payload `m` | 空 `""` | 原始 payload `m` | 原始 payload `m` |
| **responseWritten 标记点** | 回调内设（但实际在同步块写） | 回调内设 + 写 | 不设 | 回调内设 + 写 | 回调内设 + 写 |
| **入库方式** | 同步事务 | **异步 goroutine**（2s 延迟 + 先发 Accept 再落库） | 同步事务 | **异步 goroutine**，非事务 | 无 |
| **入库表** | `remote_likes`（INSERT） | `remoteusers` + `remoteuserkeys` + `remotefollows` | `remote_likes`（DELETE） | `remotefollows`（DELETE） | - |
| **回调错误 → 最终 HTTP 状态** | 静默 200 空 body | 静默 200 空 body | 静默 200 空 body | 静默 200 空 body | - |
| **同步块 DB 错误 → 最终 HTTP 状态** | 500（return err 冒泡） | - | 500（return err 冒泡） | - | - |
| **异步块错误 → 最终 HTTP 状态** | - | 不影响（已回 200），仅 log | - | 不影响（已回 200），仅 log | - |
| **重复键处理** | 也报错 → 500 | 幂等放过（remoteuserkeys 重复、remotefollows 重复均不回滚） | - | - | - |
| **删除是否真删数据** | - | - | 是（DELETE remote_likes） | 是（DELETE remotefollows） | 否（仅 200） |
| **代码位置（回调）** | [`L379-L426`](./activitypub.go#L379-L426) | [`L428-L476`](./activitypub.go#L428-L476) | [`L488-L504`](./activitypub.go#L488-L504) | [`L515-L537`](./activitypub.go#L515-L537) | [`L539-L547`](./activitypub.go#L539-L547) |
| **代码位置（入库）** | [`L563-L602`](./activitypub.go#L563-L602) | [`L664-L720`](./activitypub.go#L664-L720)（异步） | [`L603-L637`](./activitypub.go#L603-L637) | [`L721-L726`](./activitypub.go#L721-L726)（异步） | - |

## 8. 标准 Webmention vs 现有 ActivityPub inbox 流程对照

为了直观回答「如果按 Webmention 的语义来看，现在这条链路分别对应到哪里、缺了什么」，下面按阶段逐项对比：

| 阶段 | 标准 Webmention 接收器 | 现有 ActivityPub inbox（WriteFreely） | 差异 / 缺失 |
| --- | --- | --- | --- |
| **入口** | `POST /webmention`，form-encoded `source` & `target` 字段 | `POST /{alias}/inbox`，ActivityStreams JSON body | 协议完全不同；但都是「通知某 URL 被外部页面引用/互动」的入站通道 |
| **来源验证** | 1. 校验 source、target 为合法 URL<br>2. 抓取 source 页面，检查其中是否真有链接指向 target（回源验证）<br>3. 可选：检查 target 是否为本站资源（目标归属校验） | 1. `getActor` 拉取远端 actor JSON，确认 actor 存在<br>2. `parsePostIDFromURL` 校验 object IRI 是否为本站文章（≈ target 归属校验）<br>3. **不做 HTTP 签名验证**（见 [`handle.go#L567`](./handle.go#L567-L567) TODO） | Webmention 是「回源查 HTML 链接」，这里是「回源查 actor JSON」；两者都验证了「来源地址可达」，但**都没实现各自协议的标准安全手段**（Webmention 的 source→target 链接检查未做，ActivityPub 的签名验证未做） |
| **内容抓取** | 抓取 source 页面 HTML → 解析 microformats2（`h-entry` 的 `p-name`/`e-content`/`p-author h-card` 等）→ 提取标题、正文摘要、作者、发布时间 | 1. 解析入站 ActivityStreams JSON（`streams.Resolver` 回调）<br>2. 再 GET actor IRI 拿作者信息（`resolveIRI`）<br>3. **不解析「提及/回复的正文内容」**（缺 Create 回调） | Webmention 抓的是「被引用页面的 HTML 内容」；这里抓的是「发件人 actor 的 JSON 元数据」。**提及正文完全不被抓取和保留**，只有 Like/Follow 这类类型级语义 |
| **归档入库** | 写入 webmentions 表：source URL、target post_id、作者信息、内容摘要、状态（待审核/已批准）、类型（mention/reply/like/repost 等） | Like → `remote_likes`<br>Follow → `remoteusers` + `remoteuserkeys` + `remotefollows`<br>**Create/Mention → 不入库** | 结构不同：Webmention 是「一条提及一条记录」，这里是「按交互类型分表存计数关联」。最大缺失是 **没有 mention/reply 类内容的表** |
| **回显** | 1. 在文章下方渲染 mention/reply 列表（作者头像、名称、内容片段、来源链接）<br>2. 可能聚合 likes/reposts 计数 | 1. 仅 Like 计数（`remote_likes COUNT`）<br>2. 仅博主本人可见（[`collection-post.tmpl#L58`](./templates/collection-post.tmpl#L58-L58) 受 `.IsOwner` 包裹）<br>3. 统计页可见总计数（[`stats.tmpl#L61`](./templates/user/stats.tmpl#L61-L61)） | 量级差异巨大：Webmention 是「完整对话回显」，这里只是「博主自用的点赞计数器」 |
| **后续动作** | 可选：发邮件通知博主、审核队列、垃圾过滤、异步重试验证 | Follow → 异步发 Accept（延迟 2s）+ 落库<br>Like → 同步落库<br>其他 → 200 静默 | WriteFreely 没有通知、审核、反垃圾流程（`spam/` 目录只有邮箱和 IP 反垃圾，未接入 inbox） |

**小结**：
- 两者在「有人从站外引用/互动了本站内容」这件事上有语义交集，但协议栈、数据形态、粒度都不同。
- WriteFreely 的 inbox 实现相对轻量化：**只消费 Like/Follow 这两类计数型交互**，对真正「带来文内容的提及/回复」（ActivityPub 中的 `Create` + `Mention` 标签）完全不处理。
- 安全性上两边都不完整：入站签名验证是 TODO，Webmention 式的回源链接校验也从未实现过。

## 9. 关键观察与风险点

1. **无入站签名校验**：[`handle.go#L567`](./handle.go#L567-L567) 的 `TODO: do any needed authentication` 至今未实现，任何人可伪造 Like/Follow 写入 `remote_likes`/`remotefollows`。与 Webmention「回源验证 source 真链向 target」的安全意图形成对照。
2. **入站提及（Create+Mention）不被处理**：缺 `CreateCallback`，[`activitypub.go#L549-L559`](./activitypub.go#L549-L559) 仅回 200。真正「提及了我」的语义在入站侧缺失；出站侧的 Mention 标签构建见 [`posts.go#L1281-L1299`](./posts.go#L1281-L1299) 与发送逻辑 [`activitypub.go#L971-L994`](./activitypub.go#L971-L994)（出站，非本主题）。
3. **Delete 是「假删除」**：[`activitypub.go#L539-L547`](./activitypub.go#L539-L547) 不清理本地 like/follow 记录。
4. **Follow 落库在异步 goroutine**：[`activitypub.go#L639`](./activitypub.go#L639-L639) `go func()`，且先 `time.Sleep(2s)` 再发 Accept 后写库，排查「收到 Follow 但 follower 未即时入库」时需注意这段延迟与异步特性。
5. **回显仅计数、仅博主可见**：[`collection-post.tmpl#L58`](./templates/collection-post.tmpl#L58-L58) 受 `.IsOwner` 包裹，访客看不到点赞数。
