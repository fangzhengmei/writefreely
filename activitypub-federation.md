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

### 3.3 Actor 查询流程

函数：[getActor](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/activitypub.go#L1053-L1096)

```
getActor(actorIRI)
    ↓
getRemoteUser(app, actorIRI)  // 本地查询
    ├─ 存在 → 返回 *RemoteUser
    └─ 不存在（404）→ 远程查询
        ↓
resolveIRI(hostName, actorIRI)  // HTTP GET 带签名
        ↓
unmarshalActor() → 解析为 Person 对象
        ↓
再次 resolveIRI(baseActor.PublicKey.Owner)  // 获取完整 Actor
        ↓
返回解析后的 Person
```

### 3.4 远程用户添加

函数：[apAddRemoteUser](file:///d:/fz/0601-1/solo-dogfeeding/code/33-writefreely/database_activitypub.go#L21-L50)

**事务流程**：
1. 插入 `remoteusers` 表，获取自增 ID
2. 插入 `remoteuserkeys` 表存储公钥
3. 任何步骤失败则 Rollback

### 3.5 本地 Actor 构造

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
