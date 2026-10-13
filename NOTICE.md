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
| 本仓库基线提交 | `958097d0559585be544cd16323a3a96eca29a461` |

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

在为本仓库配置 CI 的过程中，跑出了两个**属于上游、不是本衍生版引入**的问题。一个已在本仓库订正，一个本仓库修不了（因为它在上游**已发布的 npm 产物**里）。

### 8.1 已订正：`install-verify` 的 `npm ls` 诊断步必然杀整步

`.github/workflows/install-verify.yml` 的 `Consumer clean install` 步最后一行
`npm ls @deepseek-ai/dsh-llm @deepseek-ai/cordis` 检查的是**本就没装的 peer 包**，
`npm ls` 对缺失包返回非零；而 GitHub 的 shell 带 `-e -o pipefail`，于是这条**预期为空**的
诊断把整步报成 `##[error]Process completed with exit code 1`——而同一段日志里 npm 其实
已经 `added 1 package, and audited 2 packages in 7s`。**6 条矩阵腿（3 平台 × 2 Node）全部**
挂在这一行。上游已给同文件另一个诊断步加了 `continue-on-error: true`，唯独漏了这条。
本仓库补了 `|| true`（与上游对其余诊断步的处置一致）。

### 8.2 **未订正**：npm 包 `@furongjun1999/dsh-memory@0.8.1` 干净安装后无法 import

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

---

_Last updated: 2026-10-13_
