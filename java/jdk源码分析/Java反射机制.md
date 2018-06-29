# Java反射机制

<br>

<br>

## Java反射机制

<br>

**反射是指在运行时，调用Reflect Api来根据对象或类获得类的构造函数，修饰符，属性，函数，实例化或者调用该类的方法, 并且能够修改对属性的值，这种动态获取类的信息以及动态调用对象的功能，称为反射。**

对任意一个类，都能够知道这个类的所有属性和方法; 对任意一个对象，都能够调用这个对象的属性和方法;

**反射Reflection APIs主要包括： java.lang.Class, java.lang.reflect中的Method,Field,Constructor.**

<br>

## 类的装载：

<br>

在正式开始学习反射之前，需要了解一下JVM类的装载的知识：

### 1.装载：查找和导入Class文件

### 2.链接：执行校验，准备和解析步骤，其中解析步骤是可以选择的

1. 校验：执行校验，准备和解析步骤，其中解析步骤是可以选择的
2. 准备：给类的静态变量分配存储空间
3. 解析：给类的静态变量分配存储空间

### 3.初始化：对类的静态变量，静态代码块执行初始化工作

取得对象的三种方式：

 1、未知类名:Class.forName(className)

 2、已知对象名:new Person().getClass()

 3、已知类名:Person.class;

<br>

<br>

## 获取类的基本信息：

```java
Field[] getDeclaredFiled(); 
Field getDeclaredField(String name); 
Method[] getMethods(); 
Method getMethod(String name,.....) .....
getMethods();//获取public (包括父类中的)
//获取本类声明(包括各种修饰符public、private)
getDeclaredMethods();
getDeclaredMethod(String name)是指获取修饰符为public的并且是方法名为name的方法
```

<br>

<br>

## 获取类的实例：

<br>

1，通过Class对象的newInstance方法，创建此Class对象所表示的类的一个实例 2，通过Constructor对象，获取类的实例

```java
/**
     * 调用私有方法
     * @param obj 调用类对象
     * @param methodName 方法名
     * @param paramTypes  参数类型
     * @param params  参数
     * @return
     * @throws Exception
     */
    public static Object invokePrivateMethod(Object obj, String methodName,
            Class<?>[] paramTypes, Object[] params) throws Exception {

        Object value = null;
        Class<?> cls = obj.getClass();

        // 注意不要用getMethod(),因为getMethod()返回的都是public方法
        Method method = cls.getDeclaredMethod(methodName, paramTypes);

        method.setAccessible(true);// 抑制Java的访问控制检查-AccessController

        value = method.invoke(obj, params);
        return value;
    }
```



