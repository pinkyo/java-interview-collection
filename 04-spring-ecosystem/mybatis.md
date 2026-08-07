# MyBatis / MyBatis-Plus

---

## 一、MyBatis 执行流程

```
1. 加载配置文件（mybatis-config.xml / application.yml）
2. 解析 Mapper XML，构建 Configuration 对象
3. 通过 SqlSessionFactoryBuilder 创建 SqlSessionFactory
4. 打开 SqlSession
5. 调用 Mapper 接口方法 → 动态代理 → MapperProxy.invoke()
6. 根据 statement id 获取 MappedStatement
7. 通过 Executor 执行 SQL
8. 参数映射（ParameterHandler）→ 执行 SQL → 结果映射（ResultSetHandler）
9. 返回结果并关闭 SqlSession
```

---

## 二、核心组件

| 组件 | 说明 |
|------|------|
| SqlSessionFactory | 创建 SqlSession 的工厂，全局唯一 |
| SqlSession | 一次会话，非线程安全，使用后关闭 |
| Executor | 执行器，负责 SQL 执行（Simple/Reuse/Batch） |
| StatementHandler | 封装 JDBC Statement 操作 |
| ParameterHandler | 参数处理 |
| ResultSetHandler | 结果集映射 |
| TypeHandler | Java 类型 ↔ JDBC 类型转换 |
| MappedStatement | 封装了一条 SQL 的所有信息 |

---

## 三、MyBatis 源码核心

### 3.1 Mapper 代理原理

```java
// MapperProxy 实现了 InvocationHandler（JDK 动态代理）
public class MapperProxy<T> implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) {
        // SQL 命令 → 交给 SqlSession 执行
        if (method 是 Object 的方法) return method.invoke(this, args);
        return sqlSession.selectOne(command.getName(), args);
    }
}
```

### 3.2 插件机制

MyBatis 插件基于**责任链 + 动态代理**，可拦截四个对象：

```java
@Intercepts(@Signature(
    type = Executor.class,
    method = "update",
    args = {MappedStatement.class, Object.class}
))
public class MyInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        // 前置处理
        Object result = invocation.proceed();
        // 后置处理
        return result;
    }
}
```

可拦截的四种对象：**Executor → ParameterHandler → ResultSetHandler → StatementHandler**。

---

## 四、MyBatis 缓存

### 4.1 一级缓存（默认开启）

- SqlSession 级别，同一个 SqlSession 中执行的查询会被缓存
- 执行 insert/update/delete 或 commit/close 后清空
- 无法跨 SqlSession 共享

### 4.2 二级缓存（需手动配置）

```xml
<!-- 1. mybatis-config.xml -->
<settings>
    <setting name="cacheEnabled" value="true"/>
</settings>

<!-- 2. Mapper XML -->
<cache
    eviction="LRU"
    flushInterval="60000"
    size="512"
    readOnly="true"/>
```

- Mapper 级别（namespace），多个 SqlSession 共享
- 事务提交后才写入缓存
- 问题：**脏数据**（分布式环境容易不一致，生产一般不开启）

---

## 五、#{} 和 ${} 的区别

| 对比 | #{} | ${} |
|------|-----|-----|
| 处理方式 | 预编译占位符（PreparedStatement） | 直接拼接字符串 |
| SQL 注入 | 安全 | 危险⚠️ |
| 何时使用 | 参数值 | 表名、列名、排序字段 |

```xml
<!-- 安全 -->
SELECT * FROM user WHERE id = #{id}
-- → SELECT * FROM user WHERE id = ?

<!-- 危险 -->
SELECT * FROM user WHERE id = ${id}
-- → SELECT * FROM user WHERE id = 1 OR 1=1
```

---

## 六、MyBatis-Plus

### 6.1 AR 模式与 BaseMapper

```java
// BaseMapper 提供标准 CRUD
public interface UserMapper extends BaseMapper<User> { }

// 直接使用
List<User> users = userMapper.selectList(
    new LambdaQueryWrapper<User>()
        .eq(User::getAge, 18)
        .gt(User::getCreateTime, date)
        .orderByDesc(User::getId)
);
```

### 6.2 分页插件

```java
@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}

// 使用
Page<User> page = new Page<>(1, 10);
Page<User> result = userMapper.selectPage(page, null);
```

### 6.3 乐观锁插件

```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
    return interceptor;
}

@Data
public class User {
    @Version
    private Integer version;  // 版本号字段
}

// UPDATE user SET name=?, version=version+1 WHERE id=? AND version=?
```

---

## 面试追问集

**Q：MyBatis 和 Hibernate 的区别？**

| 对比 | MyBatis | Hibernate |
|------|---------|-----------|
| 定位 | 半自动 ORM（SQL 映射） | 全自动 ORM |
| SQL 控制 | 手动编写 | 自动生成（HQL） |
| 灵活性 | 高（复杂查询可手写 SQL） | 低（复杂查询困难） |
| 学习成本 | 低 | 高 |
| 适用场景 | 互联网，SQL 优化要求高 | 传统企业，对象模型复杂 |

**Q：MyBatis 如何实现动态 SQL？**

通过 OGNL 表达式：`<if>` / `<choose>` / `<where>` / `<set>` / `<foreach>` / `<trim>`。

**Q：返回自增主键的方式？**

```xml
<insert id="insert" useGeneratedKeys="true" keyProperty="id">
    INSERT INTO user (name) VALUES (#{name})
</insert>
```
