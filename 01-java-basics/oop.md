# 面向对象（OOP）

---

## 一、封装、继承、多态

### 1.1 封装

封装是将数据和操作数据的方法绑定在一起，隐藏内部实现细节，对外暴露 API。

```java
public class BankAccount {
    private double balance;  // 私有字段，外部不可直接访问

    public void deposit(double amount) {
        if (amount > 0) balance += amount;  // 通过方法控制访问
    }

    public double getBalance() {
        return balance;
    }
}
```

**追问：访问修饰符有哪些？**

| 修饰符 | 同一个类 | 同包 | 子类 | 其他包 |
|-------|---------|------|------|-------|
| private | ✓ | ✗ | ✗ | ✗ |
| default | ✓ | ✓ | ✗ | ✗ |
| protected | ✓ | ✓ | ✓ | ✗ |
| public | ✓ | ✓ | ✓ | ✓ |

### 1.2 继承

- **单继承**：Java 只支持单继承，但可通过接口实现多继承效果
- **构造方法链**：子类构造方法会隐式调用父类无参构造，如果父类没有无参构造，必须显式 super()

### 1.3 多态

```java
// 编译时多态（重载）：方法名相同，参数不同
void print(int a) { }
void print(String s) { }

// 运行时多态（重写）：子类重写父类方法，通过父类引用调用子类方法
Animal a = new Dog();
a.sound();  // 调用 Dog 的 sound()
```

**追问：重载和重写的区别？**

| 对比维度 | 重载（Overload） | 重写（Override） |
|---------|----------------|-----------------|
| 发生位置 | 同一类 / 父子类 | 只能父子类 |
| 方法签名 | 方法名相同，参数不同 | 方法名和参数完全一致 |
| 返回类型 | 可以不同 | 相同或子类（协变返回） |
| 访问修饰符 | 可以不同 | 不能更严格 |
| 异常 | 可以不同 | 不能抛出更宽泛的检查异常 |
| 多态类型 | 编译时多态 | 运行时多态 |

---

## 二、抽象类 vs 接口

| 对比维度 | 抽象类 | 接口（JDK 8+） |
|---------|--------|---------------|
| 实例化 | 不能实例化 | 不能实例化 |
| 构造方法 | 有 | 没有 |
| 方法实现 | 可以有具体方法 | default/static 方法可有实现 |
| 成员变量 | 任意类型 | 只能是 public static final 常量 |
| 继承数量 | 单继承 | 多实现 |
| 使用场景 | is-a 关系 | can-do 能力 |
| 设计层面 | 自底向上抽象 | 自顶向下定义契约 |

**追问：什么时候用抽象类，什么时候用接口？**

- 抽象类：有共同属性和行为，且存在默认实现 → 模板方法模式
- 接口：定义行为规范，与继承体系解耦 → 策略模式、适配器模式

---

## 三、内部类

### 3.1 内部类分类

| 类型 | 定义位置 | 特点 |
|------|---------|------|
| 成员内部类 | 类的方法外 | 可访问外部类所有成员 |
| 静态内部类 | static 修饰 | 只能访问外部类静态成员 |
| 局部内部类 | 方法内部 | 作用域仅限于方法 |
| 匿名内部类 | 表达式实例化 | 没有类名，Lambda 可替代 |

### 3.2 匿名内部类和 Lambda 的区别

- 匿名内部类编译生成 `$1.class`，Lambda 不生成额外类文件
- Lambda 只能用于函数式接口（单抽象方法）
- Lambda 中 this 指向外部类，匿名内部类中 this 指向自身

---

## 四、Object 类的方法

| 方法 | 说明 |
|------|------|
| equals() | 比较对象内容，默认比较地址（==） |
| hashCode() | 返回哈希码，equals 相等则 hashCode 必须相等 |
| toString() | 返回类名@哈希码的字符串表示 |
| clone() | 浅拷贝，需实现 Cloneable 接口 |
| finalize() | JDK 9 已废弃，不建议使用 |
| getClass() | 获取运行时类 |
| wait()/notify()/notifyAll() | 线程间通信 |

**追问：为什么重写 equals 必须重写 hashCode？**

HashMap/HashSet 等容器先用 hashCode 定位桶，再用 equals 比较。如果两个对象 equals 相等但 hashCode 不同，会导致 HashMap 中出现 "重复 Key" 的 bug。

**追问：深拷贝和浅拷贝的区别？**

```java
// 浅拷贝：只拷贝基本类型，引用类型仍然指向原对象
public class User implements Cloneable {
    private Address address;  // 浅拷贝时不会复制 address 对象

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone(); // 浅拷贝
    }
}

// 深拷贝：把引用类型也一起复制
@Override
protected User clone() throws CloneNotSupportedException {
    User user = (User) super.clone();
    user.address = (Address) this.address.clone(); // 深拷贝
    return user;
}
```

深拷贝实现方式：
1. 实现 Cloneable + 递归 clone()
2. 序列化 + 反序列化（实现 Serializable）
3. 通过拷贝构造器或工厂方法
