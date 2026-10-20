# 更新日志

## 0.23.0

### 变更

- **一个共享文件夹**（#619）：`trail init` 现在共享 `.trail/hook.cjs` 和 `.claude/settings.json` 中的一行 SessionStart。其余均为个人配置：`.claude/settings.local.json` 和 `~/.claude` 中的 skill。按旧方式配置的仓库会在下次会话时自动迁移；迁移后提交一次 `.trail/` 和 `.claude/settings.json`。
- **代码地图存放在 `.trail/cache/`**，位于 trail 仓库中，由 `.trail/.gitignore` 排除在 git 之外。已有的 `graft/` 地图会被移动过去，而非重建。
- **Claude Code 不再使用 MCP 服务器。** init 不再添加它，并移除旧版 init 添加的那个。Cursor 保留其服务器。
- **编辑后按函数显示重叠**（#617）：仅当你和队友都修改了同一函数时，才会提及队友的开放分支；已合并或已关闭的分支不再计入。
- **学习记录的成本从其工作起点开始计算**（#620），而非从长会话的起点开始。

### 修复

- 全新的 `trail init` 不再遗留未跟踪的 `graft/` 文件夹。

## 0.22.1

### 新增

- **未安装 trail 的队友仍能获取学习记录**（#611）：在已提交 `.trail/` 的仓库中，Claude 会读取最新的学习记录，记录其使用的那条，然后提供一次安装 trail 的选项。
- **`trail init` 允许你勾选要保留的学习记录**，并显示每条记录来自多少次会话。
- **`trail init` 从 PR 审查评论中学习**：审查者在最近 50 个已合并 PR 中标记过 2 次及以上的问题会成为团队学习记录。
- **`trail check` 会标记对队友学习记录的编辑。**

### 修复

- **`trail init` 能发现仅作为 VS Code 或 Cursor 扩展安装的 Claude Code**，不再无声跳过过往会话。再次运行 `trail init` 以获取遗漏的学习记录。

### 移除

- GitHub App 的 brain 构建（#613）。

## 0.22.0

graft 现更名为 **Trail-code**。同一构建以两个包发布：`@trailhq/trail-code`（`trail` 命令）和 `@nanonets/graft`（`graft` 命令，不变）。两者都会持续获得每个版本。

### 新增

- **`npm install -g @trailhq/trail-code`** 提供 `trail` 命令：代码地图加上团队在 `.trail/` 中共享的学习记录。
- **`graft upgrade` 在 graft 旁添加 `trail`**，graft 接入的仓库中下次 Claude Code 会话会自动迁移到 trail。
- **代理会传递消息**：在装有 graft 但无 trail 的机器上，Claude Code 每周会被提醒一次提及 `graft upgrade` 会添加 trail。

### 修复

- **每个 Claude Code Hook 只响应一次**，而非同时从仓库和 `~/.claude` 的副本响应，因此学习记录不再在每个提示中到达两次。
- **Hook shim 能找到 `@trailhq/trail-code`**，且 `trail` 会在旧版 graft Hook 无法访问它的仓库中发出警告。
- **npm 12 上不再崩溃**，该版本跳过语法的安装脚本：语法按语言加载并回退到 WASM。
- 包指向 `github.com/trailhq/Trail-Code`。

## 0.19.0

### 新增

- **从终端推送仓库的 brain**（#422）：回环交接将已完成的 brain 交给浏览器，而非半成品——推送等待构建完成，然后将浏览器定位到结果。
- **无需安装 App 即可读取公开仓库**（#349）：访问权限独立应答，无需先读取仓库。

### 修复

- **完成的推送落在 Trail 中**（#417），而非本地页面；注册跳转到 Trail 自己的前端，而非共享的 agents 主机。
- **仓库读取在子进程中运行**，因此应用在工作时仍能响应，且摘要在子进程断开前完成发送。
- **公开读取借用安装令牌**，而非回退到匿名速率限制。

## 0.18.0

### 新增

- **Trail Brain 集成**（#322）：graft 可以从仓库构建 *brain*，并将其规则带入每次 `ask`——双向链接，因此检索由 brain 积累的团队上下文塑造，而非仅由代码图塑造。
- **brain 在挖掘前验证检出**（#344）：graft 将工作副本与 brain 期望的仓库进行比对，确保规则不会从错误的树中挖掘。

### 修复

- **brain 从维护中刷新其规则**（#343），因此单次空拉取不再使其卡在无规则状态。

## 0.17.0

### 新增

- **Claude Code 会话现在报告 graft 节省的美元价值**（#282），而不仅是 token 数量——状态栏上的累计总额已计价，因此通过图路由查询而非整文件读取的收益在终端中可见。
- **`graft init` 也将 Claude Code 接线写入 `~`**（#276），因此从同一 home 创建的工作树和新 shell 保持 graft 可用，而非在离开仓库根目录时丢失 Hook 和状态栏。

### 修复

- **PR 审查 GitHub App** 将审查页面保留在磁盘上（#278），使已发布的链接在重启后仍有效；通过 head ref 与合并基线的差异审查**已合并和已关闭的 PR**（#279、#280），使已合并的 PR 保持可用的图链接；并**在各自进程中运行每次审查**（#281），使一次慢审查不再阻塞服务器。
- **绿色 main**（#277）：遥测测试在断言前清除所有 CI 环境变量，CodeQL action 固定版本重新对齐。

## 0.16.0

### 新增

- **`graft init --no-statusline`**（及 `GRAFT_NO_STATUSLINE=1`）跳过写入 Claude Code 的 `statusLine` / `subagentStatusLine`。自定义栏——在项目的 `.claude/settings.json` 或 `~/.claude/settings.json` 中——保持在前：项目级字段否则会隐藏用户级字段。该选择记录在接线标记中，因此后续会话刷新无法恢复 Graft 的栏。Graft 仍识别自己的辅助程序（`graft-statusline.cjs`），并在重新 init 时更新该命令。

## 0.15.0

### 新增

- **Swift 获得全保真（深度层）提取**，以与 Kotlin 相同的方式从广度层提升（#130）——其 swift tags 查询完全没有调用捕获，因此 Swift 仓库索引的符号零接线。`tree-sitter-swift`（原生，`^0.7.1`——首个安装时编译随附解析器而非重新生成的版本，后者在 npm 提升下会出错；根 `overrides` 条目为其开发安装固定 `tree-sitter` peer）解析 `.swift`。一个 `class_declaration` 节点覆盖 `class` / `struct` / `enum` / `actor` / `extension`，通过各自关键字区分：class 和 actor → `class`，struct → `struct`，enum → `enum`，`protocol` → `interface`，`typealias` → `type`，顶层 `let`/`var` → `variable`。`extension Point` 节点采用被扩展类型自身的名称，因此其成员生成为 Point 方法，Point 接收者上的成员调用解析到它们。扩展节点本身为 `module` 类型，而非第二个同名 class——否则每个 `Point()` 调用和 `: Point` 继承目标都会变得模糊并被丢弃。`init` 成为以其类型命名的方法（类似 Java 构造函数）；`func` 在任何类型内是方法，在其他地方是函数。调用通过 `call_expression` 解析，接收者为 `navigation_expression` / `self` / `super`。类型体内的裸小写调用是**一条**边，按 Swift 自身的内层作用域优先顺序携带两种解读：先成员解读（所有者限定索引 + 仓库内祖先链——`extension Set` 内的 stdlib 调用如 `contains` 会被丢弃，而非绑定到无关类型的唯一同名方法，这是在 swift-composable-architecture 上 dogfooding 发现的误报），仅当链上无成员时才采用自由函数解读——因此同时定义为两者的名称仅产生成员边，正如 Swift 的分派。`super.method()` 针对声明自身的超类解析（第一个 `:` 条目，Swift 语法将其置于任何协议之前），爬过当前类的重写。重载通过声明参数数量与调用点参数数量区分（Java 的精确机制——默认值和可变参数使参数数量成为最小值）；参数数量无法区分的重载集（`save(Int)` vs `save(String)`）**丢弃**，而非采用同文件决胜，后者会将先出现的重载标记为 `extracted`。初始化器调用（`Animal(legs: 4)`——普通调用节点，无 `new`）在函数找不到时回退到 class/struct/enum 目标，Python 的构造函数回退形态。`:` 继承子句产生 `extends` 边（裸名称——Swift 语法上无法说明哪个说明符是超类），`import` 声明产生模块路径导入边，可见性映射到 `exported`，仅 `private`/`fileprivate` 隐藏——Swift 默认的 `internal` 是模块级，对于单模块仓库即 API 表面。Swift 绑定收集器从置信的语法局部线索推断接收者类型：类型化参数（`func feed(animal: Animal)`，处理参数标签）、类型化属性、初始化器调用赋值（`let vet = Vet()`——UpperCamelCase 被调用者，与 Go 的 `NewX` 相同的约定信任），以及裸绑定和 `self.` 前缀绑定的字段——因此 `vet.check()`、`keeper.wave()`、`self.repo.save()` 和类型成员调用（`Animal.census()`）都通过所有者限定方法索引解析。无局部线索的接收者（链式调用的结果）保持未解析而非猜测。
### 修复

- **`allowScripts` 现在按 npm 实际匹配的身份命名 R。** 条目为 `tree-sitter-r@1.3.0`，即 `dependencies` 中的别名，但 npm 从 lockfile 中解析的包派生身份——`@davisvaughan/tree-sitter-r@1.3.0`。旧键不匹配任何内容，因此语法的安装脚本被视为未审查并被阻止；`npm ci --strict-allow-scripts` 因此失败。实践中无害仅因该包为每个支持的平台提供预构建。注意 `overrides` 键必须保持别名——两个字段的键方式不同。
- **`graft check` 不再将每个容器层节点报告为 `removed`。** `checkGraph` 在深度层和广度层上分支，但从未在构建用于 `.vue` 的容器层上分支，因此 `genericLangOf` 返回 null，`generic!` 断言抛出，catch 将其吞为解析失败——每个 `.vue` 节点在干净构建上落入 `removed`，而 check 建议运行的 `graft build` 已写入它们。check 现在镜像构建的三路分支，与通用语法一起预热容器语法，并将"无层声明此文件"视为显式情况而非非空断言，因此下一个添加的层会在类型检查器中失败，而非静默报告漂移。（[#236](https://github.com/trailhq/Graft/issues/236)）

### 新增

- **遥测现在能区分"graft 节省了 token"和"用户被告知"。** `saved_tokens_bucket` 统计 graft *计算*的内容——PostToolUse 累加器扫过的每个 `[graft] tokens saved ≈ N` 页脚——因此静默节省 20,000 token 的轮次和未节省的轮次是同一数字。`session_summary` 现在还携带 `graft_turns_bucket` 和 `reported_turns_bucket`：在使用 graft 的轮次中，有多少以 `SKILL.md` 和 `SAVINGS_TURN_NUDGE` 都要求的一行统计结束。

  代理自身的文本位于 Hook 可到达的一处，因此在使用 graft 的轮次结束时，Stop Hook 读取其 stdin 上命名的主机转录尾部，并检查回复中是否有"graft saved ~N tokens"行。仅保留轮次计数，且仅作为桶——无回复、无片段，甚至无长度。`TELEMETRY.md` 完整记录了本地读取。

  无法检查的轮次——Stop Hook 未命名转录的主机、不可读文件、Stop 与转录写入竞争——在*两个*总数中都不计入，因此比率意味着"在我们能读取的轮次中"，而非在每个编辑器上除 Claude Code 外都降为零。一次回复是一轮，无论 Stop 触发多少次，由最后检查的回复 id 保护。

- **`scripts/tally-audit.mjs`**，离线且全分辨率的相同比率：哪些轮次静默、代理报告的数字是否匹配 graft 自身的页脚、是否命名了调用次数。当聚合结果令人惊讶时运行它。它还会标记大到足以是伪影而非节省的每轮节省估计。


- **`graft init` 收敛而非累积，`graft uninstall` 完全移除 graft。** `init` 写入所选代理需要的文件，从不查看其余，因此由旧版本——或同一版本不同 `--agents`——接线的仓库永远保留该次运行的文件，而会话启动刷新随后保持它们*最新*。`init` 现在撤回它不打算写入的每个代理，`graft uninstall` 撤回全部。

  仅触及 graft 自身的贡献：共享文件内仅标记围栏块，配置内仅 `graft` 键——外部 MCP 服务器、Hook、状态栏和忽略条目逐字节保留。空文件被删除而非截断为空壳，清空的目录被修剪。无法解析的配置被报告并保持原样。`uninstall` 在 `-y` 前为试运行。

  目标列表从 `init` 写入所用的同一注册表派生，因此后续添加的主机可免费撤回；仅从注册表*移除*的主机需要手写条目，`LEGACY_TARGETS` 说明了这一点。排除按路径和主机 id 进行：三个主机写入 `AGENTS.md`，保留其中任何一个都必须保留该块。

### 修复

- **过时的 `[mcp_servers.graft]` 现在被替换而非跳过。** TOML 写入器在头部存在时立即返回，将启动命令冻结在首次 `init` 写入的任何内容——在 graft 不在 `PATH` 时接线的仓库永远保留慢速 `npx` 形式，任何升级都无法纠正。Codex 和 Grok 都经过该路径。外部表在两种情况下均不受影响。

- **`.claude/settings.json` 不再累积 graft 自身的条目。** 允许列表和 `footerLinksRegexes` 仅追加，因此重命名的调用形式与其替代品并存于用户设置中，无法移除。两者现在在添加当前集合前丢弃 graft 的先前条目，与 Hook 合并已有的做法相同。范围限于 graft 实际被调用的形式，因此手写的 `Bash(graft-mytool:*)` 得以保留。

- **`graft blast` 建议标记谁。** 注释已命名 diff 更改的区域及其可能影响的区域；现在还会命名其背后的人，从 git 历史读取，无 API 调用、无配置文件。每个区域在其自身文件上运行一次 `git log`，偏向近期工作（120 天半衰期），diff 仅*到达*的区域按 0.6 计，更改区域按 1.0 计——你的更改可能破坏其代码的人正是仅凭 diff 永远不会浮现的审查者。Markdown 报告在测试行下增加 `Tag:` 行和折叠的 `Who knows this code` 表；导出页面在每个气泡上增加首字母徽章、详情面板中的 *Who knows this* 块，以及 `people` 图例切换。

  句柄从不猜测：仅 GitHub noreply 提交地址解析为 `@mention`，其他人打印为普通无链接名称，因为猜测的提及会 ping 陌生人。仓库可通过 `.mailmap` 条目永久修复，git 会将其应用于 `blast` 读取的名称。合并提交、机器人、diff 范围内所有提交作者，以及——无 `--base` 的本地运行——你自己的 git 身份均被排除。`--no-owners` 关闭该层；`--pr-author <who...>` 接受登录名、名称或邮箱；捆绑 action 增加 `suggest-reviewers` 输入，默认 true。

## 0.13.0

### 新增

- **Grok（xAI）是一等 `graft init` 主机。** 检测 `~/.grok` 或仓库 `.grok/`，写入 `.grok/skills/graft/SKILL.md` 以及 `.grok/config.toml`（Grok 的 MCP 配置）中的仓库级 `[mcp_servers.graft]` 块。用 `graft init --agents grok` 选择。
- **R 语言支持。** `tree-sitter-r`（`npm:@davisvaughan/tree-sitter-r`；无作用域的 npm 名称是被抢注的占位符）解析 `.R`/`.r` 文件。普通函数：每个 `name <- function(...)` / `name = function(...)` / `function(...) -> name` 赋值成为 `function` 节点（语法的 `function_definition` 无名称字段，因此名称来自外层赋值；右赋值有其自己的 AST 形态）。类——R 的类系统是库约定而非语法，因此通过调用惯用法识别：**R6**（`R6::R6Class(...)`——类节点，`public =`/`private =`/`active =` 条目作为方法，`inherit =` 继承，`self$`/`