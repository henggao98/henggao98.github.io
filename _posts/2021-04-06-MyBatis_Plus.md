---
layout: post
title:  Spring Boot注解基础及Mybatis Plus入门与拓展
category: [framework]
tags: [mybatisplus]
modified: 2021-04-06
---

# Spring Boot注解基础及Mybatis Plus入门与拓展

Spring Boot注解基础回顾 + 过一遍Mybatis Plus + 拓展

## Spring Boot配置之注解类

yml文件

```xml
#配置类
school:
  name: "School Name 1"

my:
  name: "my Name"

```

配置类

```java
@Component
@ConfigurationProperties(prefix = "school")
public class School {

    private String name;

    private String website;
```

controller

```java
    @Autowired
    private School school;

    @Value("${my.name}")
    private String myName;

    @RequestMapping("/hello")
    public @ResponseBody
    String hello(){
        return "Hello" + school.getName() + myName;
    }
```

### 中文乱码问题

修改idea中file encoding

## MyBatis和MyBatis-PLUS

### JDBC配置

```xml
datasource:
  username: root
  password: 123456
  url: jdbc:mysql://localhost:3306/数据库名?useSSL=false&useUnicode=true&characterEncoding=utf-8&serverTimezone=GMT%2B8
  driver-class-name: com.mysql.cj.jdbc.Driver
```

- useSSL=false&useUnicode=true
- &characterEncoding=utf-8
- &serverTimezone=GMT%2B8  (mysql 8.x 还需要设置时区）)
- driver-class-name: com.mysql.cj.jdbc.Driver (mysql 8.x)
- driver-class-name: com.mysql.jdbc.Driver（mysql 5.x)

### mybatis传统流程

Pojo-dao(连接mybatis-配置mapper.xml)-service-controller

### plus升级后的流程

- pojo

  ```java
  //自动生成getter和setter
  @Data
  //自动生成有参构造
  @AllArgsConstructor
  //自动生成无参构造
  @NoArgsConstructor
  public class User {
      private Long id;
      private String name;
      private Integer age;
      private String email;
  }
  ```

- mapper接口

  ```java
  @Repository //代表这是持久层的 DAO层
  //使用了mybatis-plus，只需要继承BaseMapper的接口就行
  public interface UserMapper extends BaseMapper<User> {
  }
  ```

- 直接使用

  ```java
  @Autowired
  private UserMapper userMapper;
  
  @Test
  void contextLoads() {
      System.out.println(("----- selectAll method test ------"));
      List<User> userList = userMapper.selectList(null);
      Assertions.assertEquals(5,userList.size());
  
      userList.forEach(System.out::println);
  }
  ```

## 配置日志

```xml
# set to simple console log output
log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

结果![image-20210407130052514](https://cdn.jsdelivr.net/gh/henggao98/imgbed/posts/image-20210407130052514.png)

## CRUD 拓展

### insert测试

```java
@Test
public void testInsert(){
    User user = new User();
    user.setName("username1");
    user.setAge(5);
    user.setEmail("123123@xx.com");
    int result = userMapper.insert(user);

    System.out.println(result); //受影响的行数
    System.out.println(user); //插入对象
	}
```

![image-20210407130911718](https://cdn.jsdelivr.net/gh/henggao98/imgbed/posts/image-20210407130911718.png)

- id是默认生成的

## 主键生成策略

### 常见策略：uuid，自增id，具体见下文。

### 如何生成全局默认id

一种能生成全球唯一id的算法：雪花算法

- 搜索博客“分布式系统唯一id生成方案”： https://www.cnblogs.com/haoxinyue/p/5208136.html

### 雪花算法源码

**即 Twitter的snowflake算法**

相关参数

```java
 private long workerId;
        private long datacenterId;
        private long sequence = 0L;

        private static long twepoch = 1288834974657L;

        private static long workerIdBits = 5L;
        private static long datacenterIdBits = 5L;
        private static long maxWorkerId = -1L ^ (-1L << (int)workerIdBits);
        private static long maxDatacenterId = -1L ^ (-1L << (int)datacenterIdBits);
        private static long sequenceBits = 12L;

        private long workerIdShift = sequenceBits;
        private long datacenterIdShift = sequenceBits + workerIdBits;
        private long timestampLeftShift = sequenceBits + workerIdBits + datacenterIdBits;
        private long sequenceMask = -1L ^ (-1L << (int)sequenceBits);

        private long lastTimestamp = -1L;
        private static object syncRoot = new object();
```

snowflake算法<u>可以根据自身项目的需要进行一定的修改</u>。比如估算未来的数据中心个数，每个数据中心的机器数以及统一毫秒可以能的并发数来调整在算法中所需要的bit数。

- snowflake是Twitter开源的分布式ID生成算法，结果是一个long型的ID。其核心思想是：使用41bit作为毫秒数，10bit作为机器的ID（5个bit是数据中心，5个bit的机器ID），12bit作为毫秒内的流水号（意味着每个节点在每毫秒可以产生 4096 个 ID），最后还有一个符号位，永远是0。具体实现的代码可以参看https://github.com/twitter/snowflake。

优点：

1）不依赖于数据库，灵活方便，且性能优于数据库。

2）ID按照时间在单机上是递增的。

缺点：

1）在单机上是递增的，但是由于涉及到分布式环境，每台机器上的时钟不可能完全同步，也许有时候也会出现不是全局递增的情况。

**最终效果：生成全球唯一id，每个节点每毫秒可以生成4096个id**

### 运用到代码中

```java
//自动生成getter和setter
@Data
//自动生成有参构造
@AllArgsConstructor
//自动生成无参构造
@NoArgsConstructor
public class User {
    @TableId(type = IdType.AUTO)
    private Long id;
    private String name;
    private Integer age;
    private String email;
}
```

```java
    @TableId(type = IdType.AUTO)
```

- IdType是一个mybatisplus提供的类
- 其中包含了各种策略

### IdType

```java
public enum IdType {
    AUTO(0),
    NONE(1),
    INPUT(2),
    ID_WORKER(3),
    UUID(4),
    ID_WORKER_STR(5);

    private int key;

    private IdType(int key) {
        this.key = key;
    }

    public int getKey() {
        return this.key;
    }
```

- auto 自增主键
  1. 实体类中注释配置@TableId(type = IdType.AUTO)
  2. 数据库中需要设置主键类型为自增！
- NONE 为设置主键
- INPUT 手动输入
- ID_WORKER 默认的全局唯一id
- UUID 全局唯一id uuid
  - 很长，无序
- ID_WORK_STRING 字符串表示法，另一个是数字

## sql动态拼接

使用mybatis plus框架，当我们mapper.updateById(user)传入的参数是一个对象时，会根据对象的filed进行动态拼接

- where filed 1= and filed 2 = and filed n =

## 自动填充

例子：自动填充操作时间

方法1:数据库级别-实体类中添加field

private Date createTime；

方法2:代码级别

第一步：在实体类上添加注解

```java
   //字段 添加处理内容
    @TableField(fill = FieldFill.INSERT)
    private  Date createTime;


    @TableField(fill = FieldFill.INSERT_UPDATE)
    private Date updateTime;
```

第二步：编写一个时间处理类

```java
@Slf4j
@Component //将这个包交给IOC容器管理，等于说让这个处理类生效
public class myMetaObjHandler implements MetaObjectHandler {
    @Override
    public void insertFill(MetaObject metaObject) {
        log.info("insert start");

//        setFieldValByName(String fieldName, Object fieldVal, MetaObject metaObject) {
        this.setFieldValByName("createTime", new Date(), metaObject);
        this.setFieldValByName("updateTime", new Date(), metaObject);
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        log.info("update start");
        this.setFieldValByName("updateTime", new Date(), metaObject);
    }
}
```

## 乐观锁

> 乐观锁：顾名思义，十分乐观，出了问题，再次更新测试
>
> 被关锁：十分悲观，假设问题总是会发生，所以做什么事都要上锁。

今天只看乐观锁

乐观锁实现方式：

- 取出记录时，获得当前version

- 更新时带上version

- 执行更新时，set version = new Version where version = oldVersion

- 如果version不变，更新失败

  >
  >
  >乐观锁：1、先查询，获得版本号 version = 1
  >
  >--线程A
  >
  >Update user set name = ’jack’, version = version + 1
  >
  >where id = 2 and version = 1
  >
  >
  >
  >—线程B — 若线程B抢先完成，会导致A失败
  >
  >Update user set name = ’jack’, version = version + 1
  >
  >where id = 2 and version = 1

- **说白了就一句话，乐观锁就是：“操作时带上version”**

> 测试Mybatis Plus的乐观锁插件

### 第一步：在数据库中增加version字段，默认值为1

### 第二步：实体类同步--增加对应字段

```java
@Version
private int version;
```

### 第三步：注册主键

创建myBatisPlusConfig配置类

```java
@Configuration
public class myBatisPlusConfig {

    //注册乐观锁插件
    //有官方文档
    @Bean
    public OptimisticLockerInterceptor optimisticLockerInterceptor(){
        return new OptimisticLockerInterceptor();
    }
}
```

最后：测试一下

## 分页查询

配置拦截器组件即可

```java
//注册分页插件
@Bean
public PaginationInterceptor paginationInterceptor() {
    return new PaginationInterceptor();
}
```

### 删除

```java
//批量删除
@Test
public void testDeleteByBatchIds(){
    userMapper.deleteBatchIds(Arrays.asList(1379669608008589314L,
            1379669608008589315L,
            1379669608008589316L
    ));
}

//批量条件删除
@Test
public void testDeleteByBatchId(){
    HashMap<String,Object> map = new HashMap<>();
    map.put("name","username1");
    userMapper.deleteByMap(map);
}
```

逻辑删除

保留数据库中的数据，仅通过变量来使该数据搜索失效

步骤

1. 数据库增加delted字段，默认为0，表示未删除

2. pojo实体类中增加属性和注解

   ```java
   @TableLogic
   private int deleted;
   ```

3. 加入逻辑删除组件

   ```java
   //注册逻辑删除插件
   @Bean
   public ISqlInjector sqlInjector(){
       return new LogicSqlInjector();
   }
   ```

4. 配置逻辑删除

```xml
global-config:
  db-config:
    logic-delete-value: 1
    logic-not-delete-value: 0
```

- 本质：是更新操作



## 性能分析插件

平时开发中会遇到一些慢sql

可以测试！druid等其他工具

MP当然也提供了性能分析，若**超时则停止**

1. 导入插件
2. 测试使用



## 条件查询 Wrapper

```java
//基础条件查询-根据id查询
@Test
public void wrapperTest02(){
    QueryWrapper<User> wrapper = new QueryWrapper<>();
    wrapper.eq("id", 1379669608008589344L);
    userMapper.selectList(wrapper).forEach(System.out::println);
}

```

```java
//模糊查询
@Test
public void wrapperTest03(){
    //查询20-30岁之间的用户
    QueryWrapper<User> wrapper = new QueryWrapper<>();

    //名字中不包含"J"
    //且以J开头
    wrapper
            .notLike("name", "y")
            .likeRight("name", "M");

    List<Map<String,Object>> maps = userMapper.selectMaps(wrapper);
    maps.forEach(System.out::println);
}
```

```java
//区间查询
@Test
public void wrapperTest04(){
    QueryWrapper<User> wrapper = new QueryWrapper<>();
    wrapper
            .inSql("id", "select id from user where id < 1379669608008589321");

    List<Object> objects = userMapper.selectObjs(wrapper);
    objects.forEach(System.out::println);
}
```



## 代码生成器

第一步 

- 导入mybatisplus的包

- 导入代码生成模版需要的velocity包

  ```xml
      <!-- 模板引擎 -->
          <dependency>
              <groupId>org.apache.velocity</groupId>
              <artifactId>velocity-engine-core</artifactId>
              <version>2.0</version>
          </dependency>
  ```



```java
package com.gh.springboot;

import com.baomidou.mybatisplus.annotation.DbType;
import com.baomidou.mybatisplus.annotation.FieldFill;
import com.baomidou.mybatisplus.annotation.IdType;
import com.baomidou.mybatisplus.generator.AutoGenerator;
import com.baomidou.mybatisplus.generator.config.DataSourceConfig;
import com.baomidou.mybatisplus.generator.config.GlobalConfig;
import com.baomidou.mybatisplus.generator.config.PackageConfig;
import com.baomidou.mybatisplus.generator.config.StrategyConfig;
import com.baomidou.mybatisplus.generator.config.po.TableFill;
import com.baomidou.mybatisplus.generator.config.rules.DateType;
import com.baomidou.mybatisplus.generator.config.rules.NamingStrategy;

import java.util.ArrayList;

/**
 * @author HENG GAO
 * @version 1.0
 * @date 2021/4/7 16:30
 */


public class CodeGenerator {
    // 代码自动生成器
    public static void main(String[] args) {
        // 需要构建一个 代码自动生成器 对象
        AutoGenerator mpg = new AutoGenerator();
        // 配置策略
        // 1、全局配置
        GlobalConfig gc = new GlobalConfig();
        String projectPath = System.getProperty("user.dir");
        gc.setOutputDir(projectPath+"/src/main/java");
        gc.setAuthor("Heng Gao");
        gc.setOpen(false);
        gc.setFileOverride(false); // 是否覆盖
        gc.setServiceName("%sService"); // 去Service的I前缀
        gc.setIdType(IdType.ID_WORKER);
        gc.setDateType(DateType.ONLY_DATE);
        gc.setSwagger2(true);
        mpg.setGlobalConfig(gc); //开启swagger
        //2、设置数据源
        DataSourceConfig dsc = new DataSourceConfig();
//        String dBName = "classicmodels";
        dsc.setUrl("jdbc:mysql://localhost:3306/classicmodels?useSSL=false&useUnicode=true&characterEncoding=utf-8&serverTimezone=GMT%2B8");
        dsc.setDriverName("com.mysql.cj.jdbc.Driver"); //mysql 8.x 驱动
        dsc.setUsername("root");
        dsc.setPassword("123456");
        dsc.setDbType(DbType.MYSQL);
        mpg.setDataSource(dsc);
        //3、包的配置
        PackageConfig pc = new PackageConfig();
        pc.setModuleName("springboot");
        pc.setParent("com.gh");
        pc.setEntity("entity");
        pc.setMapper("mapper");
        pc.setService("service");
        pc.setController("controller");
        mpg.setPackageInfo(pc);
        //4、策略配置
        StrategyConfig strategy = new StrategyConfig();

        strategy.setInclude("customers"); // 设置要映射的表名
        strategy.setNaming(NamingStrategy.underline_to_camel);
        strategy.setColumnNaming(NamingStrategy.underline_to_camel);
        strategy.setEntityLombokModel(true); // 自动lombok；
        strategy.setLogicDeleteFieldName("deleted");
        // 自动填充配置
        TableFill gmtCreate = new TableFill("gmt_create", FieldFill.INSERT);
        TableFill gmtModified = new TableFill("gmt_modified", FieldFill.INSERT_UPDATE);
        ArrayList<TableFill> tableFills = new ArrayList<>();
        tableFills.add(gmtCreate);
        tableFills.add(gmtModified);
        strategy.setTableFillList(tableFills);
        // 乐观锁
        strategy.setVersionFieldName("version");
        strategy.setRestControllerStyle(true);
        strategy.setControllerMappingHyphenStyle(true); //localhost:8080/hello_id_2
        mpg.setStrategy(strategy);
        mpg.execute(); //执行
    }
}

```

