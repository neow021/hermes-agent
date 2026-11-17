# neow021/hermes-agent fork 说明

这个 fork 用来开发、维护本机 Hermes 的本地改动。本分支（`neo/notes`）只放说明，不含代码，也不进任何 PR。

## 当前的本地改动

| 改动 | 分支 | 上游 PR | 状态 |
|---|---|---|---|
| 飞书话题：@ 后自动开话题回复（`thread_reply`）；话题内不用再 @，由最后被触发的 bot 接手（`thread_follow: last_active`，`true` 同义；旧行为 `owner` = 开话题的 bot 一直接手） | `feat/feishu-topic-follow` | NousResearch/hermes-agent#151706 | 未合并，本机已在用 |
| 飞书话题投递：发到话题的消息改用 reply API（`reply_in_thread`），不再用 `receive_id_type=thread_id` 调 create（飞书必拒 99992402）。否则 cron 提醒、话题里的图片/文件 全部静默丢 | `feat/feishu-topic-reply`（叠在 `feat/feishu-topic-follow` 之上，不进 PR #151706） | 上游已有同类 PR：#94306、#146771 | 未合并，本机已在用 |

两个补丁目前基于上游 `40d1e95f37`（2026-11-17）。比 2026-11-12 更早的 Hermes 打不上（上游 11-12 改了 `adapter.py`），要先 `hermes update`。

## 在别的机器上安装

安装步骤是私有 repo `neow021/.agents` 里的 skill `hermes-feishu-setup`（会先把权限申请链接发给你）。让那台机器上的 agent（需已登录 `gh`）读取并照做：

`gh api repos/neow021/.agents/contents/skills/hermes-feishu-setup/SKILL.md -H "Accept: application/vnd.github.raw"`

补丁文件放在本分支：`feishu-topic-follow.patch`（话题功能）和 `feishu-topic-reply.patch`（话题投递走 reply API 的修复，必须一起打，否则 cron 提醒和话题里的图片/文件全部 99992402 静默丢），skill 从这里下载。改安装步骤时改那个 skill，不改这里。

## 本机的位置

| 用途 | 路径 |
|---|---|
| fork 开发目录 | `~/dev/hermes-agent`（`origin` = 本 fork，`upstream` = NousResearch） |
| 本说明所在的 worktree | `~/dev/hermes-agent-notes`（`neo/notes` 分支） |
| 补丁备份 | `~/dev/hermes-patches/feishu-topic-follow.patch`、`~/dev/hermes-patches/feishu-topic-reply.patch` |
| 正在运行的 Hermes | `~/.hermes/hermes-agent`，补丁以**未提交改动**的形式打在上面 |

## 本机配置和权限

- feishu 配置（`~/.hermes/profiles/feishu/config.yaml`）中打开了：
  - `platforms.feishu.extra.thread_reply: true`
  - `platforms.feishu.extra.thread_follow: true`（= `last_active`：话题里不带 @ 的消息，由最近一条"bot 发的或 @ 了某个 bot 的消息"对应的 bot 接手；想要旧行为写 `owner`）
  - fable profile（`~/.hermes/profiles/fable/config.yaml`）同样打开了这两个开关；两个 bot 都要跑同一版补丁，判定才一致
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
   应该能看到 `feishu_topic_follow.py`（新文件）和 `adapter.py`（有改动），以及 `cron/scheduler_delivery.py`（有改动）。如果没看到，或者提示有冲突，就把未提交的改动全部撤掉（同上面第 2 步的 `git checkout` 和 `rm`），再重新打两个补丁：
   ```bash
   cd ~/.hermes/hermes-agent
   git apply ~/dev/hermes-patches/feishu-topic-follow.patch
   git apply ~/dev/hermes-patches/feishu-topic-reply.patch
   ```
   报 `patch does not apply`：本机 Hermes 比补丁基准旧就先升级；比补丁基准新，说明上游又改了相关代码，按下面"同步 fork"重新生成补丁，不要手工改代码。
4. **重启 gateway**：在飞书里发 `/restart`，或者在 gateway 之外的终端执行 `hermes gateway restart`。
5. **实测**：
   - 在群里 @ 机器人：会开一个新话题，先出现"收到，处理中…"，再出现正式回复。
   - 在那个话题里不 @ 说一句：机器人接着回。
   - 在话题里 @ 一位同事说一句：机器人不回。
   - 话题投递（第二个补丁）：让 cron 往这个话题投一次（或 `hermes send --to feishu:<oc_id>:<omt_id> "测试"`）。消息要出现在话题里；如果日志出现 `[99992402] field validation failed`，说明第二个补丁没打上或没重启 gateway。

## 同步 fork（上游有新代码时）

两个改动是 fork 里叠在一起的两个提交：`feat/feishu-topic-follow`（1 个提交，就是 PR #151706）和 `feat/feishu-topic-reply`（在它上面再加 1 个提交）。

```bash
cd ~/dev/hermes-agent
git fetch upstream main
git checkout feat/feishu-topic-reply
git rebase upstream/main                     # 两个提交一起变基；冲突一般只在 adapter.py 的 import 和 class 那两行
git branch -f feat/feishu-topic-follow HEAD~1
HERMES_PYTHON=~/.hermes/hermes-agent/venv/bin/python scripts/run_tests.sh \
  tests/gateway/test_feishu*.py tests/agent/test_i18n*.py tests/gateway/test_stream_consumer_thread_routing.py
export PATH=~/.hermes/tools/node-26.12.1-linux-arm64/bin:~/.hermes/tools/npm-12.0.2-linux-arm64/bin:$PATH
~/.hermes/hermes-agent/venv/bin/python scripts/check --base upstream/main   # 必须写 --base：fork 自己的 main 很旧，默认对比会扫上千个文件、十几分钟跑不完
git push -f origin feat/feishu-topic-follow feat/feishu-topic-reply          # PR #151706 会自动更新
git format-patch -1 feat/feishu-topic-follow --base=upstream/main --stdout > ~/dev/hermes-patches/feishu-topic-follow.patch   # --base 写入 base-commit 行，setup skill 靠它判断对方 Hermes 是否太旧
git format-patch -1 feat/feishu-topic-reply  --stdout > ~/dev/hermes-patches/feishu-topic-reply.patch
cp ~/dev/hermes-patches/*.patch ~/dev/hermes-agent-notes/                    # setup skill 从 neo/notes 下载
git -C ~/dev/hermes-agent-notes commit -am "chore: refresh feishu patches" && git -C ~/dev/hermes-agent-notes push
```

- 上游的代码检查有一条规则：`adapter.py` 只能变小、不能变大。新逻辑放进 `feishu_topic_follow.py`。
- 测试要在 `~/.hermes` 之外的目录里跑（比如 `~/dev`）：放在 `~/.hermes` 下面的副本，会被测试自带的"不许碰真实 Hermes 目录"检查拦下，出现假失败。
- 发布前在一个干净的上游副本上按 setup skill 的第 3、3c 步试装一遍，确认两个补丁都能直接打上。
