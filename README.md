# IP 智慧解答专家系统（文档仓库）

> 本仓库用于存放参赛项目的文档、数据集与演示资料，代码仓库另行维护。此 README 旨在帮助团队成员快速了解目录结构、协作规范与 Git LFS 使用方法。

## 目录结构
```
docs/                 （当前仓库根）
competition_docs/     赛事官方文件
whitepaper/           技术白皮书
system_design/        系统详细设计文档
datasets/             数据集（raw / processed / embeddings）
demo/                   演示视频、截图与 PPT
   ├─ video/             演示视频
   ├─ screenshots/       截图与动图
   └─ presentations/     路演 / 答辩 PPT
experiments/          试验笔记与结果
```

## Git Commit 提交规范
采用 **Conventional Commits** 风格，格式：
```
<type>(scope): <subject>

- <body>
```
- **type**：feat / fix / docs / style / refactor / perf / test / chore / build / ci / revert
- **scope**：可选，说明影响范围，如 `datasets`、`whitepaper`
- **subject**：一句话描述，不超过 50 字符，使用祈使句（如"添加了.../修改了..."）
- **body**：可选，说明修改原因、对比或参考等，用空行隔开


### type 含义速查
| type | 含义 | 典型场景 |
|------|------|-----------|
| feat | 新功能 | 新增模块、接口、脚本 |
| fix | Bug 修复 | 修正逻辑错误、边界条件 |
| docs | 文档 | 修改 README、设计文档 |
| style | 代码格式 | 空格、缩进、删除未用变量（不影响逻辑）|
| refactor | 代码重构 | 重组结构、优化设计（非功能/性能变更）|
| perf | 性能优化 | 提升速度、内存、I/O 效率 |
| test | 测试 | 新增/更新单元或集成测试 |
| chore | 杂务 | 更新依赖、脚本、配置文件 |
| build | 构建 | 修改构建流程、打包脚本 |
| ci | 持续集成 | 修改 GitHub Actions、Jenkins 配置 |
| revert | 回滚 | 撤销先前一次提交 |

> **建议**：`<subject>` 与 `<body>` 使用简体中文，保持沟通一致性；但 `<type>` 必须使用表格所示的英文小写关键词，以符合 Conventional Commits 规范。

中文示例：
```
fix(system_design): 修正时序图链接失效

- 时序图文件重命名导致 README 内部链接无法跳转，更新引用路径并测试通过。
```

## 文件命名规范
1. **英文小写 + 连字符 `-`**，避免空格与特殊字符。
2. 日期使用 `YYYYMMDD` ，如 `20250521-announcement.md`。
3. 多版本文件后缀 `v1`、`v2`，避免使用"final""new"。
4. 数据集文件统一使用下划线 `_` 分隔，如 `ospf_logs_cleaned.jsonl`。
5. 演示视频使用 `topic-version.ext`，如 `demo-overview-v1.mp4`。

> 备注：官方原始文件（如赛事通知、赛题 PDF、官方手册等）可直接使用原文件名，无需遵循上述命名规范。

## 启用 Git LFS
仓库已预置 `.gitattributes` 并追踪常见大文件：PDF、PPT、MP4、数据压缩包等。

启用步骤：
```bash
# 安装（仅首次）
# macOS (Homebrew)
brew install git-lfs

# Windows (任选其一)
choco install git-lfs             # Chocolatey
winget install --id GitHub.GitLFS -e # Winget
# 或从 https://git-lfs.github.com 下载安装包

# 初始化 LFS（安装后仅在仓库根执行一次）
git lfs install
```
查看 LFS 文件列表：
```bash
git lfs ls-files
```

如需追踪新类型文件：
```bash
git lfs track "*.ext"
```
提交 `.gitattributes` 即生效。

---
如对规范有疑问或需调整，请及时联系。