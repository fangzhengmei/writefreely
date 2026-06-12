# ActivityPub 联邦交互分析

## 概述

WriteFreely 的 ActivityPub 联邦功能主要由以下核心文件实现：

- [activitypub.go](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go) - ActivityPub 核心处理逻辑
- [database_activitypub.go](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/database_activitypub.go) - 联邦相关数据库操作
- [webfinger.go](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/webfinger.go) - WebFinger 协议实现
- [routes.go](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/routes.go) - 路由配置（L155-L158）

---

## 1. 远程关注流程（Follow / Accept）

### 1.1 Follow 活动接收

入口函数：[handleFetchCollectionInbox](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L317-L738)

当远程用户发起关注请求时，Activity 流通过 POST `/api/collections/{alias}/inbox` 送达。

**处理流程：**

```
Follow 活动到达 → JSON 解码 → streams.Resolver 分发 → FollowCallback
```

**FollowCallback 关键步骤** [activitypub.go:L428-L476](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L428-L476):

1. **Activity ID 处理**：
   - 从 Follow 活动提取 ID
   - 生成 Accept 活动的唯一 ID：`{accountRoot}#accept-{random20chars}`

2. **构建 Accept 活动**：
   ```go
   a := streams.NewAccept()
   a.AppendObject(f.Raw())    // 将 Follow 作为 Accept 的 object
   a.AppendActor(obj)         // 被关注者 Actor IRI
   ```

3. **Actor 信息获取**：
   - 调用 `getActor(app, to.String())` 获取关注者的完整信息
   - 优先从本地数据库查找，不存在则远程请求

4. **同步响应**：
   - 立即返回 HTTP 200 OK 给远程服务器
   - 实际的 Accept 活动通过异步 goroutine 发送

### 1.2 Accept 活动异步发送

在 [activitypub.go:L639-L728](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L639-L728) 的 goroutine 中：

1. **延迟发送**：`time.Sleep(2 * time.Second)` - 避免与同步响应冲突

2. **序列化 Accept**：
   ```go
   am, err := a.Serialize()
   am["@context"] = []string{activitystreams.Namespace}
   ```

3. **HTTP POST 发送**：
   - 调用 `makeActivityPost()` 发送到关注者的 inbox
   - 携带 HTTP 签名（Digest + Signature 头）

4. **数据库持久化**：
   - 开启事务
   - 若远程用户不存在，插入 `remoteusers` 和 `remoteuserkeys`
   - 插入关注关系到 `remotefollows` 表
   - 提交事务

### 1.3 Unfollow 处理

通过 `UndoCallback` 处理 `Undo:Follow` 活动 [activitypub.go:L515-L537](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L515-L537):

1. 识别 Undo 活动的 object 为 Follow
2. 获取取消关注的 Actor
3. 异步发送 Accept 响应
4. 从 `remotefollows` 表删除关系：
   ```sql
   DELETE FROM remotefollows 
   WHERE collection_id = ? AND remote_user_id = (
       SELECT id FROM remoteusers WHERE actor_id = ?
   )
   ```

---

## 2. 活动收发机制

### 2.1 Inbox 接收

**路由**：`POST /api/collections/{alias}/inbox` [routes.go:L155](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/routes.go#L155)

**支持的活动类型**：

| 类型 | 回调函数 | 处理逻辑 |
|------|---------|---------|
| `Follow` | `FollowCallback` | 建立关注关系 |
| `Undo` | `UndoCallback` | 处理取消关注/取消点赞 |
| `Like` | `LikeCallback` | 记录点赞到 `remote_likes` |
| `Delete` | `DeleteCallback` | 仅返回 200 OK（占位实现） |

**同步 vs 异步处理**：

- **同步处理**：`Like` / `Unlike` 活动在请求线程中完成数据库操作
- **异步处理**：`Follow` / `Unfollow` 的 Accept 响应和数据库写入在 goroutine 中完成

### 2.2 Outbox 发送

**路由**：`GET /api/collections/{alias}/outbox` [routes.go:L156](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/routes.go#L156)

**处理函数**：[handleFetchCollectionOutbox](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L154-L215)

**分页逻辑**：
- 无 `page` 参数：返回 OrderedCollection 摘要（仅总数）
- 有 `page` 参数：返回 OrderedCollectionPage，包含具体 Create 活动列表

### 2.3 出站活动分发（federatePost）

函数：[federatePost](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L896-L999)

**触发时机**：
- 发布新文章：`go federatePost(app, newPost, ..., false)` [posts.go:L687](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/posts.go#L687)
- 更新文章：`go federatePost(app, pRes, ..., true)` [posts.go:L805](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/posts.go#L805)

**分发逻辑**：

1. **前置检查**：
   - 私有实例不联邦
   - 私有/保护博客不联邦

2. **获取关注者列表**：
   ```go
   followers, err := app.db.GetAPFollowers(&p.Collection.Collection)
   ```

3. **收件箱去重**：
   - 按 shared_inbox 分组，同一实例的多个关注者共享一次投递
   - 无 shared_inbox 则使用个人 inbox

4. **构造 Activity**：
   - 新文章：`Create` 活动
   - 更新文章：`Update` 活动，设置 `Updated` 字段

5. **投递**：
   - 对每个收件箱分组，设置 CC 字段为该组所有关注者
   - 调用 `makeActivityPost()` 发送

6. **@提及处理**：
   - 遍历文章中的 `Mention` 标签
   - 对每个被提及用户单独发送活动到其 inbox

### 2.4 删除活动分发（deleteFederatedPost）

函数：[deleteFederatedPost](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L851-L894)

**特殊处理**：
- Delete 活动 ID 追加 `#Delete` 后缀以兼容 Pleroma
  ```go
  da.ID += "#Delete"  // https://git.pleroma.social/pleroma/pleroma/issues/1481
  ```

### 2.5 HTTP 签名与请求

函数：[makeActivityPost](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L740-L797)

**请求构造**：
```go
r, _ := http.NewRequest("POST", url, bytes.NewBuffer(b))
r.Header.Add("Content-Type", "application/activity+json")
r.Header.Set("User-Agent", ServerUserAgent(hostName))
r.Header.Add("Digest", "SHA-256="+base64.StdEncoding.EncodeToString(h.Sum(nil)))
```

**HTTP 签名**：
- 签名算法：`RSASHA256`
- 签名字段：`(request-target)`, `date`, `host`, `digest`
- 密钥来源：`collectionkeys` 表中的私钥

**客户端配置**：
- 超时：15 秒
- 实现：[activityPubClient](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L106-L110)

### 2.6 同步返回与异步处理的深度解析

本小节从代码层面**逐行拆解** Follow 请求从入站到返回的完整路径，解释"为什么先回 200、再异步处理"的每一个设计决策。

---

#### 2.6.1 同步阶段：从请求到 200 OK 的完整代码路径

入口函数：[handleFetchCollectionInbox](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L317-L738)

整个处理流程使用 `streams.Resolver` 回调机制驱动。代码中有一个关键标志变量 `responseWritten`，贯穿整个函数：

```go
var responseWritten bool  // L354：标记 HTTP 响应是否已写入
```

这个变量的作用是：回调可以选择"提前写响应"，函数末尾通过它判断要不要补写兜底响应。

##### 阶段一：JSON 解码（L360-L368）

```go
var m map[string]any
if err := json.NewDecoder(tee).Decode(&m); err != nil {
    log.Error("Failed decoding JSON: %v", err)
    return err  // ← 注意：只有 JSON 解码失败才真正返回错误
}
```

**注意**：这是整个函数里**唯一会返回非 200 错误**的地方。JSON 都解析不了，说明请求完全非法，返回错误让对方知道。

##### 阶段二：streams.Resolver 分发（L378-L548）

`res.Deserialize(m)` 根据 Activity 的 `type` 字段分发到对应的回调：

```go
res := &streams.Resolver{
    LikeCallback:   func(l *streams.Like) error { ... },
    FollowCallback: func(f *streams.Follow) error { ... },
    UndoCallback:   func(u *streams.Undo) error { ... },
    DeleteCallback: func(d *streams.Delete) error { ... },
}
if err := res.Deserialize(m); err != nil {
    // ...
}
```

每个回调的返回值是 `error`，它会被 `Deserialize` 原样返回。

##### 阶段三：FollowCallback 内部的 9 个步骤

Follow 回调 [activitypub.go:L428-L476](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L428-L476) 内部的精确执行顺序：

| 步骤 | 代码行 | 操作 | 性质 |
|------|--------|------|------|
| 1 | L439-L449 | 从 Follow 提取 ID，生成 Accept 的 ID | 纯内存操作 |
| 2 | L450 | `a.AppendObject(f.Raw())` - Follow 作为 Accept 的 object | 纯内存操作 |
| 3 | L451 | `_, to = f.GetActor(0)` - 取出关注者 IRI | 纯内存操作 |
| 4 | L452-L463 | 取出被关注者 IRI（先试 object IRI，不行试 object 本身） | 纯内存操作 |
| 5 | L464 | `a.AppendActor(obj)` - 设置 Accept 的 actor | 纯内存操作 |
| 6 | L467-L473 | 校验 `to` 不为空，然后 `getActor(app, to.String())` | **可能阻塞（网络请求）** |
| 7 | L474 | `responseWritten = true` | 设置标志位 |
| 8 | L475 | `impart.RenderActivityJSON(w, m, http.StatusOK)` | **写 HTTP 响应** |
| 9 | L476 | `return nil` | 回调返回成功 |

**关键观察**：第 6 步 `getActor` 在同步路径上。如果本地没有该用户缓存，这里会发起 2 次 HTTP GET，最多阻塞 30 秒。

**第 8 步的双重作用**：`impart.RenderActivityJSON` 会真正写入 `ResponseWriter`，HTTP 状态码 200，body 是原始 Activity JSON。远程服务器在此刻就收到了 200 OK，可以关闭连接了。

##### 阶段四：回调后的同步处理（L562-L637）

`res.Deserialize` 返回 nil 后，函数继续往下执行。这一段处理**同步活动**：

```go
if isLike {
    // 同步写入 remote_likes
    // ...
    impart.RenderActivityJSON(w, "", http.StatusOK)
    return nil
} else if isUnlike {
    // 同步删除 remote_likes
    // ...
    impart.RenderActivityJSON(w, "", http.StatusOK)
    return nil
}
```

Like 和 Unlike 的数据库操作在**同步路径**完成，完成后才返回 200。这是因为它们只涉及本地数据库写，速度可控。

##### 阶段五：启动 goroutine（L639）

```go
go func() {
    // ... 异步发送 Accept + 写入数据库 ...
}()
```

**只有 Follow 和 Unfollow 会走到这里**。Like/Unlike 在阶段四已经 `return nil` 了。

##### 阶段六：兜底响应（L730-L735）

```go
if !responseWritten {
    impart.RenderActivityJSON(w, "", http.StatusOK)
}
return nil
```

如果没有任何回调写过响应（比如未知的活动类型），在这里兜底返回 200 OK。

---

#### 2.6.2 反序列化错误为什么也返回 200？

代码 [activitypub.go:L549-L560](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L549-L560):

```go
if err := res.Deserialize(m); err != nil {
    log.Error("Unable to resolve Activity: %v", err)
    if t, ok := m["type"]; ok {
        log.Error("Unhandled activity type: %v", t)
    }
    impart.RenderActivityJSON(w, "", http.StatusOK)  // ← 仍然 200
    return nil
}
```

**即使 Deserialize 出错，也返回 200 OK，而且 body 是空字符串。** 这有三个层面的原因：

**原因一：协议层面——ActivityPub 的 inbox 语义**

ActivityPub 规范 [Section 7.1 Inbox](https://www.w3.org/TR/activitypub/#inbox) 规定：

> The server MUST be capable of processing activities, or else return a 5xx type error.

但"处理"不等于"执行"。收到一个不认识的活动类型，不代表投递失败——活动确实送达了，只是本服务器不处理它。返回 2xx 表示"我收到了"，至于"我处理不处理"是另一回事。

**原因二：工程层面——避免重试风暴**

如果返回 4xx 或 5xx，发送方会认为投递失败并重试。对于**未知活动类型**，重试是毫无意义的——再发 100 次还是不认识。但重试会：
- 浪费双方的带宽和计算资源
- 如果批量出现未知类型（比如新协议扩展），重试流量可能把服务打满
- 远程实例的投递队列会被永远堆积（因为永远"失败"）

返回 200 是"**收到了，我不会处理，但别再发了**"的信号。

**原因三：安全层面——不暴露内部状态**

返回特定的错误码可能泄露信息：
- 404 可能暴露"这个博客不存在"
- 401 可能暴露"这个博客需要认证"
- 500 可能暴露"我们出 bug 了"

对所有无法处理的入站活动统一返回 200 OK，是一种最小信息披露原则。

> **对比**：JSON 解码失败（阶段一）是真正的协议级别错误——连基本结构都不对，请求本身无效，所以返回错误。而 Deserialize 失败意味着"结构是合法的 Activity，但我们不认识这个类型"，所以返回 200。

---

#### 2.6.3 为什么 Accept 发送和关注落库都放进 goroutine？

Follow 处理的**两个核心副作用**——发送 Accept 活动和写入关注关系——都被放进了同一个 goroutine [activitypub.go:L639-L728](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L639-L728)。这不是随意的安排，每个决策都有依据。

##### 决策一：Accept 发送必须异步

Accept 活动需要 POST 到远程实例的 inbox，这是一个**出站网络请求**，耗时完全不可控：
- 快的话：几十毫秒（同区域、低延迟）
- 慢的话：几秒到十几秒（网络拥塞、对方实例负载高）
- 最坏：15 秒超时（`activityPubClient` 的超时设置）

如果同步等待 Accept 发送完成，Follow 请求的总耗时就完全被远程实例的速度绑架了。异步发送把"接收 Follow"和"回应 Accept"解耦成两个独立操作。

##### 决策二：数据库写入也放进同一个 goroutine

关注落库（`remotefollows` 表的 INSERT）理论上可以同步做——本地数据库操作很快。但代码选择把它也放进 goroutine，原因有三：

**1. 与 Accept 发送的事务一致性**

如果 Accept 已经发送出去了，但数据库写入失败了怎么办？远程用户会以为关注成功了，但本地没有记录。后续该用户发的活动也不会被投递到本地——这是一种静默的不一致。

把两者放在同一个 goroutine 里，**Accept 先发，DB 后写**：
- Accept 成功 + DB 成功 → 正常
- Accept 成功 + DB 失败 → 不一致（但概率低）
- Accept 失败 + DB 不写 → 一致（用户没收到 Accept，以为还在待处理）

> 注意：代码里的顺序是"先发 Accept，再写 DB"，不是"写 DB 成功再发 Accept"。这意味着如果 Accept 发送成功但 DB 写入失败，远程端会显示已关注，但本地没有记录——这是一个已知的设计权衡。

**2. 避免阻塞同步请求**

虽然数据库写入通常很快，但在高并发或数据库慢查询时，单次 INSERT 也可能阻塞。特别是 Follow 涉及三张表的写入（`remoteusers`、`remoteuserkeys`、`remotefollows`），在一个事务中完成。把它放进异步路径，确保 HTTP 响应速度不受数据库瞬时负载影响。

**3. 为未来的批处理留空间**

如果将来要实现关注关系的批量写入或队列化，所有异步操作都在 goroutine 里，更容易重构。

##### 决策三：2 秒延迟的作用

```go
time.Sleep(2 * time.Second)  // L647
```

这个固定延迟有两个实际作用：

**1. 确保时序正确性**

远程实例发送 Follow 请求后，需要时间处理 200 OK 响应并准备接收 Accept。如果 Accept 到达得太快，远程端可能还没来得及记录 Follow 活动，收到 Accept 时找不到对应的 Follow，就会丢弃。

2 秒是一个经验值，给远程实例足够的处理窗口。

**2. 流量削峰**

如果瞬间有大量 Follow 请求（比如被大 V 转发带来的关注潮），2 秒延迟可以把 Accept 发送的峰值摊平，降低对远程实例和本地数据库的瞬时压力。

> 代价：正常用户关注后需要等 2 秒才能看到"已关注"状态。对社交媒体来说，这是可接受的延迟。

##### 决策四：Like 为什么不同步返回 200 再异步写 DB？

对比 Follow 和 Like 的处理模式可以发现一个有趣的差异：

| 操作 | 写响应的时机 | 数据库写入时机 |
|------|-------------|---------------|
| Follow | 回调内（getActor 之后） | 异步 goroutine |
| Like | DB 写入完成之后 | 同步（在响应之前） |

Like 在 [activitypub.go:L563-L602](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L563-L602) 的处理：
```go
if isLike {
    t, err := app.db.Begin()
    // ... 插入 remote_likes ...
    err = t.Commit()
    impart.RenderActivityJSON(w, "", http.StatusOK)  // ← DB 写完才返回
    return nil
}
```

**为什么 Like 不也先返回 200 再异步写？**

可能的原因：
1. Like 不需要向远程回发任何活动（没有 Accept/Reject），处理链路短
2. Like 只涉及一张表的 INSERT，数据库操作快且确定
3. 设计上的不一致——这可能是历史遗留，两种模式出自不同时期的代码

实际上 Like 的 `getActor` 也可能触发远程 HTTP 请求（最多 30 秒阻塞），所以 Like 的同步处理也并不安全。这是一个**潜在的性能隐患**。

---

#### 2.6.4 同步路径上的 getActor：必要的恶？

前面提到 `getActor` 在同步路径上执行，可能阻塞 30 秒。这是一个明显的设计问题，但不是 bug——而是刻意的权衡。

**为什么不把 getActor 也移到 goroutine 里？**

```go
// 如果改成这样：
FollowCallback: func(f *streams.Follow) error {
    isFollow = true
    _, to = f.GetActor(0)
    // 不调 getActor，直接返回
    return impart.RenderActivityJSON(w, m, http.StatusOK)
}
// 然后 goroutine 里再调 getActor
go func() {
    fullActor, _, err := getActor(app, to.String())
    // ... 发 Accept ...
}()
```

这样同步阶段就只剩纯内存操作，几毫秒就能返回 200。但代码没这么做，可能的顾虑：

1. **验证成本**：如果 `getActor` 在 goroutine 里失败了（比如 actor 不存在），没有补救手段——远程端会一直等 Accept，但永远等不到。
2. **信息完整性**：`remoteUser` 变量（本地缓存的用户 ID）也需要在同步阶段拿到，否则 goroutine 里无法判断是"更新现有用户"还是"插入新用户"。
3. **简单性优先**：先把信息拿齐再返回，逻辑更直观，出错了至少能在日志里看到。

**但这确实是一个设计缺陷**：在最坏情况下（远程实例完全不可达），每个 Follow 请求会阻塞 30 秒，如果有 100 个并发 Follow，请求处理线程会被全部占满。

---

#### 2.6.5 Unfollow 的同步/异步差异

Unfollow（`Undo:Follow`） [activitypub.go:L515-L537](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L515-L537) 的模式与 Follow 基本一致，但有一个关键区别：

```go
// Follow 用 getActor（可能远程请求）
fullActor, remoteUser, err = getActor(app, to.String())

// Unfollow 用 getRemoteUser（纯本地）
remoteUser, err = getRemoteUser(app, to.String())
```

Unfollow 选择只查本地数据库，不做远程请求：

| 场景 | 行为 | 结果 |
|------|------|------|
| 本地有记录 | 异步发 Accept + 删 DB | 正常取消关注 |
| 本地无记录 | 回调返回 error → 最终仍返回 200 OK | 静默忽略 |

**设计理由**：
1. 既然用户之前关注过，本地应该有记录（除非记录被清理了）
2. 取消关注是一个"删除"操作，宁可不处理也不要误处理
3. 不需要完整的 Actor 信息来发 Accept——`to`（actor IRI）就够了，因为 inbox 地址可以从 IRI 推断？不，实际上还是需要，但代码里 Unfollow 的 goroutine 也用 `fullActor.Inbox`，如果 `fullActor` 是从 `remoteUser.AsPerson()` 构造的，那 inbox 来自本地缓存

> 注意：Unfollow 的 goroutine 里发 Accept 用的是 `fullActor.Inbox`，而 `fullActor` 是通过 `remoteUser.AsPerson()` 从本地记录构造的。如果本地记录的 inbox 地址过时了，Accept 会发错地方。

---

## 3. Actor 记录管理

### 3.1 数据结构

**RemoteUser** [activitypub.go:L67-L88](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L67-L88):
```go
type RemoteUser struct {
    ID          int64
    ActorID     string    // ActivityPub Actor IRI
    Inbox       string    // 个人收件箱
    SharedInbox string    // 共享收件箱
    URL         string    // 个人主页 URL
    Handle      string    // @user@domain 格式
    Created     time.Time // 关注时间
}
```

### 3.2 数据库表结构

**remoteusers** 表 [sqlite.sql:L170-L176](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/sqlite.sql#L170-L176):
| 字段 | 类型 | 说明 |
|------|------|------|
| id | INTEGER | 主键，自增 |
| actor_id | TEXT | 唯一约束，Actor IRI |
| inbox | TEXT | 收件箱 URL |
| shared_inbox | TEXT | 共享收件箱 URL |
| url | TEXT | 主页 URL（可空） |
| handle | TEXT | @user@domain（可空） |

**remoteuserkeys** 表 [sqlite.sql:L157-L162](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/sqlite.sql#L157-L162):
| 字段 | 类型 | 说明 |
|------|------|------|
| id | TEXT | 公钥 ID（如 `acct#main-key`） |
| remote_user_id | INTEGER | 外键到 remoteusers |
| public_key | BLOB | PEM 格式公钥 |

**remotefollows** 表 [sqlite.sql:L144-L149](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/sqlite.sql#L144-L149):
| 字段 | 类型 | 说明 |
|------|------|------|
| collection_id | INTEGER | 被关注的博客 ID |
| remote_user_id | INTEGER | 关注者 ID |
| created | DATETIME | 关注时间 |
| PRIMARY KEY | (collection_id, remote_user_id) | 联合主键 |

**remote_likes** 表 [migrations/v16.go:L20-L25](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/migrations/v16.go#L20-L25):
| 字段 | 类型 | 说明 |
|------|------|------|
| post_id | CHAR(16) | 文章 ID |
| remote_user_id | INTEGER | 点赞者 ID |
| created | DATETIME | 点赞时间 |

### 3.3 找出 Actor 的多种方法

WriteFreely 提供了三种查询远程用户的入口，分别通过不同的标识查找：

| 方法 | 参数 | 查找字段 | 返回类型 | 文件 |
|------|------|---------|---------|------|
| `getRemoteUser` | actor IRI | `actor_id` | *RemoteUser | [activitypub.go:L1001-L1017](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L1001-L1017) |
| `getRemoteUserFromHandle` | @user@domain | `handle` | *RemoteUser | [activitypub.go:L1021-L1034](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L1021-L1034) |
| `getRemoteUserFromURL` | 主页 URL | `url` | *RemoteUser | [activitypub.go:L1037-L1051](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L1037-L1051) |

三者都是纯本地数据库查询，不涉及网络请求。

#### 3.3.1 找不到记录时的精确返回语义

这三个函数在找不到记录时的返回值**各不相同**，直接影响上游调用者的错误处理逻辑：

| 函数 | `sql.ErrNoRows` 时的返回 | 错误类型 | 具体值 |
|------|------------------------|---------|--------|
| `getRemoteUser` | `nil, impart.HTTPError{Status: 404, Message: "No remote user with that ID."}` | 值类型（非指针） | 直接构造 |
| `getRemoteUserFromHandle` | `nil, ErrRemoteUserNotFound` | 值类型全局变量 | `impart.HTTPError{Status: 404, Message: "Remote user not found."}` |
| `getRemoteUserFromURL` | `nil, ErrRemoteUserNotFound` | 同上 | 同上 |

**精确代码**：

`getRemoteUser` [activitypub.go:L1005-L1007](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L1005-L1007):
```go
switch {
case err == sql.ErrNoRows:
    return nil, impart.HTTPError{http.StatusNotFound, "No remote user with that ID."}
```

`getRemoteUserFromHandle` / `getRemoteUserFromURL` [activitypub.go:L1026-L1027](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L1026-L1027):
```go
case err == sql.ErrNoRows:
    return nil, ErrRemoteUserNotFound
```

其中 `ErrRemoteUserNotFound` 定义于 [errors.go:L52](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/errors.go#L52):
```go
ErrRemoteUserNotFound = impart.HTTPError{http.StatusNotFound, "Remote user not found."}
```

#### 3.3.2 为什么返回语义不统一？

`getRemoteUser` 用内联构造而其他两个用 `ErrRemoteUserNotFound`，这是**历史遗留的不一致**，但实际影响不大，因为上游 `getActor` 只通过**类型断言 + Status 码**来判断，不比较 Message：

```go
if iErr, ok := err.(impart.HTTPError); ok {
    if iErr.Status == http.StatusNotFound {
        // 本地没有缓存，转为远程请求
    }
}
```

所以只要 Status 是 404，语义就是"本地不存在"，后续就会触发远程抓取流程。

#### 3.3.3 `getActor` 对 404 的特殊处理

`getActor` 是唯一会"**吃掉 404 错误**"的函数：当 `getRemoteUser` 返回 404 时，它**不向上返回错误**，而是转入远程请求分支。只有以下情况才会把错误向上抛出：

1. 错误不是 `impart.HTTPError` 类型（如数据库连接错误）
2. 是 `impart.HTTPError` 但 Status 不是 404（几乎不可能发生）
3. 远程请求过程中发生任何错误（网络超时、解析失败等）→ 包装为 500 错误返回

**关键返回值含义**：`getActor` 返回 `(*Person, *RemoteUser, error)` 的三元组，其中第二返回值 `*RemoteUser` 是否为 nil 有明确含义：
- 非 nil：本地缓存命中，数据来自 `remoteusers` 表
- nil：远程请求获取成功，**尚未落库**，需要调用者判断是否要持久化

### 3.4 Actor 查询流程（getActor）

函数：[getActor](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L1053-L1096)

`getActor` 是一个**组合函数**：先查本地缓存，查不到则远程抓取。它返回两个值：
- `*activitystreams.Person` - 完整的 Actor 数据对象
- `*RemoteUser` - 本地数据库记录（若存在）

**详细流程**：

```
getActor(actorIRI)
    ↓
第一步：本地查询 getRemoteUser(actorIRI)
    ├─ 命中 → actor = remoteUser.AsPerson()，直接返回
    └─ 未命中 → 进入远程获取流程
        ↓
第二步：第一次 resolveIRI(actorIRI)
    ├─ HTTP GET + 签名请求 Actor 端点
    ├─ unmarshalActor 解析为 baseActor（基础信息）
    └─ 失败 → 返回 500 错误
        ↓
第三步：第二次 resolveIRI(baseActor.PublicKey.Owner)
    ├─ 用 publicKey.owner 字段再次请求
    ├─ 获取完整的 Actor 信息（含完整公钥）
    └─ 失败 → 返回 500 错误
        ↓
返回 (fullActorPerson, nil, nil)
```

> **为什么要两次请求？** 某些 ActivityPub 实现（如 Mastodon）首次返回的 Actor 可能不包含完整公钥，需要通过 `publicKey.owner` 指向的真实 Actor 端点再次获取。这是为了兼容多种联邦实例的差异。

**只读性质**：`getActor` **本身不落库**，它只负责"获取数据"。是否持久化由调用方决定。

### 3.5 远程用户记录的完善与更新

`remoteusers` 表的记录可能处于不完整状态（历史版本遗留、部分字段缺失等）。系统在多个时机尝试补全记录：

#### 3.5.1 Handle 补全

场景：通过 actor_id 能查到记录，但 `handle` 字段为空（老版本数据）。

触发位置：[GetProfileURLFromHandle](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L1117-L1124)

```go
remoteUser, err := getRemoteUserFromHandle(app, handle)
if err != nil {
    actorIRI = RemoteLookup(handle)              // WebFinger 查得 actor IRI
    _, errRemoteUser := getRemoteUser(app, actorIRI)
    if errRemoteUser == nil {
        // 记录存在但 handle 为空 → 更新 handle
        app.db.Exec("UPDATE remoteusers SET handle = ? WHERE actor_id = ?", handle, actorIRI)
    }
}
```

#### 3.5.2 URL 补全

场景：记录存在但 `url`（个人主页 URL）字段为空。

触发位置：[GetProfileURLFromHandle](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L1142-L1154)

```go
if remoteUser.URL == "" {
    newRemoteActor, err := activityserve.NewRemoteActor(remoteUser.ActorID)
    if err == nil {
        app.db.Exec("UPDATE remoteusers SET url = ? WHERE actor_id = ?",
            newRemoteActor.URL(), remoteUser.ActorID)
    }
}
```

#### 3.5.3 新建完整记录

当本地完全没有该用户记录时，使用 `activityserve.NewRemoteActor()` 拉取完整信息后插入：

**方式一**：通过 handle 查找时（完整字段）[activitypub.go:L1126-L1140](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L1126-L1140)
```sql
INSERT INTO remoteusers (actor_id, inbox, shared_inbox, url, handle) VALUES(?, ?, ?, ?, ?)
```

**方式二**：关注/点赞时（通过 `apAddRemoteUser`，含公钥）[database_activitypub.go:L23-L37](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/database_activitypub.go#L23-L37)
```sql
INSERT INTO remoteusers (actor_id, inbox, shared_inbox, url) VALUES (?, ?, ?, ?)
INSERT INTO remoteuserkeys (id, remote_user_id, public_key) VALUES (?, ?, ?)
```

> **注意**：两种插入方式字段不完全一致。`apAddRemoteUser` 会同时插入公钥（`remoteuserkeys` 表），而 `GetProfileURLFromHandle` 路径只插入 `remoteusers`。

### 3.6 查询 Actor 时：只读 vs 落库的区别

这是一个关键的设计区别：**`getActor` 只是读取，落库发生在实际需要建立关系时**。

#### 3.6.1 仅读取（不落库）的场景

`getActor` 本身永远不落库。调用它但不触发写入的情况：

- **Follow 回调中的预获取**：`FollowCallback` 调用 `getActor` 仅为了获取 inbox 地址和公钥信息，用于后续构建 Accept 活动。此时**不落库**。
- **Like 回调中的预获取**：`LikeCallback` 调用 `getActor` 同理，仅为了获取用户信息用于校验，此时**不落库**。

#### 3.6.2 需要落库的场景

落库是由**业务动作**触发的，发生在 `getActor` 调用之后：

| 场景 | 落库函数 | 时机 | 位置 |
|------|---------|------|------|
| 点赞 | `apAddRemoteUser` | 同步处理 Like 时，插入点赞记录前 | [activitypub.go:L570-L575](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L570-L575) |
| 关注 | 直接 SQL INSERT | 异步 goroutine 中，插入关注关系前 | [activitypub.go:L673-L703](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L673-L703) |

**点赞流程中的落库逻辑**：
```go
if remoteUser != nil {
    remoteUserID = remoteUser.ID     // 本地有记录，直接用 ID
} else {
    remoteUserID, err = apAddRemoteUser(app, t, fullActor)  // 没记录，先落库
}
// 然后才插入 remote_likes
```

**设计思路**：
- `getActor` 保持纯粹的"获取"语义，单一职责
- 落库与业务绑定（点赞需要用户 ID 作为外键，关注也一样）
- 避免"为了缓存而缓存"，只在真正需要时才写入数据库

### 3.7 本地 Actor 构造

函数：[PersonObject](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/collections.go#L330-L359)

```go
func (c *Collection) PersonObject(ids ...int64) *activitystreams.Person {
    p := activitystreams.NewPerson(accountRoot)
    p.PreferredUsername = c.Alias
    p.Name = c.DisplayTitle()
    p.Summary = c.Description
    // ...
    pub, priv := c.db.GetAPActorKeys(collID)  // 自动生成密钥
    p.AddPubKey(pub)
    p.SetPrivKey(priv)
    return p
}
```

**密钥自动生成**：[GetAPActorKeys](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/database.go#L2717-L2735)
- 查询 `collectionkeys` 表
- 不存在则调用 `activitypub.GenerateKeys()` 生成新密钥对
- 自动插入数据库

### 3.8 远程 Handle 的处理方式

Handle 格式：`@username@domain.tld`（或无前导 `@`）

WriteFreely 通过 `GetProfileURLFromHandle` 函数处理 handle 到 profile URL 的解析，这是一个典型的"缓存优先 + 回源补全"模式。

#### 3.8.1 处理流程全景

函数：[GetProfileURLFromHandle](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L1098-L1159)

```
GetProfileURLFromHandle(handle)
    │
    ├─ 非联邦实例检查（silobridge）
    │     └─ 匹配 → 直接返回第三方平台 URL
    │
    ├─ 第一步：按 handle 查本地库 getRemoteUserFromHandle()
    │     ├─ 命中 → 检查 URL 字段
    │     │     ├─ URL 非空 → 直接返回 URL
    │     │     └─ URL 为空 → 远程拉取补全 URL，更新数据库
    │     └─ 未命中 → 进入第二步
    │
    ├─ 第二步：WebFinger 远程解析 RemoteLookup(handle)
    │     └─ 得到 actor IRI
    │
    ├─ 第三步：按 actor IRI 查本地库 getRemoteUser()
    │     ├─ 命中 → 说明是老数据（有 actor_id 无 handle）
    │     │     └─ 更新 handle 字段（补全）
    │     └─ 未命中 → 全新用户
    │
    └─ 第四步：全新用户 → activityserve.NewRemoteActor() 拉取
          └─ 完整插入 remoteusers 表（含 inbox/shared_inbox/url/handle）
```

#### 3.8.2 核心设计特点

1. **三级回退查找**：handle → actor_id → 远程拉取，层层回退
2. **惰性补全**：记录可能是不完整的，在使用过程中逐步补全字段
3. **失败容忍**：handle 更新失败、URL 补全失败都只记日志，不影响主流程
4. **双路径插入**：
   - 通过 handle 发现的用户：插入 `remoteusers`（含 handle、不含公钥）
   - 通过关注/点赞发现的用户：插入 `remoteusers` + `remoteuserkeys`（含公钥、不含 handle）

#### 3.8.3 使用场景

- **博客认证**：用户在博客设置中填写 `@someone@mastodon.social` 作为验证链接，系统解析为 profile URL [database.go:L989-L997](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/database.go#L989-L997)
- **文章 @提及**：解析文章中提到的联邦用户并单独投递活动
- **用户搜索**：通过 handle 查找并展示远程用户

---

## 4. 失败分支与错误处理

### 4.1 Inbox 处理失败

**JSON 解码失败** [activitypub.go:L361-L365](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L361-L365):
```go
if err := json.NewDecoder(tee).Decode(&m); err != nil {
    log.Error("Failed decoding JSON: %v", err)
    log.Error("Raw body: %s", rawBody.String())
    return err  // 返回错误给 HTTP 处理层
}
```

**Activity 反序列化失败** [activitypub.go:L549-L560](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L549-L560):
```go
if err := res.Deserialize(m); err != nil {
    log.Error("Unable to resolve Activity: %v", err)
    // 记录活动类型，返回 200 OK 避免重试风暴
    impart.RenderActivityJSON(w, "", http.StatusOK)
    return nil
}
```

> **设计决策**：反序列化失败时返回 200 OK，而不是错误状态码。这是为了避免远程服务器因收到错误而不断重试，造成"重试风暴"。

### 4.2 数据库事务失败

**点赞事务** [activitypub.go:L563-L602](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L563-L602):
```go
t, err := app.db.Begin()
// ... 操作 ...
_, err = t.Exec("INSERT INTO remote_likes ...")
if err != nil {
    if !app.db.isDuplicateKeyErr(err) {
        t.Rollback()
        return fmt.Errorf(...)
    }
    // 重复键也 Rollback（但不视为严重错误）
    t.Rollback()
    return fmt.Errorf(...)
}
err = t.Commit()
if err != nil {
    t.Rollback()
    return fmt.Errorf(...)
}
```

**关注事务** [activitypub.go:L664-L720](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L664-L720):
- 与点赞类似，但在 goroutine 中执行
- 失败仅记录日志，无重试机制

### 4.3 远程请求失败

**Actor 远程获取失败** [activitypub.go:L1062-L1085](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L1062-L1085):
```go
actorResp, err := resolveIRI(app.cfg.App.Host, actorIRI)
if err != nil {
    log.Error("Unable to get base actor! %v", err)
    return nil, nil, impart.HTTPError{http.StatusInternalServerError, "Couldn't fetch actor."}
}
```

**HTTP 请求超时**：
- `activityPubClient()` 设置 15 秒超时
- 超时错误沿调用链向上传递

### 4.4 签名失败

**签名头生成失败** [activitypub.go:L760-L768](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L760-L768):
```go
privKey, err := activitypub.DecodePrivateKey(p.GetPrivKey())
if err != nil {
    return err  // 私钥解码失败，终止发送
}
// ...
err = signer.SignSigHeader(r)
if err != nil {
    log.Error("Can't sign: %v", err)  // 仅记录，继续发送（无签名）
}
```

> **注意**：签名失败仅记录日志，请求仍会发送。这可能导致远程服务器拒绝未签名的请求。

### 4.5 重复键处理

**重复关注** [activitypub.go:L706-L713](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L706-L713):
```go
_, err = t.Exec("INSERT INTO remotefollows ...")
if err != nil {
    if !app.db.isDuplicateKeyErr(err) {
        t.Rollback()
        return
    }
    // 重复键：静默忽略，不 Rollback 其他操作
}
```

**重复密钥** [database_activitypub.go:L36-L47](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/database_activitypub.go#L36-L47):
- 重复键会 Rollback 整个事务
- 视为错误返回

### 4.6 静默失败场景

以下场景仅记录日志，不返回错误：

1. **出站投递失败**：`federatePost()` 中 `makeActivityPost()` 失败仅 `log.Error`
2. **删除投递失败**：`deleteFederatedPost()` 同理
3. **提及投递失败**：循环中跳过失败的提及
4. **Handle 更新失败**：`GetProfileURLFromHandle()` 中更新 handle 失败仅记录

### 4.7 集合（博客）不存在或被封禁

所有 inbox/outbox 处理前检查 [activitypub.go:L125-L146](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L125-L146):
```go
silenced, err := app.db.IsUserSilenced(c.OwnerID)
if silenced {
    return ErrCollectionNotFound  // 返回 404，不暴露封禁状态
}
```

---

### 4.8 Inbox 错误分流全景图

`handleFetchCollectionInbox` 不是简单的「出错就 200」或「出错就 500」两分法。真正要分成 **4 个阶段** 看：**Resolver 前预检**、**Resolver/回调阶段**、**Deserialize 之后的同步收尾**、**goroutine 里的异步收尾**。只有把这 4 段拆开，才能准确回答哪些路径会返回非 200、哪些会被吞成 200。

#### 4.8.1 真正会返回非 200 的路径

**A. Resolver 前预检阶段（4 条）**

这 4 条路径发生在 `res.Deserialize(m)` 之前，HTTP 响应还没写出，返回的 error 会经过 `handler.All()` → `handleError()` → `handleHTTPError()` 渲染为真实状态码：

| # | 代码行 | 触发条件 | 返回的错误 | 最终 HTTP 状态码 |
|---|--------|---------|-----------|----------------|
| 1 | L329-L331 | `GetCollection(alias)` 失败 | 数据库原始错误 | 500 |
| 2 | L334-L336 | `IsUserSilenced()` 查询失败 | `ErrInternalGeneral` | 500 |
| 3 | L338-L339 | 博客所有者被封禁 | `ErrCollectionNotFound` | 404 |
| 4 | L361-L364 | JSON 解码失败 | 解码原始错误 | 500 |

**B. Deserialize 之后的同步事务阶段（Like / Unlike 仍可能返回非 200）**

这里是前几轮最容易漏掉的核心点：`res.Deserialize(m)` 成功之后，`handleFetchCollectionInbox` **并没有立刻结束**。`Like` / `Undo:Like` 还会继续执行同步事务，这些 error 不会走「吞成 200」那条分支，而是会直接 return，最终仍然变成 500。

| # | 分支 | 代码行 | 触发条件 | 最终 HTTP 状态码 |
|---|------|--------|---------|----------------|
| 5 | Like | L563-L567 | `app.db.Begin()` 失败 | 500 |
| 6 | Like | L578-L586 | `INSERT remote_likes` 失败（含重复键） | 500 |
| 7 | Like | L589-L593 | `Commit()` 失败 | 500 |
| 8 | Undo:Like | L604-L608 | `app.db.Begin()` 失败 | 500 |
| 9 | Undo:Like | L619-L623 | `DELETE remote_likes` 失败 | 500 |
| 10 | Undo:Like | L626-L630 | `Commit()` 失败 | 500 |

**更隐蔽的一层：`apAddRemoteUser` 错误会被覆盖**

在 Like / Undo:Like 路径里，如果 `remoteUser == nil`，代码会先执行：

```go
remoteUserID, err = apAddRemoteUser(app, t, fullActor)
```

但后面**没有立即检查 `err`**，而是继续执行 `t.Exec(...)`。而 `apAddRemoteUser` 自己一旦失败，已经先 `t.Rollback()` 了。于是后续 `t.Exec(...)` 往往只会报出「transaction has already been committed or rolled back」之类的次生错误，真正的根因（插 remoteusers / remoteuserkeys 失败）被覆盖掉。这仍然会返回 500，但**返回的是被污染后的错误信息**，不是最初失败点。

#### 4.8.2 会被吞成 200 OK 的路径

真正被统一吞成 200 的，是 **Resolver 回调本身返回的 error**，以及 `Deserialize` 遇到未知活动类型时返回的 error：

```go
if err := res.Deserialize(m); err != nil {
    log.Error("Unable to resolve Activity: %v", err)
    impart.RenderActivityJSON(w, "", http.StatusOK)
    return nil
}
```

| # | 回调 | 代码行 | 触发条件 | 错误内容 |
|---|------|--------|---------|---------|
| 1 | Like | L394-L395 | Like 没有第 0 个 object | `"no object for Like activity at index 0"` |
| 2 | Like | L408-L409 | Like 的 ObjectIRI 为空 | `"didn't get ObjectIRI to Like"` |
| 3 | Like | L411-L413 | 文章 ID 解析失败 | `parsePostIDFromURL` 的错误 |
| 4 | Like | L418-L419 | Like 没有 actor | `"No valid actor string"` |
| 5 | Like | L421-L423 | `getActor` 失败 | 404 转远程抓取失败后的 500，或数据库/网络错误 |
| 6 | Follow | L467-L468 | Follow 的 actor 为空 | `"No valid 'to' string"` |
| 7 | Follow | L470-L472 | `getActor` 失败 | 500 或数据库/网络错误 |
| 8 | Undo→Like | L493-L494 | Unlike 没有 ObjectIRI | `"didn't get ObjectIRI for Undo Like"` |
| 9 | Undo→Like | L496-L498 | 文章 ID 解析失败 | `parsePostIDFromURL` 的错误 |
| 10 | Undo→Like | L500-L502 | `getActor` 失败 | 500 或数据库/网络错误 |
| 11 | Undo→Follow | L522-L529 | `getRemoteUser` 本地查找失败 | 404 或其他错误 |
| 12 | Deserialize | L549-L558 | 未知活动类型 / 无对应回调 | `Deserialize` 自身错误 |

**准确的边界不是「`Deserialize` 前后」**，而是：

- **回调返回给 `Deserialize` 的 error** → 被吞成 200
- **`Deserialize` 之后同步事务里的 error** → 仍然返回非 200

也就是说，之前把「第 549 行之前是非 200、第 549 行之后都是 200」当成规律是不准确的。真正的分界点是 **错误发生在回调里，还是发生在回调结束后的同步事务里**。

#### 4.8.3 已经固定成 200、只剩日志的路径

还有第三类经常和「吞成 200」混在一起，但本质不同：**响应已经先写出 200，后面的失败只会留在日志里**。

这类路径主要在 Follow / Undo:Follow 的 goroutine 中：

| 分支 | 代码行 | 失败点 | 对 HTTP 状态码的影响 |
|------|--------|--------|-------------------|
| Follow | L647-L658 | `a.Serialize()` / `makeActivityPost()` 失败 | 无影响，200 已在回调里写出 |
| Follow | L662-L719 | 写 `remoteusers` / `remoteuserkeys` / `remotefollows` 失败 | 无影响，只记日志 |
| Undo:Follow | L722-L727 | `DELETE remotefollows` 失败 | 无影响，只记日志 |

这类错误不是「被 `Deserialize` 吞掉」，而是**HTTP 响应已经结束，根本没有机会再改状态码**。

#### 4.8.4 错误流向的完整调用链

```
handleFetchCollectionInbox
    │
    ├─ 预检失败（GetCollection / silenced / JSON decode）
    │    └─ return error → handler.All() / handleHTTPError() → 404 或 500
    │
    ├─ res.Deserialize(m)
    │    ├─ 回调返回 error
    │    │    └─ L549-L558 吞成 200 OK
    │    └─ 成功
    │
    ├─ 同步收尾（Like / Undo:Like 事务）
    │    ├─ 事务失败
    │    │    └─ return error → handler.All() / handleHTTPError() → 500
    │    └─ 成功 → RenderActivityJSON(..., 200)
    │
    └─ goroutine 收尾（Follow / Undo:Follow）
         ├─ 失败 → 只记日志，状态码不变
         └─ 成功 → 本地状态补齐
```

**注意**：`handleHTTPError` 中的类型断言使用 `err.(impart.HTTPError)` 值类型，这一点和 `getActor` 一致，也正好构成了下面 Undo 回调 bug 的参照物。

---

### 4.9 Undo 回调中的类型断言 Bug

这是一个**实际存在的代码缺陷**，会导致 404 专项日志永远无法输出。

#### 4.9.1 问题代码

Undo:Follow 分支 [activitypub.go:L522-L529](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L522-L529):

```go
remoteUser, err = getRemoteUser(app, to.String())
if err != nil {
    if iErr, ok := err.(*impart.HTTPError); ok {  // ← 指针类型断言
        if iErr.Status == http.StatusNotFound {
            log.Error("No remoteuser info for Undo event!")
        }
    }
    return err
}
```

#### 4.9.2 为什么断言永远失败？

`getRemoteUser` 返回 404 时的代码 [activitypub.go:L1005-L1007](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L1005-L1007):

```go
return nil, impart.HTTPError{http.StatusNotFound, "No remote user with that ID."}
```

返回的是 `impart.HTTPError` **值类型**。

Go 的类型断言规则：
- `err.(impart.HTTPError)` → 匹配值类型 ✓
- `err.(*impart.HTTPError)` → 匹配指针类型 ✗

当 `getRemoteUser` 返回 `impart.HTTPError{...}` 时，接口 `error` 中存储的是**值**。`err.(*impart.HTTPError)` 尝试断言为指针，**永远不匹配**，`ok` 永远是 `false`。

#### 4.9.3 对比：getActor 中的正确写法

`getActor` 中 [activitypub.go:L1058](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L1058):

```go
if iErr, ok := err.(impart.HTTPError); ok {  // ← 值类型断言，正确！
    if iErr.Status == http.StatusNotFound {
```

同样是调用 `getRemoteUser`，`getActor` 用值类型断言，能正确识别 404。Undo 回调用指针类型断言，识别不了。

#### 4.9.4 实际影响

**1. 日志丢失**

`"No remoteuser info for Undo event!"` 这条日志**永远不会输出**。当远程用户不存在于本地数据库时，开发者无法通过日志区分"用户不存在（404）"和"数据库出错（500 等）"。

**2. 错误仍被吞成 200**

断言失败并不影响错误传播——`return err` 仍然执行，error 被传到 `res.Deserialize`，最终渲染为 200 OK。所以远程端不受影响，只是本地丢失了诊断信息。

**3. 全项目断言风格统计**

搜索整个代码库中 `impart.HTTPError` 的类型断言：

| 断言方式 | 代码库中出现次数 | 位置 |
|---------|----------------|------|
| `err.(impart.HTTPError)` 值类型 | 20+ 处 | handle.go, posts.go, database.go, collections.go, account.go, oauth_test.go, activitypub.go:getActor |
| `err.(*impart.HTTPError)` 指针类型 | **仅 1 处** | activitypub.go:L524（Undo 回调） |

唯一使用指针断言的地方就是 Undo 回调，这几乎确定是一个笔误而非有意为之。

#### 4.9.5 修复方案

将 L524 的指针断言改为值断言：

```go
// 修复前：
if iErr, ok := err.(*impart.HTTPError); ok {

// 修复后：
if iErr, ok := err.(impart.HTTPError); ok {
```

修复后，当 `getRemoteUser` 返回 404 时，日志能正确输出 `"No remoteuser info for Undo event!"`，开发者可以区分"用户不存在"和"其他数据库错误"。

#### 4.9.6 更深层的设计问题

即使修复了类型断言，这段逻辑仍然存在设计上的局限性：

1. **只区分 404，不区分其他错误**：如果 `getRemoteUser` 因为数据库连接断开而返回错误（非 `impart.HTTPError` 类型），Undo 回调会直接 `return err`，没有任何特殊处理。
2. **404 时仍然返回错误**：断言成功后只记录日志，然后继续 `return err`。这意味着即使是正常的"用户不在本地"场景，也会走 `Deserialize` 的吞错路径返回 200。远程端看到的仍然是 200 OK，但 Accept 不会被发送。
3. **Unfollow 的容错逻辑本应更宽容**：取消关注时，如果本地没有该用户记录，合理的做法是直接返回成功（200 OK），而不是返回错误再被吞。当前实现虽然最终效果相同（都是 200），但绕了一圈不必要的错误传播。

---

## 5. WebFinger 发现协议

**远程用户查找**：[RemoteLookup](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/webfinger.go#L103-L145)

```
handle @user@domain
    ↓
GET https://domain/.well-known/webfinger?resource=acct:user@domain
    ↓
查找 rel="self" 的链接（application/activity+json）
    ↓
返回 Actor IRI
```

**本地用户查询**：[wfResolver.FindUser](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/webfinger.go#L32-L91)
- 检查联邦功能是否启用
- 检查用户是否被封禁
- 返回 `self` 链接指向 `/api/collections/{alias}`

---

## 附录：关键数据流图

### 关注流程
```
远程实例                    WriteFreely
    |                            |
    |  POST Follow 到 inbox      |
    | -------------------------> |
    |                            |  handleFetchCollectionInbox
    |                            |    ├─ 解析 Follow
    |                            |    ├─ getActor() 查关注者
    |                            |    └─ 返回 200 OK
    | <------------------------- |
    |                            |
    |  [2秒后]                   |
    |  POST Accept 到关注者 inbox |
    | <------------------------- |  goroutine 异步发送
    |                            |    └─ 写入 remotefollows
```

### 文章分发流程
```
用户发布文章
    |
    |  go federatePost()
    |
    v
GetAPFollowers() → 获取所有关注者
    |
    v
按 shared_inbox 分组 → 减少请求次数
    |
    v
对每个分组：
    ├─ 构造 Create/Update Activity
    ├─ CC = 该组所有关注者
    └─ makeActivityPost() → HTTP POST + 签名
    |
    v
对每个 @提及用户：
    └─ 单独发送到用户 inbox
```
