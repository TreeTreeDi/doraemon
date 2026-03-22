# Doraemon Skills CLI 安装闭环实施方案

日期：2026-03-21

关联调研：
- [Survey A: SkillHub/skillhub CLI 模式调研](/Users/dsy/code/doraemon/docs/research/2026-03-21-survey-a-skillhub-cli-mode.md)

## 1. 目标

本方案不是复刻 SkillHub 的品牌或全部生态，而是复刻它已经被验证有效的最小安装闭环：

```text
前端展示安装入口
    ->
后端按 slug 提供安装材料
    ->
CLI 执行下载、解压、落盘
    ->
本地 skills 目录出现目标 skill
```

对 Doraemon 来说，本轮要完成的是：

1. 把 Skills 详情承载从独立详情页升级为首页 modal。
2. 在 modal 中提供 `Human / Agent` 两类安装方式。
3. 引入 Doraemon 自己的 CLI 安装主线：

```bash
doraemon-skills install <slug>
```

4. 让前端、后端、CLI 三层围绕同一套安装语义工作。

## 2. 当前状态

当前仓库已经具备部分能力，但还没有“平台分发包安装”这条完整主线。

### 2.1 已有能力

- Skills 列表、详情、相关技能、文件内容接口已存在：[app/router.js:152](/Users/dsy/code/doraemon/app/router.js#L152) 到 [app/router.js:158](/Users/dsy/code/doraemon/app/router.js#L158)
- 已有技能下载接口：`GET /api/skills/download`，[app/router.js:156](/Users/dsy/code/doraemon/app/router.js#L156)
- 已有 zip 打包逻辑：`getSkillArchive(slug)`，[app/service/skills.js:328](/Users/dsy/code/doraemon/app/service/skills.js#L328)
- 列表页目前点击卡片进入独立详情页：[app/web/pages/skills/index.tsx:228](/Users/dsy/code/doraemon/app/web/pages/skills/index.tsx#L228) 到 [app/web/pages/skills/index.tsx:285](/Users/dsy/code/doraemon/app/web/pages/skills/index.tsx#L285)
- 详情页已具备：
  - 详情与相关技能加载：[app/web/pages/skills/detail/index.tsx:217](/Users/dsy/code/doraemon/app/web/pages/skills/detail/index.tsx#L217)
  - 文件内容懒加载：[app/web/pages/skills/detail/index.tsx:261](/Users/dsy/code/doraemon/app/web/pages/skills/detail/index.tsx#L261)
  - 安装命令展示：[app/web/pages/skills/detail/index.tsx:472](/Users/dsy/code/doraemon/app/web/pages/skills/detail/index.tsx#L472)
  - 下载命令展示：[app/web/pages/skills/detail/index.tsx:498](/Users/dsy/code/doraemon/app/web/pages/skills/detail/index.tsx#L498)

### 2.2 当前缺口

- 还没有 Doraemon 自己的 CLI 安装器。
- 还没有给 CLI 消费的稳定安装元信息协议。
- 前端当前主安装语义仍偏向 `npx skills add ...`：[app/service/skills.js:516](/Users/dsy/code/doraemon/app/service/skills.js#L516)
- 详情当前由独立路由承载，不符合本轮想模仿的 modal 交互。

## 3. 目标架构

### 3.1 总体链路

```text
Skills 首页卡片
    ->
打开详情 modal
    ->
用户选择 Human / Agent
    ->
复制命令块或指令块
    ->
CLI 根据 slug 请求安装元信息
    ->
CLI 下载 zip 包
    ->
CLI 解压并安装到本地 skills/<slug>
```

### 3.2 分层职责

#### 前端

- 展示 skill 基础信息
- 展示安装方式 tabs
- 复制 Human 命令块
- 复制 Agent 指令块
- 下沉展示高级内容：文件树、`SKILL.md`、相关技能

#### 后端

- 根据 `slug` 返回 skill 详情
- 根据 `slug` 返回下载包
- 新增适合 CLI 的安装元信息接口

#### CLI

- 接收 `install <slug>`
- 调用后端 `install-meta`
- 下载 zip 包
- 解压到临时目录
- 落盘到目标安装目录
- 输出明确结果和错误

## 4. 关键决策

本轮先冻结以下决策，避免实现时返工。

### 4.1 CLI 主命令

建议采用：

```bash
doraemon-skills install <slug>
```

第一版最小命令集建议是：

- `doraemon-skills install <slug>`
- `doraemon-skills list`

`upgrade/search/remove/self-upgrade` 不纳入本轮闭环目标。

### 4.2 安装协议

建议新增：

```text
GET /api/skills/install-meta?slug=<slug>
```

返回最小字段：

```json
{
  "slug": "summarize",
  "name": "Summarize",
  "downloadUrl": "http://host/api/skills/download?slug=summarize",
  "packageType": "zip",
  "packageVersion": "v1",
  "packageRootMode": "find-skill-md",
  "installDirName": "summarize",
  "version": "",
  "sha256": "",
  "sourceRepo": ""
}
```

设计原则：

- `downloadUrl` 由后端提供，不让 CLI 自己拼域名和路由。
- `packageType` 第一版固定为 `zip`。
- `packageRootMode` 第一版固定为 `find-skill-md`，要求 CLI 以 `SKILL.md` 所在目录作为真正的 skill 根目录，而不是直接把解压临时目录整体 move。
- `installDirName` 由后端明确给出，不让 CLI 猜目录名。
- `sha256` 和 `version` 第一版可以为空，但字段预留。

### 4.3 安装根目录与覆盖策略

第一版冻结如下：

- 默认安装根目录：当前工作目录下的 `./skills`
- CLI 支持显式覆盖目标根目录，例如 `--dir <path>`
- 默认不覆盖已存在目录
- 第二阶段再考虑 `--force`

原因：

- 这和当前本机 `skillhub` CLI 的默认行为一致，更容易让用户理解
- 也更贴近当前 Doraemon 仓库本地协作模式
- 可以避免第一版就引入全局目录、锁文件、升级语义

### 4.4 来源类型策略

第一版按“是否能稳定生成下载包”来决定是否支持 Doraemon CLI 安装，不按 `sourceRepo` 是否存在来决定。

冻结规则：

- 远程来源 skill：支持 Doraemon CLI 安装
- 上传来源 skill：如果 `downloadUrl` 可用，也支持 Doraemon CLI 安装
- 无法生成稳定下载包的 skill：前端降级为“仅支持下载 zip / 手动处理”，不展示 Doraemon CLI 主安装命令

原因：

- 当前下载链路是按 `slug` 打包，不依赖 `sourceRepo` 才能工作
- 如果把来源类型和 CLI 能力直接绑定，会把上传 skill 错误排除在外

### 4.5 CLI 分发方式

第一版建议采用和 SkillHub 类似的“curl 安装脚本”方式分发 Doraemon CLI。

目标文案形态：

```bash
curl -fsSL <doraemon-cli-install-url> | bash
```

原因：

- 最接近用户已经看到并理解的安装方式
- 能自然承接 `Human` tab 第一条命令
- 比第一版就发 npm 包或二进制更轻

说明：

- 本轮只冻结“安装方式”，不要求现在就实现完整 CLI 自升级
- 如果后续改成 npm 包，也可以保留前端结构不变，只替换命令块文案

### 4.6 前端安装语义

第一版前端采用“双 tab + 不自动执行”的语义。

#### Human

给用户 shell 命令块：

1. 安装 Doraemon CLI
2. 安装当前 skill

推荐形态：

```text
在终端中执行以下命令，安装 Doraemon CLI
[命令块 + 复制]

在终端中执行以下命令，安装当前技能
[命令块 + 复制]
```

#### Agent

给用户可复制的指令块，而不是前端直接触发自动安装：

```text
请先检查 Doraemon CLI 是否已安装。
若未安装，请执行以下命令安装 Doraemon CLI：
<cli install command>

安装完成后，执行：
doraemon-skills install <slug>

若目标安装目录不明确，请先询问用户，不要自行猜测。
```

#### 本轮不做

- 前端一键拉起本地 CLI
- 前端驱动 Agent 自动执行安装
- 仿 SkillHub 的外部商店/CLI 安装逻辑

## 5. Modal 结构

目标不是复刻 SkillHub 的品牌样式，而是复刻它的安装决策顺序。

```text
基础信息
  ->
安装方式选择
  ->
执行内容复制
  ->
补充说明
  ->
高级内容（文件树 / SKILL.md / 相关技能）
```

### 5.1 推荐区块

1. 顶部摘要区
- 名称
- slug
- 一句话描述
- tags / 分类
- 关闭按钮
- 来源仓库 / 更新时间

2. 状态区
- Stars
- 文件数或更新时间
- 来源类型

说明：
- 只展示 Doraemon 当前真实有的数据
- 不伪造下载量、安装量、收藏量

3. 安装方式区
- 标题：`安装方式`
- tabs：`我是 Human` / `我是 Agent`

4. 安装内容区
- 命令块或指令块
- 复制按钮
- 极短说明文案

5. 高级内容区
- 文件树
- `SKILL.md` frontmatter 表格
- Markdown 正文
- 相关技能

### 5.2 结构图

```text
[Header]
name / slug / desc / meta / close

[Install Section]
tab: Agent | Human

[Action Blocks]
copyable commands or instructions

[Advanced Section]
file tree
frontmatter
markdown body
related skills
```

## 6. 实施分解

本需求拆成 4 个工作包。

### WP1: 后端安装协议

目标：
- 保留现有 `download` 能力
- 新增 `install-meta`
- 明确 zip 包结构约束

交付：
- 新路由
- 新 controller 方法
- 新 service 方法
- 错误处理与字段说明

### WP2: CLI 安装器

目标：
- 建立 `doraemon-skills install <slug>` 的最小闭环

交付：
- `install`
- `list`
- 安装根目录策略
- 覆盖策略
- 明确日志输出

### WP3: 前端 modal 与安装区

目标：
- 把详情承载切换为 modal
- 实现 `Human / Agent` 安装区
- 把高级内容下沉展示

交付：
- modal 壳子
- 顶部摘要区
- tab 区
- 命令块/指令块组件
- 高级内容复用

### WP4: 联调与验收

目标：
- 跑通前端 -> CLI -> 本地落盘闭环

交付：
- 验收记录
- 浏览器截图
- CLI 运行结果样例

## 7. 串行 / 并行关系

### 7.1 必须串行

这些任务必须按顺序推进。

1. 定义安装协议
- 决定 CLI 主命令
- 决定安装根目录
- 决定覆盖策略
- 决定 `install-meta` 字段

2. 后端补齐 CLI 契约
- 先有 `install-meta`
- CLI 才能基于稳定协议开发

3. CLI install 主链路联调
- 真实验证下载、解压、落盘路径

4. 前端冻结最终安装文案
- 因为最终文案取决于 CLI 主链路是否成立

### 7.2 可以并行

这些任务可以平行推进，但不要提前冻结最终语义。

1. 前端 modal 外壳和布局
- 弹窗容器
- 顶部摘要区
- tabs
- 复制按钮

2. 高级内容复用
- 文件树
- frontmatter 表格
- Markdown 正文
- 相关技能

3. 安装区组件骨架
- `InstallSection`
- `InstallTabPanel`
- `CommandBlock`
- `InstructionBlock`

说明：
- 这里只允许做到“占位层并行”
- 最终展示的命令块数量、命令内容、降级文案，必须等 CLI 主命令、`install-meta` 字段和安装根目录策略冻结后再定
4. CLI 本地目录与锁文件草案
- 可以先讨论和预设计
- 但编码仍以协议冻结后为准

### 7.3 推荐执行顺序

```text
1. 定协议
2. 后端 install-meta
3. CLI install
4. 前端接入安装命令/指令
5. 联调验收
```

## 8. 执行阶段建议的 Sub Agent 分工

本节不是现在立刻执行，而是后续进入实现阶段时的推荐拆法。

### 推荐数量

建议 3 个执行子代理 + 1 个验收/审查子代理。

### Agent A: 后端协议

负责范围：
- `app/router.js`
- `app/controller/skills.js`
- `app/service/skills.js`

任务：
- 新增 `install-meta`
- 明确 zip 结构契约
- 错误处理

### Agent B: CLI 工具

负责范围：
- 新增 CLI 目录或脚本
- 安装根目录策略
- 下载、解压、落盘实现

任务：
- `install`
- `list`
- `--force` 是否纳入第一版

### Agent C: 前端 modal

负责范围：
- `app/web/pages/skills/index.tsx`
- `app/web/pages/skills/detail/*`
- `app/web/pages/skills/style.scss`

任务：
- modal 化
- 安装区
- Human / Agent tabs
- 高级内容复用

### Agent D: 审查/验收

负责范围：
- 不拥有主文件
- 负责检查：
  - 文案与命令是否一致
  - 前端展示与 CLI 行为是否一致
  - 回归点是否遗漏

### 当前阶段说明

当前仍处于“计划冻结”阶段，不进入并行编码。
只有在本方案确认后，才进入上述执行拆分。

## 9. 验收标准

### 9.1 后端验收

- `GET /api/skills/install-meta?slug=<slug>` 返回字段完整。
- `downloadUrl` 可直接访问。
- `GET /api/skills/download?slug=<slug>` 返回合法 zip 包。
- zip 内目录结构稳定，可被 CLI 一致识别。
- 缺失 `slug`、不存在 `slug`、非法路径时有明确错误。

### 9.2 CLI 验收

- 执行：

```bash
doraemon-skills install summarize
```

可以输出：

```text
Downloading: ...
Installed: summarize -> <path>
```

- 安装结果真实落在目标目录，例如：

```text
<install-root>/summarize
```

- 重复安装时行为明确：
  - 默认拒绝覆盖并给出提示
  - 或通过 `--force` 覆盖

- 下载失败、非 zip、解压失败、目标目录冲突时，错误信息可读。
- 对至少一个真实 slug，CLI 能正确识别解压后的真正 skill 根目录，而不是错误地多包一层目录。

### 9.3 前端单体验收

1. 点击 skill 卡片后，打开 modal，不再依赖独立详情页承载。
2. modal 首屏能看到：
- 名称
- slug
- 描述
- 基础元信息
- 安装方式 tabs

3. `Human` tab 至少包含：
- CLI 安装命令块
- 当前 skill 安装命令块
- 两块都可复制

4. `Agent` tab 至少包含：
- 一段可复制的 Agent 指令块
- 文案中正确注入当前 `slug` / `name`

5. 高级内容区仍正常：
- 文件树切换预览正常
- `SKILL.md` frontmatter 表格正常
- Markdown 正文正常
- 旧的独立详情页是否保留必须有明确策略；如果保留，则原路由仍可访问；如果不保留，则要有明确回退方案

### 9.4 联调验收

- 前端展示的安装命令与 CLI 实际命令一致。
- 从前端拿到的 `slug`，CLI 能真实完成安装。
- 至少用一个真实 skill 跑通闭环：

```text
前端查看
-> 复制命令
-> 终端执行
-> 成功安装到目标目录
```

- 至少验证 2 类 skill 的策略：
  - 远程来源 skill
  - 上传来源或无 `sourceRepo` skill

- 上传来源或无 `sourceRepo` skill 必须明确：
  - 支持 Doraemon CLI 安装
  - 或前端降级展示
  - 不允许含糊

### 9.5 浏览器验收

按仓库约束，Skills 页面改动后至少验证：

- 文件树切换预览正常
- 安装命令可复制
- Doraemon CLI 安装命令可复制

并且要用真实浏览器完成一次验收，保留截图证据。

## 10. 本轮完成定义

如果以下条件全部满足，则认为本轮需求完成：

1. Doraemon 有自己的 `install <slug>` CLI 主线。
2. 后端能稳定提供 `install-meta + download`。
3. 前端 Skills 详情已 modal 化。
4. `Human / Agent` 安装区以 Doraemon 自己的 CLI 语义展示。
5. 至少一个真实 slug 能跑通安装闭环。

## 11. 按 Superpower 的下一步

当前阶段对应：

- `Survey`：已完成
- `Plan`：本文件即为实施计划冻结稿
- `Execute`：待本文件确认后开始
- `Verify`：执行完成后进入

因此，按 superpower 的顺序，下一步应该是：

1. 以本文件为准冻结协议与范围
2. 再进入执行拆分
3. 按 WP1 -> WP2 -> WP3 -> WP4 推进

一句话结论：

> 先冻结 Doraemon 自己的安装语义，再开始并行实现；不要在 CLI、后端、前端三层协议未统一前直接开工。

## 12. 执行决策表

本节用于让新线程或新 agent 能直接进入执行，而不是重新讨论语义。

### 12.1 `install-meta` 返回决策表

| 场景 | HTTP | success | 处理要求 |
|------|------|---------|----------|
| `slug` 存在且支持 Doraemon CLI 安装 | `200` | `true` | 返回完整 `install-meta` |
| `slug` 不存在 | `404` | `false` | 返回明确错误，前端与 CLI 都按“技能不存在”处理 |
| `slug` 存在但当前不支持 Doraemon CLI 安装 | `200` | `true` | 返回 `installable: false` 和 `reason`，前端降级，CLI 拒绝继续 |
| 参数缺失或非法 | `400` | `false` | 返回明确错误，不做兜底猜测 |

建议成功返回在现有字段基础上加：

```json
{
  "installable": true,
  "reason": ""
}
```

不支持安装时：

```json
{
  "installable": false,
  "reason": "upload package unavailable"
}
```

### 12.2 来源类型处理表

| 来源类型 | `downloadUrl` 可用 | Doraemon CLI 安装 | 前端展示 |
|----------|--------------------|-------------------|----------|
| 远程来源 skill | 是 | 支持 | 展示 Doraemon CLI 主安装命令 |
| 上传来源 skill | 是 | 支持 | 展示 Doraemon CLI 主安装命令 |
| 任意来源 skill | 否 | 不支持 | 降级为“下载 zip / 手动处理” |

规则说明：

- 是否支持 CLI 安装，以“平台能否稳定提供下载包”为准。
- 不以 `sourceRepo` 是否存在为唯一判断依据。

### 12.3 旧详情页路由策略表

| 项目 | 决策 |
|------|------|
| 首页 skills 列表主交互 | 改为打开 modal |
| `/page/skills/:slug` 是否保留 | 保留，作为深链与回退入口 |
| modal 是否完全替代旧页 | 否，替代首页主交互，不替代深链能力 |
| 旧详情页与 modal 的关系 | 复用同一套详情数据与主体渲染组件 |

执行要求：

- 新线程实现时，不得删除原详情页能力。
- 正确做法是“抽壳复用”，不是“复制一份详情逻辑”。

### 12.4 CLI 路径与参数规则表

| 项目 | 冻结决策 |
|------|----------|
| 默认安装根目录 | 当前工作目录下的 `./skills` |
| 自定义安装根目录 | 支持 `--dir <path>` |
| 相对路径基准 | 基于执行命令时的当前工作目录 |
| 目标目录不存在 | CLI 负责创建 |
| 目标目录已存在 | 第一版默认拒绝覆盖 |
| `list` 读取目录 | 与 `install` 相同根目录；默认 `./skills`，可由 `--dir` 覆盖 |

### 12.5 包结构识别规则表

| 项目 | 冻结决策 |
|------|----------|
| 包格式 | `zip` |
| 根目录识别方式 | `packageRootMode = find-skill-md` |
| 真实 skill 根目录 | 解压结果中 `SKILL.md` 所在目录 |
| 多一层顶层目录时 | CLI 继续向下定位 `SKILL.md`，不直接整体 move 解压目录 |
| 未找到 `SKILL.md` | 视为非法包，CLI 失败并输出明确错误 |

## 13. Handoff 建议

### 13.1 当前线程建议

建议当前线程在此结束，不继续进入编码。

原因：

- 当前线程已完成 `Survey + Plan` 两个阶段。
- 继续在这个线程里做实现，会把“调研上下文、计划上下文、执行上下文”混在一起。
- 目前最适合做的是基于本文件开一个全新执行线程。

### 13.2 下一线程目标

下一线程只做：

```text
WP1: 后端安装协议
```

范围：

- 新增 `install-meta`
- 明确成功/失败返回
- 明确 `installable` 语义
- 明确包根目录识别协议字段

不做：

- 前端 modal
- CLI 工具
- Agent/Human 最终文案

### 13.3 下一线程启动输入

建议新线程只带下面这段最小输入：

```text
请按文档执行：
/Users/dsy/code/doraemon/docs/research/2026-03-21-implementation-plan-doraemon-skills-cli-install.md

当前阶段：
Execute

本轮只做：
WP1 后端安装协议

目标：
实现 install-meta，并以文档第 12 节决策表为准。

不做：
前端、CLI、联调。
```

### 13.4 后续线程顺序

建议严格按下面顺序开线程：

1. `WP1 后端安装协议`
2. `WP2 CLI install 主链路`
3. `WP3 前端 modal 与安装区`
4. `WP4 联调与浏览器验收`

### 13.5 每轮线程结束时的要求

每完成一个 WP，都要把结果回填到本文件或新增执行记录，至少写清楚：

- 已完成内容
- 未决问题
- 对下一 WP 的输入

这样后续线程只依赖文档，不依赖长聊天历史。
