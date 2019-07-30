---
title: shell git
---

常用 mac、git 命令行。

<!-- more -->

## gradle 打包

./gradlew assembleRelease
./gradlew assembleDebug

## mac 常用命令

ls 查看当前文件夹下的文件
pwd 查看当前路径
wq 退出文本浏览界面
cd ~ 回去根目录
cd .. 回到上级目录
-a  全部
-v  显示差异类容

## vim 简单使用

git status 查看 git 空间的状态
git diff 查看文件的差别
vim 终端编辑文件
esc 退出文件编辑状态
shift + : 进入命令状态
w 写文件
q 退出  一般 wq 连用

## 创建本地 git 仓库

mkdir WebApp      创建文件夹 WebApp
cd WebApp      进入文件夹
git init      初始化本地文件夹
touch README          创建文件 README
git add README 添加文件
git commit -m 'first commit'
git remote add origin git@github.com:daixu/WebApp.git	 增加一个远程服务器端

## git 使用流程

git status 
git diff
git add
git commit
git pull
git push

## branch 分支使用

### 查看分支

git fetch  同步远程信息
git branch -a  查看本地和远程所有分支

### 创建分支

git checkout -b lion  创建本地 lion 分支
git checkout -b local-branchname origin/remote_branchname  从远程分支，创建新分支
git branch lion master  从主分支 master 创建 lion 分支
git push origin lion  新分支推送到远程
git push --set-upstream origin lion  创建远程 lion 分支
git clone -b master https://github.com/dreaminglion/dreaminglion.github.io.git   克隆 master branch 到本地

### 删除分支

git push origin :lion  删除远程分支 lion（:和分支名称连写）同3，不同写法
git branch -D lion  删除本地分支 lion 
git push origin --delete <branchName>   删除远程分支branchName

### 切换分支

git checkout master  切换本地分支到 master

### 合并分支

在当前使用分支，合并 lion 分支过来。

git rebase lion
git merge lion
git pull origin lion

## git tag

git tag v1.5   生成 tag v1.5
git push origin v1.5  发送对应版本的 tag
git push origin --tags  把所有 tag 发到服务器

## git 回到以前版本：

git log  查看日志获取提交 commitId
git reset --hard  commitId 
git push origin HEAD --force  删除服务器上的比你多的版本
git reflog  查看按顺序的 commit 和 reset 的 id

## 删除已版本控制的文件

 git rm -r -n --cached "bin/"  -n 列出要被删除的文件
 git rm -r --cached  "bin/"    删除版本控制

## 重新设置远程分支

git remote set-url origin git@192.168.0.231:android/more-chat.git

## Command line instructions

### Git global setup

git config --global user.name "shi.li"
git config --global user.email "763258230@qq.com"

### Create a new repository

git clone git@192.168.0.231:shi.li/self-test.git
cd self-test
touch README.md
git add README.md
git commit -m "add README"
git push -u origin master

### Existing folder

cd existing_folder
git init
git remote add origin git@192.168.0.231:shi.li/self-test.git
git add .
git commit -m "Initial commit"
git push -u origin master

### Existing Git repository

cd existing_repo
git remote rename origin old-origin
git remote add origin git@192.168.0.231:shi.li/self-test.git
git push -u origin --all
git push -u origin --tags
