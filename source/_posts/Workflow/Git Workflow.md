## pull branch
显示所有的远程的分支

```bash
git branch -r
```

选择远程分支: 拉取所有的远程分支和直接修改远程分支都不是最佳的实践, 这样会破坏仓库的独立性, 最好的做法是创建一个关联到远程分支的本地分支

```bash
git switch -c feature/login origin/feature/login
```
## commit & push 
> 在commit之前需要先获取所有的更新来同步在你创建这个分支到你commit之前这个branch发生的变更, 如果团队使用rebase来MR, 在commit后一般还需要rebase到origin/main

获取本地分支的最新远程更新

```bash
# 会删除不存在远程引用的本地分支, 也就是这个本地引用的远程分支如果被删除了, 这个本地分支也会被删除
git fetch -p 
```
### stage的最佳实践
#### 添加修改到stage
- **使用GUI** (尤其推荐vscode中的GUI, 能够精确看到stage区和work区, 更有利于细粒度的把控)
- **stage发生在: local阶段性的修改**

> 之所以并不是很推荐特别频繁的stage, 因为这样在有了新的修改以后能通过vscode的界面在change处看到自己在工作区中对比于上一次stage做了哪些修改. 能确定自己的修改的作用范围

在commit之前需要将需要commit的变更添加到stage区中

```Shell
git add . # 添加所有的change到stage区中
git add <file-path-relative-path> # 添加特定文件的变更到stage区中
```

上述的两种方式都并不推荐在实践中使用
- `git add .` -> 会将所有的change都stage上去, **无法精细地控制自己要stage哪些change**, 这是一件相对危险的事情, **在stage/commit之前, 一定要再三检查自己要stage/commit的内容**
- `git add <specific-file>` -> 往往要提交的修改里的文件都在不同的位置, 而这个命令要敲出文件的相对路径, 使用这个命令会让add变得相当繁琐, 而且相当困难

#### 移除在stage中的修改
- 同样**推荐GUI** (同样推荐vscode)

```shell
git reset HEAD <file>
```

> 将修改从stage中移除出来并不会对工作区有任何的变更, 这个过程一般发生在我们要控制自己提交的内容为工作区中change的子集的时候, 或者想要回滚/丢弃某个修改的时候

### commit的最佳实践
> 一般来说, 团队的仓库都是需要保证main分支的线性, 所以mr的时候一般都是rebase命令, 少有merge, 接下来的推荐的实践也都是基于这个基础之上 (如果不能理解这段话, 搜索"rebase保证提交历史线性")

- **个人更喜欢命令行, 但是GUI也挺好**. 命令行更能精细地把控commit的内容
1. 第一次的提交不携带选项, 提交信息是最后要在main分支展示的提交信息
2. 小的修改携带fix/squash option提交

```shell
git commit -m "<commit-message>"
```

- fixup commit

```shell
git commit --fixup=HEAD # 意思是这个commit是对HEAD(最新的commit)的一次fixup
```

- squash commit

```shell
git commit --squash=HEAD -m "<commit-message>"
```

fixup和squash commit之间的区别以及该在什么时候使用哪种
- 区别
	- `--fixup=HEAD` -> 指定为fixup的commit**不携带**message, git创建的默认的commit message是`fixup! <HEAD-commit-message>`会在`git rebase -i --autosquash`的时候被标记为`fixup!`类型
	- `--squash=HEAD -m "<commit-message>` -> 指定为squash的commit**携带**message, git创建的默认的commit message是`squash! <HEAD-commit-message> <squash-commit-message>`会在`git rebase -i --autosquash`的时候被标记为`squash!`类型
- 各自的适用情况
	- `fixup` -> 小修小补, 比如typo, 修log等, 合并到HEAD-commit中的时候不会携带message
	- `squash` -> 需要记录的/修改message的修改, 合并到HEAD-commit的时候会请求修改HEAD-commit的message, 来问你要如何合并squash-message和HEAD-message. 适用于需要补充message的非key-commit

- 不增加commit的提交
```bash
git commit --amend
```

这个commit会直接将修改合并到上一个commit, 并且不会产生message, 但是这种commit需要--force option 才能推送远端(因为这种时候push并不是添加commit, 而是要push重复的commit)

```bash
git push origin HEAD -f
```
### rebase最佳实践
在前面的基础上, 我们rebase的时候加上`--autosquash` option, 这个option会作用于前面我们commit的时候--fixup/suqash, 会将我们的commit标记为pick!/fixup!/squash!
最后rebase的时候, 只会保留pick-commit, fixup/squash都会合并到前面的最近的pick-commit上

```bash
git rebase -i --autosquash origin/main
```