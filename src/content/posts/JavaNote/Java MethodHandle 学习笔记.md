---
title: Java MethodHandle学习笔记
published: 2026-03-23
description: "探索Java中的MethodHandle 使用"
# image: "./cover.jpeg"
tags: ["Java", "反射", "JVM"]
category: Java
draft: false
---

# Java MethodHandle 学习笔记

最近在研究 Java 动态调用的新玩法，翻到了一段 MethodHandle 的示例代码，一开始看着一堆陌生的 API 一头雾水：

`Lookup`？`MethodType`？这和我之前学的反射有啥区别？

索性花了一下午把这堆东西啃透，整理成这篇学习笔记，既是记录，也希望能帮到同样对这块懵圈的朋友。

---

## 引子：那段让我一脸懵的代码

最开始我看到的是这段代码，功能很简单，就是动态调用不同对象的 `sound()` 方法：

```java
import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodHandles;
import java.lang.invoke.MethodType;

public class MethodHandleDemo {
    static class Bike {
        String sound() {
            return "ding ding";
        }
    }

    static class Animal {
        String sound() {
            return "wow wow";
        }
    }
    static class Man extends Animal {
        @Override
        String sound() {
            return "hou hou";
        }
    }
    String sound(Object o) throws Throwable {
        // 这行是啥？我第一个疑问
        MethodHandles.Lookup lookup = MethodHandles.lookup();
        // 这又是啥？MethodType？
        MethodType methodType = MethodType.methodType(String.class);
        MethodHandle methodHandle = lookup.findVirtual(o.getClass(), "sound", methodType);

        // 这个invoke和反射的invoke一样吗？
        String obj = (String) methodHandle.invoke(o);
        return obj;
    }

    public static void main(String[] args) throws Throwable {
        String str = new MethodHandleDemo().sound(new Bike());
        System.out.println(str);
        str = new MethodHandleDemo().sound(new Animal());
        System.out.println(str);
        str = new MethodHandleDemo().sound(new Man());
        System.out.println(str);
    }
}
```

运行结果很简单，就是输出不同对象的声音：

```Plain Text
ding ding
wow wow
hou hou
```

但代码里这几个陌生的 API，让我产生了一连串的疑问，今天就把这些疑问一个个解开。

---

## 第一个疑问：`MethodHandles.lookup()` 到底是个啥？

我第一个搞不懂的就是这行：

```java
MethodHandles.Lookup lookup = MethodHandles.lookup();
```

这玩意儿创建出来的对象，到底是干嘛的？

### 一句话讲懂：它是你的「方法搜查证」

查了源码我才发现，这玩意儿的源码居然就一行：

```java
public static Lookup lookup() {
    return new Lookup(Reflection.getCallerClass(), LOOKUP_FLAGS);
}
```

哦！原来它创建 Lookup 的时候，偷偷藏了两个关键信息：

1. **你的身份**：`callerClass` —— 它记住了，**是你这个类创建的我**

2. **你的权限**：`LOOKUP\_FLAGS` —— 它带着你这个类的全部访问权限

说白了，Lookup 就是：

> 你当前类的「代理人」，带着你的身份和权限，帮你去别的类里找方法。
> 
> 

就像警察带着搜查证去搜房子：

- 警察 = Lookup

- 搜查证 = 你的访问权限

- 他不能越权，你在代码里本来访问不到的方法，他也访问不到

### 那我想找别的类的方法，怎么用？

很简单！Lookup 本身不用换，你只要在找方法的时候，指定**目标类**就行了。

#### 1\. 调用别的类的 public 方法

比如我有个别的类 `Car`：

```java
class Car {
    public String run() {
        return "car running";
    }
}
```

用 Lookup 调用它，只需要把目标类传进去：

```java
// 还是同一个 lookup
MethodHandles.Lookup lookup = MethodHandles.lookup();

// 指定目标类是 Car
MethodType mt = MethodType.methodType(String.class);
MethodHandle mh = lookup.findVirtual(Car.class, "run", mt);

// 调用
Car car = new Car();
String result = (String) mh.invoke(car);
```

#### 2\. 调用静态方法

如果是静态方法，用 `findStatic` 就行，不用传对象：

```java
class Tool {
    public static String hello() {
        return "hello";
    }
}

// 查找静态方法
MethodHandle mh = lookup.findStatic(Tool.class, "hello", mt);
String s = (String) mh.invoke(); // 不需要传对象
```

#### 3\. 调用构造方法

创建对象也可以，用 `findConstructor`：

```java
// 构造方法的返回值固定是 void.class
MethodType mt = MethodType.methodType(void.class);
MethodHandle mh = lookup.findConstructor(Car.class, mt);

Car car = (Car) mh.invoke();
```

#### 4\. 想访问别的类的 private 方法？

默认的 lookup 是带你的权限的，你访问不到别人的 private 方法，强行调用会抛 `IllegalAccessException`。

Java 9\+ 提供了 `privateLookupIn`，可以拿到目标类的权限：

```java
// 拿到 Other 类的权限 lookup
Lookup privateLookup = MethodHandles.privateLookupIn(Other.class, lookup);
// 现在就能访问它的 private 方法了
MethodHandle mh = privateLookup.findVirtual(Other.class, "privateMethod", mt);
```

---

## 第二个疑问：MethodType 又是个啥？

搞懂了 Lookup，我又懵了：那 `MethodType.methodType(String.class)` 这行是干嘛的？

### 一句话讲懂：它是方法的「签名模板」

你要告诉 JVM：**我要找的方法，长什么样？**

- 返回值是什么？

- 参数是什么？顺序是什么？

就这么简单！

### 万能公式，背下来就会用

```java
MethodType.methodType( 返回值类型, 参数1类型, 参数2类型... );
```

就这么一行，所有情况都能覆盖。

我整理了一张速查表，以后直接抄：

|方法签名|MethodType 怎么写|
|---|---|
|`String fun\(\)`|`methodType\(String\.class\)`|
|`int add\(int a,int b\)`|`methodType\(int\.class, int\.class, int\.class\)`|
|`void test\(\)`|`methodType\(void\.class\)`|
|`User create\(String name, int age\)`|`methodType\(User\.class, String\.class, int\.class\)`|

### 注意几个坑

1. 基本类型直接写：`int.class`、`double.class`，别用包装类

2. 返回值 `void` 固定写 `void.class`

3. 参数顺序必须和方法完全一致，错一个都找不到方法

---

## 第三个疑问：都是 `invoke`，它和反射的 `invoke` 有啥区别？

最后我发现，不管是 MethodHandle 还是反射，调用的时候都叫 `invoke`？这俩名字一样，难道是同一个东西？

查完我才发现，这俩**名字一样，底层完全不是一个东西**，差了十万八千里！

我给你整理了一张对比表，一眼就能看明白：

|对比维度|反射 `Method\.invoke`|MethodHandle `invoke`|
|---|---|---|
|**本质**|Java 层模拟调用，每次都要绕路|JVM 原生方法指针，直接调用|
|**性能**|慢，每次都要做安全检查、参数包装|快，接近原生调用，JIT 能直接优化|
|**异常处理**|把方法异常包装成 `InvocationTargetException`，要 `getCause()` 才能拿到真实异常|原样抛出异常，方法抛啥它抛啥，甚至能直接抛受检异常|
|**参数处理**|必须把参数包装成 `Object[]` 数组|直接传参，和正常方法调用一样，不需要包装|
|**调用体验**|繁琐，一堆异常要处理|简洁，就像直接调用方法|

### 通俗比喻

- **反射 invoke**：你要进朋友家，**每次都要去物业登记、安检、搜身**，折腾半天才能进去

- **MethodHandle invoke**：你**直接拿朋友家的钥匙开门**，没人拦你，进去就完了

### 举个异常的例子

反射调用的时候，如果方法内部抛了异常，你得这么写：

```java
try {
    method.invoke(obj);
} catch (InvocationTargetException e) {
    // 真实异常藏在里面！
    Throwable realEx = e.getCause();
}
```

但 MethodHandle 就简单多了，它直接把异常抛给你：

```java
// 方法抛啥，这里就直接抛啥，不用包装
String obj = (String) methodHandle.invoke(o);
```

这也是为什么我们的方法要声明 `throws Throwable`，因为它能直接抛出任何异常。

---

## 最后：学完我才明白，MethodHandle 到底好在哪？

以前我一直以为，Java 动态调用就只有反射这一种方式。

学完 MethodHandle 我才发现，这玩意儿简直是反射的「升级版」：

1. **快太多了**：性能接近原生调用，比反射快几倍到几十倍

2. **更灵活**：可以做方法的转换、适配，甚至把方法当成变量传来传去

3. **更安全**：权限控制更清晰，不会像反射那样随便就打破封装

4. **用起来更顺手**：API 更直观，不用再处理一堆包装异常

现在很多框架，比如 Spring、MyBatis，都在大量用 MethodHandle 来替代反射，就是因为它的性能和灵活性。

---

## 总结

回头看最开始那段懵圈的代码，现在再看，每一行都清晰了：

```java
// 1. 创建一个带着我权限的查找器
MethodHandles.Lookup lookup = MethodHandles.lookup();
// 2. 定义我要找的方法的样子：无参，返回 String
MethodType methodType = MethodType.methodType(String.class);
// 3. 用查找器，找到目标对象的 sound 方法
MethodHandle methodHandle = lookup.findVirtual(o.getClass(), "sound", methodType);
// 4. 直接调用，几乎和原生调用一样快
String obj = (String) methodHandle.invoke(o);
```

原来这堆陌生的 API，背后是这么清晰的逻辑。

如果你也和我一样，之前只知道反射，不妨试试 MethodHandle，打开新世界的大门～
