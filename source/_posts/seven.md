---
title: "MyBatis-Plus动态数据源"
date: 2019-4-5 14:20:12
categories: 
    - "数据库"
toc_number: false
tags:
	- MyBatis-Plus
	- 数据源
---
# 前言

## MyBatis-Plus
官网简介:MyBatis-Plus是MyBatis的功能强大的增强工具包，用于简化开发。该工具包为MyBatis提供了一些高效，有用，即用的功能。
它是一款Mybatis动态SQL自动注入 Mybatis增删改查CRUD操作中间件，减少开发周期优化动态维护XML实体字段，无入侵全方位ORM辅助层。
在使用期间，最大的感受就是减少了以前爆炸的xml配置，多数据库类型不同语法的兼容。还有它支持代码生成，基础CRUD通用方法封装。
它的作业方式和jpa有些相似，都是通过实体类映射数据库表、字段，用java面向对象的方式封装构造sql。
但是对复杂业务不太友好，仅支持简单的sql，并且只能是单表操作。这也恰恰说明了它只是mybatis增强包，无法与jpa比拟完备性，胜在轻量和学习成本低。
<!--more-->
### 特性
- 无侵入：只做增强不做改变，引入它不会对现有工程产生影响，如丝般顺滑
- 损耗小：启动即会自动注入基本 CURD，性能基本无损耗，直接面向对象操作
- 强大的 CRUD 操作：内置通用 Mapper、通用 Service，仅仅通过少量配置即可实现单表大部分 CRUD 操作，更有强大的条件构造器，满足各类使用需求
- 支持 Lambda 形式调用：通过 Lambda 表达式，方便的编写各类查询条件，无需再担心字段写错
- 支持主键自动生成：支持多达 4 种主键策略（内含分布式唯一 ID 生成器 - Sequence），可自由配置，完美解决主键问题
- 支持 ActiveRecord 模式：支持 ActiveRecord 形式调用，实体类只需继承 Model 类即可进行强大的 CRUD 操作
- 支持自定义全局通用操作：支持全局通用方法注入（ Write once, use anywhere ）
- 内置代码生成器：采用代码或者 Maven 插件可快速生成 Mapper 、 Model 、 Service 、 Controller 层代码，支持模板引擎，更有超多自定义配置等您来使用
- 内置分页插件：基于 MyBatis 物理分页，开发者无需关心具体操作，配置好插件之后，写分页等同于普通 List 查询
- 分页插件支持多种数据库：支持 MySQL、MariaDB、Oracle、DB2、H2、HSQL、SQLite、Postgre、SQLServer 等多种数据库
- 内置性能分析插件：可输出 Sql 语句以及其执行时间，建议开发测试时启用该功能，能快速揪出慢查询
- 内置全局拦截插件：提供全表 delete 、 update 操作智能分析阻断，也可自定义拦截规则，预防误操作


# 问题
我接手了一个项目，就使用了MyBatis-Plus。前期开发的人，没有考虑到会出现需要动态切换数据源的情况，而我在想要升级的时候，却遇到了一些麻烦。
## 动态数据源
MyBatis-Plus是支持多数据源的，而且配置简单，但是需要满足它指定的格式。
```yaml
spring:
  datasource:
    dynamic:
      datasource:
        mysql:
        oracle:
        sqlserver:
        postgresql:
        h2:
```
然后在service层加入注解@DS即可，如@DS("mysql")

## 无法兼容
但是我们现在的项目都纳入了微服务管理，集中配置，所有的项目都使用一套数据库配置。格式上是不满足上面MyBatis-Plus的要求的，而且也不可能为了一个项目而去改变。
之前在做其他项目的时候，切换数据源是另一种方式，就是构造多个数据源，在代码里注入，获取连接然后显式的调用。这种方式在目前这个项目是否适用呢？
显然是不行，因为mybatis-plus已经封装很多CRUD的基础方法，并且在这个项目中也大量使用到.
![littleCoder](dynamic-1.png)
改源码，加拦截器？显然都不是一个好办法。于是考虑从mybatis动态数据源入手，最终找到了解决之道，方式和mybatis类似，但还是有些差别。
# 解决方案
- 首先定义一个动态数据源，这里继承的是AbstractRoutingDataSource，它是spring提供的一个轻量级数据源切换方式，实现可动态路由的数据源。
```java
public class DynamicDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return DBContextHolder.getDbType();
    }
}
```

- 定义数据源
```java
@Configuration
public class DatasourceConfig {



    @Bean(name = "oneDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.one")
    public DataSource druidDataSourceOne() {
        DruidDataSource druidDataSource = new DruidDataSource();
        return druidDataSource;
    }

    @Bean(name = "twoDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.two")
    public DataSource druidDataSourceTwo() {
        DruidDataSource druidDataSource = new DruidDataSource();
        return druidDataSource;
    }

    /*将初始化好的多数据源加入动态数据源管理*/
    @Bean("multipleDataSource")
    @Primary
    public DataSource multipleDataSource(@Qualifier("oneDataSource") DataSource oneDataSource, @Qualifier("twoDataSource") DataSource twoDataSource) {
        DynamicDataSource dynamicDataSource = new DynamicDataSource();
        Map< Object, Object > targetDataSources = new HashMap<>();
        targetDataSources.put(DBTypeEnum.ONE.getCode(), oneDataSource);
        targetDataSources.put(DBTypeEnum.TWO.getCode(), twoDataSource);
        dynamicDataSource.setTargetDataSources(targetDataSources);
        dynamicDataSource.setDefaultTargetDataSource(oneDataSource);//默认数据源
        return dynamicDataSource;
    }



    /*配置数据库类型，同时支持oracle和mysql*/
    @Bean
    @Primary
    public DatabaseIdProvider getDatabaseIdProvider(){
        DatabaseIdProvider databaseIdProvider = new VendorDatabaseIdProvider();
        Properties properties = new Properties();
        properties.setProperty("MySQL","mysql");
        properties.setProperty("Oracle","oracle");
        databaseIdProvider.setProperties(properties);
        return databaseIdProvider;
    }
}
```

- 然后定义数据源切换方法
```java
public class DBContextHolder {

    private static final ThreadLocal contextHolder = new ThreadLocal<>();

    /**
     * 设置数据源
     * @param dbTypeEnum
     */
    public static void setDbType(DBTypeEnum dbTypeEnum) {
        contextHolder.set(dbTypeEnum.getCode());
    }

    /**
     * 取得当前数据源
     * @return
     */
    public static String getDbType() {
        return (String) contextHolder.get();
    }

    /**
     * 清除上下文数据
     */
    public static void clearDbType() {
        contextHolder.remove();
    }
}
```

- 将数据源定义成枚举,便于管理
```java
public enum DBTypeEnum {

    ONE("ONE", "db one"),
    TWO("TWO", "db two");

    DBTypeEnum(String code, String msg) {
        this.code = code;
        this.msg = msg;
    }

    @Getter
    private String code;
    private String msg;
}
```

- 最后是配置mybatis-plus的基础配置
```java
@Configuration
@MapperScan("com.lxh.test.mpPlus.mapper.*")
public class MybatisPlusConfig {

    @Value("${spring.datasource.one.driverClassName}")
    private String dbDialect;

    private String mybatis_mapperLocations = "classpath*:com/lxh/**/mapper/xml/*.xml";


    /**
     * 主键策略
     */
    private IKeyGenerator iKeyGenerator(){
        if(dbDialect.contains("mysql")){
            return new MysqlKeyGenerator();
        }else{
            return new OracleKeyGenerator();
        }
    }

    /**
     *
     * @return
     */
    @Bean("sqlSessionFactory")
    public MybatisSqlSessionFactoryBean sqlSessionFactory(@Qualifier("multipleDataSource") DataSource multipleDataSource, DatabaseIdProvider databaseIdProvider) {
        try {
            MybatisSqlSessionFactoryBean mybatisSqlSessionFactoryBean = new MybatisSqlSessionFactoryBean();
            mybatisSqlSessionFactoryBean.setGlobalConfig(globalConfig(iKeyGenerator()));
            mybatisSqlSessionFactoryBean.setConfiguration(mybatisConfiguration(iKeyGenerator()));
            mybatisSqlSessionFactoryBean.setDatabaseIdProvider(databaseIdProvider);
            PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
            mybatisSqlSessionFactoryBean.setMapperLocations(resolver.getResources(mybatis_mapperLocations));
            mybatisSqlSessionFactoryBean.setDataSource(multipleDataSource);
            mybatisSqlSessionFactoryBean.setPlugins(new Interceptor[]{paginationInterceptor()});
            return mybatisSqlSessionFactoryBean;
        } catch (IOException e) {
            e.printStackTrace();
            return null;
        }
    }


    private GlobalConfig globalConfig(IKeyGenerator iKeyGenerator) {
        GlobalConfig globalConfig = new GlobalConfig();
        GlobalConfig.DbConfig dbConfig = new GlobalConfig.DbConfig();
        dbConfig.setIdType(IdType.INPUT);
        dbConfig.setDbType(DbType.MYSQL);
        dbConfig.setKeyGenerator(iKeyGenerator);
        dbConfig.setFieldStrategy(FieldStrategy.NOT_EMPTY);
        dbConfig.setTableUnderline(true);
        dbConfig.setLogicDeleteValue("0");
        dbConfig.setLogicNotDeleteValue("1");
        globalConfig.setDbConfig(dbConfig);
        LogicSqlInjector logicSqlInjector = new LogicSqlInjector();
        globalConfig.setSqlInjector(logicSqlInjector);
        return globalConfig;
    }

    private MybatisConfiguration mybatisConfiguration(IKeyGenerator iKeyGenerator) {
        MybatisConfiguration mybatisConfiguration = new MybatisConfiguration();
        // 源码里面如果有configuration, 不会注入BaseMapper 里面的方法, 所以这里要这样写
        mybatisConfiguration.init(globalConfig(iKeyGenerator));
        mybatisConfiguration.setMapUnderscoreToCamelCase(true);
        mybatisConfiguration.setCacheEnabled(false);
        return mybatisConfiguration;
    }

    /**
     * 分页插件
     */
    private PaginationInterceptor paginationInterceptor() {
        return new PaginationInterceptor();
    }
}
```

- 那怎么使用呢，不可能在代码里面显示的调用，那工作量太大，耦合性太高。所以这里利用了spring的AOP。
```java
@Component
@Aspect
@Order(-100)
public class DBAspect {

    private static Logger logger = LoggerFactory.getLogger(DBAspect.class);

    @Pointcut("execution(* com.lxh.test.mpPlus.service..*.*(..))")
    private void pointcut() {
    }

    @Before("pointcut()")
    public void before(JoinPoint joinPoint) {
        Object target = joinPoint.getTarget();
        String methodName = joinPoint.getSignature().getName();
        Class<?> clazz = target.getClass();

        logger.info("before [ class = {}, method = {} ] execute", clazz.getName(), methodName);

        Class<?>[] parameterTypes = ((MethodSignature)joinPoint.getSignature()).getMethod().getParameterTypes();

        try {

            Method method = clazz.getMethod(methodName, parameterTypes);

            if (method != null) {
                DBTypeEnum dbType;
                if (clazz.isAnnotationPresent(DBAnnotation.class)) {
                    DBAnnotation dbAnnotation = clazz.getAnnotation(DBAnnotation.class);
                    dbType = dbAnnotation.dbSource();
                } else if ( method.isAnnotationPresent(DBAnnotation.class)) {
                    DBAnnotation dbAnnotation = method.getAnnotation(DBAnnotation.class);
                    dbType = dbAnnotation.dbSource();
                } else {
                    dbType = DBTypeEnum.ONE;
                }
                String preDbType = DBContextHolder.getDbType();
                if(!dbType.getCode().equals(preDbType)){
                    DBContextHolder.setDbType(dbType);
                    logger.info("[class = {}, method = {} ] 切换数据源 [ {} ] 成功", clazz.getName(), methodName, dbType);
                }
            }

        } catch (Exception e) {
            logger.error(" 数据源切换异常", e);
        }
    }
}
```

- 这里可以看到，我们定义了一个注解DBAnnotation
```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD,ElementType.TYPE})
public @interface DBAnnotation {
    DBTypeEnum dbSource() default DBTypeEnum.ONE;
}
```
在这里，我定义它可以在类和方法上声明，以便细粒度的管理。

- 使用方式
```java
@Service
@DBAnnotation(dbSource = DBTypeEnum.ONE)
public class TestServiceImpl implements TestService{

    @DBAnnotation(dbSource = DBTypeEnum.TWO)
    void test(){
        
    }

}
```
