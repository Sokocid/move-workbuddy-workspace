---
name: move-workbuddy-workspace
description: Use when the user asks to move, migrate or relocate a WorkBuddy workspace or project folder to another drive or directory on Windows. Covers the safe procedure - robocopy, per-file hash verification, updating workspace-picker.json, and sending the source to the Recycle Bin - including the pitfall that WorkBuddy holds an exclusive handle on the workspace root, so the source's contents move but the now-empty root directory cannot be deleted until the app restarts.
---

# 迁移 WorkBuddy 工作空间

**English quick start** — This skill relocates a WorkBuddy workspace folder on Windows
safely: (1) recon read-only and confirm with the user; (2) `robocopy /E` to the new
location; (3) verify every file by SHA256 **before** touching the source; (4) update
`workspace-picker.json`; (5) send the source to the Recycle Bin via `SHFileOperationW`
with `FOF_ALLOWUNDO`. Expect exit code `32 (ERROR_SHARING_VIOLATION)`: WorkBuddy holds an
exclusive handle on the workspace **root**, so the contents move but the now-empty root
directory survives until the app restarts. That is a normal outcome, not a failure.
Details below are in Chinese.

**适用范围**：Windows + WorkBuddy（桌面端）。其他宿主（Claude Code / CodeBuddy 等）
步骤 2 不适用，其余步骤通用。

## 为什么需要这个流程

直接把工作空间文件夹「剪切粘贴」会踩三个坑：

1. **删源一定失败**：WorkBuddy 进程持有工作空间**根目录**的独占句柄（文件监听 + 把它当工作目录）。
   `SHFileOperationW` 返回 **32 (ERROR_SHARING_VIOLATION)**，`os.rename` 也失败。
   实测行为：**目录内的文件可以成功删除，只有根目录本身删不掉** —— 所以会留下一个空壳目录。
2. **路径记录散落多处**：不更新的话，界面上工作空间指向一个已不存在的路径。
3. **项目历史不跟随**：会话 / 改动 / 文件历史按旧路径哈希存放，新位置会表现为全新项目。

## 步骤

### 0. 先只读勘察，并跟用户确认

```bash
ls -la "<workspace>"          # 内容与大小
du -sh "<workspace>"
ls -la "<target-parent>"      # 目标是否存在、是否为空
```

必须告知用户的：

- 工作空间里有哪些内容（尤其有没有用户的业务文件）。
- 本次会话会中断（工作目录失效），需要重启 WorkBuddy。
- 旧的项目历史（对话、改动记录）不会自动跟随。
- 有副作用的操作，**拿到用户明确确认再动手**。

### 1. 复制 + 校验（不要先删）

```bash
export MSYS2_ARG_CONV_EXCL="*"   # 关键！否则 Git Bash 会把 /E /COPY 当路径转换掉
robocopy "<src>" "<dst>" /E /COPY:DAT /DCOPY:DAT /R:1 /W:1 /NP /NFL /NDL
# robocopy 退出码 0-7 都算成功；16+ 才是失败
```

然后逐文件哈希比对，**必须在删源之前**：

```bash
cd "<src>" && find . -type f -print0 | xargs -0 -r sha256sum | sort -k2 > /tmp/src.txt
cd "<dst>" && find . -type f -print0 | xargs -0 -r sha256sum | sort -k2 > /tmp/dst.txt
diff /tmp/src.txt /tmp/dst.txt && echo "一致"
```

> `-r` 是给空目录兜底的（否则 `sha256sum` 会退化成读 stdin 而空转）。
> 非 Git Bash 环境（macOS / Linux）把 `sha256sum` 换成 `shasum -a 256`。

比对的两个要点：两边都用 `cd` 进目录再 `find .`，**输出里的路径才会是同构的相对路径**；
`sort -k2` 保证行序一致，否则 `diff` 会因顺序差而误报。

### 2. 更新 WorkBuddy 的工作空间列表

```bash
grep -rl "<旧路径关键字>" ~/.workbuddy/storage/ 2>/dev/null
```

典型位置：`~/.workbuddy/storage/user-<uuid>-personal/scoped/<hash>/workspace-picker.json`

结构很简单，改 `explicit-workspaces[].path` 为新路径（保留 `label`）即可。
**grep 没结果就跳过这步**——说明用户没用过工作空间选择器，下次启动时应用会自己重建。

> 注意：WorkBuddy 正在运行时，内存里的旧列表**可能覆盖这个文件**。改完要提醒用户重启后再确认一次。

### 3. 把源目录送回收站（不要硬删）

用 `SHFileOperationW` + `FOF_ALLOWUNDO`，而不是 `rm -rf`：

```python
import ctypes
from ctypes import wintypes

class SHFILEOPSTRUCTW(ctypes.Structure):
    _fields_ = [("hwnd", wintypes.HWND), ("wFunc", wintypes.UINT),
                ("pFrom", wintypes.LPCWSTR), ("pTo", wintypes.LPCWSTR),
                ("fFlags", ctypes.c_uint16), ("fAnyOperationsAborted", wintypes.BOOL),
                ("hNameMappings", ctypes.c_void_p), ("lpszProgressTitle", wintypes.LPCWSTR)]

op = SHFILEOPSTRUCTW()
op.wFunc  = 3                                    # FO_DELETE
op.pFrom  = r"C:\path\to\workspace" + "\0\0"      # 必须双 \0 结尾；且必须绝对路径
op.fFlags = 0x0040 | 0x0010 | 0x0004 | 0x0400    # ALLOWUNDO|NOCONFIRMATION|SILENT|NOERRORUI
print(ctypes.windll.shell32.SHFileOperationW(ctypes.byref(op)))
```

返回码的含义：

| 返回码 | 含义 |
| --- | --- |
| **32** | 目录内容进回收站成功，根目录因句柄锁失败 —— **最常见，属正常结果** |
| **0** | 全部成功（宿主没有锁住根目录），同样正常，不是异常 |

两种情况都不需要重试。**判断成败要看源目录的实际状态，不是返回码。**

> 别被返回码误导：`32` 并不意味着「什么都没发生」。实测中目录内容已经进了回收站，只有根目录那一步失败。
> 反过来，同一目录重复删除时 `32` 会被重复返回，别据此判断第一次没生效。

### 4. 验证并把状态如实报告

```bash
ls -la "<src>"     # 大概率只剩空目录
ls -la "<dst>"     # 应完整
```

想看回收站里的东西（确认可恢复），解析 `$I*` 元数据：

```python
import os, struct
rb = r"C:\$Recycle.Bin\<SID>"     # 当前用户 SID 那个目录
for n in os.listdir(rb):
    if not n.startswith("$I"):
        continue
    d = open(os.path.join(rb, n), "rb").read()
    nlen = struct.unpack_from("<I", d, 24)[0]
    print(d[28:28 + nlen * 2].decode("utf-16-le").rstrip("\0"))
```

### 5. 交给用户做最后一步

WorkBuddy 运行期间**空壳目录删不掉**（改名也不行）。必须：

1. 重启 WorkBuddy；
2. 从新路径打开项目，确认工作空间列表指向正确；
3. 再清掉旧的空壳目录（回收站即可）。

## 关键判断

- **永远先复制 + 校验，再删源**。任何时刻都要有一份完整副本存在。
- **不要为了删源去 kill WorkBuddy 进程** —— 那会杀掉你自己的会话。
- 不要尝试改 `~/.workbuddy/projects/`、`changes-index/`、`file-history/` 等历史数据去「保住历史」：那些按路径哈希组织，手工改容易损坏。**如实告诉用户历史不会跟随。**
- 如果同一目录被删除两次，`SHFileOperationW` 第二次会重复返回 32，别据此判断第一次没生效。

## 宿主环境先检查（沙箱化 agent 常见坑）

这类操作常发生在被沙箱限制的 agent 宿主里。动手前先确认通道可用：

- **优先尝试 bash，别只依赖 PowerShell 工具**：部分宿主里 PowerShell 工具的每次调用都会静默失败（退出码 1、无任何输出）。用 bash 跑 `echo` / `pwd` 先验证通道。
- **bash 工具的 PATH 可能是空的**：`ls`、`find`、`sha256sum` 全部报 command not found。先定位宿主自带的 POSIX 工具链目录再导出。若宿主（如 WorkBuddy）把 Git 打包在 `binaries/PortableGit/` 下：

  ```bash
  export PATH="<PortableGit>/usr/bin:/c/Windows/System32:$PATH"
  ```

  环境变量**不会跨工具调用保留**，每次调用都要重新导出。

- **bash 里调用 `powershell` / `rundll32` / `reg.exe` 可能被安全策略直接拦截**，不要把它们当成可行路径。
- **需要解析 Windows 二进制元数据时用 Python**（回收站 `$I*` 文件、`CreateFileW` 句柄探测等）。
- 判断「谁锁住了目录」的可靠方法：用 `CreateFileW` 以 `dwShareMode=0` 打开，失败即被占用。如果**根目录被锁而内部文件都没被锁**，就是有进程持有目录句柄（工作目录或目录监听），而不是文件级占用。

## 跨宿主最容易踩的坑：MSYS 路径 vs 原生程序

Git Bash 用 `/c/...`、`/d/...` 形式表示盘符，**但原生 Windows 程序不认这套写法**。
把 `/d/foo` 传给 Python / robocopy 这类原生 exe，会得到「文件不存在」而不是报错提示。

```bash
# 反例：Python 是原生 exe，读不到 MSYS 路径
python -c "import os; print(os.path.exists('/d/Software D Data'))"   # -> False

# 正确：给原生程序用 Windows 式路径
python -c "import os; print(os.path.exists(r'D:\Software D Data'))"  # -> True
```

规则：

| 程序类型 | 传给它的路径写法 |
| --- | --- |
| bash 内建 / Git Bash 自带工具（`ls`、`find`、`sha256sum`、`diff`） | `/c/...`、`/d/...` |
| 原生 exe（`python.exe`、`robocopy`、系统工具） | `C:\...`、`D:\...` |

这条同样适用于本节里的两个 Python 片段——**其中 `pFrom` 和 `rb` 必须写成 `C:\...` 形式**。
