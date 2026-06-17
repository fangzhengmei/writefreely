# 联邦投递重试与退避策略分析

## 概述

WriteFreely 的联邦投递（ActivityPub）当前采用 **fire-and-forget（发射后不管）** 模式，尚未实现复杂的重试与退避机制。投递通过 goroutine 异步触发，失败仅记录错误日志，不会自动重试。

本文档从 **入队条件**、**退避梯度**、**终态判定** 三个维度分析当前实现走向。

---

## 一、入队条件

### 1.1 触发场景

联邦投递在以下五种场景下被触发，均以 goroutine 异步方式执行：

| 场景 | 代码位置 | 说明 |
|------|----------|------|
| 发布新文章 | [posts.go:687](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/posts.go#L687-L687) | `go federatePost(app, newPost, newPost.Collection.ID, false)` |
| 更新文章 | [posts.go:805](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/posts.go#L805-L805) | `go federatePost(app, pRes, pRes.Collection.ID, true)` |
| 删除文章 | [posts.go:944](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/posts.go#L944-L944) | `go deleteFederatedPost(app, pp, collID.Int64)` |
| 导入文章 | [account_import.go:166](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/account_import.go#L166-L166) | `go federatePost(...)` |
| 认领文章 | [posts.go:1002](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/posts.go#L1002-L1002) | `go federatePost(app, pRes.Post, pRes.Post.Collection.ID, false)` |

### 1.2 前置检查条件

#### 触发点检查（posts.go）

在调用 `federatePost` 之前，需要满足以下条件：

```go
if newPost.Collection != nil {
    if !app.cfg.App.Private && app.cfg.App.Federation && !newPost.Created.After(time.Now()) {
        go federatePost(app, newPost, newPost.Collection.ID, false)
    }
}
```

条件拆解：
1. **文章属于集合**：`newPost.Collection != nil` — 只有集合（博客）中的文章才会联邦投递
2. **应用非私有**：`!app.cfg.App.Private` — 私有实例不进行联邦投递
3. **联邦功能启用**：`app.cfg.App.Federation` — 配置中开启了联邦功能
4. **非未来文章**：`!newPost.Created.After(time.Now())` — 创建时间晚于当前时间的文章不立即投递

#### 认领文章的触发点检查（posts.go:995-1003）

认领文章（Claim）是独立于发布/更新的第五个触发场景。用户将一篇散落在个人名下、尚未归属任何集合的文章"认领"到某个集合后，该文章变为集合内容，需要向联邦推送 Create 活动。

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

条件拆解：
1. **认领成功**：`pRes.Code == http.StatusOK` — 只有认领操作成功的文章才触发投递，失败或冲突的跳过
2. **应用非私有 + 联邦启用**：与发布场景相同的全局检查
3. **非未来文章**：与发布场景相同，未来文章不立即投递
4. **isUpdate = false**：认领相当于首次发布，使用 Create 活动而非 Update

#### 函数内部检查（federatePost）

进入 `federatePost` 函数后，还会进行两层检查：

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

条件拆解：
1. **应用非私有**（重复检查，防御性编程）
2. **集合可见性**：私有集合（`CollPrivate`）和受保护集合（`CollProtected`）的文章不投递

### 1.3 投递目标分组逻辑

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

### 1.4 @提及用户的单独投递

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
发布/更新/删除/认领文章
    ↓
posts.go 中检查触发条件
    ↓
go federatePost()  // 异步 goroutine
    ↓
federatePost() / deleteFederatedPost()
    ├─ 检查应用是否私有
    ├─ 检查集合可见性
    ├─ 获取关注者列表
    ├─ 按收件箱分组（共享收件箱优先）
    └─ 遍历每个收件箱
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
| **入队条件** | 5 个触发场景 + 4 项前置检查 |
| **重试机制** | 无，fire-and-forget |
| **退避梯度** | 无（Follow 回复有 2 秒初始延迟，非重试退避） |
| **终态判定** | 无（失败即终态，仅记录日志） |
| **HTTP 状态码** | 不检查，4xx/5xx 均视为"成功"返回 nil |
| **失败处理** | 仅网络层错误记 log.Error，4xx/5xx 静默忽略 |

### 5.2 设计走向观察

从现有代码（尤其是邮件发布队列的设计）可以观察到以下走向：

1. **队列模式已有先例**：邮件发布使用了基于数据库的 `publishjobs` 队列，说明项目具备队列化异步处理的基础设计模式
2. **延迟执行而非重试**：邮件队列只有固定延迟，没有重试梯度，说明当前设计更倾向于"延迟执行"而非"失败重试"
3. **时间窗口机制**：`GetJobsToRun` 的时间窗口设计（delay 到 delay+5 分钟）暗示了简单的"错过即丢弃"哲学
4. **联邦投递更轻量**：联邦投递直接使用 goroutine，没有数据库持久化，说明对可靠性要求低于邮件发布

### 5.3 相关文件索引

| 文件 | 主要内容 |
|------|----------|
| [activitypub.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/activitypub.go) | 联邦投递核心逻辑，包含 `federatePost`、`deleteFederatedPost`、`makeActivityPost` |
| [posts.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/posts.go) | 文章发布/更新/删除入口，触发联邦投递 |
| [jobs.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/jobs.go) | 邮件发布队列消费逻辑 |
| [database.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/database.go) | `publishjobs` 表的数据库操作（InsertJob、GetJobsToRun、DeleteJob 等） |
| [email.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/email.go) | 邮件发送逻辑，`emailSendDelay` 常量 |
| [database_activitypub.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/database_activitypub.go) | 联邦相关数据库操作（远程用户添加） |
| [migrations/v13.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-writefreely/migrations/v13.go) | `publishjobs` 表创建迁移 |
