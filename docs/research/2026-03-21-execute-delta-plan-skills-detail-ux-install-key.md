# Execute Delta Plan：Skills 详情交互与安装语义修订

日期：2026-03-21
阶段：Superpower / Execute 中的变更冻结（Delta Plan）

## 1. 当前阶段判断

当前不需要回到 Survey。

当前属于：

- 已完成一次 Execute 闭环后的需求修订
- 需要先冻结新的交互与协议语义
- 然后重新放行后续实现

结论：

- Superpower 当前阶段应视为：
  - `Execute -> Change Request -> Delta Plan`
- 不是重新开大范围方案讨论
- 也不是直接继续改前端样式

## 2. 为什么需要修订

本轮新输入推翻了两处已经实现的行为：

1. 详情交互分工不对
- 当前实现：点击 skill 卡片会打开 modal
- 新要求：点击整个 skill 卡片，应进入独立详情页
- 新要求：点击“查看详情”按钮，才打开独立 modal

2. 安装命令绑定对象不对
- 当前实现：安装命令直接使用 Doraemon 内部存储 slug
- 例如：`upload-skill-creator-default-skill-creator`
- 新要求：安装命令应绑定“用户感知名称对应的安装标识”
- 例如页面展示为 `skill-creator`，安装命令也应直接使用 `skill-creator`
- 不能把上传链路生成的存储 slug 暴露给用户

3. Human / Agent 安装方式展示需要按参考图收敛
- Human：
  - 先安装 Doraemon CLI
  - 再安装当前技能
- Agent：
  - 给 Agent 一段可复制指令
  - 语气与 SkillHub 参考实现对齐，但替换为 Doraemon 语义

## 3. 参考图收敛结论

说明：

- 用户已在会话中提供 3 张参考图
- 当前工具不能把聊天附件原图直接落盘到仓库
- 因此本文件将其转写为执行参考，供下一个 agent 直接按此实现

### 3.1 图 1 / 图 3：modal 目标形态

目标不是“把完整详情页整个塞进 modal”。

目标 modal 形态应更接近：

- 顶部摘要信息
  - 图标
  - 名称
  - 安装标识
  - 标签 / 推荐标识 / 版本等轻量信息
- 中部统计信息
  - 下载量
  - 收藏
  - 安装量
- 下部是安装方式
  - `我是 Agent`
  - `我是 Human`
- modal 重点是“快速了解 + 快速安装”
- 不应承载完整高级详情浏览

结论：

- modal 应做成轻量安装弹层
- 独立详情页才承载：
  - 文件树
  - frontmatter 表格
  - Markdown 正文
  - 相关技能

### 3.2 图 2：Agent 文案目标形态

Agent 区应是面向外部 Agent 的一段可复制指令，而不是内部实现说明。

目标结构：

1. 先检查 Doraemon CLI / 商店是否已安装
2. 若未安装，按给定安装文档只安装 CLI
3. 再安装当前技能
4. 若已安装，则直接安装当前技能

## 4. 新冻结决策

### 4.1 交互分工冻结

冻结为：

- 点击整个 skill 卡片：
  - 进入独立详情页 `/page/skills/:slug`
- 点击“查看详情”按钮：
  - 打开轻量 modal
- modal 不再承担完整详情主体
- 独立详情页继续保留完整详情能力

这是一次明确的交互反转，后续实现不得再把“卡片点击=modal”当作主交互。

### 4.2 安装标识冻结

必须新增一个“安装标识”概念，不能再直接暴露内部存储 slug。

建议字段名：

- `installKey`

语义：

- `slug`
  - Doraemon 内部技能存储标识
  - 可保留现状
- `installKey`
  - 用户可见、CLI 可输入的安装标识
  - 应与页面展示名强绑定
  - 例如 `skill-creator`

冻结要求：

- 前端命令块展示 `installKey`
- CLI `install` 支持输入 `installKey`
- 后端负责把 `installKey` 解析到真实 skill
- 前端、CLI 不允许自行推断映射规则

### 4.3 CLI 服务发现冻结

旧实现要求显式传 `--server <url>`。

该行为不再适合作为默认用户路径，因为新要求的 Human 安装方式应为：

1. 执行 CLI 安装脚本
2. 直接执行：
   - `doraemon-skills install <installKey>`

因此需要新增一层 CLI 引导能力，二选一：

1. bootstrap 安装脚本在安装 CLI 时写入默认 server 配置
2. CLI 自带默认 server 解析策略

本次 delta plan 推荐：

- 采用方案 1
- 即：安装脚本负责把 Doraemon server 写入 CLI 配置
- `doraemon-skills install <installKey>` 成为默认展示命令

原因：

- 更接近参考产品的使用方式
- 不把环境特定 server 硬编码进前端展示文案
- 能保留 `--server` 作为高级覆盖参数

### 4.4 Human 安装区冻结

Human 区改为两步：

1. 安装 CLI：

```bash
curl -fsSL https://skillhub-1388575217.cos.ap-guangzhou.myqcloud.com/install/install.sh | bash -s -- --no-skills
```

在 Doraemon 实现中，不应继续展示“待提供 URL”的占位说明。

必须改成 Doraemon 自己的真实安装入口，形式与上面一致：

```bash
curl -fsSL <doraemon-cli-install.sh> | bash -s -- --no-skills
```

2. 安装技能：

```bash
doraemon-skills install <installKey>
```

### 4.5 Agent 安装区冻结

Agent 区目标文案：

```text
请先检查是否已安装 Doraemon CLI 商店，若未安装，请根据 <doraemon-cli-install-doc> 安装 Doraemon 商店，但是只安装 CLI，然后安装 <installKey> 技能。

若已安装，则直接安装 <installKey> 技能。
```

说明：

- 这是一段给 Agent 的可复制指令
- 不再出现内部存储 slug
- 不默认暴露 `--server`

## 5. 协议改动边界

这次不是纯前端改动。

必须先冻结跨层协议，再放行前端。

新增或修订的协议项建议如下：

```json
{
  "slug": "upload-skill-creator-default-skill-creator",
  "installKey": "skill-creator",
  "name": "skill-creator",
  "installable": true,
  "downloadUrl": "...",
  "packageRootMode": "find-skill-md",
  "installDirName": "skill-creator"
}
```

执行约束：

- `installKey` 由后端给出
- `installDirName` 与 `installKey` 对齐
- CLI 安装目标目录默认也应是：
  - `./skills/<installKey>`

## 6. 修订后的执行任务板

### Gate 0：先冻结协议与命令语义

必须先做：

- 冻结 `installKey`
- 冻结 CLI bootstrap 安装入口
- 冻结 CLI 默认 server 发现策略
- 冻结 Human / Agent 文案模板

若这些不冻结，前端不能再继续改。

### WP1-B：后端协议增量修订

负责人：后端协议负责人

范围：

- `app/router.js`
- `app/controller/skills.js`
- `app/service/skills.js`

任务：

- 在 `install-meta` 中增加 `installKey`
- 后端负责 `installKey -> slug` 的唯一映射
- `installDirName` 从内部 slug 调整为 `installKey`
- 保持 `download` 与原详情接口不回归

输出：

- 已完成
- 未决问题
- 对 WP2-B 的输入

### WP2-B：CLI 安装入口增量修订

负责人：CLI 负责人

范围：

- `bin/doraemon-skills`
- `scripts/doraemon-skills.js`
- 新增 Doraemon CLI bootstrap / config 能力

任务：

- `install` 支持 `installKey`
- 允许继续兼容 `--server`，但不作为主展示路径
- 支持读取 bootstrap 写入的默认 server 配置
- 安装目录从 `installKey` 命名
- `list` 默认输出用户可感知的安装目录名

输出：

- 已完成
- 未决问题
- 对 WP3-B 的输入

### WP3-B：前端交互与安装区修订

负责人：前端负责人

范围：

- `app/web/pages/skills/index.tsx`
- `app/web/pages/skills/detail/*`
- `app/web/pages/skills/style.scss`

任务：

- 反转交互：
  - 卡片点击进入详情页
  - “查看详情”按钮打开 modal
- 重做 modal 为轻量安装弹层
- 独立详情页保留高级内容
- Human 区改为：
  - CLI bootstrap 命令
  - `doraemon-skills install <installKey>`
- Agent 区改为参考图文案
- 页面上所有用户可见安装命令统一使用 `installKey`

输出：

- 已完成
- 未决问题
- 对 WP4-B 的输入

### WP4-B：联调与验收

负责人：验收 / 审查负责人

任务：

- 验证卡片点击进入独立详情页
- 验证“查看详情”按钮打开 modal
- 验证 Human 区两段命令均可复制
- 验证 Agent 区文案与真实 CLI 语义一致
- 验证 `doraemon-skills install <installKey>` 可跑通
- 验证不再向用户暴露内部存储 slug

## 7. 串行 / 并行约束

### 必须串行

1. 先冻结 `installKey` 与 CLI bootstrap 策略
2. 再改后端 `install-meta`
3. 再改 CLI 安装入口
4. 最后改前端命令块与交互
5. 最后联调验收

### 可以并行

只有在 Gate 0 冻结之后，以下可以并行：

- modal 轻量视觉壳子
- 独立详情页样式调整
- Agent / Human 文案组件拆分

## 8. 对当前已完成实现的影响评估

保留有效的部分：

- `install-meta` 基础协议框架
- zip 下载与 `packageRootMode=find-skill-md`
- CLI 下载 / 解压 / 落盘主链路
- 独立详情页主体内容复用

需要回退或重做的部分：

- 卡片点击直接打开 modal
- modal 承载完整详情主体
- Human 区占位式 CLI 安装文案
- 用户侧命令直接暴露内部 slug
- Agent 指令中出现 `--server` 作为默认展示路径

## 9. 给下一执行线程的指令

下一线程不要重做 Survey。

下一线程只做：

```text
Gate 0 -> WP1-B
```

先做协议修订，不要先改前端文案。
