# WP2 执行记录：CLI install 主链路

日期：2026-03-21  
负责人：Agent B

## 已完成

1. 新增可执行 CLI 入口 `doraemon-skills`（`bin/doraemon-skills`），并在 `package.json` 注册 `bin` 字段。
2. 实现 `doraemon-skills install <slug>`：
   - 支持 `--server <url>` 与环境变量 `DORAEMON_SKILLS_SERVER`。
   - 若两者都未提供，直接报错，不猜测服务地址。
   - 严格调用 `GET /api/skills/install-meta?slug=<slug>`。
   - `installable=false` 时拒绝继续并输出 `reason`。
   - 下载 `downloadUrl`，校验 zip，解压并按 `packageRootMode=find-skill-md` 定位 `SKILL.md` 所在目录。
   - 默认安装到 `./skills/<installDirName>`，支持 `--dir <path>` 覆盖根目录。
   - 目标目录已存在时默认拒绝覆盖。
   - 成功输出 `Downloading: ...` 与 `Installed: <slug> -> <path>`。
3. 实现 `doraemon-skills list`：
   - 默认读取 `./skills`，支持 `--dir <path>`。
   - 输出已安装目录列表（每行一个）。
4. 覆盖了核心错误场景输出：
   - server 未提供/URL 非法
   - 请求失败或 meta 非 success
   - `installable=false`
   - 非 zip / 解压失败
   - 找不到 `SKILL.md`
   - 目标目录冲突

## 最小验证

说明：受当前沙箱限制，无法监听本地端口启动 stub HTTP server，因此采用 `node-fetch` mock 方式完成协议级验证。

1. install 成功路径（mock `install-meta + download`）：
   - 输出包含：
     - `Downloading: http://mock.local/api/skills/download?slug=demo`
     - `Installed: demo -> /tmp/doraemon-skills-wp2-test/skills/demo`
2. 冲突目录拒绝覆盖：
   - 同目录重复安装输出：
     - `Error: Target directory already exists: /tmp/doraemon-skills-wp2-test/skills/demo`
3. list 输出：
   - `doraemon-skills list --dir /tmp/doraemon-skills-wp2-test/skills`
   - 输出：`demo`

## 未决问题

1. 当前未引入 `--force`（按 WP2 冻结决策，第一版默认拒绝覆盖）。
2. `packageRootMode` 第一版仅支持 `find-skill-md`，后续若扩展新模式需同步补充分支。

## 对下一 WP 的输入

1. 前端命令文案可直接使用：
   - `doraemon-skills install <slug> --server <url>`
2. 若前端提供“自定义安装目录”提示，可对应 `--dir <path>`。
3. 当前 CLI 不猜 server，前端必须提供可复制的 `--server` 版本命令，或明确要求用户先设置 `DORAEMON_SKILLS_SERVER`。
