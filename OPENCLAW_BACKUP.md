# OpenClaw 本地每日备份

`openclaw-daily-local-backup` 是 OpenClaw 自身的命令型定时任务。每天 04:30（Asia/Shanghai）运行 `scripts/openclaw_daily_backup.py`，调用官方 `openclaw backup create --verify`，并再次调用 `openclaw backup verify`。

归档位于 `/Users/guyanhua/claw_backup/openclaw-YYYY-MM-DD.tar.gz`，每天最多一个。仅在当天归档通过验证后，清理此目录中符合这个文件名规则、日期超过保留窗口的旧归档；最多保留最近七天的七个归档。其他文件及其他目录不会被清理。目录权限为 `0700`，归档权限为 `0600`。失败时不轮换已有归档。

这是本机备份：如果硬盘损坏、整机丢失，或该目录也被删除，归档会一同丢失。归档包含凭据，应当像密钥一样保护。

## 核实及恢复

先选择一个归档，运行：

```sh
openclaw backup verify /Users/guyanhua/claw_backup/openclaw-YYYY-MM-DD.tar.gz
openclaw backup restore /Users/guyanhua/claw_backup/openclaw-YYYY-MM-DD.tar.gz --target "$HOME/openclaw-restore-staging"
```

`restore` 只校验并解压到一个新的空暂存目录，**不会直接覆盖正在运行的 `~/.openclaw`**。检查暂存目录中的 `manifest.json`，根据 `assets` 的源路径和归档路径找到配置、状态、额外的 agent 根目录与工作区。停止 Gateway 和使用这些文件的节点，把现有状态移到可回退的位置，再将暂存的文件放回清单记录的原始路径；运行 `openclaw doctor` 后重启 Gateway。

官方完整归档默认包含配置、`credentials/`、认证档案、渠道与模型凭据、会话、agent 数据库和工作区；数据库使用一致性快照，但配置与所有数据库不是同一时刻的原子快照。归档不包含 macOS 钥匙串中单独保存的秘密、外部软件和插件 `node_modules`。恢复后可能需要重装或更新插件、运行 `openclaw skills list` 重建索引，并对过期授权的渠道重新登录。先阅读 `openclaw backup restore --help` 和 OpenClaw 的恢复文档；不要把解压的目录直接覆盖进运行中的 Gateway。
