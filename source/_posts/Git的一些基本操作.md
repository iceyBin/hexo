---
title: Git的一些基本操作
date: 2019-06-27 16:29:56
tags: git
categories: 版本控制

---

## Git的一些基本操作

> 设置SSH KEY

	ssh-keygen -t rsa -C "yangbinh@yonyou.com" -f '~/.ssh/id_rsa_gitlab'

> **fork** 操作

可以通过在github/gitlab上，点击 fork 进行操作将已有的项目fork到自己的repository中

> **Clone** 操作

	git clone <repository_url>

> 创建分支

- 根据本地分支创建： 
	
		git branch <branch_name>

- 根据远程分支创建

		git checkout -b <branch_name> origin/<branch_name>

> 创建remote

	git remote add <remote_name> <remoet_url>

> 更新remote

	git remote update <remote_name> <remoet_url>

> 更新remote的url地址

	git remote set-url <remote_name> <remoet_url>

> 删除remote

	git remote remove <remote_name>

> 同步上游 - **fetch**命令

	git fetch --all
	git fetch <remote_name>
	git fetch <remote_name> <branch_name>

> 更新本地分支

	git pull <remote_name> <branch_name>

> 删除/强制删除本地分支

	git branch -d <branch_name>
	git branch -D <branch_name>

> 合并指定分支到当前分支

	git merge <branch_name>

> 回退远程分支

	git log --online -10
	git reset --hard <commit_id>
	git git push --force

> 清除本地文件提交状态

	git rm -r --cached <file>
	git rm --cached .

> 本地分支与远程分支建立映射关系

	git branch --set-upstream <remote_name>/<branch_name>

> 基于远程分支创建本地分支

- 创建并切换到新建的分支
		
		git checkout -b <local_branch_name> <remote_name>/<remote_branch_name>

- 仅仅创建不切换

		git fetch origin <local_branch_name> <remote_name>/<remote_branch_name>

> 暂存本地的修改

- 暂存时使用默认的备注信息
	
		git stash

- 暂存指定备注信息

		git stash push -m <message>

> 查看所有暂存

	git stash list

> 获取使用某个暂存

	git stash apply [stash@{1}]

> 恢复使用某个暂存同时删除该暂存

	git stash pop [stash@{1}]

> 删除某个暂存

	git stash drop [stash@{1}]

> 清空暂存

	git stash clear

> 显示某个暂存

	git stash show [stash@{1}]

> 放弃本地某个文件的修改

- 未进行add操作
		
		git checkout -- <file_path>
		git checkout .

- 已进行add未进行commit

		git reset HEAD <file_path>
		git reset HEAD .

- 已进行commit，未push
	- 回退到上一次commit的状态

			git reset --hard HEAD^	

	- 回退到任意版本
					
			git reset --hard  <commit_id>

> 删除commit

- 删除中间的某个commit

		git rebase -i <commit_id>

- 删除指定的commit之后的所有的commit

		git reset --hard <commit_id>

> 最近的一次提交内容归到上一次的commit中

	git commit -amend

> 删除 untracked files

	git clean -f

>  连 untracked 的目录也一起删掉

	git clean -fd

> 连 gitignore 的untrack 文件/目录也一起删掉 （慎用）

	git clean -xfd

> 在用上述 git clean 前，墙裂建议加上 -n 参数来先看看会删掉哪些文件，防止重要文件被误删

	git clean -nxfd
	git clean -nf
	git clean -nfd

> 基于tag创建分支

	git branch <new-branch-name> <tag-name>

> git merge命令若报错: refusing to merge unrelated histories

	git merge <branch-name> --allow-unrelated-histories

> 批量删除远程分支

	git branch -r |grep -E -v 'master|upstream' | sed 's/origin\// /g' |xargs git push -d <remote_name>

>> 解释说明

1. git branch -r: 获取所有的远程分支
2. grep -E： 使用正则表达式筛选结果
3. grep -v：将筛选结果反选
4. sed 's/需要被替换的字符串/替换后的字符串/g': 批量替换每行指定字符串
5. 使用管道命令 |xargs 将参数传给命令 git push -d origin