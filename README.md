# local-dev-env-pitfalls

**本地开发环境的坑位清单**：终端编码 / 沙箱受限环境 / 文件读写与命令行 /
语言与库 —— 全部按"**源头规避 + 兜底排查**"组织
（SKILL.md，兼容 DSH / Claude 风格 skills 格式）。

> 覆盖四层，而且四层的**判据是同一条**：
> 症状看起来像"业务代码写错了"，实际错在更底下；
> 识别信号是"**错的地方太基础了、不像我会犯**"。
> 所以不要按名字以为它只管编码 —— 遇到"环境/工具层面说不通"的问题都先翻这里。

## 思路

90% 的"诡异问题"来自显示层、环境层或工具层，而不是业务逻辑。与其等它出现再排查，不如从源头把它们钉死。每个部分先讲"从源头规避"，出了问题再走排查兜底。

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

## 第三部分：文件读写与命令行（工具会话层）

**源头规避**

- **一个文件只走一条路**：要么全程用编辑器工具，要么全程用 shell；混用时切换前先 `read` 一次
- 批量 / 程序化修改优先走 shell（一次成型，不用来回同步状态）
- heredoc 一律带引号 `<<'EOF'`；多行或含特殊字符的提交信息用 `git commit -F -`

**排查兜底**

- `write`/`edit` 报 `file changed since it was read` / `file no longer exists`
  = 工具侧观察过期（**不是权限问题，重试无用**）⇒ `read` 一次即可恢复
- `git commit -m "... >50% ..."` 里的 `>` 被当重定向 ⇒ 提交**静默失败**
- `python - <<'PYEOF'` 里塞了中文却没设 `PYTHONUTF8=1`
  ⇒ 报 `SyntaxError: unterminated triple-quoted string`（**看起来像语法错，其实是编码错**）

## 第四部分：语言、库与工具的陷阱

**源头规避**

- numpy：`frombuffer` 是只读（要 `.copy()`）；`view`/`reshape` 之后**先 print shape**
- f-string 不嵌套同种引号（Python < 3.12）；短变量名（`d`/`w`）在长脚本里会撞车
- 拿到数组先**断言量纲**（0~1 还是 0~255）
- Windows 仓库加 `.gitattributes`（`* text=auto eol=lf`）

**排查兜底**

- 外部工具调不通（如 `texconv.dll` 报 `80004002`）⇒ **先问"自己实现要多久"**，
  几十行的（BC1 解码之类）直接自己写，别修 COM
- **"能打开" ≠ "解对了"**：`PIL` 能开 BC7 的 DDS 但 alpha 是常量 ⇒ 关键结论要交叉验证
- `import` 别的脚本会执行它的模块级代码；长命令加 `timeout` 或丢后台

## 用法

```bash
# 作为 DSH skill 使用
cp SKILL.md ~/.dsh/skills/local-dev-env-pitfalls/SKILL.md
```

或直接作为清单参考。
