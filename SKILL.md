---
name: console-encoding-triage
description: Windows/Linux 终端中文乱码与沙箱环境坑的系统性排查法：先隔离显示层再查逻辑层（ascii()、UTF-8 强制、文件重定向），以及沙箱/受限 shell 环境类异常（tempfile 清理 PermissionError、shell 会话损坏）的识别与绕开。
whenToUse: 输出/报错里出现乱码（������、锟斤拷、mojibake）、python -c 传中文参数乱码、测试诡异失败疑似编码问题、或异常栈出现在 tempfile/shutil 清理代码或工具会话层（PermissionError [WinError 5]、shell reset）却与业务逻辑无关时。
user-invocable: true
---

# 控制台编码与沙箱环境排错 Skill

> 沉淀自真实项目教训：曾因乱码误导，把 1 分钟能定位的断言 bug 排查了 20 分钟。
> 核心原则：**90% 的"诡异问题"来自显示层或环境层，先把它们隔离掉，再花时间在逻辑层。**

## 第一部分：中文乱码 / 编码问题排查

### 1. 先理解症状来源（三层模型）

| 层 | 常见表现 | 典型原因 |
|---|---|---|
| 显示层 | 输出全是 `������`、`锟斤拷`、`烫烫烫` | 终端码页 vs 输出编码不一致（Windows 控制台 cp936/GBK 读 UTF-8 字节） |
| 传输层 | 管道/文件里内容正常，终端显示乱 | 重定向或管道时 Python 回退到 `locale.getpreferredencoding()`，不再用宽字符控制台 API |
| 逻辑层 | 数据本身真的损坏（罕见） | 文件被错误编码重写、字符串里混入错误引号等 |

**判断方法（最重要、最快）**：不要用眼睛读乱码下结论，用无歧义的输出来判断——

```python
# 一律用 ascii() 做无歧义判定：它对 str 和 list/dict 都会转义所有非 ASCII 字符
print(ascii(some_string))        # '\u6b66\u5668...'  —— 看到转义就是对的
print(ascii(some_list))          # "['\\u6b66\\u5668', ...]"  —— 容器同样安全
# 注意：repr() 不转义可打印的 CJK（repr('武器') == "'武器'"），不适合做这个判定
```

- `ascii()` 输出正确（`\uXXXX` 都是规范码点）→ 数据层没问题，乱码只是显示层 → 去修输出编码，**不要改业务代码**
- `ascii()` 输出就是错的 → 才是真编码 bug，才轮得到查文件编码/解码

### 2. 标准排查流程（按顺序执行）

1. **隔离显示层**：把同样的输出用 `ascii()` 打印一遍。
2. **强制 UTF-8 再看**：
   - 环境变量：`$env:PYTHONUTF8=1`（PowerShell）或 `export PYTHONUTF8=1`
   - 命令行：`python -X utf8 script.py`
   - 控制台：先执行 `chcp 65001` 再跑程序
   - 不开 UTF-8 模式也可强制输出编码：`PYTHONIOENCODING=utf-8`（stdout/stderr 按 UTF-8 写，绕过 console 码页）
3. **重定向到文件再读**（绕开终端码页）：
   ```bash
   python script.py > out.txt 2>&1
   # 然后用 UTF-8 显式读取该文件（PowerShell: Get-Content out.txt -Encoding UTF8）
   ```
4. **对比磁盘字节**：怀疑文件本身时，读原始字节看规范 UTF-8 序列：
   ```python
   raw = open('file.py', 'rb').read()
   raw.find('目标文本'.encode('utf-8'))   # 用 bytes 搜索，别用 str
   ```
5. **检查两边码点**：两个字符串看起来一样但比较失败时，逐码点对比：
   ```python
   for i, (a, b) in enumerate(zip(s1, s2)):
       if a != b: print(i, ascii(a), ascii(b), hex(ord(a)), hex(ord(b)))
   ```

### 3. 常见陷阱与反模式

- ⚠️ `python -c "print('中文')"` 直接传中文：**仅 Linux/macOS** 上，非 UTF-8 模式下 Python 才用 `locale.getpreferredencoding()` 解码 `-c` 的源码串；**Windows 原生 Python 的 `-c` 来自 UTF-16 宽命令行，源码不会乱**（实测 cp936 + 非 UTF-8 模式下 `python -c "print(ascii('中文'))"` → `'\u4e2d\u6587'` 完好）。Windows 上 `-c` 传中文出现乱码，基本都在 stdout 显示层，按第一部分排查即可，别去怀疑源码解码。写脚本文件执行仍是跨平台最稳做法。
- ❌ 用 `cat`/`echo` 在 git-bash 里写含中文的行并拼接：码页不同结果不同。
- ✅ 要可靠写入含中文的行，用 `printf '%s\n' '中文' > out.txt`（git-bash 默认按 UTF-8 写字节），必要时 `iconv -f <src> -t utf-8` 显式转码，别靠终端码页猜。
- ❌ 看见乱码就清 `__pycache__`、怀疑文件编码损坏 → 先做第 1 步（ascii 判定），90% 是白折腾。
- ❌ 在 Windows 控制台输出里用肉眼核对中文字符串是否相等。
- ✅ 调试探针写在断言**旁边**（同一次调用），并同时打印 `ascii(needle)` 和 `ascii(haystack)`——
  曾经踩过的坑：探针放在测试方法开头、断言在结尾，结果两边数据"看起来都对"，真相是 `assertIn` 对 list 做的是元素等值而非子串匹配（`"武器大师" in ["武器大师: ..."]` 为 False）。
  这类是**逻辑层**陷阱（与编码/环境无关）：确认对象是 list 后直接回业务代码改断言，不要再在编码或环境上耗。
- ✅ 给用户的程序可加启动提示：`chcp 65001` 或 `PYTHONUTF8=1`，避免用户环境同样踩乱码。

### 4. 本机/通用速查

| 操作 | 命令 |
|---|---|
| 切换控制台码页 | `chcp 65001`（回退 936） |
| Python 强制 UTF-8 模式 | `python -X utf8` 或 `PYTHONUTF8=1` |
| 只强制 stdout 编码 | `PYTHONIOENCODING=utf-8`（不开 UTF-8 模式也可用） |
| 无歧义调试输出 | `print(ascii(x))`（`repr()` 不转义 CJK，勿用） |
| 读文件字节 | `open(p,'rb').read()` + `bytes.find(...)` |
| 源文件解码异常 | Python 3 默认 UTF-8；`# -*- coding: gbk -*-` 只对旧项目 |

## 第二部分：沙箱 / 受限环境坑排查

### 1. 识别"环境层异常"的启发式

异常栈**不在你的业务代码里**，而在下列位置 → 极可能是环境层问题，**不要往自己代码上查**：

- `tempfile` 的 `_resetperms` / `TemporaryDirectory.__exit__` 里的 `PermissionError: [WinError 5]`
- `shutil.rmtree` 的 `onerror`/`onexc` 回调
- OS 层：`[WinError 5] 拒绝访问`、`[WinError 32]` 文件占用
- 工具会话层：`shell write failed; the shell was reset`、`[exit code: 1]` 且无业务 traceback

典型场景（真实踩过）：Windows 沙箱/受限 token 下，`tempfile.TemporaryDirectory` **创建**没问题，**清理时**要 `chmod(0o700)` 重置权限 → 被拒绝 → 整个测试报 PermissionError，误以为测试逻辑挂了。

### 2. 标准处理流程

1. **定位异常归属**：完整看 traceback，最后几帧是业务代码还是框架/环境代码？
   - 框架清理代码 → 环境坑，绕开或忽略，别改业务
2. **绕开 tempfile 清理**：改用项目内本地目录 + 手动清理：
   ```python
   work = os.path.join(os.path.dirname(os.path.abspath(__file__)), "_tmp")
   os.makedirs(work, exist_ok=True)
   try:
       ...  # 测试用 work
   finally:
       shutil.rmtree(work, ignore_errors=True)   # ignore_errors 兜底
   ```
   或捕获清理异常：`with contextlib.suppress(PermissionError): tmp.cleanup()`
   注：`os.path.dirname(__file__)` 所在目录可能只读，`makedirs` 会失败；优先选工作区内已确认可写的临时目录。
3. **工具会话故障不硬刚**：bash 会话报 `shell reset` / `write failed` 时，换 pwsh 工具继续；命令被杀后 `[exit code: 1]` 无信号标记 = 中断不是失败。
4. **区分"测试失败"与"环境噪音"**：`Ran 28 tests ... FAILED (failures=1, errors=2)` 里 errors 可能全是环境噪音，先看 failures；清掉环境噪音后重跑确认。

### 3. 反模式

- ❌ 环境异常出现后反复审查业务代码、重构无关逻辑。
- ❌ `tempfile.TemporaryDirectory` 在受限环境反复重试（每次都会在清理时失败）。
- ❌ 把 `rmtree` 的权限错误当成数据损坏去排查。

## 结尾检查清单

排查"诡异失败"时按顺序过一遍：

- [ ] 输出看了 ascii() 版本吗？（显示层已隔离？）
- [ ] PYTHONUTF8=1 / -X utf8 / chcp 65001 试过了？
- [ ] 断言对象是 list 还是 str？（list 的 in 是元素等值！——逻辑层陷阱，确认后回业务代码，别在编码上耗）
- [ ] traceback 最后一帧在业务代码还是框架/环境代码？
- [ ] 环境类异常（PermissionError/壳重置）已用本地目录/换工具绕开？