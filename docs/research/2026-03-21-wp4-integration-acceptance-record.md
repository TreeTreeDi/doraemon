# WP4 执行记录：联调与验收

日期：2026-03-21
工作包：WP4（Agent D / 主线程收敛）

## 已完成

- 使用本地服务完成真实联调：
  - 启动命令：
    - `NODE_OPTIONS=--openssl-legacy-provider ./node_modules/.bin/egg-scripts start --env local --workers=1 --port=7001 --title doraemon-skills-wp4`
  - 服务健康检查：
    - `curl -I http://127.0.0.1:7001`
    - 返回 `302 /page/toolbox`
- 使用真实数据验证后端协议：
  - `GET /api/skills/list?pageNum=1&pageSize=5`
  - 获取真实 slug：`upload-skill-creator-default-skill-creator`
  - `GET /api/skills/install-meta?slug=upload-skill-creator-default-skill-creator`
  - 返回 `installable=true`，并包含可下载 `downloadUrl`
- 使用真实 slug 跑通 Doraemon CLI 安装闭环：
  - 执行：
    - `./bin/doraemon-skills install upload-skill-creator-default-skill-creator --server http://127.0.0.1:7001 --dir /tmp/doraemon-skills-e2e`
  - 输出包含：
    - `Downloading: http://127.0.0.1:7001/api/skills/download?slug=upload-skill-creator-default-skill-creator`
    - `Installed: upload-skill-creator-default-skill-creator -> /tmp/doraemon-skills-e2e/upload-skill-creator-default-skill-creator`
  - 结果校验：
    - `./bin/doraemon-skills list --dir /tmp/doraemon-skills-e2e`
    - 输出：`upload-skill-creator-default-skill-creator`
    - 安装目录存在：`/tmp/doraemon-skills-e2e/upload-skill-creator-default-skill-creator/SKILL.md`
- 使用真实浏览器完成前端验收：
  - 首页 `/page/skills` 可打开 skill 详情 modal
  - modal 中可见：
    - `我是 Human / 我是 Agent` tabs
    - frontmatter 表格
    - Markdown 正文
    - 文件树与预览区
  - Human tab 复制校验：
    - Doraemon CLI 安装说明复制后剪贴板内容为：
      - `# 待提供：Doraemon CLI 安装脚本 URL（例如 curl -fsSL <...> | bash）`
    - 当前 skill 安装命令复制后剪贴板内容为：
      - `doraemon-skills install upload-skill-creator-default-skill-creator --server http://127.0.0.1:7001`
  - Agent tab 复制校验：
    - 复制后剪贴板内容包含：
      - 先检查 `doraemon-skills --help`
      - 若未安装，先执行 Human 区 CLI 安装步骤
      - 当前技能安装命令
      - `--dir <path>` 提示
  - 独立详情页深链仍可用：
    - modal 内“打开独立详情页”跳转后，浏览器 URL 为：
      - `http://127.0.0.1:7001/page/skills/upload-skill-creator-default-skill-creator`
- 浏览器截图证据已保存：
  - modal：`/Users/dsy/.agent-browser/tmp/screenshots/screenshot-1774087551307.png`
  - 独立详情页：`/Users/dsy/.agent-browser/tmp/screenshots/screenshot-1774087803190.png`

## 计划对齐补充

本记录作为执行期补充，用于覆盖计划文档中已被真实实现收敛的运行时细节。

- 对 9.2 CLI 验收的补充：
  - 实际验收命令不是裸 `doraemon-skills install <slug>`
  - 当前实现要求：
    - 显式传 `--server <url>`
    - 或预先设置 `DORAEMON_SKILLS_SERVER`
  - 第一版 CLI 不猜测服务地址，这是 WP2 已冻结的执行决策
- 对 9.3 前端单体验收的补充：
  - Human tab 的第二块“当前 skill 安装命令块”是可直接执行命令
  - Human tab 的第一块“Doraemon CLI 安装命令块”当前仍为占位说明，不是可执行 bootstrap 命令
  - 原因是仓库内尚未提供 Doraemon CLI 安装脚本 URL，前端没有伪造地址
- 对 12.2 来源类型处理表的补充：
  - 当前实现的可安装判断，不仅要求能下载 zip
  - 还要求平台返回的包结构可被 CLI 通过 `packageRootMode=find-skill-md` 稳定识别
  - 执行期语义以 `install-meta.installable/reason/packageRootMode` 为准

## 未决问题

- Doraemon CLI bootstrap 安装地址尚未提供，因此 Human 区第一块仍为占位说明；若后续补齐安装脚本 URL，前端可直接替换为真实 bootstrap 命令。
- 浏览器自动化已确认文件树与预览区可见，但未能通过 `agent-browser` 拿到树节点 ref 做一次自动切换；当前证据为真实页面渲染截图，若要把“切换预览”也做成自动化断言，需补充更细的浏览器操作能力。
- `$ download --local` 区的下载命令复制按钮在本轮浏览器自动化中未能取得确定性的剪贴板变化证据；源码已接入与其他命令块相同的 `copyToClipboard`，但该项仍建议在后续补一轮人工点击复核。

## 对后续输入

- 若要完全满足计划文档 9.3 的“两个 Human 命令块都可直接执行”，下一步只需补齐 Doraemon CLI 安装脚本 URL，不需要改动当前 install-meta / CLI / modal 主链路。
- 若要收紧验收标准，建议新增一轮专门的浏览器操作脚本，覆盖：
  - 文件树切换预览
  - `$ download --local` 下载命令复制
- 本轮已完成最小闭环，后续新增 skill 时应继续以 `install-meta` 为单一语义源，避免前端、CLI、后端各自推断安装方式。
