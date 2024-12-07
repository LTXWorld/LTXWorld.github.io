+++
date = '2024-12-07T09:06:52+08:00'
title = 'Debug系列01 Postgresql'
categories = ["通用技术"]
tags = ["Debug","Postgresql","Docker"]
+++

# 前言

欢迎来到我的Debug系列第一集，我目前还没有完全想好这个系列该怎么去组织规划，但是我想记录下来遇到的大大小小的bug以及debug的过程应该于我自己而言是一件蛮有意思的事情。

![](/img/ys/晚上好兄弟们.jpg)

即使人们常说我们所遇到的bug别人一定都遇到过，去Google一定都能解决，但是如果遇到相同bug的人并没有发表于网络呢？那我们可能真的搜索不到这个bug。并且每个人的操作环境总会有大大小小的差异，所以我还是认为每个人的每个bug都是独一无二的，要说搜索解决方法，不如说是去搜索类似的问题来启发自己。

所以，让我们来记录这一切，万一哪天能启发到别人呢？让自己感受文字的温暖吧。

# Postgresql的连接问题

## 问题背景

我在启动一个新的项目时，使用docker启动Postgresql，使用了下方这样一个docker命令写在了我的Makefile中。

```makefile
lanch_postgres:
	docker run --name postgres_urls \
	-e POSTGRES_USER=LTX \
	-e POSTGRES_PASSWORD=Lutaol123 \
	-e POSTGRES_DB=urldb \
	-p 5432:5432 \
	-d postgres
```

命令的含义是运行一个名为postgres_urls的容器，特权用户为LTX（这里直接使用特权用户操作数据库了），密码为Lutaol123,数据库名称为urldb,端口映射主机的5432映射到容器中的5432端口，-d使容器在后台持续运行——以postgres这个镜像为基础的容器。

执行`make lanch_postgres`后成功启动了这个容器，接着来到数据库迁移（*使数据库操作变得可以回滚，变得像Git一样的可以进行版本控制*）具体如何迁移见[官方链接]()，**最终效果就是在容器中的数据库执行了一条建表的语句**。

这一阶段的makefile如下:

```makefile
databaseURL="postgres://LTX:password@localhost:5432/urldb?sslmode=disable"

migrate_up:
	migrate -path="./database/migrate" -database=${databaseURL} up
```

可是执行makefile，`make migrate_up`时报错:

```bash
postgres://LTX:Lutaol123@localhost:5432/urldb?sslmode=disable
migrate -path="./database/migrate" -database=postgres://LTX:Lutaol123@localhost:5432/urldb?sslmode=disable up
error: failed to open database, "postgres://LTX:Lutaol123@localhost:5432/urldb?sslmode=disable": pq: role "LTX" does not exist
make: *** [migrate_up] Error 1
```

![](/img/ys/药水挥拳.webp)

## 问题处理过程

### 错误信息

错误信息是打开数据库失败，"LTX"这个用户不存在。

经检查可以发现path是没有问题的，意味着我们写的数据库连接URL是正确的。而最后的`pq: role "LTX" does not exist`才是真正的问题所在。

但是上面我们使用docker创建这个Postgresql容器时不是成功了吗？这里怎么又不存在了呢？

![](/shu/开枪.webp)

### 尝试解决

1. 检查makefile中的命令有无错误

* 通过错误信息可以得知，我们的路径信息都是正确的，即迁移命令和docker命令中的信息是吻合的。
* 所以一定不是这里的问题。

2. 检查Docker容器

* 这里我们取巧，直接去Dockerdesktop桌面化环境中去检查，打开指定的容器后，输入命令`psql -U LTX -d urldb`进入到数据库中，发现进入成功了？那用户怎么会不存在呢？
* 再执行一系列的Postgresq命令检查当前连接信息`\conninfo`显示正常连接, 检查数据库信息`\l`显示所有的数据库发现存在urldb;
* 但是查询数据库中的所有表信息时出现了错误，`\dt`显示并没有表？

![](/img/debug/pq1.png)

所以，很明显迁移命令没有执行成功，因为表都没有。

看到这里你也许会有疑惑，为什么数据库创建好了呢？

* 这和最初的docker命令有关，**指定了POSTGRES_DB和POSTGRES_USER之后容器启动时会自动创建我们所指定的数据库和超级用户**。
* 可以使用`\du`命令检查用户权限

![](/img/debug/pq2.png)

### 最终解决

那到底为什么没有连接成功呢？我突然一想，自己之前做过一个项目也是使用的Postgres，并且我是在本地使用的，不会这个迁移命令给我连接到了本地了吧?!

```bash
# 进入本地的数据库
psql postgres
# 检查当前数据库信息
\l
```

坏啦！怎么本地数据库里面有`urldb`这个数据库啊!那一定是连接到本地了（因为我之前并没有操作本地Postgres）。再连接这个数据库查查表信息确认一下。

```bash
# 连接urldb数据库并检查表信息
\c urldb
\dt
```

![](/img/pq3.png)

这里Owner变成了ltx_urldb是因为后来我改了一次用户名（因为当初我还以为是用户名的问题）

哦豁，真的是迁移到本地的Postgresql中了，并没有去连容器中的。

### 错误原因

那到这里，错误原因也很明显了，就是`databaseURL="postgres://LTX:password@localhost:5432/urldb?sslmode=disable"`这个数据库链接指向的是localhost，如果本地的运行着，自然就连接到本地了。

这里做个回调小实验，将本地的关闭然后再次尝试向容器中迁移。

```bash
# 因为我是用homebrew安装的
brew services stop postgresql
```

再次执行迁移命令发现成功了

![](/img/debug/pq4.png)

进入容器中也查到了相应的表。

# 总结

一个localhost引发的本地数据库和容器数据库之间的冲突。

其中更底层的原因我询问了GPT:

* PostgreSQL 默认在 /var/run/postgresql/.s.PGSQL.<port> 路径监听 Unix 套接字,**如果 PostgreSQL 客户端检测到本地套接字文件，就会优先通过它连接。**
* 如果本地套接字不可用，则尝试通过 localhost:5432 使用 TCP/IP 连接。并使用端口映射到容器中的5432端口。

![](/img/pq5.png)

好的到这里我们的这次Postgresql的容器连接问题就到这里结束了。

**这里是LTX，感谢您阅读这篇博客，人生海海，和自己对话，像只蝴蝶纵横四海。**