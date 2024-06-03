---
description: git
---

# Git

## git基本操作 <a href="#git-ji-ben-cao-zuo" id="git-ji-ben-cao-zuo"></a>

```git
## 克隆远程仓库到本地
git clone 远程仓库地址 (localDirectory)
# 如果默认有了dev和master分支，则会看到如下三个分支
# master[本地主分支] origin/master[远程主分支] origin/dev[远程开发分支]
# 新克隆下来的代码默认master和origin/master是关联的，也就是他们的代码保持同步
# 但是origin/dev分支在本地没有任何的关联，所以我们无法在那里开发

# 新建一个分支用于开发
git branch dev

# 切换到新建的分支
git checkout dev  # 通常配置快捷指令后  gco dev 即可

# 合并上两步操作
git checkout -b dev  # 相当于 branch + checkout


## 创建本地分支dev，并且和远程origin/dev分支关联，本地dev分支的初始代码和远程的dev分支代码一样
git checkout -b dev origin/dev  


## 添加文件到暂存区
git add .


##  将暂存区内容添加到仓库中。
git commit -m"XXX"
git commit


## 将暂存区内容推到origin上，提交merge request
git push origin 自己的开发分支
git push --set-upstream origin dev # 向远程仓库推送代码并新建远程分支并连接
# 第二和第一步的区别在于第二步会使本地dev和远程dev连接


# 本地分支和远程分支连接后，可以直接使用push pull来同步本地分支和远程分支的代码
git push # 将本地分支推到对应远程分支上
git pull # 拉取对应远程分支代码到本地
```

## 分支管理 <a href="#fen-zhi-guan-li" id="fen-zhi-guan-li"></a>

```git
## 删除分支 本地/远程
git branch -d dev # 如果分支commit过则需要改为-D
git push origin --delete dev


# 设置本地分支和远程分支的关联
git branch --set-upstream-to=origin/dev dev 


## 合并分支代码
git merge dev # 将dev分支的代码合到当前分支
git merge --squash 远程分支名 # 远程分支代码同步到本地当前分支上
# 取消合并
git merge --abort


## 更新本地git分支保持和远程分支一致
git remote update origin --prune

## 查看所有分支（包括远程）
git branch -a

## 查看本地分支和远程分支的对应情况
git branch -vv
```

## 仓库管理 <a href="#cang-ku-guan-li" id="cang-ku-guan-li"></a>

```git
## 初始化仓库
git init 


## 如果是本地项目推到远程仓库需要执行以下命令
git remote add origin 仓库地址


## 查看连接远程仓库地址
git remote -v

## 本地库设置个人信息 
git config --global user.name "你的姓名，最好由没有符合和空格的英文字母组成"
git config --global user.email <邮件名>@<邮箱服务商后缀>
```

## merge冲突解决 <a href="#merge-chong-tu-jie-jue" id="merge-chong-tu-jie-jue"></a>

```git
git checkout master

##2. 执行git pull --rebase命令，拉取master最新代码
git pull --rebase

git checkout 冲突的开发分支

##4. 执行git merge master命令，将master代码合并到你的开发分支，此时可能会有代码冲突，本地将代码冲突解决
git merge master

##5. 代码冲突解决完毕后，将代码push到你的远程开发分支
git push origin ...

##6. 重新创建merge request即可，如还有冲突，重复1-5步


# 解决冲突2
git checkout master
git pull
git merge --squash 原开发分支
git diff
git status
git checkout -b 新开发分支
# 从新开发分支提交mr
```

## git回退版本 <a href="#git-hui-tui-ban-ben" id="git-hui-tui-ban-ben"></a>

```sh
## 1.寻找需要回退到之前的版本号
git log

## 2.使用reset或者revert将版本回退
git reset --hard 版本号
git revert -n 版本号

## 3.强制push到对应的远程分支（如提交到develop分支）
git push -f -u origin develop

## 3.提交并推送远程分支。通知其他人更新代码
git commit -m XXX
git push

## 通过reset的方式，把head指针指向之前的某次提交，reset之后，后面的版本就找不到了

## revert不会把版本往前回退，而是生成一个新的版本。所以，你只需要让别人更新一下代码就可以了，你之前操作的提交记录也会被保留下来
```

## 压缩commit <a href="#ya-suo-commit" id="ya-suo-commit"></a>

```sh
# 压缩commit   n代表要压缩的个数
git rebase -i HEAD~n

# 取消rebase
git rebase --abort


# 执行rebase 后，可以看到出现的commit前面都是pick
pick    	# 保留该commit  缩写p
squash  	# 用该commit合并到前一个老的commit中去  缩写s
reword 		# 和pick类似，但可修改commit时的提交信息 缩写r
edit      	# 使用commit，但停下来进行修改，可能用于解决冲突。缩写e
fixup		# 和squash类似，但舍弃commit信息。缩写f
exec		# 执行shell命令。缩写x

# 将出现的信息除第一个以外的pick都改为s（这样commit就压缩到第一个内）
wq

# 出现提交信息，修改后保存（不想修改可直接wq）
wq


# 完成后需要一次强制push
git push --force  # 覆盖掉之前的commit
```

## 开发中暂存代码 <a href="#kai-fa-zhong-zan-cun-dai-ma" id="kai-fa-zhong-zan-cun-dai-ma"></a>

```sh
# 开发过程中如果需要切换其他分支可执行stash（将git中修改代码存入stash）
git stash  
# 这样git status则无修改记录，则可以切换分支或者其他


# stash列表（查看stash列表，可用序号来恢复代码）
git stash list


# 应用之前的stash
git stash apply	 		  # 恢复最近一次的stash
git stash apply 序号 		 # 序号一般为 0
git stash apply --index	  # 恢复暂存区的内容


# 删除stash
git stash drop 序号


# 默认情况下，它只会将工作区中的修改的文件进行 stash，也就是指 stash 一些 modified 状态的文件。如果要将 untracked 和暂存区中的文件都进行 stash。要使用如下命令：

git stash --index --include-untracked  # git stash -u

# 该命令将暂存区中 tracked 和 untracked 的内容全部进行 stash 暂存
```

## .gitignore <a href="#gitignore" id="gitignore"></a>

这个文件一般用于使git忽略某些文件的提交

```gitignore
# 例如(以下文件)
## jetbrains系列开发工具的一个隐藏文件
.idea/
## mac系统的文件夹的一个隐藏文件
.DS_Store
## window系统的文件夹下的一个隐藏文件（同mac的.DS_Store文件）
desktop.ini

## 可选忽略文件类型 
*.csv
*.txt
*tmp.*
*Tmp.*
*.xml
*.iml

## python
venv/
*__pycache__
node_modules/
package-lock.json

## java
target/
bin/
out/
.vscode/
.settings/
.mvn/
.classpath
*.class

## golang
# Binaries for programs and plugins
*.exe
*.exe~
*.dll
*.so
*.dylib
```

在`.gitignore`文件中声明后，git会忽略这些文件的修改

**注意：**

如果之前提交了这些文件之后才在文件内声明则git还是会出现这些文件的修改

```sh
## 删除github上的误上传文件.idea ，删除之后，push上去之后则将远程的文件删掉，之后git就不会出现该文件的修改记录了
git rm -rf --cached .idea   


## 忽略.gitignore声明强行添加
git add -f 文件
```
