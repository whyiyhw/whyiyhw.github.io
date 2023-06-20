---
title: go-zero单体化改造（一）gorm的接入
abbrlink: 500f78a8
date: 2023-06-20 23:30:28
categories: golang
index_img: /img/post/05.png
tags:
  - golang
  - go-zero
  - gorm
---

主要分三点讨论如何在 go-zero 中 接入与使用 gorm 。

大多数时候，应用都是单体架构，go-zero 的好处就是，可以快速的搭建一个单体应用，但是随着业务的发展，单体应用的压力也会越来越大，这时候就需要对单体应用进行拆分，拆分的方式有很多，比如拆分成微服务，多个单体应用，多个模块...

而 go-zero 提供了方案，可以在业务小的时候单体，规模变大时候，很方便的拆分成微服务，这就是 go-zero 的独特魅力。

对于 go-zero 中，我最难受的点就在db 这部分，写过 laravel 的人，再去写 go-zero 的 db 部分，会觉得很难受，因为单体应用也很难达到，数据承压的极限，所以我最终选择用 gorm 这种 orm 框架，一个是快速开发，第二个是我的开源项目会用到 pgsql.

之前已经在项目中用 MySQL 实现了 基础的业务逻辑，所以目前主要就是对 db 部分进行替换。

第一步 `service/demo/api/internal/svc/servicecontext.go` 修改 服务上下文

```go
package svc

import	"gorm.io/driver/postgres"
import	"gorm.io/gorm"

type ServiceContext struct {
	Config    config.Config
	DbEngin   *gorm.DB
}

func NewServiceContext(c config.Config) *ServiceContext {
	//启动Gorm支持
	db, err := gorm.Open(postgres.New(postgres.Config{
		DSN:                  c.PGSql.DataSource,
		PreferSimpleProtocol: true, // disables implicit prepared statement usage
	}), &gorm.Config{})

	if err != nil {
		panic(err)
	}

	return &ServiceContext{
		Config:    c,
		DbEngin:   db,
	}
}
```

这样就可以在服务上下文中，添加 gorm 的支持，然后在 `service/demo/api/internal/config/config.go` 中添加 pgsql 的配置

```go
type Config struct {

	PGSql struct {
		DataSource string
	}
}
```
改造就完成了，是的就是这么简单，但是现在的易用性很差，各种方法也都需要自己去封装。用 gorm 肯定会定义 model, 我这个时候，发现了一个新的工具

第二步 orm 的易用性工具

- `gentool` 用于生成 gorm 的 model 与对应的实用方法

```shell
go install gorm.io/gen/tools/gentool@latest
```

比如 在项目根目录下执行
```shell
gentool -db "postgres" -dsn "host=localhost user=root password=123456 dbname=demo port=18886 sslmode=disable TimeZone=Asia/Shanghai" -outPath "./service/demo/dao"
```

![/img/post/go-zero01.png](/img/post/go-zero01.png)

可以看到连接数据库后，自动生成对应的结构体，基础的 curd 方法都实现了，很久没有用 gorm ，再次使用，给我的感受是，emmm 还挺不错的~ 易用性加强了很多

然后再次修改 `service/demo/api/internal/svc/servicecontext.go` 服务上下文

```go

type ServiceContext struct {
	UserModel *dao.Query
}

return &ServiceContext{
    Config:    c,
    UserModel: dao.Use(db),
}
```

最后就就可以在 `service/demo/api/internal/logic` 中使用了

```go
// 用户是否存在
table := l.svcCtx.UserModel.User

user, selectErr := table.WithContext(l.ctx).Where(table.ID.Eq(req.ID)).First()

if !errors.Is(selectErr, gorm.ErrRecordNotFound) && err != nil {
    err = errors.Wrapf(xerr.NewErrCodeMsg(xerr.DBError, "查询用户失败"), "查询用户失败 %v", err)
    return
}

```

一个简单的逻辑，但是已经可以看到，使用 gorm 后，代码的可读性，易用性都提高了很多。

比较坑的是 `gorm.ErrRecordNotFound` 这个 error , 一开始我以为是 db 的错误，但是后来发现，这个错误是 gorm 的一个特性，如果查询不到数据，会返回这个错误，数据会给一个 null。

以上就是我在 go-zero 中接入 gorm 的过程，如果你也在用 go-zero ，可以参考一下，如果你有更好的方法，欢迎留言讨论。