# git 补课

## 第一次把本地项目创建为仓库

```bash
git init
git branch -M main # 把本地的当前分支改名为 main

git add .
git commit -m "first commit"

git remote add origin git@github.com:用户名/仓库名.git # 绑定远程 GitHub 仓库
# origin 不是固定语法关键字，但它是 Git 默认习惯用名。大多数项目都把“主要远程仓库”叫 origin。

git push -u origin main # 第一次推送，并把本地 main 分支和远程 origin/main 关联起来。之后就可以直接用 git push 了。origin是远程仓库的名字，main是这个远程仓库里的分支名字。
```

## git status

查看当前状态。这个命令非常重要，Git 学不会的时候，第一反应就敲它。它会告诉你哪些文件被修改了、哪些文件已经加入暂存区、当前在哪个分支、有没有东西可以提交。

```bash
git status
```

例如，结果可能为：

```bash
PS D:\notes> git status
On branch main
Your branch is up to date with 'origin/main'.

Changes to be committed: # 已经add，待commit的修改
  (use "git restore --staged <file>..." to unstage)
        new file:   "git\350\241\245\350\257\276.md"
        modified:   "wsl\351\203\250\347\275\262px4\347\216\257\345\242\203.md"

Changes not staged for commit: # 还没add的修改
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   "git\350\241\245\350\257\276.md"
```

## git pull


```bash
git pull # 从远程仓库拉取最新提交，并合并到当前本地分支
git fetch # 把远程最新信息拿回来，但不会修改当前文件，执行后本地的 origin/main 会更新，但本地 main 不一定变化
git merge # 合并到当前分支
```
git pull = git fetch + git merge

git pull 会先 fetch，再把远程对应分支合并到当前所在的本地分支。
如果当前在 main，并且 main 绑定 origin/main，那么 pull 更新的就是本地 main。

## branch基础

### 查看分支

```bash
PS D:\notes> git branch # 本地分支
* main
PS D:\notes> git branch -r # 远端分支
  origin/main
PS D:\notes> git branch -a # 所有分支
* main
  remotes/origin/main
```

### 创建分支

```bash
git branch dev # 创建分支，只创建但不切换
git switch dev # 切换分支
git switch -c dev # 创建并且切换分支
```

### 合并分支

```bash
git switch -c test
# 然后自己修改一些文件
git add .
git commit -m "测试新功能"
git switch main
git merge test
```

## git clone

执行clone命令之后，会自动做这些事情：

```
1. 下载仓库文件
2. 下载历史提交记录
3. 自动创建并切换到远程仓库的默认分支，通常是 main，也可能是 master
4. 自动绑定远程仓库为 origin
5. 自动建立 main 和 origin/main 的关联
```

## git log

```bash
PS D:\yyx_blog> git log --oneline # 一次显示一行
6c002c1 (HEAD -> main, origin/main, origin/HEAD) 更新跑步小记模块
9f47b72 更新毕业祭和两篇学习博客笔记
e7326e8 更新租房博客
35fddf0 修改笔记错误内容
4d60ba2 更新博客
7805ac7 更新SPF论文笔记
e37b1ea 更新博客和笔记
...
```

按q退出日志查看。

## git remote

### 查看远程仓库

`git remote` 专门用来管理“远程仓库地址”。你可以把它理解成：给本地 Git 仓库登记几个远程服务器地址，比如 GitHub、Gitee、GitLab。

```bash
PS D:\yyx_blog> git remote # 只看远程仓库名字
origin
PS D:\yyx_blog> git remote -v # 详细信息，fetch和push的地址
origin  git@github.com:salt235/yyx_blog.git (fetch)
origin  git@github.com:salt235/yyx_blog.git (push)
```

### 添加远程仓库

```bash
git remote add origin git@github.com:用户名/仓库名.git
```

如果同时添加github和gitee仓库，那就不能叫origin了。

```power
git remote add github git@github.com:用户名/仓库名.git
git remote add gitee git@gitee.com:用户名/仓库名.git
```

## git restore

`git restore` 是用来“恢复文件内容”的命令。它主要解决一个问题：文件改乱了，想把它恢复到某个 Git 记录里的状态。它是比较新的 Git 命令，用来替代以前 `git checkout -- 文件名` 那种容易混淆的写法。

如果已经add的文件，用：

```bash
git restore --staged 文件名
# 会取消add的暂存区提交，但是文件本身不会变化
```

如果还没add，只是修改了，用：

```bash
git restore 文件名
# 前提是这个文件被git跟踪过，已经add或者commit过
# 文件会之前回退到上一次提交
```

## branch进阶

### 本地新建分支dev并推到 GitHub

```bash
git switch main
git pull
git switch -c dev
git push -u origin dev
```

### GitHub 上已有分支，本地拉下来

```bash
# 如果远程多了一个dev分支
git fetch origin
git branch -r
git switch dev # 会自动创建本地dev并绑定远程dev

git branch -vv # 可以看到各个分支的绑定关系
```

更稳的写法：

```bash
git fetch origin
git switch -c dev --track origin/dev
```

### 只clone某个分支

```bash
git clone -b <branch-name> --single-branch <repo-url>
```

## 冲突

### 远程比本地新，push 被拒绝，不一定是真冲突

远程有你本地没有的提交，Git 不允许你直接覆盖远程历史。

```j
      B  GitHub/main（已经提交远程的修改B）
     /
A
     \
      C  设备B/main （还是在A的基础上修改得到的C）
```

这时候需要先：

```
git pull --no-rebase
git push
```

如果两个设备没有改同一文件，或者没有改同一文件的相同区域，会自动合并：

```
      B
     / \
A       M
     \ /
      C
```

如果有冲突，那就要去手动解决。

### 遇到真冲突了

例如，pull之后可能，会提示手动处理冲突文件，这个时候打开文件，会发现变成这样：

```bash
<<<<<<< HEAD
Git 是用来管理代码历史的工具。 # 本地的内容
=======
Git 是一个分布式版本控制工具。 # 远程拉下来的内容
>>>>>>> origin/main
```

`<<<<<<< HEAD` 到 `=======` 之间，是你当前本地的内容。

`=======` 到 `>>>>>>> origin/main` 之间，是远程拉下来的内容。

手动更改之后提交就行。

### rebase和no-rebase

在 git pull 的语境下，--no-rebase 表示 fetch 后用 merge 合并；--rebase 表示 fetch 后用 rebase 把本地提交接到远程最新提交之后。

```bash
git pull --no-rebase
git pull --rebase
```

具体区别图示：

![image-20260703143608269](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/image-20260703143608269.png)

rebase情况下，三个绿色球的提交历史会被更改，生成一组 **新的 commit**。

## fork

如果clone了别人的仓库，自己又没有权限提交，那么可以用fork这个标准的开发流程：

```bash
1. fork (GitHub)
2. git clone your fork
3. git remote add upstream
4. git checkout -b feature/xxx
5. 修改代码
6. git add / commit
7. git push origin feature/xxx
8. GitHub 创建 PR
```

命令流程：

```bash
# 1. GitHub 网页 Fork 原仓库
# 原仓库：https://github.com/original/repo.git
# 你的 fork：https://github.com/yourname/repo.git

# 2. 克隆自己的 fork
git clone https://github.com/yourname/repo.git
cd repo

# 3. 添加原仓库为 upstream
git remote add upstream https://github.com/original/repo.git
git remote -v

# 4. 同步原仓库的main
git switch main
git pull upstream main
git push origin main
# 解释如下：
# upstream/main  →  本地 main  →  origin/main
# 原仓库 main        你电脑 main     你的 fork main

# 5. 新建开发分支
git switch -c feature/my-change

# 6. 修改代码后提交
git add .
git commit -m "feat: my change"

# 7. 推送到自己的 fork
git push -u origin feature/my-change

# 8. 去 GitHub 创建 PR
# base: original/main
# compare: yourname/feature/my-change

# 9. 如果 PR 后续还要修改
git add .
git commit -m "fix: update change"
git push
```

本地会有 main 和 feature/my-change 两个分支，main 用于对接原仓库，feature/my-change 用于自己开发。

## ssh 连接不上 github

如果遇到：

```
ssh -T git@github.com
告诉你
port 22: Connection timed out
```

那就是 22 端口连不上。推荐解决方案：把 GitHub SSH 改走 443 端口。

编辑 SSH 配置文件：

```bash
nano ~/.ssh/config
# 加入
Host github.com
  HostName ssh.github.com
  User git
  Port 443
```

之后就行了。

## commit message 的标准格式

例如：

```bash
# 模板, 一般scope可以省略
# scope 表示影响范围，可以是模块名、目录名或笔记主题，例如 git、px4、docs、controller。
git commit -m "<type>(scope): <简短描述>"
# 例子
git commit -m "feat(auth): add JWT login"
git commit -m "fix(px4): correct mavlink message parsing"
git commit -m "docs(git): add fork workflow notes"
```

具体的 type 类型：

```
feat     新功能
fix      修复 bug
docs     文档修改
style    代码格式修改，不影响逻辑，比如空格、缩进、格式化
refactor 重构代码，不是新增功能，也不是修 bug
perf     性能优化
test     添加或修改测试
chore    杂项维护，比如依赖更新、配置修改
build    构建系统相关修改
ci       CI/CD 配置修改
```

