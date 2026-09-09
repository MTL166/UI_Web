# SRS:Galaxy 灵感搜索(Galaxy Search)— 软件需求规格

**Status**: Draft ・ **Author**: Alex(PM)・ **Version**: 0.2
**上游文档**: galaxy-search-PRD-v0.4.md(业务需求,本文为其软件侧派生)
**Last Updated**: 2026

> 本文只含**软件需求**(功能/非功能/数据/接口/约束);业务理由与指标见 PRD v0.4。凡与 PRD 冲突,以 PRD 为准并回改本文。

---

## 1. 引言

### 1.1 目的
定义 Galaxy Search 网站及其构建期数据管道的软件需求,作为工程实现、测试与验收的依据。

### 1.2 范围
- **运行时产品**:Web 网站(React/Vite SPA),部署于阿里云轻量服务器(Nginx 静态站 + Node 登录/收藏 API),面向公司/团队内部用户。
- **构建期工具链**:数据抽取 / 预览渲染 / 索引与向量生成(开发机或 CI 上一次性运行,产物随站分发)。
- **登录/收藏服务**:Node API + 官方 Supabase(`users`/`favorites` 表)。
- 范围外:官网抓取、数据更新、桌面版、注册/找回、第三方登录(见 PRD §3)。

### 1.3 术语
| 术语 | 含义 |
|---|---|
| 组件 | galaxy 目录下单个 HTML 文件(唯一标识 = 文件名,如 `0x3ther_heavy-dragon-56`) |
| 类型 | 组件所在 galaxy 目录(Buttons/loaders/Toggle-switches/Inputs/Checkboxes/Radio-buttons) |
| 描述文档 | 组件侧文本 = tags + 代码可判特征 + 类名,供向量化 |
| 双塔 | 微调 bge-small-zh 双塔检索模型(query 编码器 + 组件描述编码器) |
| 规则路 | 类型词硬过滤 + 风格词典/特征匹配的检索实现(双塔回退方案) |
| 构建期 / 运行时 | 数据管道执行期 / 网站运行期 |
| 登录服务 | Node API:校验 `users` 表并签发会话(JWT cookie) |
| Supabase | 托管 Postgres + Table Editor,存 `users`/`favorites` 表 |

---

## 2. 总体描述

### 2.1 运行环境与用户角色
- 用户:内部同事,通过浏览器访问;无安装、无管理员权限假设。
- 目标浏览器:**Chrome / Edge 最近两个大版本**(Windows 10/11 与 macOS 均可;Firefox/Safari 尽力兼容,不做专项验证)。
- 网络:需联网。首次访问需下载静态资产(页面、模型 ~24MB、向量 ~5MB、预览图懒加载);登录与收藏依赖 Node API → Supabase;检索与推理在浏览器本地完成,不依赖服务器计算。

### 2.2 设计约束(工程)
- 前端主工程为 **Node.js/TypeScript + React + Vite**;本机无 python/docker,主工程**不引入 Python 组件**。
- 检索与推理全部在浏览器端:query 编码用 **onnxruntime-web**(模型 = bge-small-zh LoRA 合并 → ONNX int8),向量为静态 JSON/二进制文件,余弦暴力打分。
- **训练脚本例外**:Python 生态(如 sentence-transformers)仅存在于云端训练环境,与主工程分离,产物为模型权重文件 + 预计算组件向量。
- 服务器侧为 **Node.js API + Nginx**(阿里云轻量):仅承担静态下发、登录、收藏三个职责;service role key / 数据库连接串仅存服务器,不下发前端。
- 组件 HTML 预览在 **sandbox iframe** 中加载:**禁用远程内容加载与脚本越权,仅允许同源静态文件**,防止恶意组件代码(§6 R1/PRD D9)。

### 2.3 数据流(构建期 → 运行时)
```
[构建期] galaxy-main(2,708 HTML)
   → ①抽取器:元数据 + 特征 + 描述文档
   → ②渲染器:Playwright 全量截图(webp)
   → ③向量器:微调双塔批量生成组件向量
   → 产物: components.json + vectors.bin + images/ + 风格标签索引 + 模型 ONNX(int8)
[运行时] 浏览器从 Nginx 加载上述产物与模型;
   登录/收藏 → Node API → Supabase(users / favorites)
```

---

## 3. 功能需求(FR)

优先级:**P0**=v1.0 必须; **P1**=收尾阶段; **P2**=后续可选。

### FR-1 登录门禁(P0)
- FR-1.1 未登录访问任意页面时,重定向到登录页;登录成功前不可进入主界面。
- FR-1.2 用户输入用户名 + 密码提交 `POST /api/login`;服务端查 Supabase `users` 表并 bcrypt 校验,成功后签发 **httpOnly cookie JWT(有效期 30 天)**,后续请求凭 cookie 免登录。
- FR-1.3 失败统一返回"用户名或密码错误",不区分用户名不存在或密码错误。
- FR-1.4 同一账号/IP 连续失败 ≥5 次后锁定 15 分钟(429),期间即使密码正确也拒绝并提示稍后再试。
- FR-1.5 无注册入口、无找回入口;账号由管理员在 Supabase Table Editor 直接建号(bcrypt 哈希)。
- FR-1.6 会话过期(HTTP 401)时,前端清除本地状态并回到登录页。
- 关联:PRD Story 4、D6/D7/F3、R5/R6;详 §6.1 API 契约。

### FR-2 自然语言搜索(P0)
- FR-2.1 输入一句描述(中/英文)提交搜索;支持回车与按钮两种提交。
- FR-2.2 类型词检测:命中 §PRD-5 类型词表(按钮/加载/开关/输入/复选/单选等 + O1 扩充同义词)时,结果**硬过滤**到对应类型,不出现无关类型。
- FR-2.3 无类型词时,全库(6 类)语义打分返回结果。
- FR-2.4 检索实现:默认双塔(浏览器 query 编码 + 全库余弦打分);**双塔不可用或未达标时自动回退规则路**(类型硬过滤 + 风格词典/特征匹配)。
- FR-2.5 结果返回 ≤ 设定条数(默认 Top-24 分页);空结果不显示空页,提示放宽描述并给分类浏览入口。
- FR-2.6 每条结果附带"命中原因"提示(如:类型=Buttons / 风格=gradient 命中 / 语义相似)。
- 关联:PRD Story1、D2/D3、C1–C4。

### FR-3 分类浏览(P0)
- FR-3.1 类型轴仅含 6 类;风格轴来自代码自动探测标签(见 FR-7.3),类型×风格可组合筛选。
- FR-3.2 每个筛选标签旁显示实时组件计数;未选任何筛选时按类型分区浏览。
- FR-3.3 界面含"更多类型即将上线"提示位(指向 Cards 等 5 类)。
- 关联:PRD Story3、D5。

### FR-4 结果网格与预览卡片(P0)
- FR-4.1 卡片展示默认态缩略图(webp)、类型标签、风格标签;无预览图组件标注"预览不可用"但仍可点开。
- FR-4.2 点击卡片进入详情页。
- 关联:PRD Story1/2、E3。

### FR-5 详情页(P0)
- FR-5.1 展示:组件原始 HTML 的**可交互预览**(sandbox iframe 加载,hover/点击真实生效)、类型/风格标签、作者、Uiverse.io 来源链接。
- FR-5.2 "查看代码"展示组件完整源码,可一键复制到剪贴板(`navigator.clipboard`,失败时降级 textarea 手动复制)。
- FR-5.3 "在 Uiverse.io 打开"新标签页打开组件页(需联网,失败时提示)。
- 关联:PRD Story2、D8/D9、E3。

### FR-6 收藏(P1,收尾阶段)
- FR-6.1 登录用户可收藏/取消收藏组件;收藏状态在详情页与卡片上可见。
- FR-6.2 收藏落 Supabase `favorites` 表(服务端鉴权,以 JWT 识别用户);请求失败时前端提示但不阻塞浏览。
- FR-6.3 提供收藏视图(网格,同 FR-4 卡片),按收藏时间倒序。
- 关联:PRD Story4/D7/F4。

### FR-7 构建期数据管道(P0,工具,不进网站前端)
- FR-7.1 **抽取器**:遍历 galaxy-main 6 类目录 2,708 个 HTML;解析元数据注释(tags),**兼容注释位于文件头与 `<style>` 内两种位置**;输出组件记录。
- FR-7.2 **特征抽取**:按 §PRD-8.C 特征草案正则探测风格标签(gradient/neon/glass/neu/animated/dark/3D/minimal…),解析颜色、圆角、动画等结构化特征;产出一份特征词典供 FR-2 规则路与合成训练数据共用。
- FR-7.3 风格标签结果写入索引,供 FR-3 分类轴计数。
- FR-7.4 **渲染器**:Playwright(headless Chromium)对每个组件渲染截图:433 个无 `<style>` 的 Tailwind 组件先注入 Tailwind CDN script 再渲染;默认态截图,输出压缩 webp。
- FR-7.5 渲染失败组件标记 `preview_available=false`,不阻塞管道;统计覆盖率供 PRD §7 gate(≥85%)。
- FR-7.6 **描述文档 + 向量化**:拼 tags+特征+类名为描述文档;用微调双塔批量编码为向量,输出 `vectors.bin`(行序与 `components.json` 一致;bge-small-zh 维度 512)。
- FR-7.7 管道**可重跑、幂等**(输入不变则输出不变);支持增量参数(未来数据源扩展预留接口,本期不实现官网抓取)。
- FR-7.8 **合成训练数据生成器**:词典 × 特征组合生成 1–3 万对 (query, 组件正/负) 训练样本,输出训练集文件供云端训练;人工评测集(50 条,PRD §8.B)单独存放,不入训练集。
- 关联:PRD §5 核心机制、D5、O2/O4。

### FR-8 训练任务(云端,P0 流程,一次性)
- FR-8.1 用合成训练集在云 GPU 微调 **bge-small-zh**(LoRA);仅微调、不从头训练(PRD C3)。
- FR-8.2 输出:合并 LoRA → 导出 ONNX → **int8 量化(~24MB)** 的 query 编码器 + 用其重算的组件向量;产物版本化,与构建期产物一致随站分发。
- FR-8.3 训练/评测脚本与主工程分离,不进入网站前端与服务器运行时。
- 关联:PRD C1–C4、D1–D4、F5。

### FR-9 降级与错误处理(P0)
- FR-9.1 双塔模型缺失/加载失败 → 自动用规则路,界面不报错(可后台日志)。
- FR-9.2 无结果 query → 引导放宽 + 分类浏览入口(FR-2.5)。
- FR-9.3 预览图缺失 → "预览不可用"卡片(FR-4.1)。
- FR-9.4 登录服务/Supabase 不可达 → 已登录会话(无状态 JWT)检索不受影响;收藏操作提示"收藏暂不可用";未登录用户可重试登录。
- FR-9.5 静态资产(索引/向量)加载失败 → 界面提示刷新并保留已加载部分;检索不可用时自动降级为纯分类浏览(分类轴不依赖向量)。
- 关联:PRD §7 Rollback Criteria。

---

## 4. 非功能需求(NFR)

| ID | 类别 | 需求(可测) |
|---|---|---|
| NFR-1 | 性能-搜索 | 提交 query 到结果渲染:**P50 < 1.0s,P95 < 2.5s**(浏览器本地推理,模型已缓存) |
| NFR-2 | 性能-首屏 | 首次访问到可搜索(含模型 ~24MB 下载):**P50 ≤ 5s**(≥10Mbps 网络);二次访问(强缓存)≤ 2s |
| NFR-3 | 性能-浏览 | 分类筛选/分页切换响应 ≤ 200ms(本地数据) |
| NFR-4 | 资源 | 浏览器页面前端内存 ≤ 512MB(含模型与向量);空闲 CPU ≤ 5% |
| NFR-5 | 容量 | 静态产物 ≤ 200MB(组件库只读产物,不含模型);模型文件 ≤ 30MB(int8) |
| NFR-6 | 兼容 | Chrome/Edge 最近两个大版本;高分屏(125%/150% DPI)显示正常;中文界面无乱码 |
| NFR-7 | 可用性 | 检索与浏览不依赖服务器计算;登录与收藏需 Node API/Supabase 在线;服务不可用时检索功能不受影响(FR-9.4) |
| NFR-8 | 可靠性 | 搜索功能崩溃率 < 0.5%/会话;索引加载失败可自动降级(FR-9.5) |
| NFR-9 | 安全 | 组件预览 sandbox iframe 隔离(2.2);密码 bcrypt 哈希存储;JWT httpOnly cookie(30 天);登录限流(FR-1.4);service role key 仅存服务器;不采集/上传用户行为数据 |
| NFR-10 | 可维护性 | 数据管道幂等可重跑(FR-7.7);产物带 schema 版本号,不兼容变更须升版本 |
| NFR-11 | 可测试性 | 评测集(50 条)可脚本化跑三路对照(规则/bge 基座/微调双塔)输出 Top-5 相关率 |

---

## 5. 数据需求

### 5.1 源数据规约(只读,随站分发)
- galaxy-main 6 类共 2,708 HTML;目录=类型;元数据注释位置不统一(文件头或 `<style>` 内);433 个 Tailwind 纯类名组件无 `<style>`;零外部资源依赖。
- 组件唯一 ID = 文件名(去 `.html`),如 `0x3ther_heavy-dragon-56`。

### 5.2 构建期产物(随站分发,只读)
| 产物 | 格式/内容 |
|---|---|
| `components.json` | 数组,每项:{ id, type, file, author, tags[], styles[], colors[], rounded, animated, hasJs, previewAvailable, descriptionDoc, sourceUrl } |
| `vectors.bin` | float32 矩阵,行序 = components.json 顺序(维度 = 512,bge-small-zh) |
| `images/{type}/{id}.webp` | 默认态缩略图;缺失 = 预览不可用 |
| `model/model_int8.onnx` | bge-small-zh LoRA 合并 → ONNX int8(~24MB),浏览器端加载 |
| `meta.json` | schema 版本、生成时间、覆盖率统计、双塔模型版本/是否启用 |
| `type-lexicon.json` | 类型词表(FR-2.2,O1 扩充后) |
| 风格词典 | 特征→风格映射(与 FR-7.2 同源) |

### 5.3 运行时服务端存储(Supabase)
| 表 | 字段 | 说明 |
|---|---|---|
| `users` | `id uuid pk`, `username text unique not null`, `password_hash text not null`, `created_at timestamptz default now()` | 管理员 Table Editor 建号;密码 bcrypt |
| `favorites` | `user_id uuid fk→users.id`, `component_id text`, `type text`, `created_at timestamptz default now()`, `primary key(user_id, component_id)` | 收藏,服务端鉴权写入 |

> 浏览器端不直连 Supabase;仅 Node API 以 service role 连接(经 Session Pooler `*.pooler.supabase.com:6543`,transaction 模式)。Node 连接池 size 3–5;服务器每日一次心跳 `SELECT 1` 防 Supabase 免费版 7 天休眠。

### 5.4 浏览器端本地存储
| 存储 | 内容 | 说明 |
|---|---|---|
| `localStorage` | UI 偏好(筛选状态等,可选) | 可清空,无业务关键数据 |
| httpOnly cookie | 登录 JWT | 前端 JS 不可读,30 天有效期 |

---

## 6. 接口需求

### 6.1 登录/收藏 API(Node)— 契约
```
POST /api/login
  Request : { username: string, password: string }
  Response: 200 { ok: true } + Set-Cookie: token=<JWT>; HttpOnly; SameSite=Lax; Max-Age=2592000
            401 { ok: false, error: "INVALID_CREDENTIALS" }   // 统一提示,不区分用户名/密码
            429 { ok: false, error: "TOO_MANY_ATTEMPTS" }     // 连续失败 ≥5 次锁 15 分钟

GET  /api/favorites                 // 需登录;返回当前用户收藏列表
POST /api/favorites                 // 需登录;{ component_id, type } → 收藏
DELETE /api/favorites/:component_id // 需登录;取消收藏
  未登录/会话过期:401 { ok: false, error: "UNAUTHORIZED" }
  语义: 服务端从 JWT 解析 user_id 后读写 Supabase favorites;前端不持有数据库凭据。
```
- 密码校验:bcript compare(服务端);登录成功后签 JWT(sub=user_id,exp=30d,HS256,密钥存服务器环境变量)。
- Supabase 连接:Session Pooler(6543,transaction 模式)+ 连接池;service role key 仅服务器环境变量。

### 6.2 内部接口
- 检索服务(浏览器端):`search(query) → [{component, score, reason}]`;分类浏览 = 对同一索引的 `filter(type × style)` 查询。前端模块边界:检索库与 React UI 解耦,可 headless 测试(见 spec Testing Decisions)。
- 详情预览:详情页以 sandbox iframe 加载同源组件 HTML(2.2 约束)。
- 剪贴板:`navigator.clipboard.writeText(code)`(失败降级 textarea 复制)。

### 6.3 外部链接
- Uiverse.io 组件页 `https://uiverse.io/components/{slug}`(slug 由构建期从源数据/注释解析,缺失则用仓库文件名兜底)。

---

## 7. 需求追踪矩阵(FR ↔ PRD)

| FR | PRD 章节/Story | 优先级 |
|---|---|---|
| FR-1 | Story 4;D6/D7/F3;R5/R6 | P0 |
| FR-2 | Story1;D2/D3;§5 检索 | P0 |
| FR-3 | Story3;D5 | P0 |
| FR-4 | Story1/2;E3 | P0 |
| FR-5 | Story2;D8/D9 | P0 |
| FR-6 | Story 4;D7/F4 | P1 |
| FR-7 | §5 数据管道;D5;§8.A/C | P0(工具) |
| FR-8 | C1–C4;D1–D4;F5 | P0(流程) |
| FR-9 | §7 Rollback | P0 |

---

## 8. 附录:工程约束清单(Spike 前核对)

- [ ] 前端栈 Node ≥ 20 + React 18/19 + Vite(本机 Node/npm 就绪)
- [ ] 本机无 python/docker → 主工程纯 Node;训练脚本注明云端 Python 环境
- [ ] Playwright 构建环境(开发机/CI)需 Chromium 系统依赖
- [ ] 模型导出链:bge-small-zh LoRA 合并 → ONNX → int8(~24MB)→ onnxruntime-web 加载验证(O3)
- [ ] 阿里云轻量部署:Nginx 静态站 + Node API + systemd/pm2 守护;安全组放行 8080 与公司出口 IP(O5)
- [ ] Supabase 连通性:Session Pooler 6543 连接串、service role key 环境变量、每日心跳保活(O5)

---

## Changelog
- **v0.2(当前)**:从 PRD v0.4 派生——运行时产品改为 Web 网站(React/Vite);FR-1 激活门禁 → 登录门禁(Supabase `users` + bcrypt + JWT cookie);FR-6 收藏落 Supabase;FR-8 锁定 bge-small-zh + ONNX int8;NFR 冷启动/安装包 → 首屏/静态体积/浏览器内存;数据需求 5.3 改为 Supabase 表;接口 6.1 改为 `/api/login` + `/api/favorites`;O3/O5 收敛。
- **v0.1**:从 PRD v0.3 派生首版(桌面应用);FR/NFR/数据/接口/追踪矩阵就位。
