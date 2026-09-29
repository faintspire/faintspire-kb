# Git 历史大文件清理（GitHub GH001）实战

> 场景：`git push` 传完几十 MB 后被服务端整包拒绝：
> `remote: error: File xxx.log is 233.95 MB; this exceeds GitHub's file size limit of 100.00 MB`
> `! refs/heads/dev [remote rejected] (pre-receive hook declined)`

## 1. 三个必须先想明白的事实

1. GitHub 单文件上限 100 MB，且是**整包校验**：包里有一个超标对象，整个 push 被拒；
2. `.gitignore` 和 `git rm --cached` 只管**未来**，历史提交里的 blob 依然在，照样被拒；
3. push 只传输**被推引用可达**的对象 —— 所以 stash / 远程跟踪引用里藏着旧大对象时，不影响推送，但会影响本地 gc 瘦身。

## 2. 清理流程（filter-branch 版，零依赖）

1. 备份：`git clone --mirror . ../repo-history-backup.git`；
2. 工作区必须干净，否则 filter-branch 拒绝执行：`git stash push --include-untracked`；
3. 剔除历史中的目标路径（PowerShell 下命令串用**单引号**，双引号会被拆词报 `bad revision`）：
   `git filter-branch --force --index-filter 'git rm -r --cached --ignore-unmatch a/logs b/logs' --prune-empty --tag-name-filter cat -- --branches --tags`
4. 删除改写备份引用（不要用管道喂 `update-ref --stdin`，PowerShell 管道转 UTF-16 会报 `expected SP`）：
   循环 `git for-each-ref --format='%(refname)' refs/original` 逐个 `git update-ref -d`；
5. 删除旧远程跟踪引用：`git for-each-ref --format='%(refname)' refs/remotes/<old>` 逐个删除；
6. 回收空间的关键一步 —— reflog 的**不可达条目**默认保留 30 天，必须显式过期：
   `git reflog expire --expire=now --expire-unreachable=now --all`
   然后 `git repack -a -d` + `git prune --expire=now`；
7. 恢复工作区：`git stash pop`；
8. 推送：新远程普通 push；旧远程历史已改写，需 force push（先确认无协作者依赖旧历史）。

## 3. 排查「gc 之后还是很大」的顺序

1. `git rev-list --objects --all | Select-String <文件名>` —— 有输出说明仍被分支/标签引用；
2. `git rev-list --objects --reflog --all` —— 有输出说明 reflog 未过期（补 `--expire-unreachable=now`）；
3. `git for-each-ref` 检查 `refs/original`、`refs/remotes/*`、`refs/stash` 三类残留；
4. 用 `git verify-pack -v <pack.idx>` 列出最大 blob 反查路径，确认目标真的已清除。

## 4. 预防

- 项目初始化就把 `logs/`、`*.log`、`target/`、`*.zip`、`*.jar`（除 wrapper）写进 `.gitignore`；
- 大资产（模型、数据集、视频）不进 Git，需要版本化用 Git LFS 或对象存储；
- CI 里加一道检查：`git rev-list --objects --all` 结合对象大小扫描，超标即失败。
