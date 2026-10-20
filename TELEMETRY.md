# 遥测

Trail-code 会收集少量**匿名**使用事件，用来判断功能是否正常：有多少仓库能通过构建、agent 是使用 Trail-code 还是退回 grep、哪些命令真正有用、以及实际使用中哪里会出问题。

本文档是完整且权威的约定：**如果某个事件或属性没有列在这里，Trail-code 就不会发送它。** 实现位于 [`src/telemetry/contract.ts`](src/telemetry/contract.ts)，并将这份清单作为硬性允许列表强制执行——未知事件和未知属性会在写入任何内容之前被丢弃，更不用说发送了。代码和本文件保持同步，而且由于仓库是开源的，你可以自行验证。

本页只涉及匿名使用统计。你选择连接的 Trail 工作区是另一回事，会发送更多内容：参见[哪些内容保留在你的机器上](README.md#what-stays-on-your-machine)。

你运行的命令永远不会发送任何内容。事件会追加到本地文件，并由一个分离的后台进程每天最多发送一次，因此任何 `trail ask` 都不会等待网络。

唯一的例外是下面的 `install` 事件：npm postinstall 钩子会记录它，并立即交给同一个分离进程——否则，一个从未运行过命令的安装就永远不会触发发送，而“已安装但从未使用”正是我们需要看到的数字。`npm install` 本身仍然不会等待任何东西：子进程是分离的，其输出会被丢弃。

## 发送的内容

每个事件只携带以下公共属性：

| 属性 | 示例 | 说明 |
| --- | --- | --- |
| `app_version` | `0.12.0` | 产生该事件的 Trail-code 版本 |
| `os` | `darwin` / `win32` / `linux` | 平台，仅此而已 |
| `arch` | `arm64` / `x64` | CPU 架构 |
| `node_major` | `20` | 仅主版本号 |
| `ci` | `false` | 始终为 false——CI 从不发送（见下文） |
| `agent_host` | `claude-code` / `cursor` / `mcp` / `cli` | Trail-code 运行在哪个界面下 |
| `cli_name` | `trail` / `graft` | 启动进程的是两个名称中的哪一个（graft 是 trail 的旧名称） |
| `repo_id` | 一个随机 UUID | 参见“如何保持匿名” |

事件：

| 事件 | 额外属性 | 触发时机 |
| --- | --- | --- |
| `install` | `global`（`npm i -g` 为 `true`，项目依赖为 `false`） | npm postinstall 钩子运行——每台机器每个版本一次 |
| `first_run` | — | 一次，当 `trail` 或 `graft` 命令首次在机器上运行时 |
| `init_completed` | `agents`（你选择的 id，已排序）、`consent` | `trail init` 完成 |
| `build_completed` | `files_bucket`、`langs`、`mode`（`fast`/`deep`）、`duration_bucket`、`incremental` | 构建成功 |
| `build_failed` | `stage`、`code`——均为固定枚举 | 构建抛出异常 |
| `query` | `command`、`surface`（`cli`/`mcp`/`hook`）、`hit`（仅 `ask`）、`notes`（`0`/`1`/`2`，`ask` 展示的学习条目数；仅在保留学习条目的仓库中） | 任何查询命令 |
| `brain_signup_opened` | `mode`（`terminal`/`agent`/`nudge`），对于 nudge 还有 `command`（`ask`/`note`/`check`）和 `wording`（`share`/`learned`） | trail 命令把登录链接交给 agent 转发（每个会话最多一次） |
| `brain_signup_settled` | `outcome`（`linked`/`timed_out`/`bad_callback`/`stopped`，对于 agent 运行的推送还有 `agent_link_opened`/`still_waiting`/`expired`/`unsupported`）、`mode`（`terminal`/`agent`/`nudge`）、`duration_bucket` | 该交接以某种方式结束——对于 agent 运行的推送，是其每次运行；对于 nudge，是每次后台等待登录（`linked`/`timed_out`/`expired`/`unsupported`） |
| `note_saved` | `has_cost`（`true`/`false`）、`touches_bucket`（该学习条目涉及多少个文件，分桶）、`scope`（`repo`/`personal`） | `trail note` 保存一条学习条目，保存在仓库的 `.trail/` 或本机上。标题、文本和路径永远不会发送 |
| `learning_reused` | `teammate`（`true`/`false`）、`saved_tokens_bucket` | `trail ask` 展示来自早期会话的学习条目。具体是哪些及其文本永远不会发送 |
| `repo_seen` | `trail_committed`（`true`/`false`）、`learnings_bucket`、`workspace`（`true`/`false`） | 每个仓库每天一次，当 Claude Code 会话在 trail 运行的仓库中启动时。仓库名称和路径永远不会发送；仅通过每个事件携带的随机 `repo_id` 标识 |
| `learnings_committed` | `added_bucket`、`total_bucket`（`HEAD` 上已提交的学习条目，分桶）、`first_look`（`true`/`false`） | Claude Code 会话在 trail 运行的仓库中启动，且该仓库提交的学习条目比上次会话更多。其名称和文本永远不会发送 |
| `cold_start` | `source`（`sessions`/`reviews`）、`outcome`（`found`/`none_found`/`no_sessions`/`no_reviews`/`no_gh`/`no_claude`/`failed`/`cancelled`）、`sessions_bucket`（会话）或 `reviews_bucket`（带有审查者评论的已合并 PR；reviews）、`offered_bucket`、`picked_bucket`（保存了多少条：即用户保留的条目）、`duration_bucket`、`model` | `trail init` 读取你在该仓库中过去的 Claude Code 会话，一次，找出你多次告诉 Claude 的内容；并读取该仓库最近 50 个已合并 PR 的审查评论，一次，找出审查者在多个 PR 上标记的内容，并将其保存为学习条目。评论使用你自己的 `gh` 获取（未安装或未登录时为 `no_gh`），读取由你自己的 Claude Code 完成；你的消息、评论及其发现的内容永远不会发送给 trail |
| `takeaway_saved` | `new_skill`（`true`/`false`） | `trail learn` 在本机（`~/.trail/`）向技能添加一条纠正。其文本永远不会发送 |
| `skills_folded` | `takeaways_bucket`（合并了多少条纠正，分桶） | `trail skills fold` 将等待中的纠正写入技能的 SKILL.md |
| `session_summary` | `graft_reads_bucket`、`source_reads_bucket`、`saved_tokens_bucket`、`graft_turns_bucket`、`reported_turns_bucket` | 一次，在 agent 会话结束后 |

上述每个值都受两条规则约束，且两条规则都在代码中强制执行，而非依赖审查：

- **每个数字都是桶。** 构建报告 `"200-999"` 个文件，绝不报告 `417`。与语言集合相邻的精确计数会开始对特定仓库进行指纹识别；桶则不会。
- **每个字符串都是固定集合的成员。** `command` 是八个已知子命令之一，`code` 是十一个已知失败代码之一。集合之外的值会被丢弃，而不是发送——这正是阻止路径、符号名称或错误消息片段搭上“字符串属性”便车的原因。

一个完整的事件示例：

```json
{
  "event": "build_completed",
  "timestamp": "2026-08-21T09:14:22.417Z",
  "properties": {
    "app_version": "0.12.0", "os": "darwin", "arch": "arm64",
    "node_major": "20", "ci": "false", "agent_host": "claude-code", "cli_name": "trail",
    "repo_id": "6f2c1e90-...", "distinct_id": "b1f3a9c2-...",
    "files_bucket": "200-999", "langs": "go,ts", "mode": "deep",
    "duration_bucket": "30s-2m", "incremental": "true",
    "$process_person_profile": false
  }
}
```

运行 `trail telemetry debug` 可打印你的机器将发送的确切批次。它不会发送任何内容。

### Trail-code 在本地读取的一项内容

其中两个属性——`graft_turns_bucket` 和 `reported_turns_bucket`——需要知道其他属性不需要知道的事情：你实际阅读的回复是否提到了 Trail-code 保存的内容。Trail-code 在每次检索调用时都会计算节省量，但一个静默节省 20,000 token 的回合和一个什么都没节省的回合在 `saved_tokens_bucket` 中看起来完全相同，而这个差异正是整个问题的关键。

agent 自己的文本存在于 Hook 可以访问的一个地方：你的编辑器写入的转录文件，其名称在 Stop 钩子的 stdin 中给出。因此，在使用 Trail-code 的回合结束时，它会读取该文件的尾部，检查回复中是否有“saved ~N tokens”一行，并递增两个计数器之一。文件中的任何内容都不会被存储，也不会被发送——不是回复，不是其片段，也不是其长度。两个回合计数会以桶的形式通过网络传输，每个会话一次。

Trail-code 无法检查的回合——Stop 钩子未指定转录文件的编辑器、无法读取的文件——不会计入任一总数，因此比率始终意味着“在我们能读取的回合中”。

## 永远不会发送的内容

不发送源代码。不发送文件路径、仓库名称、组织名称、git 远程地址或分支名称。不发送符号名称、节点摘要或图中的任何内容。不发送查询字符串、提示词或 agent 输出——Trail-code 会在本地读取使用它的回合的最后一条回复，以判断是否提到了节省量（参见“Trail-code 在本地读取的一项内容”）；只发送结果计数，绝不发送文本。不发送错误消息或堆栈跟踪——失败只贡献固定列表中的一个代码，仅此而已。不发送环境变量、API 密钥、模型名称、主机名、用户名或电子邮件地址。

## 如何保持匿名

- 事件发送到 [PostHog](https://posthog.com)，并带有 `$process_person_profile: false`，这使它们成为**匿名事件**：不会创建个人资料，也不会存储身份。
- 唯一的标识符是两个**随机 UUID**。`install_id` 在首次运行时于 `~/.graft/telemetry.json` 中生成；`repo_id` 在每个检出目录的 `graft/.cache/` 中生成。两者都不是从你的机器、账户或 git 远程地址派生而来——它们是抛硬币的结果，被记录下来。远程 URL 的哈希至少是我们能猜测并确认的东西；随机 UUID 则不是。删除 `~/.graft/` 或 `graft/`，你看起来就像一个新用户。
- 基于 IP 的地理定位仅用于推导国家/地区以进行聚合统计；PostHog 不会在事件上保留 IP。

## 选择退出

以下任意一项都会完全禁用遥测：

1. **在 `trail init` 选择器中取消勾选“匿名使用统计”。** 立即生效，并在此机器上被记住。
2. **随时运行 `trail telemetry disable`。** `trail telemetry status` 显示当前状态，`trail telemetry enable` 将其重新打开。
3. **将 [`DO_NOT_TRACK`](https://web.archive.org/web/20251005054121/https://consoledonottrack.com/)** 设置为除 `0` 以外的任何值。无条件遵守——其优先级高于 Trail-code 自身的设置。
4. **在 CI 中运行。** `CI`、`GITHUB_ACTIONS`、`GITLAB_CI` 等会未经询问地将其关闭。构建服务器不是用户。
5. **从源码构建。** PostHog 密钥仅在发布时打入（`scripts/stamp-telemetry-key.mjs`）；克隆、Fork 或本地 `npm run build` 会使用空密钥编译，遥测模块处于惰性状态。Fork 永远不会向任何地方发送事件。

## 事件发送到哪里

官方构建发送到 `https://events.nanonets.com`，这是 Nanonets 自己的 PostHog 入口，因此 Trail-code 的数字与产品其余部分位于同一个项目中，而不是单独的项目。客户端向 `/batch/` 发送单个批次。

主机是发布时设置（`GRAFT_POSTHOG_HOST`），且 PostHog 本身是开源且可自托管的，因此端点可以在无需更改任何代码的情况下迁移。