---
layout: post
title:  "spring ioc"
date:   2016-10-31 16:54:00 +0800
categories: java
---

java spring 知识点。

<!-- more -->

## spring

- IOC：**控制反转**，控制权转移，应用程序本身不负责依赖对象的创建和维护，而是由外部外部容器负责创建和维护。
- **DI（依赖注入）是其一种实现方式**
- **目的：** 创建对象并且组装对象之间的关系。

### spring 注入方式 - 设值注入、构造器注入

dao 对象

```
public class InjectionDAOImpl implements InjectionDAO {

	@Override
	public void save(String arg) {
		// 模拟数据库操作
		System.out.println("保存数据：" + arg);
	}

}

```

service 对象

```
public class InjectionServiceImpl implements InjectionService {

	private InjectionDAO injectionDAO;

	// 构造器注入
	public InjectionServiceImpl(InjectionDAO injectionDAO) {
		this.injectionDAO = injectionDAO;
	}


	// 设值注入
	public void setInjectionDAO(InjectionDAO injectionDAO) {
		this.injectionDAO = injectionDAO;
	}


	@Override
	public void save(String arg) {
		// 模拟业务操作
		System.out.println("Service接收参数：" + arg);
		arg = arg + ":" + this.hashCode();
		injectionDAO.save(arg);
	}

}

```

spring xml 配置文件

```
<bean id="injectionService" class="com.imooc.ioc.injection.service.InjectionServiceImpl">
        <constructor-arg name="injectionDAO" ref="injectionDAO"></constructor-arg>
</bean>

<bean id="injectionDAO" class="com.imooc.ioc.injection.dao.InjectionDAOImpl"></bean>

```

TestInjection 测试对象

```
@RunWith(BlockJUnit4ClassRunner.class)
public class TestInjection extends UnitTestBase{

	public TestInjection() {
		super("classpath:spring-injection.xml");
	}

	@Test
	public void testSetter() {
		InjectionService service = super.getBean("injectionService");
		service.save("这是要保存的数据");
	}

	@Test
	public void testCons() {
		InjectionService service = super.getBean("injectionService");
		service.save("这是要保存的数据");
	}
}

```

小结：通过在 spring 配置文件，对 java 文件进行配置（配置文件 name 必须与 java 对应的数据或参数完全一致，否则无法反射 ），直接注册对象，由框架进行管理。自己不用去创建和销毁，提高效率。

### bean 的配置项

- `id` bean 的唯一标识
- `class` 具体要实例化的那个 class
- `scope` 范围、作用域，类型：singleton、prototype、request、session、global session
- `constructor arguments` 构造器参数
- `properties` 类的属性
- `autowiring mode` 自动装配模式
- `lazy-initialization mode` 懒加载模式
- `initialization/destruction method` 「初始化/销毁」的方法

### bean 的生命周期

- 定义（上文 xml）、初始化、使用（注入直接使用）、销毁

### aware 接口

- 实现了 aware 接口的 bean 在被初始化之后，可以获得相应的资源。
- 对 spring 进行扩展提供了方便的入口。

### autowiring mode

- no：不用任何操作（默认选项）
- byname：根据属性名自动装配。
- bytype：更具 class 的类型装配。
- constructor：通过构造器装配，与 bytype 相似。

### resource 针对资源文件的统一接口。

例：classpath:com/myapp/config.xml、file:/data/config.xml、http:/myserver/logo.png ，用于拿到资源文件的名称、大小、内容。

- URLResource：URL 对应的资源，更具一个 url 即可构建。
- ClassPathResource：获得该类路径下的资源文件。
- FileSystemResource：获取文件系统里面的资源。
- ServletContextResource
- InputSystemResource
- ByteArrayResource
