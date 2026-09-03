# console-encoding-triage

Windows/Linux 终端中文乱码与沙箱环境坑的**源头规避 + 兜底排查**（SKILL.md，兼容 DSH / Claude 风格 skills 格式）。

## 思路

90% 的"诡异问题"来自显示层或环境层，而不是业务逻辑。与其等它出现再排查，不如从源头把它们钉死。每个部分先讲"从源头规避"，出了问题再走排查兜底。

## 第一部分：中文乱码

**源头规避（UTF-8 单编码铁律）**

- 读写文件永远显式 `open(path, "w"/"r", encoding="utf-8")`，绝不裸写 `open(path)` 用 locale 默认编码
- 脚本开头 `sys.stdout/stderr/stdin.reconfigure(encoding="utf-8")`
- 进程/会话级 `PYTHONUTF8=1`、`PYTHONIOENCODING=utf-8`、`python -X utf8`、终端 `chcp 65001`
- 源文件第一行 `# -*- coding: utf-8 -*-`
- git-bash 写中文用 `printf '%s\n' '中文' > out.txt`，别用 `cat`/`echo` 拼

**排查兜底**

- 用 `ascii()` 做无歧义判定（`repr()` 不转义 CJK，勿用）
- 重定向到文件再读、磁盘字节对比（`bytes.find`）
- 三层模型：显示层 / 传输层 / 逻辑层
- 常见坑：list 的 `in` 是元素等值而非子串匹配、`python -c` 传中文的平台差异

## 第二部分：沙箱 / 受限环境

**源头规避**

- `tempfile.TemporaryDirectory(ignore_cleanup_errors=True)`（Python 3.10+），一行跳过清理失败
- 或用工作区内本地目录 + `shutil.rmtree(work, ignore_errors=True)` 手动清
- 临时文件落本地目录：`tempfile.mkstemp(dir=...)` 或在 `work` 里显式 UTF-8 写

**排查兜底**

- 先看 traceback 最后几帧在业务代码还是框架/环境代码
- 清理阶段 `PermissionError [WinError 5]` 来自 `_resetperms` 的 chmod 被拒，不是逻辑挂了
- 区分"测试失败"与"环境噪音"；shell 会话损坏（`shell write failed; the shell was reset`）换工具绕开

## 用法

```bash
# 作为 DSH skill 使用
cp SKILL.md ~/.dsh/skills/console-encoding-triage/SKILL.md
```

或直接作为清单参考。
