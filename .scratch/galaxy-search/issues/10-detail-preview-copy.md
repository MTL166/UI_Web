# 10: 详情页交互预览 + 复制代码

**What to build:** 点击结果卡片进入详情页的完整竖切：sandbox iframe 加载组件原始 HTML，hover/点击真实交互生效；展示类型/风格标签、作者与 Uiverse.io 来源链接；一键复制完整代码。预览不可用的组件仍可查看与复制代码。

**Blocked by:** 08 (搜索与分类浏览界面)

**Status:** ready-for-agent

- [ ] 点击卡片进入详情页，展示交互预览（sandbox iframe 加载原始 HTML，hover/点击生效）
- [ ] iframe 隔离：禁远程内容加载、禁脚本越权，组件代码不能触及登录态与页面数据
- [ ] 展示类型/风格标签、作者与 Uiverse.io 来源链接（新标签页打开）
- [ ] 「查看代码」展示完整源码并可一键复制（`navigator.clipboard`，失败降级手动复制）
- [ ] `preview_available=false` 的组件详情页不阻塞复制代码

## Comments
