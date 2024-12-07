+++
date = '2024-12-03T16:34:41+08:00'
draft = true
title = 'Golang源码小试牛刀:Gorm框架源码小探01'
+++

# 前言

## Gorm框架介绍


# 日常使用

## 使用注意事项

1. 方法声明顺序

查询举例：

```go
db.Where(maps).Offset(pageNum).Limit(pageSize).Find(&tags)

db.Select("id").Where("name = ?", name).First(&tag)
```

注意这几个方法的调用顺序不能颠倒，因为GORM 按**链式调用的顺序逐步构建 SQL 查询**。所以在Gorm中查询常见的顺序是——*SELECT语句最先，条件方法其次，分页时先用Offset再用Limit，最后执行查询方法例如Find、First、Create、Delete。*



# 源码

来自于GORMv1.9.16版本（并非最新版本）

## Open()

首先看到源码中的注释 
> Open initialize a new db connection, need to import driver first

意味着需要先import一个**数据库驱动**例如mysql

然后我们拿一个普通的调用作为示例
`db, err := gorm.Open("mysql", "user:password@/dbname?charset=utf8&parseTime=True&loc=Local")`

进入源码我们分层次来看

1. 第一部分

```go
func Open(dialect string, args ...interface{}) (db *DB, err error)
```

传入数据库类型和源信息参数，返回一个**db连接池**和错误

```go
if len(args) == 0 {
	err = errors.New("invalid database source")
	return nil, err
}
var source string
var dbSQL SQLCommon
var ownDbSQL bool
```

先判断是否传入了数据库源参数，如果没有传入则返回错误；其次声明了三个变量：

* source:数据库连接的源信息，从args提取
* dbSQL:是一个接口的实例——标准的*sql.DB对象,其中**SQLCommon**的注释为
  * > SQLCommon is the minimal database connection functionality gorm requires.  Implemented by *sql.DB.
  * 抽象底层数据库连接对象
* ownDbSQL:表示dbSQL是否由当前GORM实例独立管理

2. 第二部分

这里是一个switch分支，通过判断传入的args第一个参数类型

```go
	switch value := args[0].(type) {
	case string:
		var driver = dialect
		if len(args) == 1 {
			source = value
		} else if len(args) >= 2 {
			driver = value
			source = args[1].(string)
		}
		dbSQL, err = sql.Open(driver, source)
		ownDbSQL = true
	case SQLCommon:
		dbSQL = value
		ownDbSQL = false
	default:
		return nil, fmt.Errorf("invalid database source: %v is not a valid type", value)
	}
```

* **string时**:将驱动设置为dialect
  * 如果只有一个参数，那么就是源信息；如果不止一个参数，那么第一个作为驱动，第二个作为源
  * 使用标准库sql打开数据库连接，**传入驱动与源**
  * 并将ownDbSQL设置为true，表示当前实例需要管理这个数据库连接的生命周期
* **SQLCommon时:说明传入的是一个已经初始化好的数据库连接对象**。
  * 将传入的连接对象直接赋值给 dbSQL
  * 标记为 false，表示传入的连接由外部管理，GORM 不负责其生命周期
* 二者都不是时，参数无效

3. 第三部分

```go
	db = &DB{
		db:        dbSQL,
		logger:    defaultLogger,
		callbacks: DefaultCallback,
		dialect:   newDialect(dialect, dbSQL),
	}
	db.parent = db
	if err != nil {
		return
	}
	// Send a ping to make sure the database connection is alive.
	if d, ok := dbSQL.(*sql.DB); ok {
		if err = d.Ping(); err != nil && ownDbSQL {
			d.Close()
		}
	}
	return
```

创建DB的实例

* 将**数据库连接对象** dbSQL 赋值给 DB 的 db 字段，作为后续 SQL 操作的基础
* 设置默认日志器，用于记录 SQL 执行信息
* 设置默认回调函数集，用于拦截和处理数据库操作（如查询、插入等）
* 初始化数据库dialect对象，用于**适配不同的数据库**（如 MySQL、PostgreSQL）。newDialect 根据传入的 dialect 名称和连接对象 dbSQL，返回一个实现特定dialect功能的实例
* 为了支持链式调用，将自身作为 parent。

这一顿属性填充完后，进行了一次连接验证:

* 使用类型断言检查是否为标准的sql.DB类型
* 如果是，进行连接验证——如果ping出错并且由GORM管理生命周期，则关闭连接。

至此，Open方法代码结束了，总结一下就是声明三个变量来存储数据库的有关信息；根据源参数确定driver,source,dbSQL,ownDbSQL这些内容；最后创建好连接对象并使用ping检查连接是否成功。

Open提供了两种初始化方式

* 直接传入驱动名称和连接信息
* 传入已经初始化好的数据库对象（自定义连接池）

Open支持多种数据库，通过dialect对象统一处理差异。

## SQLCommon,SqlDb,sqlTx接口

1. SQLCommon
从上面的Open代码可以看出，这个SQLCommon比较重要

```go
type SQLCommon interface {
	Exec(query string, args ...interface{}) (sql.Result, error)
	Prepare(query string) (*sql.Stmt, error)
	Query(query string, args ...interface{}) (*sql.Rows, error)
	QueryRow(query string, args ...interface{}) *sql.Row
}
```

其抽象了底层的数据库操作，如执行非查询的语句，预编译查询，执行结果为多行的查询，执行结果为某行的查询。

2. sqlDb

```go
type sqlDb interface {
	Begin() (*sql.Tx, error)
	BeginTx(ctx context.Context, opts *sql.TxOptions) (*sql.Tx, error)
}
```

抽象了对数据库连接对象（通常是 *sql.DB）的事务操作方法

* 开启一个新事务
* 开启一个带有上下文和配置的事务
  * 这里的上下文我们在之前的[Context源码](https://www.bfsmlt.top/posts/005golang并发01_context/)中提到过，提供了取消和超时的能力，允许对长时间运行的事务进行控制。
  * opts指定事务的隔离级别

3. sqlTx

抽象了事务对象（*sql.Tx）的控制方法

```go
type sqlTx interface {
	Commit() error
	Rollback() error
}
```

显然有着提交和回滚事务两个方法。