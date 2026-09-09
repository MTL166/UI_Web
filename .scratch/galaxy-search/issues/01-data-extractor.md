# 01: 数据抽取器与特征词典

**What to build:** 跑通 galaxy-main 全部 2,708 个组件的构建期数据抽取：对每个组件产出元数据（类型、作者、tags）与风格特征（gradient/neon/glass/neu/animated/dark/3D/minimal 等 + 颜色/圆角/动画），并生成类型词表与风格词典，使后续检索、训练数据、分类浏览都能基于同一份「代码可判特征」。

**Blocked by:** None (can start immediately)

**Status:** ready-for-agent

- [ ] 遍历 6 类目录（Buttons/loaders/Toggle-switches/Inputs/Checkboxes/Radio-buttons）共 2,708 个 HTML，组件唯一 ID = 文件名（去 `.html`）
- [ ] 解析元数据注释（tags），兼容注释位于文件头与 `<style>` 内两种位置
- [ ] 按 PRD §8.C 特征草案探测风格标签，并解析颜色、圆角、动画等结构化特征
- [ ] 输出 `components.json`（字段：id/type/file/author/tags/styles/colors/rounded/animated/hasJs/previewAvailable/descriptionDoc/sourceUrl）、类型词表、风格词典
- [ ] 管道幂等可重跑：输入不变输出不变（golden 可复现）
- [ ] 脚本输出 6 类计数与风格分布统计，供人工核对与评测集编制

## Comments
