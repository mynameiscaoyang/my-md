#### **SSM整合基本配置**

简述：

​	Spring + SpringMVC + Mybatis + Redis 整合的秒杀系统

#### web.xml 配置

​	配置内容： 

​			♦  spring核心控制器 ：DispatcherServlet

​			♦  spring配置文件：spring-dao.xml  、 spring-service.xml 、 spring-web.xml

​			♦  让springmvc处理所有的http请求

​			♦  配置一些全局的编码等等

~~~xml
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
                      http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
	version="3.1" metadata-complete="true">
	
	<!-- 修改servlet版本为3.1 -->
	<servlet>
		<servlet-name>seckill-dispatcher</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		
		<!-- 配置springMVC需要加载的配置文件
			spring-dao.xml,spring-service.xml,spring-web.xml
			Mybatis - > spring -> springmvc
		 -->
		 <init-param>
		 	<param-name>contextConfigLocation</param-name>
		 	<param-value>classpath:spring/spring-*.xml</param-value>
		 </init-param>
	</servlet>

	<servlet-mapping>
		<servlet-name>seckill-dispatcher</servlet-name>
		<!-- 默认匹配所有的请求 -->
		<url-pattern>/</url-pattern> 
	</servlet-mapping>

</web-app>

~~~





#### spring-dao.xml 配置

​	配置内容：

​			♦   数据库连接配置，从jdbc.properties文件中读取

​			♦   数据库连接池 C3P0

​			♦   mybatis 的 SqlFactory 工厂

​			♦   mybatis 的 Mapper接口的动态扫描配置

​			♦   Redis 注入 到 Spring容器

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:context="http://www.springframework.org/schema/context"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.1.xsd">


	<!-- 配置整合 mybatis过程 -->
	
	<!-- 1.  配置数据库相关 参数 properties的属性 ， ${url}-->
	<context:property-placeholder location="classpath:jdbc.properties"/> 
	
	<!-- 2.数据库连接池 -->
	<bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource">
		<!-- 配置连接池属性 -->
		<property name="driverClass" value="${jdbc.driverClass}" />
		<property name="jdbcUrl" value="${jdbc.jdbcUrl}" />
		<property name="user" value="${jdbc.username}" />
		<property name="password" value="${jdbc.password}" />
		
		<!-- c3p0 连接池的私有属性 -->
		<property name="maxPoolSize" value="30" />
		<property name="minPoolSize" value="10" />
		<!-- 关闭连接后不自动commit -->
		<property name="autoCommitOnClose" value="false" />
		<!-- 连接超时时间 -->
		<property name="checkoutTimeout" value="10000" />
		<!-- 当连接失败，重试次数 -->
		<property name="acquireRetryAttempts" value="2"></property>
	</bean>
	
	<!-- 3. 配置SqlSessionFactory 对象 -->
	<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
		<!-- 注入数据库连接池 -->
		<property name="dataSource" ref="dataSource" />
		<!-- 配置mybatis 全局配置文件 -->
		<property name="configLocation" value="classpath:mybatis-config.xml" />
		<!-- 扫描entity包， 使用别名 -->
		<property name="typeAliasesPackage" value="org.seckill.entity" />
		<!-- 扫描sql 配置文件 mapping需要的xml文件 -->
		<property name="mapperLocations" value="classpath:mapping/*.xml" />
	</bean>
	
	<!-- 4. 配置扫描Dao接口，动态实现Dao接口， 注入到Spring容器中 -->
	<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
		<!-- 注入sqlSessionFactory -->
		<property name="sqlSessionFactoryBeanName" value="sqlSessionFactory" />
		<!-- 指定扫描的路径 -->
		<property name="basePackage" value="mykill.dao" />
	</bean>
	
	<!-- 5.配置redis 注入到spring容器中 -->
	<bean id="redisDao" class="mykill.dao.cache.RedisDao">
		<constructor-arg index="0" value="localhost"></constructor-arg>
		<constructor-arg index="1" value="6379"></constructor-arg>
	</bean>
	
</beans>

~~~



#### spring-service.xml配置

​	配置内容： 因为services层用到了事务，所以要配置事务，并启用事务注解 @Tranactional

​			♦   配置包扫描 （识别@Service）

​			♦   配置事务管理器以及启用事务注解（可以使用@Transactional）

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:context="http://www.springframework.org/schema/context"
	xmlns:tx="http://www.springframework.org/schema/tx"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.1.xsd
		http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-4.1.xsd">

	<!-- 1. 配置包扫描 -->
	<context:component-scan base-package="mykill.service" />
	
	<!-- 2.配置事务管理器 -->
	<bean id="transactionManager"
		  class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
		  <property name="dataSource" ref="dataSource" />
	
	</bean>
	
	<!-- 3.启用事务注解 -->
	<tx:annotation-driven transaction-manager="transactionManager"/>

</beans>

~~~



#### spring-web.xml 配置

​	配置内容： 主要配置spring 相关内容，如：视图解析器，简单的注解模式配置，静态资源处理Servlet等

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:mvc="http://www.springframework.org/schema/mvc"
	xmlns:context="http://www.springframework.org/schema/context"
	xsi:schemaLocation="http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc-4.1.xsd
		http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.1.xsd">
	
	<!--  1. 开启spring 注解模式
		简化配置： 
		(1)自动注册DefaultAnootationHandlerMapping,AnotationMethodHandlerAdapter 
		(2)提供一些列：数据绑定，数字和日期的format @NumberFormat, @DateTimeFormat, xml,json默认读写支持 
	
	 -->
	<mvc:annotation-driven />
	
	<!-- 2. 静态资源默认Servlet 处理器 
		(1)加入对静态资源的处理：js,gif,png
		(2)允许使用"/"做整体映射
	-->
	<mvc:default-servlet-handler/>
	
	<!-- 3. 配置视图解析器 -->
	<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
		<property name="viewClass" value="org.springframework.web.servlet.view.JstlView" />
	 	<property name="prefix" value="/WEB-INF/jsp/" />
	 	<property name="suffix" value=".jsp" />
	</bean>
	
	<!-- 4. 指定扫描web相关bean -->
	<context:component-scan base-package="mykill.web"></context:component-scan>
	
</beans>

~~~



#### mybatis的配置

​	配置内容： 因为spring-dao.xml层中已经配合 mybatis到 spring中，里面会用到一个mybatis的全局配置文件，就是当前如下配置：

~~~xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
	<!-- 配置全局属性 -->
	<settings>
		<!-- 使用jdbc的getGeneratedKeys获取数据库自增主键值 -->
		<setting name="useGeneratedKeys" value="true" />

		<!-- 使用列别名替换列名 默认:true -->
		<setting name="useColumnLabel" value="true" />

		<!-- 开启驼峰命名转换:Table{create_time} -> Entity{createTime} -->
		<setting name="mapUnderscoreToCamelCase" value="true" />
	</settings>
</configuration>
~~~



#### mybatis-generator 配置

​	mybatis-generator 是一个 自动化生成数据库中的表对应的POJO，以及mapper接口、mapper.xml文件的工具，如果是eclipse 工具，要下载一个插件：http://www.mybatis.org/generator/running/runningWithEclipse.html  , 然后可以方便执行下面的配置文件，来生成相应的mybatis的内容

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
  PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
  "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
 
<generatorConfiguration>
  <classPathEntry location="D:\maven-repository\mysql\mysql-connector-java\5.1.37\mysql-connector-java-5.1.37.jar" />
 
  <context id="Mysql2Tables" targetRuntime="MyBatis3">
    <jdbcConnection driverClass="com.mysql.jdbc.Driver"
        connectionURL="jdbc:mysql://localhost:3306/mykill"
        userId="root"
        password="123456">
    </jdbcConnection>

    <javaTypeResolver >
      <property name="forceBigDecimals" value="false" />
    </javaTypeResolver>

    <javaModelGenerator targetPackage="mykill.entity" targetProject="src/main/java">
      <property name="enableSubPackages" value="true" />
      <property name="trimStrings" value="true" />
    </javaModelGenerator>

    <sqlMapGenerator targetPackage="mapping"  targetProject="src/main/resources">
      <property name="enableSubPackages" value="true" />
    </sqlMapGenerator>

    <javaClientGenerator type="XMLMAPPER" targetPackage="mykill.dao"  targetProject="src/main/java">
      <property name="enableSubPackages" value="true" />
    </javaClientGenerator>

   <table tableName="seckill"  enableSelectByExample="false" 
    enableDeleteByExample="false" enableCountByExample="false" enableUpdateByExample="false" />
    <table tableName="success_killed"  enableSelectByExample="false" 
    enableDeleteByExample="false" enableCountByExample="false" enableUpdateByExample="false" />
  </context>
</generatorConfiguration>

~~~



#### Redis使用及配置



1. 编写redis通用工具类代码  

   ~~~java
   package mykill.dao.cache;
   
   import org.slf4j.Logger;
   import org.slf4j.LoggerFactory;
   
   import com.dyuproject.protostuff.LinkedBuffer;
   import com.dyuproject.protostuff.ProtostuffIOUtil;
   import com.dyuproject.protostuff.runtime.RuntimeSchema;
   
   import redis.clients.jedis.Jedis;
   import redis.clients.jedis.JedisPool;
   
   public class RedisDao {
   	private final Logger logger = LoggerFactory.getLogger(this.getClass());
   	
   	private final JedisPool jedisPool;
   	
   	public RedisDao(String host, int port) {
   		jedisPool = new JedisPool(host, port);
   	}
   	
   	
   	/**
   	 *  redis中读数据 （配合使用 序列化框架）
   	 * @param keyName
   	 * @return
   	 */
   	@SuppressWarnings("unchecked")
   	public <T> T getObject(Class<T> c ,String keyName) {
   		Jedis jedis = jedisPool.getResource();
   		byte[] bytes = jedis.get(keyName.getBytes());
   		try {
   			if(bytes != null) {
   				RuntimeSchema schema = RuntimeSchema.createFrom(c.newInstance().getClass());
   				 T t = (T)schema.newMessage();
   				ProtostuffIOUtil.mergeFrom(bytes, t , schema);
   				return t;
   			}
   		}  catch(Exception e) {
   			logger.error(e.getMessage() ,e);
   		} finally {
   			jedis.close();
   		}
   		return null;
   	}
   	
   	
   	/**
   	 *  redis 中写数据（配合使用 序列化框架）
   	 * @param t
   	 * @param keyName
   	 * @return
   	 */
   	public <T> String writeObject(T t , String keyName) {
   		Jedis jedis = jedisPool.getResource();
   		try {
   			RuntimeSchema schema = RuntimeSchema.createFrom(t.getClass());
   			byte[] bytes = ProtostuffIOUtil.toByteArray(t, schema, 
   					LinkedBuffer.allocate(LinkedBuffer.DEFAULT_BUFFER_SIZE));
   			
   			// 缓存超时时间  
   			int timeout = 60 * 60;
   			String result = jedis.setex(keyName.getBytes(), timeout, bytes);
   			return result;
   		}catch (Exception e) {
   			logger.error(e.getMessage(), e);
   		}finally {
   			jedis.close();
   		}
   		return null;
   	}
   	
   }
   
   ~~~

   

2. 配置redis到spring容器中

   ~~~xml
   <?xml version="1.0" encoding="UTF-8"?>
   <beans xmlns="http://www.springframework.org/schema/beans"
   	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   	xmlns:context="http://www.springframework.org/schema/context"
   	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
   		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.1.xsd">
       <!-- 5.配置redis 注入到spring容器中 -->
   	<bean id="redisDao" class="mykill.dao.cache.RedisDao">
   		<constructor-arg index="0" value="localhost"></constructor-arg>
   		<constructor-arg index="1" value="6379"></constructor-arg>
   	</bean>
   </beans>
   ~~~

   