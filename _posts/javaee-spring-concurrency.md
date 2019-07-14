---
layout: post
title:  "java webapp concurrency"
categories: java
---

springMVC 项目搭建，处理抢购高并发 concurrency 层。

<!-- more -->

## 高并发优化分析
红色是可能需要优化的点，绿色不需要优化。
![](https://raw.githubusercontent.com/dreaminglion/dreaminglion.github.io/master/images/javaee-spring-concurrency-situation.png)


## CDN 的理解
商品详情页-静态页面 html,css,js 存储在 CDN。
CND(内容分发网络)加速用户获取数据的系统。
部署在离用户最近的网络节点上。
命中 CND 不需要访问后端服务器。
互联网公司自己搭建或租用。

## 抢购优化点
获取系统时间不用优化。
访问一次内存（Cacheline）大约10ns（一秒十亿次）。

秒杀地址接口分析：
无法使用 CDN 缓存
合适服务器缓存：redis等（一秒十万次），使用集群可以达到百万次。
一致性维护成本低。

秒杀地址接口优化：
请求地址 -> 一致性维护 超时穿透/主动更新（redis） -> Mysql

秒杀操作优化分析：
无法使用 CDN 缓存
后端缓存困难：库存问题
一行数据竞争：热点商品

其它分案分析：执行秒杀
原子计数器 技术-> redis/NoSQL
记录行为消息 技术-> 分布式 MQ
消费消息并落地 技术-> MySQL

成本分析：
运维成本和稳定性：NoSql，MQ等。
开发成本：数据一致性，回滚方案。
幂等性难保证：重复秒杀问题。
不适合新手~ ~

抢购瓶颈分析：
java 与 MySql 的交互会有网络延迟，造成事务变慢（java，MySql实际都很高效）。
update 减库存  网络延迟、GC-->  insert 购买明细  网络延迟、GC--> commit/rollback  

延迟分析：
同城机房：0.5ms-2ms max（1000qps）
update 后 JVM-GC (50ms左右) max（20qps）
异地机房：需要多（15ms-20ms）

优化分析：
行级锁在 Commit 之后释放
优化方向：减少行级锁持有时间

优化思路：
把客户端逻辑放到 MySql 服务端，避免网络延迟和 GC 影响。

如何放到 MySql 服务端：
定制 SQL 方案： update / + [auto_commit] /, 需要修改 MySql 源码。
使用存储过程：整个事务在 MySql 端完成。

优化总结：
前端控制：暴露接口，按钮防重复。
动静态数据分离：CDN 缓存，后端缓存。
事务竞争优化：减少事务锁时间。

## redis 接入，优化接口
本机环境下载并安装 redis https://redis.io/download
在项目 pxm.xml 中，配置 java 的 redis 客户端 jedis。
创建 RedisDao 实现对 Seckill 对象的缓存，缓存中获取，以及实现缓存需要的「序列化」，「反序列化」功能（protostuff 实现）。
在 spring-dao.xml 中，配置 redisDao 的注入数据。
修改 SeckillServiceImpl 中的实现方法，采用 redisDao 获取，减少对数据库的访问量。

redis-server 启动服务
redis-cli 客户端

## 并发优化
![](https://raw.githubusercontent.com/dreaminglion/dreaminglion.github.io/master/images/javaee-spring-concurrency-2.png)
![](https://raw.githubusercontent.com/dreaminglion/dreaminglion.github.io/master/images/javaee-spring-concurrency-3.png)

存储过程
1.存储过程优化：事务行级锁持有的时间
2.不要过度依赖存储过程
3.简单的逻辑可以使用存储过程
4.一个秒杀单6000/qps(6000次/秒)

## 系统可能用到那些服务
CDN
WebServer：Nginx+Jetty
Redis
MySql
