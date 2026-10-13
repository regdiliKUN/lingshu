# 变更记录

本项目是 [FuRongJun-1999/dsh-memory](https://github.com/FuRongJun-1999/dsh-memory) 的**优化衍生版**。
本文件记录**本仓库相对上游基线的改动**。上游自身的完整变更历史请见
[上游 Releases](https://github.com/FuRongJun-1999/dsh-memory/releases)。

格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)。
版本号沿用上游版本，附加衍生版后缀 `-opt.N`。

---

## [0.8.1-opt.2] — 2026-10-13

**基于上游提交** `4169c5bdb90f71f5bba99b56c4b99650472ee52c`（上游 `main`，
比 `opt.1` 的基线 `958097d` **前进 54 个提交 / 81 个文件**）。

本版是**同步上游 + 对既有改动做诚实修订**，不新增优化轮次。

### 同步上游（54 个提交）

冲突 5 处，全部解决：

| 文件 | 冲突性质 | 处置 |
|---|---|---|
| `md_cg/mdcg.py` | **真实语义冲突 2 处** | 保留本衍生版的「枚举 → 并发读盘 → 串行解析」三遍结构；把上游新增的**密级值域判据**移植进解析遍 |
| `README.md` | COGMAP 自动生成段 2 处 | **不手工解**，交由 `scripts/cogmap_sync.py build` 重生成（含链接前缀与行号重算） |
| `WORKSPACE_INDEX.md` · `docs/mdcg/功能调用映射表_v0.1.md` · `docs/reference/配置参数表_v0.1.md` | 生成件 | 分别用 `workspace_index.py --write` / `cogmap_sync.py build` / `gen_config_doc.py` 重新生成 |

自动合并成功的文件包括 `md_cg/mdcos.py`、`md_cg/config_registry_bulk.py`、`scripts/workspace_index.py`。

### R5 / R6 的处置（诚实修订）

**上游已自行修复我们 R5/R6 针对的两个问题**（issue #172 与 #144 均已 CLOSED），
且上游的修法在关键处**更好**。故本版对这两轮做如下修订（**结论已逐行核对合并后的代码**）：

| 轮次 | 上游处置 | 本衍生版处置 |
|---|---|---|
| **R6**（#144：证伪裁决一次调用即硬删，不可恢复） | 已修：改走 `_forget_apply` **单点**——与 `forget` 同一条可恢复路径（`trash/` + 删除清单 + `_unstage` + 遗忘审计），纯 `MdCG` 兜底也**绝不退化成硬删**；并额外修掉「证伪正文前 200 字以 internal 明文落盘」的泄露 | **一半让位、一半保留**。让位部分：上游方案更彻底（本衍生版原方案在「回收站搬不动」时会退回 `os.remove`）⇒ 删除本衍生版的 `_retire_path`（已无调用点）。保留部分：上游的 `verify(falsified)` 从 `protect.guard_forget` **直接跳到 `add_rejected`**，直到 `_forget_apply` 内部才碰路径 ⇒ 路径越界时**副作用已产生**；本衍生版在 `add_rejected` **之前**的 `_node_disk_path` 闸（越界即 `falsified_blocked`、零副作用）上游没有，**予以保留** |
| **R5**（#172：`restore()` 路径穿越） | 只修了 `restore()` **入口**（`_restore_id_ok`，与 `add` 的 P0-1 白名单同源、先拦后读、零 IO）；`_forget_apply` 的 `src` / `dst` **两处至今仍是裸拼** | **完整保留，覆盖面是上游的超集**：`_trash_path` 单点同时覆盖 `restore` 的 `src` 与 `_forget_apply` 的 `dst`（并**保留**上游的 `_restore_id_ok` 作纵深）；`_forget_apply` 的 `src` 另走读面同源的 `_node_disk_path`（P2-20）。即上游两条裸拼路径本衍生版都堵了（见下节） |

### 仍然有效的独立发现（本衍生版独有）

- **`_forget_apply` 的 `src` / `dst` 双裸拼 —— 上游 #172 未覆盖的那一半。**
  上游 #172 的正文明确写着：「`add()` 有 P0-1 白名单，**`forget()` 走索引检查**，唯独
  `restore()` 漏网」——即上游认为 forget 面是安全的。但这个前提不成立：`forget` 的
  `node_id` 取自**索引**，而索引 id 直接抄盘上 frontmatter（`_scan_nodes` 只做 `str` 归一、
  **不校验形状**）⇒ 手改一个 `.md` 的 `id:` 字段即可污染它，使 `os.replace(src, dst)` 的
  **目标**落在 `trash/` 之外 = **任意路径覆写**。上游 `main`（`4169c5bd`）至今仍是
  `src = os.path.join(self.root, e["path"])` 与 `dst = os.path.join(self.trash_dir, f"{node_id}.md")`
  **两条都是裸拼、无闸**。
  本衍生版在 `dst` 处保留单点闸 `_trash_path`（字面闸 + realpath 闸，越界即结构化负路由、
  零 IO）；`src` 处走读面同源的 `_node_disk_path`（P2-20）。**关掉 issue ≠ 修全了面。**

### 修复（上游遗留 · 本衍生版顺带订正）

- **`package.json` 的 `prepare` 脚本误伤 `npm pack --dry-run`，致上游发版门禁
  `publish-artifact-check` 恒红。** 上游 issue #161 的「不可用包」检查在 `typescript`
  未安装且 `lib/index.js` 不存在时主动 `exit 1`；而 `scripts/check_publish_artifact.py`
  靠 `npm pack --dry-run --json` 取发布清单，`npm pack` 会触发 `prepare` ⇒ 门禁取不到清单，
  以 `exit 2`（fail-closed）恒红。
  **实测上游 `main` 连续 7 次 `publish-artifact-check` 全部 failure**（含最新 `4169c5bd`），
  CI 日志与本机复现**逐字一致**。
  修法：在 `prepare` 的**报错分支之前**插入 dry-run 分支，判据
  `process.env.npm_config_dry_run === 'true'`（实测 npm 在 `npm pack --dry-run` 时会设该变量）。
  **#161 对真实安装的保护逐位不变**——非 dry-run 时仍 `exit 1`（实测确认）。
  **已上报上游 [#221](https://github.com/FuRongJun-1999/dsh-memory/issues/221)**。

### 验证

- 守卫链全绿：`cogmap` / `workspace-index`（顶层目录 21 · 追踪 2487 件）/ `check_local_paths` /
  `check_publish_artifact`（R1–R7）/ `config_registry_guard`（PASS 7 / FAIL 0）/
  `test_check_config_env_registry`（20/20）/ `verify_discipline`（10/10）
- **本机定向验证 23/23 通过**（覆盖手工解冲突所涉的全部代码路径）：`test_issue146_frontmatter_quarantine`
  （移植的密级值域判据）、`test_p28_refcheck`、`test_p9_forget_protect`、
  `test_n130_verify_falsified_protect`、`test_n209_verify_write_face_gates`、`test_p39_verify_flow`、
  `test_lifecycle_retire_leak`、上游新增的 `test_issue197/211/214_*`、上游改动的
  `test_export_snapshots` / `test_i50c_meta_passthrough` / `test_interop_judgment` /
  `test_issue130_*` / `test_issue91_*` / `test_p47_*` / `test_recall_face_guards` / `test_tool_face`。
  另全量 `md_cg` 组按字母序跑到 `d` 的 45 项亦全通过（全量串行约需 1 小时，权威读数以 CI 为准）。
- **`md_cg` 目标数 327 → 341**：上游 issue #185 把收集面从**非递归**改成**递归**，
  14 个子目录测试**首次进入收集面**（此前既不是红也不是绿，是**不存在**）。
- **CI 状态变化（如实记录）**：`cogmap` / `workspace-index` / `publish-artifact` /
  `conformance` / `hive` / `discipline` 全绿；**`python-tests` 由绿转红**——3 项失败，
  **全部继承自上游**（上游同 SHA `4169c5bd` 的 CI 是同一份清单：`462/465 通过 · 7 跳过`）。
  逐项根因与处置见 `NOTICE.md §8.4`；已上报上游
  [#234](https://github.com/FuRongJun-1999/dsh-memory/issues/234)。
  **刻意不修**（②③ 的修法要改测试文件，违背「零测试改动」承诺；只修①买不到绿）。
- **未修改任何测试文件**（上游带来的新测试文件随合并进入）

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
  修法：补 `|| true`（与上游对该文件其余诊断步的处置一致）。订正后该步 6/6 通过。
  **已上报上游 [#223](https://github.com/FuRongJun-1999/dsh-memory/issues/223)**。
  > 诚实标注：这是**上游工作流的遗留缺陷**，不是本衍生版引入的；本衍生版基线
  > `958097d` 原样带着它。因为该工作流只在 `workflow_dispatch` 或改它自身时触发，
  > 普通推送不会让它变红——但一个**不可能通过**的工作流留在仓库里是负债，故顺手订正。

### 已知边界（上游缺陷 · 本衍生版无法修复）

- **npm 包 `@furongjun1999/dsh-memory@0.8.1` 干净安装后无法 import**。`install-verify` 的
  `ESM import check` 步报出：

  ```
  ESM FAIL: ERR_MODULE_NOT_FOUND  Cannot find package '@deepseek-ai/schemastery'
    imported from .../node_modules/@furongjun1999/dsh-memory/lib/index.js
  ```

  根因：`@deepseek-ai/schemastery` 是 `optional: true` 的 peer 依赖（`dependencies` 为空），
  而入口 `src/index.ts` 对它做**静态顶层导入** ⇒ npm 7+ 不自动装 optional peer
  ⇒ 消费者干净安装后**任何** import 都失败。
  **本仓库刻意不把这条检查改成通过**——那等于藏掉一个真缺陷。该缺陷位于**上游已发布的
  npm 产物**中，本仓库不发布 npm 包，改本仓库 `package.json` 无法影响 registry 上的 tarball。
  用户绕过办法：`npm install @furongjun1999/dsh-memory @deepseek-ai/schemastery`
  （README 的 FAQ 已收录）。详见 `NOTICE.md` §8。
  **已上报上游 [#222](https://github.com/FuRongJun-1999/dsh-memory/issues/222)**。

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
   > 〔opt.2 修订〕本项已在 `[0.8.1-opt.2]` 中解决：基线已前进到上游 `4169c5bd`。保留此条是为了让读者看到当时的真实边界。

---

## 上游历史

上游的完整变更历史（v0.5.0 起逐版说明）见
[上游 README 的 Release 说明段](https://github.com/FuRongJun-1999/dsh-memory#-文档导航)
与 [上游 Releases](https://github.com/FuRongJun-1999/dsh-memory/releases)。
