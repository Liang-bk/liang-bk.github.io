---
title: 'javaweb 笔记'
publishDate: '2025-10-20'
updatedDate: '2025-10-23'
description: ''
tags:

  - javaweb
  - notes
language: '中文'
---

# javaweb

## maven

1. 管理jar包
2. 跨平台构建
3. 标准化目录结构

### 配置

1. 下载地址[Download Apache Maven – Maven](https://maven.apache.org/download.cgi)，解压目录不要有中文

2. `apache-maven-xxx/conf/settings.xml`修改：

   ```xml
   <!--找到localRepository， 添加本地仓库地址-->
   <localRepository>xxx\apache-maven-3.9.11\repo</localRepository>
   
   <!--mirrors 标签中新增标签mirror，为阿里云-->
   <mirror>
       <id>alimaven</id>
       <name>aliyun maven</name>
       <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
       <mirrorOf>central</mirrorOf>
   </mirror>
   ```

3. `apache-maven-xxx/bin`添加到环境变量中

### idea集成

#### 创建maven项目

`setting`->`Build...`->`Build Tools`->`Maven`

- 修改`maven home path`为maven根目录
- 修改`setting file`为`setting.xml`文件
- 修改`Local repository`为设置的本地仓库文件夹

创建空项目后，新创建`module`，`build system`选择maven即可

#### maven坐标

```xml
<groupId>org.example</groupId>
<artifactId>maven-project01</artifactId>
<version>1.0-SNAPSHOT</version>
```

1. groupId：项目隶属组织名称（域名反写）
2. artifactId：项目名称
3. version：项目版本号

这三项在后面配置依赖时会用到

#### 导入maven项目

1. 将文件夹复制到空项目中
2. `File`->`Project Structures`->`Modules`->`+`->`import module`->`选择文件夹中的pom.xml`文件

### 依赖管理

#### 依赖

[依赖下载官网](https://mvnrepository.com/)

在`<dependencies>`标签下增加`<dependency>`标签，然后刷新xml文件即可

```xml
<dependencies>
    <!-- https://mvnrepository.com/artifact/org.springframework/spring-context -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>6.1.4</version>
    </dependency>
</dependencies>
```

#### 排除依赖

在对应的`<dependecy>`标签内增加`<exclusions>`标签：

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>6.1.4</version>
    <!--排除依赖-->
    <exclusions>
        <exclusion>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-observation</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

#### 依赖范围

在`<dependency>`标签内的maven坐标下添加`<scope>`标签，标签值和作用如下：

![](./maven_scope.png)

Y表示可以使用，-表示不行

比如`<scope>test</scope>`表示让某一个依赖包只能在项目的test包下使用，其他包不能使用

### 生命周期

三个周期相互独立

1. clean：清除之前编译的结果
2. default：编译->测试->打包->安装 （执行后一个前会先执行前一个）
3. site：上线

直接在idea的maven面板中的`LifeCycle`运行或者在模块目录下运行以下命令：

```bash
mvn [clean/compile/test/pacakge/install]
```

## JUnit（单测）

### 使用

`pom.xml`添加依赖：

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.9.1</version>
</dependency>
```

类名：`xxxTest`

位置：`test/java/com.xxxx/xxxTest`

方法：必须是`public void`，加上注解`@Test`

```java
public class MainTest {
    @Test
    public void test() {
        
    }
}
```

### 断言

在`Assertions`包，最后一个参数通常设置为错误提示信息（String）

文档：[Assertions (JUnit 5.0.1 API)](https://docs.junit.org/5.0.1/api/org/junit/jupiter/api/Assertions.html)

### 常见注解

#### 环境准备和销毁

- `@BeforeAll`：方法必须为static，通常作为setup初始化资源，在所有测试启动前仅运行一次
- `@AfterAll`：方法必须为static，通常用于释放资源，在所有测试完成后仅运行一次
- `@BeforeEach`：通常作为setup初始化资源，在每个测试启动前都运行一次
- `@AfterEach`：通常用于释放资源，在每个测试完成后都运行一次

#### 参数化测试

同时使用下面两个注解：

- `@ParameterizedTest`：标识方法会使用多个参数进行测试
- `@ValueSource([])`：在()内添加一个数组来指明使用的参数

#### 设置测试名

`@DisplayName("xxx")`：显示测试类/方法名为"xxx"

## Spring Boot

### 创建

idea可以自动化配置spring boot：

`new module`->`Spring Boot`->`Type: Maven`->`设置项目名与jdk版本`->`next`

->`选择Spring boot版本`->`Dependency:Web:Spring Web`

#### 启动入口

`main/java/xxx/SpringbootxxxApplication`中的main方法，带有`@SpringBootApplication`注解

#### 请求注解

1. `@RestController`：类注解，标识其是一个请求处理类
2. `@RequestMapping("")`：方法注解，填写对应的请求路径，标识其是该请求路径对应的处理方法

```java
@RestController
public class HelloController {
    @RequestMapping("/hello")
    public String Hello(String name) {
        System.out.println("name: " + name);
        return "Hello " + name + "~";
    }
}
```

### HTTP

- 无状态，不同请求之间相互独立
- 一次请求对应一次响应，请求和响应都由 **行/头/体**组成
- 基于TCP

#### spring boot处理请求

内置的tomcat web服务器会将http文本封装为`HttpServletRequest`对象，在使用时可以直接将其作为函数参数：

```java
String method = request.getMethod();	// get/post
String url = request.getRequestURL().toString();	// http://localhost:8080/request
String uri = request.getRequestURI();	// request
String protocol = request.getProtocol();	// HTTP/1.1
String name = request.getParameter("name");	// 
String age = request.getParameter("age");
// 获取请求头参数
String accept = request.getHeader("accept");
String contentType = request.getHeader("content-type");
```

#### 响应码

- 1xx：响应中，临时状态码
- 2xx：成功
- 3xx：重定向
- 4xx：客户端错误
- 5xx：服务端错误

#### spring boot返回响应

1. 用`HttpServletResponse`对象来设置：

   ```java
   response.setStatus(HttpServletResponse.SC_OK);
   response.setHeader("name", "1");
   response.getWriter().write("<h1>hello response</h1>");
   ```

2. 使用`ResponseEntity`对象来链式设置

   ```java
   @RequestMapping("/response")
   public ResponseEntity<String> response(HttpServletRequest request) {
       return ResponseEntity
           .status(401)
           .header("Access-Control-Allow-Origin", "*")
           .body("");
   }
   ```

一般不手动设置响应状态码和响应头

### 三层架构

- Controller（控制层）：接受请求、响应数据
- Service（业务逻辑层）：处理具体的业务逻辑
- dao（数据访问层）：负责数据访问（增删改查...）

架构：

```bash
--controller
	--xxxController.java
--dao
	--impl
		xxxDaoImpl.java
	xxxDao.java
--service
	--impl
		xxxServiceImpl.java
	xxxService.java
Application.java
```

其中，`impl`存接口实现，`xxxDao`存的是接口，`service`同理

1. controller直接使用service的功能，将请求数据解析后调用service处理业务逻辑

2. service直接使用dao的功能，在处理业务逻辑时涉及数据的增删改查，最后将数据返回给controller

3. dao负责提供数据增删改查的接口，一般和model（模型），数据库关联

4. controller在调用完service完成业务逻辑处理后，将数据封装一层作为相应的数据返回给客户端

#### 分层解耦

为了解决在Controller中主动new Service对象和在Service中主动new Dao对象导致的不方便改变接口实现的问题

通过Spring boot框架提供的**Bean**对象来解决上述问题**（Bean对象是容器中创建和管理的对象）**

##### 控制反转（IOC）

将对象的创建控制权转移给外部容器

##### 依赖注入（DI）

容器为应用程序提供运行时所依赖的资源

##### 解决方案1

使用`@Component`和`@Autowired`注解：

```java
// IOC: 将对象交给容器管理
@Component
public class UserDaoImpl implements UserDao {
    ...
}

// DI: 从容器中取对象
@Component
public class UserServiceImpl implements UserServcie{
    @Autowired
    private UserDao userDao;
}

// DI: 从容器中取对象
@RestController
public class UserController {
    @Autowired
    private UserDao userDao;
}
```

##### 解决方案2

```java
@Repository
public class UserDaoImpl implements UserDao {
    ...
}

@Serivce
public class UserServiceImpl implements UserService {
    @Autowired
    private UserDao userDao;
}

@RestController
public class UserController {
    @Autowired
    private UserDao userDao;
}
```

`@Repository`和`@Service`细化了`@Component`注解，controller一般只设置`@RestController`注解，因为其包含了`@Controller`

##### DI详解

1. 基于`@Autowired`注解的三种注入：

   - 在变量名上添加：

     ```java
     @Autowired
     private UserDao userDao;
     ```

   - 构造函数上添加：

     ```java
     @Serivce
     public class UserServiceImpl implements UserService {
         private final UserDao userDao;
         @Autowired
         UserServiceImpl(UserDao userDao) {
             this.userDao = userDao;
         }
     }
     ```

   - 设置set方法（省略，一般用前两种）

2. `@Autowired`默认按照类型注入，即如果有多个类实现了同一个接口，bean对象就不知道该保存哪个类型，从而报错，解决方案如下：

   - 使用`@Primary`注解表明哪个类是主要实现的类

     ```java
     @Primary
     @Repository
     public class UserDaoImpl implements UserDao {
         ...
     }
     ```

   - 使用`@Qualifier`注解变量或构造函数表明使用哪个实现类

     ```java
     @Autowired
     @Qualifier("userDaoImpl")
     private UserDao userDao;
     ```

   - 使用`@Resource`注解变量表明使用的实现类的名称（**不是Spring框架的提供，推荐使用第二个实现**）

     ```java
     @Resource(name = "userDaoImpl")
     private UserDao userDao;
     ```

### 配置文件

将原来的`.property`文件换为`.yml`文件（键值对配置）

```yaml
spring:
  applicaiton:
    name: springboot-mybatis-quickstart
  datasource:
    type: ..
    url: ..
    driver-class-name: ..
```

- 同缩进同级
- 键和值之间有一个空格

### 测试

测试类要在引导程序同包或子包下，同时对类添加`@SpringBootTest`注解

## MYSQL

### 连接

`mysql -u{username} -p [-h{ip} -P{port}]`

- `{username}`：要填的用户名，自用填`root`
- `{ip}`：远程数据库的ip地址
- `{port}`：远程数据库的端口

### SQL基本操作

DDL/DML/DQL/DCL（定义/操作/查询/控制）

#### DLL

定义数据库对象（数据库，表，字段）

##### 数据库操作

```mysql
-- 查询数据库
show database;
-- 创建指定数据库
create database dbname;
-- 删除指定数据库
drop database dbname;
-- 查看数据库表
select database();
-- 使用指定数据库
use dbname;
```

##### 表操作

```mysql
-- 创建表 []代表可选项
create table tablename(
    字段1 字段类型 [约束] [comment 注释],
    ...
)[comment 表注释]
```

1. 约束：
   - 非空`not null`
   - 唯一`unique`
   - 主键`primary key`
   - 默认`default` 跟一个值，表示字段默认值
   - 外键`foreign`
   - 自增`auto increment`
2. 数据类型：
   - 数字：`tinyint`，`int`，`bigint`
   - 字符串：`char`（固定），`varchar`（不固定）
   - 日期：`date`，`datetime`

```mysql
-- 查看当前数据库所有表
show tables;
-- 查看表的所有字段
desc tablename;
-- 查询建表语句
show create table tablename;
-- 向表中增加，修改，重命名，删除字段以及重命名表名
alter table tablename add 字段 类型 [comment] [约束];
alter table tablename modify 字段 类型;
alter table change 旧字段名 新字段名 类型 [comment] [约束];
alter table tablename drop column 字段;
alter table tablename rename to 表名;
-- 删除表
drop table [if exists] tablename;
```

#### DML

- 插入

  ```mysql
  insert into tablename(字段1， 字段2) values (值1 ，值2);
  insert into tablename values (值1， 值2);
  -- 可叠加
  ```

- 更新

  ```mysql
  update tablename set 字段1 = 值1, 字段2 = 值2, ... [where ...]
  -- 不加where条件表示更新所有行
  ```

- 删除

  ```mysql
  delete from tablename [where ...]
  -- 不加where会删除所有行
  ```

#### DQL

```mysql
select 
	-- distinct 去重
	[distinct] 字段 [AS 别名1] 
from
	tablename
where
	-- 比较 <, >, =, !=, between ... and, in(...), like _/%, is null
	-- 逻辑 &&, ||, !
	条件
group by
	-- 聚合函数 count(), max(), min(), avg(), sum(), select用
	-- 使用group by, 前面的select只能选择分组的字段或者聚合函数
	分组字段
having 
	-- 解决where后不能跟聚合函数的问题
	分组后条件列表
order by
	-- 默认asc升序排序, 要降序需指明desc
	排序字段列表 [asc]	
limit
	-- 如0, 5表示查出来的结果从第0条开始展示，最多展示5条, 第一个参数为0可以省略
	起始索引, 查询记录数
```

### JDBC

一套接口，由数据库厂商实现，比如使用mysql要引入对应的jar包：

```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>8.0.33</version>
</dependency>
```

简单使用：

```java
// 1. 注册
class.forName("com.mysql.cj.jdbc.Driver");
// 2. 连接
String url = "jdbc:mysql://localhost:3306/dbname";
String username = "root";
String password = "123456";
Connection conn = DriverManager.getConnection(url, username, password);
// 3. 获取sql执行对象
Statement statement = connection.createStatement();
// 4. 执行语句, i是执行语句后影响的行数
int i = statement.executeUpdate("update user set age = 25 where id = 1");
// 5. 释放连接
statement.close();
conn.close();
```

#### 预编译执行查询

```java
String sql = "select id, username from user where username = ?";
pStatement = conn.prepareStatement(sql);
// 设置sql语句中的?占位符
pStatement.setString(1, "xxx");
// 执行语句, 得到结果集
resultSet = pStatement.executeQuery();
// 遍历结果集
while (resultSet.next()) {
    // getXxx: 可以输入列标号, 也可以输入列名
    resultSet.getInt("id");
}
resultSet.close();
pStatement.close();
```

- 防sql注入
- 数据库会缓存语句，然后对占位符`?`替换，性能更高

### Mybatis

ORM（对象关系映射）框架，简化数据库操作，在这里是对JDBC的进一步封装，底层使用了连接池初始化了一定数量的连接来实现资源复用

#### 配置

在项目`application`文件写入：

```xml
spring.datasource.url=jdbc:mysql://localhost:3306/dbname
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.username=root
spring.datasource.password=1234
```

#### 使用方法

新建mapper包，定义`Mapper`接口，使用`@Mapper`注解

##### 注解

在`Mapper`接口的方法中加入`@Select()`，`@Update()`，`@Insert()`，`@Delete()`注解，并填入SQL语句，以实现对应的功能

**框架会在程序运行时自动为接口创建一个实现类对象（代理对象），并将该对象存入IOC容-**

以查询为例：

```java
public interface UserMapper {
    // #{} 在实际执行时会被替换为 ?
    // 当函数有多个参数时需要使用@Param注解为对应的参数起名， 且与#{}中一致
    @Select("select * from user where username = #{username} and password = #{password}")
    public User findByUsernameAndPassword(@Param("username") String username, @Param("password") String password);
    // 查询结果会被自动封装到User实体类中
    // 多行结果可以使用List接收
}
```

##### XML映射文件

注解适合写一些简单的SQL语句，复杂的一般在XML文件中写

目录结构：

![](./mybatis_xml.png)



- 同包同名
- xml文件的namespace属性为Mapper接口全限定名一致
- xml文件的sql语句的id与Mapper接口中的方法名一致，并保持返回类型一致

![](./mybatis_xml2.png)



xml文件中的SQL语句写法和注解相同，使用`#{}`来代表参数占位，由函数参数传入，如果有多个，就在函数参数上加入`@Param`注解来标明

## RESTful

URL定位资源

HTTP请求方法描述操作（`GET`/`DELETE`,`POST`,`PUT`），分别为查删增改





