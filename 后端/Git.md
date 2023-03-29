

#### 记录操作

##### 移除文件

```shell
git rm
```

##### 移动文件

```shell
git mv
```



#### 撤销操作

##### 覆盖commit

有时候我们提交完了才发现漏掉了几个文件没有添加，或者提交信息写错了。 此时，可以运行带有 `--amend` 选项的提交命令来重新提交：

```shell
$ git commit -m 'initial commit'
$ git add forgotten_file
$ git commit --amend
```

与其说是修复旧提交，倒不如说是完全用一个 **新的提交** 替换旧的提交

##### 取消暂存

取消add操作

```shell
git reset HEAD <file>...
```

##### 撤销文件修改

```shell
git checkout -- <file>...
```

Git 会用最近提交的版本覆盖掉它，对那个文件在本地的任何修改都会消失



#### 远程仓库

显示远程仓库的简写和对应URL

```shell
git remote -v
```

```shell
git remote add <shortname> <url>
```

使用 `clone` 命令克隆了一个仓库，命令会自动将其添加为远程仓库并默认以 “origin” 为简写

##### 推送到远程仓库

```shell
git push <remote> <branch>
git push origin master
```

当你和其他人在同一时间克隆，他们先推送到上游然后你再推送到上游，你的推送就会毫无疑问地被拒绝。 你必须先抓取他们的工作并将其合并进你的工作后才能推送

##### 查看远程仓库

```shell
git remote show origin
```



#### 标签

列出标签

```shell
git tag
```

##### 附注标签

```shell
git tag -a v1.4 -m "text" <sha-1>
```

```shell
git show v1.4
```

默认情况下，`git push` 命令并不会传送标签到远程仓库服务器上。 在创建完标签后你必须显式地推送标签到共享服务器上。

```shell
 git push origin --tags
```

使用带有 `--tags` 选项的 `git push` 命令。 这将会把所有不在远程仓库服务器上的标签全部传送到那里。



#### 分支

...

##### 推送

```shell
git push <remote> <local branch>:<remote branch>
```

##### 跟踪分支

```shell
git checkout --track <branch> <remote>/<branch>
```





#### rebase

```shell
git rebase <branch>
```

把branch指向的节点作为当前的基底