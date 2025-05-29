# git

目前网络栈尚不完善，因此只测试了git的基础功能，包括创建仓库，提交，历史，分支等功能。

```bash
git config --global core.pager ""
cd test
git init
echo "123" > a.txt
git add .
git diff --staged --color
git status --color
git config user.name dragon
git config user.email undefined@os
git commit -m "add a"
git log
git show --color
git checkout -b new_branch
mkdir foo
echo "new file" > foo/bar
git status --color
vi a.txt
git diff --color
git add . && git commit -m "new commit"
git branch -vv --color
git log --color
git checkout master
git merge new_branch
git log --color
```
