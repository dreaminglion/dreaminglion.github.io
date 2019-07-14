---
layout: post
title:  "java webapp service"
categories: java
---

springMVC 项目搭建 service 层。

<!-- more -->

## 建立 dto 业务处理对象层
**只有配置过 mybatis xml 的 dao 才和数据库表数据打交到，dto 层数据，是在 service 中使用参数 new 出来的对象。**
Exposer 暴露秒杀地址。
```
// 是否开启秒杀
private boolean exposed;

// 一种加密措施
private String md5;

// id
private long seckillId;

// 系统当前时间（毫秒）
private long now;

// 开启时间
private long start;

// 结束时间
private long end;

```

SeckillExecution 封装秒杀执行后结果。
```
private long seckillId;

//秒杀执行结果状态
private int state;

//秒杀表示
private String stateInfo;

//秒杀成功对象
private SuccessKilled successKilled;

```

## 定义异常 SeckillExecption extends RuntimeException
RepeatKillException extends SeckillExecption 重复秒杀异常。
SeckillCloseExecption extends SeckillExecption 秒杀关闭异常。

## 配置 service 的事务配置
```
<!--扫描 service 包下所有使用注解的类型-->
<context:component-scan base-package="org.seckill.service"/>

<!--配置事务管理器-->
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <!--注入数据库连接池，ref="dataSource" 在 spring-dao.xml 中，一起引用即可-->
    <property name="dataSource" ref="dataSource"/>
</bean>

<!--配置基于注解的声明式事务，默认使用注解来管理事务行为-->
<tx:annotation-driven transaction-manager="transactionManager"/>

```

## 定义 service 接口并实现。
使用 spring 的注入注解，生成对象实例，@Service、@Autowired。
```
@Service
public class SeckillServiceImpl implements SeckillService {

    private Logger logger = LoggerFactory.getLogger(getClass());

    @Autowired
    private SeckillDao seckillDao;

    @Autowired
    private SuccessKilledDao successKilledDao;

```

@Transactional 使用事务，完成抢购：减库存 + 记录购买行为的事务过程。
**当出现运行时异常，Transactional 事务会被回滚并抛出异常。使用 Transactional 要合理的使用 try catch，使用不当会造成数据库执行一半成为脏数据，但是合理的异常如：重复购买异常我们需要处理。**
```
@Transactional
public SeckillExecution executeSeckill(long seckillId, long userPhone, String md5) throws SeckillExecption, RepeatKillException, SeckillCloseExecption {
    if (md5 == null || !md5.equals(getMD5(seckillId))) {
        throw new SeckillExecption("seckill data rewrite");
    }
    //执行秒杀逻辑：减库存 + 记录购买行为
    Date nowTime = new Date();
    int updateCount = seckillDao.reduceNumber(seckillId, nowTime);
    try {
        if (updateCount <= 0) {
            //没有更新记录
            throw new SeckillCloseExecption("seckill is closed");
        } else {
            //记录购买行为
            int insertCount = successKilledDao.insertSuccessKilled(seckillId, userPhone);
            //唯一组合主键
            if (insertCount <= 0) {
                throw new RepeatKillException("seckill repeated");
            } else {
                //秒杀成功
                SuccessKilled successKilled = successKilledDao.queryByIdWithSeckill(seckillId, userPhone);
                return new SeckillExecution(seckillId, SeckillStateEnum.SUCCESS, successKilled);
            }
        }
    //把发现的异常全部 throw 出，回滚事务。   
    } catch (SeckillCloseExecption e1) {
        throw e1;
    } catch (RepeatKillException e2) {
        throw e2;
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
        //所有编译期异常 转化为运行期异常
        throw new SeckillExecption("seckill inner error:" + e.getMessage());
    }
}
```

## 测试抢购逻辑

```
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration({"classpath:spring/spring-dao.xml", "classpath:spring/spring-service.xml"})
public class SeckillServiceTest {

//测试代码逻辑完整，注意可重复执行。
@Test
public void exportSeckillLogic() throws Exception {
    Exposer exposer = seckillService.exportSeckillUrl(id);
    if (exposer.isExposed()) {
        userExecuteSeckill(phone, exposer.getMd5());
    } else {
        logger.warn("exposer={}", exposer);
    }
}

private void userExecuteSeckill(long phone, String md5) {
    try {
        SeckillExecution execution = seckillService.executeSeckill(id, phone, md5);
        logger.info("result={}", execution);
    } catch (RepeatKillException e) {
        logger.error(e.getMessage());
    } catch (SeckillCloseExecption e) {
        logger.error(e.getMessage());
    }
}

```
