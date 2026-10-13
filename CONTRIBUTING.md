# 贡献指南

感谢你有兴趣为本项目做贡献。本项目是 [FuRongJun-1999/dsh-memory](https://github.com/FuRongJun-1999/dsh-memory) 的**优化衍生版**，因此贡献流程有两个来源需要注意。

---

## 目录

- [先读这个](#先读这个)
- [我想报告问题](#我想报告问题)
- [我想提交代码](#我想提交代码)
- [工程约定（重要）](#工程约定重要)
- [开发环境](#开发环境)
- [跑测试](#跑测试)
- [与上游同步](#与上游同步)
- [提交信息规范](#提交信息规范)

---

## 先读这个

**问题该提给谁？**

| 问题类型 | 提到哪 |
|---|---|
| 本仓库的 11 轮优化引入的行为变化 | [本仓库](https://github.com/regdiliKUN/lingshu/issues) |
| 上游本身的功能、缺陷、评测数据 | [上游仓库](https://github.com/FuRongJun-1999/dsh-memory/issues) |
| 不确定 | 先提本仓库，我们会帮你判断 |

**提 PR 前请先开 Issue 讨论。** 这个项目的改动门槛较高（见下方工程约定），未经讨论的 PR 很可能因为方向不符而被关闭，白费你的时间。

---

## 我想报告问题

请使用 [Issue 模板](.github/ISSUE_TEMPLATE/)，并尽量附上：

1. **环境**：操作系统、Python 版本、宿主（DSH / Claude Code / …）
2. **复现步骤**：最小可复现的命令或配置
3. **实际行为** vs **期望行为**
4. **日志**：宿主日志、`python -m md_cg.mcp_server --show-config` 输出
5. **是否用了本衍生版的优化**：如果怀疑与优化有关，请附 `MDCG_READ_CACHE=0 MDCG_DIRSCAN=0 MDCG_NODE_SIDECAR=0` 下的对比结果

> ⚠️ **不要在 Issue 里贴记忆库的真实内容**。如果需要，请用脱敏后的最小样例。

---

## 我想提交代码

### 流程

```bash
# 1. Fork 本仓库，然后克隆
git clone https://github.com/<你的用户名>/lingshu.git
cd lingshu

# 2. 从 main 开分支
git checkout -b fix/描述你的改动

# 3. 改代码 → 跑测试（见下）
# 4. 提交 → 推送 → 开 PR
```

### PR 要求

- **一个 PR 只做一件事**。混入无关改动会被要求拆分。
- **必须附验证证据**。本项目重视「能红的守卫测试」——即你的改动如果被回退，应该有测试会失败。
- **PR 描述里写清楚**：改了什么、为什么改、怎么验证的。

---

## 工程约定（重要）

本项目继承了上游的一套较严格的工程纪律。以下几条是**硬性要求**，违反会被要求修改：

### 1. 不要为了通过测试而修改测试

如果改动导致测试失败，**修实现，不修判据**。本项目历史上所有回归都是通过改实现解决的。

### 2. 不要裸迭代共享字典

```python
for k, v in cg.index["nodes"].items():   # ❌ 并发下会 RuntimeError
for k, v in list(cg.index["nodes"].items()):   # ✅ 先取快照
```

仓库里有静态守卫会扫描这类模式。

### 3. 每个 `def` 上方要有一行生效条件注释

```python
# 生效条件：<什么情况下这个函数会被走到>
def my_function(...):
    ...
```

这是本仓库的既有约定，守卫会检查。

### 4. 读失败必须 fail-closed

读不到的节点**不得**参与检索，**不得**泄漏半读内容。不要用缓存快照顶替「读不到就不给」的语义。

### 5. 不要引入第三方 Python 依赖

`md_cg/` 是**纯标准库**实现。这是项目的核心设计约束，不是偶然。

### 6. 一切文本按 UTF-8 处理

新增代码里 `open()` / `read_text()` **一律显式 `encoding="utf-8"`**，不吃 locale 缺省（中文 Windows 缺省是 cp936）。

### 7. 不要提交测试产物

`_md_cg_p*/`、`data/mdcg/`、`__pycache__/` 等已在 `.gitignore` 中，请勿强制添加。

---

## 开发环境

```bash
# Python 侧：无依赖，直接用系统 Python 3.12+
python --version

# Node 侧（仅插件形态需要）
npm install          # NODE_ENV=production 时须加 --include=dev
npm run build        # TypeScript 编译

# Rust 侧（仅跑全量测试需要）
cd rust && cargo build --release
cd ../hive && cargo build --release
```

---

## 跑测试

### Python 全量套件

```bash
# 必须从仓库根运行，必须用 python -m
python -m md_cg.run_tests md_cg --jobs 4

# 先看有哪些用例
python -m md_cg.run_tests md_cg --list

# 定向跑
python -m md_cg.run_tests md_cg -k <关键字>

# 串行（默认并发 4；排查并发假红时用）
python -m md_cg.run_tests md_cg --jobs 1
```

> ⚠️ **不要和其它重负载任务并行跑**。套件会吃满 CPU，会把慢用例顶成超时假红。
> 逐用例日志在 `%TEMP%\md_cg_testlogs\`（Windows）。

### Node 侧集成测试

```bash
npm test
```

### 本衍生版的优化验证脚本

位于 `docs/fork/`，用于验证优化前后结果等价：

```bash
# 结果等价性对拍（需要上游基线副本作为对照）
# 详见 docs/fork/十一轮源码优化报告.md 附录 A
```

---

## 与上游同步

```bash
git remote add upstream https://github.com/FuRongJun-1999/dsh-memory.git
git fetch upstream
git log --oneline HEAD..upstream/main      # 看看上游有哪些新提交

# 合并（注意：本仓库的优化改动可能与上游冲突）
git merge upstream/main
```

> **冲突提示**：本衍生版的改动集中在 `md_cg/mdcg.py`、`md_cg/mdcos.py`、`md_cg/readcache.py`。
> 上游若改动同一区域，冲突大概率出现在这几处。合并后请重跑
> `python -m md_cg.run_tests md_cg -k <相关用例>` 确认行为未被破坏。

---

## 提交信息规范

沿用上游风格，建议格式：

```
<类型>(<范围>): <简短描述>

<可选：详细说明>
```

常用类型：`fix` · `feat` · `perf` · `docs` · `test` · `refactor` · `chore`

示例：

```
perf(md_cg): 目录条目表消除逐节点 stat/islink

用一次 os.scandir 带回全部 size/mtime，替代逐路径 os.stat。
2000 节点下 pathpass 段 169.8ms → 6.1ms。
结果等价性对拍逐位一致。
```

---

## 行为准则

参与本项目即表示你同意遵守 [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md)。
