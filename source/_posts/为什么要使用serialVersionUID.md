---
title: 为什么要使用serialVersionUID
comments: true
date: 2019-05-19 17:53:52
tags:
    - 序列化
---

### 1. 序列化是什么
- **把对象转换为字节序列的过程称为对象的序列化 。**
- **把字节序列恢复为对象的过程称为对象的反序列化。**

### 2. Java中如何使用序列化
只需要让对象实现 `Serializable`接口即可，如下：
<!-- more -->
```
/**
 * @author lirong
 * @desc 序列化类
 * @date 2019/05/18 12:04
 */
public class Person implements Serializable {

    private static final long serialVersionUID = 1L;

    private int id;
    private String name;

    public Person(int id, String name) {
        this.id = id;
        this.name = name;
    }
    
    @Override
    public String toString() {
        return "Person: " +
                ", id: "+ id +
                ", name: " + name;
    }
}
```
让我们来看看`Serializable` 接口都帮我们做了什么工作？

```
 If a serializable class does not explicitly declare a serialVersionUID,
   then the serialization runtime will calculate a default 
   serialVersionUID value for that class based on various aspects of the class, 
  as described in the Java(TM) Object Serialization Specification. 
```
大致意思就是：如果没有为Class显示的声明`serialVersionUID `的话，运行时JVM将默认为Class根据Class的各个字段属性生成一个`serialVersionUID `，具体生成方式就不做研究了，大致应该是根据各字段的hash值凑成的吧。
### 3. 测试代码
**3.1 序列化测试类：**
```
/**
 * @author lirong
 * @desc 序列化测试类
 * @since 2019/05/18 12:09
 */
public class SerialTest {

    public static void main(String[] args) throws IOException {
        Person person = new Person(1234, "Janche");
        System.out.println("Person Serial" + person);
        FileOutputStream fos = new FileOutputStream("Person.txt");
        ObjectOutputStream oos = new ObjectOutputStream(fos);
        oos.writeObject(person);
        oos.flush();
        oos.close();
    }
}
```
**3.2 反序列化测试类**
```
/**
 * @author lirong
 * @desc 反序列化测试类
 * @date 2019/05/18 12:16
 */
public class DeserialTest {

    public static void main(String[] args) throws IOException, ClassNotFoundException {
        FileInputStream fis = new FileInputStream("Person.txt");
        ObjectInputStream ois = new ObjectInputStream(fis);
        Person person = (Person) ois.readObject();
        ois.close();
        System.out.println("Person Deserial" + person);
    }
}

```
>#### 4. 测试条件
>a. 不实现`Serializable`接口
>b. 实现`Serializable`接口
>c. 未自定义`serialVersionUID `字段
>d. 自定义`serialVersionUID `字段
>e. 反序列化前增加字段
>f. 反序列化前减少字段
>g. 反序列化前修改字段的类型
>**说明：反序列化前 表示 在序列化操作完成之后到反序列化开始之前这段时间，更改`Person`类的字段**

>##### 5. 测试结果
>1.只要有a出现,就会导致`java.io.NotSerializableException`异常，即无法完成序列化。
>2.当条件为 b+c  或者 b+d 时，序列化和反序列化均正常执行。
>3.当条件为 b+c+e 或者 b+c+f 时，反序列化时将抛出`InvalidClassException`异常。
>4.当条件为 b+d+e 或者 b+d+f 时，序列化和反序列一切正常，增加字段时，反序列化出来该字段为缺省值，减少字段时，反序列化出来也将没有此字段。
>5.当条件包含 g 时，反序列化也将抛出`InvalidClassException`异常。

>##### 6. 测试总结
>**1. 尽量为每一个需要序列化的类自定义`serialVersionUID `值，这样可避免因为Class中字段的修改而带来的反序列化异常。**
>**2. 尽量不要修改序列化类中的字段类型，如果一定要修改，请重新序列化后再反序列化。**
>**3. 如果要为序列化的类中忽略某个字段，可添加`transient`关键字**


