WorkBuddy Skills
给 WorkBuddy / CodeBuddy 这类本地 agent 使用的 Skill 集合。
每个子目录是一个独立 skill；根目录的 README.md 只负责索引与安装说明。
安装
一个 skill 就是一份 SKILL.md（可选带 references/、scripts/ 等）。把它放进宿主 agent 的 skills 目录即可：
      宿主
      skills 目录
      WorkBuddy
      ~/.workbuddy/skills/
      CodeBuddy
      ~/.codebuddy/skills/
      Claude Code
      ~/.claude/skills/
      Cursor
      ~/.cursor/skills/
      Codex
      ~/.agents/skills/
git clone https://github.com/Sokocid/workbuddy-skills.git
cp -r workbuddy-skills/move-workbuddy-workspace ~/.workbuddy/skills/
装好后重启 agent 会话，skill 才会被重新扫描发现。

Skill 列表

move-workbuddy-workspace

安全地把 WorkBuddy 工作空间（项目文件夹）迁移到其他磁盘或目录。
适用场景：想把工作空间从 C 盘挪到 D 盘，或改工作目录位置。
它解决的三个坑
1. 删源必然失败 —— WorkBuddy 进程持有工作空间根目录的独占句柄（文件监听 + 把它作为工作目录）。删除时 SHFileOperationW 返回 32 (ERROR_SHARING_VIOLATION)，os.rename 同样失败。实际行为是：目录内的文件能成功删除，只有根目录本身删不掉，于是留下一个空壳目录。
2. 路径记录散落多处 —— 不更新的话，界面上工作空间会指向一个已不存在的路径。
3. 项目历史不会跟随 —— 会话、改动记录、文件历史都按旧路径哈希存放，新位置会表现为一个全新项目。
核心原则
- 先复制 + 逐文件哈希校验，确认无误后才动源目录。
- 删除走回收站（FOF_ALLOWUNDO）而不是硬删。
- 不要为了删源去 kill WorkBuddy —— 那会杀掉你自己的会话。
预期结果：复制与校验全部通过；原目录内容进回收站；根目录留下一个空壳，必须重启 WorkBuddy 后才能清掉。这是真实结果，不是失败。
同一位置还有一份跨宿主坑位说明（Git Bash 的 /c/... 路径传不过去原生 python.exe，会静默变成「文件不存在」），如果你在别的宿主里跑这套流程，先看那一节。
License
MIT
