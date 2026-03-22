# Survey A: SkillHub/skillhub CLI 模式调研

日期：2026-03-21

目标：理解用户描述的 SkillHub/`skillhub` CLI 模式，尤其是类似 `skillhub install summarize` -> 下载包 -> 安装到本地 `skills` 目录 这一套机制，对 Doraemon 可借鉴什么。

## 结论先说

如果只看“用户可见行为”，SkillHub 这套模式的核心不是复杂的 agent 自动化，而是把三件事串成一个稳定闭环：

1. 前端和详情页提供可发现的技能入口。
2. 后端按 `slug` 输出一个可下载的技能包。
3. CLI 接住这个下载包，负责落盘、解压、放入本地 `skills` 目录。

对 Doraemon 来说，最值得借鉴的是这条分层：

- 前端负责发现、选择、复制或触发安装。
- 后端负责根据 `slug` 交付技能包。
- CLI 负责本地安装动作。

这比第一版就复刻“selenium + agent 自动执行安装”更贴近现有基础，也更容易落地。

## 1. 这套模式的核心用户体验和最小流程

### 用户体验

从用户给出的终端行为看：

```bash
skillhub install summarize
Downloading: https://lightmake.site/api/v1/download?slug=summarize
Installed: summarize -> /Users/dsy/code/doraemon/skills/summarize
```

用户实际感知到的是一个两段式体验：

1. `install <slug>` 这种极短命令。
2. 工具自动完成“下载 -> 解压 -> 安装到本地目录”。

对用户来说，这套模式解决的是三个需求：

- 不需要手动打开网页找下载地址。
- 不需要手动处理 zip 包。
- 不需要自己判断技能目录应该放在哪里。

### 最小流程

ASCII 流程图：

```text
用户输入 CLI 命令
    |
    v
CLI 解析 slug
    |
    v
CLI 请求后端下载接口
    |
    v
后端返回技能包
    |
    v
CLI 下载到临时目录
    |
    v
CLI 解压并识别 skill 根目录
    |
    v
CLI 拷贝到本地 skills 目录
    |
    v
CLI 输出 Installed 路径
```

如果把它压缩成“最小可实现版本”，其实只需要：

- 一个 `install <slug>` 命令
- 一个 `GET /download?slug=...` 接口
- 一个明确的本地安装目标目录
- 一个 zip 解压 + 拷贝流程

## 2. 从用户提供的线索里可推断出的工具职责边界

### 前端职责

前端负责“发现”和“触发”：

- 展示技能列表、详情、说明和安装入口
- 暴露 `slug`
- 暴露下载入口或复制 CLI 命令入口
- 不直接负责本地安装动作

当前 Doraemon 已经具备这部分的一部分基础：

- Skills API 路由已存在 [app/router.js:152](/Users/dsy/code/doraemon/app/router.js:152)
- 详情页已有下载路径拼装 [app/web/pages/skills/detail/index.tsx:203](/Users/dsy/code/doraemon/app/web/pages/skills/detail/index.tsx:203)
- 详情页已有 `wget` 下载命令生成 [app/web/pages/skills/detail/index.tsx:212](/Users/dsy/code/doraemon/app/web/pages/skills/detail/index.tsx:212)

### 后端职责

后端负责“按技能身份提供安装材料”：

- 根据 `slug` 返回技能详情
- 根据 `slug` 生成可下载技能包
- 可选：补充更适合 CLI 使用的元信息

当前 Doraemon 已经具备核心后端能力：

- 下载路由：`GET /api/skills/download` [app/router.js:156](/Users/dsy/code/doraemon/app/router.js:156)
- 技能包生成：`getSkillArchive(slug)` 将技能文件重新打成 zip [app/service/skills.js:328](/Users/dsy/code/doraemon/app/service/skills.js:328)

这说明 Doraemon 现在已经有“分发能力”，只是还没有“CLI 安装器”。

### CLI 职责

CLI 负责“把下载材料变成本地可用技能”：

- 接收 `slug`
- 调后端下载接口
- 下载 zip 到临时目录
- 解压
- 找到 skill 根目录
- 安装到本地 `skills` 目录
- 输出成功路径或失败原因

这层职责和前端/后端都不同，应该独立存在。否则前端只能做到“下载文件”，做不到“安装完成”。

## 3. Doraemon 如果照搬“可见行为”，最小可实现版本需要哪些能力

### 3.1 前端

前端最小需要：

- Skills 列表/详情页里展示“CLI 安装”入口
- 让用户明确知道安装命令形如：

```bash
doraemon-skills install <slug>
```

或者：

```bash
skillhub install <slug>
```

如果 Doraemon 要自带工具，建议不要直接复用对方名字，避免产品边界混淆。

如果继续做你前面提的 modal 详情页，那么最自然的产品形态是：

- `Human` tab：复制 CLI 安装命令
- `Agent` tab：复制给 agent 的安装指令块

第一版前端不必承担“真正调用本地 CLI”的责任，展示和复制即可。

### 3.2 后端

后端最小需要：

1. 保持现有 `download` 接口可用
2. 明确技能包格式约定
3. 保证 zip 内部目录结构稳定

当前证据表明 Doraemon 已经可以做到：

- 用技能名或 slug 作为 zip 根目录 [app/service/skills.js:342](/Users/dsy/code/doraemon/app/service/skills.js:342)
- 按文件路径恢复技能目录树 [app/service/skills.js:343](/Users/dsy/code/doraemon/app/service/skills.js:343)

但如果要面向 CLI 稳定安装，还需要补强两个点：

- 明确“zip 内哪个目录算 skill 根目录”
- 明确“下载包是否一定包含 `SKILL.md`”

### 3.3 CLI

这是 Doraemon 目前缺少、但也是照搬可见行为时最关键的一层。

CLI 最小需要：

1. 命令解析

```text
install <slug>
```

2. 下载能力

```text
GET /api/skills/download?slug=<slug>
```

3. 文件处理能力

- 下载到临时目录
- 解压 zip
- 识别 skill 根目录
- 安装到目标目录，例如：

```text
<repo>/skills/<slug>
```

4. 输出能力

```text
Downloading: ...
Installed: <slug> -> <path>
```

### 3.4 数据和产品语义

当前 Doraemon 详情数据里已经有一个安装命令字段：

- `installCommand` 暴露给前端 [app/service/skills.js:140](/Users/dsy/code/doraemon/app/service/skills.js:140)
- 其默认生成逻辑是 `npx skills add ... --skill ...` [app/service/skills.js:516](/Users/dsy/code/doraemon/app/service/skills.js:516)

这说明 Doraemon 当前的产品语义更接近：

```text
远程来源仓库 -> npx skills add 安装
```

而你现在想借鉴的 SkillHub 可见行为更接近：

```text
平台下载包 -> 本地 CLI 安装
```

这不是同一条安装路径。

因此，如果 Doraemon 要照搬 SkillHub 的可见行为，建议不要把它理解成“继续优化 installCommand”，而应该理解成：

“新增第二条安装路径：平台分发包 + Doraemon CLI 安装”

## 4. 风险和未知项

### 风险 1：当前只有下载，没有统一安装协议

现有后端能生成 zip 包，但 zip 包本身不是“安装器”。

差的部分包括：

- 安装目标目录规范
- 覆盖更新策略
- 已存在目录时如何处理
- 临时目录清理

### 风险 2：当前产品语义有两条安装路线，容易冲突

现有路线：

```text
sourceRepo -> npx skills add
```

拟引入路线：

```text
download zip -> 本地 CLI install
```

如果不提前定义清楚，前端会出现两个“安装”概念：

- 一个是安装远端 skills
- 一个是安装 Doraemon 平台托管的下载包

必须明确哪个是主路线，哪个是 fallback。

### 风险 3：本地目标目录不稳定

用户示例里安装到了：

```text
/Users/dsy/code/doraemon/skills/summarize
```

但这更像当前演示环境或当前仓库下的约定路径，不一定适合作为通用规则。

如果 Doraemon 真要做 CLI，需要先定义：

- 默认安装到当前工作目录下的 `skills/`
- 还是安装到用户全局目录
- 还是支持 `--target <path>`

第一版建议支持显式目标目录，避免把路径写死。

### 风险 4：Agent 自动执行和 CLI 安装不是同一难度

如果只是复刻可见行为，CLI 安装已经够了。

如果想进一步做：

- agent 自动下载
- agent 自动解压
- agent 自动放到正确目录
- agent 判断依赖是否存在

那复杂度会从“CLI 工具”升级到“安装协议 + 执行代理”。

第一版不建议把这两个问题混在一起。

### 风险 5：下载包格式命名仍然有歧义

当前代码证据是 zip 包，而不是用户口中的 `CIP`。

所以当前最稳的说法应该是：

- Doraemon 已有 zip 下载能力
- 是否要定义新的标准包格式，仍是后续设计问题

不要在方案里把 `CIP` 当成现有已落地事实。

## Doraemon 可借鉴的最小方案

一句话版本：

> 先不要照搬对方的 agent 自动安装链路，只照搬“`install <slug>` -> 下载平台包 -> 本地安装到 skills 目录”这个闭环。

最小方案分层如下：

### 前端

- 在 Skills 详情页或 modal 里展示 Doraemon CLI 安装命令
- 继续保留下载按钮

### 后端

- 复用现有 `GET /api/skills/download`
- 必要时补一个更适合 CLI 的元信息接口，例如：

```text
GET /api/skills/install-meta?slug=<slug>
```

返回：

- 下载 URL
- 推荐目录名
- 包格式版本

### CLI

- 新增一个极小命令：

```bash
doraemon-skills install <slug>
```

功能只做：

- 下载
- 解压
- 安装到目标目录
- 输出 Installed 路径

## 建议的下一步

如果把这份 Survey A 作为后续方案输入，下一步最值得继续收敛的是这三个问题：

1. Doraemon 是否要正式引入“平台分发包安装”作为第二条安装主线
2. CLI 默认安装目录和覆盖策略怎么定义
3. 前端详情页里 `npx skills add` 和 `doraemon-skills install` 谁是主按钮，谁是备选

在这三个问题没有定清之前，不建议直接写前端 `Agent/Human` 双 tab 的最终文案，因为那会把安装语义提前写死。
