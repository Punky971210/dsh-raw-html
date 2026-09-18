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

**能力位前置**：宿主未提供 `connection.fetch.register` 时静默跳过（不抛错、不影响渲染与字体功能）。

**扩展点（待办，不属本次改动）**：token / 一次性核销 / 配额 / 上传口，统一挂本路由。

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
