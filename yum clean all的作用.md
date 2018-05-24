## yum clean all的作用

今天发现一台机器/var > 70% ,查了下是/var/cache/yum目录。

使用yum clean all 清除，很方便，绕开了没有root权限的问题。

该命令介绍如下，作用：清除YUM缓存。

```
yum 会把下载的软件包和header存储在cache中，而不自动删除。如果觉得占用磁盘空间，可以使用yum clean指令进行清除，更精确 的用法是yum clean headers清除header，yum clean packages清除下载的rpm包，yum clean all一全部清除。
```

搭配命令：

yum clean all

yum makecache