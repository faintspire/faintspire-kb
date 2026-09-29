# Faintspire 知识库

个人**公共**知识库 · 远程：`github.com/faintspire/faintspire-kb`（Public）· 分支 `main`

## 内容结构

| 路径 | 内容 |
|---|---|
| `GitHub操作文档.md` | GitHub 全操作手册：建库 / 拉取 / 推送 / IDEA Token / 外观 / Pages / 删除 / 退出 / 大文件清理，配 `images/` 无水印 SVG |
| `SSH密钥使用文档.md` | SSH 密钥原理与 GitHub、部署服务器双侧配置，含 SSH over 443 实战 |
| `notes/` | 单篇通用技术笔记（每篇独立文件，登记到本 README 索引） |
| `public-site/` | SangerBox 对外公共文档站源（Docsify：index.html / README / _sidebar / XXL-JOB 对接文档），可选开 Pages |
| `images/` | 自绘无水印 SVG 示意图（界面字段依据 github.com / docs.github.com 真实页面） |

## 笔记索引

| 笔记 | 主题 |
|---|---|
| [notes/git-history-large-file-cleanup.md](notes/git-history-large-file-cleanup.md) | Git 历史大文件清理（GitHub GH001）实战 |

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

- 推送：`git push -u origin main`（SSH over 443，见《SSH密钥使用文档》第 6 章）；
- 可选 Pages：将 `public-site/` 重命名为 `docs/` 后，在 Settings → Pages 选 `main` + `/docs`，即得 `https://faintspire.github.io/faintspire-kb/` 在线文档站。
