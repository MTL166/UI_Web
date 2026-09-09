# 12: 阿里云部署上线

**What to build:** 把网站部署到阿里云轻量服务器并内部分发：Nginx 静态站 + Node API，`http://公网IP:8080` 免备案上线；Supabase 经 Session Pooler 6543 连接并加每日心跳保活；安全组只放行公司出口 IP。冒烟走通「登录 → 搜索 → 详情 → 复制代码」全流程。

**Blocked by:** 10 (详情页交互预览 + 复制代码)

**Status:** ready-for-agent

- [ ] 阿里云轻量 2核2G：Nginx 静态站（页面/预览图/向量/模型）+ Node API（systemd/pm2 守护）
- [ ] Supabase 连接：Session Pooler（6543, transaction 模式）+ 连接池（size 3–5）+ 每日心跳 `SELECT 1` 防 7 天休眠
- [ ] 安全组放行 8080 端口，并只放行公司出口 IP；service role key / JWT 密钥存服务器环境变量
- [ ] 静态资产缓存策略：模型/向量强缓存，webp 懒加载；首屏（含模型下载）P50 ≤ 5s（≥10Mbps）
- [ ] `http://公网IP:8080` 冒烟测试通过：登录 → 搜索 → 详情 → 复制代码
- [ ] 保留上一版产物可回滚；更新流程文档化（内部分发地址变更时通知用户）

## Comments
