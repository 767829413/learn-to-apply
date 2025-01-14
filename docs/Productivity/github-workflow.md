# Git Feature Branch Workflow

Zion项目我们采用[Feature Branch Workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/feature-branch-workflow)，即每个特性在`branch`中开发，`master`始终保持稳定。特性开发完成，需提交`pull request`，接受其他成员的code review，同时可以在`PR`中围绕该特性进行讨论，`PR`记录了开发过程的细节。

由于是内部项目，我们没有使用`fork`机制，代码都维护在Github上的一个仓库：[apusic/zion](https://github.com/apusic/zion)。在看具体的流程前，先有一个全局视图：

![git](../../media/Pictures/git.png)

## 基本工作流程

* **从远程clone respository**

```bash
git clone https://github.com/apusic/zion.git
```

* **创建特性分支**

首先让本地的master处于最新状态：

```bash
git fetch
git checkout master
git rebase origin/master
```

创建分支

```bash
git checkout -b myfeature
```

* **在分支上进行开发**

```bash
git status # View the state of the repo
git add # Stage a file
git commit # Commit a file
```

进行一个功能特性开发时，可以多次提交到本地仓库`Repository`，不必每次`commit`都push到远程仓库`Remote`。

* **开发过程中，保持分支和最新代码同步**

```bash
# While on your myfeature branch.
git fetch
git rebase origin/master
```

关于`rebase`的详细说明，请参考 [The "Git Branching - Rebasing" chapter from the Pro Git book.](https://git-scm.com/book/en/v2/Git-Branching-Rebasing)

后面会单独介绍`rebase`冲突的处理。

* **将分支发布到中心仓库**

所有改动都提交后，执行：

```bash
git push -f origin myfeature
```

* **创建pull request**

访问[项目主页](https://github.com/apusic/Zion)，点击`Compare & pull request`创建`pull request`

* **code review & discussions**

可以要求一个或两个项目成员进行review，也可以围绕该特性进行讨论。

* **Merge pull requests**

根据项目的配置，`pull requests`在merge进master之前要满足一些条件，例如至少两个成员review，通过集成测试等。

所有的检查都通过后，这个`pull request`就可以merge了。详细的操作参见 [Merging a pull request](https://help.github.com/articles/merging-a-pull-request/)

这里有三个merge选型：`Merge pull request`，`Squash and merge`，`Rebase and merge`，关于它们的区别请参考[GitHub的帮助](https://help.github.com/articles/about-pull-request-merges/)。 一般建议选择`Squash and merge`。

至此，一个特性就开发完成了。

* **删除分支**

已经完成merged的 `pull requests` 关联的分支可以在GitHub删除，详细的操作步骤参见GitHub文档 [Deleting and restoring branches in a pull request](https://help.github.com/articles/deleting-and-restoring-branches-in-a-pull-request/)

## 冲突处理

上面的流程中，在保持分支和最新代码同步时，最有可能产生冲突。

`rebase`提示冲突，会列出冲突文件，执行下列步骤：

* 手工解决冲突
* `git add <some-file>` 将发生冲突的文件放回`index`区
* `git rebase --continue` 继续进行rebase
* 提示rebase成功

在`Merge pull requests`过程中也可能产生冲突，可以在GitHub的界面上解决冲突，详细的操作轻参考[Addressing merge conflicts](https://help.github.com/articles/addressing-merge-conflicts/)。

如果冲突较多，建议先在客户端执行rebase，按照上面的步骤解决完冲突，再进行`Merge pull requests`。

## 合并提交

合并提交，就是将多个 `commit` 合并为一个 `commit` 提交。建议你把新的 `commit` 合并到主干时，只保留 2~3 个 `commit` 记录。

在 `Git` 中，我们主要使用 `git rebase` 命令来合并。

```bash
$ git checkout -b feature/user
Switched to a new branch 'feature/user'

$ git log --oneline
7157e9e docs(docs): append test line 'update3' to README.md
5a26aa2 docs(docs): append test line 'update2' to README.md
55892fa docs(docs): append test line 'update1' to README.md
89651d4 docs(doc): add README.md

$ git log --oneline
4ee51d6 docs(user): update user/README.md
176ba5d docs(user): update user/README.md
5e829f8 docs(user): add README.md for user
f40929f feat(user): add delete user function
fc70a21 feat(user): add create user function
7157e9e docs(docs): append test line 'update3' to README.md
5a26aa2 docs(docs): append test line 'update2' to README.md
55892fa docs(docs): append test line 'update1' to README.md
89651d4 docs(doc): add README.md

$ git rebase -i 7157e9e
```

执行命令后，会进入到一个交互界面，在该界面中，可以将需要合并的 4 个 commit，都执行 squash 操作，如下图所示：

![img](https://pic.imgdb.cn/item/65d6dc189f345e8d031dba2f.webp)

修改完成后执行:wq 保存，会跳转到一个新的交互页面，在该页面，我们可以编辑 Commit Message，编辑后的内容如下图所示：

![img](https://pic.imgdb.cn/item/65d6dc739f345e8d031ebe19.webp)

**注意**

* `git rebase -i` 这里的一定要是需要合并 `commit` 中最旧 `commit` 的父 `commit ID`。

* 希望将 `feature/user` 分支的 5 个 `commit` 合并到一个 `commit`，在 `git rebase` 时，需要保证其中最新的一个 `commit` 是 `pick` 状态，这样我们才可以将其他 4 个 `commit` 合并进去。