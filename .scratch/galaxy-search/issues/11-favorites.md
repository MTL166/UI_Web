# 11: 收藏（P1）

**What to build:** 收藏功能的完整竖切：详情页可收藏/取消收藏，收藏落 Supabase `favorites` 表（服务端按 JWT 识别用户），提供收藏视图按时间倒序回看。收藏请求失败只提示、不阻塞浏览。

**Blocked by:** 10 (详情页交互预览 + 复制代码)

**Status:** ready-for-agent

- [ ] `GET/POST/DELETE /api/favorites` 服务端鉴权（JWT → user_id），读写 Supabase `favorites`（PK = user_id + component_id）
- [ ] 详情页收藏/取消收藏按钮，状态即时反馈；卡片上可见收藏状态
- [ ] 收藏视图：网格卡片、按收藏时间倒序
- [ ] 收藏按用户隔离；未登录/会话过期返回 401
- [ ] Supabase 不可达时前端提示「收藏暂不可用」，不影响搜索浏览

## Comments
