

### Java类加载器的委托机制

http://ifeve.com/jvm-classloader/



Java 类加载器使用的是委托机制，也就是一个类加载器在加载一个类时候会首先尝试让父类加载器来加载。那么问题来了，为啥使用这种方式？

使用委托第一这样可以避免重复加载，第二，考虑到安全因素，下面我们看下ClassLoader类的loadClass方法：



