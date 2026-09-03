---
name: console-encoding-triage
description: Windows/Linux 终端中文乱码与沙箱环境坑的系统性规避与排查：从源头把编码钉死（UTF-8 单编码铁律：显式 encoding、PYTHONUTF8/reconfigure、chcp 65001），把受限环境的清理坑规避掉（TemporaryDirectory(ignore_cleanup_errors=True)、本地目录手动清），再退到排查法兜底（ascii()、UTF-8 强制、文件重定向、tempfile 本地目录绕开）。
whenToUse: 写脚本/跑测试前想预先避免中文乱码；或输出/报错里出现乱码（������、锟斤拷、mojibake）、python -c 传中文参数乱码、测试诡异失败疑似编码问题；或异常栈出现在 tempfile/shutil 清理代码或工具会话层（PermissionError [WinError 5]、shell reset）却与业务逻辑无关时。
user-invocable: true
---

# 控制台编码与沙箱环境：从源头规避

> 核心原则：**先让问题不发生，再兜底排查。** 90% 的"诡异问题"来自显示层或环境层，从源头把它钉死，它根本不会出现。每个部分先讲"从源头规避"，再讲排查兜底。

## 第一部分：中文乱码 —— 从源头规避

### 0. 先懂一件事：乱码的源头是"多编码共处"，不是"数据坏了"

乱码几乎从不来自数据真损坏，而是来自**同一个字节流被两种不同编码解读**：写入时用了 locale 默认编码、读取却只按 UTF-8；或终端码页是 cp936 却收到 UTF-8 字节。**只要全链路统一成 UTF-8，乱码根本不会发生。**

**UTF-8 单编码铁律（三条）：**
1. 永不依赖 locale 默认编码——凡是文件/流，都显式写 `encoding="utf-8"`。
2. 进程/会话级钉死 UTF-8——让 `PYTHONUTF8=1` / `PYTHONIOENCODING=utf-8` 在进程启动前生效。
3. 终端码页对齐 UTF-8——`chcp 65001`；git-bash 里写中文用 `printf` 而非 `echo`。

### 1. 源头规避三件套（写代码/跑脚本前就用，不是出问题才做）

**A. 脚本/库开头钉死（最稳，放文件最顶端）：**
```python
import sys
# 把本进程及它启动的每个子进程都钉在 UTF-8（环境变量需在进程启动前设才最有效，
# 此处是为让子进程/后续插件继承）；并直接重配标准流，绕过控制台码页。
try:
    sys.stdout.reconfigure(encoding="utf-8")
    sys.stderr.reconfigure(encoding="utf-8")
    sys.stdin.reconfigure(encoding="utf-8")
except AttributeError:
    pass  # Python < 3.7 没有 reconfigure，改用下面 B/C 步
```

**B. 读写文件永远显式 UTF-8（真正的源头，务必写全）：**
```python
with open(path, "w", encoding="utf-8") as f: ...   # 写入
with open(path, "r", encoding="utf-8") as f: ...   # 读取
# 绝不裸写 open(path) —— 它用 locale 默认编码，跨平台/环境不一致，这就是乱码源头。
```

**C. 进程/会话级一次性生效，之后所有脚本自动继承：**
| 做法 | 命令 |
|---|---|
| PowerShell 环境变量 | `$env:PYTHONUTF8=1`；`$env:PYTHONIOENCODING=utf-8` |
| bash 环境变量 | `export PYTHONUTF8=1; export PYTHONIOENCODING=utf-8` |
| 命令行启动 | `python -X utf8 script.py` |
| 只强制输出编码 | `PYTHONIOENCODING=utf-8 python script.py` |
| 终端码页 | 先 `chcp 65001`（Windows，回退 936） |

**D. 源文件第一行** `# -*- coding: utf-8 -*-`：Python 3 默认已是 UTF-8，写上可防老项目/部分编辑器解读偏差。

**E. git-bash 写含中文的行**：`printf '%s\n' '中文' > out.txt`（默认按 UTF-8 写字节）；别用 `cat`/`echo` 拼接——码页不同结果不同。

**✔ 应用 A–E 之后，乱码基本不会出现。** 万一仍出现 → 走"2. 排查（兜底）"。

### 2. 排查（兜底）：三步定位，别用眼睛读乱码

1. **无歧义判定（最重要、最快）**——用 `ascii()` 而不是 `repr()`（`repr()` 不转义可打印的 CJK，`repr('武器')=='"\'武器\'"'`，不适合做判定）：
   ```python
   print(ascii(some_string))   # '\u6b66\u5668...' 看到规范码点就是对的
   print(ascii(some_list))     # "['\\u6b66\\u5668', ...]" 容器同样安全
   ```
   - `ascii()` 正确（`\uXXXX` 规范）→ 数据层没问题，乱码只是显示层 → 修输出，**别改业务代码**。
   - `ascii()` 就错 → 才是真编码 bug，才查文件编码/解码。
2. **重定向到文件再读**（绕开终端码页）：`python script.py > out.txt 2>&1`，再用 `Get-Content out.txt -Encoding UTF8`（PowerShell）读。
3. **怀疑文件本身时比字节**：`raw=open('file.py','rb').read(); raw.find('目标文本'.encode('utf-8'))`（用 bytes 搜，别用 str）。

### 3. 三层模型 & 逻辑层陷阱

| 层 | 常见表现 | 典型原因 |
|---|---|---|
| 显示层 | `������`、`锟斤拷`、`烫烫烫` | 终端码页 vs 输出编码不一致（cp936/GBK 读 UTF-8 字节） |
| 传输层 | 管道/文件正常，终端显示乱 | 重定向/管道时 Python 回退 `locale.getpreferredencoding()`，不用宽字符控制台 API |
| 逻辑层 | 数据真损坏（罕见） | 文件被错误编码重写、字符串混入错误引号等 |

- ⚠️ `python -c "print('中文')"` 传中文：**仅 Linux/macOS** 且在非 UTF-8 模式下，Python 用 `locale.getpreferredencoding()` 解码 `-c` 源码串；**Windows 原生 Python 的 `-c` 来自 UTF-16 宽命令行，源码不会乱**（实测 cp936 + 非 UTF-8 模式 `python -c "print(ascii('中文'))"` → `'\u4e2d\u6587'` 完好）。所以 Windows 上 `-c` 传中文出乱码基本都在 stdout 显示层，按第一部分排查即可，别怀疑源码解码。
- ❌ 在 Windows 控制台输出里用肉眼核对中文字符串是否相等。
- ❌ 看见乱码就清 `__pycache__`、怀疑编码损坏 → 先做 ascii 判定，90% 白折腾。
- ✅ 调试探针写在断言**旁边**（同一次调用），同时打印 `ascii(needle)` 和 `ascii(haystack)`——曾踩坑：探针放方法开头、断言在结尾，数据"看起来都对"，真相是 `assertIn` 对 list 做元素等值而非子串匹配（`"武器大师" in ["武器大师: ..."]` 为 False）。这类是**逻辑层**陷阱：确认对象是 list 后回业务代码改断言，别再在编码/环境上耗。
- ✅ 给用户/团队的入口统一加启动提示：`chcp 65001` 或 `PYTHONUTF8=1`，从源头避免他人环境再踩乱码。

## 第二部分：沙箱 / 受限环境 —— 从源头规避

### 0. 先懂一件事：罪魁是 tempfile 清理时的权限重置

`tempfile.TemporaryDirectory` 在受限 token/沙箱下**创建没问题，清理时会 `chmod(0o700)` 重置权限** → 被拒 → `PermissionError: [WinError 5]`，整个测试报错，还误以为是测试逻辑挂了。**源头规避 = 让清理动作不触发这次 chmod。**

### 1. 源头规避（优先用，别在受限环境反复重试）

**A. 首选：`ignore_cleanup_errors=True`（Python 3.10+）——直接跳过清理失败，一行搞定：**
```python
import tempfile
with tempfile.TemporaryDirectory(ignore_cleanup_errors=True) as tmp:
    ...   # 干净利落，清理阶段不会再报 PermissionError
# contextlib 版同样支持：
from contextlib import TemporaryDirectory
with TemporaryDirectory(ignore_cleanup_errors=True) as tmp:
    ...
```

**B. 或用项目内本地目录 + 手动清理（可控，兼容旧 Python）：**
```python
work = os.path.join(os.getcwd(), ".work")   # 选工作区内已确认可写的目录
os.makedirs(work, exist_ok=True)
try:
    ...                     # 测试用 work
finally:
    shutil.rmtree(work, ignore_errors=True)   # ignore_errors 兜底
```
（`os.path.dirname(__file__)` 所在源码目录可能只读，`makedirs` 会挂；优先工作区内已确认可写的临时目录。更彻底是把这段封成 helper / pytest fixture，从源头统一接管清理。）

**C. 临时文件也落本地**：`tempfile.mkstemp(dir=<可写目录>)`，或在 `work` 里 `open(..., "w", encoding="utf-8")`（顺手遵循第一部分的 UTF-8 铁律）。

**✔ 从源头规避后，清理阶段的 PermissionError 不再出现。** 若仍出现 → "2. 兜底确认"。

### 2. 兜底确认（如果确实又看到了）

1. **定位异常归属**：看 traceback 最后几帧在业务代码还是框架/环境代码。框架清理代码（`tempfile._resetperms`、`shutil.rmtree` 的 `onerror`/`onexc`、OS 层 `[WinError 5]`、工具会话层 `shell reset`/`write failed`）→ 环境坑，绕开或忽略，别改业务。
2. **旧写法绕开**：`with contextlib.suppress(PermissionError): tmp.cleanup()`。
3. **工具会话故障不硬刚**：bash 报 `shell reset`/`write failed` 换 pwsh 继续；命令被杀后 `[exit code: 1]` 无信号标记 = 中断不是失败。
4. **区分"测试失败"与"环境噪音"**：`Ran 28 tests ... FAILED (failures=1, errors=2)` 里 errors 可能全是环境噪音，先看 failures；清掉噪音再重跑确认。

### 3. 反模式

- ❌ 环境异常出现后反复审查业务代码、重构无关逻辑。
- ❌ `tempfile.TemporaryDirectory` 在受限环境反复重试（每次都在清理时失败）——应改用 `ignore_cleanup_errors=True` 或本地目录。
- ❌ 把 `rmtree` 的权限错误当成数据损坏去排查。

## 结尾检查清单（先做源头规避，一次到位，再谈兜底）

- [ ] UTF-8 单编码铁律已落地？——文件读写显式 `encoding="utf-8"`、脚本/标准流已钉死 UTF-8、终端已 `chcp 65001`？
- [ ] 受限环境下临时目录用了 `ignore_cleanup_errors=True` 或本地目录 + `ignore_errors=True`？（清理阶段不再冒 PermissionError）
- [ ] 若乱码/异常仍出现：输出看过 `ascii()` 版本了吗？`PYTHONUTF8=1 / -X utf8 / chcp 65001` 试过了吗？
- [ ] 断言对象是 list 还是 str？（list 的 `in` 是元素等值！——逻辑层陷阱，确认后回业务代码）
- [ ] traceback 最后一帧在业务代码还是框架/环境代码？
