# Spring注解@Transactional失效

@Transactional注解在Spring框架中用于声明式事务管理，它通过AOP（面向切面编程）技术，在方法执行前后进行事务性的增强处理。

然而，在某些情况下，@Transactional注解可能会失效，导致事务控制不按预期执行。

以下是一些常见的@Transactional失效场景：



### 1. 同一个类中方法调用

当在同一个类中，一个方法调用另一个使用@Transactional注解的方法时，事务可能不会生效。

这是因为Spring的AOP代理是基于接口的（如果使用JDK动态代理）或基于类的（如果使用CGLIB代理），而在同一个类中的方法调用不会通过代理对象，因此事务拦截器无法捕获到事务注解。

**解决方案**：

- • 将需要事务支持的方法移到另一个类中。
- • 在原类中注入自己，通过代理对象调用事务方法。



### 2. 方法被final、static修饰

被final或static修饰的方法不能被代理，因为它们要么不能被重写（final），要么不依赖于类的实例（static）。

因此，这些方法上的@Transactional注解会失效。

**解决方案**：

- • 移除final或static修饰符。



### 3. 访问权限问题

Spring AOP要求被代理的方法必须是public的。

如果@Transactional注解应用在非public（如private、protected、默认访问权限）修饰的方法上，事务将不会生效。

**解决方案**：

- • 将方法的访问权限改为public。



### 4. 异常处理不当

如果事务方法中的异常被捕获且没有被重新抛出，或者抛出的异常类型不在@Transactional的rollbackFor属性指定的范围内，事务将不会回滚。

**解决方案**：

- • 确保捕获异常后重新抛出。
- • 正确设置rollbackFor属性，包括所有需要触发事务回滚的异常类型。



### 5. 事务传播行为设置错误

@Transactional的propagation属性定义了事务的传播行为。

如果设置了不支持事务的传播行为（如PROPAGATION_NOT_SUPPORTED、PROPAGATION_NEVER），则事务将不会生效。

**解决方案**：

- • 检查并设置正确的事务传播行为，如PROPAGATION_REQUIRED。



### 6. 类未被Spring管理

如果使用了@Transactional注解的类没有被Spring容器管理（例如，没有使用@Service、@Component等注解），那么@Transactional注解将不会生效。

**解决方案**：

- • 确保类被Spring容器管理，通常是通过添加@Service、@Component等注解来实现的。



### 7. 多线程调用

在Spring中，事务是通过ThreadLocal来管理的，这意味着事务只在当前线程中有效。

如果在一个事务方法中启动了新线程来执行数据库操作，那么这些操作将不会在当前事务中执行。

**解决方案**：

- • 避免在事务方法中启动新线程来执行数据库操作。



### 8. 数据库不支持事务

如果使用的数据库或表不支持事务（如MyISAM存储引擎），那么@Transactional注解将不会生效。

**解决方案**：

- • 确保数据库和表支持事务，例如使用InnoDB存储引擎。



### 失效示例

```
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    // 这个方法被final修饰，导致@Transactional失效
    @Transactional
    public final void createUserFinal(User user) {
        userRepository.save(user);
        // 假设这里有一些逻辑，可能抛出异常
        if (user.getName().equals("invalid")) {
            throw new RuntimeException("Invalid user name");
        }
    }

    // 这个方法调用createUserFinal，由于是在同一个类中调用，@Transactional可能失效（取决于代理方式）
    public void createUserWrapper(User user) {
        createUserFinal(user);
    }
    
    // 正确的做法应该是将需要事务管理的方法放在不同的类中，或者通过注入的代理对象调用
}

// 假设这是UserRepository的接口定义，它可能是Spring Data JPA的一部分
interface UserRepository {
    void save(User user);
}

// 假设这是User的实体类
class User {
    private Long id;
    private String name;
    // getters and setters
}

// 配置类或启动类（省略了大部分配置细节）
// @SpringBootApplication
// public class Application { ... }
```

在这个例子中，`createUserFinal`方法被`final`修饰，这意味着它不能被重写，因此Spring AOP无法为它创建代理，导致`@Transactional`注解失效。

另外，即使`final`修饰符被移除，`createUserWrapper`方法直接调用`createUserFinal`也可能导致`@Transactional`失效，因为这通常不会通过代理对象进行调用（除非使用了自代理或类似的机制）。

要修复这个问题，可以将需要事务管理的方法放在不同的服务类中，或者通过注入的代理对象（例如使用`@Autowired`注入当前类的代理）来调用事务方法。



## 小结

| 序号 | 失效场景                                   | 原因说明                                                     |
| ---- | ------------------------------------------ | ------------------------------------------------------------ |
| 1    | 同一个类中方法调用                         | Spring AOP基于代理实现，类内部方法调用不会通过代理，因此事务拦截器无法捕获到事务注解。 |
| 2    | 方法被final、static修饰                    | final方法不能被重写，static方法不依赖于类的实例，因此无法被代理。 |
| 3    | 访问权限问题（非public方法）               | Spring AOP要求被代理的方法必须是public的，非public方法上的@Transactional注解将不会生效。 |
| 4    | 异常处理不当                               | 如果事务方法中的异常被捕获且没有被重新抛出，或者抛出的异常类型不在@Transactional的rollbackFor属性指定的范围内，事务将不会回滚。 |
| 5    | 事务传播行为设置错误                       | 如设置了PROPAGATION_NOT_SUPPORTED、PROPAGATION_NEVER等不支持事务的传播行为，事务将不会生效。 |
| 6    | 类未被Spring管理                           | 如果使用了@Transactional注解的类没有被Spring容器管理，那么@Transactional注解将不会生效。 |
| 7    | 多线程调用                                 | Spring事务是基于ThreadLocal的，新线程中的数据库操作不会在当前事务中执行。 |
| 8    | 数据库或表不支持事务                       | 如果使用的数据库或表不支持事务，那么@Transactional注解将不会生效。 |
| 9    | 嵌套事务回滚处理不当                       | 在嵌套事务中，如果内部事务回滚但外部事务没有正确处理，可能导致数据不一致。 |
| 10   | 事务超时设置不合理（timeout属性）          | 如果事务超时时间设置过短，可能导致事务在合理时间内未完成而被强制回滚。 |
| 11   | AOP配置问题（如使用AspectJ而非Spring AOP） | AOP配置错误可能导致@Transactional注解无法被正确识别和处理。  |