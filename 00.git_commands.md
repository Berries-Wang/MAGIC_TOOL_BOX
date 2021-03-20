## git 命令
### 01.查看日志

|命令|作用|
|----|----|
| git log |查看详细的提交日志|
| git log --pretty=oneline|	每一次的提交日志只占用一行|
| git log --oneline|更加简洁的显示日志|
| git reflog|有oneline简洁日志的同时显示移动版本步数(所有的记录，包括git reset的)|

----------
## 02.如何删除远程跟踪分支(前提是 对应的远程分支已经删除了)
+ 若有远程分支: master , a , b , c, git fetch 之后,会对所有的远程分支跟踪,git branch -a 可以看到这四个远程分支的远程跟踪分支,如果远程的a分支被删除了,git branch -a 会发现仍在对a分支远程跟踪,这时候如果想清除无用的远程跟踪分支,需要进行如下两步：
   1. git remote prune origin --dry-run  // 列出仍在远程跟踪但是远程已被删除的无用分支，上面例子此处应输出  '* [将删除] origin/a'
   2. git remote prune origin  // 清除上面命令列出来分支的远程跟踪，输出 '* [已删除] origin/a'
---------
## 03.git log  查看详细的提交日志
1. git log -p   , -p 用来显示每次提交的内容差异,也可以加上 -2 来仅显示最近两次提交

   |选项|说明|
   |---|---|
   | -p  | 按补丁格式显示每个更新之间的差异。|
   | --stat | 显示每次更新的文件修改统计信息。|
   | --shortstat | 只显示 --stat 中最后的行数修改添加移除统计。|
   | --name-only |仅在提交信息后显示已修改的文件清单。|
   | --name-status |显示新增、修改、删除的文件清单。|
   | --abbrev-commit| 仅显示 SHA-1 的前几个字符，而非所有的 40 个字符。|
   | --relative-date |使用较短的相对时间显示（比如，“2 weeks ago”）。|
   | --graph| 显示 ASCII 图形表示的分支合并历史。|
   | --pretty| 使用其他格式显示历史提交信息。可用的选项包括 oneline，short，full，fuller 和 format（后跟指定格式）。|

### git log --graph 选项
+ --decorate 标记会让git log显示每个commit的引用(如:分支、tag等) 
+ --oneline 一行显示
+ --simplify-by-decoration 只显示被branch或tag引用的commit
+ --all 表示显示所有的branch，这里也可以选择，比如我指向显示分支ABC的关系，则将--all替换为branchA branchB branchC
#### git log --graph解读
- \*表示一个commit， 注意不要管*在哪一条主线上
- |表示分支前进
- /表示分叉
- \表示合入
------
## 04.删除远程分支
+ 命令: git push origin --delete master 
    - origin: 远程仓库名
    - master: 需要删除的远程分支

------
### 05. git status  
+ git status 命令将为你展示工作区及暂存区域中不同状态的文件。 这其中包含了已修改但未暂存，或已经暂 存但没有提交的文件。 一般在它显示形式中，会给你展示一些关于如何在这些暂存区域之间移动文件的提示。 
+ 即显示git中三棵树的状态  