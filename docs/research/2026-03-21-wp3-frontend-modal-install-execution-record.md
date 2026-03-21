# WP3 执行记录：前端 modal 与安装区

日期：2026-03-21
工作包：WP3（Agent C）

## 已完成

- 首页 `Skills` 列表点击卡片改为打开详情 modal，不再以列表点击直接跳独立详情页。
- 旧路由 `/page/skills/:slug` 保留，作为深链与回退入口。
- 详情主体抽壳为可复用组件，modal 与旧详情页共享同一套主体渲染：
  - 文件树
  - `SKILL.md` frontmatter 表格
  - Markdown 正文
  - 相关技能
- 接入 `GET /api/skills/install-meta`，安装区改为 `我是 Human / 我是 Agent` 两个 tab。
- 安装命令按 WP2 语义展示：
  - `doraemon-skills install <slug> --server <current origin>`
- `installable=false` 时执行降级展示：
  - 不展示正常 Doraemon CLI 安装命令
  - 显示降级原因与 zip 下载/手动处理提示
- modal 中分享行为改为复制深链 URL（`/page/skills/:slug`）。

## 未决问题

- Doraemon CLI 安装脚本 URL 在当前仓库未提供，Human 区目前按“待提供”文案占位，未伪造真实地址。
- 浏览器真实验收与截图证据（FE-ACPT-001）待 WP4 执行。

## 对下一 WP 的输入

- WP4 联调请重点验证：
  - modal 内命令复制可用（Human/Agent）
  - `--server` 参数使用当前页面 origin
  - `installable=false` 的降级分支文案与交互
  - 深链复制后可直接打开旧详情页
