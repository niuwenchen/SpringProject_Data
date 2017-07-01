## 10 通过Spring和JDBC征服数据库

	定义Spring对数据访问的支持
	配置数据库资源
	使用Spring的JDBC模版
	
数据持久化过程：初始化数据访问框架，打开连接，处理各种异常和关闭连接。如果上述操作出现任何问题，都有可能损坏或者删除珍贵的企业数据。

Spring自带了一组数据访问框架，集成了多种数据访问技术。不管是直接通过JDBC还是像Hibernate这样的对象关系映射ORM框架实现数据持久化，Spring都能够帮你消除持久化代码中那些枯燥的数据访问逻辑。

当开发Spring应用的持久层的时候，可以使用JDBC，Hibernate，Java持久化API（JPA）或者其他任意的持久化框架。

### 10.1 Spring的数据访问哲学
Spring的目标之一就是允许我们在开发应用程序时，能够遵循面向对象OO原则中的针对接口编程。Spring对数据访问的支持也不例外。

像很多应用程序一样，spittr应用需要从某种类型的数据库中读取和写入数据。为了避免持久化逻辑分散到应用的各个组件中，最好将数据访问的功能放到一个或多个专注于此项任务的组件中。这样的组件通常称为数据访问对象DAO或者Repository

		服务对象----> Repository接口
								Repository实现
								
服务对象通过接口来访问Repository。第一，服务对象易于测试，因为它们不再与特定的数据访问实现绑定在一起。实际上，你可以为这些数据访问接口创建mock实现，这样无需连接数据库就能测试服务对象，而且会显著提升单元测试的效率并排除因数据不一致锁造成的测试失败
``
### 10.1.1 Spring数据访问异常体系

    SQLExcetption异常
        * 应用程序无法链接数据库
        * 要执行的查询存在语法错误
        * 查询中所使用的表或列不存在
        * 视图插入或更新的数据违反了数据库约束
       
异常体系在框架中是很重要的，Hibernate提供了二十几个左右的异常，分别对应于特定的数据访问问题。这样就可以针对想处理的异常编写catch代码块

但是Hibernate的异常是其本身所有的，我们需要将特定的持久化机制独立于数据访问层。如果抛出了Hibernate所特有的异常，那我们对Hibernate的使用将会
渗透到应用程序的其他部分。如果不这样做，就需要捕获持久化平台的异常，然后将其作为无关的异常再次抛出

JDBC太简单，Hibernate的异常体系是其本身独有的，一种具有描述性而且与特定的持久化框架无关的异常体系是必须的

### Spring提供的平台无关的持久化异常
Spring提供了多个数据访问异常，分别描述了对应的问题。
![](file:///C:\Users\loneve\Pictures\spring\0.png)

所有的异常都是继承自DataAccessException,非检查型异常


### 10.1.2 数据访问模版化
Spring将数据访问过程中固定的和可变的部分明确划分为两个不同的类:模版和回调，模版管理过程中固定的部分，而回调处理自定义的数据昂访问代码
![](file:///C:\Users\loneve\Pictures\spring\1.png)

针对不同的持久化平台，Spring提供了多个可选的模版。

    jdbc.core.JdbcTemplate  JDBC连接
    HibernateTemplate   Hibernate
    JpaTemplate         Java持久化API的实体管理器
    
## 10.2 数据源的配置
三种方式
    
    * 通过JDBC驱动程序定义的数据源
    * 通过JNDI查找的数据源
    *  连接池的数据源

### 10.2.1 JNDI数据源
外部注入，外部管理的数据源，并不涉及应用程序，而且应用服务器中管理的数据源通常以池的方式组织，具备更好的性能，并且还支持系统管理员对其进行热切换。

Bean方式引用JNDI数据源

    <jee:jndi-lookup id="dataSource" jndi-name="/jdbc/SpitterDS"
    resource-ref="true"/>
    如果应用程序运行在Java应用服务器中，需要将resource-ref设置为true，给定的jndi-name会自动
    加上  java:comp/env/ 前缀
    
    也可以使用Java配置为Bean
    
    @Bean
    public JbdiObjectFactoryBean dataSource(){
        JndiObjectFactoryBean jndiObjectFB = new JndiObjectFactoryBean();
        jndiObjectFB.setJndiName("jdbc/SpitterDS")
        jndiObjectFB.setResourceRef(true)
        jndiObjectFB.setFactoryInterface(javax.sql.DataSource.class)
        return jndiObjectFB;
     }
     

### 使用数据源连接池

 * Apache Commons  DBCP(http://jakarta.apache.org/commons/dbcp)
 * c3p0 (http://sourceforge.net/projects/c3p0/) 
 * BoneCP (http://jolbox.com/)

配置方式

    <bean id="dataSource"  class="org.apache.commons.dbcp.BasicDataSource"
        p:driverClassName="org.h2.Driver"
        p:url="jdbc:h2:tcp://localhost/~/spitter"
        p:username="sa"
        p:password=""
        p:initialSize="5"
        p:maxActive="10"
    />
    
    Java 配置
    
    @Bean 
    public BasicDataSource dataSource(){
        BasicDataSource ds = new BasicDataSource();
        ds.setDriverClassName("org.h2.Driver")
        ds.setUrl("jdbc:h2:dbcp://localhost/~/spitter");
        ....
        return ds;
      }
      

### 基于JDBC驱动的数据源
* DriverManagerDataSource: 每个连接请求时都会返回一个新建的连接，并没有进行池化管理
* SimpleDriverDataSource: 直接使用JDBC驱动，来解决在特定环境下的类加载问题
* SingleConnectionDataSource: 每个连接请求都会返回同一个的连接。

配置方式与上面的DBCP一样


### 使用嵌入式的数据源
嵌入式数据库(embedded database),嵌入式数据库作为应用的一部分运行，而不是应用连接的独立数据库服务器。
在开发和测试来讲，嵌入式数据库都是很好的可选方案。这是因为每次重启应用或运行测试的时候，都能够重新
填充测试数据。

Spring的jdbc命名空间能够简化嵌入式数据库的配置。

    @Bean
    public DataSource dataSource(){
        return new EmbeddedDataSourceBuilder()
        .setType(EmbeddedDatabaseType.H2)
        .addScript('classpath:schema.sql')
        .addScript('classpath:test-data.sql')
        .build();


##10.2.5 使用profile选择数据源
在开发期来说，<jdbc:embedded-database>是很适合的
在QA环境中， DBCP的BasicDataSource
在生产部署环境中， <jee:jndi-lookup> 更好

借助Spring的profile特性能够在运行时选择数据源

    @Profile("development")
    @Bean
        DataSource  embeddedSource
        
    @Profile("qa")
    @Bean
        BasicDataSource
    
    @Profile("production")
    @Bean
        JndiObjectFactoryBean
        
    选择运行状态，决定一切
    

## 10.3 在Spring中使用JDBC
模版jdbcTemplate 提供的方法。

    
        


