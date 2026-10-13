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

_Last updated: 2026-10-13_
