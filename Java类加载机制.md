# Java类加载机制

## 一.类加载过程

![未命名文件.png](https://i.loli.net/2019/12/12/1ex5Ao3mPjBCwhf.png)



### 1.加载

通过一个类的全限定名来获取定义此类的二进制字节流

将这个字节流所代表的静态存储结构转换为方法区的运行时数据结构

在内存中生成一个嗲表这个类的Java.lang.Class对象，作为方法区这个类的各种数据结构的访问入口

### 2.连接

#### 2.1验证

- 目的：

  确保Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全

- 验证内容
  - 文件格式验证
  - 元数据验证
  - 字节码验证
  - 符号引用验证

#### 2.2准备

#### 2.3解析

### 3.初始化

### 4.使用

### 5.卸载 



##　二.类加载顺序

  	虚拟机把描述类的数据从 Class 文件加载到内存，并对数据进行校验，解析和初始化，最终形成可以被虚拟机直接使用的 java 类型。 

## 双亲委派机制

### 加载过程
​	如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的加载器都是这样的，因此所有的加载请求最终都应该传送到顶层的启动类加载器中，只有父类加载器在它的搜索范围中无法找到所需的类时，子类才会自己去加载

### 优点：
- JAVA类随着加载器具备带有优先级的关系，顶级类(例如java.lang.Object)都是由启动类加载器去做加载 
### 源码分析

## 三.类加载器

### 启动类加载器
​	负责加载<JAVA_HOME>/lib或被-Xbootclasspath指定路径下的类库，开发者不可以直接使用

### 扩展类加载器
​	负责加载<JAVA_HOME>/lib/ext或被java.ext.dirs系统变量指定的路径中的所有类库，开发者可以直接使用

### 系统类加载器
​	这个类加载器是ClassLoader.getSystemClassLoader()的返回值，负责加载用户类路径上所指定的类库，开发者可以直接使用这个类加载器，如果应用程序没有自定义过类加载器，那么系统默认使用这个类加载器。

### 用户自定义类加载器   

### 注意：
​	JVM虚拟机总共定义了四种类加载器，因为双亲委派模型的要求，类加载器之间的父子关系不是以继承关系来实现的，都是通过组合的方式实现。

![微信图片_20191211235351.png](https://i.loli.net/2019/12/12/TeDB7uHaVygzirE.png)


## 四.关于类加载的几个示例代码

### 被动引用

```java
package org.jake.leanring.classloader;

public class SuperClass {

    static {
        System.out.println("SuperClass Init");
    }

    public static int value = 123;

}
```

````java
package org.jake.leanring.classloader;

public class SubClass extends SuperClass {

    static {
        System.out.println("subClass init");
    }

}
````

```java
package org.jake.leanring.classloader;

public class TestMain {
    public static void main(String[] args) {
        System.out.println(SubClass.value);
    }
}



```

#### 输出结果如下：

​	只输出父类SuperClass Init，没有输出SubClass Init

![微信图片_20191211235351.png](https://i.loli.net/2019/12/11/2Pj3dosQtIuXDUC.png)



#### 分析：

​	对于静态字段，只有直接定义这个字段的类才会被初始化，因此通过其子类来引用父类中定义的静态字段，只会出发父类的初始化而不会触发子类的初始化

### 2通过只数组定义来引用类，不会触发类初始化

```java
package org.jake.leanring.classloader;

public class NotInitialization {

    public static void main(String[] args) {
        SuperClass[] superClasses = new SuperClass[111];
    }
}
```

#### 输出结果如下：

​	什么都没有输出，没有输出SubClass Init信息

![微信图片_20191212000323.png](https://i.loli.net/2019/12/12/cK6lj4eqfID2hpF.png)

#### 分析

​	对于引用类型的一维数组，它是由虚拟机自动生成的，直接继承于java.lang.Object的子类，创建动作由字节码指令newarrray触发


### 常量引用问题

```java
package org.jake.leanring.classloader;

public class ConstClass {
    static {
        System.out.println("ConstClass");
    }

    public static final String HELLO_WORLD = "hello world";
}

```

```java
package org.jake.leanring.classloader;

public class ConstClassTestMain {
    public static void main(String[] args) {
        System.out.println(ConstClass.HELLO_WORLD);
    }
}

```

#### 输出结果如下：

​	未打印出 ConstClass 信息

![微信图片_20191211235351.png](https://i.loli.net/2019/12/12/pDKULis7OuSZdNy.png)

#### 分析

​	常量在编译阶段会存入调用类的常量池中，本质上并没有直接引用到定义常量的类，因此不会触发定义常量类的初始化

## 五.带深入的地方

- 双亲委派模型的破外问题

> 参考 深入理解Java虚拟机 第二版