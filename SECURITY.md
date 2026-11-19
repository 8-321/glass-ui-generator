# 安全审查（Glass UI Generator）

审查对象：`index.html`（单文件，无构建、无后端）
线上地址：https://8-321.github.io/glass-ui-generator/
审查日期：2026-11-19

## 1. 结论

| 项目 | 结果 |
|---|---|
| 密钥 / 凭据泄露 | 0（源码 + git 历史都扫过） |
| 第三方资源请求（CDN / 字体 / 图片 / 统计） | 0 |
| 内联事件处理器（onclick= 等） | 0 |
| `eval` / `new Function` / `document.write` | 0 |
| `localStorage` / `cookie` / `fetch` / `XMLHttpRequest` | 0 |
| 用户输入进入 DOM | 只经 `esc()` 转义后写入，实测注入 `<img onerror>` 不执行 |
| 外链 | 2 个（Capafy 商品页、mailto），均带 `rel="noopener noreferrer"` |
| 数据外传可能 | 无。CSP `connect-src 'none'` 直接禁掉一切网络请求 |

页面是纯前端单文件：没有账号、没有上传、没有表单提交，因此不存在服务端注入、越权、会话类风险。

## 2. 已加固

1. `Content-Security-Policy`（meta）
   `default-src 'none'; img-src 'self' data:; style-src 'unsafe-inline'; script-src 'unsafe-inline'; connect-src 'none'; base-uri 'none'; form-action 'none'`
2. `<meta name="referrer" content="no-referrer">` —— 点外链不带来源页。
3. `esc()` 完整转义 `& < > " '`（修掉了单引号未转义的 bug）。
4. 邮箱地址运行时拼接，源码里不再有完整 `mailto:` 字面量（只防爬虫，不是安全边界）。
5. 外链 `target="_blank"` 全部配 `rel="noopener noreferrer"`。
6. 键盘焦点 `:focus-visible` 描边（可访问性，非安全项，一并记录）。

## 3. 残留风险（已知、可接受）

| 风险 | 说明 | 取舍 |
|---|---|---|
| CSP 含 `'unsafe-inline'` | 单文件内联 `<script>`/`<style>` 必须放行 | 无用户内容进入脚本；若要更严，可拆成外部 `.js/.css` 后用 hash 白名单 |
| meta 形式的 CSP 不支持 `frame-ancestors` | GitHub Pages 无法自定义响应头，页面理论上可被 iframe 嵌套 | 页面没有敏感操作（无登录/支付/写操作），点击劫持无收益 |
| mailto 地址可被还原 | 拼接只是减速混淆 | 本来就是公开联系邮箱 |
| 依赖 GitHub Pages | HTTPS 由平台提供；平台不可用时页面不可用 | 静态托管的选择，无数据损失风险 |

## 4. 复现审查

```powershell
cd D:\兼职赚钱平台\web-glass

# 密钥类关键词（应只剩 "tokens" 这个词本身）
Select-String -Path index.html -Pattern 'api[_-]?key|secret|password|Bearer|sk-[A-Za-z0-9]|gho_|PRIVATE KEY'

# 危险 API（应为 0）
Select-String -Path index.html -Pattern 'eval\(|new Function|document\.write|localStorage|cookie|fetch\(|XMLHttpRequest'

# 内联事件（应为 0）
Select-String -Path index.html -Pattern 'on(click|load|error|mouseover)\s*='

# 外链（应只有 2 个）
Select-String -Path index.html -Pattern 'https?://'

# git 历史里的密钥
git log -p | Select-String -Pattern 'sk-[A-Za-z0-9]{20,}|gho_[A-Za-z0-9]{20,}|PRIVATE KEY'
```

浏览器侧实测（playwright，19 项）：
- 7 surface × 5 theme = 35 组合全部渲染，`backdrop-filter` 均生效
- 文本输入注入 `<img src=x onerror=window.__xss=1>` → 未生成元素、未执行脚本
- 控制台 0 error、0 CSP 违规
- 无横向溢出（1160 / 390 视口）

## 5. 变更记录

- 2026-11-19 首轮静态审查 + 加固 + 19 项浏览器实测
