显示所有的远程的分支

```bash
git branch -r
```

获取本地分支的最新远程更新

```bash
# 会删除不存在远程引用的本地分支
git fetch -p 
```

选择远程分支: 拉取所有的远程分支和直接修改远程分支都不是最佳的实践, 这样会破坏仓库的独立性, 最好的做法是创建一个关联到远程分支的本地分支

```bash
git switch -c feature/login origin/feature/login
```

不增加commit的提交
```bash
git commit --amend
```

这种提交提交后需要强行推送才能推送远端

```bash
git push origin HEAD -f
```

rebase 修整分支

```bash
git rebase -i origin/dev
```

将需要合并的分支从pickup修改成squash/s