---
title: Oracle数据库读取US7ASCII编码转换问题
description: Oracle数据库读取US7ASCII编码转换问题
date: 2025-05-20 18:33:00 +0800
categories: [Blogging]
tags: [course]
pin: true
math: true
mermaid: true
---

##### 问题背景：
在搭建的Java转发服务程序中，发现推送消息的JSON对象中，原本是中文的内容出现乱码。通过PL/SQL和Navicat连接查看一切正常
![示意图](/assets/img/2026-05-20/095f589f471e522f5e2dd19fdce25cbe.png)

##### 排查步骤：
查询了数据库的编码格式，发现使用了US7ASCII作为字符集编码，Oracle数据库版本为11g。那么在读取数据时，设置为对应的编码格式读取是否就能正常读出来了呢？
很遗憾，Oracle无法通过`characterEncoding=US7ASCII`这样的形式来指定编码为US7ASCII。由于这里使用了多数据源配置，那么通过配置环境变量来指定JDBC通过对应编码读取数据，并不适用。
![示意图](/assets/img/2026-05-20/Pasted image 20260520163921.png)
查询了一些资料，发现有这样的解决方式：
通过在application.yml中，配置Druid数据源
```yml
datasource:  
  his:
    url: jdbc:oracle:thin:@localhost:1521/spectra  
    username: your_username
    password: your_password
    driver-class-name: oracle.jdbc.OracleDriver  
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      # 配置监控统计拦截的filters，用于转码
      filters: stat,wall,slf4j,encoding
      # 在创建连接时指定编码属性
      connection-properties: serverEncoding=ISO-8859-1;clientEncoding=GBK
```
并且降低JDBC依赖版本，使用了与11g对应的ojdbc6
```xml
<dependency>
    <groupId>com.oracle.database.jdbc</groupId>
    <artifactId>ojdbc6</artifactId>
    <version>11.2.0.4</version>
</dependency>
```
重新启动服务后，发现该问题依旧存在。这里猜测是由于Oracle JDBC不支持读取这样的写法，JDBC客户端使用的编码，是由 JVM 环境变量 `NLS_LANG` 或Driver自动映射决定的。
##### 解决方案：
于是考虑是否能在ORM框架层进行处理，对字符串进行编码转换后，再交给业务层，对业务的侵入性较低。以下是通过Mybatis提供的`TypeHandler`，在读取时进行处理。

```java
@Slf4j
// @Component 如果需要全局启用，使用该注解实例化。此处由于我使用了多数据源，不能够进行全局转换
@MappedTypes(String.class)
@MappedJdbcTypes(JdbcType.VARCHAR)
public class StringTypeHandlerConfig extends BaseTypeHandler<String> {

    /**
     * 将对请求入参进行转码（涉及的主要方法）
     */
    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType) throws SQLException {
        parameter = StringUtil.GBKtoISO(parameter);
        ps.setString(i, parameter);
    }

    /**
     * 将返回结果转码（涉及的主要方法）
     */
    @Override
    public String getNullableResult(ResultSet rs, String columnName) throws SQLException {
        try {
            if (columnName != null && !columnName.isEmpty()) {
                InputStream inputStream = rs.getAsciiStream(columnName);
                if (inputStream != null) {
                    String result = convertStreamToString(inputStream, "GBK");
                    // log.info("{}======>{}", columnName, result);
                    return result;
                }
            }
        } catch (IOException e) {
            log.error("转换异常：", e);
        }
        return rs.getString(columnName);
    }


    @Override
    public String getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        try {
            InputStream inputStream = rs.getAsciiStream(columnIndex);
            if (inputStream != null) {
                String result = convertStreamToString(inputStream, "GBK");
                // log.info("columnIndex {}======>{}", columnIndex, result);
                return result;
            }
        } catch (IOException e) {
            log.error("转换异常：", e);
        }
        return rs.getString(columnIndex);
    }

    @Override
    public String getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        return cs.getString(columnIndex);
    }

    private String convertStreamToString(InputStream inputStream, String encoding) throws IOException {
        if (inputStream == null) {
            return "";
        }
        BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream, encoding));
        StringBuilder sb = new StringBuilder();
        String line;
        while ((line = reader.readLine()) != null) {
            sb.append(line);
        }
        reader.close();
        return sb.toString();
    }

}
```
这里由于使用了多数据源，并且只有特定一个数据源需要进行转换，直接在Mapper层进行了手动指定转换String类型字段。
![示意图](/assets/img/2026-05-20/Pasted image 20260520174710.png)
转换后的数据：
![示意图](/assets/img/2026-05-20/Pasted image 20260520175057.png)