---
layout: post
title:  Spring Cloud 入门与Ribbon的使用
category: [springcloud]
tags: [Spring Cloud]
modified: 2021-03-26
---

# Dubbo和Spring Cloud技术选型

1. 架构完整度
   1. dubbo only provide service register和服务治理

2. 社区活跃度
   1. spring cloud全世界都在用，活跃度很高。
3. 通讯协议
   1. dubbo使用rpc（底层是netty）（基于ip，port，osi第三第四层）
   2. Spring Cloud使用的HTTP REST（osi第七层），通讯效率高？
   3. Spring Cloud也有gRpc（google提供的）
   4. Spring Cloud Dubbo。
4. 技术改造与微服务开发
   1. 原有项目改造为微服务，涉及改动较少，可以选择dubbo。spring cloud的改造=重写。





##Eureka VS ZooKeeper

### What is Eureka？

- Netfilex公司开发
- 基于http rest
- Sping Cloud 封装了Eureka，二次封装
- C/S 客户端 服务端架构，服务端作为注册中心，客户端连接到服务端，维持心跳连接。
- 客户端是一个java客户端，用来简化与服务器的交互、负载均衡，故障切换等。（以jar包的形式存在，可以放入项目）

###与zookeeper区别

前提：CAP理论 Consistence,Availabile,Partition Tolerance.

- zookeeper保持CP，而eureka保证AP。
  - zookeeper采用leader election，有时候会不可用。
  - Eureka的节点是平等的，节点的挂掉，不影响整体，但是不保证强一致性。

##Spring Cloud 与Eureka的配置与搭建

创建一个注册中心模块，引入server模块，标注application为eureka server，配置对应端口和注册中心地址。

同理，将service provider做类似操作。

## RestTemplate+固定ip和port调用—》升级为根据app名调用（Ribbon支持）

之前怎么搞的？

BeanConfig

```java
@Configuration
public class BeanConfig {
    @Bean
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }
}
```



Controller

```java
    @Autowired
    RestTemplate restTemplate;

    @RequestMapping("/web/hello")
    public String hello(){
        //逻辑处理忽略

        //借助restTemplate远程调用，ip和端口是写死的。
        return restTemplate.getForEntity("http://localhost:9100/service/hello",String.class).getBody();
    }
```



如何升级？（从固定ip和port，升级为根据应用名进行远程服务调用）

⬆️ans：通过负载均衡来实现（load balance）

在BeanConfig中加入@LoadBalanced（由Ribbon支持）

```java
@Configuration
public class BeanConfig {
    @LoadBalanced
    @Bean
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }
}
```

有了Ribbon的支持之后，调用不需要写死的IP和端口，仅需要服务的名称即可。

##Load Balance

### 负载均衡的种类

- 硬件方法：F5、深信服、Array
- 软件方法：Nginx、LVS、HAProx

###负载均衡的原理：

1. 通过心跳检测来剔除故障节点
2. 维护一个可用的服务端清淡
3. 接收到客户端请求时，通过轮训、权重、最小连接数等算法分配服务器节点

##What is Ribbon？

核心任务是调用服务

- 服务的发现由eureka client实现，但是服务的调用实际由Ribbon完成。

原理/本质是一个基于HTTP和TCP的**客户端**负载均衡（Nginx是服务端负载均衡）

- 通过维护一个服务清单来实现服务的调用：当Ribbon对服务进行访问的时候，它会扩展Eureka客户端的服务发现功能，实现从Eureka注册中心**获取服务端列表**，并通过Eureka客户端来确定服务端是否已经启动。

Ribbon的使用

- Spring Cloud对RIboon进行了二次封装，可以让我们使用RestTemplate的服务请求，自动转换成客户端负载均衡的服务调用。
- 所以在使用时，Ribbon主要和RestTemplate对象配合使用，Ribbon会自动化配置RestTemplate对象，通过@LoadBalanced开启RestTemplate对象调用时的负载均衡。
- 总结一下步骤，仅仅2步：
  1. 服务注册到注册中心（常规操作）
  2. 在消费者中使用@LoadBalanced修饰RestTemplate来调用远程调用步骤1中注册的服务

Ribbon的方便之处

1. RIbbon支持多种负载均衡算法，还支持自定义的负载均衡算法。
2. Ribbon只是一个工具类的框架，小巧，Spring Cloud对其进行封装后使用方便，不像服务注册中心、配置中心、API网关需要独立配置，Ribbon可以直接在代码中使用。

Ribbon和Nginx的区别：

1. Ribbon是客户端的负载均衡，Nginx是服务端的负载均衡。区别在于服务端清单的存储位置不同。
   - 因为Ribbon是客户端的负载均衡，所以获取服务端清单，需要取服务注册中心去获取，比如Eureka Registry。

2. Ribbon作为客户端的负载均衡，和Nginx这样服务端的负载均衡的架构也具有相似之处，都需要**心跳**取维护服务端清单的健康，这个步骤需要注册中心的配合。Spring Cloud中对Ribbon进行了二次封装，默认进行自动化整合配置。

### Ribbon的负载均衡策略

默认是RoundRobin（轮询）

![image-20210325195123610](https://cdn.jsdelivr.net/gh/henggao98/imgbed/posts/image-20210325195123610.png)

1. 随机RandomRule

2. CilentConfigEnabledRoundRobinRule

   ​	—BestAvailableRUle

   ​	—PredicateBaseRule

   ​		—AvailabilityFilteringRule

   ​		—ZoneAvoidanceRule

3. 轮询RoundRobinRule（默认）

   ​	—WeightedResponseTimeRule

4. RetryRule

总结：

| 1                         | 2                                                            |
| ------------------------- | ------------------------------------------------------------ |
| Random                    | 随机                                                         |
| AvailabilityFilteringRule | 先过滤掉多次访问故障的服务，以及并发连接数超过阈值的服务，然后对剩下的服务进行轮询 |
| WeightedResponseTimeRule  | 按照平均响应时间来计算所有服务的权重，响应速度快的服务将获得更大权重，基于统计数据决定性能最优服务。特殊的地方是，当服务刚启动时统计数据不足，则采用RoungRobin进行轮询。 |
| Retry                     | 先按照RoundRobinRule，若不能访问，则在指定时间内进行重试（分配其他服务） |
| BestAvailabilityRule      | 过滤掉多次访问故障的服务，并选择并法连接数量最小的服务。     |
| ZoneAvoidanceRule         | 综合判断服务节点所在区域的性能和服务节点的可用性来决定选择哪个服务。 |



## RestTemplate的基本使用

###使用restTemplate.getForEntity()

首先我们在consumer中写一个controller拦截来自web的请求。

然后在controller中使用restTemplate进行远程服务调用，同时由2种参数的传递方法：

方法1：使用String Array来传递参数

```java
远程服务调用--带参数-01-使用array传递参数
        String[] strArray = {"105","Mike","13862355555"};
        ResponseEntity<User> responseEntity = restTemplate.getForEntity("http://01-SPRINGCLOUD-SERVICE-PROVIDER/service/getUser?id={0}&name={1}&phone={2}",User.class, (Object[]) strArray);
```

方法2：使用map传递参数

```java
  //远程服务调用--带参数-02-使用map传递参数
        Map<String, Object> paramMap = new ConcurrentHashMap<>();
        paramMap.put("id", 105);
        paramMap.put("name", "Jake");
        paramMap.put("phone", "13862355555");
        ResponseEntity<User> responseEntity = restTemplate.getForEntity("http://01-SPRINGCLOUD-SERVICE-PROVIDER/service/getUser?id={id}&name={name}&phone={phone}",User.class, paramMap);

```

###使用restTemplate.getForObject()

直接返回Body，并转化为对象，等价于restTemplate.getForObject().getBody();



| 传参方法                                                     | GET  | POST | PUT  | DELETE |
| ------------------------------------------------------------ | ---- | ---- | ---- | ------ |
| 普通MAP<String,Object> = new ConcurrentHashMap<String, Object>(); | Y    | N    | N    | Y      |
| MultiValueMap<String,Object> = new LinkedMultiValueMap<String, Object>(); | N    | Y    | Y    | N      |
| String[] strArray= {“id”,”name”,”phone"}                     | Y    | N    | N    | Y      |



##RestTemplae使用实例

####@GET

```java
@GetMapping("/web/getUser")
    public User getUser(){
        //逻辑判断
        //省略

        //远程服务调用--带参数-01-使用array传递参数
        //String[] strArray = {"105","Mike","13862355555"};
        //ResponseEntity<User> responseEntity = restTemplate.getForEntity("http://01-SPRINGCLOUD-SERVICE-PROVIDER/service/getUser?id={0}&name={1}&phone={2}",User.class, (Object[]) strArray);

        //远程服务调用--带参数-02-使用map传递参数
        Map<String, Object> paramMap = new ConcurrentHashMap<String, Object>();
        paramMap.put("id", 105);
        paramMap.put("name", "Jake");
        paramMap.put("phone", "13862355555");
        ResponseEntity<User> responseEntity = restTemplate.getForEntity("http://01-SPRINGCLOUD-SERVICE-PROVIDER/service/getUser?id={id}&name={name}&phone={phone}",User.class, paramMap);

        int responseCode = responseEntity.getStatusCodeValue(); // 响应状态码的值 int
        User body = responseEntity.getBody(); // 响应体
        HttpStatus httpStatus = responseEntity.getStatusCode(); //状态码 HTTP_STATUS_CODE
        HttpHeaders httpHeaders = responseEntity.getHeaders(); // 头 header

        return body;

    }
```

####@POST

```java
 @RequestMapping("/web/addUser")
    public User addUser(){

        //post方法，使用表单传递参数(注意，此处不能使用hash Map）
        MultiValueMap<String,Object> dataMap = new LinkedMultiValueMap<String, Object>();
        dataMap.add("id", 199);
        dataMap.add("name", "Elise");
        dataMap.add("phone", "15200099888");
        //不再使用URL绑定参数，但是需要增加一个RequestBody，来传递参数。
        //此处URL中还是可以继续使用map或者array来绑定参数的，只是此处不用。
        ResponseEntity<User> responseEntity = restTemplate.postForEntity("http://01-SPRINGCLOUD-SERVICE-PROVIDER/service/addUser",dataMap,User.class);

        int responseCode = responseEntity.getStatusCodeValue(); // 响应状态码的值 int
        User body = responseEntity.getBody(); // 响应体
        HttpStatus httpStatus = responseEntity.getStatusCode(); //状态码 HTTP_STATUS_CODE
        HttpHeaders httpHeaders = responseEntity.getHeaders(); // 头 header

        return body;
    }
```

####@PUT

```java
 @RequestMapping("/web/updateUser")
    public String updateUser(){

        MultiValueMap<String,Object> dataMap = new LinkedMultiValueMap<String, Object>();
        dataMap.add("id", 199);
        dataMap.add("name", "Elise");
        dataMap.add("phone", "15200099888");

        //PUT方法没有返回值
        restTemplate.put("http://01-SPRINGCLOUD-SERVICE-PROVIDER/service/updateUser",dataMap);

        return "Success";
    }

```

#### @DELETE

```java
    @RequestMapping("/web/deleteUser")
    public String deleteUser(){

        //不再需要用这个map了，这个map
//        MultiValueMap<String,Object> dataMap = new LinkedMultiValueMap<String, Object>();
//        dataMap.add("id", 199);
//        dataMap.add("name", "Elise");
//        dataMap.add("phone", "15200099888");

        //PUT方法没有返回值
        Map<String, Object> paramMap = new ConcurrentHashMap<String, Object>();
        paramMap.put("id", 105);
        paramMap.put("name", "Jake");
        paramMap.put("phone", "13862355555");
        restTemplate.delete("http://01-SPRINGCLOUD-SERVICE-PROVIDER/service/deleteUser?id={id}&name={name}&phone={phone}",paramMap);

        //String[] strArray = {"105","Mike","13862355555"};
        //ResponseEntity<User> responseEntity = restTemplate.deleteForEntity("http://01-SPRINGCLOUD-SERVICE-PROVIDER/service/deleteUser?id={0}&name={1}&phone={2}",User.class, (Object[]) strArray);

        return "Success";
    }
```



## Feign

Feign和Ribbon是类似的

- ribbon面向服务名
- feign面向接口，当然还有注解啦，更符合java开发。