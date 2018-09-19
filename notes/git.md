[TOC]

#### 一. 对git流程的第一印象图

![git流程](http://images.gitbook.cn/b2ec3d60-48fa-11e7-8b18-b174b07d5eed)

![img](D:/mkdown/qq119604AC6A460688138C5CD0E2484784/387333de83b54472a1d9b4b3ca3cfba9/untitle.png) 

#### 二. 常用命令

##### 1. **配置帐号信息：**

> git config -e [--global] # 编辑Git配置文件
>
> git config --global user.name keke #编辑姓名
>
> git config --global user.email keke.li@gmail.com #编辑邮箱地址
>
> git config --list #查看配置的信息
>
> git help config #获取帮助信息

##### 2. **配置自动换行：**

> git config --global core.autocrlf input #提交到git是自动将换行符转换

##### 3.**配置密钥：**

> ssh-keygen -t rsa –C “emailAddress” #邮箱账户地址
>
> ssh -T git@github.com #测试是否成功

##### 4.**配置别名，git的命令没有自动完成功能，可以配置别名简化命令名称：**

> git config --global alias.st status #git st
>
> git config --global alias.co checkout #git co
>
> git config --global alias.br branch #git br
>
> git config --global alias.ci commit #git ci

##### 6.**本地新建仓库：**

> git init #初始化
>
> git status #获取状态
>
> git add [file1] [file2] ... # .或*代表全部添加或者-A
>
> git commit -m "message" #将内容提交,在冒号里面最好填写英文注释
>
> git remote add origin git@github.com ##此时本地工程与远程仓库已经建立了联系
>
> git push -u origin master #将本地的内容同步到远程仓库中
>
> git show [十六进制码] #显示某一个特定的提交的日志
>
> git log #查看自己提交的历史
>
> git ls-files -u #查看冲突未处理的文件列表
>
> git pull origin master #获取最新的远程仓库代码
>
> git stash list #获取暂存列表

##### 7.**从现有仓库克隆：**

> git clone <https://github.com/KeKe-Li/shadowsocks> #把远程项目克隆到本地
>
> git add -A #跟踪新文件
>
> git add -u [path] # 添加[指定路径下]已跟踪文件
>
> rm *&git rm * # 移除文件
>
> git rm -f * # 移除文件
>
> git rm --cached * # 停止追踪指定文件，但该文件会保留在工作区
>
> git mv file*from file*to # 重命名跟踪文件
>
> git log # 查看自己提交的历史
>
> git commit # 提交更新
>
> git commit [file1] [file2] ... # 提交指定文件
>
> git commit -a #跳过使用暂存区域，把所有已经跟踪过的文件暂存起来一并提交
>
> git commit --amend #修改最后一次提交
>
> git commit -v #提交时显示所有diff信息
>
> git reset HEAD *#取消已经暂存的文件
>
> git reset --soft HEAD * #重置到指定状态，不会修改索引区和工作树
>
> git reset --hard HEAD * #重置到指定状态，会修改索引区和工作树
>
> git reset -- files #重置index区文件
>
> git reset --hard 11d881aea55e844dc0ebf0f3e5bf12a3ca999001 # 还原指定的提交之前的状态
>
> git revert HEAD~ #还原前前一次操作
>
> git checkout -- file #取消对文件的修改（从暂存区——覆盖worktree file）
>
> git checkout branch | tag|commit -- file_name#从仓库取出file覆盖当前分支
>
> git checkout -- .#从暂存区取出文件覆盖工作区
>
> git diff file #查看指定文件的差异
>
> git diff --stat #查看简单的diff结果
>
> git diff #比较Worktree和Index之间的差异
>
> git diff --cached #比较Index和HEAD之间的差异
>
> git diff HEAD #比较Worktree和HEAD之间的差异
>
> git diff branch #比较Worktree和branch之间的差异
>
> git diff branch1 branch2 #比较两次分支之间的差异
>
> git diff commit commit #比较两次提交之间的差异

##### 8.**git log 查看最近的提交日志：**

> git log --pretty=oneline #单行显示提交日志
>
> git log --graph # 图形化显示
>
> git log --abbrev-commit # 显示log id的缩写
>
> git log -num #显示第几条log（倒数）
>
> git log --stat # 显示commit历史，以及每次commit发生变更的文件
>
> git log --follow [file] # 显示某个文件的版本历史，包括文件改名
>
> git log -p [file] # 显示指定文件相关的每一次diff
>
> git stash #将工作区现场（已跟踪文件）储藏起来，等以后恢复后继续工作。
>
> git stash list #查看保存的工作现场
>
> git stash apply #恢复工作现场
>
> git stash drop #删除stash内容
>
> git stash pop #恢复的同时直接删除stash内容
>
> git stash apply stash@{0} #恢复指定的工作现场，当你保存了不只一份工作现场时。

##### 9.**创建分支：**

> git branch -b test #创建并切换test分支
>
> git branch #列出本地分支
>
> git branch -r #列出远端分支
>
> git branch -a #列出所有分支
>
> git branch -v #查看各个分支最后一个提交对象的信息
>
> git branch --merge #查看已经合并到当前分支的分支
>
> git branch --no-merge #查看为合并到当前分支的分支
>
> git branch branch [branch|commit|tag] # 从指定位置出新建分支
>
> git branch --track branch remote-branch # 新建一个分支，与指定的远程分支建立追踪关系
>
> git branch -m old new #重命名分支
>
> git branch -d test #删除test分支
>
> git branch -D test #强制删除test分支
>
> git branch --set-upstream dev origin/dev #将本地dev分支与远程dev分支之间建立链接
>
> git checkout test #切换到test分支
>
> git checkout -b test dev#基于dev新建test分支，并切换
>
> git merge test#将test分支合并到当前分支
>
> git merge --squash test # 合并压缩，将test上的commit压缩为一条
>
> git cherry-pick commit #拣选合并，将commit合并到当前分支
>
> git cherry-pick -n commit #拣选多个提交，合并完后可以继续拣选下一个提交‘’

##### 10.**Rebase:**

> git rebase master #将master分之上超前的提交，变基到当前分支
>
> git rebase --onto master 119a6 #限制回滚范围，rebase当前分支从119a6以后的提交
>
> git rebase --interactive #交互模式
>
> git rebase --continue# 处理完冲突继续合并
>
> git rebase --skip# 跳过
>
> git rebase --abort# 取消合并

##### 11.**Merge**

> git merge origin/branch#合并远端上指定分支

##### 12.**远程仓库的操作：**

> git fetch origin remotebranch[:localbranch] #从远端拉去分支[到本地指定分支]
>
> git pull origin remotebranch:localbranch #拉去远端分支到本地分支
>
> git push origin branch #将当前分支，推送到远端上指定分支
>
> git push origin remote branch # 删除远端指定分支
>
> git push origin remote branch --delete # 删除远程分支
>
> git branch -dr branch # 删除本地和远程分支
>
> git checkout -b [--track] test origin/dev#基于远端dev分支，新建本地test分支[同时设置跟踪]

##### 13.**git是一个分布式代码管理工具，所以可以支持多个仓库，服务器上的仓库在本地称之为remote.**

> git remote add origin #地址
>
> git remote #显示全部远程仓库
>
> git remote –v #显示全部源+详细信息
>
> git remote rename origin1 origin2 #重命名
>
> git remote rm origin #删除
>
> git remote show origin #查看指定源的全部信息

##### 14.**标签：**

> git tag #列出现有标签
>
> git tag -a v0.1 -m 'my version' #新建带注释标签
>
> git checkout tagname #切换到标签
>
> git push origin v1.5 #推送分支到远程仓库上
>
> git push origin –tags #一次性推送所有分支
>
> git tag -d v0.1 #删除标签
>
> git push origin :refs/tags/v0.1 #删除远程标签
>
> git push origin #推送标签到远程仓库
>
> git push origin :refs/tags/ #删除远程标签需要先删除本地标签
>
> git checkout # 放弃工作区的修改
>
> git reflog #显示本地执行过git命令
>
> git remote set-url origin #修改远程仓库的url
>
> git whatchanged --since='2 weeks ago' #查看两个星期内的改动
>
> git ls-files --others -i --exclude-standard #展示所有忽略的文件
>
> git clean -X -f #清除gitignore文件中记录的文件
>
> git status --ignored #展示忽略的文件

##### 15.**强制推送远程仓库：**

> git push -f #强制推送
>
> git log --all --grep='' #在commit log中查找相关内容



#### 三. 实践

##### 1. 解决pull冲突

1. 采用的方式是：    git stash  ，然后 git pull
2. 查看下状态 git status
3. 更新下来之后最后     git stash pop 

##### 2.fatal:authen... 问题

* git config --system --unset credential.helper 

##### 3.未add，想恢复修改之前。

* git checkout -- .   //恢复全部 
* git checkout -- <file> //恢复某个文件 

##### 4.已add，想恢复修改之前。之前的本地修改不会消失，只是恢复到未add。 

* git reset HEAD <file>  //恢复某个文件 
* git reset HEAD .      //恢复所有 

##### 5.已经add，还commit了。 想恢复修改之前。

* git reset -- hard HEAD ^    //回退到上一次commit的状态。 
* git reset -- hard commitid  //回退到指定版本，commitid可以使用git log 查看提交历史中的SHA码，只需要前几位即可。 

##### 6.合并分支 

* git merge dev //合并dev到master，这个命令会留下dev分支，多了一个merge节点 
* git rebase dev //也是合并到主分支，但是dev的commit信息都会合并到master上了，好像从没有dev分支一样。 

##### 7.删除分支

* git branch -d dev //删除本地分支 
* git push origin --delete <branch>  //删除远程分支，支持在v1.7.0之后。之前使用git push origin :<branch> 推送个空分支到远程仓库达到删除目的。 

####三.git客户端问题

#####1.client does not support authentication

~~~mysql
mysql> alter user 'root'@'localhost' identified with mysql_native_password by '123456';
Query OK, 0 rows affected (0.10 sec)
mysql> flush privileges;
Query OK, 0 rows affected (0.01 sec)
~~~



