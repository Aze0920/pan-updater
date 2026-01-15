# 在线网盘（PHP 单文件后台 + 分享/直连/API）

> 这是一个轻量级的 PHP 网盘/文件发布系统：**后台登录后管理文件**，对外提供 **分享链接**、**直连下载**（仅文件）、以及 **API 元信息接口**。  
> 重点安全策略：**强制首次初始化（根目录/密钥/密码）**、下载走 **签名 token**、目录访问做 **路径安全校验**，并建议把落盘目录放到**站点目录外**，从根源避免 URL 直链绕过。

---

## 目录与入口说明

- `index.php`
  - 登录页入口（未登录只显示登录页）
  - **首次初始化**（强制弹窗）与设置页逻辑也在这里
  - 已登录访问 `index.php` 会自动跳转到 `app.php`
- `app.php`
  - 主入口（定义 `APP_ENTRY` 后 `require index.php` 输出主页面）
- `functions.php`
  - 通用函数库：安全校验、目录扫描、分享 token、share_links 读写、统计、设置加密等
- `share.php`
  - 公开分享页（无需登录）：支持文件/文件夹分享；文件夹分享展示文件卡片列表
- `share_download.php`
  - 公开下载入口（无需登录）：所有下载都走这里（签名 token 校验 + 统计）
- `api.php`
  - 公开 API（无需登录）：通过分享短码输出 JSON 元信息（标题/大小/更新时间/简介）
- `config.php`
  - 基础配置（部署时可改），但**关键项会被 settings.json 覆盖**
- `settings.json`
  - 本地可变配置（首次初始化写入）：`root_dir`、`auth_password_hash`、`download_secret_enc` 等
- `settings.key`
  - **自动生成**的本地密钥文件（用于加密 `settings.json` 的敏感字段）
  - 请勿泄露、请勿删除（删除会导致解密失败；系统会重新生成，但旧加密字段无法还原）
- `share_links.json`
  - 分享映射与统计存储（短码→token、以及访问/下载次数）
- `recycle_bin/`
  - 回收站目录（删除/覆盖会先移到这里，支持恢复/清空）

---

## 功能清单（你现在这套代码已具备）

### 1) 登录与权限

- **后台访问密码**：后端 `session` 鉴权，密码只保存哈希（`password_hash`）。
- **CSRF 防护**：所有会修改状态的 POST 都校验 CSRF token。
- **首次强制初始化**（最重要）：
  - 必须设置 `root_dir`（落盘根目录）
  - 必须设置 `download_secret`（分享/下载签名密钥，>=32 位）
  - 必须修改默认密码（默认 `123456` 不允许继续使用）
  - 在未完成初始化前：上传/建目录/分享等功能被禁用（只允许完成初始化/改密等）

### 2) 文件管理（后台）

- **上传文件**：支持拖拽上传/选择上传
  - 最大上传大小遵循服务器 `upload_max_filesize` 与 `post_max_size`（取较小值）
  - 同名文件默认覆盖：覆盖前会把旧文件移入回收站
- **文件下载（后台）**：后台页面下载也走受控路径（不会直接暴露真实服务器路径）
- **新建文件夹**
- **文件/文件夹重命名**
- **移动文件到其它目录**
- **删除文件/文件夹**
  - 文件删除：移入回收站
  - 文件夹删除：仅允许删除空文件夹（避免误删大量文件）

### 3) 回收站

- 删除/覆盖的旧文件进入 `recycle_bin/`
- 支持：
  - 恢复到原目录
  - 永久删除
  - 一键清空
- 支持保留天数（`recycle_days`），系统会自动清理过期文件

### 4) 分享链接（share）

- **文件分享**：生成分享页短码 `share.php?c=xxxx`
  - 支持设置：标题、简介、有效期、密码
  - 支持“保存设置不换链接”（不会重新生成短码）
- **文件夹分享**：生成分享页短码 `share.php?c=xxxx`
  - **不提供直连**（这是你明确要求的）
  - 分享页展示：标题、简介、文件卡片列表（可逐个下载）
  - 支持密码/有效期
- 分享统计：
  - 文件/文件夹分享：有 **访问次数** + **下载次数**
  - 访问统计在 `share.php` 页面成功展示后累加
  - 下载统计在 `share_download.php` 成功输出文件后累加

### 5) 分享直连（direct，仅文件）

- 仅文件支持直连（文件夹不支持）
- 直连入口是 `share_download.php?c=xxxx`
- 支持有效期（到期自动失效）
- 直连统计：只有 **下载次数**

### 6) API（无需登录）

为每个文件/文件夹提供独立的 API 元信息接口（你新增的需求）：

- 后台列表每个对象有一个 **「API」按钮**：
  - 若该对象已经有分享短码：直接复制 `api.php?c=<分享短码>`
  - 若没有分享：自动创建一个默认分享（永久/无密码/标题=文件名），再复制 API 地址

API 返回字段（JSON）：

- `title`：标题
- `desc`：简介
- `size_bytes`：大小（文件=文件大小；文件夹=递归统计到一定上限）
- `updated_at`：更新时间（`Y-m-d H:i:s`）

密码分享的 API 调用：

- 若分享设置了密码：需要追加 `p=密码`
  - 示例：`api.php?c=xxxx&p=1234`

---

## 首次部署与使用（一步一步来）

### 1) 准备运行环境

推荐：

- PHP **7.4+**（8.x 也可以）
- 必需扩展/函数：
  - `openssl`（用于 `settings.json` 密钥加密存储）
  - `fileinfo`（用于下载时识别 MIME，非强制但建议开启）
  - `session`（默认都带）

> 如果 `openssl` 不可用：首次初始化保存密钥时会失败（因为需要写入 `download_secret_enc`）。

### 2) 放置项目文件

把这些文件放到站点目录（例如 `public/` 或 `wwwroot/`）：

- `index.php`、`app.php`、`functions.php`、`share.php`、`share_download.php`、`api.php`
- `config.php`
- `style.css`、`script.js`
- `recycle_bin/`（可为空目录）

### 3) 配置 `config.php`（建议先改）

`config.php` 返回一个数组（`return [ ... ];`），常用项：

- **download_secret（必填，固定不变）**
  - 用于分享 token/下载 token 的签名验签
  - 必须是 **32 位以上随机字符串**
  - **不要频繁更改**：更改会导致旧 token 失效（虽有兼容旧字段逻辑，但不建议依赖）

- **auth_password_hash（默认是 123456 的 hash）**
  - 仅用于第一次启动时的默认值
  - 首次初始化会强制你改掉 123456，并写入 `settings.json` 覆盖

- **root_dir**
  - 这里默认是空：表示**必须首次初始化时由用户设置**

其它常用项：

- `recycle_bin_dir`：回收站目录（默认 `./recycle_bin/`）
- `log_file`：日志文件（默认 `./file_manager.log`）
- `recycle_days`：回收站保留天数
- `dir_permissions` / `file_permissions`：新建目录/文件权限（Linux 下有效，Windows 可能忽略）
- `enable_folder_nesting`：是否允许 `123/456` 这种嵌套目录访问（默认 true）

### 4) 访问站点并登录

- 打开 `index.php`（或站点根域名）
- 输入默认密码：`123456` 登录

### 5) 首次初始化（必须完成，否则无法使用）

登录后会强制弹出设置框，要求一次性填写并提交：

- **根目录 root_dir（必填）**
  - 这是文件真实落盘目录（上传文件都会写到这里）
  - **强烈建议放到“站点目录外”**（例如 `../data/`），这样用户无法通过 URL 直接访问真实文件
  - Windows 也支持绝对路径：例如 `D:\pan\data\`
  - 如果你填在站点目录内：必须配合 Nginx/IIS/Apache 禁止直链访问（下面有模板）

- **下载/分享密钥 download_secret（必填，>=32 位）**
  - 用于生成/校验分享 token
  - 首次初始化会把它**加密写入** `settings.json` 的 `download_secret_enc`
  - 同时生成/使用 `settings.key`（AES-256-GCM）
  - **一旦设置后将锁定不可更改**（防止误操作导致历史链接全部失效）

- **新密码 & 确认新密码（必填）**
  - 必须 != `123456`，长度 >= 6

初始化成功后：

- 自动跳转到 `app.php`
- 会尝试删除占位目录 `__not_configured__`（保持目录干净）

---

## 日常使用指南（后台）

### 1) 进入后台

- 正常访问站点：走 `index.php` 登录页
- 登录成功后会跳到 `app.php`（主页面）

### 2) 文件/文件夹操作

在列表中每个文件/文件夹右侧都有操作按钮（不同对象按钮略有差异）：

- **打开**：进入子文件夹（仅文件夹）
- **下载**：下载文件
- **分享**：打开分享弹窗（文件/文件夹均支持）
- **直连**：仅文件支持（在分享弹窗里）
- **API**：复制外网可调用的 JSON 元信息接口（文件/文件夹均支持）
- **重命名**：文件/文件夹都支持
- **移动**：仅文件支持
- **删除**：文件删除进回收站；文件夹仅允许删除空目录

### 3) 回收站

点击「回收站」可看到删除/覆盖产生的旧文件，支持：

- 恢复（按记录的原目录信息恢复）
- 永久删除
- 清空回收站

> 回收站文件名包含原目录信息（用于恢复），无需你手工关心。

### 4) 系统日志

后台提供日志查看：

- 记录登录、分享生成/取消、设置变更、上传、删除等
- 支持分页查看、导出、清空

---

## 分享功能详解（重点）

### A. 分享数据存在哪？

分享相关数据统一存放在 `share_links.json`，结构大概是：

- `shares`：分享页短码（`share.php?c=`）记录
- `directs`：直连短码（`share_download.php?c=`）记录
- `active_share` / `active_direct`：保证同一对象同一时间只保留一个“有效短码”
- 每条记录会保存：
  - token（签名后的 payload）
  - 标题、简介、有效期、是否带密码、pw_hash
  - `views` / `downloads` 统计（share 有 views+downloads；direct 只有 downloads）

系统会自动清理 `share_links.json`：只保留 active 指向的有效记录，避免无限膨胀。

### B. 文件分享（share.php）

分享地址：

- `share.php?c=<短码>`

支持：

- 设置标题/简介
- 设置有效期（0=永久）
- 可选密码
- **保存设置不换链接**：修改标题/简介/有效期/密码都不会生成新短码

### C. 文件夹分享（share.php）

同样是：

- `share.php?c=<短码>`

特点：

- 分享页展示该文件夹下的文件卡片列表
- 可逐个点击下载（下载入口仍走 `share_download.php?d=<token>`）
- **不会提供“直连分享”**（满足你之前的需求）

### D. 直连下载（share_download.php）

直连地址（后台生成，短码）：

- `share_download.php?c=<短码>`

下载 token（临时 token，不一定落盘）：

- `share_download.php?d=<token>`
  - 这个 token 可能由 `share.php` 临时生成（文件夹分享页里“下载某个文件”时会生成）
  - 因为是临时 token，所以**不要求必须存在 share_links.json**，只要签名正确即可下载

统计规则：

- `share_download.php` 成功输出文件后：
  - 如果属于分享页下载（token payload 带 `sc`）：累加 `shares[sc].downloads`
  - 如果属于直连短码：累加 `directs[code].downloads`

---

## API 使用说明（外网调用）

### 1) API 地址如何获取？

后台列表每个对象右侧有「API」按钮：

- 若对象已有分享短码：直接复制 `api.php?c=<短码>`
- 若没有：自动创建一个默认分享（永久/无密码/标题=文件名），再复制 API 地址

### 2) API 返回格式

GET：

- `api.php?c=<短码>`

返回 JSON：

- `title`：标题
- `desc`：简介
- `size_bytes`：大小（字节）
- `updated_at`：更新时间（字符串）

辅助字段：

- `ok`：成功/失败
- `is_dir`：是否文件夹
- `code`：短码
- `updated_ts`：更新时间戳

### 3) 带密码的分享如何调用 API？

如果分享设置了密码，API 需要带参数：

- `api.php?c=<短码>&p=<密码>`

否则返回：

- HTTP 401 + `{ ok:false, error:"需要密码" }`

---

## 安全建议与服务器配置（非常重要）

### 最佳方案：root_dir 放到站点目录外

强烈建议把 `root_dir` 设置为：

- Linux：`../data/` 或 `/data/pan/`
- Windows：`D:\pan\data\`

这样用户无法通过 `http(s)://domain/...` 直接访问真实文件，必须走 PHP（可做验签/统计/作废/密码）。

### 如果你必须把 root_dir 放站点目录内（不推荐）

请务必在 Web 服务器层禁止访问该目录，否则可能出现你之前提到的：

> 用户直接访问 `domain/上传目录/xxx` 就能下载，绕过分享体系。

#### Nginx（示例）

假设你把文件落在站点目录的 `/uploads/`：

```nginx
location ^~ /uploads/ {
    deny all;
    return 403;
}
```

如果你不知道真实 URL 路径，对应的原则是：**把落盘目录映射到 URL 后能直接访问的那一段全部 deny**。

#### Apache（示例）

在落盘目录内放 `.htaccess`（系统也会尽量自动生成）：

```apache
Options -Indexes
<IfModule mod_authz_core.c>
  Require all denied
</IfModule>
<IfModule !mod_authz_core.c>
  Deny from all
</IfModule>
```

#### IIS（示例）

在落盘目录内放 `web.config`（系统也会尽量自动生成）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <security>
      <authorization>
        <remove users="*" />
        <add accessType="Deny" users="*" />
      </authorization>
    </security>
  </system.webServer>
</configuration>
```

### 其它安全点（代码层已做）

- `sanitize_folder_path` / `is_file_in_directory`：防目录遍历、限制只能访问 root_dir 内
- `X-Content-Type-Options: nosniff` 等安全响应头（HTML/下载/JSON）
- 分享 token HMAC-SHA256 签名，防篡改
- `download_secret` 固定，避免出现 “Invalid token（secret mismatch）”

---

## 常见问题（FAQ）

### Q1：为什么第一次必须设置 root_dir？

因为默认目录（例如以前的 `2/`）非常危险：

- 很容易被 Web 服务器当作静态目录直链访问
- 会导致你分享系统/权限系统形同虚设

强制初始化能把“必需安全项”一次性卡住，避免上线后再补洞。

### Q2：为什么 download_secret 一旦设置就不可更改？

因为它决定了所有分享/下载 token 的验签：

- 你一改密钥，历史 token 全部验签失败，用户会看到 Invalid token
- 为了避免误操作导致大面积事故，所以代码把它锁死

### Q3：上传大小怎么改？

本系统会使用服务器限制：

- `upload_max_filesize`
- `post_max_size`

请在 `php.ini` 或面板里改这两个，并重启 PHP-FPM/服务。

### Q4：分享文件夹能不能像网盘一样层层点进去？

目前分享页的定位是：

- 文件夹分享页展示该文件夹下的文件列表（并支持下载）
- 如果要做“分享页内可浏览子文件夹/面包屑跳转”，需要在 `share.php` 增加目录浏览路由与二次校验逻辑（后续可扩展）

---

## 你需要自定义的内容（品牌信息）

按你的要求：品牌信息写死在 `index.php`（不走 `settings.json`），防止加密/壳后被篡改：

- `$APP_VERSION`
- `$BRAND_LOGO_URL`
- `$BRAND_NAME`
- `$BRAND_INTRO`
- `$BRAND_LINKS`（QQ/GitHub 等）

修改后刷新页面即可生效。


