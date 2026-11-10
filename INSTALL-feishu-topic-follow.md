# 安装说明：飞书话题补丁（给 agent 执行）

> 给 agent 的话：你被要求阅读这份文件并在**本机**安装这个补丁。按顺序执行，每一步检查结果；任何一步失败就停下来，把命令和输出原样报告给用户，不要自己改代码绕过去。
>
> 本文件地址：`https://raw.githubusercontent.com/neow021/hermes-agent/neo/notes/INSTALL-feishu-topic-follow.md`
> 注意：`neow021/hermes-agent` 的 `main` 分支根目录也有 `AGENTS.md`/`README.md`，那是上游写给 Hermes 开发者的，与本补丁无关。

## 补丁做什么

两个开关，放在 `platforms.feishu.extra` 下：

- `thread_reply`：群里 @ 机器人后，机器人新开一个话题回复（先发一句"收到，处理中…"把话题开出来）。
- `thread_follow`：在机器人开的话题里，后续消息不用再 @ 它；话题里 @ 了别人的消息它不回。

上游 PR：NousResearch/hermes-agent#151706。补丁文件：
`https://raw.githubusercontent.com/neow021/hermes-agent/neo/notes/feishu-topic-follow.patch`

## 规则

- **不要读取、打印或发送 `FEISHU_APP_SECRET`**，也不要 `source` 整个 `.env`。本流程只需要 `FEISHU_APP_ID` 和 `FEISHU_DOMAIN`，用 `grep '^FEISHU_APP_ID='` 这样只取一行。
- 改配置用 `hermes config set`，不要手动编辑 `config.yaml`。
- 不要 commit、push、开 PR，不要运行 `hermes update`。
- 补丁打在 Hermes 安装目录上，是**未提交改动**；不要 `git stash`/`git checkout` 掉已有的其他本地改动。

## 第 1 步：找到飞书 profile，**先把权限申请链接发给用户**

这一步要用户在飞书后台操作（可能还要管理员审批），最慢，所以放在最前面，发出去后继续做后面的步骤。

```bash
grep -l '^FEISHU_APP_ID=' ~/.hermes/.env ~/.hermes/profiles/*/.env 2>/dev/null
```

- 命中 `~/.hermes/.env` → 默认 profile，后面的 `hermes` 命令不加 `-p`。
- 命中 `~/.hermes/profiles/<名字>/.env` → 后面的命令都写成 `hermes -p <名字> ...`。
- 命中多个 → 问用户是哪一个。

取应用 ID 和域名（下面用 `<ENV>` 表示上面命中的文件）：

```bash
grep -E '^FEISHU_(APP_ID|DOMAIN)=' <ENV>
```

拼出权限申请链接（`FEISHU_DOMAIN` 为 `lark` 时把域名换成 `open.larksuite.com`，否则用 `open.feishu.cn`）：

```
https://open.feishu.cn/app/<FEISHU_APP_ID>/auth?q=im:message.group_msg&op_from=openapi&token_type=tenant
```

**立刻**把下面这段发给用户（醒目标出，这是需要用户本人操作的事项）：

> ⚠️ **需要你操作**：点开下面的链接，为机器人申请「获取群组中所有消息」（`im:message.group_msg`）权限：
> <链接>
> 申请后还要在开发者后台 **创建版本并发布**；这是敏感权限，可能需要企业管理员审批。
> 不开这个权限，"话题里不用 @"不会生效（飞书只会把 @ 机器人的消息发过来）。完成后回复我"已开通"。

如果用户说之前已经开过，也照样发链接让他点开确认，页面会显示已开通。

## 第 2 步：确认要不要打补丁

```bash
HERMES_DIR=~/.hermes/hermes-agent   # 若不存在，问用户 Hermes 装在哪
git -C "$HERMES_DIR" rev-parse --is-inside-work-tree
ls "$HERMES_DIR/plugins/platforms/feishu/feishu_topic_follow.py" 2>/dev/null && echo ALREADY_PRESENT
curl -s https://api.github.com/repos/NousResearch/hermes-agent/pulls/151706 | grep -E '"(state|merged)"'
```

- `ALREADY_PRESENT`：代码已经在了（补丁打过，或上游已合并且本机已升级）→ 跳到第 4 步。
- PR 已合并（`"merged": true`）但文件不在 → 不要打补丁，告诉用户先运行 `hermes update`，之后跳到第 4 步。
- 其他情况 → 第 3 步。

## 第 3 步：打补丁

```bash
cd "$HERMES_DIR"
curl -fsSL -o /tmp/feishu-topic-follow.patch \
  https://raw.githubusercontent.com/neow021/hermes-agent/neo/notes/feishu-topic-follow.patch
git apply --check /tmp/feishu-topic-follow.patch && git apply /tmp/feishu-topic-follow.patch
```

`git apply --check` 报 `patch does not apply` 时（本机 Hermes 比补丁基线新，上下文行变了），改用三方合并再试一次，然后把它暂存的改动退回为未暂存：

```bash
git apply -3 /tmp/feishu-topic-follow.patch
git reset -q -- locales plugins/platforms/feishu tests/gateway/test_feishu_topic_follow.py website/docs/user-guide/messaging/feishu.md
```

- `git apply -3` 输出里出现 `conflict`/`Unmerged`，或 `git diff --name-only --diff-filter=U` 有输出 → 停下，报告输出，等补丁维护者（neow021）更新补丁。不要手动解决冲突。

检查：

```bash
git status --short plugins/platforms/feishu/ tests/gateway/test_feishu_topic_follow.py
```

应看到 `?? plugins/platforms/feishu/feishu_topic_follow.py`、`?? tests/gateway/test_feishu_topic_follow.py` 和 ` M plugins/platforms/feishu/adapter.py`（另有 17 个 `locales/*.yaml` 和一份文档的改动）。

## 第 4 步：打开开关

```bash
hermes [-p <profile>] config set platforms.feishu.extra.thread_reply true
hermes [-p <profile>] config set platforms.feishu.extra.thread_follow true
hermes [-p <profile>] config get platforms.feishu.extra
```

可能出现"not a recognized config key — it was saved anyway"的提示，属正常（未打补丁的上游版本不认识这两个键），值已保存。

**不要**为了这两个开关新建 `group_rules` 条目：没写 `policy` 的条目默认是 `open`，会让原来的用户白名单失效。已有的 `group_rules` 条目里可以加这两个键做单群覆盖。

## 第 5 步：重启 gateway

- 如果你本身就跑在这个 gateway 里（例如用户在飞书里和你对话），不要自己重启——请用户在飞书里发 `/restart`。
- 否则：`hermes [-p <profile>] gateway restart`。

## 第 6 步：实测（等用户回复"已开通"之后）

请用户在一个机器人所在的群里依次做：

1. @ 机器人说一句 → 应新开一个话题，先出现"收到，处理中…"，再出现正式回复。
2. 在这个话题里**不 @** 说一句 → 机器人接着回。
3. 在这个话题里 @ 一位同事说一句 → 机器人不回。

第 2 步不回时，查日志确认消息是否到达：

```bash
grep '\[Feishu\] Received raw message' ~/.hermes[/profiles/<profile>]/logs/gateway.log | tail -5
```

- 用户发消息的时间点没有对应日志 → 飞书没把消息推过来，权限没生效：请用户确认已开通 `im:message.group_msg` **并发布了新版本**，然后再试。
- 有日志但不回 → 把相关日志行报告给用户。

## 第 7 步：向用户报告

列出：用的 profile 和 Hermes 目录、补丁是否打上（或为何跳过）、两个开关的值、权限状态、三项实测结果。并提醒：

- 补丁是未提交改动。以后 `hermes update` 后要检查 `git -C ~/.hermes/hermes-agent status --short plugins/platforms/feishu/` 是否还能看到 `feishu_topic_follow.py`；看不到就重新按本文件第 2、3 步打补丁。
- 上游 PR #151706 合并后，先撤掉本地补丁（只撤补丁本身，不动其他本地改动）：`git -C ~/.hermes/hermes-agent apply -R <补丁文件>`，再 `hermes update`，开关保持不变。
