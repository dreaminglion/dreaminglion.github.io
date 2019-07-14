---
layout: post
title:  "java webapp dao"
categories: java
---

springMVC 项目搭建 dao 层。

<!-- more -->

## 使用 maven 生成项目
mvn archetype:generate -DgroupId=org.seckill -DartifactId=seckill -DarchetypeArtifactId=maven-archetype-webapp -DarchetypeCatalog=local

mysql -uroot -p 使用命令行，登录 mysql
mysql -uroot -p -D seckill 连接指定数据库

use seckill; 使用 seckill 数据库

show tables; 查看数据库的表

show create table seckill\G 查看表的创建 sql
show create procedure execute_seckill\G 查看存储过程的创建 sql

使用 idea 打开 maven 项目，打开 webapp/WEB-INF/web.xml 修改 servlet 版本为 3.1

## pom.xml 添加第三方依赖
- 日志 java日志
- 数据库相关依赖
- servlet web 相关依赖
- spring 依赖

## mybatis-config.xml
- 使用 jdbc 的 getGeneratedKeys 获取数据库自增主键值
- 使用列别名替换列名 默认：true
- 开启驼峰命名转换：Table（create_time） -> Entity(createTime)

## 编写 DAO 接口，由 mybatis 自动映射实现
在接口中定义，业务需要的增删改查方法
```
List<Seckill> queryAll(@Param("offset") int offset,@Param("limit") int limit);
```

## 在 mapper/SeckillDao.xml
实现 dao 定义方法的 sql
```
<select id="queryAll" resultType="Seckill">
        SELECT seckill_id,name,number,start_time,end_time,create_time
        FROM seckill
        ORDER BY create_time DESC
        limit #{offset},#{limit}
</select>

```

## jdbc.properties
定义 mysql 的连接地址，用户名，密码等信息。

## 配置 spring 信息
<context:property-placeholder location="classpath:jdbc.properties"/> 配置数据库相关参数 properties 的属性:${url}
数据库连接池
```
<bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource">
        <!--配置连接池属性-->
        <property name="driverClass" value="${driver}"/>
        <property name="jdbcUrl" value="${url}"/>
        <property name="user" value="${username}"/>
        <property name="password" value="${password}"/>

        <!--c3p0 连接池的私有属性-->
        <property name="maxPoolSize" value="30"/>
        <property name="minPoolSize" value="10"/>
        <!--关闭连接后不自动 commit-->
        <property name="autoCommitOnClose" value="false"/>
        <!--获取连接超时时间-->
        <property name="checkoutTimeout" value="1000"/>
        <!--当获取连接失败重试次数-->
        <property name="acquireRetryAttempts" value="2"/>
</bean>

```

配置 SQLSessionFactory 对象
```
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <!--注入数据库连接池-->
    <property name="dataSource" ref="dataSource"/>
    <!--配置 MyBatis 全局配置文件：mybatis-config.xml-->
    <property name="configLocation" value="classpath:mybatis-config.xml"/>
    <!--扫描 entity 包 使用别名-->
    <property name="typeAliasesPackage" value="org.seckill.entity"/>
    <!--扫面 sql 配置文件：mapper需要的 xml 文件-->
    <property name="mapperLocations" value="classpath:mapper/*.xml"/>
</bean>

```

配置扫描 dao 接口包，动态实现 dao 接口，注入到 spring 容器中
```
<!--会在引用配置文件的时候，自动调用这个配置，所以不需要定义 id 属性-->
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <!--注入 sqlSessionFactory-->
    <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
    <!--给出需要扫描 dao 接口包-->
    <property name="basePackage" value="org.seckill.dao"/>
</bean>

```

## 测试 mybatis 与 spring 的调用
- 加载配置 spring（spring 配置中，引用 mybatis 的配置） @RunWith(SpringJUnit4ClassRunner.class) @ContextConfiguration({"classpath:spring/spring-dao.xml"})
- @Resource private SeckillDao seckillDao; spring 注入对象
- 调用 dao 接口方法： Seckill seckill = seckillDao.queryById(id); System.out.println(seckill.toString());
