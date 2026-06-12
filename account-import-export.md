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

## 六、账号搬迁建议流程

基于代码分析，推荐的账号搬迁（从实例 A 迁移到实例 B）流程：

### 6.1 导出阶段（源实例 A）
1. 登录实例 A，访问 `/me/export`
2. 下载完整账号导出：`GET /me/export.json?pretty=1`（保留所有集合和文章结构）
3. 可选：下载 ZIP 格式备份便于人工阅读

### 6.2 导入阶段（目标实例 B）
**注意：当前实现没有直接支持 JSON 全量导入的 API！** 需要分步处理：

1. **注册账号**：在实例 B 上注册相同用户名的账号（或任意用户名）
2. **创建集合**：根据导出 JSON 中的 collections 数组，逐个创建博客集合
   - 集合别名（alias）需确保在目标实例上唯一
3. **批量导入文章**：
   - 将每篇文章转换为 `.txt` / `.md` 文件
   - 使用 `/api/me/import` 接口分批上传
   - 上传时指定对应 `collection` 参数
   - 确保 `fileDates` JSON 中包含原始创建时间戳

### 6.3 搬迁局限性

| 限制项 | 说明 |
|-------|------|
| **无全量 JSON 导入** | 当前仅支持纯文本/Markdown 文件导入，需自行编写转换脚本 |
| **ID 不保留** | 导入后文章 ID 重新生成，原 URL 会失效 |
| **Slug 可能变更** | 如目标集合已有同名 Slug，会自动追加后缀（如 `-2`） |
| **密码不迁移** | 需在新账号重新设置密码 |
| **关联数据丢失** | 浏览量、点赞数、评论、订阅者等统计数据不保留 |
| **OAuth 绑定** | 第三方 OAuth 关联需重新连接 |
| **集合设置** | 样式表、脚本、签名等自定义设置需手动重建 |

---

## 七、关键数据结构速查表

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

## 八、导入流程时序图（简化）

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

## 九、安全与权限校验点

| 校验点 | 代码位置 | 校验规则 |
|-------|---------|---------|
| 集合所有权 | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L73-L77 | `coll.OwnerID == u.ID`，禁止向他人集合导入 |
| 登录态 | 路由层 `handler.User()` | 所有导入导出接口均需有效登录 |
| 文件大小 | [account_import.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/account_import.go) L59 | `r.ParseMultipartForm(10 << 20)` 限制 10MB |
| 邮箱加密存储 | [users.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/users.go) | 导出 JSON 中邮箱为加密值 |
| 密码排除 | [users.go](file:///d:/fz/0601-1/solo-dogfeeding/code/37-writefreely/users.go) L69 | `HashedPass` 标记 `json:"-"`，导出不包含 |
