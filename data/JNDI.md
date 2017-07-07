## 配置TomcatJNDI数据源
找到Tomcat的server.xml找到工程的Context节点,添加一个私有数据源

    <Resource name="jdbc/test"  auth="Container"
    type="javax.sql.DataSource"
    username="zsj" 
    password="zsj"
    driverClassName="oracle.jdbc.driver.OracleDriver" 
    url="jdbc:oracle:thin:@localhost:1521:zsj" 
    maxActive="100" 
    maxIdle="30" 
    maxWait="10000"/>

GlobalNamingResources节点,在节点下加一个全局数据源 

