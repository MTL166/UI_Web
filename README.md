# UI_Web — Galaxy 灵感搜索（Galaxy Search）

> ⚠️ **新会话必读**：本文件是项目状态交接文档。每次开启新会话，请**先读本文件**，再读 `.scratch/galaxy-search/spec.md`（当前执行规格）与 `.scratch/galaxy-search/tasklist.md`（实现任务清单）。

## 项目一句话

把 galaxy 社区 3,000+ UI 组件变成"看得见、搜得到、拿得走"的**内部网站灵感搜索工具**（初期 ≤10 人）。

## 关键决策（v0.4，已定稿）

- **形态**：Web 网站（React/Vite SPA），废弃 Electron 桌面版
- **部署**：阿里云轻量 2核2G，Nginx 静态站 + Node 登录/收藏 API，`http://公网IP:8080` 免备案先行（备案域名后续平滑升级）
- **数据库**：官方 Supabase（`users` / `favorites` 两表；Node 经 Session Pooler 6543 连接；每日心跳防 7 天休眠）
- **检索**：类型词硬过滤 + 微调双塔；基座锁定 **bge-small-zh**（LoRA → ONNX int8 ~24MB），浏览器端 `onnxruntime-web` 推理，失败自动回退规则路
- **登录**：用户名+密码（bcrypt）→ httpOnly cookie JWT（30 天）；管理员在 Supabase Table Editor 直接建号，无注册/找回

## 本次会话完成（截至 2026-09-09）

- **决策收敛**：grill-me 全程访谈，产品形态从 Electron 桌面改为网站版（见 `galaxy-search-PRD-v0.4.md`）
- **文档产出**：PRD v0.4、SRS v0.2、spec v2、12 张 tracer-bullet 工单、54 个实现任务清单
- **Git 管理**：已初始化并推送 `git@github.com:MTL166/UI_Web.git`（main；SSH 密钥 `~/.ssh/id_ed25519` 已配好并添加到 GitHub）

## 下一步（未开始）

- **可立即开工**：Ticket 01 数据抽取器（从任务 1.1 起）、Ticket 07 前端壳 + 登录门禁（从任务 7.1 起），两线可并行
- 依赖顺序与全部任务见 `.scratch/galaxy-search/tasklist.md`

## 文档导航

| 文件 | 内容 |
|---|---|
| `galaxy-search-PRD-v0.4.md` | 业务需求（当前权威版） |
| `galaxy-search-SRS-v0.2.md` | 软件需求规格 |
| `.scratch/galaxy-search/spec.md` | 工程执行规格（ready-for-agent） |
| `.scratch/galaxy-search/issues/01–12-*.md` | 12 张工单 |
| `.scratch/galaxy-search/tasklist.md` | 54 个 30–60 分钟实现任务 |

## 环境注意（本机实测）

- bash 的 `for` 循环变量展开异常（空值），批量文件处理请用 Node 脚本；Node 在 `G:/K_1/Learn/Node_js/node_1/node.exe`（npm 同目录）
- 本机 `git` 实为 2.25.1（不支持 `git init -b`），建分支用 `git init` + `git symbolic-ref HEAD refs/heads/main`
