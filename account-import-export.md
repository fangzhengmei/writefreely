# WriteFreely 导入导出与账号搬迁技术分析

## 一、核心代码文件索引

| 功能模块 | 文件路径 | 关键函数 |
|---------|---------|---------|
| 文章导入处理 | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) | `viewImport`, `handleImport` |
| 导出功能实现 | [export.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/export.go) | `exportPostsCSV`, `exportPostsZip`, `compileFullExport`, `viewExportPosts`, `viewExportFull` |
| 路由注册 | [routes.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/routes.go) | L106-L124 导入导出路由 |
| 数据库创建文章 | [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/database.go) | `CreatePost` (L675-L772) |
| 重复键检测 | [database-sqlite.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/database-sqlite.go) L38-L52, [database-no-sqlite.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/database-no-sqlite.go) L21-L29 | `isDuplicateKeyErr` |
| 导入页面模板 | [import.tmpl](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/templates/user/import.tmpl) | 前端 UI |
| 用户与导出结构 | [users.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/users.go) L66-L86 | `User`, `ExportUser` |
| 集合与文章结构 | [collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/collections.go) L52-L82, [posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/posts.go) L92-L144 | `Collection`, `CollectionObj`, `SubmittedPost`, `Post`, `PublicPost` |
| Slug 生成逻辑 | [posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/posts.go) L1339-L1343 | `getSlug`, `getSlugFromPost` |

---

## 二、路由总览

### 2.1 导入路由
| 方法 | 路径 | 权限 | 处理函数 | 说明 |
|------|------|------|---------|------|
| GET | `/me/import` | 登录用户 | `viewImport` | 显示导入页面，列出用户博客集合供选择 |
| POST | `/api/me/import` | 登录用户 | `handleImport` | 执行文件上传与文章导入 |

### 2.2 导出路由
| 方法 | 路径 | 权限 | 处理函数 | 说明 |
|------|------|------|---------|------|
| GET | `/me/export` | 登录用户 | `viewExportOptions` | 显示导出选项页面 |
| GET | `/me/posts/export.csv` | 登录用户 | `viewExportPosts` | 导出文章为 CSV |
| GET | `/me/posts/export.zip` | 登录用户 | `viewExportPosts` | 导出文章为 ZIP 压缩包 |
| GET | `/me/posts/export.json` | 登录用户 | `viewExportPosts` | 导出文章为 JSON |
| GET | `/me/export.json` | 登录用户 | `viewExportFull` | 完整账号导出（含所有集合+匿名文章） |

---

## 三、文件格式详解

### 3.1 导入文件格式

**支持格式：** 纯文本文件（`.txt`）或 Markdown 文件，MIME 类型为 `text/*`

**上传限制：** 单批次最大 10MB（`10 << 20`）

**导入表单字段：**
- `files`: 多文件上传控件（支持多选）
- `collection`: 目标集合别名（空值表示导入为草稿 / 匿名文章）
- `fileDates`: JSON 字符串，键值对为 `{文件名: Unix时间戳(秒)}`

**文件日期处理流程（前端 JS 实现）：**
```javascript
// 时区偏移（秒）
const tzOffsetSec = new Date().getTimezoneOffset() * 60;
// 遍历文件，转换最后修改时间为 Unix 秒（修正时区）
dateMap[file.name] = Math.round(file.lastModified / 1000) + tzOffsetSec;
```

**文件解析依赖：** 使用外部库 `github.com/writeas/import` 的 `FromFile()` 函数

**解析错误类型：**
- `wfimport.ErrEmptyFile`: 文件为空，跳过并提示
- `wfimport.ErrInvalidContentType`: 不支持的内容类型，跳过并提示
- 其他错误：加入错误列表继续处理

**文件内容解析后结构：**
```go
// wfimport.FromFile 返回的 post 对象大致包含：
// - Title: 文章标题
// - Content: 正文内容
// - Created: 创建时间（通过 fileDates 映射补充）
```

---

### 3.2 CSV 导出格式

**生成函数：** `exportPostsCSV()` 在 [export.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/export.go) L23-L46

**表头字段：**
```
id, slug, blog, url, created, title, body
```

| 字段 | 说明 | 来源 |
|------|------|------|
| `id` | 文章唯一 ID（友好随机字符串） | `p.ID` |
| `slug` | URL 友好路径 | `p.Slug.String` |
| `blog` | 所属集合别名 | `p.Collection.Alias` |
| `url` | 规范化完整 URL | `p.CanonicalURL(hostName)` |
| `created` | ISO 8601 创建时间 | `p.Created8601()` |
| `title` | 文章标题 | `p.Title.String` |
| `body` | 正文内容（换行转义为 `\n`） | `strings.Replace(p.Content, "\n", "\\n", -1)` |

**文件名规则：** `{username}-posts-{YYYYMMDDHHmm}.csv`

---

### 3.3 ZIP 导出格式

**生成函数：** `exportPostsZip()` 在 [export.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/export.go) L54-L101

**压缩包内部结构：**
```
{collection-alias}/{slug}_{post-id}.txt   // 集合文章
{slug}_{post-id}.txt                       // 无集合的匿名文章
{post-id}.txt                              // 无 slug 的文章
```

**单文件命名规则：**
1. 属于集合的文章：`{alias}/{slug}_{id}.txt`
2. 有 slug 的文章：`{slug}_{id}.txt`
3. 无 slug 的文章：`{id}.txt`

**单个 TXT 文件内容格式：**
```
# {文章标题}（如果标题非空）

{正文内容}
```

**文件元数据：** ZIP 文件头的修改时间设置为文章创建时间（`head.SetModTime(file.Mod)`）

**文件名规则：** `{username}-posts-{YYYYMMDDHHmm}.zip`

---

### 3.4 JSON 文章导出格式

**生成函数：** `viewExportPosts()` 在 [account.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account.go) L605-L667

**顶层结构：** `[]PublicPost` 数组

**PublicPost / Post 主要字段：**
```json
{
  "id": "abc123",
  "slug": "hello-world",
  "appearance": "norm",
  "language": "zh",
  "rtl": false,
  "created": "2024-01-15T10:30:00Z",
  "updated": "2024-01-15T10:30:00Z",
  "likes": 0,
  "title": "Hello World",
  "body": "文章正文内容...",
  "tags": ["tag1", "tag2"],
  "views": 100,
  "url": "https://example.com/blog/hello-world",
  "collection": { /* CollectionObj */ }
}
```

**查询参数：** `?pretty=1` 输出格式化缩进 JSON

---

### 3.5 完整账号导出格式（Full Export）

**生成函数：** `compileFullExport()` + `viewExportFull()` 在 [export.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/export.go) L103-L132 和 [account.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account.go) L669-L687

**ExportUser 结构：**
```json
{
  "username": "johndoe",
  "has_pass": true,
  "email": "encrypted-string",
  "created": "2023-01-01T00:00:00Z",
  "status": 0,
  "collections": [
    {
      "alias": "my-blog",
      "title": "我的博客",
      "description": "博客描述",
      "public": true,
      "views": 1234,
      "style_sheet": "",
      "format": "",
      "monetization_pointer": "",
      "verification_link": "",
      "total_posts": 10,
      "posts": [ /* PublicPost 数组 */ ]
    }
  ],
  "posts": [ /* 匿名文章 PublicPost 数组 */ ]
}
```

**导出内容：**
1. 用户基本信息（不含密码哈希和内部 ID）
2. 所有集合及其全部文章
3. 所有匿名/草稿文章（不属于任何集合）

**文件名规则：** `{username}-{YYYYMMDDHHmm}.json`

---

## 四、文章去重机制

### 4.1 去重策略：基于 Slug 唯一性（非内容去重）

**重要说明：WriteFreely 的导入机制不基于文章标题或内容进行去重检测。** 如果导入相同内容的文章两次，会创建两篇独立的文章记录，但 Slug 可能冲突。

### 4.2 Slug 生成流程

**代码位置：** `CreatePost()` 在 [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/database.go) L694-L718

**Slug 优先级：**
1. **用户显式指定**：如果 `post.Slug` 非空，使用 `getSlug()` 规范化
2. **基于标题生成**：如果标题非空，调用 `getSlug(*post.Title, language)`
3. **基于内容生成**：标题为空时，取正文前 N 字符 `getSlug(*post.Content, language)`
4. **使用友好 ID**：以上均为空时，直接使用文章的 `friendlyID`

### 4.3 Slug 冲突处理（自动重命名）

**代码位置：** [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/database.go) L744-L756

```go
_, err = stmt.Exec(...)
if err != nil {
    if db.isDuplicateKeyErr(err) {
        // 检测到重复键（唯一约束冲突），生成安全的唯一 Slug
        slug = sql.NullString{id.GenSafeUniqueSlug(slug.String), true}
        _, err = stmt.Exec(...) // 重试插入
        if err != nil {
            return nil, handleFailedPostInsert(...)
        }
    }
}
```

**处理逻辑：**
1. 首次 INSERT 失败且错误为重复键（MySQL 1062 / SQLite UNIQUE 约束）
2. 调用 `id.GenSafeUniqueSlug(originalSlug)` 生成带后缀的唯一 Slug（如 `hello-world-2`）
3. 用新 Slug 重试插入
4. 重试仍失败则返回错误，该篇文章导入失败

**数据库驱动适配：**
- **MySQL**：错误号 `1062` (`mySQLErrDuplicateKey`) - [database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/database.go) L46
- **SQLite**：通过错误信息字符串匹配检测

---

## 五、部分失败状态处理机制

### 5.1 整体策略：部分成功 + 错误聚合

**设计原则：** 单篇文章导入失败不影响其他文章，采用"尽力而为"模式。

### 5.2 错误收集机制

**代码位置：** `handleImport()` 在 [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go)

**关键变量：**
```go
var fileErrs []error       // 累积所有错误
filesSubmitted := len(files) // 提交文件总数
var filesImported int        // 成功导入计数
```

### 5.3 各阶段失败处理

| 阶段 | 可能错误 | 处理方式 | 错误是否累计 |
|------|---------|---------|------------|
| **文件打开** | 文件读取失败 | 记录错误，`continue` 跳过 | ✅ 累计 |
| **临时文件创建** | 磁盘空间不足 / 权限问题 | 记录错误，`continue` 跳过 | ✅ 累计 |
| **文件复制** | I/O 错误 | 记录错误，`continue` 跳过 | ✅ 累计 |
| **文件 stat** | 文件系统错误 | 记录错误，`continue` 跳过 | ✅ 累计 |
| **内容解析（空文件）** | `ErrEmptyFile` | 单独 flash 提示，**不计入** fileErrs | ❌ 单独提示 |
| **内容解析（类型错误）** | `ErrInvalidContentType` | 单独 flash 提示，**不计入** fileErrs | ❌ 单独提示 |
| **内容解析（其他）** | 读取失败 | 记录错误，`continue` 跳过 | ✅ 累计 |
| **数据库创建** | DB 错误 / 唯一约束重试失败 | 记录错误，`continue` 跳过 | ✅ 累计 |

### 5.4 结果状态反馈

**代码位置：** [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L180-L193

**状态分级：**

| 条件 | 反馈等级 | Flash 消息格式 |
|------|---------|---------------|
| `filesImported == filesSubmitted` | **SUCCESS** | `SUCCESS: Import complete, {N} posts imported.` |
| `filesImported > 0 && filesImported < filesSubmitted` | **INFO** | `INFO: {N} of {M} posts imported, see details below.` |
| `filesImported == 0` | （纯错误） | 无成功消息，仅展示错误列表 |

**错误列表渲染：**
- 使用 `github.com/hashicorp/go-multierror` 的 `ListFormatFunc` 格式化为 HTML 列表
- 在模板中通过 `.Flashes` 渲染为红色错误提示框

### 5.5 联邦/ActivityPub 异步处理

**代码位置：** [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L165-L177

```go
if app.cfg.App.Federation && coll.ID > 0 {
    go federatePost(
        app,
        &PublicPost{...},
        coll.ID,
        false,
    )
}
```

**特性：**
- 仅当目标为集合（非草稿）且启用联邦时触发
- 使用 goroutine **异步执行**，不阻塞导入流程
- 联邦分发失败不影响导入结果（已在数据库中创建成功）
- 不等待联邦完成即返回结果

---

## 六、账号搬迁详细分析：字段保留与丢失

### 6.1 导出字段完整对照表

以下对比数据库中完整字段与实际导出内容的差异，标注每个字段在不同导出格式中的保留情况。

#### 6.1.1 用户字段（User）

| 字段名 | 数据库/结构体 | 完整JSON导出 | 文章导出JSON | 说明 |
|-------|-------------|-------------|-------------|------|
| `ID` | `int64` | ❌ 丢失 | ❌ 丢失 | `json:"-"` 标签，内部 ID 不对外暴露 |
| `Username` | `string` | ✅ 保留 | ❌ 不包含 | 用户名在完整导出中作为 User 字段保留 |
| `HashedPass` | `[]byte` | ❌ 丢失 | ❌ 丢失 | `json:"-"` 标签，安全考虑不导出密码哈希 |
| `HasPass` | `bool` | ✅ 保留 | ❌ 不包含 | 标识用户是否设置了密码 |
| `Email` | `zero.String` | ✅ 保留（加密值） | ❌ 不包含 | 导出的是加密后的密文，无法直接使用 |
| `Created` | `time.Time` | ✅ 保留 | ❌ 不包含 | 账号创建时间 |
| `Status` | `UserStatus` | ✅ 保留 | ❌ 不包含 | 账号状态（0=正常, 1=禁言） |
| `clearEmail` | `string` | ❌ 丢失 | ❌ 丢失 | 内存缓存的解密邮箱，`json:"email"` 标签但值来自加密字段 |

**代码来源：** [users.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/users.go) L66-L76

#### 6.1.2 集合字段（Collection）

| 字段名 | 数据库/结构体 | 完整JSON导出 | 文章导出JSON | CSV导出 | ZIP导出 | 说明 |
|-------|-------------|-------------|-------------|---------|---------|------|
| `ID` | `int64` | ❌ 丢失 | ❌ 丢失 | ❌ 不包含 | ❌ 不包含 | 内部 ID，`json:"-"` |
| `Alias` | `string` | ✅ 保留 | ✅ 保留 | ✅ 保留（blog列） | ✅ 保留（目录名） | 集合别名/URL 路径 |
| `Title` | `string` | ✅ 保留 | ✅ 部分 | ❌ 不包含 | ❌ 不包含 | 集合显示名称 |
| `Description` | `string` | ✅ 保留 | ✅ 部分 | ❌ 不包含 | ❌ 不包含 | 集合描述 |
| `Direction` | `string` | ❌ 丢失（空值省略） | ❌ 不包含 | ❌ 不包含 | ❌ 不包含 | 排版方向，**TODO: add Direction to db**（数据库无此字段），`omitempty` 导致空值省略，无 `datastore` 标签不参与 DB 映射 |
| `Language` | `string` | ❌ 丢失（空值省略） | ❌ 不包含 | ❌ 不包含 | ❌ 不包含 | 集合默认语言，**TODO: add Language to db**（数据库无此字段），`omitempty` 导致空值省略，无 `datastore` 标签不参与 DB 映射 |
| `StyleSheet` | `string` | ✅ 保留 | ❌ 不包含 | ❌ 不包含 | ❌ 不包含 | 自定义 CSS |
| `Script` | `string` | ✅ 保留 | ❌ 不包含 | ❌ 不包含 | ❌ 不包含 | 自定义 JS |
| `Signature` | `string` | ❌ 丢失 | ❌ 不包含 | ❌ 不包含 | ❌ 不包含 | 文章签名/页脚，`json:"-"` |
| `Public` | `bool` | ✅ 保留 | ❌ 不包含 | ❌ 不包含 | ❌ 不包含 | 是否公开 |
| `Visibility` | `collVisibility` | ❌ 丢失 | ❌ 不包含 | ❌ 不包含 | ❌ 不包含 | `json:"-"`，隐私等级 |
| `Format` | `string` | ✅ 保留 | ❌ 不包含 | ❌ 不包含 | ❌ 不包含 | 集合格式/主题 |
| `Views` | `int64` | ✅ 保留 | ✅ 部分 | ❌ 不包含 | ❌ 不包含 | 访问量统计 |
| `OwnerID` | `int64` | ❌ 丢失 | ❌ 丢失 | ❌ 不包含 | ❌ 不包含 | 所有者 ID，`json:"-"` |
| `PublicOwner` | `bool` | ❌ 丢失 | ❌ 丢失 | ❌ 不包含 | ❌ 不包含 | `json:"-"` |
| `URL` | `string` | ✅ 保留 | ✅ 保留 | ❌ 不包含 | ❌ 不包含 | 完整 URL |
| `Monetization` | `string` | ✅ 保留 | ❌ 不包含 | ❌ 不包含 | ❌ 不包含 |  monetization 指针 |
| `Verification` | `string` | ✅ 保留 | ❌ 不包含 | ❌ 不包含 | ❌ 不包含 | 验证链接 |
| `TotalPosts` | `int` | ✅ 保留 | ❌ 不包含 | ❌ 不包含 | ❌ 不包含 | 文章总数（CollectionObj 字段） |

**代码来源：** [collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/collections.go) L52-L82

#### 6.1.3 文章字段（Post / PublicPost）

| 字段名 | 数据库/结构体 | 完整JSON导出 | 文章JSON导出 | CSV导出 | ZIP导出 | 导入时是否保留 |
|-------|-------------|-------------|-------------|---------|---------|-------------|
| `ID` | `string` | ✅ 保留 | ✅ 保留 | ✅ 保留（id列） | ✅ 保留（文件名） | ❌ 重新生成 |
| `Slug` | `null.String` | ✅ 保留 | ✅ 保留 | ✅ 保留（slug列） | ✅ 保留（文件名） | ⚠️ 可能变更 |
| `Font` | `string` | ✅ 保留(appearance) | ✅ 保留(appearance) | ❌ 不包含 | ❌ 不包含 | ❌ 固定为 "norm" |
| `Language` | `zero.String` | ✅ 保留 | ✅ 保留 | ❌ 不包含 | ❌ 不包含 | ❌ 不保留 |
| `RTL` | `zero.Bool` | ✅ 保留 | ✅ 保留 | ❌ 不包含 | ❌ 不包含 | ❌ 不保留 |
| `Privacy` | `int64` | ❌ 丢失 | ❌ 丢失 | ❌ 不包含 | ❌ 不包含 | `json:"-"` |
| `OwnerID` | `null.Int` | ❌ 丢失 | ❌ 丢失 | ❌ 不包含 | ❌ 不包含 | `json:"-"` |
| `CollectionID` | `null.Int` | ❌ 丢失 | ❌ 丢失 | ❌ 不包含 | ❌ 不包含 | `json:"-"`，通过 collection 字段替代 |
| `PinnedPosition` | `null.Int` | ❌ 丢失 | ❌ 丢失 | ❌ 不包含 | ❌ 不包含 | `json:"-"`，置顶位置 |
| `Created` | `time.Time` | ✅ 保留 | ✅ 保留 | ✅ 保留（created列） | ✅ 保留（修改时间） | ✅ 可通过 fileDates 保留 |
| `Updated` | `time.Time` | ✅ 保留 | ✅ 保留 | ❌ 不包含 | ❌ 不包含 | ❌ 重置为导入时间 |
| `ViewCount` | `int64` | ✅ 保留(views) | ✅ 保留(views) | ❌ 不包含 | ❌ 不包含 | ❌ 重置为 0 |
| `LikeCount` | `int64` | ✅ 保留(likes) | ✅ 保留(likes) | ❌ 不包含 | ❌ 不包含 | ❌ 重置为 0 |
| `Title` | `zero.String` | ✅ 保留 | ✅ 保留 | ✅ 保留（title列） | ✅ 保留（文件首行# 标题） | ✅ 保留 |
| `Content` | `string` | ✅ 保留(body) | ✅ 保留(body) | ✅ 保留（body列） | ✅ 保留（文件正文） | ✅ 保留 |
| `Tags` | `[]string` | ✅ 保留 | ✅ 保留 | ❌ 不包含 | ❌ 不包含 | ⚠️ 隐含在正文中 |
| `Images` | `[]string` | ✅ 保留 | ✅ 保留 | ❌ 不包含 | ❌ 不包含 | ❌ 图片需单独迁移 |
| `IsPaid` | `bool` | ❌ 丢失 | ❌ 丢失 | ❌ 不包含 | ❌ 不包含 | 付费文章标识 |

**代码来源：** [posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/posts.go) L103-L144

#### 6.1.4 匿名文章导出的字段差异

**注意：** 匿名文章（`GetAnonymousPosts`）的查询字段比完整文章少。

**完整文章查询字段（`postCols`）：**
```
id, slug, text_appearance, language, rtl, privacy, owner_id,
collection_id, pinned_position, created, updated, view_count, title, content
```

**匿名文章查询字段（只有 7 个字段）：**
```
id, view_count, title, language, created, updated, content
```

**匿名文章缺失的导出字段：**
- `slug` - URL 路径
- `text_appearance` - 字体样式
- `rtl` - 从右到左排版
- `privacy` - 隐私等级
- `collection_id` - 所属集合（本身就是无集合）
- `pinned_position` - 置顶位置

**代码来源：**
- 完整查询：[database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/database.go) L1122
- 匿名文章：[database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/database.go) L2141

#### 6.1.5 导出格式字段覆盖矩阵

| 数据类型 | JSON完整导出 | JSON文章导出 | CSV导出 | ZIP导出 | 实际可导入 |
|---------|-------------|-------------|---------|---------|----------|
| 用户信息 | ✅ 完整 | ❌ | ❌ | ❌ | ❌ 需手动注册 |
| 集合列表 | ✅ 完整 | ⚠️ 部分 | ⚠️ 仅别名 | ⚠️ 仅目录名 | ❌ 需手动创建 |
| 文章内容 | ✅ 完整 | ✅ 完整 | ✅ 完整 | ✅ 完整 | ✅ 通过 TXT 导入 |
| 文章标题 | ✅ 保留 | ✅ 保留 | ✅ 保留 | ✅ 保留 | ✅ 保留 |
| 创建时间 | ✅ 保留 | ✅ 保留 | ✅ 保留 | ⚠️ 文件修改时间 | ✅ 通过 fileDates |
| 更新时间 | ✅ 保留 | ✅ 保留 | ❌ | ❌ | ❌ 重置 |
| Slug | ✅ 保留 | ✅ 保留 | ✅ 保留 | ⚠️ 文件名含 | ❌ 重新生成 |
| 字体样式 | ✅ 保留 | ✅ 保留 | ❌ | ❌ | ❌ 固定 norm |
| 语言设置 | ✅ 保留 | ✅ 保留 | ❌ | ❌ | ❌ 不保留 |
| 浏览统计 | ✅ 保留 | ✅ 保留 | ❌ | ❌ | ❌ 清零 |
| 点赞统计 | ✅ 保留 | ✅ 保留 | ❌ | ❌ | ❌ 清零 |
| 标签 | ✅ 保留 | ✅ 保留 | ❌ | ❌ | ⚠️ 隐含在正文 |

---

### 6.2 上传失败完整处理流程与边界条件

#### 6.2.1 导入请求完整时序（含所有错误分支）

```
请求入口
   │
   ▼
【路由层：登录态校验】
   │
   ├─ 失败 → 返回 401 错误 → END（整单中断，无任何处理）
   │
   ▼
进入 handleImport 函数 [account_import.go L57]
   │
   ├─ L59: r.ParseMultipartForm(10 << 20)  ── 10MB 限制，未检查 err
   │
   ├─ L61: collAlias = r.PostFormValue("collection")
   │
   ├─【目标集合存在性校验】[L66-L71]
   │   │
   │   ├─ collAlias 非空时执行
   │   ├─ app.db.GetCollection(collAlias)
   │   ├─ 失败 → return err → END（整单中断）
   │   │
   │   ▼
   ├─【集合所有权校验】[L73-L77]
   │   │
   │   ├─ 检查 coll.OwnerID == u.ID
   │   ├─ 失败 → 加 Flash + return err → END（整单中断）
   │   │
   │   ▼
   ├─【fileDates JSON 格式校验】[L82-L86]
   │   │
   │   ├─ json.Unmarshal(fileDates JSON)
   │   ├─ 失败 → return 400 Bad Request → END（整单中断）
   │   │
   ├─ 以上 4 项全部通过 → 进入文件处理循环
   │
   ▼
┌─────────────────────────────────────────────────────────┐
│  for _, formFile := range files {                       │  ← 循环开始，单文件失败不影响其他文件
│     │                                                  │
│     ├─【文件打开】[L94-L99]                             │
│     │  ├─ 失败 → 加 err + continue → 下一个文件        │
│     │                                                  │
│     ├─【临时文件创建】[L102-L107]                       │
│     │  ├─ 失败 → 加 err + continue → 下一个文件        │
│     │                                                  │
│     ├─【文件复制】[L110-L115]                           │
│     │  ├─ 失败 → 加 err + continue → 下一个文件        │
│     │                                                  │
│     ├─【文件 Stat】[L117-L122]                          │
│     │  ├─ 失败 → 加 err + continue → 下一个文件        │
│     │                                                  │
│     ├─ 以上全部成功 → 进入内容解析                     │
│     │                                                  │
│     ├─【内容解析】[L130-L143]                           │
│     │  │                                               │
│     │  ├─ ErrEmptyFile → 单独 Flash + continue        │  ← 不计入 fileErrs
│     │  ├─ ErrInvalidContentType → 单独 Flash + continue│  ← 不计入 fileErrs
│     │  └─ 其他错误 → 加 err + continue → 下一个文件    │
│     │                                                  │
│     ├─【数据库创建文章】[L157-L162]                     │
│     │  │                                               │
│     │  ├─ CreatePost 内部包含 Slug 冲突重试            │
│     │  ├─ 重试成功 → 文章创建成功                      │
│     │  └─ 失败（包括重试失败）→ 加 err + continue      │
│     │                                                  │
│     ├─【联邦分发（异步）】[L165-L177]                   │
│     │  ├─ go federatePost(...)                         │
│     │  └─ 后台执行，失败不影响导入结果                  │
│     │                                                  │
│     └─ filesImported++ → 成功计数 +1                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
   │
   ▼
【结果状态判定】[L180-L192]
   │
   ├─ filesImported == filesSubmitted → SUCCESS Flash
   ├─ filesImported > 0 → INFO Flash
   └─ 全部失败 → 无成功/信息 Flash，仅错误列表
   │
   ▼
return 302 Redirect /me/import → END
```

#### 6.2.2 整单中断 vs 单文件继续：精确边界对照表

| 检查阶段 | 代码位置 | 触发条件 | 处理方式 | 已创建文章是否保留 | 临时文件是否清理 |
|---------|---------|---------|---------|-----------------|----------------|
| **登录态校验** | 路由层 `handler.User()` | Session/Token 无效 | **整单中断** | ❌ 无 | ❌ 无 |
| **集合存在性** | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L66-L71 | `collAlias` 非空但集合不存在 | **整单中断** | ❌ 无 | ❌ 无 |
| **集合所有权** | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L73-L77 | `coll.OwnerID != u.ID` | **整单中断** | ❌ 无 | ❌ 无 |
| **fileDates JSON** | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L82-L86 | JSON 解析失败 | **整单中断** | ❌ 无 | ❌ 无 |
| **文件打开失败** | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L94-L99 | `formFile.Open()` 失败 | **单文件继续** | ✅ 之前成功的保留 | ⚠️ defer 关闭已打开的 |
| **临时文件创建失败** | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L102-L107 | `os.CreateTemp()` 失败 | **单文件继续** | ✅ 之前成功的保留 | ⚠️ 源文件 defer 关闭 |
| **文件复制失败** | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L110-L115 | `io.Copy()` 失败 | **单文件继续** | ✅ 之前成功的保留 | ⚠️ 源文件 defer 关闭，临时文件 defer Close |
| **文件 stat 失败** | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L117-L122 | `tempFile.Stat()` 失败 | **单文件继续** | ✅ 之前成功的保留 | ⚠️ defer 关闭 |
| **内容解析-空文件** | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L131-L134 | `wfimport.ErrEmptyFile` | **单文件继续** | ✅ 之前成功的保留 | ⚠️ defer 关闭，临时文件残留 |
| **内容解析-类型错误** | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L135-L138 | `wfimport.ErrInvalidContentType` | **单文件继续** | ✅ 之前成功的保留 | ⚠️ defer 关闭，临时文件残留 |
| **内容解析-其他错误** | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L139-L143 | 其他解析错误 | **单文件继续** | ✅ 之前成功的保留 | ⚠️ defer 关闭，临时文件残留 |
| **数据库创建失败** | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L157-L162 | `CreatePost()` 失败（含 Slug 重试失败） | **单文件继续** | ✅ 之前成功的保留 | ⚠️ defer 关闭，临时文件残留 |
| **联邦分发失败** | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L165-L177 | goroutine 内异步失败 | **不计入失败** | ✅ 文章已创建 | ⚠️ 不影响 |

#### 6.2.3 边界条件详细说明

**整单中断的 4 个必要条件：**
1. 必须发生在 **for 循环之前**
2. 必须通过 `return err` 直接跳出函数
3. 此时还没有处理任何文件
4. 用户需要重新选择文件并提交

**整单中断后系统状态：**
- ✅ 数据库中没有创建任何文章
- ✅ 没有创建任何临时文件
- ✅ 没有触发任何联邦分发
- ✅ 已添加的 Flash 消息（所有权校验失败时）会显示给用户

**单文件继续处理的 3 个必要条件：**
1. 必须发生在 **for 循环内部**
2. 必须通过 `continue` 跳过当前迭代
3. 已成功处理的文件不受影响

**临时文件残留问题：**
- 临时文件命名模式：`post-upload-*.txt`
- 创建位置：操作系统临时目录（`os.TempDir()`）
- 清理时机：匿名函数返回时 `Close()`，但**不会删除**
- 依赖操作系统的临时文件清理机制自动删除
- 导入失败或成功后，这些文件都不会被主动清理

---

### 6.3 导入前置校验：导致整单中断的场景

**前置校验定义：** 在进入文件循环处理之前执行的检查，一旦失败则整个导入请求终止，不处理任何文件。

#### 6.3.1 校验点详情

| 序号 | 校验点 | 代码位置 | 失败表现 | HTTP状态码 | 说明 |
|-----|-------|---------|---------|-----------|------|
| 1 | **登录态校验** | 路由层 `handler.User()` [handle.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/handle.go) | 返回错误页面/JSON | 401 Unauthorized | 未登录或 Token 无效则直接拦截 |
| 2 | **Multipart 表单解析** | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L59 | Go 内置错误处理 | 通常 400 | `r.ParseMultipartForm(10 << 20)` 解析失败或超过 10MB 限制 |
| 3 | **目标集合存在性** | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L67-L71 | 返回数据库错误 | 500 / 404 | `collAlias` 非空时查询集合，不存在则整单失败 |
| 4 | **集合所有权校验** | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L73-L77 | flash + 错误返回 | 401 | `coll.OwnerID != u.ID` 时禁止导入到他人集合 |
| 5 | **fileDates JSON 格式** | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L82-L86 | Bad Request 错误 | 400 | `json.Unmarshal` 失败则整单中断 |

#### 6.3.2 各校验点详细分析

**1. 登录态校验（路由中间件）**
- **触发时机**：请求到达 `handleImport` 函数之前
- **校验逻辑**：`handler.User()` 中间件检查 Session 或 Access Token
- **失败影响**：直接返回 `ErrNotLoggedIn` 或 `ErrBadAccessToken`，不执行任何导入逻辑
- **错误返回**：Web 请求重定向到登录页，API 请求返回 JSON 错误

**2. Multipart 表单解析（10MB 限制）**
```go
r.ParseMultipartForm(10 << 20)
```
- **触发时机**：进入函数后第一行代码
- **校验逻辑**：Go 标准库 `ParseMultipartForm` 限制请求体大小为 10MB
- **失败影响**：解析失败直接导致后续所有表单字段为空，可能引发后续错误
- **注意**：`ParseMultipartForm` 超过限制时 Go 会返回 `ErrMessageTooLarge` 错误，但代码中未检查 `err` 返回值

**3. 目标集合存在性校验**
```go
if collAlias != "" {
    coll, err = app.db.GetCollection(collAlias)
    if err != nil {
        log.Error("Unable to get collection for import: %s", err)
        return err
    }
}
```
- **触发时机**：解析表单后，文件循环前
- **校验逻辑**：指定了 `collection` 参数时，查询数据库确认集合存在
- **失败影响**：直接 `return err`，整单失败
- **可能原因**：集合别名拼写错误、集合已被删除

**4. 集合所有权校验**
```go
if coll.OwnerID != u.ID {
    err := ErrUnauthorizedGeneral
    _ = addSessionFlash(app, w, r, err.Message, nil)
    return err
}
```
- **触发时机**：确认集合存在后
- **校验逻辑**：验证当前登录用户是集合的所有者
- **失败影响**：添加 Flash 消息后返回错误，整单失败
- **安全意义**：防止越权向他人博客导入文章

**5. fileDates JSON 格式校验**
```go
fileDates := make(map[string]int64)
err = json.Unmarshal([]byte(r.FormValue("fileDates")), &fileDates)
if err != nil {
    log.Error("invalid form data for file dates: %v", err)
    return impart.HTTPError{http.StatusBadRequest, "form data for file dates was invalid"}
}
```
- **触发时机**：文件循环前
- **校验逻辑**：解析前端传入的 `fileDates` JSON 字符串
- **失败影响**：返回 400 Bad Request，整单失败
- **失败原因**：前端 JS 异常、恶意构造请求、JSON 格式错误

#### 6.3.3 前置校验失败后的状态

**所有前置校验失败都具有以下共性：**
- ✅ 数据库中不会创建任何文章
- ✅ 不会产生任何临时文件
- ✅ 不会触发任何联邦分发
- ❌ 不会有 Flash 成功/部分成功消息
- ❌ 用户需要重新选择文件并提交

---

### 6.4 单文件失败场景分析

**单文件失败定义：** 在文件循环内发生的错误，仅影响当前文件，其他文件继续处理。

**代码结构：** `for _, formFile := range files` 循环内的所有错误都使用 `continue` 跳过当前文件。

#### 6.4.1 单文件失败类型总览

| 类别 | 错误场景 | 错误消息格式 | 是否计入 fileErrs | 反馈方式 |
|-----|---------|-------------|------------------|---------|
| **文件操作** | 文件打开失败 | `Unable to read file {filename}` | ✅ 是 | 错误列表 |
| **文件操作** | 临时文件创建失败 | `Internal error for {filename}` | ✅ 是 | 错误列表 |
| **文件操作** | 文件复制失败 | `Internal error for {filename}` | ✅ 是 | 错误列表 |
| **文件操作** | 文件 stat 失败 | `Internal error for {filename}` | ✅ 是 | 错误列表 |
| **内容解析** | 文件为空 | `{filename} was empty, import skipped` | ❌ 否 | 单独 Flash 提示 |
| **内容解析** | 不支持的内容类型 | `{filename} is not a supported post file` | ❌ 否 | 单独 Flash 提示 |
| **内容解析** | 其他解析错误 | `failed to read copy of {filename}` | ✅ 是 | 错误列表 |
| **数据库** | 创建文章失败 | `failed to create post from {filename}` | ✅ 是 | 错误列表 |

#### 6.4.2 各阶段失败详细分析

**第一阶段：文件读取与临时化（4 种失败）**

```go
ok := func() bool {
    file, err := formFile.Open()        // 1. 打开上传的文件
    // ...
    tempFile, err := os.CreateTemp(...)  // 2. 创建临时文件
    // ...
    _, err = io.Copy(tempFile, file)     // 3. 复制内容到临时文件
    // ...
    info, err := tempFile.Stat()         // 4. 获取文件信息
    // ...
}()
if !ok {
    continue  // 以上任一失败都跳过当前文件
}
```

**失败原因与影响：**
- **文件打开失败**：上传文件损坏、网络中断导致部分上传、文件被占用
- **临时文件创建失败**：服务器磁盘满、权限问题、临时目录不可写
- **文件复制失败**：读取过程中连接断开、磁盘 I/O 错误
- **文件 stat 失败**：罕见的文件系统异常

**共同点：** 都加入 `fileErrs` 错误列表，以 `Internal error` 或 `Unable to read file` 消息呈现给用户。

**第二阶段：内容解析（3 种失败）**

```go
post, err := wfimport.FromFile(...)
if err == wfimport.ErrEmptyFile {
    // 空文件：单独 Flash，不计入错误列表
    _ = addSessionFlash(app, w, r, fmt.Sprintf("%s was empty, import skipped", formFile.Filename), nil)
    continue
} else if err == wfimport.ErrInvalidContentType {
    // 不支持的类型：单独 Flash，不计入错误列表
    _ = addSessionFlash(app, w, r, fmt.Sprintf("%s is not a supported post file", formFile.Filename), nil)
    continue
} else if err != nil {
    // 其他解析错误：计入错误列表
    fileErrs = append(fileErrs, fmt.Errorf("failed to read copy of %s", formFile.Filename))
    continue
}
```

**错误分级设计：**
- **空文件**：用户可能误传，属于"善意"错误，给予温和提示
- **不支持的类型**：用户上传了非文本文件，提示格式要求
- **其他解析错误**：未预期的解析失败，归入技术错误列表

**为什么空文件和类型错误单独提示？**
- 这类错误通常是用户操作失误，不是系统故障
- 从错误列表中分离出来，避免用户混淆"真错误"和"小提示"
- 代码注释：`// not a real error so don't log`

**第三阶段：数据库写入（1 大类，内含重试）**

```go
rp, err := app.db.CreatePost(u.ID, coll.ID, &submittedPost)
if err != nil {
    fileErrs = append(fileErrs, fmt.Errorf("failed to create post from %s", formFile.Filename))
    log.Error("import textfile: create db post: %v", err)
    continue
}
```

**`CreatePost` 内部的子失败场景：**
1. **Slug 重复（可恢复）**：
   - 首次 INSERT 触发唯一键冲突
   - 自动调用 `GenSafeUniqueSlug()` 生成带后缀的 Slug
   - 重试一次 INSERT
   - 重试成功 → 文章正常创建（单文件不失败）
   - 重试失败 → 进入下面的"其他数据库错误"

2. **其他数据库错误（不可恢复）**：
   - 数据库连接断开
   - 约束违反（非 Slug 的唯一键）
   - 数据截断（内容过长）
   - 事务超时
   - 统一表现为 `failed to create post from {filename}`

**第四阶段：联邦分发（异步，不计入失败）**

```go
if app.cfg.App.Federation && coll.ID > 0 {
    go federatePost(...)  // 异步执行，不阻塞导入流程
}
filesImported++  // 无论联邦成功与否，都算作导入成功
```

- **性质**：异步 goroutine，后台执行
- **失败影响**：联邦失败不影响文章已创建的事实
- **用户感知**：用户无法从导入结果中得知联邦是否成功
- **日志**：联邦失败会记录到服务端日志，但不返回给用户

#### 6.4.3 错误消息的两种渲染方式

**方式一：错误列表（`fileErrs` 聚合）**
```go
if len(fileErrs) != 0 {
    _ = addSessionFlash(app, w, r, multierror.ListFormatFunc(fileErrs), nil)
}
```
- 使用 `go-multierror` 的 `ListFormatFunc` 格式化为 HTML 列表
- 在模板中通过 `.Flashes` 渲染为红色错误框
- 每个错误一行，形如 `* failed to create post from foo.txt`

**方式二：单独 Flash 提示（空文件、类型错误）**
```go
_ = addSessionFlash(app, w, r, fmt.Sprintf("%s was empty, import skipped", formFile.Filename), nil)
```
- 每个文件单独添加一条 Flash
- 在模板中也通过 `.Flashes` 渲染
- 与错误列表混合显示，没有视觉区分

#### 6.4.4 成功/失败计数逻辑

```go
filesSubmitted := len(files)  // 提交文件总数
var filesImported int          // 成功导入计数
// ... 循环处理 ...
filesImported++  // 只有完全成功（通过 CreatePost）才递增
```

**计数规则：**
- 空文件 → `filesSubmitted` 计数，`filesImported` 不计数
- 类型错误 → `filesSubmitted` 计数，`filesImported` 不计数
- 其他错误 → `filesSubmitted` 计数，`filesImported` 不计数
- 成功导入 → 两者都计数

**最终状态判定：**
```go
if filesImported == filesSubmitted {
    // SUCCESS：全部成功
} else if filesImported > 0 {
    // INFO：部分成功
} else {
    // （无特殊提示）全部失败
}
```

---

## 七、账号搬迁建议流程

### 7.1 导出阶段（源实例 A）
1. 登录实例 A，访问 `/me/export`
2. 下载完整账号导出：`GET /me/export.json?pretty=1`（保留所有集合和文章结构）
3. 可选：下载 ZIP 格式备份便于人工阅读

### 7.2 导入阶段（目标实例 B）
**注意：当前实现没有直接支持 JSON 全量导入的 API！** 需要分步处理：

1. **注册账号**：在实例 B 上注册相同用户名的账号（或任意用户名）
2. **创建集合**：根据导出 JSON 中的 collections 数组，逐个创建博客集合
   - 集合别名（alias）需确保在目标实例上唯一
3. **批量导入文章**：
   - 将每篇文章转换为 `.txt` / `.md` 文件
   - 使用 `/api/me/import` 接口分批上传
   - 上传时指定对应 `collection` 参数
   - 确保 `fileDates` JSON 中包含原始创建时间戳

### 7.3 搬迁局限性

| 限制项 | 说明 |
|-------|------|
| **无全量 JSON 导入** | 当前仅支持纯文本/Markdown 文件导入，需自行编写转换脚本 |
| **ID 不保留** | 导入后文章 ID 重新生成，原 URL 会失效 |
| **Slug 可能变更** | 如目标集合已有同名 Slug，会自动追加后缀（如 `-2`） |
| **密码不迁移** | 需在新账号重新设置密码 |
| **关联数据丢失** | 浏览量、点赞数、评论、订阅者等统计数据不保留 |
| **OAuth 绑定** | 第三方 OAuth 关联需重新连接 |
| **集合设置** | 样式表、脚本、签名等自定义设置需手动重建 |
| **更新时间丢失** | `Updated` 字段重置为导入时间，原修改历史不可追溯 |
| **字体样式丢失** | 文章的 `text_appearance`（字体风格）统一重置为 `norm` |
| **置顶状态丢失** | `pinned_position` 不保留，需重新设置置顶 |

---

## 八、关键数据结构速查表

### SubmittedPost（导入提交）
[posts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/posts.go) L92-L100
```go
type SubmittedPost struct {
    Slug     *string                  // URL 路径（可选）
    Title    *string                  // 标题（指针以区分空值和未设置）
    Content  *string                  // 正文内容
    Font     string                   // 字体样式 (norm/mono/serif)
    IsRTL    converter.NullJSONBool   // 是否从右到左
    Language converter.NullJSONString // 语言代码
    Created  *string                  // 创建时间 ISO 8601
}
```

### User（用户对象）
[users.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/users.go) L66-L76
```go
type User struct {
    ID         int64       // 内部 ID（导出时隐藏，json:"-"）
    Username   string      // 用户名
    HashedPass []byte      // 密码哈希（导出时隐藏）
    HasPass    bool        // 是否已设置密码
    Email      zero.String // 加密后的邮箱
    Created    time.Time   // 创建时间
    Status     UserStatus  // 账号状态（0=正常 1=禁言）
}
```

### Collection（博客集合）
[collections.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/collections.go) L52-L75
```go
type Collection struct {
    ID           int64          // 内部 ID
    Alias        string         // 别名（URL 路径段）
    Title        string         // 显示名称
    Description  string         // 描述
    Public       bool           // 是否公开
    Views        int64          // 访问量
    OwnerID      int64          // 所有者用户 ID
    StyleSheet   string         // 自定义 CSS
    Script       string         // 自定义 JS
    Monetization string         //  monetization 指针
    Verification string         // 验证链接
}
```

---

## 九、导入流程时序图（简化）

```
用户浏览器                     后端服务                    数据库
   |                            |                          |
   |-- GET /me/import --------->|                          |
   |                            |-- GetCollections(u) ---->|
   |                            |<-- 集合列表 -------------|
   |<-- 导入页面（含集合下拉）--|                          |
   |                            |                          |
   |-- POST /api/me/import ---->|                          |
   |   (multipart/form-data)    |                          |
   |   files + collection +     |                          |
   |   fileDates(JSON)          |                          |
   |                            |                          |
   |                    遍历每个文件:                      |
   |                            |-- 临时文件写入磁盘       |
   |                            |-- wfimport.FromFile()    |
   |                            |   解析 Title/Content     |
   |                            |                          |
   |                            |-- 构造 SubmittedPost     |
   |                            |   + Created(来自fileDates)|
   |                            |                          |
   |                            |-- db.CreatePost() ------>|
   |                            |                          |-- INSERT
   |                            |                          |   (可能触发DuplicateKey
   |                            |                          |    重试 GenSafeUniqueSlug)
   |                            |<-- 成功/失败 ------------|
   |                            |                          |
   |              [成功] [可选] go federatePost() 异步分发 |
   |                            |                          |
   |                    循环直到所有文件处理完成            |
   |                            |                          |
   |                            |-- addSessionFlash ------>|
   |                            |   (SUCCESS/INFO/错误列表)|
   |<-- 302 Redirect /me/import|                          |
   |                            |                          |
   |-- GET /me/import --------->|                          |
   |<-- 结果页面（含Flash消息）-|                          |
```

---

## 十、安全与权限校验点

| 校验点 | 代码位置 | 校验规则 |
|-------|---------|---------|
| 集合所有权 | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L73-L77 | `coll.OwnerID == u.ID`，禁止向他人集合导入 |
| 登录态 | 路由层 `handler.User()` | 所有导入导出接口均需有效登录 |
| 文件大小 | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L59 | `r.ParseMultipartForm(10 << 20)` 限制 10MB |
| 邮箱加密存储 | [users.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/users.go) | 导出 JSON 中邮箱为加密值 |
| 密码排除 | [users.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/users.go) L69 | `HashedPass` 标记 `json:"-"`，导出不包含 |
