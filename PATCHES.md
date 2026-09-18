# PATCHES.md — 本 fork 相对上游的本地改动

- fork：`Punky971210/dsh-raw-html`（上游：`plolpl789/dsh-raw-html`）
- 基线：上游 commit `ab1f1d5`（2026-08-28）
- 目的：让插件自带「内联图片 + 一键下载」所需的**文件字节通道**，替代已弃用的自建文件桥 `dsh-file-portal`

---

## P-1 本地文件服务路由（`lib/index.js`）

**位置**：`apply()` 内、`// --- loopback RPC` 注释之前的一整段；注册调用在既有
`ctx.inject(['connection'], …)` 回调的**第一行**（`installLocalFileRoute(connection)`）。

**新增**：
- 常量 `FILE_ROUTE = '/api/dsh-raw-html/file'`、`FILE_MAX_BYTES = 20 MiB`、`FILE_MIME` 表
- `fileAsciiName()` / `fileAllowedRoots()` / `serveLocalFile()` / `installLocalFileRoute()`

**行为**：
- `GET|HEAD /api/dsh-raw-html/file?path=<绝对路径>[&inline=1]`
- 白名单：宿主已注册的各**会话工作区目录**（`ctx.workspaceRegistry.list()` → `record.path`），
  经 `realpath` 归一后前缀比对，防 `..` 与符号链接逃逸
- 上限 20 MiB（对齐宿主 `attachments.imageLimits.maxImageBytes`）
- 默认 `Content-Disposition: attachment`（真下载；文件名 `filename="ASCII 回退"` + `filename*=UTF-8''真名`）；
  `?inline=1` 改 inline 供预览

**为什么 path 必须在 `/api` 之下**（改动前必读，否则会掉进无鉴权陷阱）：
宿主的 `connection.fetch.register` 只接受 `/api` 前缀路径（`assertFetchRoute` 用
`endpointFromPath('/api', route.path)` 校验），这些 exact Fetch route 由挂在 `/api`
前缀上的共享 handler 分发，而那层 handler 带 **Host/Origin fence + 浏览器 cookie 鉴权**
（`dsh-client-connection/lib/index.js:767-781`）。宿主自己的 `/api/file` 就是同一机制。
反过来，`webServer.register` 注册的裸路由（插件的 `/fonts`、`/vendor`）**没有任何鉴权**——
绝不可用它注册文件读取。

**扩展点（待办，不属本次改动）**：token / 一次性核销 / 配额 / 上传口，统一挂本路由。

---

## P-1 修订（2026-09-18 · 通路更换；上文第 26–33 行的通路说明已被实测证伪，以本节为准）

**原实现**：`connection.fetch.register`（exact Fetch route，path 必须在 `/api` 之下，
靠 `/api` 前缀路由上的 Host/Origin fence 与 cookie 鉴权）。

**实测：该通路对本插件不可用。** 加诊断日志后重启复现：

```
[raw-html] file route 注册成功: /api/dsh-raw-html/file | route 表命中=true | route 表大小=3
[raw-html] file route 延迟回读(5s): 命中=false 大小=5          ← 注册被清理
```

页面内请求恒 404，响应体是 connection shared handler 的兜底文案 `"not found"`（9 字节）；
同一页面对宿主自带的 `/api/file` 请求返回 200 —— 说明分发机制正常，被清理的只是本插件这条
（宿主那条由 session-controller 在内核编排期注册，故稳定）。

**现实现**：改用插件自己的 `webServer.register({kind:'prefix'})`（与 `/fonts`、`/vendor`
同处一个 effect，生命周期随插件），鉴权移到 handler 内复用 `connection.requestRejection(req)`
（与 `/api` 同一套 Host/Origin fence + 浏览器 cookie 校验）；取不到 connection 服务时一律 **401**，
绝不裸放行。connection 在**请求时**动态取（`ctx.get('connection')`），不依赖注入时机。

**路由随之变更**：`/api/dsh-raw-html/file` → **`/dsh-raw-file`**（避开 RPC 通道前缀
`/dsh-raw-html` 与 `/api` 命名空间）。

**重启后零依赖判据**：`GET /dsh-raw-file?path=x`（无 cookie）应返回 **401**（= 已注册且鉴权在拦）；
返回 404 即未注册。带 cookie 的页面内请求才是 200 / 403 / 404 / 413 四态。
**插件文件改动必须重启宿主**；前端补丁改动只需刷新页面 —— 两者不要混。

---

## 为什么需要这个改动（必要性说明）

本改动**不是功能必需**——零开发方案（宿主 `GET /api/file?path=<绝对路径>` + HTML
`<a href="同源 URL" download="文件名">`）已经能完成「内联显示 + 一键下载」，且已实测闭环。
加它的理由按重要性排序：

1. **白名单收敛（唯一实质安全收益）**：宿主 `/api/file` 的模块注释自陈
   「Paths and MIME types do not restrict access」——无扩展名限制、无工作区包含检查，
   任何已登录页面可读全盘；本路由把可读范围收敛到**宿主已注册的会话工作区目录**，
   越界一律 403。
2. **服务端下载语义**：`Content-Disposition: attachment` 由服务端给出，不依赖 HTML
   `download` 属性（后者仅同源有效，且语义停留在前端）。
3. **未来扩展的落点**：token / 一次性核销 / 配额 / 上传口全部挂本路由即可，
   不必再改宿主或新开插件。

**代价与边界（同样要知道）**：
- host 侧改动**必须重启宿主**才生效（前端补丁不受影响，刷新即可）；
- 与上游分叉扩大，故 P-1 集中为一段、附 rebase 步骤；
- 若某个会话工作区本身是宽目录（例如整盘为工作区），白名单会退化为「等于没有」，
  此时应改为显式配置的窄根；
- 不使用时的回退：`git revert` P-1 对应提交即可，本路由与前端渲染补丁**互不影响**。

## 决策留痕

| 日期 | 决策 | 依据 |
|---|---|---|
| 2026-09-18 | **保留本路由**（用户裁决，选项 A） | 白名单策略取「会话工作区目录」；单文件上限取 20 MiB（对齐宿主 `attachments.imageLimits.maxImageBytes` 默认值） |
| 2026-09-18 | 暂不实现 token / 核销 / 配额 / 上传口 | 留待 SaaS 化时再议，届时挂同一路由 |
| 2026-09-18 | **批量打包下载（zip / 目录递归）暂不跟进** | 用户裁决：批量**展示**链接已可用（本机 `probes/export-links.mjs`），打包等有具体需求再议；外部下载器路径因无 cookie 恒 401，须先 token 化 |

---

## 打包下载：暂不跟进（用户裁决 2026-09-18）

| 能力 | 状态 |
|---|---|
| 单文件下载（任意类型 · ≤20 MiB · 工作区内） | ✅ 可用 |
| 图片内联显示 + 下载按钮 | ✅ 可用 |
| **批量展示多个下载按钮** | ✅ 可用（`probes/export-links.mjs` / `file-card.mjs`） |
| **批量打包下载（zip / 目录递归）** | ⛔ 暂不实现（等具体需求） |
| **外部下载器批量拉（IDM / 迅雷 / aria2）** | ⛔ 当前不可能（无浏览器 cookie ⇒ 401） |

**为什么外部下载器被否**：鉴权 = 浏览器 cookie（`dsh-auth-<sha256(authority)>`），外部下载器没有 ⇒
请求一律 **401**（命令行无 cookie 实测即 401）。要支持必须让 URL 自带凭据（**token 化**），
与 token / 核销 / 配额 / 上传口同属一批待办。

**若重启此需求，按此执行**（要点存档，避免重新论证）：
- 入口 `GET /dsh-raw-file?zip=1&path=<文件或目录>[&path=…]`；stored（不压缩）起步，需要时换 `zlib.deflateRawSync`
- 三道上限：总量（建议 200 MiB）/ 条目数（500）/ 递归深度（8）
- 安全：逐项 realpath 校验在工作区内、**跳过符号链接**、拒绝 `..`
- 硬约束：多文件只能靠重复 `path` 参数（接近 URL 长度上限）⇒ 批量必须走**目录模式**
- 估量：多文件 ~120–180 行 + 目录递归 ~60–100 行 + 卡片/测试

---

## P-2 可发现性：常驻能力段 + `file_download_link` 工具（2026-09-18）

**问题**：下载能力的说明原本写在 VCP 协议段内，而那段以 `render` 开关为条件
（`text: () => render ? 协议全文 : DISABLED_TEXT`）⇒ **关掉渲染就无人知晓**。
实测：`render` 默认 `false` 且状态文件不存在 ⇒ 该实例上**所有会话的 agent 都看不到这个能力**。

**改法**（两条互补）：
1. **常驻段**：`systemPrompt.section({ name: 'raw-html:file-download', order: 210, text: () => FILE_CAPABILITY_TEXT })`
   —— 不受 `render` 约束，每轮注入，任何会话都能看到；
2. **工具**：`file_download_link`（入参 `path` / `inline` / `origin`）→ 返回**可粘贴的纯文本 URL**，
   **不依赖 HTML 渲染开关**；已校验存在性 + 会话工作区白名单 + 20 MiB 上限。`inject` 增加 `tools`。

### ⚠️ 事故与红线（务必遵守）：第三方 import 会让整个插件加载失败

首次实现时在文件顶部写了 `import { defineTool } from '@deepseek-ai/dsh-tools'`，实机启动**直接崩**：

```
Error: dsh: plugin tree failed to load: ... Cannot find package '@deepseek-ai/dsh-tools'
       imported from D:\dsh\dsh-raw-html\lib\index.js
[ERR_MODULE_NOT_FOUND]
```

根因：本项目从 git 获取，**分发形态没有 `node_modules`**，ESM 从 `lib/index.js` 向上逐级找
`node_modules` 全部落空。这正是上游 README「依赖声明铁律」警告过的情形。
**危险性**：插件加载失败会让**整个 plugin tree 起不来**（不是单个工具不可用）——
若在役宿主此时重启，会直接起不来。

**现约定**：
- `@deepseek-ai/dsh-tools` 只用 **动态** `import()` 解析，失败则降级为「不注册工具」，
  插件其余功能（渲染、字体、文件路由）照常工作；
- `apply` 因此是 `async`（cordis 支持）；
- 本机若要让工具真正注册，需保证该包可解析：本机做法是在插件目录下建
  `node_modules/@deepseek-ai/dsh-tools` → junction 指向宿主内核的同名包
  （已被 `.gitignore` 忽略，不进仓库）；
- 新增任何第三方 `import` 前，先跑一次 `node -e "import('file:///.../lib/index.js')"` 自检。

---

## 如何合并上游更新

```powershell
cd D:\dsh\dsh-raw-html
git fetch upstream
git rebase upstream/main      # 冲突只可能落在 P-1 段落；该段落自成一块，便于整体取用
git push origin main --force-with-lease
```
改上游文件时请保持 P-1 **集中在一处**并保留顶部契约注释块，便于 rebase 时整段搬运。

## 与前端补丁的关系（不在本仓库内）

前端渲染补丁（本机 DSH 0.1.6-alpha.1 的适配版）由本机脚本
`D:\AI_Workspace\DSH\DSH\probes\raw-html-016-adapter.mjs` 施加，作用于
`dsh-web-frontend/dist/assets/index-*.js`；该补丁与本 fork 的 P-1 互相独立：
- 补丁负责「HTML 能被渲染」⇒ 卡片里的 `<img>` / `<a download>` 才能成立
- P-1 负责「文件字节可取」⇒ 下载有服务端语义（Content-Disposition）
两者都不在岗位时可分别回滚。

### 前端补丁含「非可信内容拦截」（2026-09-18 增补）

同一适配器（`raw-html-016-adapter.mjs`）另注入一段**非可信标签拦截**到宿主 DOM→vdom 函数
（`C8`）体首，形态对齐上游 `patch/trusted-patch.cjs` 的**条件式**：`window.__vcpTrusted()`
为真（可信模式开启）才放行，默认一律丢弃该节点及其子树。

丢弃清单：`script` / `iframe` / `object` / `embed`（与上游 SCRIPT_FILTER 一致）
+ `link` / `base` / `meta`（本地扩展）。

依据（实测）：`<script>` 经 React 的 createElement 路径**会真实执行**（同源任意 JS）；
`<iframe>` 挂载即加载外部页面；`<base>` 可篡改卡片内相对 URL 的解析基准；
`<meta http-equiv="refresh">` 可用于跳转；`<link rel=stylesheet>` 可外联 CSS。
`<style>` 予以保留（卡片样式一律内联，且其 `@import` / `url()` 外联面属已知剩余项）。

复验脚本：`D:\AI_Workspace\DSH\DSH\probes\cdp-tag-filter-probe.mjs`
（期望：七类标签在 vdom 内均为 `null`，而 `<style>` 保留、`javascript:` href 与 `file://` src 仍被剥离）。
