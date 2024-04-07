
# 事务事件

用于在一个事务执行后提交事件或者回滚事件时执行 相应逻辑

[TransactionalEventListener使用场景以及实现原理，最后要躲个大坑 - 掘金 (juejin.cn)](https://juejin.cn/post/7011685509567086606)

这里举个业务场景，假如我们有个需求，用户创建成功后给用户发送一个邮件。这里有两个事情要做：

1. 创建用户
2. 给用户发送邮件

对于这种需求，我们可能会不假思索的有以下实现。

```java
@Entity  
public class User {  
    @Id  
    @GeneratedValue(strategy = GenerationType.IDENTITY)  
    private Long id;  
    private String name;  
    private String email;  
  
    public User() {}  
    ...  
    //getters  
    //equals and hashcode}
```


为User创建个Repository

```java
@Entity  
public class User {  
    @Id  
    @GeneratedValue(strategy = GenerationType.IDENTITY)  
    private Long id;  
    private String name;  
    private String email;  
  
    public User() {}  
...  
    //getters  
    //equals and hashcode}

```

对于上面的实现，是最容易实现的，但这种实现是有问题的。我们想一下，这个功能的核心是创建用户，而发送邮件是一个副作用（发送邮件不能影响用户的创建），如果把这两个操作放在一个事务中会有什么问题？其实很明显，如果创建用户时抛出异常，事务回滚，方法提前退出，那么也不会发送邮件，这是正常的。但是下面两个场景是不可接受的：

1. 如果邮件发送失败，事务发生回滚，用户创建失败。
2. 如果邮件发送成功后，事务提交失败，这下就尴尬了，用户收到了邮件，可是用户创建失败。

虽然这些情况出现的概率很小，但作为对自己有要求的程序猿，这是不可容忍的，我们要对自己写的业务负责。

好了，我们对上面的实现做个重构，直接将创建用户和发送邮件的业务代码拆开，使用Spring application event的方式解耦实现。

修改后的Service是这样的


```java
  
@Service  
public class CustomerService {  
  
    private final UserRepository userRepository;  
    private final ApplicationEventPublisher applicationEventPublisher;  
  
    public CustomerService(UserRepository userRepository, ApplicationEventPublisher applicationEventPublisher) {  
        this.userRepository = userRepository;  
        this.applicationEventPublisher = applicationEventPublisher;  
    }  
  
    @Transactional  
    public Customer createCustomer(User user) {  
        User newUser = userRepository.save(user);  
        final UserCreatedEvent event = new UserCreatedEvent(newUser);  
        applicationEventPublisher.publishEvent(event);  
        return newUser;  
    }  
}
```

从上面的代码，我们知道**UserService**依赖两个beans：

1. UserRepository - 做持久化工作
    
2. ApplicationEventPublisher - 发送Spring内部事件
    
    public class UserCreatedEvent {
    
    private final User user;
    
    public UserCreatedEvent(User user) { this.user = user; }
    
    public User getUser() { return user; }
    
    ... //equals and hashCode }
    

注意这个类只是个简单POJO对象，自从Spring 4.2，我们不用继承**ApplicationEvent**而能发布任何对象，Spring会把它们包装成**PayloadApplicationEvent**。

我们需要一个Event Listener处理上面的事件。

```java
	@Component  
public class UserCreatedEventListener {  
    private final EmailService emailService;  

    public UserCreatedEventListener(EmailService emailService) {  
        this.emailService = emailService;  
    }  
  
    @EventListener  
    public void processUserCreatedEvent(UserCreatedEvent event) {  
        emailService.sendEmail(event.getUser().getEmail());  
    }  
}
```
通过上面的重构，我们将创建用户和发送邮件的业务代码拆开来了，但是有解决上面提到的问题吗？答案是没有，虽然我们用**EventListener**的方式解耦了业务代码，可是这在底层两个功能还是在同一个事务中执行（有人可能想问在Listener方法上加@Async让异步执行可以吗？当然不行，邮件必须在用户创建成功后发送，这里有业务依赖），意思就是，上面的两种情况依然会发生。那么问题来了，有没有解决方案呢?

当然有，就是用@TransactionalEventListener替换@EventListener，结果就是在创建用户并提交事务后发送邮件通知。

### TransactionalEventListener

TransactionalEventListener是对EventListener的增强，被注解的方法可以在事务的不同阶段去触发执行，如果事件未在激活的事务中发布，除非显式设置了 **fallbackExecution()** 标志为true，否则该事件将被丢弃；如果事务正在运行，则根据其 TransactionPhase 处理该事件。

Notice：你可以通过注解@Order去排序所有的Listener，确保他们按自己的设定的预期顺序执行。

我们先看看TransactionPhase有哪些：

- AFTER_COMMIT - 默认设置，在事务提交后执行
- AFTER_ROLLBACK - 在事务回滚后执行
- AFTER_COMPLETION - 在事务完成后执行（不管是否成功）
- BEFORE_COMMIT - 在事务提交前执行

改造后的Listener是这样的

```java
@Component  
public class UserCreatedEventListener {  
    private final EmailService emailService;  
  
    public UserCreatedEventListener(EmailService emailService) {  
        this.emailService = emailService;  
    }  
  
    @TransactionalEventListener  
    public void processUserCreatedEvent(UserCreatedEvent event) {  
        emailService.sendEmail(event.getUser().getEmail());  
    }  
}
```

好了，现在我们能保证我们的业务正常的运行，创建用户不会受发送邮件的影响。接下来我们深挖一下，看看TransactionalEventListener是怎么做到的。

### 原理分析

下面给出Spring的处理源码，大家会一目了然：

```java
@Override  
public void onApplicationEvent(ApplicationEvent event) {  
    if (TransactionSynchronizationManager.isSynchronizationActive() && TransactionSynchronizationManager.isActualTransactionActive()) {  
        TransactionSynchronization transactionSynchronization = createTransactionSynchronization(event);  
    } else if (this.annotation.fallbackExecution()) {  
        if (this.annotation.phase() == TransactionPhase.AFTER_ROLLBACK && logger.isWarnEnabled()) {  
            logger.warn("Processing " + event + " as a fallback execution on AFTER_ROLLBACK phase");  
        }  
        processEvent(event);  
    } else {            // No transactional event execution at all          
		if (logger.isDebugEnabled()) {  
            logger.debug("No transaction is active - skipping " + event);  
        }  
    }  
}
```
解释一下上面的代码：

1. 如果当前处于激活的事务当中，那么会创建一个**TransactionSynchronization**，并把它放到一个集合当中。意思就是先不执行，只是临时存了起来。
2. 如果没有事务，并且明确设置了fallbackExecution为true，那么直接执行，该效果和EventListener一样。
3. 如果没有事务，并且fallbackExecution 为false，那么直接丢弃该Event不做任何处理。

既然将TransactionSynchronization存放了起来，那么什么时机触发执行呢？

这里以**AFTER_COMMIT**为例（其他阶段实现差不多），看这段代码：

AbstractPlatformTransactionManager类

```java
private void processCommit(DefaultTransactionStatus status) throws TransactionException {  
        boolean beforeCompletionInvoked = false;  
        boolean unexpectedRollback = false;  
        prepareForCommit(status);  
        triggerBeforeCommit(status);  
        triggerBeforeCompletion(status);  
        beforeCompletionInvoked = true;  
        if (status.hasSavepoint()) {  
            if (status.isDebug()) {  
                logger.debug("Releasing transaction savepoint");  
            }  
            unexpectedRollback = status.isGlobalRollbackOnly();  
            status.releaseHeldSavepoint();  
        } else if (status.isNewTransaction()) {  
            if (status.isDebug()) {  
                logger.debug("Initiating transaction commit");  
            }  
            unexpectedRollback = status.isGlobalRollbackOnly();  
            doCommit(status);  
        } else if (isFailEarlyOnGlobalRollbackOnly()) {  
            unexpectedRollback = status.isGlobalRollbackOnly();  
        }
```
接着看**triggerAfterCommit**的实现

```java
private void triggerAfterCommit(DefaultTransactionStatus status) {  
    if (status.isNewSynchronization()) {  
        if (status.isDebug()) {  
            logger.trace("Triggering afterCommit synchronization");  
        }  
        TransactionSynchronizationUtils.triggerAfterCommit();  
    }  
}
```


这里调用了TransactionSynchronizationUtils的triggerAfterCommit方法，继续往下跟

```java
public static void triggerAfterCommit() {  
    invokeAfterCommit(TransactionSynchronizationManager.getSynchronizations());  
}  
  
public static void invokeAfterCommit(@Nullable List<TransactionSynchronization> synchronizations) {  
    if (synchronizations != null) {  
        for (TransactionSynchronization synchronization : synchronizations) {  
            synchronization.afterCommit();  
        }  
    }  
}
```

看到了吧，先是拿到所有的**TransactionSynchronization**，然后调用他们的afterCommit方法，就会真正开始处理该Event。

### 总结

现在我们做一个总结，如果你遇到这样的业务，操作B需要在操作A事务提交后去执行，那么TransactionalEventListener是一个很好地选择。这里需要**特别注意**的一个点就是：当B操作有数据改动并持久化时，并希望在A操作的AFTER_COMMIT阶段执行，那么你需要将B事务声明为**PROPAGATION_REQUIRES_NEW**。这是因为A操作的事务提交后，事务资源可能仍然处于激活状态，如果B操作使用默认的**PROPAGATION_REQUIRED**的话，会直接加入到操作A的事务中，但是这时候事务A是不会再提交，结果就是程序写了修改和保存逻辑，但是数据库数据却没有发生变化，解决方案就是要明确的将操作B的事务设为**PROPAGATION_REQUIRES_NEW**。
