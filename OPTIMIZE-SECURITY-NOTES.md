# 加速与安全修补说明（基于 commit 00c1fd2）

## 加速
1. 冷启动不再执行 initDB：新增 settings.schema_version，库结构已是最新时只读一次设置（原先每个新 isolate 串行约 35 条 D1 语句，低流量博客几乎每次访问都是冷启动）。默认设置与索引改为 D1 batch。
2. 新增索引 (status, published_at DESC)，匹配首页列表查询。
3. marked / highlight.js / DOMPurify / Vue / axios 改为 public/vendor 同源自托管，不再依赖 cdnjs；文章页三个脚本并行加载并 preload。
4. public/_headers：/vendor/* 一年强缓存，/icon/* 7 天。

## 安全
1. 文章正文插入前经 DOMPurify 3.2.6 净化（原先 marked 输出直接 innerHTML）。
2. 修复 </script> 闭合注入：标题、标签、正文嵌入 <script> 时转义 <（原版标题含 </script> 即可在页面注入 HTML，本地已复现；大小写 </SCRIPT> 也能绕过原来的替换）。
3. 自定义 JS 中含 </script> 不再把页面脚本截断（原版 bug）。
4. R2 图片按扩展名决定 Content-Type，加 nosniff 与 sandbox CSP，直接打开 SVG 不再执行脚本。
5. 全站加 X-Content-Type-Options、Referrer-Policy、X-Frame-Options。

## 未改（建议）
- 管理员令牌仍存 localStorage、48 小时有效、无法单独吊销；更换 ADMIN_PASSWORD 可让所有令牌失效。
- 文章密码仍兼容 ?password= URL 参数（会进日志/历史记录）。
- MCP 写权限 Key 等同管理员写文章，按需开启。

## 第二轮（2026-11-06）
1. RSS 泄露加密文章内容：原版 /rss.xml 不过滤密码文章，会输出正文前 200 字（本地已复现）。现改为“此文章受密码保护”。
2. 文章密码 URL 参数 ?password= 无限流，可无限爆破（本地已复现）。现与 /api/post-auth 共用 5 次/小时限流。
3. 限流并发绕过：原实现读-改-写 settings，20 个并发错误登录全部放行、无一 429（本地已复现）。改为独立 rate_limits 表“先插入后计数”，同样测试 5 个 401、15 个 429。限流记录不再写入 settings，getSettings 不再随访客 IP 增多变慢。
4. BUG：MCP update_post 只改标题也会用正文覆盖手写摘要。
5. BUG：后台编辑文章会把发布时间截成当天 00:00 UTC，打乱同日文章排序；非法日期导致 500。
6. SCHEMA_VERSION 升为 3：部署后首次访问自动建 rate_limits 表并清理 settings 里的旧限流记录。

## 第三轮（2026-11-06 渗透测试）
- BUG：两篇同标题文章（或标题全是符号）slug 唯一约束冲突，发布直接 500。现冲突时自动加后缀。
- BUG：登录接口收到非 JSON 请求返回 500，现返回 400。
- 渗透测试通过项：15 个管理接口未登录全部 401；8 种伪造令牌全部 401；标题/正文/标签/分类/摘要/封面/站点设置注入 XSS 载荷，headless Chromium 渲染前台、文章页、后台各页均未执行（custom_js 阳性对照可执行，证明测试有效）；MCP 未开启 403、无效 Key 拒绝、只读 Key 无法写；全站密码开启后 API 全部 401、伪造 Cookie 无效；CORS 不带凭据；前端第三方库与 cdnjs 官方文件哈希一致。
