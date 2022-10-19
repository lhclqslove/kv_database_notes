# 常用命令
```c
1.git init 
  初始化仓库，把一个普通文件夹变成git仓库，使用git命令去管理（同目录下多出一个.git文件夹）
2.git add 文件名或 git add --all  
  添加文件到追踪暂存区
3.git status
  查看当前git仓库的修改文件的状态
4.git commit -m "xxx"
  提交历史版本，没被追踪的文件不会提交到历史版本
5.git log
  查看历史版本记录
6.git reset 版本号  --hard
  回滚到指定版本号
7.git reflog 
  记录每次提交的版本号
8.git remote add origin xxx 
  添加别名映射，将远程仓库地址xxx映射为origin  （origin也可以叫别的，只是一个名字而已）
9.git remote -v 
  查看当前有哪些别名映射
10.git remote remove origin 
   删除origin别名映射
11.git pull origin master  
   拉取远程origin仓库的内容到本地仓库
12.git push origin master 
   推送本地仓库的历史修改到远程origin仓库
   git push origin master -f   #强制push到远程origin

```

