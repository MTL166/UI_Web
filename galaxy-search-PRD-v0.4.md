# PRD:Galaxy 灵感搜索(Galaxy Search)

**Status**: Draft ・ **Author**: Alex(PM)・ **Version**: 0.4
**Stakeholders**: 待定(Engineering / Design / Marketing)
**Last Updated**: 2026(随迭代更新)

> **v0.4 范围修订(2026)**:产品形态由 Electron 桌面应用改为 **Web 网站(React/Vite SPA)**,部署于 **阿里云轻量服务器**(Nginx 静态站 + Node 登录/收藏 API),以 `http://公网IP:8080` 免备案先行、备案域名后续平滑升级;检索机制保持「微调双塔」并**锁定基座 bge-small-zh**(浏览器端 `onnxruntime-web` 推理,ONNX int8);授权由激活码改为 **数据库用户表登录**(官方 Supabase:`users` 表 + bcrypt + JWT cookie);收藏由本地 SQLite 改为 **Supabase `favorites` 表**。数据源仍为 galaxy-main 静态打包(不抓官网、不更新)。范围仍为 6 类高频交互组件(Buttons / Checkboxes / Toggle-switches / loaders / Inputs / Radio-buttons,共 2,708 个,占全库 71%)。本版融合 grill-me 全程决策,详见 §8.E 决策记录。

---

## 1. Problem Statement

**问题**:拥有 3,000+ 社区 UI 组件的 galaxy 库,对非开发者几乎不可用。它只有按文件名的代码存档(如 `rare-rattlesnake-22.html`),没有预览图、没有可理解的名称、没有搜索。想找"一个渐变颜色的按钮"的人,面对的是分类目录里的数百个无名 HTML 文件。

**产品范围(v0.4)**:面向公司/团队内部的**网站灵感搜索工具**。用户输入一句想法(如"我想要一个渐变颜色的按钮"),浏览器本地检索即返回几个视觉预览最相似的组件,可点开交互预览、复制代码。聚焦用户最常描述的 6 类交互组件——按钮、输入框、开关、单选、复选、加载动画。

**谁在受这个问题困扰,多频繁,代价是什么**:

| 用户 | 场景 | 当前代价 |
|---|---|---|
| 非开发者的设计灵感寻求者(设计师/产品/市场/学生/创业者) | 想知道某类 UI"可以长什么样" | 看不懂代码 → 直接放弃,或去 Dribbble/小红书搜图,拿不到可直接用的实现 |
| 轻度前端用户 | 想快速找"大概这个感觉"的组件 | 在成百上千个文件中盲翻,平均花 10–20 分钟 |

**Evidence 状态说明**:本 PRD 属 0→1,用户证据将在 Spike 阶段补齐(见 §7 验证计划);当前依据为对 galaxy 仓库结构的实测考察(clone master:6 类共 2,708 个组件)+ 行业同类痛点推理。

---

## 2. Goals & Success Metrics

**North Star(内部发布后 90 天)**:单次会话中找到并打开 ≥3 个"符合想法"的组件预览的用户占比。

| Goal | Metric | Baseline | Target | Window |
|---|---|---|---|---|
| 描述能搜到东西 | 有结果率(无结果 query 占比) | — | 无结果 < 5% | 30 天 |
| 结果真的相关 | Top-5 相关率(人工评测集) | — | ≥ 75% | GA 前固化 |
| 质量按类分层把关(应对 Buttons 一家独大) | Buttons 类 Top-5 相关率(评测集按数量配比加大采样) | — | ≥ 80% | GA 前固化 |
| | Radio-buttons / Checkboxes 等小类 Top-5 相关率(每类固定条数采样) | — | ≥ 70% | GA 前固化 |
| 视觉是核心价值 | 搜索后点击预览率 | — | ≥ 50% | 30 天 |
| 拿得走 | 结果详情页 → 复制代码 / 跳转 Uiverse.io | — | ≥ 25% | 60 天 |
| 值得再来 | 7 日回头率 | — | ≥ 15% | 60 天 |
| 快 | 搜索 P50 耗时(模型已缓存,浏览器本地推理) | — | < 1.0s | 30 天 |
| 快(首访) | 首屏可搜索 P50(含 24MB 模型下载,≥10Mbps 网络) | — | ≤ 5s | 30 天 |

---

## 3. Non-Goals(v1 明确不做)

- **不做桌面版 / 不做移动端 App / 不做小程序**——产品形态为 Web 网站,内部分发
- **不做账号注册、找回、第三方登录**(飞书登录已取消)——授权用 **数据库用户表登录**(Supabase `users`),管理员直接建号
- **本期不做 ICP 备案与自定义域名**——先以 `http://公网IP:8080` 内部分发;备案域名 + HTTPS 作为后续平滑升级(服务器不变)。**Revisit condition**:需要对外正式网址/微信内可打开时,注册域名并备案
- **不做官网抓取 / 不做数据更新**——数据源 = galaxy-main 静态快照,随部署分发,不更新。**Revisit condition**:扩 Cards 等 5 类或确有数据增量需求时,单独立项评估官网一次性抓取(见 §7)
- **不做用户上传组件**——数据单源来自 galaxy(避免内容审核与质量成本)
- **不做可视化定制/改样式编辑器**——用户改样式去 Uiverse.io 或自己项目里改
- **不做高精度个性化推荐**——"相似组件"用同类别+向量邻近即可
- **不追求 100% 渲染覆盖率**——预览渲染目标 ≥85% 即发布,失败组件降级为"仅代码"展示(见 §6 风险 R1)
- **本期不做 Cards、Forms、Notifications、Patterns、Tooltips 的搜索**(共 1,094 个组件)。原因:先验证"想法→匹配"在 6 个高频交互类上的质量与用户价值,再横向扩展。**Revisit condition**:6 类检索质量达标且分类轴带来自然扩展诉求时,按类型逐类放开(每放开一类跑一次该类评测集)。

---

## 4. Personas & Stories

**Primary Persona — 林悦,28 岁,产品运营**
非开发者。要做一个活动落地页,想先看看"高级感一点的渐变按钮"有哪些好看的样子再定风格。她不关心代码,关心"有没有长这样的、我要的那种感觉"。

**Secondary Persona — 阿哲,24 岁,前端实习生**
知道 Uiverse.io,但记不住组件名。会直接说"帮我找一个霓虹灯效的开关",拿代码回项目改。

### Story 1(核心主线)
> 作为林悦,我想输入一句我对界面元素的描述,就能看到几个"长成我描述那样"的组件预览,以便快速获得设计灵感。

**Acceptance Criteria**:
- [ ] Given 我输入"渐变颜色的按钮",when 提交搜索,then ≤1s 内返回结果网格,每张卡片是可点击的视觉预览图
- [ ] Given 描述含"按钮"(组件类型词),when 返回结果,then 结果以 Buttons 类型为主(硬过滤命中时不出无关类型)
- [ ] Given 描述不含任何类型词(如"炫酷发光"),when 返回结果,then 走纯语义检索,仍返回相关组件并显示命中原因
- [ ] Given 无相关结果,when 提交,then 不显示空页,提示放宽描述并给出分类浏览入口
- [ ] Performance:搜索 P50 < 1.0s / P95 < 2.5s(浏览器本地推理,模型已缓存);首屏可搜索 P50 ≤ 5s(≥10Mbps 网络)

### Story 2(拿代码)
> 作为阿哲,我想打开某个结果看它的代码,以便直接复制到我的项目里。

**Acceptance Criteria**:
- [ ] Given 我点击某张预览卡片,when 详情打开,then 展示大图预览(可交互)、组件类型/风格标签、原始作者与 Uiverse.io 来源链接
- [ ] Given 该组件为纯 HTML/CSS,when 我点"查看代码",then 可一键复制完整代码
- [ ] Given 该组件渲染失败(无预览图),when 我浏览结果,then 卡片仍可见并明确标注"预览不可用",不阻塞复制代码
- [ ] Given 我在详情页,when 我 hover/点击组件本体,then 组件真实交互生效(sandbox iframe 加载原始 HTML 预览)

### Story 3(分类浏览)
> 作为林悦,我想按"组件类型 × 风格"筛选浏览,以便不靠描述也能逛到灵感。

**Acceptance Criteria**:
- [ ] Given 首页分类浏览,when 我选择类型=Buttons、风格=Gradient,then 只显示同时满足两条件的组件
- [ ] Given 类型筛选轴仅含 6 类(按钮/加载/开关/输入/复选/单选),when 我打开筛选器,then 不出现 Cards 等空分类;页面提供"更多类型即将上线"提示位
- [ ] Given 风格轴来自代码自动探测,when 我切换筛选,then 每个标签旁显示组件数量
- [ ] Given 我未选任何筛选,when 浏览,then 按类型分区展示(与 galaxy 目录一致)

### Story 4(登录门禁,v0.4 新增)
> 作为内部用户,我想用管理员发给我的用户名和密码登录网站,以便只有被授权的同事能使用。

**Acceptance Criteria**:
- [ ] Given 未登录访问,when 打开任意页面,then 重定向到登录页,不可进入主界面
- [ ] Given 正确用户名密码,when 提交登录,then 签发 httpOnly cookie 会话(30 天),进入主界面,期间刷新/重开无需重复登录
- [ ] Given 错误用户名或密码,when 提交,then 显示统一中文错误提示"用户名或密码错误",不区分具体是哪一项错误
- [ ] Given 连续失败 ≥5 次,when 再提交,then 触发限流(如 15 分钟内拒绝尝试),防止爆破
- [ ] 无注册入口、无找回入口;账号由管理员在 Supabase Table Editor 直接建号

---

## 5. Solution Overview

**产品形态**:Web 网站(**React/Vite SPA**),部署在 **阿里云轻量服务器(2核2G)**。主界面同时提供两条路径——居中的自然语言搜索框 + 左侧"类型 × 风格"分类筛选器;结果与浏览均为**视觉预览网格**(缩略图卡片),点击进入详情(大图交互预览 + 标签 + 作者/来源 + 查看/复制代码)。**Nginx 静态下发**全部前端资产(页面、预览图、向量索引、模型);**Node API** 仅承担登录与收藏两个在线接口;数据库为 **官方 Supabase**。检索与推理全部在用户浏览器本地完成。

**端到端流程**:
1. 用户登录(用户名 + 密码)→ Node API 查 Supabase `users` 表(bcrypt 校验)→ 签发 httpOnly cookie JWT(30 天);
2. 用户输入一句话(或点筛选)→ 查询理解:检测是否含**组件类型词**(按钮/加载/开关/输入/复选/单选 → 映射到 galaxy 目录),以及**风格/视觉词**;
3. 检索:类型词命中则先在对应目录内硬过滤,再用**微调双塔向量打分**在候选中找最接近描述者;无类型词则全库(6 类)向量打分;2,708 条本地暴力余弦 <10ms,无需 ANN;
4. 结果以预览图卡片呈现 → 点击看详情(交互预览)→ 复制代码或跳转 Uiverse.io。

**核心机制——数据管道(产品成立的前提)**:galaxy 提供的是代码,而本产品要卖的是"预览"。建库时对每个组件做三件事:① 抽取元数据(组件类型=所在目录;风格特征=从 HTML/CSS 代码启发式探测,如 `gradient`/`neumorphism`/`glassmorphism`/`neon`/`3D`/`animated`/`minimal`/`dark`,解析颜色、圆角、动画等;tags 解析需兼容注释在文件头与 `<style>` 内两种位置);② **构建期用 headless 浏览器(Playwright)把组件渲染成缩略图 webp**(433 个 Tailwind 纯类名组件注入 Tailwind CDN 后渲染;默认态静态图);③ 把"代码特征+标签+类名"拼成组件描述文档,供向量化与检索。**特征抽取器是本产品的核心组件,一器三用:规则层检索、合成训练数据生成、风格分类轴**。

**检索与模型——微调双塔(浏览器端推理)**:
- **模型**:微调开源双塔基座 **bge-small-zh(LoRA),仅微调、不从头训练**;微调后合并 LoRA → 导出 ONNX → **int8 量化(~24MB)**。组件侧向量在训练环境批量生成后**离线预计算并随站分发**;线上(浏览器)只跑 query 编码 + 全库余弦打分(`onnxruntime-web`)。
- **训练数据**:程序化合成打底——中文风格/颜色/形状词典 × 代码可判特征自动生成 1–3 万对 (query, 组件) 正负样本,tags 增强;人工评测集 50 条只做验收、不进训练。
- **算力**:训练用按量云 GPU(数小时,几十元;或 Kaggle/Colab 免费额度);推理在用户浏览器 CPU/WebGPU(bge-small-zh int8,~50–150ms/query)。
- **验收**:Spike 阶段起在人工评测集上**三路对照**(词典规则路 / 现成 bge 基座 / 自训练双塔)× **双门槛**(绝对线:Buttons Top-5 ≥80%、小类 ≥70%;相对优势线:较规则路高 ≥10pp)。不达标**回退规则路**(规则路本身即 MVP 可用形态,双塔为增强项)。

**类型词 → 目录映射(v0.4,Spike 内扩充同义词)**:

| 用户说法(词表起点) | 映射目录 | 组件数 |
|---|---|---|
| 按钮 / 按键 / button / btn | Buttons | 1,231 |
| 加载 / 转圈 / spinner / 进度动画 | loaders | 718 |
| 开关 / 滑块开关 / toggle / switch | Toggle-switches | 260 |
| 输入框 / 输入 / input / 文本框 | Inputs | 226 |
| 复选框 / checkbox / 勾选 | Checkboxes | 171 |
| 单选 / 单选按钮 / radio | Radio-buttons | 102 |

**授权——用户表登录(替代激活码)**:应用首次访问需登录。`POST /api/login` 查 Supabase `users` 表(username + bcrypt hash),成功签发 httpOnly cookie JWT(30 天)。管理员在 Supabase Table Editor 直接建号;无注册、无找回、无第三方登录。服务端每日心跳(`SELECT 1`)防止 Supabase 免费版 7 天无活动自动休眠。

### Key Design Decisions

| # | 决策 | 理由 | Trade-off |
|---|---|---|---|
| D1 | **预览渲染是 MVP 必做,构建期一次性全量渲染并随站分发** | 目标用户是泛用户,看不懂代码,"看见"才是价值 | 一次性工程成本(批量渲染+失败处理),但无它则产品只是"对开发者友好的 grep";静态分发使运行时零渲染 |
| D2 | **类型硬过滤 + 向量打分的混合检索** | 中文口语描述与"组件类型"是两回事;类型词必须精确,风格与感觉交给向量 | 依赖一份中文类型词表(规模小、易维护),未命中词走纯语义兜底 |
| D3 | **自训微调双塔(bge-small-zh LoRA),浏览器端 ONNX int8 推理,组件向量离线预计算** | 目标函数直接建模「这句中文和这个组件有多像」;浏览器推理免常驻服务器、免 API 成本、隐私好 | 浏览器端锁定小模型(24M 参数,int8 ~24MB),语义容量小于 bge-m3;需合成训练数据 + 一次性云 GPU;效果需三路对照验证,不达标回退规则路 |
| D4 | **数据源 = galaxy-main 静态快照随站分发,不抓官网、不更新** | 官方仓库 MIT、自包含、目录即类型,2,708 个足够验证价值;仓库 2024-09 停更,静态化消除维护成本 | 数据不增长;未来扩类/增量需另立项(触发条件见 §3) |
| D5 | **特征抽取器一器三用(规则层检索 / 合成训练数据 / 风格分类轴)** | 三个消费方共享同一份「代码可判特征」,词典与正则只维护一份 | 抽象风格(如"高级感")探测不到 → 交由向量语义承担 |
| D6 | **阿里云轻量服务器(2核2G)承载:Nginx 静态站 + Node 登录/收藏 API;上线走 `http://公网IP:8080` 免备案** | 大陆直连、一台机器解决静态下发与在线接口;非标准端口避开未备案 80/443 拦截;~¥68/年近乎零成本;备案域名后可平滑升级 80/443 + HTTPS | HTTP 明文(缓解:登录限流 + bcrypt + 安全组白名单公司出口 IP);需基本运维(重启/更新);公网 IP 变更需重新分发地址 |
| D7 | **数据库 = 官方 Supabase(免费额度):`users` + `favorites` 两表;Node 用 Session Pooler(6543)连接;service role key 仅存服务器** | 用户已有 Supabase 项目;托管 Postgres + Table Editor 免费提供"管理员建号"界面;两表规模极小 | 跨境延迟 0.7–1.2s/次(仅登录与收藏受影响,检索零依赖);免费版 7 天无活动休眠(服务器心跳保活);IPv6 优先需用 pooler 绕开 |
| D8 | **结果详情强制展示作者与 Uiverse.io 来源链接** | MIT 允许免署名但鼓励;对社区友好、也符合"灵感库"调性 | 无(纯收益) |
| D9 | **详情页 = sandbox iframe 加载原始 HTML 交互预览(非静态图)** | 静态图截不出 hover/checked 等状态;浏览器直接跑组件 HTML,交互真实且除组件本体外零网络 | 需 iframe sandbox 隔离(禁远程内容、禁脚本越权),兼容性需 Spike 验证 |

---

## 6. Technical Considerations

### Dependencies

| 依赖 | 用途 | Owner | 时间风险 |
|---|---|---|---|
| uiverse-io/galaxy(git,已解压 galaxy-main) | 唯一数据源(6 类 2,708 组件,~12MB),随站分发 | Eng | Low(公开只读,已本地化) |
| React + Vite | 网站前端(SPA)与构建 | Eng | Low(本机 Node/npm 就绪) |
| Playwright(headless Chromium) | **构建期**批量渲染组件 → 预览图(433 个 Tailwind 组件注入 CDN) | Eng | Med(见 R1) |
| bge-small-zh 基座 + LoRA 微调 → ONNX int8 | 浏览器端 query 编码器(~24MB) | Eng | Low(训练在云 GPU) |
| onnxruntime-web | 浏览器端 ONNX 推理(CPU/WebGPU) | Eng | Low |
| 云 GPU(按量)/ Kaggle-Colab | 一次性训练(数小时) | Eng | Low |
| 阿里云轻量服务器(2核2G)+ Nginx + Node.js | 静态站 + 登录/收藏 API,~¥68/年 | Eng | Low |
| 官方 Supabase(免费额度) | `users`/`favorites` 表;Session Pooler(6543)连接;心跳保活 | Eng | Med(跨境延迟/休眠,见 R5) |
| bcrypt + JWT(httpOnly cookie) | 密码哈希与登录会话 | Eng | Low |

### Known Risks

| 风险 | 概率 | 影响 | Mitigation |
|---|---|---|---|
| R1 **渲染失败率高**:组件依赖外部 CDN/Tailwind/字体/图片,或需 JS 交互(hover/点击)才显示,截图可能空白或走样 | Med-High | High(预览是核心价值) | Spike **全量渲染 2,708 个**实测失败率;构建期一次完成;GA gate = 覆盖率 ≥85%;失败项降级"仅代码"卡片;Tailwind 组件注入 CDN;hover 态交还详情页交互预览 |
| R2 **类型规模失衡损害小类召回**:Buttons(1,231)占 6 类 45%,若检索只按全库优化,Radio-buttons(102)/Checkboxes(171) 的召回易被淹没 | High | High | §2 分层质量门槛;评测集按类配比(大类加采样、小类固定条数);类型硬过滤先圈定类别再打分 |
| R3 **合成训练数据分布偏差**:模板化 query 与真实口语(零碎/抽象)有差距,双塔可能对合成样本过拟合、在人工评测集退化 | Med-High | High | 三路对照双门槛验收;评测集覆盖抽象 query(如"高级感""赛博朋克");不达标回退规则路(撤退预案) |
| R4 **双塔工程成本 vs 收益不成立** | Med | Med | 规则路先落地即为 MVP 可用形态,双塔作为并行增强,验收不过不阻塞发布 |
| R5 **Supabase 跨境延迟与休眠**:大陆服务器连官方 Supabase 每次 0.7–1.2s;免费版 7 天无活动自动休眠 | High | Med(仅登录/收藏,检索零依赖) | Node 用 Session Pooler(6543,transaction 模式)+ 连接池;服务器每日心跳 `SELECT 1` 保活;登录为一次性成本、收藏为 P1;接受该延迟作为内部工具代价 |
| R6 **HTTP 明文登录**:`http://IP:8080` 无 TLS,密码与 JWT 可被链路窃听 | Med | Med | 登录接口限流(≥5 次失败锁 15 分钟)+ bcrypt 哈希 + 安全组只放行公司出口 IP;备案域名后升级 HTTPS(服务器不变) |
| R7 **静态资产体积与首屏**:HTML 12MB + 预览图 webp ~100–150MB + 模型 24MB + 向量 ~5MB | Low | Med | 预览图压缩 webp + 懒加载;模型/向量强缓存(仅首次下载);Nginx gzip/brotli;首屏目标 ≤5s(≥10Mbps) |
| R8 数据源停更导致库容陈旧 | Low(已接受) | Low | 静态化是有意决策(§3 Non-Goals);未来扩数据按触发条件另立项 |

### Open Questions(dev 开始前必须解决)

- [ ] **O1** 6 类类型词表与同义词扩充(起点见 §5 映射表)— Owner: PM — Deadline: Spike 结束
- [ ] **O2** 合成训练数据的特征词典/模板质量验证(能否覆盖评测集抽象 query)— Owner: Eng — Deadline: Spike 结束
- [ ] **O3**(已收敛)模型运行时 = onnxruntime-web,基座 = bge-small-zh(LoRA → ONNX int8);Spike 仅需验证 int8 导出与浏览器推理耗时
- [ ] **O4** 人工评测集 50 条最终编制(按类配比 + 抽象 query 覆盖)与标注执行方式 — Owner: PM+Eng — Deadline: Spike 结束
- [ ] **O5**(已收敛)数据库 = 官方 Supabase;Node 经 Session Pooler 6543 连接;Spike 需实测阿里云轻量 → Supabase 的连通性与登录延迟

---

## 7. Launch Plan(网站内部工具形态)

| Phase | 周期 | 范围 | Success Gate |
|---|---|---|---|
| **Spike(可行性验证)** | 第 1 周 | 数据抽取脚本**全量 2,708 个跑通**;构建期渲染试点(50–100 个)度量渲染可行性;合成数据管道原型;**评测集 50 条编制**;产出 O1–O4 结论;**部署 Spike**:阿里云轻量 + Nginx + Node API + Supabase pooler 连通性/延迟实测;锁定 O3/O5 | 渲染方案可行(预期 ≥85% 覆盖率可达成)且评测集可用;Supabase 登录链路可用(延迟可接受),否则回炉调整方案再立项 |
| **MVP Build** | 第 2–5 周 | 数据管道全量(2,708)+ 构建期全量渲染 + 检索(**先规则路落地**,双塔训练并行:合成数据就绪 → 云 GPU 微调 → 三路对照评测)+ Web 前端(React/Vite:搜索/筛选/详情交互/复制代码)+ 登录/收藏 API + Supabase `users`/`favorites` 表 | 双塔达标则切双塔,不达标用规则路;网站可登录、可搜索,评测集达标(§2) |
| **Alpha** | 第 6 周 | 内部 + ≤10 名设计/产品同事,经 `http://IP:8080` 使用 | 无 P0 bug;用户能在 5 分钟内完成"登录→描述→拿到结果" |
| **Beta** | 第 7–8 周 | 内部种子用户 30–50 人 | 无结果率 <10%;搜索 P50 <1s;首屏 P50 ≤5s;CSAT ≥4/5 |
| **内部发布 v1.0** | 第 9 周起 | 全团队分发访问地址(IP:8080,或已备案域名) | 指标达 §2 目标;预览覆盖率 ≥85%;登录门禁上线 |
| **收尾阶段(发布后)** | v1.0 后 | 备案域名 + HTTPS(可选)、收藏功能打磨、安全组/限流调优 | — |

**Rollback Criteria**:检索不可用时,应用自动降级为纯分类浏览(类型×风格筛选不依赖向量);双塔不达标自动回退规则路;渲染管线故障只影响构建期,不影响运行时既有数据;Supabase/登录服务不可用 → 已登录会话(JWT 无状态)检索不受影响,仅登录与收藏降级提示;静态资产更新失败 → 保留上一版产物回滚。

---

## 8. Appendix

### A. 数据源实测结构(v0.4,galaxy-main 本地实测)

| 类型(galaxy 目录) | 组件数 | 是否纳入 |
|---|---:|---|
| Buttons | 1,231 | ✅ |
| loaders | 718 | ✅ |
| Toggle-switches | 260 | ✅ |
| Inputs | 226 | ✅ |
| Checkboxes | 171 | ✅ |
| Radio-buttons | 102 | ✅ |
| **6 类合计** | **2,708(占全库 71%)** | ✅ |
| Cards | 726 | ❌ 本期不做 |
| Forms | 180 | ❌ |
| Patterns | 103 | ❌ |
| Tooltips | 62 | ❌ |
| Notifications | 23 | ❌ |
| **全库合计** | **3,802** | |

> 6 类源码合计约 12MB。**实测渲染特征**:含 `<style>` 块 3,369 个(89%,自包含可直渲);无 `<style>` 的 433 个为 Tailwind 纯类名组件(需注入 Tailwind 运行时);含 `@keyframes` 动画 1,356 个(36%);含 JS 93 个;外部 CDN/字体依赖 0(自包含)。元数据注释位置不统一(文件头 或 `<style>` 内),抽取器需兼容两种。

### B. 评测集样例(spike 用,目标 50 条,按类配比 + 抽象 query 覆盖)

| Query | 期望 | 类型 |
|---|---|---|
| 渐变颜色的按钮 | Buttons ∩ gradient | 类型+风格 |
| 霓虹灯效的开关 | Toggle-switches ∩ neon | 类型+风格 |
| 转圈圈的加载动画 | loaders | 类型 |
| 圆角输入框 | Inputs(语义) | 语义 |
| 红色选中态的复选框 | Checkboxes | 类型+颜色 |
| 胶囊形状的单选 | Radio-buttons(语义) | 语义 |
| 炫酷发光 | 跨类 neon/glow(语义) | 纯语义(无类型词) |
| 赛博朋克感觉的按钮 | Buttons(语义,验收 R3) | 抽象风格 |
| …(共 50 条:大类加采样、小类固定条数、≥10 条纯语义/抽象 query) | | |

### C. 风格探测特征草案

| 风格标签 | 代码特征(正则起点) |
|---|---|
| gradient | `linear-gradient` / `radial-gradient` / `conic-gradient` |
| neon / glow | 彩色 `box-shadow` / `text-shadow` |
| animated | `@keyframes` / `transition` |
| glassmorphism | `backdrop-filter` |
| neumorphism | 同色系双阴影(box-shadow 亮+暗) |
| 3D / minimal / dark 等 | 颜色数、圆角、配色统计(Spike 内定稿) |

### D. 合规

- galaxy 仓库全部组件 **MIT**;产品打包分发、内部使用无授权障碍;界面展示"作者 × Uiverse.io"署名与来源链接。
- 本版**不做官网抓取**,不涉及官网 ToS/Cloudflare 合规问题。
- 登录机制最小化数据收集:仅存 `username` + 密码哈希(bcrypt)与收藏记录,存于 Supabase 官方云(海外节点);不采集个人身份信息、无设备指纹。内部工具风险等级低,已接受跨境存储。
- `http://IP:8080` 属内部分发,不对外公开运营;若后续对外提供网站服务,按法规要求完成 ICP 备案。

### E. 决策记录(grill-me 全程收敛,v0.4 固化)

| # | 决策 | 定案 |
|---|---|---|
| C1 | 匹配机制 | 本地**微调双塔**,不依赖现成 embedding API 当主力 |
| C2 | 模型形态 | 双塔匹配模型(query 编码器 + 组件描述编码器) |
| C3 | 训练范畴 | **仅微调**(bge 系 LoRA),**不从头训练**;基座锁定 **bge-small-zh** |
| D1 | 训练数据 | **程序化合成打底**(词典 × 代码可判特征 → 1–3 万对,tags 增强);人工评测集只验收 |
| D2 | 算力 | 训练=按量云 GPU 数小时 / Kaggle-Colab 免费;推理=浏览器端 CPU/WebGPU |
| D3 | 检索架构 | **类型硬过滤 + 双塔全库打分**(2,708 暴力余弦 <10ms,无需 ANN) |
| D4 | 验收 | **三路对照 × 双门槛**(绝对线 + 相对优势 ≥10pp);不达标回退规则路 |
| — | Dify | **排除**,主线纯双塔 |
| E1 | 渲染范围 | 构建期全量 2,708,gate ≥85%,失败降级「仅代码」 |
| E2 | Tailwind 组件(433) | 构建期**注入 Tailwind CDN** 渲染(产物静态分发,运行时离线) |
| E3 | 状态策略 | 网格=默认态静态图;详情=**sandbox iframe 交互预览**(hover/点击交还用户) |
| F1 | 产品形态 | **Web 网站(React/Vite SPA)**,废弃 Electron 桌面版;不做移动端/小程序 |
| F2 | 部署架构 | 阿里云轻量 2核2G:Nginx 静态站 + Node 登录/收藏 API;`http://公网IP:8080` 免备案先行 |
| F3 | 授权 | **数据库用户表登录**:Supabase `users`(bcrypt)+ httpOnly cookie JWT(30 天);管理员 Table Editor 建号;无注册/找回/飞书/激活码 |
| F4 | 数据/收藏 | 静态数据随站分发;收藏落 Supabase `favorites` 表(服务端鉴权) |
| F5 | 推理运行时 | `onnxruntime-web` + bge-small-zh ONNX int8(~24MB);组件向量离线预计算随站分发 |

---

## Changelog

- **v0.4(当前)**:产品形态 Electron 桌面 → **Web 网站(React/Vite SPA)**;部署 = 阿里云轻量(Nginx 静态 + Node API)+ `http://IP:8080` 免备案;授权激活码 → **Supabase 用户表登录**(bcrypt + JWT cookie);收藏本地 SQLite → Supabase `favorites`;模型基座锁定 **bge-small-zh**、运行时锁定 **onnxruntime-web**(ONNX int8 ~24MB);Non-Goals/依赖/风险/Open Questions/Launch Plan/决策记录全面更新;新增 Story 4 登录门禁。
- **v0.3**:产品形态 Web → Electron 桌面(纯本地单机);检索现成 embedding → 微调双塔(合成数据/云 GPU/三路对照验收);数据源定稿 galaxy-main 静态打包(不抓官网、不更新);授权登录/飞书 → 激活码(每码一次);渲染定稿构建期全量 + Tailwind CDN 注入 + 详情交互预览;Non-Goals/依赖/风险/Open Questions 全面更新;附录新增 E 决策记录。连带 UI 决策:分类轴仅显示 6 类 + "更多类型即将上线"提示位。
- **v0.2**:范围收敛为 6 类(2,708 个);新增分层质量指标(§2);Non-Goals 增补 5 类不做及 revisit 条件(§3);类型词映射表定稿 6 类(§5);Spike 改全量渲染(§7);附录更新实测数据表/评测集。
- **v0.1**:初稿,全库 11 类范围。
