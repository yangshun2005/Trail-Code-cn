# Trail-code GitHub App

在每个 Pull Request 上发布爆炸半径评论，并在签名链接背后托管交互式图谱。

它之所以存在，是因为有一个工作流 YAML 无法绕过的限制：**Fork 上的 `pull_request` 任务拿到的是只读令牌**，因此无法发表评论；而 `pull_request_target` 若不选择运行 Fork 代码，就无法检出它。App 的安装令牌属于*基础*仓库，所以 Fork PR 是普通工作。由此自然得到两点好处：页面由 App 提供（私有仓库无需公开的 `gh-pages`），安装只需一键，而非每个仓库一个工作流文件。

## 每个 Pull Request 做什么

1. 验证 webhook 签名，将任务入队，返回 `202` —— GitHub 在十秒后放弃投递，而一次审查耗时更长。
2. 浅克隆获取 `refs/pull/<n>/merge` 和基础分支；对于已关闭或已合并的 Pull Request（其合并引用已被 GitHub 删除），回退到 `refs/pull/<n>/head`。然后将 head 引用与合并时的基础分支做 diff，因此半径仍是该 Pull Request 自身的变更。
3. 构建结构图谱，计算半径，渲染评论。
4. 存储查看器页面，并用签名 URL 链接它。
5. 编辑其已有评论，而非每次推送新增一条。

工作按 Pull Request 被取代：一分钟内五次推送只产生一次审查，而非五次，因为前四条评论反正会被覆盖。

## 设置

### 1. 注册 App

<https://github.com/settings/apps/new>（或你组织的设置 → Developer settings → GitHub Apps → New）。

| 字段 | 值 |
| --- | --- |
| Webhook URL | `https://<your-host>/webhook` |
| Webhook secret | 一个长随机字符串 —— 保存好，它就是 `GRAFT_WEBHOOK_SECRET` |
| 仓库权限 | **Contents: Read-only**、**Pull requests: Read & write** |
| 订阅事件 | **Pull request** |
| 可安装位置 | 由你选择 |

无需其他。Contents-read 用于克隆代码；pull-requests-write 用于发布评论。它从不需要 Actions、Checks、Administration 或代码写权限。

然后**生成私钥** —— 下载的文件是唯一副本 —— 并记下 **App ID**。

### 2. 运行它

```bash
docker build -t graft-app .
docker run -p 3000:3000 \
  -e GRAFT_APP_ID=123456 \
  -e GRAFT_APP_PRIVATE_KEY="$(cat graft.private-key.pem)" \
  -e GRAFT_WEBHOOK_SECRET=... \
  -e GRAFT_PUBLIC_URL=https://graft.example.com \
  -v graft-pages:/var/lib/graft/pages \
  graft-app
```

镜像运行时只需要 `git`，以非 root 运行，并通过 `/healthz` 返回队列深度。任何容器主机都行 —— Fly、Cloud Run、ECS、VM。`GRAFT_PUBLIC_URL` 必须是 GitHub 和你的审查者能访问的源，因为评论中的链接由它构建。

四个变量缺任何一个，进程都会拒绝启动：没有 webhook secret 就启动的服务器看似健康，却会静默拒绝每次投递。

卷是第五项，且是可选的 —— 但没有它，已发布到 Pull Request 的每个链接都会在容器首次被替换时失效。链接中的令牌由页面 id 派生而非存储，因此链接仍能验证，只是返回 404，评论中没有任何说明。镜像已将 `GRAFT_PAGE_DIR` 指向 `/var/lib/graft/pages`（500 页 × 约 50 kB，上限约 25 MB）；在那里挂载任何持久存储，页面就会随进程恢复。在非此镜像的主机上自行设置该变量，或留空以将存储保留在内存中 —— 它故意不是必需的，因为拒绝启动的服务器什么也审查不了。

### 3. 在仓库上安装

App 设置 → Install App → 选择仓库。**安装需要仓库管理员权限**（组织范围安装需组织所有者）—— 这是 App 唯一无法绕过的限制。

## 安全

App 在持有基础仓库令牌的同时，会在每个 Fork PR 上克隆陌生人编写的代码，因此：

- **仓库中的任何内容都不会被执行** —— 没有 `npm install`，没有构建步骤，没有 postinstall。图谱来自 tree-sitter 读取源文本。
- **Git 也被指示不运行任何东西**：`core.hooksPath=/dev/null`、`GIT_TERMINAL_PROMPT=0`、`GIT_CONFIG_NOSYSTEM=1`，不递归子模块。
- **令牌从不落入检出目录。** 它按调用作为认证头传递，而非写入远程 URL（那会被 `.git/config` 和 reflog 保留）。在记录任何日志之前，它会从错误文本中被脱敏。
- **页面是能力，不是公开 URL。** `/p/<id>?t=<hmac>` —— 未知页面和错误令牌都返回 `404`，因此该端点无法用于发现哪些 Pull Request 存在。链接随其指向的页面过期。

## 尚未构建的部分

- **命名。** 区域回退到其枢纽符号。`--name` 的一次缓存 LLM 调用尚未接入，且将私有仓库源码发送给模型应作为每次安装的显式选择加入，而非默认。
- **持久化。** 页面存于内存，因此部署会丢失它们；下次推送到 PR 会重建其页面。当有多个实例时，共享存储是解决方案。
- **评论折叠列表中的证据引用**将在 #180 落地时到来（`markdownReport(report, { root })` —— 增量式，此处一行）。

## 部署到 AWS App Runner

`deploy/apprunner.sh` 就是全部部署：为 x86_64 构建，推送到 ECR，创建或更新服务。每次新构建都重新运行它。

### 首次部署前的一次性操作

```bash
aws sso login                      # 或此账户的其他认证方式

# 1. 密钥。私钥是多行 PEM，原样保留。
aws secretsmanager create-secret --name graft/app-id         --secret-string 123456
aws secretsmanager create-secret --name graft/webhook-secret --secret-string "$(openssl rand -hex 32)"
aws secretsmanager create-secret --name graft/private-key    --secret-string file://graft.private-key.pem

# 2. App Runner 用于从 ECR 拉取镜像的角色。
aws iam create-role --role-name AppRunnerECRAccessRole --path /service-role/ \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"build.apprunner.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
aws iam attach-role-policy --role-name AppRunnerECRAccessRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSAppRunnerServicePolicyForECRAccess

# 3. 运行中的容器用于读取这些密钥的角色。
aws iam create-role --role-name graft-app-instance \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"tasks.apprunner.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
aws iam put-role-policy --role-name graft-app-instance --policy-name read-graft-secrets \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"secretsmanager:GetSecretValue","Resource":"arn:aws:secretsmanager:*:*:secret:graft/*"}]}'
```

两个角色，因为 App Runner 将它们分开：一个由*构建*侧假定以拉取镜像，另一个由*运行中*的任务假定以读取配置。给第二个仅 `graft/*` 权限，使此服务远离账户中的其他所有密钥。

### 每次部署

```bash
GRAFT_PUBLIC_URL=https://graft.nanonets.ai ./deploy/apprunner.sh
```

然后一次性：`associate-custom-domain`，将其打印的 CNAME 添加到 Route 53（一个验证证书，一个将域名指向服务），并将 App 的 webhook URL 设为 `https://graft.nanonets.ai/webhook`。

### 配置刻意为之的两点

- **`--platform linux/amd64`。** App Runner 仅支持 x86_64；在 Apple Silicon Mac 上构建的镜像推送正常，但运行时因 exec 格式错误失败，看起来像入口点损坏。
- **单实例，固定。** 查看器页面存于进程内，因此第二个实例会对第一个实例生成的链接返回 404。提高 `--max-size` 意味着先将 `PageStore` 迁移到 S3 或数据库 —— 上限是正确性，不是成本控制。