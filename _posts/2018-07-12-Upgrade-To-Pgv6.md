---
title: DB驱动升级&替换存储过程In探探
categories:
- daily_stuff
---

# Brief

近期将在探探的一个后端模块中尝试了将**存储过程**替换成*sql Builder* 同时升级了DB驱动由**PBV3**到**PGV6** 效果不错， 本文记录一下变更中遇到的一些问题，以及升级后的性能数据

# Background

在服务端开发的过程中，操作关系型数据库（Mysql， Postgres）是必不可少的操作。 在`golang`里面操作数据库一般会分成三层， 从上到下分别是：

+ ORM层：负责将关系型的数据映射成对象
+ `database/sql`层：Golang标准库里面提供的Sql抽象，隔离了上层的使用与下层的实现
+ `driver`层： 驱动层，用于真正处理DB连接

然而在探探， 数据库上我们使用的是Postgres，操作数据库的工具也没有按照这三层去做， 我们使用的是[`pgv3`](https://github.com/go-pg/pg/tree/v3)，
pgv3将这三层合成了一层来处理，它既提供了ORM映射， 也提供了Postgresql的驱动。 在使用姿势上来讲， 我们也一直使用的是**存储过程**， 所有的数据库操作会封装成一个DB层的函数，然后上层的GO程序直接调用DB层的函数。这种方式存在一些问题

1. **PGV3** 太老了, 现在`go-pg/pg`库已经到了V6版本， V3版本的最后更新时间是3年前
2. **PGV3** 是一个不太完整的版本，无法在代码层面开启事务，很多工作无法做
3. 存储过程管理起来太麻烦，交付周期长，可维护性差且性能不高, 替换是必然要做的选择

因此我们计划在重构完成后进行**驱动的更换**和**存储过程的替换**


# 技术选型

本次升级，首先想做的还是按照`GOLANG`推荐的三层的方式来，选择一个好用的`ORM`做映射， 同时`ORM`包最好支持一些**sql builder**的工作， 这样就不用裸写SQL了。然后再选择一个性能比较高的驱动。

按照这个思路 考量了如下几个ORM 

+ `Beego ORM`: https://beego.me/docs/mvc/model/overview.md
+ `XORM`: http://xorm.io/


考量的驱动有如下几个：

+ `PQ`: https://github.com/lib/pq 
+ `PGX`: https://github.com/jackc/pgx


同时因为我们之前使用`PGV3`所以自然而然也考虑到了使用最新版本的`PGV6`


## ORM选型（SqlBuilder选型）

ORM的选型主要是从易用性角度来考虑的。与其说是ORM的选型不如说是SqlBuilder的选型， 因为ORM故名思义 `Object Relationship Map` 基本上所有的包都会提供关系数据到对象映射的功能。性能上大家都是用的反射差不了多少， 功能上就是一个映射也没有多大区别。
然而SqlBuilder的方式， 易用性确千差万别。

GO的ORM也有很多，之所以重点考量`Beego ORM` 和 `XORM`是因为以下因素：

1. 在上一家公司多盟， 我们一直用的Beego的ORM，蛮好用的
2. Revel推荐的是XORM，同时在最新版本的thrift编译器中也把XORM的tag编译进去了给人一种很Popular的感觉
3. 同时GOlang还很很多各种各样的ORM， github上的星也不多， 所以就没有去看

在`XORM`和`Beego ORM`中，最终选择了`XORM`, 原因如下

1. Beego的Model需要注册，且不能太方便的支持Master， Slave的方式，
2. XORM的sql builder更加易用， 还支持一些事件的Hook， 写起代码来很爽

然而选择了`XORM` 感觉入了一个很大的坑， 它的代码质量不是很高， 有BUG。于是，在使用过程中还给提了几个PR。 总体来讲，感觉XORM还蛮易用的，但还需要一段成长

## 驱动选型

在驱动上，看GOlang的官方博客更推荐`PGX`, 再者发现PQ链接PGBouncer时有点小问题， 所以就选择了`PGX` 

> PGX对PGBouncer的支持也不是太友好， 在使用PGBouncer时，PGX只能试用`PreferSimpleProtocol`模式， 在这种模式下， 只支持有限的几个数据类型.

## Option： PGV6

除了上述之外， 也同时考量了PGV6这个备选方案， PGV6延续了PGV3的特性，将驱动， SqlBuilder， ORM几个功能都耦到了一起。一开始看这个模块的时候，感觉太耦合了，使用它我是拒绝的。 后来测试了一下性能， 发现性能非常好， 同时提供的SqlBuilder功能也不差， 最后竟然成了我们最终的方案


# 性能测试方案及结果

选取了 **PGX + XORM+ SqlBuilder**， **PGX +XORM+ 裸写SQL**， **PGV6+SqlBuilder**, **PRV6+裸写SQL** 几种方案进行性能测试。基准选择的就是老的PGV3 + 存储过程方案

整体压力测试拓扑如下 
![Topology](http://www.plantuml.com/plantuml/png/XOvD2i9038NtEKN0zHJRfQjwbEbCgCByJ9EC6-dTjOXKRCLTU1_Vo-j5BMkD0LBsXBOK8RuHunqGNOub9qgA8nVNH8e3iLokfL75mtcg5kQNvwtQmGfzQMKSSasEkDsFEy1LBLbqP98fj4V03nUD-Gcx3PnXPwmnQzqVtW6uaAj7qMUba02yh-NNLZuj4VIKV8tX0G00)

其中`grpc.membership.tt`是性能测试的主要模块。该模块部署在一台单独的APP Server上， APP Server的机器情况为**4核 23G内存**， DB机器 **48核 378G内存** 上面部署了一个Postgresql10，并挂了两个PGBouncer。

之所以使用两个PGBouncer，是因为测试过程中发现Pgbouncer只能使用单核， 当单核的容量打满了只有`30K` QPS, 就不再增加了。 于是PGBouncer成了瓶颈。


同时因为我们的业务场景主要是查询用户是否有某些特权， 数据量比较小， 读多，写少。 因此压测的时候主要测试我们的读接口`GetUserPrivielgesForCounters`

## 性能测试结果

### 基准情况

基准选取的是原来线上运行的PGV3+存储过程的版本, 压力下表现如下

+ QPS
![QPS](/static_images/static/2018-07-12-17:55:23.png)
+ CPU 情况
![CPU](/static_images/static/2018-07-12-17:56:16.png)

**存在问题**： 无论是直连Postgres， 还是调大线程池，都无法将Appserver的CPU压满， 这个问题很奇怪，但没有找到原因

### PGX

+ With SQL Builder

![QPS](/static_images/static/2018-07-12-17:58:51.png)

+ Without SQL Builder

![QPS](/static_images/static/2018-07-12-17:59:30.png)


### PGV6

![QPS](/static_images/static/2018-07-12-18:03:44.png)

## 结论
1. PGV6的性能要优于PGX和PGV3
2. 使用SqlBuilder会有一些性能上的损失

# 最终上线方案
鉴于以上压测结果， 我们选取了PGV6的方案。 同时对于性能要求比较高的核心接口直接裸写SQL

# 上线后的数据情况

从响应时间上来看，这次操作使响应时间大概有20%左右的提升，且响应时间更加平稳
![线上时时QPS情况](/static_images/static/2018-07-12-17:24:57.png)


# 附录

## 关于SqlBuilder
个人比较倾向于写SqlBuilder的方式来生成sql，而不是裸写sql， SqlBuilder的优点有：

+ 代码美观整洁，换行，对齐比较方便
+ 可以利用到语言本身的高亮
+ 减少了一些拼写上的错误
+ 可读性上更优一些

但是不可避免的SqlBuilder会存在一些性能上的损耗

## PGV6里面的奇葩占位符

PGV6的占位符比较奇葩， 不紧能占位参数，而且能占位SQL中任意的一部分, EG:

```golang
func (o *userTotalPrivilegeOperator) SelectPrivilegsNotExpiredByTypes(
	privilegeTypes []string,
	columns ...string,
) ([]*domain.UserTotalPrivilege, error) {
	log.Debug("Get privielge types uid %s %+v", o.UserID, privilegeTypes)

	var records []*domain.UserTotalPrivilege
	sql := "SELECT ? FROM ?shard.user_total_privileges WHERE user_id = ? AND status = 'default' AND privilege_type IN (?) AND end_time > ?"
	selectFields := pg.Q("*")
	if len(columns) > 0 {
		fields := make([]types.ValueAppender, len(columns))
		for i, column := range columns {
			fields[i] = pg.F(column)
		}
		selectFields = pg.In(fields)
	}

	ormer := o.getOrmer()

	_, err := ormer.Query(&records, sql,
		selectFields,
		o.UserID,
		pg.In(privilegeTypes),
		time.Now().UTC(),
	)

	for _, record := range records {
		record.AfterSelect(ormer)
	}
	// err := o.getOrmer().Model(&records).
	//	Where("user_id = ?", o.UserID).
	//	Where("status = ?", "default").
	//	Where("privilege_type IN (?)", pg.In(privilegeTypes)).
	//	Where("end_time > ?", time.Now().UTC()).
	//	Column(columns...).Select()
	if err != nil {
		log.Err("Query database error %+v", err)
	}

	return records, err
}

```


## 关于Circuit Breaker

1. PGV6自带了Circuit Breaker机制
2. 个人并不认为Circuit Breaker有多大的做用
> 1. 应用层的错误不应该熔断
> 2. DB响应慢了的时候连接池本身就起到了限流的做用
> 3. 网络错误时熔断反而提高了故障恢复的时间
