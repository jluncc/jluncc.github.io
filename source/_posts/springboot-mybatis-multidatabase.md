---
title: SpringBoot2 整合 Mybatis 多数据源流程
author: 陈加菲
date: 2019-09-22 16:50:00
tags:
 - SpringBoot
 - Mybatis
---

最近在搭建项目的框架，使用的是 SpringBoot 跟 Mybatis，并接入了多数据源。这里总结一下整合的流程以及踩过的坑。你亦可到 github 上查看我的工程代码：[springboot-mybatis-multidatabase](https://github.com/jluncc/my-springboot-demo/tree/master/springboot-mybatis-multidatabase)。


## SpringBoot2 整合 Mybatis，接入多数据源流程

### 01 测试数据

在本地测试，我使用了 MySQL，创建 test_db1 / test_db2 两个数据库，分别新建一张测试表来模拟多数据源的接入。数据如下：

> 我的工程代码链接：[springboot-mybatis-multidatabase](https://github.com/jluncc/my-springboot-demo/tree/master/springboot-mybatis-multidatabase)，你也可以从中获取我使用的数据脚本。

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/springboot-mybatis-multidatabase/test-data-1.png" alt="测试数据1-student表" title="测试数据1-student表" style="zoom:61%;" />

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/springboot-mybatis-multidatabase/test-data-2.png" alt="测试数据2-category表" title="测试数据2-category表" style="zoom:60%;" />



### 02 pom 依赖

我们新建 SpringBoot 项目（你可在 https://start.spring.io 自动创建，或使用 IDEA 等 IDE 来辅助创建，不在此详述），先来看看 pom 文件的依赖。我是基于 SpringBoot2 来操作的，具体使用到的版本为 2.1.8.RELEASE。

此外使用到 MySQL 数据库，lombok 插件，swagger 调用测试接口来校验结果：

> 实际使用中，下面介绍的方法也测试过 MySQL 与 PostgreSQL 混用，同样正常运行。若接入其他数据库，需在 pom 中接入相应的依赖。

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.1.0.RELEASE</version>
    <relativePath /> <!-- lookup parent from repository -->
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
    <dependency>
    	<groupId>mysql</groupId>
    	<artifactId>mysql-connector-java</artifactId>
    	<scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>1.3.2</version>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <dependency>
        <groupId>io.springfox</groupId>
        <artifactId>springfox-swagger2</artifactId>
        <version>2.9.2</version>
    </dependency>
    <dependency>
        <groupId>io.springfox</groupId>
        <artifactId>springfox-swagger-ui</artifactId>
        <version>2.9.2</version>
    </dependency>
</dependencies>
```

### 03 yml 配置

我使用了 yml 配置文件格式，在这里，有三部分的配置

1. database 数据源信息；
2. Mybatis 配置。这里每个数据源单独配置了一份 Mybatis 的配置信息，后续会说明这样做的原因；
3. 服务暴露的端口 

```yml
mybatis:
  configuration:
    db1:
      log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
      map-underscore-to-camel-case: true
    db2:
      log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
      map-underscore-to-camel-case: true

spring:
  datasource:
    db1:
      jdbc-url: jdbc:mysql://localhost:3306/test_db1
      username: your username
      password: your password
      driver-class-name: com.mysql.jdbc.Driver
    db2:
      jdbc-url: jdbc:mysql://localhost:3306/test_db2
      username: your username
      password: your password
      driver-class-name: com.mysql.jdbc.Driver

server:
  port: 8090
```

### 04 数据源配置

当在 SpringBoot 配置文件中创建多个数据源的配置时，若使用 Mybatis，mapper 是不能自动识别哪个 mapper 该基于哪个数据源连接的。这里我们的关键是需要**为 mapper 指定使用的数据源**。

根据我的调研，有几种指定的思路，例如使用 AOP，或者本文介绍的，**显式指定不同 mapper 使用不同的数据源**。

新建 Db1MybatisConfig.java，使用 @Configuration 指定为 Spring 的配置文件。接着，我们使用 @MapperScan 来指定，该份配置文件需要扫描哪些包（basePackages 参数），以及所扫描的包，使用哪份模板文件来与数据库操作（sqlSessionTemplateRef 参数）。

```java
@Configuration
@MapperScan(basePackages = "com.example.demo.mapper.db1", sqlSessionTemplateRef = "db1SqlSessionTemplate")
public class Db1MybatisConfig {
    ...
}
```

当指定了这两个参数，我们只要进一步的，让当前使用的模板文件与特定的数据库连接，便可实现**特定包下的 mapper  与特定数据源连接**。

> 1. 特定数据源 —> 2. 特定 sqlSessionTemplate —> 3. 特定包目录 —> 4. 特定 mapper

@MapperScan 实现了 3 -> 4 步骤的过程，剩下的 1 ->2 步骤需要在上述配置类中实现，详见下方代码：

```java
    @Bean(name = "db1DataSource")
    @ConfigurationProperties(prefix = "spring.datasource.db1") // 读取yml数据源配置，自动注入到DataSource
    @Primary // 注意若配置了多数据源，需要使用@Parimary显式指定一个主数据源，其余数据源不需再配置该注解
    public DataSource db1DataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "db1Config")
    @ConfigurationProperties(prefix = "mybatis.configuration.db1") // 读取yml Mybatis配置，自动注入到Configuration
    @Primary
    public org.apache.ibatis.session.Configuration db1MybatisConfig() {
        return new org.apache.ibatis.session.Configuration();
    }

    @Bean(name = "db1SqlSessionFactory")
    @Primary
    public SqlSessionFactory db1SqlSessionFactory(@Qualifier("db1DataSource") DataSource dataSource,
                                                  @Qualifier("db1Config") org.apache.ibatis.session.Configuration config)
            throws Exception {
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(dataSource);
        bean.setConfiguration(config); // 多数据源需要在各自的SessionFactory中设置配置Mybatis属性
        bean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:/mapper/db1/*.xml"));
        return bean.getObject();
    }

    @Bean(name = "db1TransactionManager")
    @Primary
    public DataSourceTransactionManager db1TransactionManager(@Qualifier("db1DataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    @Bean(name = "db1SqlSessionTemplate")
    @Primary
    public SqlSessionTemplate db1SqlSessionTemplate(@Qualifier("db1SqlSessionFactory") SqlSessionFactory sqlSessionFactory) throws Exception {
        return new SqlSessionTemplate(sqlSessionFactory);
    }
```

1. 首先使用 @ConfigurationProperties 获取 yml 文件中，我们配置的数据源信息，自动注入到 DataSource 对象；
2. 与 1 类似，使用 @ConfigurationProperties 获取 yml 中特定数据源的 Mybatis 配置信息，自动注入到 Configuration 对象；
3. 准备好上述数据后，进一步手动封装 SqlSessionFactory，再由其生成对应的 SqlSessionTemplate，并使用该数据源来进行事务管理 DataSourceTransactionManager。

至此我们便绑定了特定数据源到特定的 SqlSessionTemplate，进而实现了特定包目录下的 mapper 文件，通过对应绑定的 SqlSessionTemplate 来操作特定的数据源。

### 05 编写代码测试

#### mapper

由于在数据源配置文件中绑定了特定的包目录，因此，我们有

1. 若想操作 db1 数据源，mapper 接口只可置于配置文件中绑定的 com.example.demo.mapper.db1 目录；
2. 若想操作 db2 数据源，mapper 接口只可置于配置文件中绑定的 com.example.demo.mapper.db2 目录。

我们在 db1 目录中新建 StudentMapper.java 来进行测试：

> 因为已在配置文件中指定了目录，因此 mapper 接口不需再使用 @Mapper。

```java
public interface StudentMapper {

    List<Student> listStudent();

    Student findStudentById(@Param("id") Integer id);
}
```

接下来使用 XML 文件编写具体的 SQL，XML 文件的位置没有具体要求，看自己如何方便管理，例如我便类似的新建了 resource/mapper/db1 目录放置 db1 的 mapper 接口 XML 文件。

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.demo.mapper.db1.StudentMapper">
    <sql id="column_list">id, name, age, create_time, update_time</sql>

    <select id="listStudent" resultType="com.example.demo.model.Student">
        select <include refid="column_list"/> from student
    </select>

    <select id="findStudentById" resultType="com.example.demo.model.Student">
        select <include refid="column_list"/> from student where id = #{id}
    </select>
</mapper>
```

#### service 与 controller

编写完 mapper，剩下的 service 与 controller 便没有特别：

```java
@Service
public class StudentService {
    @Resource
    private StudentMapper studentMapper;

    public List<Student> listStudent() {
        return studentMapper.listStudent();
    }
    public Student getStudentById(Integer id) {
        return studentMapper.findStudentById(id);
    }
}
```

> service 我直接编写实现类，而不是编写接口，针对接口编程。



```java
@RestController
@RequestMapping("/api/test/")
public class TestController {

    @Autowired
    private StudentService studentService;

    @ApiOperation("根据id查询学生信息")
    @GetMapping("/student/{id}")
    public String getStudent(@PathVariable("id") Integer id) {
        Student student = studentService.getStudentById(id);
        return student == null ? "no data." : student.toString();
    }

    @ApiOperation("查询学生列表")
    @GetMapping("/student/list")
    public String students() {
        List<Student> students = studentService.listStudent();
        return CollectionUtils.isEmpty(students) ? "no data." : students.toString();
    }
}
```

> @ApiOperation 是 swagger 的注解，启动项目后 swagger 可自动将接口生成文档，同时也可模拟接口调用。这里我将使用 swagger 来测试接口。

### 06 结果

到这里代码编写完毕。启动项目，打开  [http://localhost:8090/swagger-ui.html](http://localhost:8090/swagger-ui.html%EF%BC%8C%E5%8F%AF%E7%9C%8B%E5%88%B0)，可看到 swagger 自动生成的接口文档，我们进行测试调用 /api/test/student/1，获取数据与数据库一致。

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/springboot-mybatis-multidatabase/result-1.png" alt="结果1" title="结果1" style="zoom:100%;" />

同样，根据该思路我们编写另一数据源 db2 的相应代码，最后测试调用 /api/test/category/1，亦正常获取到数据

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/springboot-mybatis-multidatabase/result-2.png" alt="结果2" title="结果2" style="zoom:100%;" />



## 踩过的坑

### 01 使用数据源配置文件后，Mybatis 配置不生效

配置了多数据源后，一开始 Mybatis 的 yml 配置（mybatis.configuration）是不生效的，例如我们上面配置的下划线转驼峰。

这是因为数据源的 Mybatis 配置信息在 SqlSessionFactory 中生成，而我们是自定义了该工厂的生成，此时，Spring 没有将 yml 中的 Mybatis 配置信息放入到 SqlSessionFactory 中，因此在上面的配置类代码中，我们需要显式使用 @ConfigurationProperties(prefix = "mybatis.configuration.db1") 加载特定数据源的 Mybatis 配置。

### 02 yml 中 Mybatis 数据源配置要分别配置

不同数据源需要分别配置，这是因为一个配置只能加载到一个 bean 中，上面两个数据源需要分别编写配置类，因此单一 bean 是满足不了的。

另外，我们很可能会遇到不同数据源，Mybatis 配置不同的情况，这也需要我们将不同数据源的 Mybatis 配置分别设置。

而对于不同环境，我们编写多份 yml 配置文件，不同文件里的 Mybatis 配置也可不相同，满足不同的具体需求。

## 参考文章

[Spring Boot(七)：Mybatis 多数据源最简解决方案](https://www.cnblogs.com/ityouknow/p/6102399.html)
[SpringBoot 中 mybatis 配置自动转换驼峰标识没有生效](https://www.cnblogs.com/flying607/p/8473075.html)
[Mybatis XML 配置](https://mybatis.org/mybatis-3/zh/configuration.html)

**END**