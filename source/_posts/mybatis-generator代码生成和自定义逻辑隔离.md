---
title: mybatis-generator代码生成和自定义逻辑隔离
date: 2022-09-18 13:38:16
tags: mybatis
Categories: 最佳实践
description: 最近使用 mybatis-generator 时，发现很多时候会有一些更加个性化的 select 或者 update 的场景，这时，如何更优雅使用 mybatis-generator 就是一个问题。
---

最近使用 mybatis-generator 时，发现很多时候会有一些更加个性化的 select 或者 update 的场景，这时，如何更优雅使用 mybatis-generator 就是一个问题。

## 场景说明

现在有一张 user 表：

```sql
create table `user`(
    `id` bigint primary key auto_increment,
    `name` varchar(10) not null unique,
    `age` int,
    `sex` varchar(2),
    `city` varchar(10),
    `country` varchar(20)
)
```

最后使用 mybatis-generator 生成的 UserMapper 如下：

```java
public interface UserMapper {
    int deleteByPrimaryKey(Long id);

    int insert(User row);

    User selectByPrimaryKey(Long id);

    List<User> selectAll();

    int updateByPrimaryKey(User row);
}
```

在此基础上，如果现有有一个根据年龄查询 User 的场景，就需要在 UserMapper 中加上 selectByAge 的方法，解决了燃眉之急，但是如果有一天 user 表变了，需要重新生成 UserMapper，这个时候，selectByAge 就会被删掉，就需要开发人员手动做一次备份。

要如何解决这个问题？

## 解决方案

就像平日做代码设计时，会常常强调将变和不变分离。

上述场景下，变的是自定义场景下的增删改查，不变的是 mybatis-generator 生成的代码，那么我们就得在不改变 UserMapper 的情况下解决问题。

我们可以使用一个新的接口，承载自定义的增删改查的能力，并继承 UserMapper，针对新接口也单独建立 mapper 的 xml 映射文件。

为了对业务代码更友好，编写自动生成 mapper 接口文件的配置时，指定 user 表对应的自动生成得 mapper 接口文件名为 UserBaseMapper，后面手动创建的自定义增删改查的接口为 UserMapper。

生成代码的配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration PUBLIC
        "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<generatorConfiguration>
    <context id="simple" targetRuntime="MyBatis3Simple">
        <commentGenerator>
            <property name="suppressDate" value="true"/>
            <property name="suppressAllComments" value="true"/>
        </commentGenerator>
        <jdbcConnection driverClass="com.mysql.cj.jdbc.Driver"
                        connectionURL="jdbc:mysql://localhost:3306/mybatis?serverTimezone=UTC&amp;nullCatalogMeansCurrent=true"
                        userId="root"
                        password="*VyvofI^dhl_:Y&amp;?/19M0vhr#3hU4151"/>

        <javaModelGenerator targetPackage="com.hanelalo.mybatis.entity" targetProject="src/main/java"/>

        <sqlMapGenerator targetPackage="mappers" targetProject="src/main/resources"/>

        <javaClientGenerator type="XMLMAPPER" targetPackage="com.hanelalo.mybatis.mapper" targetProject="src/main/java"/>

        <table tableName="user" mapperName="UserBaseMapper"/>
    </context>
</generatorConfiguration>
```

生成的 UserBaseMapper：

```java
public interface UserBaseMapper {
    int deleteByPrimaryKey(Long id);

    int insert(User row);

    User selectByPrimaryKey(Long id);

    List<User> selectAll();

    int updateByPrimaryKey(User row);
}
```

对应的 UserBaseMapper.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.hanelalo.mybatis.mapper.UserBaseMapper">
  <resultMap id="BaseResultMap" type="com.hanelalo.mybatis.entity.User">
    <id column="id" jdbcType="BIGINT" property="id" />
    <result column="name" jdbcType="VARCHAR" property="name" />
    <result column="age" jdbcType="INTEGER" property="age" />
    <result column="sex" jdbcType="VARCHAR" property="sex" />
    <result column="city" jdbcType="VARCHAR" property="city" />
    <result column="country" jdbcType="VARCHAR" property="country" />
  </resultMap>
  <delete id="deleteByPrimaryKey" parameterType="java.lang.Long">
    delete from user
    where id = #{id,jdbcType=BIGINT}
  </delete>
  <insert id="insert" parameterType="com.hanelalo.mybatis.entity.User">
    insert into user (id, name, age, 
      sex, city, country)
    values (#{id,jdbcType=BIGINT}, #{name,jdbcType=VARCHAR}, #{age,jdbcType=INTEGER}, 
      #{sex,jdbcType=VARCHAR}, #{city,jdbcType=VARCHAR}, #{country,jdbcType=VARCHAR})
  </insert>
  <update id="updateByPrimaryKey" parameterType="com.hanelalo.mybatis.entity.User">
    update user
    set name = #{name,jdbcType=VARCHAR},
      age = #{age,jdbcType=INTEGER},
      sex = #{sex,jdbcType=VARCHAR},
      city = #{city,jdbcType=VARCHAR},
      country = #{country,jdbcType=VARCHAR}
    where id = #{id,jdbcType=BIGINT}
  </update>
  <select id="selectByPrimaryKey" parameterType="java.lang.Long" resultMap="BaseResultMap">
    select id, name, age, sex, city, country
    from user
    where id = #{id,jdbcType=BIGINT}
  </select>
  <select id="selectAll" resultMap="BaseResultMap">
    select id, name, age, sex, city, country
    from user
  </select>
</mapper>
```

现在新建一个 UserMapper 接口继承 UserBaseMapper：

```java
@Mapper
public interface UserMapper extends UserBaseMapper{

    List<User> selectByAge(@Param("age") int age);
}
```

对应的 xml 文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.hanelalo.mybatis.mapper.UserMapper">
    <resultMap id="BaseResultMap" type="com.hanelalo.mybatis.entity.User">
        <id column="id" jdbcType="BIGINT" property="id"/>
        <result column="name" jdbcType="VARCHAR" property="name"/>
        <result column="age" jdbcType="INTEGER" property="age"/>
        <result column="sex" jdbcType="VARCHAR" property="sex"/>
        <result column="city" jdbcType="VARCHAR" property="city"/>
        <result column="country" jdbcType="VARCHAR" property="country"/>
    </resultMap>
    <select id="selectByAge" resultType="com.hanelalo.mybatis.entity.User">
        select id,
               name,
               age,
               sex,
               city,
               country
        from user
        where age = #{age}
    </select>
</mapper>
```

在使用的时候，直接注入 UserMapper 即可。

```java
@RestController
@RequestMapping("users")
public class UserController {

    @Autowired
    private UserMapper userMapper;

    @GetMapping("/{age}")
    public List<User> queryByAge(@PathVariable("age") int age) {
        return userMapper.selectByAge(age);
    }

    @GetMapping
    public List<User> queryAll() {
        return userMapper.selectAll();
    }

    @PostMapping()
    public void add(@RequestBody User user) {
        userMapper.insert(user);
    }

}
```



```bash
& curl -X GET http://localhost:8080/users          
[{"id":1,"name":"hanelalo","age":25,"sex":"1","city":"cq","country":"CN"},{"id":2,"name":"killer","age":22,"sex":"1","city":"cq","country":"CN"},{"id":3,"name":"driftwood","age":23,"sex":"1","city":"cq","country":"CN"},{"id":4,"name":"king","age":27,"sex":"1","city":"cq","country":"CN"}]%
```

```bash
& curl -X GET http://localhost:8080/users/22
[{"id":2,"name":"killer","age":22,"sex":"1","city":"cq","country":"CN"}]
```

