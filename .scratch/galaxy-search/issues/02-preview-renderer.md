# 02: 构建期渲染器（全量截图）

**What to build:** 构建期用 headless 浏览器把全部 2,708 个组件渲染成默认态缩略图 webp，失败的组件明确标记「预览不可用」但不阻塞管道，并输出覆盖率统计，作为 ≥85% 发布门槛的数据来源。

**Blocked by:** 01 (数据抽取器与特征词典)

**Status:** ready-for-agent

- [ ] Playwright 全量渲染 2,708 个组件为默认态缩略图 webp
- [ ] 433 个无 `<style>` 的 Tailwind 纯类名组件先注入 Tailwind 运行时再渲染
- [ ] 渲染失败组件标记 `preview_available=false`，不中断整批任务
- [ ] 输出覆盖率统计（成功/失败/占比），支撑 PRD §7 的 ≥85% gate 判断
- [ ] 产物按 `images/{type}/{id}.webp` 组织，缺失即「预览不可用」
- [ ] 渲染器只做抽样集成断言，不做逐组件单元测试

## Comments
