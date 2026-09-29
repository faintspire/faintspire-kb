# Faintspire 知识库

个人**公共**知识库 · 远程：`github.com/faintspire/faintspire-kb`（Public）· 分支 `main`

根目录只保留本 README 与 `.gitignore`，全部内容收纳在 `doc/` 下。

## 内容结构

| 路径 | 内容 |
|---|---|
| `doc/GitHub操作文档.md` | GitHub 全操作手册：登录 / 建库 / 拉取 / 推送 / IDEA Token / 外观 / Pages / 删除 / 退出 / 大文件清理 |
| `doc/SSH密钥使用文档.md` | SSH 密钥原理与 GitHub、部署服务器双侧配置，含 SSH over 443 实战 |
| `doc/XXL-JOB执行器对接文档.md` | 统一调度中心执行器接入规范（Spring Boot 3.x / JDK 17+），敏感值已脱敏 |
| `doc/notes/` | 单篇通用技术笔记 |
| `doc/images/` | 自绘无水印 SVG 示意图（界面字段依据 github.com / docs.github.com 真实页面） |

## 文档索引

| 文档 | 主题 |
|---|---|
| [doc/GitHub操作文档.md](doc/GitHub操作文档.md) | GitHub 全流程操作手册（10 章 + 21 条 FAQ） |
| [doc/SSH密钥使用文档.md](doc/SSH密钥使用文档.md) | SSH 密钥：原理、GitHub / 服务器配置、SSH over 443 |
| [doc/XXL-JOB执行器对接文档.md](doc/XXL-JOB执行器对接文档.md) | XXL-JOB 执行器接入与存量任务迁移 |
| [doc/notes/git-history-large-file-cleanup.md](doc/notes/git-history-large-file-cleanup.md) | Git 历史大文件清理（GH001）实战笔记 |

## 收录原则

1. 只放可公开内容：通用技术笔记、排障记录、工具用法；公司接口、拓扑、账号、客户信息一律不进；
2. 图片**仅限自绘无水印 SVG 示意图**；公司截图、业务架构图、客户数据不进本库；
3. 公司业务文档（接口 / 部署 / 架构 / 技术选型）在公司私有仓库 `faintspire/sangerbox-server` 的 `doc/` 目录，不进本库；
4. 敏感值（token / 密码 / 内网 IP）出现即脱敏为占位符；已在聊天或截图出现过的秘密值视为泄露，需轮换；
5. 图示优先编号步骤与表格（禁 mermaid / ASCII 图）。

## 与公司仓库的关系（嵌套独立仓库）

1. 本目录物理嵌套在公司仓库工作区内，但**自身是独立 Git 仓库**（自有 `.git`、独立 `main` 分支）；
2. 公司仓库 `.gitignore` 以 `/faintspire-kb/` 隔离，父库 `git status` / `git add .` 完全看不见本目录；
3. ⚠️ 切勿在公司库执行 `git add faintspire-kb`：嵌套仓库会被登记为 gitlink(160000)，内部文件全部脱离父库跟踪；
4. 本目录内的 commit / push 只作用于本库，公司库提交 likewise 不碰本目录。

## 发布与在线阅读

- 推送：在 `faintspire-kb/` 目录内 `git add . && git commit && git push`（SSH over 443）；
- 在线阅读：GitHub 网页直接浏览 `doc/` 下 Markdown 与 SVG；
- 可选站点化：日后如需 Docsify 站点，在**根目录**加 `index.html` + `_sidebar.md` 并开 Pages（源选 `main` + `/ (root)`），站点地址 `https://faintspire.github.io/faintspire-kb/`。
