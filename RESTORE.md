# OpenClaw 备份恢复（同一台 Mac）

这份教程适用于 `/Users/guyanhua/claw_backup/openclaw-2026-10-03.tar.gz` 及相同布局的后续备份。请在 **macOS 终端**操作，而不是依赖待恢复的 OpenClaw 会话。先安装可用的 OpenClaw CLI，最好与备份时的版本相同或兼容；此归档由 OpenClaw 2026.9.6 创建。

以下命令在**同一个终端窗口**依次运行。每一步有报错就停下来，不要继续下一步。此流程会用备份替换新安装的 OpenClaw 配置；新安装原有的目录会改名保留，以便恢复失败时撤销这次替换。

## 1. 选归档并验证

把第一行改成你要恢复的归档实际路径，然后运行：

```sh
archive="$HOME/claw_backup/openclaw-2026-10-03.tar.gz"
openclaw backup verify "$archive"
```

只有验证成功才继续。归档含模型和渠道凭据，不要上传或分享。

## 2. 解压到新的暂存目录

```sh
staging=$(mktemp -d "$HOME/openclaw-restore.XXXXXXXX")
openclaw backup restore "$archive" --target "$staging"
restored=$(find "$staging" -path '*/payload/posix/Users/guyanhua/.openclaw' -type d -print -quit)
test -f "$restored/openclaw.json" && printf '找到备份配置：%s\n' "$restored"
```

必须看到“找到备份配置”才能继续。官方 `restore` **只在暂存目录解压，不会替换当前配置**。这份归档只有 `/Users/guyanhua/.openclaw` 一个待恢复目录，因此无需手工阅读 `manifest.json`；若未来归档布局不同，不要猜路径。

## 3. 停止 Gateway，保留现有状态

```sh
openclaw gateway stop --disable
openclaw gateway status
```

确认 Gateway 已停止后继续；`--disable` 是防止 macOS 的服务在恢复过程中自动重启。请不要在有其他 OpenClaw 进程写入状态时替换目录。

```sh
old_state="$HOME/.openclaw.before-restore-$(date +%Y%m%d-%H%M%S)"
mv "$HOME/.openclaw" "$old_state"
ditto "$restored" "$HOME/.openclaw"
test -f "$HOME/.openclaw/openclaw.json" && echo '配置文件已放回'
```

不要删除 `old_state`；如果 `mv` 或 `ditto` 报错，请跳到下方“恢复失败时撤销”，不要启动有缺失文件的 Gateway。

## 4. 启动并检查

```sh
openclaw gateway start
openclaw gateway status
openclaw doctor
openclaw agents list
openclaw automations list --all
```

如果 `start` 报服务未安装，可运行 `openclaw gateway install` 后再查状态。检查模型、渠道、插件、两个 agent、工作区与定时任务。归档不包含外部软件、macOS 钥匙串单独保存的秘密或插件的 `node_modules`；缺的依赖需要重装，过期的授权需要重新登录。确认运行正常后，再处理包含密钥的暂存目录和旧目录；不要把它们上传。

## 恢复失败时撤销本次操作

**只有恢复失败时才执行这一节；恢复成功则不用执行。**这里会把步骤 3 保留的目录放回去，让 OpenClaw 回到开始本次恢复操作之前的状态。例如，若开始时是“只配置了 OpenAI 的全新安装”，撤销后也只会回到那个全新安装，**不会**变成备份里的旧状态。备份归档本身不会被修改。

在**同一个终端窗口**，`old_state` 仍指向步骤 3 保留的原目录：

```sh
openclaw gateway stop --disable
[ ! -e "$HOME/.openclaw" ] || mv "$HOME/.openclaw" "$HOME/.openclaw.failed-restore-$(date +%Y%m%d-%H%M%S)"
mv "$old_state" "$HOME/.openclaw"
openclaw gateway start
openclaw gateway status
```

如果关掉了终端，先运行 `ls -d "$HOME"/.openclaw.before-restore-*` 找到保留目录，用它的完整路径替换 `"$old_state"`。整个流程不删除原有目录。
