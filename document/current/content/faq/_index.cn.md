+++
pre = "<b>6. </b>"
title = "FAQ"
weight = 6
chapter = true
+++

#### 1. 如果SQL在ShardingSphere中执行不正确，该如何调试？

回答：

在Sharding-Proxy以及Sharding-JDBC 1.5.0版本之后提供了`sql.show`的配置，可以将解析上下文和改写后的SQL以及最终路由至的数据源的细节信息全部打印至info日志。
`sql.show`配置默认关闭，如果需要请通过配置开启。

#### 2. 阅读源码时为什么会出现编译错误?

回答：

ShardingSphere使用lombok实现极简代码。关于更多使用和安装细节，请参考[lombok官网](https://projectlombok.org/download.html)。

sharding-orchestration-reg模块需要先执行`mvn install`命令，根据protobuf文件生成gRPC相关的java文件。

#### 3. 使用Spring命名空间时找不到xsd?

回答：

Spring命名空间使用规范并未强制要求将xsd文件部署至公网地址，但考虑到部分用户的需求，我们也将相关xsd文件部署至ShardingSphere官网。

实际上sharding-jdbc-spring-namespace的jar包中META-INF\spring.schemas配置了xsd文件的位置：META-INF\namespace\sharding.xsd和META-INF\namespace\master-slave.xsd，只需确保jar包中该文件存在即可。

#### 4. Cloud not resolve placeholder ... in string value ...异常的解决方法?

回答：

行表达式标识符可以使用`${...}`或`$->{...}`，但前者与Spring本身的属性文件占位符冲突，因此在Spring环境中使用行表达式标识符建议使用`$->{...}`。

#### 5. inline表达式返回结果为何出现浮点数？

回答：

Java的整数相除结果是整数，但是对于inline表达式中的Groovy语法则不同，整数相除结果是浮点数。
想获得除法整数结果需要将A/B改为A.intdiv(B)。

#### 6. 如果只有部分数据库分库分表，是否需要将不分库分表的表也配置在分片规则中？

回答：

是的。因为ShardingSphere是将多个数据源合并为一个统一的逻辑数据源。因此即使不分库分表的部分，不配置分片规则ShardingSphere即无法精确的断定应该路由至哪个数据源。
但是ShardingSphere提供了两种变通的方式，有助于简化配置。

方法1：配置default-data-source，凡是在默认数据源中的表可以无需配置在分片规则中，ShardingSphere将在找不到分片数据源的情况下将表路由至默认数据源。

方法2：将不参与分库分表的数据源独立于ShardingSphere之外，在应用中使用多个数据源分别处理分片和不分片的情况。

#### 7. ShardingSphere除了支持自带的分布式自增主键之外，还能否支持原生的自增主键？

回答：是的，可以支持。但原生自增主键有使用限制，即不能将原生自增主键同时作为分片键使用。

由于ShardingSphere并不知晓数据库的表结构，而原生自增主键是不包含在原始SQL中内的，因此ShardingSphere无法将该字段解析为分片字段。如自增主键非分片键，则无需关注，可正常返回；若自增主键同时作为分片键使用，ShardingSphere无法解析其分片值，导致SQL路由至多张表，从而影响应用的正确性。

而原生自增主键返回的前提条件是INSERT SQL必须最终路由至一张表，因此，面对返回多表的INSERT SQL，自增主键则会返回零。

#### 8. 指定了泛型为Long的`SingleKeyTableShardingAlgorithm`，遇到`ClassCastException: Integer can not cast to Long`?

回答：

必须确保数据库表中该字段和分片算法该字段类型一致，如：数据库中该字段类型为int(11)，泛型所对应的分片类型应为Integer，如果需要配置为Long类型，请确保数据库中该字段类型为bigint。

#### 9. 使用SQLSever和PostgreSQL时，聚合列不加别名会抛异常？

回答：

SQLServer和PostgreSQL获取不加别名的聚合列会改名。例如，如下SQL：

```sql
SELECT SUM(num), SUM(num2) FROM tablexxx;
```

SQLServer获取到的列为空字符串和(2)，PostgreSQL获取到的列为空sum和sum(2)。这将导致ShardingSphere在结果归并时无法找到相应的列而出错。

正确的SQL写法应为：

```sql
SELECT SUM(num) AS sum_num, SUM(num2) AS sum_num2 FROM tablexxx;
```

#### 10. Oracle数据库使用Timestamp类型的Order By语句抛出异常提示“Order by value must implements Comparable”?

回答：

针对上面问题解决方式有两种：
1.配置启动JVM参数“-oracle.jdbc.J2EE13Compliant=true”
2.通过代码在项目初始化时设置System.getProperties().setProperty("oracle.jdbc.J2EE13Compliant", "true");

原因如下:

com.dangdang.ddframe.rdb.sharding.merger.orderby.OrderByValue#getOrderValues()方法如下:

```java
    private List<Comparable<?>> getOrderValues() throws SQLException {
        List<Comparable<?>> result = new ArrayList<>(orderByItems.size());
        for (OrderItem each : orderByItems) {
            Object value = resultSet.getObject(each.getIndex());
            Preconditions.checkState(null == value || value instanceof Comparable, "Order by value must implements Comparable");
            result.add((Comparable<?>) value);
        }
        return result;
    }
```

使用了resultSet.getObject(int index)方法，针对TimeStamp oracle会根据oracle.jdbc.J2EE13Compliant属性判断返回java.sql.TimeStamp还是自定义oralce.sql.TIMESTAMP
详见ojdbc源码oracle.jdbc.driver.TimestampAccessor#getObject(int var1)方法：

```java
    Object getObject(int var1) throws SQLException {
        Object var2 = null;
        if(this.rowSpaceIndicator == null) {
            DatabaseError.throwSqlException(21);
        }

        if(this.rowSpaceIndicator[this.indicatorIndex + var1] != -1) {
            if(this.externalType != 0) {
                switch(this.externalType) {
                case 93:
                    return this.getTimestamp(var1);
                default:
                    DatabaseError.throwSqlException(4);
                    return null;
                }
            }

            if(this.statement.connection.j2ee13Compliant) {
                var2 = this.getTimestamp(var1);
            } else {
                var2 = this.getTIMESTAMP(var1);
            }
        }

        return var2;
    }
```

#### 11. 使用`Proxool`时分库结果不正确？

回答：

使用Proxool配置多个数据源时，应该为每个数据源设置alias，因为Proxool在获取连接时会判断连接池中是否包含已存在的alias，不配置alias会造成每次都只从一个数据源中获取连接。

以下是Proxool源码中ProxoolDataSource类getConnection方法的关键代码：

```java
    if(!ConnectionPoolManager.getInstance().isPoolExists(this.alias)) {
        this.registerPool();
    }
```

更多关于alias使用方法请参考[Proxool官网](http://proxool.sourceforge.net/configure.html)。

PS：sourceforge网站需要翻墙访问。

#### 12. ShardingSphere提供的默认分布式自增主键策略为什么是不连续的，且尾数大多为偶数？

回答：

ShardingSphere采用snowflake算法作为默认的分布式分布式自增主键策略，用于保证分布式的情况下可以无中心化的生成不重复的自增序列。因此自增主键可以保证递增，但无法保证连续。

而snowflake算法的最后4位是在同一毫秒内的访问递增值。因此，如果毫秒内并发度不高，最后4位为零的几率则很大。因此并发度不高的应用生成偶数主键的几率会更高。

在3.1.0版本中，尾数大多为偶数的问题已彻底解决，参见：https://github.com/sharding-sphere/sharding-sphere/issues/1617

#### 13. Windows环境下，通过Git克隆ShardingSphere源码时为什么提示文件名过长，如何解决？

回答：

为保证源码的可读性，ShardingSphere编码规范要求类、方法和变量的命名要做到顾名思义，避免使用缩写，因此可能导致部分源码文件命名较长。由于Windows版本的Git是使用msys编译的，它使用了旧版本的Windows Api，限制文件名不能超过260个字符。

解决方案如下：

打开cmd.exe（你需要将git添加到环境变量中）并执行下面的命令：
```
git config --global core.longpaths true
```
参考资料：
https://docs.microsoft.com/zh-cn/windows/desktop/FileIO/naming-a-file
https://ourcodeworld.com/articles/read/109/how-to-solve-filename-too-long-error-in-git-powershell-and-github-application-for-windows

#### 14. Windows环境下，运行Sharding-Proxy，找不到或无法加载主类 org.apache.shardingshpere.shardingproxy.Bootstrap，如何解决？

回答：

某些解压缩工具在解压Sharding-Proxy二进制包时可能将文件名截断，导致找不到某些类。

解决方案：

打开cmd.exe并执行下面的命令：
```
tar zxvf apache-shardingsphere-incubating-${RELEASE.VERSION}-sharding-proxy-bin.tar.gz
```

#### 15. Type is required 异常的解决方法?

回答：

ShardingSphere中很多功能实现类的加载方式是通过[SPI](https://shardingsphere.apache.org/document/current/cn/features/spi/)注入的方式完成的，如分布式主键，注册中心等；这些功能通过配置中type类型来寻找对应的SPI实现，因此必须在配置文件中指定类型。

#### 16. 为什么我实现了`ShardingKeyGenerator`接口，也配置了Type，但是自定义的分布式主键依然不生效？

回答：

[Service Provider Interface (SPI)](https://docs.oracle.com/javase/tutorial/sound/SPI-intro.html)是一种为了被第三方实现或扩展的API，除了实现接口外，还需要在META-INF/services中创建对应文件来指定SPI的实现类，JVM才会加载这些服务。

具体的SPI使用方式，请大家自行搜索。

与分布式主键`ShardingKeyGenerator`接口相同，其他ShardingSphere的[扩展功能](https://shardingsphere.apache.org/document/current/cn/features/spi/)也需要用相同的方式注入才能生效。
