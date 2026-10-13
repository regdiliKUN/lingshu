# 变更记录

本项目是 [FuRongJun-1999/dsh-memory](https://github.com/FuRongJun-1999/dsh-memory) 的**优化衍生版**。
本文件记录**本仓库相对上游基线的改动**。上游自身的完整变更历史请见
[上游 Releases](https://github.com/FuRongJun-1999/dsh-memory/releases)。

格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)。
版本号沿用上游版本，附加衍生版后缀 `-opt.N`。

---

## [0.8.1-opt.1] — 2026-10-13

**基于上游提交** `958097d0559585be544cd16323a3a96eca29a461`（上游 v0.8.1）。

改动规模：**优化主体 3 个文件，+769 / −57 行**；发布后按 CI 反馈追加 2 个文件（配置登记面 + 其生成件）。
**未修改任何测试文件。**

完整的目标 / 改动 / 效果对比见 [docs/fork/十一轮源码优化报告.md](docs/fork/十一轮源码优化报告.md)。

### 性能

> 核心洞察：这套系统的**算法层不慢，慢的全是系统调用**。2000 个文件逐次 `open()+read` 需 2751 ms，
> 而一次 `os.scandir` 目录枚举把全部 size/mtime/属性免费带回只需 3.9 ms（**52×**）。
> 因此优化主线只有一条：**把「逐文件系统调用」换成「一次目录枚举 + 一次落盘快照」**。

实测（2000 节点库，min-of-N，同会话串行）：

| 指标 | 上游原始 | 本衍生版 | 提升 |
|---|---:|---:|---|
| 首次冷启动 | 1681.4 ms | 537.1 ms | **3.1×** |
| 快照丢失后恢复装载 | 1618.8 ms | 73.1 ms | **22.1×** |
| 热启动后首次检索 | 878.0 ms | 123.1 ms | **7.1×** |
| 路径解析段 | 169.8 ms | 6.1 ms | **28×** |
| 全量重建索引 | 1594.8 ms | 79.7 ms | **20×** |
| 首次检索 `.md` open 次数 | 2000 | 0 | 归零 |
| 路径解析 `realpath` 次数 | 2000 | 3 | 归零 |
| 热启动 / 稳态检索 / 单节点读 | 20.5 / 17.3 / 0.1 ms | 21.9 / 19.6 / 0.2 ms | 持平（本机噪声内） |

- **R1** 目录级 realpath 缓存 —— 消除 O(N) 符号链接解析（`pathpass_realpath` 2000 → 3）
- **R2** 正文旁路缓存 `_nodecache.json` —— 首次检索零 open。**独立文件**，不污染 `_index.json`（保证两次重建字节一致）
- **R3** 目录条目表 —— 一次 `os.scandir` 免去逐节点 `stat` / `islink`；TTL 1 s 收敛外部改写的新鲜度窗口
- **R4** 扫描复用旁路缓存 —— 冷启动也免 open（`recover_init` 2032.5 ms → 80.9 ms）
- **R7** 旁路缓存健壮性 —— 9 种坏档形态全部自愈；50000 条目 / 64 MiB / 128 MiB 三重上限
- **R8** 冷启动并发读盘 —— 三遍扫描 + 线程池（w=8 时 1814.8 ms → 498.8 ms，3.6×）
- **R10** 生产类 `MdCGSecure` 双读归零 —— 真冷启动 `open` 次数 2000 → 1000，560.6 ms → 228.4 ms；
  热启动 `open` 次数 4000 → 0，2464.3 ms → 106.8 ms

### 修复

- **R9** 修复 R4 引入的**真实缺陷**：`consolidate` 一类写方直写文件 + `rebuild_index()`，**不经脏标记**，
  导致扫描从 TTL 目录表拿到**写前的旧 stat**，命中旧旁路缓存后**新正文被旧正文顶掉**。
  改用 `_iter_md_files()` 自枚举（遍历序与 `os.walk(topdown=True, followlinks=False)` 逐位一致），
  身份与名字同源同代。
- **R9** 修复 2 处工程纪律违规：旁路缓存 I/O 曾放在契约要求「零 `OSError` 捕获」的 `readcache.py`（已迁至 `mdcg.py`）；
  `save_sidecar` 对共享字典裸迭代（已改取快照）。
- **R11** 修复 `direct_read` 的**接线缺陷**：该函数是治理面「写前重查 / 回滚比对」的唯一真源，
  但实际**没有**穿透旁路缓存——会读到装载时的旧正文做读-改-写，**静默覆盖**另一实例刚落盘的修正。
  该缺陷被掩盖了 9 轮。修法：合并两处赋值，真源优先 `_read_from_disk`。

### 修复（发布后 · 由 CI 发现）

- **补齐优化引入的 4 个环境变量的登记面**。CI 的 `python-tests` 在首次推送后报红：
  `scripts.test_check_config_env_registry` 的判据 C1（代码读取面 ⊆ 登记面）出现 4 项新缺口 ——
  `MDCG_DIRSCAN`、`MDCG_DIRSCAN_TTL`、`MDCG_NODE_SIDECAR`、`MDCG_SCAN_WORKERS`。
  这四项是本衍生版为「目录条目表 / 节点旁路缓存 / 扫描并发」新引入的开关，代码读取点齐备但
  **未进登记面**，故反向对账守卫（issue #120）转红。
  > 诚实标注：这是**本次优化自己引入的缺陷**，且是发布之后才被 CI 抓到的。根因是优化期间
  > 只跑了 `md_cg` 组套件（327 目标），**漏跑 `scripts` 组**（41 目标，全仓 455 目标）——
  > 而这条守卫恰好住在 `scripts/` 组里。红基线是可见的，所以能被自愈闭环接住；此处如实记录
  > 而不隐藏，并保留「新 env 必须同步登记」这条工程纪律的强制力。
- 修法遵循本仓既有先例（issue #120 的 22 项补登记）：在 `md_cg/config_registry_bulk.py` 的
  `ENV_CATEGORY_ROWS`（只登记分类与值域类型、不登记缺省值）补 4 行，并同步 `META.env_category_count`
  90 → 94；随后重生成派生的 `docs/reference/配置参数表_v0.1.md`（守卫 G3 要求生成件与真源同步）。
- 验证：`python -m scripts.test_check_config_env_registry` → **通过 20 / 失败 0**，
  登记面 721 → 725，C1 缺口 4 → **0**；`md_cg.config_registry_guard` → **PASS 7 / FAIL 0**（含 G3 新鲜度）。

### 修复（上游遗留 · 本衍生版顺带订正）

- **`.github/workflows/install-verify.yml` 的 `npm ls` 诊断步永远杀整步**。该步最后一行
  `npm ls @deepseek-ai/dsh-llm @deepseek-ai/cordis` 检查的是**本就没装的 peer 包**，
  `npm ls` 对缺失包返回非零；而 GitHub 的 shell 带 `-e -o pipefail`，于是这条**预期为空**的
  诊断让整个 step 报 `##[error]Process completed with exit code 1` —— 而同一段日志里
  npm 其实已经 `added 1 package, and audited 2 packages in 7s`。6 条矩阵腿（3 平台 × 2 Node）
  **全部**挂在这一行。上游已给另一个诊断步加了 `continue-on-error: true`，唯独漏了这一条。
  修法：补 `|| true`（与上游对该文件其余诊断步的处置一致）。
  > 诚实标注：这是**上游工作流的遗留缺陷**，不是本衍生版引入的；本衍生版基线
  > `958097d` 原样带着它。因为该工作流只在 `workflow_dispatch` 或改它自身时触发，
  > 普通推送不会让它变红——但一个**不可能通过**的工作流留在仓库里是负债，故顺手订正。

### 安全

- **R5** 新增 `_trash_path(node_id)` 单点闸门（对应上游 issue #172）：`restore` / `_forget_apply` 的
  `node_id` 是调用方可控入参（MCP 工具面直达），此前可越出回收站目录。现统一做字面闸（复用 `_NODE_ID_RE`）
  + realpath 必须落在 `trash_dir` 内。
- **R6** 证伪裁决改为**可恢复摘除**（对应上游 issue #144）：`cg(op=verify, verdict=falsified)` 此前会硬删除记忆，
  绕过 `forget` 的 admin 闸与回收站。现改为经回收站的 `_retire_path`，语义可恢复。
  > 诚实标注：经复核，「任意文件删除」这一说法**不成立**；真实缺陷是**不可恢复性**。

### 文档

- 新增衍生版门户段（README 顶部）：项目用途、核心功能、技术特点、使用场景、快速上手、使用示例、目录结构、FAQ、许可与归属
- 新增 `NOTICE.md`（归属与改动声明）、`CONTRIBUTING.md`、`CHANGELOG.md`、`SECURITY.md`、`CODE_OF_CONDUCT.md`
- 新增 `.github/ISSUE_TEMPLATE/` 与 `PULL_REQUEST_TEMPLATE.md`
- 新增 `docs/fork/` —— 十一轮优化报告与深度分析报告

### 验证

- **结果等价性**：优化全开 vs 全关，8 组查询 ×（id, score, 资格状态）、recall id 序列、
  400 个索引键、400 个内容哈希 —— **逐位一致**
- **枚举面等价性**：`_iter_md_files` vs `os.walk` —— 文件集合一致、**遍历序逐位一致**（含符号链接与坏链接）
- **全量测试**：`317/327 通过 · 3 跳过 · 10 失败`；10 项**全部落在上游基线失败集合内** ⇒ **新增失败 0**
- **未修改任何测试文件**

### 已知边界（如实标注）

1. 所有性能读数基于**合成 2000 节点库**，万级 / 十万级规模未实测。
2. 本机墙钟噪声 ±15~30%（Windows 实时防护对 `open` 的拦截开销波动），故以**系统调用计数**为主证。
3. 读路径存在 ≤1 s 的新鲜度窗口（`MDCG_DIRSCAN_TTL`，默认 1.0 s），可用 `=0` 完全退回基线口径。
   扫描路径**已无此窗口**。
4. `_nodecache.json` 只在 `close()` 时落盘；强杀进程不落盘，退化为下次冷启动。
5. 本衍生版基线为上游 `958097d`，**不包含此后上游的新提交**。

---

## 上游历史

上游的完整变更历史（v0.5.0 起逐版说明）见
[上游 README 的 Release 说明段](https://github.com/FuRongJun-1999/dsh-memory#-文档导航)
与 [上游 Releases](https://github.com/FuRongJun-1999/dsh-memory/releases)。
