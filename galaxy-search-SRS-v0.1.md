# SRS:Galaxy 灵感搜索(Galaxy Search)— 软件需求规格

**Status**: Draft ・ **Author**: Alex(PM)・ **Version**: 0.1
**上游文档**: galaxy-search-PRD-v0.3.md(业务需求,本文为其软件侧派生)
**Last Updated**: 2026

> 本文只含**软件需求**(功能/非功能/数据/接口/约束);业务理由与指标见 PRD v0.3。凡与 PRD 冲突,以 PRD 为准并回改本文。

---

## 1. 引言

### 1.1 目的
定义 Galaxy Search 桌面应用及其构建期数据管道的软件需求,作为工程实现、测试与验收的依据。

### 1.2 范围
- **运行时产品**:Electron 桌面应用(纯本地单机),面向公司/团队内部用户。
- **构建期工具链**:数据抽取 / 预览渲染 / 索引生成(开发机或 CI 上一次性运行,产物随安装包分发)。
- **激活服务**:Serverless 云函数(全产品唯一在线依赖)。
- 范围外:官网抓取、数据更新、Web 站、登录/账号(见 PRD §3)。

### 1.3 术语
| 术语 | 含义 |
|---|---|
| 组件 | galaxy 目录下单个 HTML 文件(唯一标识 = 文件名,如 `0x3ther_heavy-dragon-56`) |
| 类型 | 组件所在 galaxy 目录(Buttons/loaders/Toggle-switches/Inputs/Checkboxes/Radio-buttons) |
| 描述文档 | 组件侧文本 = tags + 代码可判特征 + 类名,供向量化 |
| 双塔 | 微调 bge 双塔检索模型(query 编码器 + 组件描述编码器) |
| 规则路 | 类型词硬过滤 + 风格词典/特征匹配的检索实现(双塔回退方案) |
| 构建期 / 运行时 | 数据管道执行期 / 桌面应用运行期 |

---

## 2. 总体描述

### 2.1 运行环境与用户角色
- 用户:内部同事,单机使用,无管理员权限假设(安装到用户目录)。
- 目标 OS:**Windows 10/11 x64**(Electron 跨平台能力保留,但 v1.0 仅验证 Windows;macOS/Linux 为非目标)。
- 网络:默认离线可用;仅激活(首次)与可选"跳转 Uiverse.io"需联网。

### 2.2 设计约束(工程)
- 运行时主工程为 **Node.js/TypeScript + Electron**;本机无 python/docker,运行时**不引入 Python 组件**。
- 检索与推理全部本地:query 编码用 onnxruntime-node 或 transformers.js(O3 待 Spike 定),向量为本地文件,余弦暴力打分。
- **训练脚本例外**:Python 生态(如 sentence-transformers)仅存在于云端训练环境,与主工程分离,产物为模型权重文件 + 预计算组件向量。
- 组件 HTML 预览在隔离上下文加载:**禁用远程内容加载、启用 contextIsolation、仅允许本地文件**,防止恶意组件代码(§6 R5)。

### 2.3 数据流(构建期 → 运行时)
```
[构建期] galaxy-main(2,708 HTML)
   → ①抽取器:元数据 + 特征 + 描述文档
   → ②渲染器:Playwright 全量截图(webp)
   → ③向量器:双塔(或 bge 基座)批量生成组件向量
   → 产物: components.json + vectors.bin + images/ + 风格标签索引
[运行时] 安装包内只读加载上述产物;用户数据(收藏/激活凭证)写本地 SQLite
```

---

## 3. 功能需求(FR)

优先级:**P0**=v1.0 必须; **P1**=收尾阶段; **P2**=后续可选。

### FR-1 激活门禁(P0)
- FR-1.1 首次启动展示激活界面,输入激活码后调用激活服务;未激活不可进入主界面。
- FR-1.2 激活成功后在本地持久化激活凭证,后续启动离线可用,不再联网校验。
- FR-1.3 激活服务保证**每码仅激活一次**:已用码返回明确错误,前端展示"激活码已被使用"。
- FR-1.4 激活失败(网络不可达)时展示可重试错误,不清空已输入内容。
- FR-1.5 应用重装后同码不可复用(服务端已记录);需新码(内部流程发放)。
- 关联:PRD D6/F3/R5;详 §6.1 API 契约。

### FR-2 自然语言搜索(P0)
- FR-2.1 输入一句描述(中/英文)提交搜索;支持回车与按钮两种提交。
- FR-2.2 类型词检测:命中 §PRD-5 类型词表(按钮/加载/开关/输入/复选/单选等 + O1 扩充同义词)时,结果**硬过滤**到对应类型,不出现无关类型。
- FR-2.3 无类型词时,全库(6 类)语义打分返回结果。
- FR-2.4 检索实现:默认双塔(本地 query 编码 + 全库余弦打分);**双塔不可用或未达标时自动回退规则路**(类型硬过滤 + 风格词典/特征匹配)。
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
- FR-5.1 展示:组件原始 HTML 的**可交互预览**(hover/点击真实生效,隔离上下文加载)、类型/风格标签、作者、Uiverse.io 来源链接。
- FR-5.2 "查看代码"展示组件完整源码,可一键复制到剪贴板。
- FR-5.3 "在 Uiverse.io 打开"调用系统浏览器打开组件页(需联网,失败时提示)。
- 关联:PRD Story2、D8、E3。

### FR-6 收藏(P1,收尾阶段)
- FR-6.1 详情页可收藏/取消收藏组件;收藏列表本地 SQLite 持久化。
- FR-6.2 提供收藏视图(网格,同 FR-4 卡片)。
- 关联:PRD v0.2 收藏语义(本地起步)。

### FR-7 构建期数据管道(P0,工具,不进安装包 UI)
- FR-7.1 **抽取器**:遍历 galaxy-main 6 类目录 2,708 个 HTML;解析元数据注释(tags),**兼容注释位于文件头与 `<style>` 内两种位置**;输出组件记录。
- FR-7.2 **特征抽取**:按 §PRD-8.C 特征草案正则探测风格标签(gradient/neon/glass/neu/animated/dark/3D/minimal…),解析颜色/圆角/动画等结构化特征;产出一份特征词典供 FR-2 规则路与合成训练数据共用。
- FR-7.3 风格标签结果写入索引,供 FR-3 分类轴计数。
- FR-7.4 **渲染器**:Playwright(headless Chromium)对每个组件渲染截图:433 个无 `<style>` 的 Tailwind 组件先注入 Tailwind CDN script 再渲染;默认态截图,输出压缩 webp。
- FR-7.5 渲染失败组件标记 `preview_available=false`,不阻塞管道;统计覆盖率供 PRD §7 gate(≥85%)。
- FR-7.6 **描述文档 + 向量化**:拼 tags+特征+类名为描述文档;用双塔(或 bge 基座)批量编码为向量,输出 `vectors.bin`(行序与 `components.json` 一致)。
- FR-7.7 管道**可重跑、幂等**(输入不变则输出不变);支持增量参数(未来数据源扩展预留接口,本期不实现官网抓取)。
- FR-7.8 **合成训练数据生成器**:词典 × 特征组合生成 1–3 万对 (query, 组件正/负) 训练样本,输出训练集文件供云端训练;人工评测集(50 条,PRD §8.B)单独存放,不入训练集。
- 关联:PRD §5 核心机制、D5、O2/O3/O4。

### FR-8 训练任务(云端,P0 流程,一次性)
- FR-8.1 用合成训练集在云 GPU 微调 bge 基座(LoRA);仅微调、不从头训练(PRD C3)。
- FR-8.2 输出:微调后模型权重(query 编码器) + 用其重算的组件向量;产物版本化,与构建期产物一致打包。
- FR-8.3 训练/评测脚本与主工程分离,不进入安装包。
- 关联:PRD C1–C4、D1–D4。

### FR-9 降级与错误处理(P0)
- FR-9.1 双塔缺失/加载失败 → 自动用规则路,界面不报错(可后台日志)。
- FR-9.2 无结果 query → 引导放宽 + 分类浏览入口(FR-2.5)。
- FR-9.3 预览图缺失 → "预览不可用"卡片(FR-4.1)。
- FR-9.4 激活服务不可达 → 已激活用户不受影响(离线凭证);未激活用户可重试。
- FR-9.5 本地索引/收藏库损坏 → 启动自检,提示重建(重建仅影响用户数据,组件库随包只读不受影响)。
- 关联:PRD §7 Rollback Criteria。

---

## 4. 非功能需求(NFR)

| ID | 类别 | 需求(可测) |
|---|---|---|
| NFR-1 | 性能-搜索 | 提交 query 到结果渲染:**P50 < 1.0s,P95 < 2.5s**(本地,含双塔 query 编码) |
| NFR-2 | 性能-启动 | 冷启动(双击图标到可搜索):**P50 ≤ 5s**;热启动(二次打开)≤ 2s |
| NFR-3 | 性能-浏览 | 分类筛选/分页切换响应 ≤ 200ms(本地数据) |
| NFR-4 | 资源 | 常驻内存 ≤ 1GB(含 Electron/Chromium + 模型);空闲 CPU ≤ 5% |
| NFR-5 | 容量 | 安装包 ≤ 500MB(目标);组件库只读产物 ≤ 200MB |
| NFR-6 | 兼容 | Windows 10/11 x64;高分屏(125%/150% DPI)显示正常;中文界面无乱码 |
| NFR-7 | 可用性 | 100% 功能离线可用(除首次激活与 Uiverse.io 外链);无网络时启动不卡顿 |
| NFR-8 | 可靠性 | 搜索功能崩溃率 < 0.5%/会话;索引加载失败可自动降级(FR-9.5) |
| NFR-9 | 安全 | 组件预览隔离加载(2.2);激活凭证本地存储;不采集/上传任何用户数据(激活仅传设备指纹) |
| NFR-10 | 可维护性 | 数据管道幂等可重跑(FR-7.7);产物带 schema 版本号,不兼容变更须升版本 |
| NFR-11 | 可测试性 | 评测集(50 条)可脚本化跑三路对照(规则/bge 基座/微调双塔)输出 Top-5 相关率 |

---

## 5. 数据需求

### 5.1 源数据规约(只读,打包)
- galaxy-main 6 类共 2,708 HTML;目录=类型;元数据注释位置不统一(文件头或 `<style>` 内);433 个 Tailwind 纯类名组件无 `<style>`;零外部资源依赖。
- 组件唯一 ID = 文件名(去 `.html`),如 `0x3ther_heavy-dragon-56`。

### 5.2 构建期产物(随包分发,只读)
| 产物 | 格式/内容 |
|---|---|
| `components.json` | 数组,每项:{ id, type, file, author, tags[], styles[], colors[], rounded, animated, hasJs, previewAvailable, descriptionDoc, sourceUrl } |
| `vectors.bin` | float32 矩阵,行序 = components.json 顺序(维度随基座:512/768/1024) |
| `images/{type}/{id}.webp` | 默认态缩略图;缺失 = 预览不可用 |
| `meta.json` | schema 版本、生成时间、覆盖率统计、双塔模型版本/是否启用 |
| `type-lexicon.json` | 类型词表(FR-2.2,O1 扩充后) |
| 风格词典 | 特征→风格映射(与 FR-7.2 同源) |

### 5.3 运行时本地存储
| 存储 | 内容 | 说明 |
|---|---|---|
| `SQLite: favorites.db` | 收藏表(component_id, type, created_at) | 用户数据,可清空重建 |
| `SQLite: app.db` | 激活凭证(device_id, activated_at, code_hash) | 离线校验用,非明文存码 |
| 只读产物 | 见 5.2 | 随包,不落用户写路径 |

---

## 6. 接口需求

### 6.1 激活服务 API(Serverless)— 契约草案(O5 收尾阶段细化)
```
POST /activate
  Request : { code: string, device_id: string, app_version: string }
  Response: 200 { ok: true, activated_at: ISO8601 }
            400 { ok: false, error: "INVALID_CODE" }
            409 { ok: false, error: "ALREADY_USED" }   // 码已被(其他设备)激活
            429 { ok: false, error: "TOO_MANY_ATTEMPTS" }
  语义: 每码仅成功一次;成功后服务端记录 code→device_id 绑定,不可复用。
```
- `device_id` = 本机特征哈希(方案 Spike/收尾定,接受 PRD R5 风险等级)。
- 服务端无状态业务外数据;不存用户身份/行为。

### 6.2 内部接口
- 检索服务(本地,主进程):`search(query) → [{component, score, reason}]`;内部经 IPC 暴露给渲染进程,渲染进程不直读向量文件。
- 详情预览:渲染进程加载本地组件 HTML 的隔离 webview/iframe(2.2 约束)。
- 剪贴板:主进程 `clipboard.writeText(code)`。

### 6.3 外部链接
- Uiverse.io 组件页 `https://uiverse.io/components/{slug}`(slug 由构建期从源数据/注释解析,缺失则用仓库文件名兜底)。

---

## 7. 需求追踪矩阵(FR ↔ PRD)

| FR | PRD 章节/Story | 优先级 |
|---|---|---|
| FR-1 | §5 激活码;D6/F3;R5 | P0 |
| FR-2 | Story1;D2/D3;§5 检索 | P0 |
| FR-3 | Story3;D5 | P0 |
| FR-4 | Story1/2;E3 | P0 |
| FR-5 | Story2;D7/D8 | P0 |
| FR-6 | v0.2 收藏语义 | P1 |
| FR-7 | §5 数据管道;D5;§8.A/C | P0(工具) |
| FR-8 | C1–C4;D1–D4;§7 MVP | P0(流程) |
| FR-9 | §7 Rollback | P0 |

---

## 8. 附录:工程约束清单(Spike 前核对)

- [ ] 运行时 Node/Electron 版本基线(Electron ≥ 28,Node ≥ 20)
- [ ] 本机无 python/docker → 主工程纯 Node;训练脚本注明云端 Python 环境
- [ ] Playwright 构建环境(开发机/CI)需 Chromium 系统依赖
- [ ] 模型运行时选型 onnxruntime-node vs transformers.js(O3)
- [ ] 设备指纹来源与哈希方案(O5)
- [ ] 安装包方案(electron-builder NSIS,user 级安装)

---

## Changelog
- **v0.1(当前)**:从 PRD v0.3 派生首版;FR/NFR/数据/接口/追踪矩阵就位;Open Questions(O1–O5)由 PRD 承接,Spike 输出回填本文。
