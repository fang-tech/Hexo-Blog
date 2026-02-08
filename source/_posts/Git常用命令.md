---
trello_plugin_note_id: jP8_tawf_fC0qdA4MPNle
trello_board_card_id: 6885bb538544f0f410c4c8c5;6976d662d259550c93a6a507
---
```bash
git fetch upstream && \
git rebase upstream/main && \
git push origin HEAD
```

```bash
# 方法1: 搜索添加了特定字符串的 commit
git log -p --all -S 'jackson-datatype-jsr310' -- '*.xml'

# 方法2: 查看某个文件的修改历史
git log -p agentscope-core/pom.xml

# 方法3: 使用 blame 查看每一行是谁添加的
git blame agentscope-core/pom.xml | grep -A2 -B2 jsr310

# 方法4: 查看某个 commit 的详细信息
git show 5f195492
```

```bash
git fetch upstream && \
git checkout main && \
git rebase upstream/main && \
git push origin main
```

```bash
# 在当前 worktree
git stash push -m "sync changes"

# 在目标 worktree
cd /Users/tianyu.fang/agentscope-java
git stash apply stash@{0}  # 或者具体的 stash 名称
```

