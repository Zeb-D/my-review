# Java Exception和Error

<br>

## Exception和Error的联系

1. Exception和Error都继承自Throwable.
2. RuntimeException继承自Exception.
3. Error和RuntimeException及其子类称称为未检查的异常(Unchecked Exception）,其他异常称为受检查异常(Checked Exception). 

![exception和Error](../image/exception和Error.jpg)

<br>

## Exception与Error的区别

1.Error类一般是指与虚拟机相关的问题，如系统崩溃，虚拟机错误，内存空间不足，方法调用栈溢出等。如：java.lang.StackOverFlowError和java.lang.OutOfMemory,对于这类错误，java编译器不去检查他们，这类错误的导致的应用程序中断，仅靠程序本身无法恢复和预防，遇到这样的错误，建议程序终止。

```java
/**
 * An {@code Error} is a subclass of {@code Throwable}
 * that indicates serious problems that a reasonable application
 * should not try to catch. Most such errors are abnormal conditions.
 * The {@code ThreadDeath} error, though a "normal" condition,
 * is also a subclass of {@code Error} because most applications
 * should not try to catch it.
 * <p>
 * A method is not required to declare in its {@code throws}
 * clause any subclasses of {@code Error} that might be thrown
 * during the execution of the method but not caught, since these
 * errors are abnormal conditions that should never occur.
 *
 * That is, {@code Error} and its subclasses are regarded as unchecked
 * exceptions for the purposes of compile-time checking of exceptions.
 * Compile-Time Checking of Exceptions
/
```

2.Exception类表示程序可以处理的异常，可以捕获且可能恢复，遇到这类异常，应该尽可能处理异常，使程序恢复运行，而不应该随意终止异常。

<br>

## checked异常和unchecked异常的区别

Checked Exception:继承自Exception类是checked exception,代码需要处理API抛出的checked exception,要么用catch语句，要么直接throws语句抛出去。

 Unchecked Exception:也成为RuntimeException,它是继承自Exception,但所有的RuntimeException的子类都有一个特点，就是代码不需要处理他们的异常也能通过编译，所以称为unchecked exception.

 总结： **检查性异常：不处理编译就不能通过，也称为非运行时异常** **非检查性异常：不处理编译可以通过，如果有抛出直接抛到控制台，也称为运行是异常** Throws和Throw的区别 throws总是出现在一个函数的头中，用来标明该成员函数可能抛出的各种异常，对于大多数Exception子类来说，java编译器会强迫声明在一个成员函数中抛出的异常的类型，如果异常的类型是Error或RuntimeException。



