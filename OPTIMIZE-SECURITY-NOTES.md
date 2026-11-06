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
