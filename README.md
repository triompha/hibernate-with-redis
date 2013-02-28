spring-cache配置文件

<?xml version="1.0" encoding="GBK"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
	xmlns:aop="http://www.springframework.org/schema/aop" xmlns:http-conf="http://cxf.apache.org/transports/http/configuration"
	xmlns:tx="http://www.springframework.org/schema/tx"
	xsi:schemaLocation="http://www.springframework.org/schema/beans 
           http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
           http://www.springframework.org/schema/context
           http://www.springframework.org/schema/context/spring-context-2.5.xsd
           http://www.springframework.org/schema/aop   
           http://www.springframework.org/schema/aop/spring-aop-2.5.xsd 
           http://www.springframework.org/schema/tx
           http://www.springframework.org/schema/tx/spring-tx.xsd" >

	<bean id="jedisPoolConfig" class="redis.clients.jedis.JedisPoolConfig">
		<property name="maxActive" value="2000" />
		<property name="maxIdle" value="10" />
		<property name="maxWait" value="2000" />
		<property name="testOnBorrow" value="true" />
		<property name="testOnReturn" value="true" />
	</bean>
	<!-- 只是单个redis的连接池 -->
	<bean id="jedisPool" class="redis.clients.jedis.JedisPool">
		<constructor-arg index="0"  ref="jedisPoolConfig" />
		<constructor-arg index="1"  value="10.10.77.48" />
		<constructor-arg index="2"  value="6380"  type="int" />
		<constructor-arg index="3"  value="30000" />
	</bean>
	<!-- 分布式散列连接池 将总体数据进行散列处理
	<bean id="shardedJedisPool" class="redis.clients.jedis.ShardedJedisPool">
        <constructor-arg index="0" ref="jedisPoolConfig" />
        <constructor-arg index="1">
            <list>
                <bean class="redis.clients.jedis.JedisShardInfo">
                    <constructor-arg index="0" value="${redis.server.name}" />
                    <constructor-arg index="1" value="${redis.server.port}" type="int" />
                </bean>
            </list>
        </constructor-arg>  
    </bean>
    -->
	<bean id="redisCache" class="com.sohu.spaces.cache.impl.RedisCacheImpl">
		<property name="jedisPool" ref="jedisPool" />
	</bean>
	
</beans>



spring-dao配置文件

<?xml version="1.0" encoding="GBK"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
	xmlns:aop="http://www.springframework.org/schema/aop" xmlns:http-conf="http://cxf.apache.org/transports/http/configuration"
	xmlns:tx="http://www.springframework.org/schema/tx"
	xsi:schemaLocation="http://www.springframework.org/schema/beans 
           http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
           http://www.springframework.org/schema/context
           http://www.springframework.org/schema/context/spring-context-2.5.xsd
           http://www.springframework.org/schema/aop   
           http://www.springframework.org/schema/aop/spring-aop-2.5.xsd 
           http://www.springframework.org/schema/tx
           http://www.springframework.org/schema/tx/spring-tx.xsd" >


	<!-- 加载配置文件 -->
	<bean id="propertyConfigurerForProject"
		class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
		<property name="order" value="1" />
		<property name="ignoreUnresolvablePlaceholders" value="true" />
		<property name="locations">
			<list>
				<value>classpath*:context/*.properties</value>
			</list>
		</property>
	</bean>
	
	<bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource"
		destroy-method="close">
		<property name="driverClass" value="com.mysql.jdbc.Driver" />
		<property name="jdbcUrl" value="jdbc:mysql://entry.db.spaces.sohu.com:3309/spaces_entry?useUnicode=true&amp;characterEncoding=UTF-8&amp;autoReconnect=true&amp;zeroDateTimeBehavior=convertToNull" />
		<property name="user" value="entry" />
		<property name="password" value="spaces" />
		<property name="maxPoolSize" value="40" />
		<property name="minPoolSize" value="1" />
		<property name="initialPoolSize" value="1" />
		<property name="maxIdleTime" value="20" />
	</bean>
	<!-- 读写分离,动态数据源 -->
	<bean id="dynamicDataSource" class="com.sohu.spaces.dao.ds.DynamicDataSource">
	   <property name="defaultTargetDataSource" ref="dataSource" />
	   <property name="targetDataSources" >
	       <map>
	           <entry key="readDataSource" value-ref="dataSource" />
	       </map>
	   </property>
	</bean>
	
	
	<!-- Hibernate SessionFactory -->
	<bean id="sessionFactory"
		class="org.springframework.orm.hibernate3.annotation.AnnotationSessionFactoryBean">
		<!-- <property name="dataSource" ref="dataSource" />  -->
		<property name="dataSource" ref="dynamicDataSource" />
		<property name="hibernateProperties">
            <value>
                hibernate.dialect=org.hibernate.dialect.MySQLInnoDBDialect
                hibernate.show_sql=false
            </value>
        </property>
        <property name="packagesToScan">
    		<list>
        		<value>com.sohu.spaces.entry.model</value>
    		</list>
		</property>
	</bean>	
	<bean id="baseDao" class="com.sohu.spaces.dao.impl.HibernateDaoImpl">
		<property name="sessionFactory" ref="sessionFactory" />
	</bean>
	
	<bean id="cachedDao" class="com.sohu.spaces.dao.impl.CachedDaoImpl">
        <property name="baseDao" ref="baseDao" />
        <property name="cache" ref="redisCache" />
    </bean>
</beans>


cache.xml配置文件

<?xml version="1.0" encoding="UTF-8" ?>
<cache>
    <object name="com.sohu.spaces.entry.model.Entry" >
        <list name="get_user_all_entry_list" sqlString="select entry_id from entry where user_id = ? and status!=-1" 
              param="userId" returnType="org.hibernate.type.LongType" />
        <list name="get_user_status_entry_list" sqlString="select entry_id from entry where status=? and user_id = ?"
              param="status,userId" />
    </object>
</cache>

