## mysql Error 1130

本地测试项目的时候，发现连接windows本地mysql数据库时如果连接参数是ip:port（例如192.168.10.71:3306）就会导致连接失败

> ERROR 1130: Host wdh.hc.com is not allowed to connect to this MySQL server

其中wdh.hc.com是个人电脑配置的域，如果没有配置大概率就是完整ip

于是找了一些别的博客尝试解决

基本都是说修改`mysql`数据库的`user`表，将其中的用户host从"localhost"改为"%"，但是我改错了字段，把user改掉了，，，结果就是完全连不上了，尝试了各种方法都恢复不了

![table_alter](https://secure2.wostatic.cn/static/cap9KoDvw3G8nkq522UGH6/image.png?auth_key=1681379073-uQZvUUxRCwe8ZzhbQQbV7D-0-5fea1f560af5ec592b9af5f8b0e2e5b4)

这是正确修改结果（这是后面重新装了数据库——老数据也找不回来了T^T，还好只是本地测试）

为了防止再出问题我选择是copy记录修改host字段，如果不能立即生效可以尝试新建查询

```SQL
flush privileges;

```

或者重启mysql服务

```Bash
net stop mysql
net start mysql
```

**更重要的是敏感操作记得提前备份数据！！**

---

中间各种尝试恢复操作就简单概括下，权当教训

因为是zip解压配置的mysql，于是找到没有删除的包，又解压了一遍全新的文件夹，然后删除旧的环境变量、注册表、服务

```Bash
mysql --remove # 移除服务
mysql install # 安装服务 

# 注：
# 安装完后续net start mysql的时候可能会提示找不到文件
# 可以win+r service.msc打开服务查看mysql可执行文件的路径
# 大概率是路径有问题 需要移除并重新安装
# 按理配置了Path环境变量应该不会出现这个路径错误问题
# 反正尝试了一下换了个盘符运行（或者又跑了mysqld --initialize-insecure --user=mysql）
# 直到服务能正常启动
```

![services](https://secure2.wostatic.cn/static/kz1zgDoveNuGUVM6CS4pWB/image.png?auth_key=1681380721-p11jtNWygnRGXDwtuzhDbm-0-c00204bd00fcbefc1af788974524347d)

本来以为简单地将旧的db文件夹复制到新的mysql data目录下即可（也包括了ibdata1文件——这部分可以参考网上mysql数据迁移，反正我看了几篇都没有效果，这里就不贴了），结果没用，一度连新的mysql服务都启动不了，最后怕了于是放弃，手动再建表，好在不多，通用配置表倒是可以直接导线上数据库的（navicat-工具-数据传输）

**再次提醒，随时备份！！**

像这次修改系统级别表的情况可以想我最后做的那样，先复制一条一样的记录再修改，再不然至少看三遍要改的字段对与否（或者copy网上博客的执行命令也行）