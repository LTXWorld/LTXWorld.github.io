+++
date = '2024-12-08T10:35:34+08:00'
draft = true
title = '009Golang项目梳理:短链接项目01'
+++

# 前言

做完了三个Golang小的项目之后，发现自己很难复现出来，所以我反思了一下对于这些项目我要进行一个逐一的复盘，从短链接到GreenLight电影搜索API项目，忘了的部分就再回去看书，我对于自己的告诫是永远不要认为任何东西看一遍就万事大吉了。

牢骚发完了，我们来看看这个项目最终实现的后端是什么效果，通常我们在浏览网页时会遇到一些长链接，复制到其他地方会很不方便——比如我在微信朋友圈分享自己的博客某个链接会因为太长只识别到前面的字符，后面的字符当作普通的文字了，导致不能直接点击跳转。所以我们使用这个短链接生成工具生成一个短的链接。

接下来是展示示例:

![](./img/goproject/display1.gif)

可以发现我们将B站视频中的长链接作为origin_url并且为其自定义了custom_code为"text",利用短链接生成器生成了下方的一个短链接"http://localhost:8888/text",使用这个短链接就可以访问刚才的网站（当然这个localhost:8888后续我会想着更改为一个实际的IP地址，这样就可以做到不止自己一个人使用）

好了看完这个示例就让我们进入项目的内部一探究竟。

本次开发环境为本地Macos15.1,Go1.23.1,使用GolandIDE进行开发。

# 项目深究

本次项目深究将从项目的流程图开始，再进入其中的具体部分的逻辑以及代码。

下面是整个架构的简单示意图:

![](/img/goproject/shortFlow.png)

## 数据库迁移

首先新建database文件夹，在其下建立migrate文件夹——这个database文件夹用来存放所有的数据库SQL语句。

数据库驱动我们使用Postgresql,并且使用docker启动，启动过程中如果遇到本地冲突详见[这一篇文章](https://www.bfsmlt.top/posts/008debug系列01-postgresql/)

我们将相关的命令写入到Makefile中

```makefile
.PHONY: lanch_postgres
lanch_postgres:
	docker run --name postgres_urls \
	-e POSTGRES_USER=LTX \
	-e POSTGRES_PASSWORD=iutaol123 \
	-e POSTGRES_DB=urldb \
	-p 5432:5432 \
	-d postgres
```

命令行执行`make lanch_postgres`后成功启动一个Docker容器搭载Postgresql数据库，可以进入容器中进行测试。

接着我们需要进行数据库的迁移，这里要使用到`go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest`这个迁移工具，可能你会问为什么要进行数据库迁移:

* 版本控制数据库结构,可以使用migrate命令进行回滚
* 自动化数据库变更
* 简化部署流程

执行命令`migrate create -seq -ext=.sql -dir=./database/migrate init_schema`，会在我们的database/migrate路径下生成两个sql文件:

![](/img/goproject/migrate1.png)

我们将建urls表的SQL语句写在up中

```sql
CREATE TABLE IF NOT EXISTS urls (
    "id" BIGSERIAL PRIMARY KEY,
    "original_url" TEXT NOT NULL,
    "short_code" TEXT NOT NULL UNIQUE,
    "is_custom" BOOLEAN NOT NULL DEFAULT FALSE,
    "expired_at" TIMESTAMP NOT NULL,
    "created_at" TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

这个表的字段包括

* 主键id，**BIGSERIAL**自动序列化递增
* 原始的url，**TEXT** 用于存储可变长度的字符串，没有长度限制
* 独特的短码，可以由用户输入，作为短url中的后缀例如:`http:/localhost:8888/short_code`
* 是否由用户自定义，即上方短码是否由用户输入
* 短url的过期时间
* 创建时间

同时我们为short_code和expired_at创建二级索引

* 因为short_code是UNIQUE的。
* 需要定期检查短链接是否过期，所以设置索引加速查询速度。

```sql
CREATE INDEX idx_short_code ON urls(short_code);
CREATE INDEX idx_expired_at ON urls(expired_at);
```