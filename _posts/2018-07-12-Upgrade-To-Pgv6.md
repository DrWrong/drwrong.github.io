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


# 性能测试

选取了 **PGX + XORM+ SqlBuilder**， **PGX +XORM+ 裸写SQL**， **PGV6+SqlBuilder**, **PRV6+裸写SQL** 几种方案进行性能测试。基准选择的就是老的PGV3 + 存储过程方案



