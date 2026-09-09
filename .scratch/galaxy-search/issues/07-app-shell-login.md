# 07: 前端壳 + 登录门禁

**What to build:** React/Vite SPA 骨架与登录门禁的完整竖切：未登录访问任何页面被重定向到登录页；用户名 + 密码经 Node API 校验（Supabase `users` 表 + bcrypt）后签发 httpOnly cookie JWT（30 天）；连续失败锁定防爆破。管理员在 Supabase Table Editor 直接建号，无注册/找回。

**Blocked by:** None (can start immediately)

**Status:** ready-for-agent

- [ ] React/Vite SPA 骨架 + 路由；未登录访问任意页面重定向到登录页
- [ ] `POST /api/login`：查 Supabase `users`（username + bcrypt 校验），成功签发 httpOnly cookie JWT（30 天）
- [ ] 错误凭据统一提示「用户名或密码错误」，不区分用户名不存在或密码错误
- [ ] 连续失败 ≥5 次锁定 15 分钟（429），期间正确密码也被拒并提示稍后再试
- [ ] 登录成功进入主界面；刷新/重开 30 天内免登录；会话过期（401）回到登录页
- [ ] Supabase `users` 表就绪（username 唯一 + password_hash），并实测登录链路延迟（接受 0.7–1.2s 跨境）
- [ ] 无注册入口、无找回入口；service role key 仅存服务器环境变量

## Comments
