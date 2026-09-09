# 04: 评测 harness（50 条评测集 + 三路对照框架）

**What to build:** 编制 50 条人工评测集（按类配比：大类加采样、小类固定条数、≥10 条纯语义/抽象 query），并搭好脚本化评测框架：先跑规则路，后续同一框架接入现成 bge 基座与微调双塔，输出各类 Top-5 相关率报告，支撑「双门槛」验收决策。

**Blocked by:** 03 (规则路检索核心)

**Status:** ready-for-agent

- [ ] 50 条评测集编制完成：覆盖 Buttons/loaders/Toggle-switches/Inputs/Checkboxes/Radio-buttons，含类型+风格、纯语义、抽象风格 query
- [ ] 评测集单独存放，绝不进入训练数据
- [ ] 脚本化跑通规则路评测，输出 Buttons 类与各小类的 Top-5 相关率
- [ ] 框架预留三路入口（规则路 / 现成 bge 基座 / 微调双塔）
- [ ] 输出格式可直接对照双门槛：绝对线（Buttons ≥80%、小类 ≥70%）与相对优势线（双塔较规则路 ≥10pp）

## Comments
