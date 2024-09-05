多窗口管理工具

# 常用命令
查看所有 session`screen -ls`

新建 session`screen -S yourname`

返回 session`screen -r yourname`

结束当前session并回到yourname这个session`screen -d -r yourname`

重命名（外部）`screen -S 【8890.foo】 -X sessionname 【new_name】`

重命名（内部）

1. <font style="color:rgb(35, 38, 41);">Press</font><font style="color:rgb(35, 38, 41);"> </font>Ctrl<font style="color:rgb(35, 38, 41);">+</font>A<font style="color:rgb(35, 38, 41);">.</font>
2. <font style="color:rgb(35, 38, 41);">Type</font>`<font style="color:rgb(35, 38, 41);"></font>:sessionname _mySessionName_`

[https://superuser.com/questions/370510/rename-screen-session](https://superuser.com/questions/370510/rename-screen-session)

screen -X



session内返回快捷键

