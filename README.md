# console-encoding-triage

Windows/Linux 终端中文乱码与沙箱环境坑的系统性排查法（SKILL.md，兼容 DSH / Claude 风格 skills 格式）。

## 内容

- **第一部分：中文乱码 / 编码问题排查**
  - 三层模型：显示层 / 传输层 / 逻辑层（90% 的"诡异问题"来自显示层或环境层，先隔离再查逻辑）
  - `ascii()` 无歧义判定（`repr()` 不转义 CJK，勿用）
  - UTF-8 强制：`PYTHONUTF8=1` / `python -X utf8` / `chcp 65001` / `PYTHONIOENCODING=utf-8`
  - 重定向到文件再读、磁盘字节对比（`bytes.find`）、逐码点对比
  - 常见陷阱：list 的 `in` 是元素等值而非子串匹配、`python -c` 传中文的平台差异
- **第二部分：沙箱 / 受限环境坑排查**
  - `tempfile.TemporaryDirectory` 清理时 `PermissionError [WinError 5]`（`_resetperms` chmod 被拒）
  - `shutil.rmtree` 权限错误、shell 会话损坏（`shell write failed; the shell was reset`）的识别与绕开
  - 区分"测试失败"与"环境噪音"

## 用法

```bash
# 作为 DSH skill 安装
cp SKILL.md ~/.dsh/skills/console-encoding-triage/SKILL.md
```

或直接作为排查 checklist 参考。

## 反模式速记

- 看见乱码别急着改业务代码，先 `print(ascii(x))` 判显示层
- 断言对象先分清 str 还是 list
- traceback 最后一帧在框架/环境代码里时，别往业务代码上查