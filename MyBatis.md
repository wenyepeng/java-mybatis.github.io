# MyBatis基本使用

## 1、测试代码块

```java
//1.雇佣工人
SqlSessionFactoryBuilder builder = new SqlSessionFactoryBuilder();
//2.工人制造工厂
SqlSessionFactory sqlSessionFactory = builder.build(MyBatisTest.class.getClassLoader().getResourceAsStream("mybatis-config.xml"));
//3.工厂开门接单
SqlSession sqlSession = sqlSessionFactory.openSession();
//4.成立一个制造User对象的车间
UserMapper mapper = sqlSession.getMapper(UserMapper.class);
//5.车间生产User对象，即调用mapper接口里的方法
List<User> users = mapper.findAllUsers();
for (User user : users) {
    System.out.println(user);
}
//6.资源释放
sqlSession.close();
```



## 2、核心配置文件

```xml
<configuration>
    <environments default="default">
        <environment id="default">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <!--1.3配置连接池需要的参数-->
                <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://192.168.42.131:3306/db6"/>
                <property name="username" value="root"/>
                <property name="password" value="Wenyepeng.123"/>
            </dataSource>
        </environment>
    </environments>

    <!--接口映射文件的路径-->
    <mappers>
        <mapper resource="com\wen\dao\UserMapper.xml"/>
    </mappers>
</configuration>
```



## 3、接口映射文件

```xml
<!--namesoace：接口类型（完整类路径）-->
<mapper namespace="com.wen.dao.UserMapper">
    <!--
	id:要实现的抽象方法
    select:执行查询语句
    resultType: 表示返回值类型，如果返回值类型是集合，那么写集合的泛型
     -->
    <select id="findAllUsers" resultType="com.wen.entity.User">
        <!-- SQL语句 -->
        select * from  USER;
    </select>
</mapper>
```





# 自定义MyBatis

## 核心步骤

### 1.解析核心配置文件和接口映射文件，得到数据库相关信息和SQL语句

#### 1.1、解析核心配置文件，创建Druid连接池

```java
//-------------------------解析核心配置文件------------------------------------------------------
static {
        try {
            SAXReader reader = new SAXReader();
            Document document = reader.read(SqlSession.class.getResourceAsStream("/mybatis-config.xml"));
            Element driverEle = (Element) document.selectSingleNode("//property[@name='driver']");
            String driver = driverEle.attributeValue("value");

            Element urlEle = (Element) document.selectSingleNode("//property[@name='url']");
            String url = urlEle.attributeValue("value");

            Element usernameEle = (Element) document.selectSingleNode("//property[@name='username']");
            String username = usernameEle.attributeValue("value");

            Element passwordEle = (Element) document.selectSingleNode("//property[@name='password']");
            String password = passwordEle.attributeValue("value");
    
```

#### 1.2、解析映射文件

```java
            //-----------------------------解析接口映射文件-------------------------------------
            Element mapper = (Element) document.selectSingleNode("//mapper");
            String resource = mapper.attributeValue("resource");

            SAXReader reader1 = new SAXReader();
            InputStream in1 = SqlSession.class.getResourceAsStream("/" + resource);
            Document document1 = reader1.read(in1);
            Element rootElement = document1.getRootElement();
            String namespace = rootElement.attributeValue("namespace");
            Element select = rootElement.element("select");
            String id = select.attributeValue("id");
            String resultType = select.attributeValue("resultType");
            String sql = select.getTextTrim();

            String key = namespace+"."+id;
            map.put(key,new Mapper(sql,resultType));

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```





### 2.链接数据库

```java
dataSource = new DruidDataSource();
dataSource.setDriverClassName(driver);
dataSource.setUrl(url);
dataSource.setUsername(username);
dataSource.setPassword(password);
```



### 3.得到接口的代理对象，执行SQL语句

```java
@SuppressWarnings("all")
    public <T> T getMapper(Class<T> type){
        T obj = (T) Proxy.newProxyInstance(
                SqlSession.class.getClassLoader(),
                new Class[]{type},
                new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        DruidDataSource dataSource = getDataSource();
                        DruidPooledConnection conn = dataSource.getConnection();

                        Class<?> cls = method.getDeclaringClass();
                        String clsName = cls.getName();
                        String name = method.getName();
                        String key = clsName+"."+name;
                        Map<String, Mapper> map = getMap();
                        Mapper mapper = map.get(key);

                        String sql = mapper.getSql();
                        String resultType = mapper.getResultType();
                        Class<?> aClass = Class.forName(resultType);

                        PreparedStatement statement = conn.prepareStatement(sql);
                        ResultSet resultSet = statement.executeQuery();

                        List<Object> list = new ArrayList<>();
                        while (resultSet.next()){
                            Object obj = aClass.newInstance();
                            Field[] fields = aClass.getDeclaredFields();
                            for (Field field : fields) {
                                Object value = resultSet.getObject(field.getName());
                                field.setAccessible(true);
                                field.set(obj,value);
                            }
                            list.add(obj);
                        }
                        return list;
                    }
                }
        );
        return obj;
    }
```



### 4.将查询结果转成对象

```java
List<Object> list = new ArrayList<>();
                        while (resultSet.next()){
                            Object obj = aClass.newInstance();
                            Field[] fields = aClass.getDeclaredFields();
                            for (Field field : fields) {
                                Object value = resultSet.getObject(field.getName());
                                field.setAccessible(true);
                                field.set(obj,value);
                            }
                            list.add(obj);
                        }
                        return list;
```



### 5.完整代码

```java
package com.wen.selfbatis;

import com.alibaba.druid.pool.DruidDataSource;
import com.alibaba.druid.pool.DruidPooledConnection;
import org.dom4j.Document;
import org.dom4j.Element;
import org.dom4j.io.SAXReader;

import java.io.InputStream;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * 自定义MyBatis类
 * @author 11433
 */
public class SqlSession {
    private DruidDataSource dataSource;

    private Map<String ,Mapper> map = new HashMap<>();

    public SqlSession(){

        try {
            SAXReader reader = new SAXReader();
            Document document = reader.read(SqlSession.class.getResourceAsStream("/mybatis-config.xml"));

            Element driverEle = (Element) document.selectSingleNode("//property[@name='driver']");
            String driver = driverEle.attributeValue("value");

            Element urlEle = (Element) document.selectSingleNode("//property[@name='url']");
            String url = urlEle.attributeValue("value");

            Element usernameEle = (Element) document.selectSingleNode("//property[@name='username']");
            String username = usernameEle.attributeValue("value");

            Element passwordEle = (Element) document.selectSingleNode("//property[@name='password']");
            String password = passwordEle.attributeValue("value");

            //创建德鲁伊连接池
            dataSource = new DruidDataSource();
            dataSource.setDriverClassName(driver);
            dataSource.setUrl(url);
            dataSource.setUsername(username);
            dataSource.setPassword(password);

            //-----------------------------解析接口映射文件-------------------------------------
            Element mapper = (Element) document.selectSingleNode("//mapper");
            String resource = mapper.attributeValue("resource");

            SAXReader reader1 = new SAXReader();
            InputStream in1 = SqlSession.class.getResourceAsStream("/" + resource);
            Document document1 = reader1.read(in1);
            Element rootElement = document1.getRootElement();
            String namespace = rootElement.attributeValue("namespace");
            Element select = rootElement.element("select");
            String id = select.attributeValue("id");
            String resultType = select.attributeValue("resultType");
            String sql = select.getTextTrim();

            String key = namespace+"."+id;
            map.put(key,new Mapper(sql,resultType));

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public DruidDataSource getDataSource() {
        return dataSource;
    }

    public Map<String, Mapper> getMap() {
        return map;
    }

    @SuppressWarnings("all")
    public <T> T getMapper(Class<T> type){
        T obj = (T) Proxy.newProxyInstance(
                SqlSession.class.getClassLoader(),
                new Class[]{type},
                new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        DruidDataSource dataSource = getDataSource();
                        DruidPooledConnection conn = dataSource.getConnection();

                        Class<?> cls = method.getDeclaringClass();
                        String clsName = cls.getName();
                        String name = method.getName();
                        String key = clsName+"."+name;
                        Map<String, Mapper> map = getMap();
                        Mapper mapper = map.get(key);

                        String sql = mapper.getSql();
                        String resultType = mapper.getResultType();
                        Class<?> aClass = Class.forName(resultType);

                        PreparedStatement statement = conn.prepareStatement(sql);
                        ResultSet resultSet = statement.executeQuery();

                        List<Object> list = new ArrayList<>();
                        while (resultSet.next()){
                            Object obj = aClass.newInstance();
                            Field[] fields = aClass.getDeclaredFields();
                            for (Field field : fields) {
                                Object value = resultSet.getObject(field.getName());
                                field.setAccessible(true);
                                field.set(obj,value);
                            }
                            list.add(obj);
                        }
                        return list;
                    }
                }
        );
        return obj;
    }
}

```

* mapper类

```java
package com.wen.selfbatis;

/**
 * @author 11433
 */
public class Mapper {
    private String sql;
    private String resultType;

    public Mapper() {
    }

    public Mapper(String sql, String resultType) {
        this.sql = sql;
        this.resultType = resultType;
    }

    /**
     * 获取
     * @return sql
     */
    public String getSql() {
        return sql;
    }

    /**
     * 设置
     * @param sql
     */
    public void setSql(String sql) {
        this.sql = sql;
    }

    /**
     * 获取
     * @return resultType
     */
    public String getResultType() {
        return resultType;
    }

    /**
     * 设置
     * @param resultType
     */
    public void setResultType(String resultType) {
        this.resultType = resultType;
    }

    public String toString() {
        return "Mapper{sql = " + sql + ", resultType = " + resultType + "}";
    }
}

```



# MyBatis高级查询

## 1、查询插入数据在数据库中的主键值

* 配置文件

```xml
<insert id="createUser" parameterType="com.wen.entity.User" >
        INSERT INTO USER VALUES(null,#{username},#{birthday},#{sex},#{address})
        <selectKey keyColumn="id" keyProperty="id" resultType="int" order="AFTER">
            SELECT LAST_INSERT_ID();
        </selectKey>
</insert>
```

* java代码

```java
@Test
    public void testMyBatis(){
        SqlSession sqlSession = Utils.getSqlSession();
        //4.通过SqlSession对象得到Mapper接口的代理对象
        UserMapper mapper = sqlSession.getMapper(UserMapper.class);

        User user = new User("太白", Date.valueOf("1988-02-21"), "男", "峨眉");
        mapper.insertUser(user);
        System.out.println(user);
        Utils.commitAndClose(sqlSession);
    }
```



## 2、模糊查询

* 配置文件

```xml
<select id="findUserLikeName" resultType="com.wen.entity.User" parameterType="String">
        SELECT * FROM USER WHERE username LIKE "%"#{username}"%";
    </select>
```

* java代码

```java
@Test
    public void testMyBatis2(){
        SqlSession sqlSession = Utils.getSqlSession();
        UserMapper mapper = sqlSession.getMapper(UserMapper.class);
        List<User> users = mapper.findUserLikeName("精");
        for (User user : users) {
            System.out.println(user);
        }
    }
```



## 3、多参数查询

* 配置文件

```xml
<select id="findUserByStartEnd" parameterType="String" resultType="User">
        select * from USER where birthday between #{start} and #{end};
    </select>
```

* 抽象类代码

```java
/**
     * 多条件查询
     * @param start 起始时间
     * @param end 结束时间
     * @return 返回查询到的用户
     * 在参数前需要使用@Param注解，应为形参不会保存到class文件中，一编译就会丢失，不加注解就会报错提示找	  *	不到start和end参数
     */
    List<User> findUserByStartEnd(@Param("start") String start,@Param("end") String end);
```





## 4、查询包装类

> 在dao包中创建一个查询包装类QueryVO，将要传入的参数作为包装类的成员变量，在查询时只需要传入一个包装类对象即可

* 包装类

```java
public class QueryVO {
    private User user;
    private String start;
    private String end;
}
```

* 配置文件

``` xml
<select id="findUserByBirthday" parameterType="QueryVO" resultType="User"> 
    <!--
	查询包装类中成员变量是一个自定义类时，获取自定义对象的成员变量的方法为：类名.成员变量名
	在xml中<>为特殊含义的开始符号，所以小于要使用&lt; 符号表示
	-->
    select * from USER where birthday between #{start} and #{end} and id&lt;#{user.id};</select>
```





## 5、#{}和${}比较

|                                  | 占位符#{变量名}                                   | 拼接符${变量名}                          |
| -------------------------------- | ------------------------------------------------- | ---------------------------------------- |
| 作用                             | 先使用？占位，后取出参数的值给？赋值，没有SQL注入 | 直接取出参数和SQL拼接字符串，会有SQL注入 |
| 基本数据类型和包装类和String写法 | #{}里面写参数名                                   | #{}里面只能写value                       |





# 核心配置文件详解

## 1、dtd文件配置

> 必须按以下顺序出现， ‘？’表示这个元素可以出现0~1次

```xml
<!ELEMENT configuration(properties?,setting?,typeAliases?,typeHandlers?,objectFactory?,objectWrapperFactory?,reflectorFactory?,plugins?,environments?databaseIdProvider?,mappers?)>
```

| 顺序 | 配置标签名称  | 说明                                                         |
| ---- | ------------- | ------------------------------------------------------------ |
| 1    | properties    | 设置外部可替代的属性，可以从java属性配置文件中读取           |
| 2    | setting       | mybatis全局的配置参数                                        |
| 3    | typeAliases   | 给Java类型定义别名                                           |
| 4    | typeHandlers  | 类型处理器，将结果集中的值以合适的方式转换成java类型         |
| 5    | objectFactory | 可以指定用于创建结果的对象工厂                               |
| 6    | plugins       | 允许使用插件来拦截mybatis中一些方法的调用                    |
| 7    | environments  | 配置对中环境，可以将SQL映射应用于多种数据库之中![1621937451593](C:\Users\11433\AppData\Roaming\Typora\typora-user-images\1621937451593.png) |
| 8    | mappers       | 定义SQL映射语句的配置文件路径                                |



# 接口映射文件详解

## 1、三种参数类型

* 简单类型：8中数据类型+String，包括对应的包装类
* POJO类型：（Plain Ordinary Java Object）简单的Java对象，实际就是普通的JavaBean，即实体类
* POJO包装类型：即实体类中包含了其他的实体类



## 2、方法中定义多个参数

* 方法一

```java
/**
 * 多个参数查询
 * @param username 用户名
 * @param sex 性别
 * @return 返回用户信息
 */
User findUser(String username,String sex);
```

```xml
<!--形式参数的变量名不会保存到编译文件中，只能以arg的序号添加数据-->
<select id="findUser" resultType="user">
        SELECT * FROM USER WHERE username=#{arg0} and sex=#{arg1};
</select>
```

* 方法二

```java
/**
 * 多个参数查询
 * @param username 用户名
 * @param sex 性别
 * @return 返回用户信息
 * 使用@Param注解将变量名保存到编译文件中
 */
User findUser(@Param("username")String username,@Param("sex")String sex);
```

```xml
<!--使用@Param注解的变量名可以保存到编译文件里，直接传入变量名调用即可-->
<select id="findUser" resultType="user">
        SELECT * FROM USER WHERE username=#{username} and sex=#{sex};
</select>
```



## 3、POJO包装类型

![1621943661362](C:\Users\11433\AppData\Roaming\Typora\typora-user-images\1621943661362.png)

```java
public interface UserMapper {
    List<User> findUsersByCondition(QueryVo queryVo);
}
```

```xml
<!--调用包装类里面的包装类的成员变量：类名.成员变量-->
<select id="findUsersByCondition" parameterType="queryvo" resultType="user">
    SELECT * FROM user WHERE id &lt; #{user.id} AND birthday BETWEEN #{start} AND #{end};
</select>
```



## 4、resultMap输出映射

* 方法一：在接口映射文件中，使用resultMap标签简历查询的列与对象属性的对应关系

```xml
<mapper namespace="com.wen.dao.OrderMapper">
    <resultMap id="orderMap" type="Order">
        <id column="o_id" property="oId"/>
        <result column="user_id" property="userId"/>
        <result column="create_time" property="createTime"/>
    </resultMap>
    <select id="findAllOrder" resultMap="orderMap">
        SELECT * FROM `order`;
    </select>
</mapper>
```

![1621944234848](C:\Users\11433\AppData\Roaming\Typora\typora-user-images\1621944234848.png)

* 方法二：在核心配置文件中，使用setting标签将useGeneratedToCamelCase属性设置为true，表示将映射下划线为驼峰命名法

```xml
<settings>
        <!--mapUnderscoreToCamelCase:将数据中下划线命名自动映射成Java中驼峰式命名-->
        <setting name="mapUnderscoreToCamelCase" value="true"/>
</settings>
```



# 动态SQL

## 1、if标签

* if标签的格式

```xml
<if test="条件">
	SQL片段
</if>
```

* if标签的作用

> 当条件满足是就拼接SQL片段

## 2、where标签

* where标签的作用
  * 相当于where关键字，自动补全where这个关键字
  * 去掉多余的and和or关键字

```xml
<select id="findAllUser" resultType="User">
        SELECT * FROM USER
        <where>
            <if test="username!=null and username!=''">
                username like "%"#{username}"%"
            </if>
            <if test="sex!=null and sex!=''">
                and sex = #{sex};
            </if>
        </where>
    </select>
```



## 3、set标签

* set标签的作用
  * 用在update语句中，相当于set关键字
  * 去掉SQL代码片段中后面多余的逗号

```xml
<update id="updateUser">
        UPDATE USER
        <set>
            <if test="username!=null and username!=''">
                username=#{username},
            </if>
            <if test="birthday!=null">
                birthday=#{birthday},
            </if>
            <if test="sex!=null and sex!=''">
                sex=#{sex},
            </if>
            <if test="address!=null and address!=''">
                address=#{address}
            </if>
        </set>
        WHERE id=#{id};
    </update>
```



## 4、foreach标签

* foreach标签的作用
  * 遍历数组或集合

| foreach标签的属性 | 作用                            |
| ----------------- | ------------------------------- |
| collection        | 遍历集合写list，遍历数组写array |
| item              | 设置变量名，代表每个遍历的元素  |
| separator         | 遍历一个元素添加的内容          |
| #{变量名}         | 先使用？占位，后面给？赋值      |
| open              | 在遍历前添加一次字符            |
| close             | 在遍历后添加一次字符            |

```xml
<delete id="deleteUser">
        DELETE FROM USER WHERE id IN
        <foreach collection="array" item="ele" open="(" close=");" separator=",">
            #{ele}
        </foreach>
</delete>
```

