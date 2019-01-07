本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

## final域

### 重排序规则

对于 final 域，遵循两个重排序规则：

1. 在构造函数内对一个 final 域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作不能重排序
2. 初次读一个包含 final 域的对象的引用，与随后初次读这个 final 域，这两个操作之间不能重排序。

```java
public class FinalExample{
    int i;
    final int j;
    static FinalExample obj;
    
    public FinalExample(){
        i=1;
        j=2;
    }
    
    public static void writer(){
        obj=new FinalExample();
    }
    
    public static void reader(){
        FinalExample object=obj;
        int a=object.i;
        int b=object.j;
    }
}
```

假设线程A执行 writer() 方法，线程B执行 reader() 方法。

**写final域的重排序规则**写 final 域的重排序规则禁止把 final 域的写重排序到构造函数之外。从而确保了在对象引用被任意线程可见之前，对象的final域已经被正确的初始化过了。在上述的代码中，线程B获得的对象，final域一定被正确初始化，普通域i却不一定。

**读final域的重排序规则** 在一个线程中，初次读对象引用与初次读该对象包含的final域，JMM禁止处理器重排序该操作。从而确保在读一个对象的final域之前，一定会先读包含这个final域的对象的引用

**final域为引用类型** 在构造函数内对一个final引用的对象的写入，与随后在构造函数外把这个被构造对象的引用赋值给一个引用变量，不能重排序。

但是，要得到上述的效果，需要保证在构造函数内部，不能让这个被构造对象的引用被其他线程所见，也就是不能有this逸出。