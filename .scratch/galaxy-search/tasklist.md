# Galaxy Search — 实现任务清单（tasklist）

**上游**: `.scratch/galaxy-search/spec.md`(v2)・`galaxy-search-PRD-v0.4.md`・`galaxy-search-SRS-v0.2.md`
**工单**: `.scratch/galaxy-search/issues/01–12.md`(本文是其 30–60 分钟级实现细分)
**约定**: 每任务一名开发 30–60 分钟可完成;不加规格外功能;验收标准 = 对应工单 AC 的细分。

> 环境注意:本机 bash 的 `for` 循环变量展开异常(空值),批量处理请用 Node 脚本;Node 二进制位于 `G:/K_1/Learn/Node_js/node_1/node.exe`(npm 同目录)。

---

## 立即开工 A — Ticket 01 数据抽取器与特征词典

规格原文要求:「遍历 6 类 2,708 个 HTML;解析元数据注释(tags),兼容注释位于文件头与 `<style>` 内两种位置;风格标签梯度探测;输出 components.json + 类型词表 + 风格词典;幂等可重跑」。

### 任务 1.1 初始化 pipeline 工程
- **做什么**: 在 `pipeline/` 建 Node 24 ESM 工程(`package.json`、`src/`),用 Node 内置 `node:test` 做测试,零第三方依赖;加 `npm run extract` 与 `npm test` 脚本。
- **验收**: `npm test` 空测试通过;`node src/extract.js --help` 可运行。
- **参考**: 工单 01 AC-5(幂等)、AC-6(统计输出)。

### 任务 1.2 目录遍历与组件 ID
- **做什么**: 递归读取 `galaxy-main/{Buttons,loaders,Toggle-switches,Inputs,Checkboxes,Radio-buttons}` 下全部 `.html`;组件 ID = 文件名去 `.html`(如 `0x3ther_heavy-dragon-56`);产出组件记录骨架(id/type/file)。
- **验收**: 共 2,708 条;6 类计数与 PRD §8.A 一致(Buttons 1,231 / loaders 718 / Toggle-switches 260 / Inputs 226 / Checkboxes 171 / Radio-buttons 102)。

### 任务 1.3 元数据注释解析(两种位置)
- **做什么**: 解析头部注释 `<!-- From Uiverse.io by <author> - Tags: a, b, c -->` 与 `<style>` 内注释 `/* From Uiverse.io by <author> - Tags: ... */`;提取 author、tags[](去重)、sourceUrl(`https://uiverse.io/components/<文件名去.html>` 兜底);无注释时 author/tags 置空、sourceUrl 用兜底。
- **验收**: 对「注释在文件头」与「注释在 `<style>` 内」两类样本均解析成功;tags 不重复、不残留空白。

### 任务 1.4 风格特征探测(PRD §8.C 草案)
- **做什么**: 正则探测——gradient(`linear|radial|conic-gradient`)、neon/glow(彩色 box-shadow/text-shadow)、animated(`@keyframes`/`transition`)、glassmorphism(`backdrop-filter`)、neumorphism(同色系双阴影)、dark/3D/minimal(颜色数、圆角、配色统计初版);同时抽 colors[]、rounded、hasJs、animated 布尔。
- **验收**: 对 PRD §8.B 样例组件(渐变按钮/霓虹开关/加载动画等)探测出的 styles 与人工判断一致;输出风格词典。
- **参考**: SRS FR-7.2、PRD §8.C。

### 任务 1.5 描述文档拼装 + 产物输出
- **做什么**: `descriptionDoc` = tags + styles + 类名(class 列表去重)拼接;输出 `components.json`(含 previewAvailable 默认 true,待 02 回填)、`type-lexicon.json`(PRD §5 映射表 + 同义词占位)、`style-dictionary.json`;stdout 打印 6 类计数与风格分布统计。
- **验收**: 产物字段与 SRS §5.2 契约一致;重跑两次输出字节一致(golden)。
- **参考**: SRS §5.2、工单 01 AC-4/5/6。

### 任务 1.6 fixture 测试与幂等 golden
- **做什么**: 从 6 类各抽 3–5 个构造 fixture(含文件头注释、`<style>` 内注释、无注释、Tailwind 纯类名、含 @keyframes);断言注释兼容、风格标签、ID 生成;golden 文件比对保证幂等。
- **验收**: `npm test` 全绿,覆盖工单 01 全部 AC。

---

## 立即开工 B — Ticket 07 前端壳 + 登录门禁

规格原文要求:「未登录访问任意页面重定向登录页;用户名+密码经 Node API 校验(Supabase `users` + bcrypt)签发 httpOnly cookie JWT(30 天);≥5 次失败锁 15 分钟;管理员 Table Editor 建号,无注册/找回」。

### 任务 7.1 React/Vite/TS 脚手架 + 路由骨架
- **做什么**: 建 `app/` 工程(React + Vite + TypeScript);路由:`/login`、`/`(主壳占位)、`*` 重定向;装 `react-router-dom`。
- **验收**: `npm run dev` 启动;访问 `/` 无登录态时重定向 `/login`。

### 任务 7.2 Supabase `users` 表 + Node API 工程
- **做什么**: 在 Supabase 项目(`alpxxqskougklqijzkqk`)建 `users` 表(id uuid pk / username unique / password_hash / created_at);建 `api/` Node 工程(express 或原生 http + `pg`);配 Session Pooler(`*.pooler.supabase.com:6543`,transaction 模式)与连接池 3–5。
- **验收**: 本地脚本能经 pooler 完成一次 `SELECT 1`;手工插入一个 bcrypt 测试用户,记录登录链路延迟(接受 0.7–1.2s)。
- **参考**: SRS §5.3、工单 07 AC-6。

### 任务 7.3 `/api/login` 实现(bcrypt + JWT + 限流)
- **做什么**: `POST /api/login`:查 `users`、`bcrypt.compare`;成功签发 HS256 JWT(sub=user_id, exp=30d)写入 httpOnly cookie(SameSite=Lax);失败统一 401「用户名或密码错误」;同账号/IP 连续失败 ≥5 次 → 429 锁 15 分钟(内存或 SQLite 计数)。
- **验收**: 正确凭据 200+Set-Cookie;错误凭据 401 且不区分错误类型;第 6 次连续失败返回 429。
- **参考**: SRS §6.1、工单 07 AC-2/3/4。

### 任务 7.4 登录页 UI + 路由守卫
- **做什么**: 登录页表单(用户名/密码、错误提示、提交 loading);前端路由守卫:无 cookie(或 `/api/me` 401)→ 重定向 `/login`;登录成功进主壳(占位页);登出按钮清 cookie。
- **验收**: 未登录重定向、登录成功进主壳、刷新免登录、登出回登录页。
- **参考**: 工单 07 AC-1/5。

### 任务 7.5 登录冒烟测试
- **做什么**: Playwright 冒烟:未登录重定向 → 正确登录 → 错误登录提示 → 锁定提示;`npm test` 覆盖。
- **验收**: 主流程走通;锁定逻辑可复现。
- **参考**: spec Testing Decisions(UI 冒烟)。

---

## Ticket 02 构建期渲染器(blocked by 01)

工单目标:「全量渲染 2,708 个为默认态 webp;433 个 Tailwind 组件注入 CDN;失败标 preview_available=false;输出覆盖率支撑 ≥85% gate」。

### 任务 2.1 Playwright 环境与单组件渲染
- **做什么**: 在 `pipeline/` 安装 Playwright + Chromium;写渲染脚本:加载一个 Button HTML,截默认态图,输出 webp。
- **验收**: 单个组件成功产出 webp;CI/本地可无头运行。

### 任务 2.2 Tailwind 组件注入运行时
- **做什么**: 识别无 `<style>` 的 433 个 Tailwind 纯类名组件;渲染前注入 Tailwind CDN `<script>`;等待渲染完成再截图。
- **验收**: 抽样 Tailwind 组件截图非空白(与人工目视一致)。

### 任务 2.3 全量渲染循环
- **做什么**: 遍历 2,708 个组件,输出 `images/{type}/{id}.webp`;控制并发(如 4–8 并行);支持断点续跑(已存在且成功的跳过);失败收集不中断。
- **验收**: 全量跑完;产物按目录组织;有失败清单文件。

### 任务 2.4 失败标记与覆盖率统计
- **做什么**: 失败项回写 `preview_available=false`;输出覆盖率统计(成功/失败/占比);空白图启发式判失败(尺寸/颜色方差)。
- **验收**: 覆盖率报告数字与产物一致;管道不因失败项中断。

### 任务 2.5 抽样集成断言
- **做什么**: 每类抽样若干做 golden 断言(文件存在、尺寸合理);断言失败项被正确标记。
- **验收**: `npm test` 全绿;覆盖率统计可复现。

---

## Ticket 03 规则路检索核心(blocked by 01)

工单目标:「类型词硬过滤 + 风格/特征打分 + 每条 reason;空结果引导;filter(type×style) 计数;fixture 确定性测试」。

### 任务 3.1 索引加载模块
- **做什么**: 浏览器无关 TS 模块,加载 `components.json` + 类型词表 + 风格词典到内存。
- **验收**: 2,708 条记录;6 类计数与 PRD §8.A 一致。

### 任务 3.2 类型词检测与硬过滤
- **做什么**: 实现类型词检测(按钮/按键/button/btn → Buttons 等 PRD §5 映射);命中则候选集限制在该目录;未命中走全库。
- **验收**: PRD §5 每行映射词命中正确;类型词命中时结果不含无关类型。

### 任务 3.3 特征打分与 reason
- **做什么**: 对候选按风格词典/特征重合度打分排序;每条结果生成 reason(类型命中/风格命中/语义相似);默认 Top-24。
- **验收**: 「渐变颜色的按钮」Top 结果以 Buttons∩gradient 为主且带正确 reason。

### 任务 3.4 filter(type×style) 与计数
- **做什么**: 组合筛选查询;每个风格标签返回实时组件计数;未选筛选按类型分区。
- **验收**: 计数与 components.json 一致;无空分类出现。

### 任务 3.5 fixture 测试
- **做什么**: 用 fixture 语料(边界:无 `<style>`、含 JS、含 @keyframes、双注释位置)写 golden 断言;不依赖网络与模型文件。
- **验收**: `npm test` 全绿,覆盖工单 03 全部 AC。

---

## Ticket 04 评测 harness(blocked by 03)

工单目标:「50 条评测集;脚本化三路入口;先跑规则路出 Top-5 报告;双门槛输出」。

### 任务 4.1 评测集编制
- **做什么**: 按 PRD §8.B 编制 50 条(大类加采样、小类固定条数、≥10 条纯语义/抽象 query);存为独立 JSON/MD,不进训练集。
- **验收**: 覆盖 6 类;含「渐变颜色的按钮」「霓虹灯效的开关」「赛博朋克感觉的按钮」等样例;存储位置与训练数据隔离。

### 任务 4.2 评测 runner 框架
- **做什么**: 脚本化 runner,预留三路入口(规则路 / 现成 bge 基座 / 微调双塔);对每条 query 取 Top-5,按类汇总相关率。
- **验收**: runner 可对任意一路跑出分报告;输出格式稳定。

### 任务 4.3 规则路基线报告
- **做什么**: 跑规则路,输出 Buttons 与各小类 Top-5 相关率。
- **验收**: 报告数字可复现;作为后续双塔对照基线。

### 任务 4.4 双门槛判定输出
- **做什么**: 实现双门槛判定:绝对线(Buttons ≥80%、小类 ≥70%)+ 相对优势线(双塔较规则路 ≥10pp);输出「达标/不达标」结论。
- **验收**: 用假数据可验证判定逻辑正确。

---

## Ticket 05 合成训练数据生成器(blocked by 01)

工单目标:「词典 × 特征生成 1–3 万对正负样本;tags 增强;同种子可复现;评测集隔离」。

### 任务 5.1 词典定稿
- **做什么**: 中文风格/颜色/形状词典(梯度/霓虹/玻璃/3D/圆角/胶囊/红色…),与 PRD §5 类型词、§8.C 风格标签对齐。
- **验收**: 词典覆盖 PRD §8.B 样例 query 用词;逐条可溯源到特征。

### 任务 5.2 正负样本生成器
- **做什么**: 词典 × 组件特征组合生成 1–3 万对 (query, 组件) 正负样本;tags 增强;输出 sentence-transformers 可消费格式。
- **验收**: 数量在 1–3 万;正负比例与类型覆盖有统计输出。

### 任务 5.3 统计与隔离守卫
- **做什么**: 输出正负比例、按类型分布;硬断言:50 条评测集 query 不进入训练集;固定种子幂等。
- **验收**: 同种子两次输出一致;隔离断言测试通过。

---

## Ticket 06 双塔微调 + ONNX int8 + 向量重算(blocked by 01, 05)

工单目标:「bge-small-zh LoRA 微调;合并 → ONNX → int8 ≤30MB;重算 vectors.bin(2,708×512);产物版本化」。

### 任务 6.1 云端训练脚本
- **做什么**: Colab/Kaggle 脚本:用合成数据对 bge-small-zh 做 LoRA 微调(仅微调);输出 adapter 权重。
- **验收**: 训练跑通;loss 下降;脚本与主工程分离。

### 任务 6.2 LoRA 合并 + ONNX 导出
- **做什么**: 合并 adapter → 基座;导出 ONNX query 编码器。
- **验收**: ONNX 可被 ort 加载;输入输出维度正确(512)。

### 任务 6.3 int8 量化
- **做什么**: int8 量化,模型 ≤30MB;用 `onnxruntime-web` 在浏览器环境做一次加载验证。
- **验收**: 模型文件 ≤30MB;加载成功且能编码一条中文 query。

### 任务 6.4 组件向量重算与版本化
- **做什么**: 用微调模型批量编码 2,708 个 descriptionDoc → `vectors.bin`(float32,行序与 components.json 一致);产物带版本号与生成时间。
- **验收**: 行数 = 2,708;维度 = 512;版本可追溯、可回滚。

---

## Ticket 08 搜索与分类浏览界面(blocked by 02, 03, 07)

工单目标:「搜索框 + 类型×风格筛选 + 结果网格;命中原因;空结果引导;性能 P50<1.0s」。

### 任务 8.1 搜索框与结果状态
- **做什么**: 首页搜索框(回车/按钮);接检索库 `search(query)`;loading/结果/空结果三态。
- **验收**: 输入「渐变按钮」出结果网格;状态切换无闪烁报错。

### 任务 8.2 筛选轴与计数
- **做什么**: 类型轴仅 6 类 + 「更多类型即将上线」;风格轴标签 + 实时计数;类型×风格组合筛选。
- **验收**: 计数与检索库一致;切换筛选结果正确;无空分类。

### 任务 8.3 结果网格卡片
- **做什么**: 卡片 = webp 缩略图(懒加载)+ 类型标签 + 风格标签 + 命中原因;预览不可用卡片标注但仍可点开。
- **验收**: 网格渲染 ≤24 张/页;懒加载生效;预览不可用卡片可见。

### 任务 8.4 空结果引导
- **做什么**: 无结果显示「放宽描述」提示 + 分类浏览入口(不显示空页)。
- **验收**: 空结果不出现死胡同;入口可直达分类浏览。

### 任务 8.5 性能实测
- **做什么**: Playwright 实测规则路搜索 P50/P95 与首屏(不含模型)耗时;记录进报告。
- **验收**: P50 <1.0s / P95 <2.5s;首屏 ≤5s(≥10Mbps)。

---

## Ticket 09 双塔上线切换 + 三路验收(blocked by 04, 06, 08)

工单目标:「界面启用 ort-web 双塔;失败回退规则路;三路对照 × 双门槛;执行切/不切」。

### 任务 9.1 浏览器端模型加载与编码
- **做什么**: 前端加载 `model_int8.onnx`(强缓存);query 编码(CPU/WebGPU);预热与错误处理。
- **验收**: 编码一次中文 query 成功;缓存生效(二次加载不重复下载)。

### 任务 9.2 双塔检索接入与回退
- **做什么**: 检索库切双塔(向量余弦 + 类型硬过滤);模型缺失/加载失败自动回退规则路,界面无感知。
- **验收**: 断网/删模型场景下搜索仍可用(规则路)。

### 任务 9.3 三路对照评测
- **做什么**: 用评测 harness 跑三路(规则路/现成 bge 基座/微调双塔),出各类 Top-5 报告。
- **验收**: 三路报告齐全且可复现。

### 任务 9.4 双门槛决策执行
- **做什么**: 判定绝对线 + 相对优势线;达标 → 双塔为默认;不达标 → 保持规则路并记录原因。
- **验收**: 决策有据可查,切换可一键执行/回退。

---

## Ticket 10 详情页交互预览 + 复制代码(blocked by 08)

工单目标:「sandbox iframe 交互预览;作者/来源链接;一键复制;预览不可用不阻塞复制」。

### 任务 10.1 详情路由与数据加载
- **做什么**: `/detail/:id` 路由;加载组件记录(components.json 条目 + 原始 HTML 地址)。
- **验收**: 从卡片点入详情,数据正确。

### 任务 10.2 sandbox iframe 预览
- **做什么**: sandbox iframe 加载同源组件 HTML(禁远程内容、禁脚本越权);hover/点击真实生效。
- **验收**: 渐变按钮 hover 态可见;组件脚本无法触及父页面登录态。

### 任务 10.3 作者与来源链接
- **做什么**: 展示类型/风格标签、作者、Uiverse.io 来源链接(新标签页)。
- **验收**: 链接正确可点;作者缺失时有兜底展示。

### 任务 10.4 复制代码
- **做什么**: 「查看代码」+ 一键复制(`navigator.clipboard`,失败降级 textarea 手动复制)。
- **验收**: 复制成功且内容为完整源码。

### 任务 10.5 预览不可用降级
- **做什么**: `preview_available=false` 组件详情页展示「预览不可用」但代码复制可用。
- **验收**: 降级不阻塞复制。

---

## Ticket 11 收藏(blocked by 10)

工单目标:「/api/favorites JWT 鉴权 → Supabase favorites;详情页收藏按钮;收藏视图;按用户隔离」。

### 任务 11.1 favorites API
- **做什么**: Node `GET/POST/DELETE /api/favorites`;JWT → user_id;读写 Supabase `favorites`(PK user_id+component_id)。
- **验收**: 增删查按 user_id 隔离;无/过期 JWT → 401。

### 任务 11.2 详情页收藏按钮
- **做什么**: 详情页收藏/取消收藏按钮;状态即时反馈;卡片可见收藏态。
- **验收**: 收藏后刷新仍为已收藏;取消后消失。

### 任务 11.3 收藏视图
- **做什么**: 收藏页网格,按收藏时间倒序,复用结果卡片。
- **验收**: 收藏列表正确、可点入详情。

### 任务 11.4 隔离与降级测试
- **做什么**: 两个用户收藏互不可见;Supabase 不可达时前端提示「收藏暂不可用」且不阻塞搜索浏览。
- **验收**: 测试覆盖隔离与降级。

---

## Ticket 12 阿里云部署上线(blocked by 10)

工单目标:「Nginx 静态 + Node API;IP:8080 免备案;pooler 6543 + 心跳;安全组白名单;全流程冒烟 + 回滚」。

### 任务 12.1 服务器初始化
- **做什么**: 阿里云轻量 2核2G;安装 Node、Nginx;安全组放行 8080 + 公司出口 IP。
- **验收**: 从公司网络可访问 `http://IP:8080`(先放一个静态页)。

### 任务 12.2 静态站 + Node API 部署
- **做什么**: 构建产物上传;Nginx 静态站(页面/预览图/向量/模型)+ 反代 `/api` 到 Node(systemd/pm2 守护);资产缓存策略(模型/向量强缓存,webp 懒加载)。
- **验收**: `IP:8080` 登录/搜索可用;二次访问资产命中缓存。

### 任务 12.3 Supabase pooler + 心跳 + 安全组
- **做什么**: Node 配 Session Pooler 6543 + 连接池 3–5;每日心跳 `SELECT 1` 防 7 天休眠;service role/JWT 密钥只存服务器环境变量。
- **验收**: 心跳任务运行;密钥不在前端产物中。

### 任务 12.4 全流程冒烟与回滚演练
- **做什么**: 冒烟「登录→搜索→详情→复制代码」;保留上一版产物,演练一次回滚。
- **验收**: 冒烟通过;回滚流程文档化并演练成功。

---

## 质量红线(来自 spec,勿越界)
- 不加规格外功能(注册/找回/第三方登录/管理后台 UI/编辑器)。
- 检索库与 React 解耦;双塔是增强项,规则路先行。
- 所有可自动化的验收必须可无头运行;不依赖网络/生产库。
- 组件预览必须 sandbox iframe 隔离;service role key 不进前端产物。
