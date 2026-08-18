# Sub-Store Workers 项目总图

> 生成时间: 2026-04-23 15:11（最近更新: 2026-04-25）
> 项目: sub-store-workers
> 类型: project-level codemap

## 1. 项目定位

将 [Sub-Store](https://github.com/sub-store-org/Sub-Store) 后端从 Node.js 移植到 Cloudflare Workers 运行时。**仅替换平台相关层，核心业务逻辑零修改**（构建时从 `../Sub-Store/backend/src/` 引入）。

## 2. 架构总览

```
┌─────────────────────────────────────────────────────┐
│              Cloudflare Workers Runtime              │
│                                                      │
│  ┌──────────┐   ┌──────────────┐   ┌──────────────┐ │
│  │ index.js │──▶│ express.js   │──▶│ 上游 restful │ │
│  │  入口     │   │ 路由适配层    │   │ 路由处理器    │ │
│  └────┬─────┘   └──────────────┘   └──────────────┘ │
│       │                                              │
│  ┌────▼─────┐   ┌──────────────┐   ┌──────────────┐ │
│  │open-api  │──▶│ KV Namespace │   │ esbuild.js   │ │
│  │存储适配层 │   │ (持久化)      │   │ 构建桥接      │ │
│  └──────────┘   └──────────────┘   └──────────────┘ │
└─────────────────────────────────────────────────────┘
```

## 3. 文件清单与职责

### 3.1 平台适配层（`src/`，本项目自有）

| 文件 | 职责 |
|---|---|
| `src/index.js` | **Workers 入口**：CORS 预检、KV 绑定校验、路径前缀鉴权（含未配置警告头）、KV 初始化、路由分发、Cron 定时同步 |
| `src/vendor/express.js` | **Express 适配**：将 Workers fetch handler 适配为类 Express 的 req/res 路由 |
| `src/vendor/open-api.js` | **OpenAPI 适配**：KV 替代 fs 存储、fetch 替代 undici、日志/通知/推送 |
| `src/core/app.js` | 单例 `$` 导出（`new OpenAPI('sub-store')`） |
| `src/utils/env.js` | 环境检测变量，`backend = 'Workers'`，`isWorker = true` |
| `src/restful/miscs.js` | 工具 API：`/api/utils/env`（暴露 `SUB_STORE_*` 给前端）、`/api/utils/worker-status`（KV/鉴权/能力诊断）、Gist 备份/还原、存储管理、刷新 |
| `src/restful/token.js` | Token 签发/删除（Workers 版，替换上游 JWT 方案为 nanoid） |
| `src/vendor/quickjs-executor.js` | **QuickJS 脚本沙箱**：替代上游 `new Function()` 执行用户脚本（Script Operator/Filter），支持 func / nodeFunc / content 三种模式，详见 4.4 |

### 3.2 构建层

| 文件 | 职责 |
|---|---|
| `esbuild.js` | **构建脚本**：4 个 esbuild 插件桥接 Workers 与上游源码；同时把上游 `createDynamicFunction` 替换为 QuickJS 沙箱执行（`SCRIPT_ENGINE=disabled` 可关闭） |
| `wrangler.toml` | Workers 部署配置：KV 绑定、Cron、环境变量（路径密码推荐改用 Worker Secret） |
| `package.json` | 依赖与脚本（`deploy`、`deploy:pages`、`rotate-secret`、`rotate-secret:sh`） |

### 3.3 运维脚本（`scripts/`）

| 文件 | 职责 |
|---|---|
| `scripts/rotate-secret.ps1` | Windows PowerShell：生成随机 URL-safe 密码，通过管道写入 Cloudflare Worker Secret `SUB_STORE_FRONTEND_BACKEND_PATH`，并复制到剪贴板 |
| `scripts/rotate-secret.sh` | Linux/macOS Bash：同上，自动选择 `pbcopy`/`wl-copy`/`xclip`/`xsel` |

### 3.4 上游源码（构建时引入，`../Sub-Store/backend/src/`）

通过 esbuild `@/` 别名解析：**Workers `src/` 优先 → 回退到上游 `src/`**。

关键上游模块：
- `restful/subscriptions` — 订阅 CRUD
- `restful/collections` — 组合订阅
- `restful/artifacts` — 制品生成 + Gist 同步
- `restful/download` / `restful/preview` — 分享链接（公开 API）
- `core/proxy-utils/` — 代理协议解析（Surge/Loon/QX peggy 语法）
- `utils/migration` — 数据迁移

## 4. 核心流程

### 4.1 HTTP 请求处理

```
Browser → fetch()
  │
  ├─ OPTIONS → 返回 CORS headers
  │
  ├─ KV 绑定校验（缺失 SUB_STORE_DATA → 500 明确错误）
  │
  ├─ 路径前缀鉴权（可选）
  │   ├─ 未配置 SUB_STORE_FRONTEND_BACKEND_PATH 且访问 /api/*
  │   │     → 控制台警告 + 响应头 X-Sub-Store-Security-Warning
  │   ├─ 已配置时：/api/* 无前缀 → 401
  │   ├─ /backendPath 精确 → 302 重定向
  │   └─ /backendPath/... → 剥离前缀
  │
  ├─ $.initFromKV(KV) → 加载缓存
  ├─ migrate() → 数据迁移
  ├─ $app.handleRequest(request) → Express 路由分发
  │   ├─ 中间件链
  │   ├─ dispatchRoute() → 最长匹配路由
  │   └─ 404 兜底
  │
  └─ ctx.waitUntil($.persistCache()) → 回写 KV
```

### 4.2 Cron 定时同步

```
scheduled() → cronSyncArtifacts()
  ├─ 检查 GitHub Token / artifacts
  ├─ 预生成订阅缓存（并行）
  ├─ 生成所有 artifacts（并行）
  ├─ syncToGist(files)
  ├─ 更新 artifact URL
  ├─ gistBackupAction('upload')
  └─ $.persistCache()
```

### 4.3 esbuild 构建管线

```
esbuild.js
  ├─ aliasPlugin: @/ → Workers src/ 优先，回退上游 src/
  ├─ peggyPrecompilePlugin: PEG 语法 → 预编译 JS 解析器
  ├─ evalRewritePlugin: eval() → Workers 兼容替换
  └─ nodeStubPlugin: Node 模块 → Proxy 存根
```

### 4.4 QuickJS 脚本执行（Script Operator/Filter）

上游用 `new Function()` 动态执行用户脚本，Workers 禁止运行时代码生成，改用 QuickJS WASM 沙箱（`src/vendor/quickjs-executor.js`）。按输入类型分三种模式：

```
createScriptFunction(script, name)
  ├─ func 模式：IIFE 包裹脚本 → 返回 function operator/filter → 调用
  ├─ 失败且输入是节点数组 → nodeFunc 模式：快捷脚本逐 $server 遍历
  └─ 失败且输入含 $content/$files（文件/覆写场景） → content 模式：
       ├─ mihomoConfig/mihomoProfile 文件：沙箱内取脚本的 main 函数，
       │    宿主侧 YAML 解析 $content → main(config) → YAML 序列化回 $content
       │    （YAML 在宿主侧处理，因沙箱内无 ProxyUtils）
       └─ 其他文件：注入 $content/$files 全局变量直接执行脚本，回读结果
```

> 背景：早期实现缺少 content 模式，$content 输入被当作节点数组遍历（对象无
> length，循环不执行），导致 convert.js 类覆写脚本不报错但输出空配置。

## 5. 数据存储

- **KV Key: `sub-store`** — 主缓存（订阅/组合/设置/tokens/artifacts）
- **KV Key: `root`** — 根数据
- **写入策略**：请求结束时 `persistCache()` 对比 snapshot，变化才写（防止无意义写入）
- **读取策略**：`cacheTtl: 60` 秒边缘缓存

## 6. 安全机制

- **路径前缀鉴权**：`SUB_STORE_FRONTEND_BACKEND_PATH`，**推荐用 Cloudflare Worker Secret 存放**（`wrangler secret put` 或 `npm run rotate-secret`），不要写在 `wrangler.toml [vars]` 里——后者明文存仓库且会被 `wrangler deploy` 覆盖同名 Secret。
- **未配置鉴权告警**：未配置时管理 API 访问会输出控制台警告并附加响应头 `X-Sub-Store-Security-Warning`，提醒部署者尽快设置密码。
- **公开路径白名单**：`/api/download`、`/api/preview`、`/api/sub/flow` 不受鉴权限制
- **CORS**：全局 `Access-Control-Allow-Origin: *`
- **Script Operator 沙箱化**：构建期改写 `createDynamicFunction` 为 QuickJS WASM 沙箱执行（内存/栈/指令数限制），避免 `eval`/`new Function`；可用 `SCRIPT_ENGINE=disabled` 关闭。
- **状态自检**：`/api/utils/worker-status` 输出 KV 绑定、鉴权、能力降级（脚本/socks/本地文件系统/cron）等运行时信息，方便部署后快速验证。

## 7. 外部依赖

| 依赖 | 用途 |
|---|---|
| `peggy` | PEG 语法编译（构建时） |
| `js-base64` | Base64 编解码 |
| `nanoid` | Token 生成 |
| `ms` | 时间字符串解析 |
| `lodash` | 工具函数 |
| `ip-address` | IP 地址处理 |
| `static-js-yaml` | YAML 解析 |
| `esbuild` | 构建（dev） |
| `wrangler` | 部署 CLI（dev） |

## 8. 已知风险与注意事项

1. **并发安全**：Workers 无状态但 `$.cache` 是全局 in-memory 对象，高并发下可能 read-modify-write 竞争
2. **KV 最终一致性**：KV 读有 60s cacheTtl，写入后可能延迟可见
3. **上游兼容性**：上游 Sub-Store 更新可能引入 Node-only API，需要构建时通过存根或适配处理
4. **CPU 时限**：Workers 免费版 10ms CPU / 请求，复杂订阅处理可能超时
5. **全局状态污染**：`globalThis.__workerEnv` 在并发请求间共享，理论上存在竞态
6. **Pages 与 Workers 配置不互通**：`wrangler.toml` 的 `[vars]`/`[[kv_namespaces]]` 不影响 Pages 项目，需要在 Cloudflare Dashboard 单独绑定 KV 与设置 `SUB_STORE_FRONTEND_BACKEND_PATH`（建议设为 Pages Secret）。
7. **`[vars]` 与 Worker Secret 同名冲突**：若同时存在，`wrangler deploy` 会用 `[vars]` 明文覆盖 Secret，破坏 CI Secret 管理流程；只用其中一种。
8. **路径密码不可恢复**：Worker Secret 在 Dashboard 看不到原文，遗忘只能通过 `npm run rotate-secret` 重置。
