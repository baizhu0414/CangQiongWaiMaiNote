# cqwm
## cqwm开发流程介绍

1. 前端请求地址和后端提供服务地址不同：NGINX
    - 原因：这个依赖于nginx的反向代理，将请求转发到后端。
        比如：[localhost:80/](http://localhost/api/employee/login) --nginx-> localhost:8080/admin/employee/login ---> tomcat服务器
    > nginx好处：1）有缓存，速度快；2）nginx负载均衡；3）保证后端安全。
    > nginx反向代理：proxy_pass; 
    > 
    > 负载均衡（upstream）策略：
    > - 轮询
    > - 权重（weight）：权重越高，被分配的客户端请求就越多
    > - 依据ip分配方式（ip hash）：这样每个访客可以固定访问一个后端服务
    > - 依据最少连接方式（least conn）：把请求优先分配给连接数少的后端服务
    > - 依据url分配方式（url hash）：这样相同的url会被分配到同一个后端服务
    > - 依据响应时间方式（fair）：响应时间短的服务将会被优先分配
    > 

2. 接口测试方式：Swagger
    - 原因：postman构造参数效率低
    - `Knife4j封装了Swagger`生成Api文档的解决方案。配置类**class WebMvcConfiguration extends WebMvcConfigurationSupport**。
    - 通过 Docket 对象配置 Swagger/Knife4j 接口文档
    
        一、Swagger/Knife4j 接口文档配置（定义Bean方法返回Docket）
        1. 自定义 API 文档元信息（ApiInfo），也就是文档首页信息。
        2. 控制接口扫描范围（apis 和 paths）
        3. groupName+定义多个Docket函数扫描admin和user文件目录，从而区分用户端和商户端
        4. 高级配置（如接口鉴权、全局参数）：可通过 Docket 添加全局请求头（如 Token 认证）、设置文档开关（生产环境关闭）等。

        二、静态资源映射（addResourceHandlers 方法）

        三、WebMvcConfigurationSupport可扩展功能
        5. 跨域配置（解决前后端分离跨域问题）
        6. 消息转换器（自定义 JSON 序列化/反序列化）
        7. 视图控制器（简化页面跳转配置）:无需通过 Controller 即可直接映射路径到页面.

        四、其他概念
        1. '@Configuration' 注解是 Spring 框架的核心注解之一，用于标识一个类为 配置类，其主要作用是 声明 Spring 应用上下文的配置信息，包括定义 Bean、注册组件、配置依赖关系等。`被 @Configuration 标记的类中，可使用 @Bean 注解定义方法，这些方法的返回值会被 Spring 注册为容器中的 Bean。`
           - `@ConfigurationProperties`可以用于配置参数的获取AliOssProperties，然后通过`@Configuration+@Bean`返回配置参数工具类。配合`@ConditionalOnMissingBean`获取配置相关的实例类比如AliOssUtil。
           - `@Configuration+@Bean`配置RedisConfiguration, 返回RedisTemplate。
        2. 在java中，继承：
        * 关于 private 方法：不可继承，更无法重写。
        * 关于 protected 方法：可继承，可重写。
        * 关于 static 方法：不可重写，但可“隐藏”.静态方法不支持多态，按声明类型 Parent 调用.
        1. Entity（实体类）：数据库表映射；DTO（Data Transfer Object：数据传输对象）：接口入参/层间数据传递；VO（Value Object：值对象）：后端→前端的数据响应，封装前端视图所需的数据。
      
    

    - 常用注解

| 注解            | 说明                                                                 |
|-----------------|:----------------------------------------------------------------------|
| @Api            | 用在类上，例如Controller，表示对类的说明                             |
| @ApiModel       | 用在类上，例如entity、DTO、VO                                        |
| @ApiModelProperty | 用在属性上，描述属性信息                                            |
| @ApiOperation   | 用在方法上，例如Controller的方法，说明方法的用途、作用               |
    > **注意：**
    > - 设计阶段可以将接口设计文档导入yapi进行统一管理；
    > - 开发阶段有了Knife4j（Swagger）就可以辅助后端人员进行开发测试。

## 常用知识介绍(SpringBoot/MyBatis)

1. 常用注解相关类
    - @Data、@Builder、@NoArgsConstructor、@AllArgsConstructor：`Employee`
    - @Configuration、@Bean、@Slf4j：`WebMvcConfiguration`
    - @ConfigurationProperties：`JwtProperties`
    - @RestControllerAdvice vs @ControllerAdvice、@ExceptionHandler、@Controller vs @RestController、：`GlobalExceptionHandler`
    - @Mapper：`EmployeeMapper`；
        - 分页查询结合PageHelper库更方便，PageHelper会让下一个查询返回的List自动被Page对象代理，原理是使用了mybatis拦截器，这样后面分页查询的sql语句也不需要limit关键字了，拦截器会自动拼接语句。https://www.bilibili.com/list/watchlater/?spm_id_from=333.1007.view_later.pip&bvid=BV1TP411v7v6&oid=315565756&p=22
    - 

2. 请求参数接收方式
   - @RequestBody：接收请求体中的 JSON/XML 等数据
   - @RequestHeader: 接收 HTTP 请求头中的参数
   - @RequestParam或直接获取：Query 参数,**URL 中/app/msg?id=123后的键值对，直接用对象接受参数可行**。如果是字符串，能够自动解析:'1,2,3' -> 'List<Long>'
   - @PathVariable：接收 URL 路径中的动态参数（也叫Query请求，如: `@RequestMapping("/user/{id}/detail"), @PutMapping("/{status}")`）
   - 

3. 小需求：
   > 1. JWT认证流程：由于每次请求tomcat服务器都会单独分配一个线程，Interceptor、controller、service都在此线程中。因此可以考虑使用ThreadLocal保存用户Id（Interceptor中），然后再在service中取出来使用。`JwtTokenAdminInterceptor`、`BaseContext（ThreadLocal）`
   > 2. 数据库存储LocalDateTime日期数据直接读取是数组对象，想要读出来之后保持日期格式，可以通过：1）`@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")`注解或者；2）SpringMvcConfig中扩展消息转换器`对象序列化和反序列化JacksonObjectMapper`。
   ---
   > 3. 定制化更新数据库参数的mybatis语句：
    ```xml
    <update id="update" parameterType="com.sky.entity.Employee"> // 参数类型Employee，自动转换。
        update employee
        <set>
            <if test="name != null">name = #{name},</if>
            <if test="username != null">username = #{username},</if>
            <if test="password != null">password = #{password},</if>
            <if test="phone != null">phone = #{phone},</if>
            <if test="sex != null">sex = #{sex},</if>
            <if test="idNumber != null">id_Number = #{idNumber},</if>
            <if test="updateTime != null">update_Time = #{updateTime},</if>
            <if test="updateUser != null">update_User = #{updateUser},</if>
            <if test="status != null">status =#{status},</if>
        </set>
        where id =#{id}
    </update>
    ```
    ---
   > 4. 针对每次不同表的数据更新，都涉及一些更新时间、修改人等统一数据，使用AOP切面实统一意拦截和处理。
   >    需要注意：insert需要修改创建人、创建时间、修改人、修改时间；update仅仅需要更新修改人、修改时间字段。
   >    涉及枚举、AOP编程、注解、反射。
   > 5. 想要操作insert产生的id，需要在Mapper.xml中这么写： `<insert id="insert" useGeneratedKeys="true" keyProperty="id">`
   >    比如insert(user),那么插入后user.id会被设置为数据库新产生的id。

4. AOP编程
    1）自定义注解AutoFill；
    2）定义切面类AutoFillAspect拦截注解的方法，通过反射给字段赋值；
        - 定义：切入点、通知（@Aspect+ @Component）
        - 切入点：
            ```java
                @Pointcut("execution(* com.sky.mapper.*.*(..)) && @annotation(com.sky.annotation.AutoFill)")
                public void autoFillPointcut(){}
            ```
        - 通知分类：`@Before("autoFillPointcut()")`前置通知
          - 内部包含反射获取参数（object）、函数（updatexxx）、注解的参数类型（insert|update）
    3）在相应数据库Mapper类方法上添加AutoFill注解。

5. 小需求：新增菜品
    1) 文件上传，返回服务器地址： @RequestMapping可以用于注解方法，但是@PostMapping只能用于注解方法。
        > MultipartFile 的坑点主要集中在：大小限制（Spring+Nginx）、临时存储(文件不能重复读取)、文件名中文编码（上传charset+URLEncoder设置charset）、name与RequestParam对齐、服务器配置（跨域需要后端配置或者CDN代理）和安全性（MIME+文件大小+`UUID命名防止文件名冲突`）。
    2) 数据库操作：同时操作两个数据库，菜品和口味，`@Transactional`注解。前提：已经在Application中开启注解`@EnableTransactionManagement`.
        - 批量插入数据库数据：
        ```xml
            <foreach collection="users" item="user" separator=",">
                (#{user.name}, #{user.age}, #{user.email})
            </foreach>
        ```
    3）子需求：菜品信息查询（多表查询出现重复字段需要起别名，保证结果字段名称和封装类名称对应），此处：// 匹配返回值DishVO中的字段名称name和category_name

    ```xml
    select d.*, c.name as category_name
    from dish d 
    left outer join category c on d.category_id = c.id
    <where>
    <if test="name != null">
        and d.name like concat('%', #{name}, '%')
    </if>
    <if test="categoryId != null">
        and d.category_id = #{categoryId}
    </if>
    <if test="status != null">
        and d.status = #{status}
    </if>
    </where>
    order by d.create_time desc

    ```


## Redis使用介绍

1. 入门
    > 基于内存的key-val结构数据库。适合用于存储热点数据、提高读写性能。提供 过期策略、事务、发布订阅、分布式锁 等。
    windows安装包下载地址（x64/zip）：https://github.com/microsoftarchive/redis/releases

    - 5种数据类型:字符串、哈希、列表、集合、有序集合、
    - 常用命令
    1. `redis-server.exe xxx.conf`
    2. `redis-cli.exe -h localhost -p 6379`
    3. 数据类型相关命令
        - 字符串操作：SET key value/ GET key/ SETEX key seconds value/ SETNX key value(key不存在才设置，分布式锁)
        - 哈希表操作：hset table field value/ hget table field/ hdel table field/ hkeys table/ hvals table；
        - 列表操作：lpush list value1 [value2]/ lrange list start stop（闭包）/ rpop list/ llen list；
        - 集合操作：sadd set member1 [member2]/ smembers set（所有成员）/ scard set（成员数）/ sinter set1 [set2] （交集）/ sunion set1 [set2](并集)/ srem set member1 [member2](删除)；
        - 有序集合操作：zadd scoreL score1 member1 [score2 member2]/ zrange scoreL start stop [withscores]/ zincrby scoreL increment member(指定成员加分)/ zrem scoreL member [member2](删除)。
    4. keys pattern(符合给定模式的所有key)/ exists key/ type key/ del key1 [key2]
    - Java操作Redis(交互)
    1. 使用Spring Data Redis操作redis，maven坐标是：spring-boot-starter-data-redis;
    2. 配置：
    ```xml
    spring:
      redis:
        host:
        port:
        password:
        database:
    ```
    3. 编写配置类，Bean函数返回RedisTemplate（设置Key序列化，否则查询不到key）
    - 小需求开发（店铺营业状态设置）
    - 如果用户user.ShopController和商家的admin.ShopController没有重命名则会发生Spring容器冲突(名称是shopController)。
      可以通过@RestController("adminShopController").

## 微信登陆流程
> 用户获取code，并发送到后端请求登录->后端携带code、appid等信息请求微信服务器->微信服务器返回openid等信息->后端储存用户信息包括openid，并生成JWT令牌->返回给前端->登陆成功

7-2
