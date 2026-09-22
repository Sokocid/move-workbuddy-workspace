# move-workbuddy-workspace

安全地把 WorkBuddy 工作空间（项目文件夹）迁移到其他磁盘或目录的 Agent Skill。

适用于 WorkBuddy / CodeBuddy 这类本地 agent——把 `SKILL.md` 放进宿主的 skills 目录即可被发现。

## 安装

```bash
git clone --depth 1 https://github.com/Sokocid/move-workbuddy-workspace ~/.workbuddy/skills/move-workbuddy-workspace
```

其他宿主的 skills 目录：

| 宿主 | skills 目录 |
| --- | --- |
| WorkBuddy | `~/.workbuddy/skills/` |
| CodeBuddy | `~/.codebuddy/skills/` |
| Claude Code | `~/.claude/skills/` |
| Cursor | `~/.cursor/skills/` |
| Codex | `~/.agents/skills/` |

装好后**重启 agent 会话**，skill 才会被重新扫描发现。

## 它解决什么问题

直接把工作空间文件夹「剪切粘贴」会踩三个坑：

1. **删源必然失败** —— WorkBuddy 进程持有工作空间**根目录**的独占句柄（文件监听 + 把它作为工作目录）。删除时 `SHFileOperationW` 返回 `32 (ERROR_SHARING_VIOLATION)`，`os.rename` 同样失败。实际行为是：**目录内的文件能成功删除，只有根目录本身删不掉**，于是留下一个空壳目录。
2. **路径记录散落多处** —— 不更新的话，界面上工作空间会指向一个已不存在的路径。
3. **项目历史不会跟随** —— 会话、改动记录、文件历史都按旧路径哈希存放，新位置会表现为一个全新项目。

## 核心原则

- 先复制 + 逐文件哈希校验，确认无误后才动源目录。
- 删除走回收站（`FOF_ALLOWUNDO`）而不是硬删。
- 不要为了删源去 kill WorkBuddy —— 那会杀掉你自己的会话。

## 预期结果

复制与校验全部通过；原目录内容进回收站；根目录留下一个空壳，**必须重启 WorkBuddy 后**才能清掉。这是真实结果，不是失败。

## 适用范围与要求

- Windows + WorkBuddy（桌面端）。其他宿主用不了「更新 workspace-picker.json」那一步，其余步骤通用。
- Python 3.8+（只用标准库）；Git Bash 或其他 POSIX shell。

## 跨宿主注意

Git Bash 的 `/c/...` 路径传不过去原生 `python.exe`，会静默变成「文件不存在」。详见 [SKILL.md](./SKILL.md) 的「MSYS 路径 vs 原生程序」一节。

## License

[MIT](./LICENSE)
