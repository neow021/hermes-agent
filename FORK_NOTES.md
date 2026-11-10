# neow021/hermes-agent fork 说明

这个 fork 用来开发、维护本机 Hermes 的本地改动。本分支（`neo/notes`）只放说明，不含代码，也不进任何 PR。

## 当前的本地改动

| 改动 | 分支 | 上游 PR | 状态 |
|---|---|---|---|
| 飞书话题：@ 后自动开话题回复（`thread_reply`）；话题内不用再 @（`thread_follow`） | `feat/feishu-topic-follow` | NousResearch/hermes-agent#151706 | 未合并，本机已在用 |

## 在别的机器上安装

安装步骤是私有 repo `neow021/.agents` 里的 skill `hermes-feishu-setup`（会先把权限申请链接发给你）。让那台机器上的 agent（需已登录 `gh`）读取并照做：

`gh api repos/neow021/.agents/contents/skills/hermes-feishu-setup/SKILL.md -H "Accept: application/vnd.github.raw"`

补丁文件放在本分支：`feishu-topic-follow.patch`，skill 从这里下载。改安装步骤时改那个 skill，不改这里。

## 本机的位置

| 用途 | 路径 |
|---|---|
| fork 开发目录 | `~/dev/hermes-agent`（`origin` = 本 fork，`upstream` = NousResearch） |
| 本说明所在的 worktree | `~/dev/hermes-agent-notes`（`neo/notes` 分支） |
| 补丁备份 | `~/dev/hermes-patches/feishu-topic-follow.patch` |
| 正在运行的 Hermes | `~/.hermes/hermes-agent`，补丁以**未提交改动**的形式打在上面 |

## 本机配置和权限

- feishu 配置（`~/.hermes/profiles/feishu/config.yaml`）中打开了：
  - `platforms.feishu.extra.thread_reply: true`
  - `platforms.feishu.extra.thread_follow: true`
  - `display.language: zh`，所以开话题时的提示语显示"收到，处理中…"
- 飞书应用（`cli_aa7a9e3b7af9dce0`）已开通 `im:message.group_msg`（获取群组中所有消息）权限。没有这个权限，"话题内不用再 @"就收不到消息。
- 改配置要用 `hermes config set`，不要手动编辑文件。
- 不要只用这两个开关去写 `group_rules` 条目：没写 `policy` 的条目默认是 `open`，会让原来的白名单失效。

## 升级 Hermes 的步骤

1. **先看 PR 有没有合并**：`gh pr view 151706 -R NousResearch/hermes-agent --json state`
2. **已合并**：升级前先撤掉本机的补丁，否则会和官方版本冲突。
   ```bash
   cd ~/.hermes/hermes-agent
   git checkout -- .
   rm plugins/platforms/feishu/feishu_topic_follow.py tests/gateway/test_feishu_topic_follow.py
   hermes update
   ```
   以后就直接用官方版本。删掉本表里这一行，以及对应的 fork 分支和补丁文件。
3. **未合并**：直接执行 `hermes update`。升级程序会先把本地改动收起来，升级完再放回去。升级后检查：
   ```bash
   git -C ~/.hermes/hermes-agent status --short plugins/platforms/feishu/
   ```
   应该能看到 `feishu_topic_follow.py`（新文件）和 `adapter.py`（有改动）。如果没看到，或者提示有冲突，就把未提交的改动全部撤掉（同上面第 2 步的 `git checkout` 和 `rm`），再重新打补丁：
   ```bash
   cd ~/.hermes/hermes-agent && git apply ~/dev/hermes-patches/feishu-topic-follow.patch
   ```
   打不上的话，按下面"同步 fork"的步骤更新补丁，再重新打。
4. **重启 gateway**：在飞书里发 `/restart`，或者在 gateway 之外的终端执行 `hermes gateway restart`。
5. **实测**：
   - 在群里 @ 机器人：会开一个新话题，先出现"收到，处理中…"，再出现正式回复。
   - 在那个话题里不 @ 说一句：机器人接着回。
   - 在话题里 @ 一位同事说一句：机器人不回。

## 同步 fork（上游有新代码时）

```bash
cd ~/dev/hermes-agent
git fetch upstream main
git checkout feat/feishu-topic-follow
git rebase upstream/main
HERMES_PYTHON=~/.hermes/hermes-agent/venv/bin/python scripts/run_tests.sh tests/gateway/test_feishu*.py tests/agent/test_i18n*.py
~/.hermes/hermes-agent/venv/bin/python scripts/check --commit HEAD
git push -f origin feat/feishu-topic-follow          # PR #151706 会自动更新
git format-patch -1 HEAD --stdout > ~/dev/hermes-patches/feishu-topic-follow.patch
# 同步到 neo/notes，别的机器按 INSTALL 文件下载的是这一份
cp ~/dev/hermes-patches/feishu-topic-follow.patch ~/dev/hermes-agent-notes/
git -C ~/dev/hermes-agent-notes commit -am "chore: refresh feishu-topic-follow patch" && git -C ~/dev/hermes-agent-notes push
```

上游的代码检查有一条规则：`adapter.py` 只能变小、不能变大。新逻辑放进 `feishu_topic_follow.py`。
