---
title: spring-boot多数据源
date: 2021-09-30 16:13:33
description: spring boot mybatis 多数据源配置
tags:
 - spring boot
 - mybatis
categories: 
 - spring boot
 - mybatis
---

# Spring boot 基于mybatis的多数据源配置

## 引入pom依赖

spring boot web, mybatis-boot 对应的数据库驱动等pom依赖引入

```pom
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-jdbc</artifactId>
		</dependency>
		
		<dependency>
			<groupId>org.mybatis.spring.boot</groupId>
			<artifactId>mybatis-spring-boot-starter</artifactId>
			<version>1.2.0</version>
		</dependency>
		
		<dependency>
			<groupId>com.oracle</groupId>
			<artifactId>ojdbc14</artifactId>
			<version>11.1.0.6.0</version>
		</dependency>
		
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
		</dependency>
```

## 在properties/yml文件中添加对应的数据库配置

oracle 数据库配置

```properties
#查询oracle的xml地址
mybatis_oracle.mapper_locations=classpath*:oracle/**/*.xml
spring.datasource.oracle.initialize=
spring.datasource.oracle.continueOnError=
spring.datasource.oracle.type=
spring.datasource.oracle.url=
spring.datasource.oracle.username=
spring.datasource.oracle.password=
spring.datasource.oracle.driver-class-name=oracle.jdbc.driver.OracleDriver
#
spring.datasource.oracle.initialSize=5
spring.datasource.oracle.minIdle=5
spring.datasource.oracle.maxIdle=10
spring.datasource.oracle.maxActive=20
spring.datasource.oracle.maxWait=60000
spring.datasource.oracle.timeBetweenEvictionRunsMillis=60000
spring.datasource.oracle.validationQuery=select 1 from dual
spring.datasource.oracle.testWhileIdle=true
spring.datasource.oracle.testOnBorrow=true
spring.datasource.oracle.testOnReturn=true

spring.datasource.oracle.poolPreparedStatements=true
spring.datasource.oracle.maxPoolPreparedStatementPerConnectionSize=20
spring.datasource.oracle.filters=stat,wall,log4j
spring.datasource.oracle.connectionProperties
```

mysql 数据库配置

```properties
#查询mysql的xml地址
mybatis_mysql.mapper_locations=classpath*:mysql/**/*Mapper.xml
spring.datasource.mysql.initialize=
spring.datasource.mysql.continueOnError=
spring.datasource.mysql.type=
spring.datasource.mysql.url=
spring.datasource.mysql.username=
spring.datasource.mysql.password=
spring.datasource.mysql.driver-class-name=
spring.datasource.mysql.initialSize=5
spring.datasource.mysql.minIdle=5
spring.datasource.mysql.maxIdle=10
spring.datasource.mysql.maxActive=20
spring.datasource.mysql.maxWait=60000
spring.datasource.mysql.timeBetweenEvictionRunsMillis=60000

spring.datasource.mysql.validationQuery=SELECT 1
spring.datasource.mysql.testWhileIdle=true
spring.datasource.mysql.testOnBorrow=true
spring.datasource.mysql.poolPreparedStatements=true
spring.datasource.mysql.maxPoolPreparedStatementPerConnectionSize=20
spring.datasource.mysql.filters=stat,wall,log4j
spring.datasource.mysql.connectionProperties=druid.stat.mergeSql=true;druid.stat.slowSqlMillis=5000
```

## 创建对应数据库的配置类

oracle的配置类

```java
@Configuration
@MapperScan(basePackages = {"com.**.mapper.oracle"},  sqlSessionTemplateRef = "oracleSessionTemplate")
public class MybatisOracleConfig {
	
	@Value("${mybatis_oracle.mapper_locations}")
	private String mapper_locations;
	
	@Bean(name = "oracleDS")
	@ConfigurationProperties(prefix = "spring.datasource.oracle") // application.properteis中对应属性的前缀
	public DataSource oracleDS() {
		return DataSourceBuilder.create().build();
	}
	
	@Bean
    public PlatformTransactionManager oralceTransactionManager(@Qualifier("oracleDS")DataSource oracleDS) {
     return new DataSourceTransactionManager(oracleDS);
    }
	
	@Bean
    public SqlSessionFactory oracleSessionFactory(@Qualifier("oracleDS") DataSource dataSource)throws Exception
	{
		SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
		factoryBean.setDataSource(dataSource);
		//添加XML目录
        ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        try {
			List<Resource> resources = new ArrayList<Resource>();
			Resource[] mappers = resolver.getResources(mapper_locations);
			resources.addAll(Arrays.asList(mappers));
        	factoryBean.setMapperLocations(resources.toArray(new Resource[resources.size()]));
 
            return factoryBean.getObject();
        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException(e);
        }
	}

	@Bean
	public SqlSessionTemplate oracleSessionTemplate(@Qualifier("oracleSessionFactory") SqlSessionFactory sqlSessionFactory) throws Exception {
		SqlSessionTemplate template = new SqlSessionTemplate(sqlSessionFactory); // 使用上面配置的Factory
		return template;
	}
	
	public String getMapper_locations() {
		return mapper_locations;
	}

	public void setMapper_locations(String mapper_locations) {
		this.mapper_locations = mapper_locations;
	}

}
```

mysql的配置类

```java

@Configuration
@MapperScan(basePackages = { "com.**.mapper.mysql" }, sqlSessionTemplateRef = "mysqlSessionTemplate")
public class MybatisMySqlConfig {
	
	@Value("${mybatis_mysql.mapper_locations}")
	private String mapper_locations;
	
	@Bean(name = "mysqlDS")
	@Primary
	@ConfigurationProperties(prefix = "spring.datasource.mysql") // application.properteis中对应属性的前缀
	public DataSource mysqlDS() {
		return DataSourceBuilder.create().build();
	}
	
	@Bean
	@Primary
    public SqlSessionFactory mysqlSessionFactory(@Qualifier("mysqlDS") DataSource dataSource)throws Exception
	{
		SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
		factoryBean.setDataSource(dataSource);
		
		//添加XML目录
        ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        try {
        	factoryBean.setMapperLocations(resolver.getResources(mapper_locations));
            return factoryBean.getObject();
        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException(e);
        }
	}
	@Bean
	@Primary
    public PlatformTransactionManager mysqlTransactionManager(@Qualifier("mysqlDS")DataSource mysqlDS) {
     return new DataSourceTransactionManager(mysqlDS);
    }
	@Bean
	@Primary
	public SqlSessionTemplate mysqlSessionTemplate(@Qualifier("mysqlSessionFactory") SqlSessionFactory sqlSessionFactory) throws Exception {
		SqlSessionTemplate template = new SqlSessionTemplate(sqlSessionFactory); // 使用上面配置的Factory
		return template;
	}

	public String getMapper_locations() {
		return mapper_locations;
	}

	public void setMapper_locations(String mapper_locations) {
		this.mapper_locations = mapper_locations;
	}

}
```

# 相关配置介绍

## properties文件相应的配置介绍

mybatis_oracle.mapper_locations 是项目中有关查询oracle sql的xml文件地址

mybatis_mysql.mapper_locations 是项目中有关查询mysql sql的xml文件地址

其余均为链接数据库方面的配置

## 配置类介绍

MybatisOracleConfig mybatis的oracle数据源配置类

MybatisMySqlConfig mybatis的mysql数据源配置类

@Configuration 标识配置类,加入到 spring boot 的bean管理

@MapperScan 标识mybatis扫描包

```java
@Configuration
@MapperScan(basePackages = { "com.instudio.shopee.**.mapper.mysql" }, sqlSessionTemplateRef = "mysqlSessionTemplate")
```

DataSource oracleDS 和 DataSource mysqlDS 通过读取对应开头的配置信息(@ConfigurationProperties(prefix = "spring.datasource.mysql"))创建数据源对象

```java
	@Bean(name = "mysqlDS")
	@Primary
	@ConfigurationProperties(prefix = "spring.datasource.mysql") // application.properteis中对应属性的前缀
	public DataSource mysqlDS() {
		return DataSourceBuilder.create().build();
	}
```

使用上面创建的数据源对象, 创建sqlSession工厂

其中需要把对应的xml文件地址添加进sqlSessionFactory

```java
@Bean
	@Primary
    public SqlSessionFactory mysqlSessionFactory(@Qualifier("mysqlDS") DataSource dataSource)throws Exception
	{
		SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
		factoryBean.setDataSource(dataSource); 
		//添加XML目录
        ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        try {
        			                 factoryBean.setMapperLocations(resolver.getResources(mapper_locations));
            return factoryBean.getObject();
        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException(e);
        }
	}
```

根据数据源DataSource对象创建对应的事务管理器

```java
@Bean
@Primary
   public PlatformTransactionManager mysqlTransactionManager(@Qualifier("mysqlDS")DataSource mysqlDS) {
    return new DataSourceTransactionManager(mysqlDS);
   }
```

根据sqlSessionFactory 创建对应的sqlSession模板

```java
@Bean
@Primary
public SqlSessionTemplate mysqlSessionTemplate(@Qualifier("mysqlSessionFactory") SqlSessionFactory sqlSessionFactory) throws Exception {
   SqlSessionTemplate template = new SqlSessionTemplate(sqlSessionFactory); // 使用上面配置的Factory
   return template;
}
```

