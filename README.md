# mybatis-plus-join

* 支持连表查询的 [mybatis-plus](https://gitee.com/baomidou/mybatis-plus)
* [演示工程](https://gitee.com/best_handsome/mybatis-plus-join-demo)
* 点个Star支持一下吧 :)
* [一对一,一对多](https://gitee.com/best_handsome/mybatis-plus-join/blob/master/MAPPING.md)

QQ群:1022221898

## 使用方法

### 安装

- Maven
  ```xml
  <dependency>
      <groupId>com.github.yulichang</groupId>
      <artifactId>mybatis-plus-join</artifactId>
      <version>1.2.4</version>
  </dependency>
  ```
- Gradle
  ```
   implementation 'com.github.yulichang:mybatis-plus-join:1.2.4'
  ```
  或者clone代码到本地执行 mvn install, 再引入以上依赖  
  <br>
  注意: mybatis plus version >= 3.4.0
  <br>

### 使用

* mapper继承MPJBaseMapper (必选)
* service继承MPJBaseService (可选)
* serviceImpl继承MPJBaseServiceImpl (可选)

## 核心类 MPJLambdaWrapper和MPJQueryWrapper

### MPJLambdaWrapper用法

#### 简单的三表查询

```java
class test {
    @Resource
    private UserMapper userMapper;

    void testJoin() {
        List<UserDTO> list = userMapper.selectJoinList(UserDTO.class,
                new MPJLambdaWrapper<UserDO>()
                        .selectAll(UserDO.class)
                        .select(UserAddressDO::getTel)
                        .selectAs(UserAddressDO::getAddress, UserDTO::getUserAddress)
                        .select(AreaDO::getProvince, AreaDO::getCity)
                        .leftJoin(UserAddressDO.class, UserAddressDO::getUserId, UserDO::getId)
                        .leftJoin(AreaDO.class, AreaDO::getId, UserAddressDO::getAreaId)
                        .eq(UserDO::getId, 1)
                        .like(UserAddressDO::getTel, "1")
                        .gt(UserDO::getId, 5));
    }
}
```

对应sql

```
SELECT 
    t.id,
    t.name,
    t.sex,
    t.head_img,
    t1.tel,
    t1.address AS userAddress,
    t2.province,
    t2.city 
FROM 
    user t 
    LEFT JOIN user_address t1 ON t1.user_id = t.id 
    LEFT JOIN area t2 ON t2.id = t1.area_id 
WHERE (
    t.id = ? 
    AND t1.tel LIKE ? 
    AND t.id > ?)
```

说明:

* UserDTO.class 查询结果返回类(resultType)
* selectAll() 查询指定实体类的全部字段
* select() 查询指定的字段,支持可变参数,同一个select只能查询相同表的字段  
  故将UserAddressDO和AreaDO分开为两个select()
* selectAs() 字段别名查询,用于数据库字段与业务实体类属性名不一致时使用
* leftJoin() 参数说明  
  第一个参数:  参与连表的实体类class  
  第二个参数:  连表的ON字段,这个属性必须是第一个参数实体类的属性  
  第三个参数:  参与连表的ON的另一个实体类属性
* 默认主表别名是t,其他的表别名以先后调用的顺序使用t1,t2,t3....
* 条件查询,可以查询主表以及参与连接的所有表的字段,全部调用mp原生的方法,正常使用没有sql注入风险

MPJLambdaWrapper其他功能    
* [简单的SQL函数使用](https://gitee.com/best_handsome/mybatis-plus-join/wikis/selectFunc()?sort_id=4082479)
* [ON语句多条件支持](https://gitee.com/best_handsome/mybatis-plus-join/wikis/leftJoin?sort_id=3496671)

#### 分页查询

```java
class test {
    @Resource
    private UserMapper userMapper;

    void testJoin() {
        IPage<UserDTO> iPage = userMapper.selectJoinPage(new Page<>(2, 10), UserDTO.class,
                new MPJLambdaWrapper<UserDO>()
                        .selectAll(UserDO.class)
                        .select(UserAddressDO::getTel)
                        .selectAs(UserAddressDO::getAddress, UserDTO::getUserAddress)
                        .select(AreaDO::getProvince, AreaDO::getCity)
                        .leftJoin(UserAddressDO.class, UserAddressDO::getUserId, UserDO::getId)
                        .leftJoin(AreaDO.class, AreaDO::getId, UserAddressDO::getAreaId));
    }
}
```

对应sql

```
SELECT 
    t.id,
    t.name,
    t.sex,
    t.head_img,
    t1.tel,
    t1.address AS userAddress,
    t2.province,
    t2.city
FROM 
    user t
    LEFT JOIN user_address t1 ON t1.user_id = t.id
    LEFT JOIN area t2 ON t2.id = t1.area_id
LIMIT ?,?
```

### MPJQueryWrapper

#### 简单的3表查询

```java
class test {
    @Resource
    private UserMapper userMapper;

    void testJoin() {
        List<UserDTO> list = userMapper.selectJoinList(UserDTO.class,
                new MPJQueryWrapper<UserDO>()
                        .selectAll(UserDO.class)
                        .select("addr.tel", "addr.address", "a.province")
                        .leftJoin("user_address addr on t.id = addr.user_id")
                        .rightJoin("area a on addr.area_id = a.id")
                        .like("addr.tel", "1")
                        .le("a.province", "1"));
    }
}
```

对应sql

```
SELECT 
    t.id,
    t.name,
    t.sex,
    t.head_img,
    addr.tel,
    addr.address,
    a.province
FROM 
    user t
    LEFT JOIN user_address addr on t.id = addr.user_id
    RIGHT JOIN area a on addr.area_id = a.id
WHERE (
    addr.tel LIKE ?
    AND a.province <= ?)
```

说明:

* UserDTO.class 查询结果类(resultType)
* selectAll(UserDO.class) 查询主表全部字段(主表实体类)默认主表别名 "t"
* select() mp的select策略是覆盖,以最后一次为准,这里的策略是追加,可以一直select  
  主表字段可以用lambda,会自动添加表别名,主表别名默认是 t ,非主表字段必须带别名查询
* leftJoin() rightJoin() innerJoin() 传sql片段 格式 (表 + 别名 + 关联条件)
* 条件查询,可以查询主表以及参与连接的所有表的字段,全部调用mp原生的方法,正常使用没有sql注入风险

#### 分页查询

```java

class test {
    @Resource
    private UserMapper userMapper;

    void testJoin() {
        IPage<UserDTO> page = userMapper.selectJoinPage(new Page<>(1, 10), UserDTO.class,
                new MPJQueryWrapper<UserDO>()
                        .selectAll(UserDO.class)
                        .select("addr.tel", "addr.address")
                        .select("a.province")
                        .leftJoin("user_address addr on t.id = addr.user_id")
                        .rightJoin("area a on addr.area_id = a.id"));
    }
}
```

对应sql

```
SELECT 
    t.id,
    t.name,
    t.sex,
    t.head_img,
    addr.tel,
    addr.address,
    a.province
FROM 
    user t
    LEFT JOIN user_address addr on t.id = addr.user_id
    RIGHT JOIN area a on addr.area_id = a.id 
LIMIT ?,?
```

#### 还可以这么操作,但不建议

```java
class test {
    @Resource
    private UserMapper userMapper;

    void testJoin() {
        List<UserDTO> list = userMapper.selectJoinList(UserDTO.class,
                new MPJQueryWrapper<UserDO>()
                        .selectAll(UserDO.class)
                        .select("addr.tel", "addr.address")
                        //行列转换
                        .select("CASE t.sex WHEN '男' THEN '1' ELSE '0' END AS sex")
                        //求和函数
                        .select("sum(a.province) AS province")
                        //自定义数据集
                        .leftJoin("(select * from user_address) addr on t.id = addr.user_id")
                        .rightJoin("area a on addr.area_id = a.id")
                        .like("addr.tel", "1")
                        .le("a.province", "1")
                        .orderByDesc("addr.id"));
    }
}
```

对应sql

```
SELECT 
    t.id,
    t.name,
    t.sex,
    t.head_img,
    addr.tel,
    addr.address,
    CASE t.sex WHEN '男' THEN '1' ELSE '0' END AS sex,
    sum(a.province) AS province
FROM 
    user t
    LEFT JOIN (select * from user_address) addr on t.id = addr.user_id
    RIGHT JOIN area a on addr.area_id = a.id
WHERE (
    addr.tel LIKE ?
    AND a.province <= ?)
ORDER BY
    addr.id DESC
```

# [wiki](https://gitee.com/best_handsome/mybatis-plus-join/wikis)




