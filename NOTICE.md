# NOTICE · 归属与改动声明

本仓库是 **[FuRongJun-1999/dsh-memory](https://github.com/FuRongJun-1999/dsh-memory)（灵枢 LINGSHU）** 的衍生作品（derivative work）。本文件说明归属关系与本仓库的具体改动，以满足 MIT 许可的署名要求，并让使用者清楚知道自己在用什么。

---

## 1. 上游项目

| 项 | 值 |
|---|---|
| 项目名 | 灵枢 LINGSHU（`dsh-memory`） |
| 仓库 | <https://github.com/FuRongJun-1999/dsh-memory> |
| 作者 | 符荣峻（FuRongJun-1999） |
| 许可 | MIT License |
| 版权声明 | `Copyright (c) 2026 FuRongJun-1999` |
| 本仓库基线提交 | `4169c5bdb90f71f5bba99b56c4b99650472ee52c`（`v0.8.1-opt.2` 起；`opt.1` 时为 `958097d0559585be544cd16323a3a96eca29a461`） |

**原始版权归 FuRongJun-1999 所有。** 本仓库的 `LICENSE` 文件**原样保留**上游的 MIT 许可全文与版权声明，未作任何修改。

---

## 2. 本仓库的改动

本仓库在上游基线之上做了 **11 轮系统性源码优化**，全部集中在 **3 个文件**，共计 **+769 / −57 行**：

| 文件 | 增 | 删 |
|---|---:|---:|
| `md_cg/mdcg.py` | 660 | 49 |
| `md_cg/mdcos.py` | 92 | 7 |
| `md_cg/readcache.py` | 17 | 1 |
| **合计** | **769** | **57** |

**未修改任何测试文件。**

### 改动性质

| 轮次 | 类别 | 摘要 |
|---|---|---|
| R1–R4, R7, R8 | 性能优化 | 目录级 realpath 缓存 · 正文旁路缓存 · 目录条目表 · 扫描复用缓存 · 缓存健壮性上限 · 冷启动并发读盘 |
| R5 | 安全修补 | 回收站路径穿越闸门（对应上游 issue #172） |
| R6 | 安全修补 | 证伪裁决改为可恢复摘除，不再硬删（对应上游 issue #144） |
| R9 | 缺陷修复 | 修掉 R4 引入的「新正文被旧缓存顶掉」真 bug + 2 处工程纪律违规 |
| R10 | 性能 + 缺陷修复 | 生产类 `MdCGSecure` 真冷启动「读两遍」（2N 次 open）归零 |
| R11 | 缺陷修复 | `direct_read` 未穿透缓存（治理面会读到陈旧正文并静默覆盖他人修正） |

完整的目标 / 改动 / 效果对比见 **[docs/fork/十一轮源码优化报告.md](docs/fork/十一轮源码优化报告.md)**。

### v0.8.1-opt.2 修订（2026-10-13，同步上游 `4169c5bd`）

同步上游 **54 个提交 / 81 个文件**后，对上述改动做了**诚实修订**：

| 项 | 处置 |
|---|---|
| **R6**（证伪裁决硬删不可恢复） | **一半让位、一半保留**——让位：上游已自行修复 #144，其「走 `_forget_apply` 单点 + 纯 `MdCG` 兜底绝不硬删」比本衍生版的 `_retire_path` 更彻底（后者兜底会退回 `os.remove`），故删除 `_retire_path`（已无调用点）。保留：上游的 `verify(falsified)` 直到 `_forget_apply` 内部才碰路径，**副作用（`add_rejected`）已先产生**；本衍生版在 `add_rejected` **之前**的 `_node_disk_path` 闸（越界即 `falsified_blocked`、零副作用）上游没有，**予以保留** |
| **R5**（路径穿越） | **完整保留，覆盖面是上游的超集**——上游 #172 只修了 `restore()` **入口**（`_restore_id_ok`）；`_forget_apply` 的 `src = os.path.join(self.root, e["path"])` 与 `dst = os.path.join(self.trash_dir, f"{node_id}.md")` **两处至今仍是裸拼**。本衍生版的 `_trash_path` 单点同时覆盖 `restore` 的 `src` 与 `_forget_apply` 的 `dst`，并**保留**上游的 `_restore_id_ok` 作纵深；`_forget_apply` 的 `src` 另走 `_node_disk_path`（P2-20）。即上游两条裸拼路径本衍生版都堵了。**这是本衍生版仍然有效的独立发现** |
| `md_cg/mdcg.py` | 保留本衍生版的「枚举 → 并发读盘 → 串行解析」三遍结构；把上游新增的**密级值域判据**移植进解析遍 |
| `package.json` | 顺带订正上游遗留缺陷：`prepare` 脚本误伤 `npm pack --dry-run`，致上游 `publish-artifact-check` 门禁恒红（详见 §8.3） |

改动文件的最终集合见下节。

### 改动许可

本仓库的改动部分同样以 **MIT License** 发布（见 `LICENSE`）。

---

## 3. 新增文件

以下文件由本仓库新增，**不属于上游项目**：

| 文件 | 说明 |
|---|---|
| `README.md` 顶部门户段 | 衍生版说明、快速上手、目录结构、FAQ、许可 |
| `NOTICE.md` | 本文件 |
| `CONTRIBUTING.md` | 贡献指南 |
| `CHANGELOG.md` | 变更记录 |
| `SECURITY.md` | 安全策略 |
| `CODE_OF_CONDUCT.md` | 行为准则 |
| `.editorconfig` | 编辑器统一配置 |
| `.github/ISSUE_TEMPLATE/` | Issue 模板 |
| `.github/PULL_REQUEST_TEMPLATE.md` | PR 模板 |
| `docs/fork/` | 本衍生版专属文档 |

---

## 4. 上游 README 的保留

上游 README 全文**原样保留**在本仓库 `README.md` 的**后半部分**（位于「上游原版完整技术文档」标题之后），未作任何删改。其中包含的评测数据、架构说明、Issue 编号、Release 说明与链接，**均指向上游项目**。

---

## 5. 不包含的内容

本仓库**不声称**以下内容：

- 不声称对上游项目拥有任何所有权或著作权；
- 不声称代表上游作者的观点或立场；
- 不声称本仓库是上游的官方版本。上游的**最新版本与完整功能**请以 [上游仓库](https://github.com/FuRongJun-1999/dsh-memory) 为准；
- 不声称本衍生版的优化已被上游接受或合并。

---

## 6. 报告问题

- **本衍生版引入的改动**相关的问题 → 提到[本仓库 Issues](https://github.com/regdiliKUN/lingshu/issues)
- **上游本身**相关的问题 → 提到[上游仓库 Issues](https://github.com/FuRongJun-1999/dsh-memory/issues)

---

## 7. 第三方组件

| 组件 | 许可 | 说明 |
|---|---|---|
| DeepSeek Harness | MIT | DeepSeek 官方开源项目，与本项目**无隶属关系** |
| 上游 dsh-memory | MIT | 见上文第 1 节 |

本仓库其余部分为纯 Python 标准库实现，**无第三方运行时依赖**。

---

## 8. 本衍生版发现的上游缺陷（如实记录）

在为本仓库配置 CI 与同步上游的过程中，跑出了**四个属于上游、不是本衍生版引入**的问题。
两个已在本仓库订正，一个本仓库修不了（因为它在上游**已发布的 npm 产物**里），一个**刻意保留红灯**。
**四个都已上报上游**，附可复现证据：

| # | 缺陷 | 本仓能否修 | 上游 issue |
|---|---|---|---|
| 8.1 | `install-verify` 的 `npm ls` 诊断步必然杀整步 | ✅ 已订正 | [#223](https://github.com/FuRongJun-1999/dsh-memory/issues/223) |
| 8.2 | npm 包干净安装后无法 import（optional peer 被静态导入） | ❌ 修不了（在已发布 tarball 里） | [#222](https://github.com/FuRongJun-1999/dsh-memory/issues/222) |
| 8.3 | `prepare` 误伤 `npm pack --dry-run`，致上游发版门禁恒红 | ✅ 已订正 | [#221](https://github.com/FuRongJun-1999/dsh-memory/issues/221) |
| 8.4 | `python-tests` 3 项恒红：#179 判据误报 + 两项 Windows-only 断言 | 📌 **刻意保留红灯**（修它需改测试文件，违背本仓「零测试改动」承诺） | [#234](https://github.com/FuRongJun-1999/dsh-memory/issues/234) |

### 8.1 已订正：`install-verify` 的 `npm ls` 诊断步必然杀整步

> 上游已上报：[#223](https://github.com/FuRongJun-1999/dsh-memory/issues/223)

`.github/workflows/install-verify.yml` 的 `Consumer clean install` 步最后一行
`npm ls @deepseek-ai/dsh-llm @deepseek-ai/cordis` 检查的是**本就没装的 peer 包**，
`npm ls` 对缺失包返回非零；而 GitHub 的 shell 带 `-e -o pipefail`，于是这条**预期为空**的
诊断把整步报成 `##[error]Process completed with exit code 1`——而同一段日志里 npm 其实
已经 `added 1 package, and audited 2 packages in 7s`。**6 条矩阵腿（3 平台 × 2 Node）全部**
挂在这一行。上游已给同文件另一个诊断步加了 `continue-on-error: true`，唯独漏了这条。
本仓库补了 `|| true`（与上游对其余诊断步的处置一致）。

### 8.2 **未订正**：npm 包 `@furongjun1999/dsh-memory@0.8.1` 干净安装后无法 import

> 上游已上报：[#222](https://github.com/FuRongJun-1999/dsh-memory/issues/222)

`install-verify` 的 `ESM import check` 步**正确地报出了一个真实缺陷**（本仓库**不修改**这条检查
——把它改成通过等于藏掉一个真缺陷）：

```
ESM FAIL: ERR_MODULE_NOT_FOUND
Cannot find package '@deepseek-ai/schemastery'
  imported from .../node_modules/@furongjun1999/dsh-memory/lib/index.js
```

**根因**：`@deepseek-ai/schemastery` 在 `package.json` 里被声明为
`peerDependencies` + `peerDependenciesMeta: {optional: true}`，且 `dependencies` 为空；
但入口 `src/index.ts` 对它做的是**静态顶层导入**（`import z from '@deepseek-ai/schemastery'`）。
npm 7+ **不会**自动安装标记为 optional 的 peer 依赖 ⇒ 消费者执行干净的
`npm install @furongjun1999/dsh-memory` 后，**任何** import 都会失败。

**为什么本仓库修不了**：缺陷位于**上游已发布的 npm 产物**中。本仓库不发布 npm 包，
改本仓库的 `package.json` 不会影响 registry 上那个 tarball。

**给用户的绕过办法**（本仓库 README 的 FAQ 也写了）：

```bash
npm install @furongjun1999/dsh-memory @deepseek-ai/schemastery
```

**为什么值得单独记一笔**：这条检查属于上游自己的「issue #10 用户视角装机复现矩阵」——
它设计出来就是为了抓这种事，而它**确实抓到了**。本仓库选择保留这条红灯并如实标注，
而不是让它变绿。

### 8.3 已订正：`prepare` 脚本误伤 `npm pack --dry-run`，致上游发版门禁恒红

> 上游已上报：[#221](https://github.com/FuRongJun-1999/dsh-memory/issues/221)

同步上游 `4169c5bd` 后，本仓库的 `publish-artifact-check` 转红。追下去发现是**上游自己的既有红**：
**上游 `main` 连续 7 次 `publish-artifact-check` 全部 failure**（含最新提交 `4169c5bd`），
CI 日志与本机复现**逐字一致**：

```
=== 本地模式 ===
  执行：npm pack --dry-run --json
  [环境错误] npm pack --dry-run 失败 rc=1
> @furongjun1999/dsh-memory@0.8.1 prepare
[prepare] 本安装会得到一个不可用的包（issue #161）：…
##[error]Process completed with exit code 2.
```

**根因**：上游 issue #161 的「不可用包」检查（`prepare` 脚本）在 `typescript` 未安装且
`lib/index.js` 不存在时**主动 `exit 1`**——这个检查对**安装**场景是对的。但
`scripts/check_publish_artifact.py` 是靠 `npm pack --dry-run --json` 取发布清单的，
而 `npm pack` 会触发 `prepare` 生命周期脚本 ⇒ 门禁取不到清单，以 `exit 2`（fail-closed）恒红。
**上游工作流注释写的「`npm pack --dry-run` 不需要安装依赖」已经不成立。**

**修法**：在 `prepare` 的**报错分支之前**插入 dry-run 分支：

```js
else if (process.env.npm_config_dry_run === 'true') {
  console.log('[prepare] dry-run：跳过构建——npm pack --dry-run 只取清单、不安装任何东西，' +
              '故 issue #161 的「不可用包」检查不适用')
}
```

判据 `npm_config_dry_run` 为**实测所得**：npm 在 `npm pack --dry-run` 时会设
`npm_config_dry_run=true`，在 `npm pack`（非 dry-run）时不设。

**保护未被削弱（实测确认）**：

| 场景 | 修前 | 修后 |
|---|---|---|
| `npm pack --dry-run`（门禁取清单） | `exit 1` ⇒ 门禁 `exit 2` 恒红 | **`exit 0`**，门禁 R1–R7 全绿 |
| `npm run prepare`（真实安装路径） | `exit 1`（报 #161） | **`exit 1`（报 #161，逐字不变）** |

### 8.4 **刻意保留红灯**：`python-tests` 在 `4169c5bd` 上有 3 项恒红（全部继承自上游）

> 上游已上报：[#234](https://github.com/FuRongJun-1999/dsh-memory/issues/234)

`v0.8.1-opt.2` 同步上游后，本仓 `python-tests` 转红。**这不是本衍生版引入的**——上游同 SHA
`4169c5bd` 的 CI 是**同一份失败清单**（`462/465 通过 · 7 跳过`，run `38401645090`）：

```
失败：md_cg.test_issue211_export_target,
      md_cg.whitebox_kb.wisdom.test_ecosystem_assembly,
      scripts.test_check_guard_coverage
```

三项根因各不相同，且**都是恒红**（不是偶发）：

| # | 目标 | 根因 | 本机 Windows |
|---|---|---|---|
| ① | `scripts.test_check_guard_coverage` | `#179` 新增的反向判据用正则扫 `package.json` 的 `scripts` **值全文**，把 `prepare` 报错文案里的构建产物 `lib/index.js` 当成「被点名但盘上不存在的脚本」。`lib/` 被 gitignore、由 `npm run build` 产出 ⇒ 裸 clone 必然不存在 ⇒ 恒红。**判据作用域过宽，不是漏登记** | 同样失败（rc=4） |
| ② | `md_cg.whitebox_kb.wisdom.test_ecosystem_assembly` | 测试里硬编码 Windows 分隔符 `io.open(f'{TOOLS}\\{fn}')`，POSIX 上 `\\` 是真实文件名字符 ⇒ `FileNotFoundError`。**它到现在才红是因为上游 #185 把 md_cg 收集面从非递归改成递归**——子目录测试此前根本不在收集面内（既不是红也不是绿，是**不存在**） | **PASS** |
| ③ | `md_cg.test_issue211_export_target` 的 U10 | 该断言的前提是「末段 `aux` 命中 Windows 保留设备名 ⇒ `datapath.aux_root()` 解析失败」，POSIX 上 `aux` 只是普通目录名 ⇒ 断言**不可能成立** | **PASS** |

**为什么刻意不修**：②③ 的修法是**改测试文件**（`os.path.join` / 平台前置 SKIP），而本衍生版对外的
承诺是「**未修改任何测试文件**」——这条承诺的价值高于「让徽章变绿」。① 的修法是改守卫判据，
但即便修掉它，②③ 仍会让这条流水线恒红，**买不到绿**。

**因此本仓不把 `run-tests` 设为分支保护的必需检查**：一条恒红的检查作为必需条件，
会让「满足条件」成为不可能，等于把门禁变成墙。

> 方法论：**关掉/绕过一条恒红的检查，与把它修好，是两件事。** 本仓选择第三条路——
> **保留红灯 + 逐项根因 + 上报上游**，并明确标注「这条流水线当前不携带信号」。
> 这比「把 3 项失败写成 `continue-on-error`」诚实，也比「默默不管」有用。

---

_Last updated: 2026-10-13_
