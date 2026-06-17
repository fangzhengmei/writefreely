# 联邦投递重试与退避策略分析

## 概述

WriteFreely 的联邦投递（ActivityPub）当前采用 **fire-and-forget（发射后不管）** 模式，尚未实现复杂的重试与退避机制。投递通过 goroutine 异步触发，失败仅记录错误日志，不会自动重试。

本文档从 **入队条件**、**退避梯度**、**终态判定** 三个维度分析当前实现走向。

---

## 一、入队条件

### 1.1 触发场景总览

联邦投递有**五个独立的出站入口**，均以 goroutine 异步方式执行。各入口的前置判断策略不同，关键差异在于**未来文章**是否被入口处主动拦截：

- **入口处主动挡未来文章**：在调用 `federatePost` 前就检查 `!created.After(time.Now())`，未来文章连投递函数都不进
- **入口处不挡未来文章**：调用前不检查未来文章，而 `federatePost` 函数**内部也不检查未来文章**，所以未来文章会被真实投递出去

后续防守（函数内部）只覆盖两项：**私有实例**和**集合可见性**，不覆盖未来文章。

| 入口 | 触发场景 | 入口处检查 | 是否挡未来文章 | 调用函数 |
|------|----------|------------|----------------|----------|
| **发布新文章** | 用户发布新文章到集合 | 4 项检查 | **是** ✓ | `federatePost(isUpdate=false)` |
| **更新文章** | 用户修改已发布文章 | 3 项检查 | **否** ✗（未来文章直接投出） | `federatePost(isUpdate=true)` |
| **删除文章** | 用户删除集合中的文章 | 3 项检查 | 不适用（删除无此概念） | `deleteFederatedPost` |
| **导入文章** | 批量导入 Markdown 文件 | 2 项检查 | **否** ✗（未来文章直接投出） | `federatePost(isUpdate=false)` |
| **认领文章** | 把散落文章归入集合 | 4 项检查 | **是** ✓ | `federatePost(isUpdate=false)` |

### 1.2 各入口前置判断详解

#### 1.2.1 发布新文章（主动挡未来文章）

代码位置：[posts.go:685-688](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/posts.go#L685-L688)

```go
if newPost.Collection != nil {
    if !app.cfg.App.Private && app.cfg.App.Federation && !newPost.Created.After(time.Now()) {
        go federatePost(app, newPost, newPost.Collection.ID, false)
    }
}
```

入口处**四项检查**全部满足才触发：
1. `newPost.Collection != nil` — 文章必须属于集合
2. `!app.cfg.App.Private` — 实例非私有
3. `app.cfg.App.Federation` — 联邦功能已启用
4. `!newPost.Created.After(time.Now())` — **创建时间不晚于当前**（未来文章直接跳过）

**结论**：入口处即挡下未来文章，不会进入 `federatePost`。

#### 1.2.2 更新文章（入口不挡未来文章 → 直接投出）

代码位置：[posts.go:800-806](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/posts.go#L800-L806)

```go
if pRes.CollectionID.Valid {
    coll, err := app.db.GetCollectionBy("id = ?", pRes.CollectionID.Int64)
    if err == nil && !app.cfg.App.Private && app.cfg.App.Federation {
        coll.hostName = app.cfg.App.Host
        pRes.Collection = &CollectionObj{Collection: *coll}
        go federatePost(app, pRes, pRes.Collection.ID, true)
    }
}
```

入口处**三项检查**：
1. `pRes.CollectionID.Valid` — 文章已归属集合
2. `!app.cfg.App.Private` — 实例非私有
3. `app.cfg.App.Federation` — 联邦功能已启用

**没有** `!Created.After(time.Now())` 检查。

**结论**：即使是未来文章（创建时间在将来），只要属于集合且满足全局条件，也会调用 `federatePost`。**注意：`federatePost` 内部不检查未来文章，所以未来文章会被真实投递出去。**
后续防守只挡"私有实例"和"集合可见性"两项，不挡未来文章。

#### 1.2.3 删除文章（特殊入口）

代码位置：[posts.go:943-945](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/posts.go#L943-L945)

```go
if coll != nil && !app.cfg.App.Private && app.cfg.App.Federation {
    go deleteFederatedPost(app, pp, collID.Int64)
}
```

入口处**三项检查**：
1. `coll != nil` — 文章属于集合
2. `!app.cfg.App.Private` — 实例非私有
3. `app.cfg.App.Federation` — 联邦功能已启用

注意事项：
- 调用的是 `deleteFederatedPost`，**不是** `federatePost`
- `deleteFederatedPost` 函数**内部没有任何前置检查**（没有检查私有、没有检查集合可见性），只要调用了就会尝试投递 Delete 活动
- 不存在"未来文章"概念（删除即删除）

**结论**：删除入口的防守最薄弱，函数内部完全没有二次检查。完整链路分析见 **1.7 删除活动完整链路**。

#### 1.2.4 导入文章（入口最松 → 未来文章直接投出，私有实例靠内部兜底）

代码位置：[account_import.go:165-177](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/account_import.go#L165-L177)

```go
if app.cfg.App.Federation && coll.ID > 0 {
    go federatePost(
        app,
        &PublicPost{
            Post: rp,
            Collection: &CollectionObj{
                Collection: *coll,
            },
        },
        coll.ID,
        false,
    )
}
```

入口处**仅两项检查**：
1. `app.cfg.App.Federation` — 联邦功能已启用
2. `coll.ID > 0` — 文章归属到有效集合

**入口处缺失的检查**：
- ❌ **没有** `!app.cfg.App.Private` 检查 — 靠 `federatePost` 内部兜底挡下
- ❌ **没有** `!Created.After(time.Now())` 检查 — **函数内部也不检查，未来文章直接投出**

**结论**：入口防守最松。两项缺失的处理不同：
- **私有实例**：如果入口没挡住（实际上入口不检查），进入 `federatePost` 后由内部的 `if app.cfg.App.Private { return nil }` 兜底挡下
- **未来文章**：入口不检查 + 函数内部也不检查，**未来文章会被真实投递出去**

#### 1.2.5 认领文章（主动挡未来文章）

代码位置：[posts.go:995-1004](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/posts.go#L995-L1004)

```go
for _, pRes := range *res {
    if pRes.Code != http.StatusOK {
        continue
    }
    if !app.cfg.App.Private && app.cfg.App.Federation {
        if !pRes.Post.Created.After(time.Now()) {
            pRes.Post.Collection.hostName = app.cfg.App.Host
            go federatePost(app, pRes.Post, pRes.Post.Collection.ID, false)
        }
    }
}
```

入口处**四项检查**：
1. `pRes.Code == http.StatusOK` — 认领操作成功（失败或冲突的跳过）
2. `!app.cfg.App.Private` — 实例非私有
3. `app.cfg.App.Federation` — 联邦功能已启用
4. `!pRes.Post.Created.After(time.Now())` — **创建时间不晚于当前**（未来文章跳过）

**结论**：与发布新文章一致，入口处即挡下未来文章。且由于认领相当于"首次在集合发布"，`isUpdate=false`，发送的是 Create 活动。

### 1.3 后续防守范围（federatePost 内部检查）

调用 `federatePost` 后，函数内部只有**两层**检查作为兜底，**不包含未来文章判断**：

```go
func federatePost(app *App, p *PublicPost, collID int64, isUpdate bool) error {
    // If app is private, do not federate
    if app.cfg.App.Private {
        return nil
    }

    // Do not federate posts from private or protected blogs
    if p.Collection.Visibility == CollPrivate || p.Collection.Visibility == CollProtected {
        return nil
    }
    // ...
}
```

内部仅检查两项：
1. `!app.cfg.App.Private` — 实例非私有（重复入口检查，防御性编程）
2. 集合可见性不是 `CollPrivate` 且不是 `CollProtected`

**关键结论**：`federatePost` 内部**没有**任何未来文章检查。一旦未来文章通过了入口（更新、导入场景），就会被真实投递出去，没有任何防守。

### 1.4 前置判断对照表

| 检查项 | 发布入口 | 更新入口 | 删除入口 | 导入入口 | 认领入口 | federatePost 内部 | deleteFederatedPost 内部 |
|--------|----------|----------|----------|----------|----------|-------------------|-------------------------|
| 属于集合 | ✓ | ✓ | ✓ | ✓ | 隐含 | - | - |
| 实例非私有 | ✓ | ✓ | ✓ | ✗ | ✓ | ✓ | ✗ |
| 联邦启用 | ✓ | ✓ | ✓ | ✓ | ✓ | - | - |
| 非未来文章 | ✓（入口挡） | ✗（直接投出） | 不适用 | ✗（直接投出） | ✓（入口挡） | ✗（不检查） | 不适用 |
| 集合非私有/非保护 | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✗ |
| 认领成功 | - | - | - | - | ✓ | - | - |

### 1.5 投递目标分组逻辑

入队后，在实际发送前会对收件箱进行分组优化：

```go
inboxes := map[string][]string{}
for _, f := range *followers {
    inbox := f.SharedInbox
    if inbox == "" {
        inbox = f.Inbox
    }
    if _, ok := inboxes[inbox]; ok {
        inboxes[inbox] = append(inboxes[inbox], f.ActorID)
    } else {
        inboxes[inbox] = []string{f.ActorID}
    }
}
```

分组规则：
1. **优先共享收件箱**：如果远程用户有 `SharedInbox`，则使用共享收件箱
2. **回退个人收件箱**：没有共享收件箱时，使用个人 `Inbox`
3. **按收件箱去重**：同一收件箱只发送一次，通过 CC 字段包含所有关注者

### 1.6 @提及用户的单独投递

除了向关注者投递外，还会向文章中 @提及的用户单独投递：

```go
for _, tag := range na.Tag {
    if tag.Type == "Mention" {
        activity = activitystreams.NewCreateActivity(na)
        remoteUser, err := getRemoteUser(app, tag.HRef)
        if err != nil {
            log.Error("Unable to find remote user %s. Skipping: %v", tag.HRef, err)
            continue
        }
        err = makeActivityPost(app.cfg.App.Host, actor, remoteUser.Inbox, activity)
        // ...
    }
}
```

特点：
- 直接发送到被提及用户的个人收件箱
- 不使用共享收件箱
- 找不到远程用户时跳过（不重试）

### 1.7 删除活动完整链路

删除活动走独立的 `deleteFederatedPost` 函数，链路与 `federatePost` 有显著差异。以下从**关注记录入库**、**删除取收件箱**、**为什么没有可见性检查**三个维度展开。

#### 1.7.1 远程关注记录怎么进入

远程用户的关注关系通过 **收件箱接收 Follow 活动** 建立，完整流程在 [activitypub.go:317-730](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/activitypub.go#L317-L730)：

```
远程实例 → POST /api/collections/{alias}/inbox
    ↓
handleFetchCollectionInbox()
    ├─ 检查集合是否存在（GetCollection/ByID）
    ├─ 检查集合所有者是否被 silenced（被静默则返回 404）
    ├─ **不检查集合可见性**（private/protected 集合也能收 Follow）
    ├─ 解析 Activity 并触发 FollowCallback
    │      ├─ isFollow = true
    │      ├─ 构造 Accept 活动
    │      ├─ getActor() 获取远程用户信息
    │      └─ 同步返回 200 OK（先响应，再处理）
    └─ 异步 goroutine（2 秒延迟后）
            ├─ 发送 Accept 到远程用户 inbox
            └─ 如果 Accept 发送成功：
                ├─ 若远程用户不存在 → INSERT INTO remoteusers（actor_id, inbox, shared_inbox, url）
                ├─ 若首次见 → INSERT INTO remoteuserkeys（public_key）
                └─ INSERT INTO remotefollows（collection_id, remote_user_id, created）
```

涉及两张核心表：

| 表名 | 关键字段 | 说明 |
|------|----------|------|
| `remoteusers` | `id`, `actor_id`, `inbox`, `shared_inbox`, `url` | 所有已知远程用户的元信息，`actor_id` 唯一 |
| `remotefollows` | `collection_id`, `remote_user_id`, `created` | 复合主键，记录某集合被哪些远程用户关注 |

**关键注意**：
- `remotefollows` 表**没有冗余保存集合当时的可见性**。只存 `collection_id` 和 `remote_user_id`。
- **入库时不检查集合可见性**。inbox 入口只检查集合存在和所有者未被静默，不检查 `Visibility`。理论上 private/protected 集合也能被远程用户关注并建立记录（虽然 ActivityPub 发现阶段可能已经挡住了）。

#### 1.7.2 删除时怎样取收件箱

`deleteFederatedPost` 在 [activitypub.go:851-894](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/activitypub.go#L851-L894)，取收件箱流程：

```go
func deleteFederatedPost(app *App, p *PublicPost, collID int64) error {
    // 没有任何前置检查，直接开始

    p.Collection.ID = collID
    followers, err := app.db.GetAPFollowers(&p.Collection.Collection)
    // ↑ 只用到了 c.ID，其他 Collection 字段不关心
```

`GetAPFollowers` 在 [database.go:1557-1577](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/database.go#L1557-L1577)，执行 SQL：

```sql
SELECT actor_id, inbox, shared_inbox, f.created
FROM remotefollows f
INNER JOIN remoteusers u ON f.remote_user_id = u.id
WHERE collection_id = ?
ORDER BY created DESC
```

拿到 `[]RemoteUser` 后，与 `federatePost` 一样按收件箱分组：

```go
inboxes := map[string][]string{}
for _, f := range *followers {
    inbox := f.SharedInbox
    if inbox == "" {
        inbox = f.Inbox
    }
    inboxes[inbox] = append(inboxes[inbox], f.ActorID)
}
```

然后构造 Delete 活动逐收件箱发送：

```go
for si, instFolls := range inboxes {
    na.CC = instFolls
    da := activitystreams.NewDeleteActivity(na)
    da.ID += "#Delete"  // 特殊后缀，兼容 Pleroma
    err = makeActivityPost(app.cfg.App.Host, actor, si, da)
}
```

**取件箱逻辑小结**：
1. 用 `collID` 查 `remotefollows` → `remoteusers` JOIN，拿到所有关注者
2. 与创建/更新一样按 shared_inbox 优先分组去重
3. 没有过滤逻辑（不排除已失效的 inbox、不重试失败地址）

#### 1.7.3 为什么不再走集合可见性兜底

`deleteFederatedPost` 不像 `federatePost` 那样检查 `Collection.Visibility`，原因有设计合理性和实现疏漏两方面：

**原因 1：语义合理性 —— Delete 是撤回已发内容**

Create/Update 与 Delete 的语义不同：
- **Create/Update**：在向外推送**新内容**。如果集合后来变成 private/protected，就不该再向外推新内容了。
- **Delete**：在**撤回**之前已经推送出去的内容。那篇文章在集合是 public 时已经发出去了，远程实例可能已经缓存、展示、传播。即使集合后来变 private，也应该通知远程实例把之前收到的那篇文章删掉，否则会出现"源站已删但远端还在展示"的不一致。

**原因 2：入库时就不检查可见性 —— 没有历史记录可查**

如 1.7.1 所述，`remotefollows` 入库时不记录集合当时的 `Visibility`，表结构里也没有冗余字段。如果删除时想判断"这批关注是在集合公开时加的还是变私有后加的"，根本查不到。

**原因 3：可能的实现疏漏 —— 与入口检查不一致**

删除入口 [posts.go:943-945](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/posts.go#L943-L943) 是有 `!app.cfg.App.Private` 和 `coll != nil` 检查的：

```go
if coll != nil && !app.cfg.App.Private && app.cfg.App.Federation {
    go deleteFederatedPost(app, pp, collID.Int64)
}
```

但函数内部把实例私有检查和集合可见性检查都丢了。这可能是**疏漏**——如果实例在删除前被管理员改为私有，入口检查能挡住，但如果删除的那一瞬间集合从 public 改成了 private，入口检查过了但函数内部该挡不挡。

**两类"后来变私有"的场景对比**：

| 场景 | 入口检查 | 函数内部检查 | 结果 |
|------|----------|--------------|------|
| 实例先改私有，再删文章 | `!app.cfg.App.Private` 挡住 | 无需到函数 | ✓ 不投递 |
| 删文章的瞬间集合从 public 改 private | `coll != nil` 已过 | 函数内部不检查 Visibility | ✗ 投递了 |
| 实例在 goroutine 调度期间改私有 | 入口检查过了 | `deleteFederatedPost` 不检查实例私有 | ✗ 投递了 |

对比 `federatePost` 有实例私有兜底检查，`deleteFederatedPost` 缺失的这两项检查（实例私有、集合可见性）更像是不一致的疏漏，而非刻意设计。

---

## 二、退避梯度

### 2.1 当前状态：无退避梯度

当前联邦投递**没有重试机制**，因此也不存在退避梯度。`makeActivityPost` 函数只执行一次 HTTP 请求：

```go
func makeActivityPost(hostName string, p *activitystreams.Person, url string, m interface{}) error {
    // ... 构造请求 ...
    resp, err := activityPubClient().Do(r)
    if err != nil {
        return err
    }
    // ...
    return nil
}
```

失败后的处理（在 `federatePost` 中）：
```go
err = makeActivityPost(app.cfg.App.Host, actor, si, activity)
if err != nil {
    log.Error("Couldn't post! %v", err)
}
```

仅记录错误日志，不进行任何重试。

### 2.2 远端返回 4xx/5xx 不被视为错误

`makeActivityPost` 的关键行为：**只要 `activityPubClient().Do(r)` 没有返回 Go 层面的 `error`，函数就返回 `nil`（即"成功"）**，完全不检查 HTTP 响应状态码。

```go
resp, err := activityPubClient().Do(r)
if err != nil {
    return err          // 只有网络层/超时错误才会走到这里
}
// ... 关闭 body、读取 body ...
return nil              // 无论 resp.StatusCode 是 200、403、410 还是 500，都走这里
```

这意味着以下远端响应**不会**被当作错误返回：

| 远端响应 | 是否被当作错误 | 说明 |
|----------|----------------|------|
| `200 OK` | 否 | 正常投递成功 |
| `202 Accepted` | 否 | Mastodon 等异步处理返回 |
| `400 Bad Request` | 否 | 请求格式问题，但调用方不知情 |
| `401 Unauthorized` | 否 | 签名问题，但调用方不知情 |
| `403 Forbidden` | 否 | 被远端拒绝，但调用方不知情 |
| `404 Not Found` | 否 | 收件箱不存在，但调用方不知情 |
| `410 Gone` | 否 | 收件箱已删除，但调用方不知情 |
| `500 Internal Server Error` | 否 | 远端服务器故障，但调用方不知情 |
| `503 Service Unavailable` | 否 | 远端暂时不可用，但调用方不知情 |

**后果**：
- 远端返回 4xx（如 410 Gone 表示实例已关停）时，调用方以为投递成功，不会触发任何重试或记录
- 远端返回 5xx（如 503 临时不可用）时，本应属于可重试的瞬态错误，但也被当作成功丢弃
- 只有 Go 的 `http.Client.Do()` 抛出 `error`（如 DNS 解析失败、连接拒绝、TLS 握手失败、15 秒超时）才会被上层捕获并记入日志

在 debug 模式下，状态码和响应体会被打印到日志（[activitypub.go:791-794](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/activitypub.go#L791-L794)），但生产环境默认不输出，且不参与任何逻辑判断。

### 2.3 HTTP 客户端超时配置

虽然没有重试，但 HTTP 客户端有超时设置：

```go
func activityPubClient() *http.Client {
    return &http.Client{
        Timeout: 15 * time.Second,
    }
}
```

- 请求超时：15 秒
- 超时后返回错误，终止本次投递

### 2.4 Follow/Unfollow 的初始延迟

在处理收到的 Follow/Unfollow 活动时，异步回复 Accept 活动有 2 秒延迟：

```go
go func() {
    // ...
    time.Sleep(2 * time.Second)
    am, err := a.Serialize()
    // ...
    err = makeActivityPost(app.cfg.App.Host, p, fullActor.Inbox, am)
    // ...
}()
```

位置：[activitypub.go:639-728](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/activitypub.go#L639-L728)

这个延迟的目的：
- 避免在处理收件箱请求的同时立即发送回复
- 给远程服务器一些准备时间
- **但这不是重试退避，只是初始延迟**

### 2.5 参考：邮件发布的延迟队列

作为对比，邮件发布（email publishing）有一个基于数据库的延迟队列机制，位于 `publishjobs` 表：

| 特性 | 说明 |
|------|------|
| 延迟时间 | 固定 15 分钟（`emailSendDelay = 15`） |
| 队列存储 | `publishjobs` 数据库表 |
| 调度周期 | 每 62 秒检查一次（`time.NewTicker(62 * time.Second)`） |
| 时间窗口 | 只执行 delay 到 delay+5 分钟之间创建的 job |

相关代码：
- 队列入队：[database.go:3306-3317](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/database.go#L3306-L3317)
- 队列消费：[jobs.go:25-42](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/jobs.go#L25-L42)
- 时间窗口查询：[database.go:3346-3370](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/database.go#L3346-L3370)

---

## 三、终态判定

### 3.1 当前状态：无终态判定

由于联邦投递没有重试机制，因此也不存在终态判定（即"什么时候停止重试"的判断）。

投递结果只有两种：
1. **成功**：HTTP 请求成功返回，无后续操作
2. **失败**：HTTP 请求失败或超时，记录错误日志，结束

失败后：
- 不会重新入队
- 不会递增重试次数
- 不会更新退避时间
- 不会标记为最终失败

### 3.2 错误类型与处理

`makeActivityPost` 可能返回的错误类型：

| 错误场景 | 是否返回 error | 处理方式 |
|----------|----------------|----------|
| JSON 序列化失败 | 是 | 直接返回错误，终止 |
| 私钥解码失败 | 是 | 直接返回错误，终止 |
| 签名失败 | 否 | 记录错误日志，但**继续发送**（无签名请求） |
| HTTP 请求失败（网络错误、超时等） | 是 | 直接返回错误，终止 |
| 响应读取失败 | 是 | 直接返回错误，终止 |
| 远端返回 4xx（400/401/403/404/410 等） | **否** | `return nil`，调用方以为成功 |
| 远端返回 5xx（500/502/503 等） | **否** | `return nil`，调用方以为成功 |

注意：
- 签名失败不会阻止请求发送，只是会发送没有签名的请求
- **远端返回任何 HTTP 状态码都不会产生 Go error**，只有网络层/传输层错误才会。这是当前实现中最大的盲区：410 Gone（实例已关停）等永久性失败和 503 Service Unavailable（临时不可用）等瞬态失败都被静默忽略

### 3.3 参考：邮件发布的终态处理

作为对比，邮件发布队列的终态处理：

```go
func runJobs(app *App, jobs []*PostJob, reqColl bool) error {
    for _, j := range jobs {
        // ...
        err = emailPost(app, p, p.Collection.ID)
        if err != nil {
            log.Error("[job #%d] Failed to email post %s", j.ID, p.ID)
            continue  // 失败不删除 job
        }
        log.Info("[job #%d] Success for post %s.", j.ID, p.ID)
        app.db.DeleteJob(j.ID)  // 成功删除 job
    }
    return nil
}
```

终态逻辑：
- **成功**：删除 job（终态）
- **失败**：不删除 job，但由于时间窗口限制（delay 到 delay+5 分钟），下次调度时也不会再执行
- **特殊情况**：文章不属于集合时删除 job（跳过）

实际上，邮件队列也没有真正的重试机制，失败的 job 会被遗留但不会再执行。

---

## 四、关键代码路径总览

### 4.1 联邦投递主流程

```
发布/更新/删除/导入/认领文章
    ↓
各入口独立的前置检查
    ├─ 发布：4项检查 ✓（入口挡未来文章）
    ├─ 更新：3项检查 ✗（未来文章直接进去）
    ├─ 删除：3项检查 → deleteFederatedPost
    ├─ 导入：2项检查 ✗（最松，未来文章直接进去）
    └─ 认领：4项检查 ✓（入口挡未来文章）
    ↓
go federatePost()  // 异步 goroutine
    ↓
federatePost() 内部兜底检查（仅2项，不含未来文章）
    ├─ 应用是否私有？
    └─ 集合可见性？
    ↓
未来文章在此处无任何防守，直接进入投递 →→→ 会被真实投出
    ↓
获取关注者列表，按收件箱分组
    ↓
遍历每个收件箱
    ↓
makeActivityPost()
    ├─ 构造 HTTP 请求
    ├─ 签名
    ├─ 发送（15 秒超时）
    ├─ 网络层错误 → return err → 上层 log.Error → 终止
    └─ 拿到响应（不管 200/4xx/5xx）→ return nil → 上层无感知
```

### 4.2 收件箱处理异步流程

```
收到 Follow/Unfollow 活动
    ↓
handleFetchCollectionInbox()
    ├─ 同步返回 200 OK
    └─ go func() {  // 异步 goroutine
            ├─ time.Sleep(2 * time.Second)  // 2 秒延迟
            └─ makeActivityPost()  // 发送 Accept
        }
```

---

## 五、总结与现状评估

### 5.1 当前实现特点

| 维度 | 现状 |
|------|------|
| **入队方式** | goroutine 异步触发，无持久化队列 |
| **出站入口** | 5 个独立入口：发布、更新、删除、导入、认领 |
| **入队前置检查** | 各入口不等：发布/认领 4 项最严，更新/删除 3 项，导入仅 2 项最松 |
| **挡未来文章** | 发布、认领 在入口处主动挡；更新、导入 入口不挡 + 内部也不挡，直接投出 |
| **federatePost 内部防守** | 仅检查实例私有 + 集合可见性，不检查未来文章 |
| **deleteFederatedPost 内部防守** | **完全没有任何检查**，入口调用即发送 |
| **重试机制** | 无，fire-and-forget |
| **退避梯度** | 无（Follow 回复有 2 秒初始延迟，非重试退避） |
| **终态判定** | 无（失败即终态，仅记录日志） |
| **HTTP 状态码** | 不检查，4xx/5xx 均视为"成功"返回 nil |
| **失败处理** | 仅网络层错误记 log.Error，4xx/5xx 静默忽略 |
| **关注记录入库** | 收件箱收 Follow → 2 秒延迟发 Accept → 入库 `remoteusers` + `remotefollows` |
| **删除取收件箱** | `GetAPFollowers` JOIN 两表按 collection_id 查，shared_inbox 优先分组去重 |
| **删除无可见性检查** | 语义合理性（撤回已发内容）+ 入库无历史记录 + 可能的实现疏漏 |

### 5.2 设计走向观察

从现有代码（尤其是邮件发布队列和删除链路的设计）可以观察到以下走向：

1. **队列模式已有先例**：邮件发布使用了基于数据库的 `publishjobs` 队列，说明项目具备队列化异步处理的基础设计模式
2. **延迟执行而非重试**：邮件队列只有固定延迟，没有重试梯度，说明当前设计更倾向于"延迟执行"而非"失败重试"
3. **时间窗口机制**：`GetJobsToRun` 的时间窗口设计（delay 到 delay+5 分钟）暗示了简单的"错过即丢弃"哲学
4. **联邦投递更轻量**：联邦投递直接使用 goroutine，没有数据库持久化，说明对可靠性要求低于邮件发布
5. **删除链路特殊处理**：Create/Update 走 `federatePost` 有完整防守，Delete 走独立的 `deleteFederatedPost` 故意省略了可见性检查，体现对"撤回已发内容"的语义优先级高于"新内容保密"
6. **设计不一致性**：删除入口有实例私有检查，但 `deleteFederatedPost` 内部没有，与 `federatePost` 的防守层次不一致，可能是实现疏漏

### 5.3 相关文件索引

| 文件 | 主要内容 |
|------|----------|
| [activitypub.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/activitypub.go) | 联邦投递核心逻辑，包含 `federatePost`、`deleteFederatedPost`、`makeActivityPost` |
| [posts.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/posts.go) | 文章发布/更新/删除/认领入口，触发联邦投递 |
| [account_import.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/account_import.go) | 文章批量导入入口，触发联邦投递 |
| [jobs.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/jobs.go) | 邮件发布队列消费逻辑 |
| [email.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/email.go) | 邮件发送逻辑，`emailSendDelay` 常量 |
| [database.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/database.go) | `publishjobs` 表的数据库操作，以及 `GetAPFollowers` 查关注者 |
| [database_activitypub.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/database_activitypub.go) | 联邦相关数据库操作（远程用户添加） |
| [schema.sql](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/schema.sql) | 数据库 schema，含 `remotefollows`、`remoteusers` 表定义 |
| [migrations/v13.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/migrations/v13.go) | `publishjobs` 表创建迁移 |
