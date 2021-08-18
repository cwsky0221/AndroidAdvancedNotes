#### Gitignore中已经标明忽略的文件目录下的文件，git push的时候还会出现在push的目录中，或者用git status查看状态，想要忽略的文件还是显示被追踪状态。

新建的文件在git中会有缓存，如果某些文件已经被纳入了版本管理中，就算是在.gitignore中已经声明了忽略路径也是不起作用的，
这时候我们就应该先把本地缓存删除，然后再进行git的提交，这样就不会出现忽略的文件了。

比如./idea文件夹下面的文件无法忽略

进入./idea文件夹，执行清除缓存的命令,并重新提交:
```
[root@cw ~]# git rm -r --cached . //清空缓存，删除本地文件
[root@cw ~]# git add . 
[root@cw ~]# git commit -m 'update .gitignore' //重新提交
[root@cw ~]# git push -u origin master

```
然后在修改.gitignore文件，增加 ./idea 就可以忽略./idea目录下的修改了














