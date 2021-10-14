## `TransactionSynchronizationManager`事务同步管理器

### `TransactionSynchronizationManager`理解

`TransactionSynchronizationManager`使用多个`ThreadLocal`维护每个线程事务独立的资源信息，配合`AbstractPlatformTransactionManager`以及其子类，
设置线程事务的配置属性和运行态，并可通过自定义实现`TransactionSynchronization`进行注册，监听事务的操作
举个业务场景：假设业务流程最后有个异步操作，业务流程依赖事务，且异步操作依赖业务流程在事务中新增或者更新的数据，
这会出现一个问题，异步操作在事务还没提交前执行了，查询的数据是未提交前的，导致异步操作无效
那么可以使用`Spring`的`TransactionSynchronizationManager`注册自定义的`TransactionSynchronization`，实现`afterCommit`方法（也有其他方法），
使得在事务的那个运行状态下，执行自定义的业务逻辑

### `TransactionSynchronizationManager`源码

```java
public abstract class TransactionSynchronizationManager {
    //线程上下文中保存着【线程池对象：ConnectionHolder】的Map对象。线程可以通过该属性获取到同一个Connection对象。
	private static final ThreadLocal<Map<Object, Object>> resources = new NamedThreadLocal<>("Transactional resources");
    //事务同步器，事务运行时的扩展代码，每个线程可以注册N个事务同步器。
	private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations = new NamedThreadLocal<>("Transaction synchronizations");
    //事务名称
	private static final ThreadLocal<String> currentTransactionName = new NamedThreadLocal<>("Current transaction name");
    //事务是否只读
	private static final ThreadLocal<Boolean> currentTransactionReadOnly = new NamedThreadLocal<>("Current transaction read-only status");
    //事务隔离级别
	private static final ThreadLocal<Integer> currentTransactionIsolationLevel = new NamedThreadLocal<>("Current transaction isolation level");
    //事务是否开启
	private static final ThreadLocal<Boolean> actualTransactionActive = new NamedThreadLocal<>("Actual transaction active");

}
```
在`org.springframework.transaction.interceptor.TransactionInterceptor#invoke`中，对事务方法进行拦截处理。在`createTransactionIfNecessary`里执行`getTransaction`时，会调用`AbstractPlatformTransactionManager#prepareSynchronization`方法初始化事务同步器。
```java
/**
 * Initialize transaction synchronization as appropriate.
 */
protected void prepareSynchronization(DefaultTransactionStatus status, TransactionDefinition definition) {
	if (status.isNewSynchronization()) {
		TransactionSynchronizationManager.setActualTransactionActive(status.hasTransaction());
		TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(
				definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT ?
						definition.getIsolationLevel() : null);
		TransactionSynchronizationManager.setCurrentTransactionReadOnly(definition.isReadOnly());
		TransactionSynchronizationManager.setCurrentTransactionName(definition.getName());
		TransactionSynchronizationManager.initSynchronization();
	}
}   
```

### `TransactionSynchronization`扩展

一般常用是`TransactionSynchronization`的afterCommit和afterCompletion方法

```java
TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
    @Override
    public void afterCommit() {
        //自定义业务操作
    }
});
```
哪里触发`afterCommit`和`afterCompletion`呢？

在`AbstractPlatformTransactionManager`的`processCommit`方法中进行回调所有`TransactionSynchronization`
```java
private void processCommit(DefaultTransactionStatus status) throws TransactionException {
    try {
        //提交事务
        doCommit(status);
        ...
        try {
           //回调所有事务同步器的afterCommit方法。
           triggerAfterCommit(status);
        }
        finally {
           //回调所有事务同步器的afterCompletion方法。
           triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
        }

    }
    finally {
        //清除TransactionSynchronizationManager的ThreadLocal绑定的数据。
        //解除Thread绑定的resources资源。
        //将Commit设置为自动提交。
        //清理ConnectionHolder资源。
        cleanupAfterCompletion(status);
    }
}
```

