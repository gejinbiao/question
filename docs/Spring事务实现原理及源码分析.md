# Spring事务实现原理及源码分析
# 流程介绍

## 主流程介绍
众所周知，Spring事务采用AOP的方式实现，我们从TransactionAspectSupport这个类开始f分析。

1. 获取事务的属性（@Transactional注解中的配置）
2. 加载配置中的TransactionManager.
3. 获取收集事务信息TransactionInfo
4. 执行目标方法
5. 出现异常，尝试处理。
6. 清理事务相关信息
7. 提交事务

	//1. 获取@Transactional注解的相关参数
	TransactionAttributeSource tas = getTransactionAttributeSource();
	// 2. 获取事务管理器
		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);
	
		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
			// 3. 获取TransactionInfo，包含了tm和TransactionStatus
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
			Object retVal = null;
			try {
				// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
				// 4.执行目标方法
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
			   //5.回滚
				// target invocation exception
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
			  // 6. 清理当前线程的事务相关信息
				cleanupTransactionInfo(txInfo);
			}
			// 提交事务
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}
	}

## doBegin做了什么
在执行createTransactionIfNecessary获取事务状态时，就准备好了开启事务的所有内容，主要是执行了JDBC的con.setAutoCommit(false)方法。同时处理了很多和数据库连接相关的ThreadLocal变量。
# 关键对象介绍

## PlatformTransactionManager
TransactionManager是做什么的？它保存着当前的数据源连接，对外提供对该数据源的事务提交回滚操作接口，同时实现了事务相关操作的方法。一个数据源DataSource需要一个事务管理器。

属性：DataSource

内部核心方法：
public commit 提交事务
public rollback 回滚事务
public getTransaction 获得当前事务状态

protected doSuspend 挂起事务

protected doBegin 开始事务,主要是执行了JDBC的con.setAutoCommit(false)方法。同时处理了很多和数据库连接相关的ThreadLocal变量。

protected doCommit 提交事务
protected doRollback 回滚事务
protected doGetTransaction() 获取事务信息
final getTransaction 获取事务状态
# 获取对应的TransactionManager

	@Nullable
	protected PlatformTransactionManager determineTransactionManager(@Nullable TransactionAttribute txAttr) {
		// Do not attempt to lookup tx manager if no tx attributes are set
		if (txAttr == null || this.beanFactory == null) {
			return getTransactionManager();
		}

		String qualifier = txAttr.getQualifier();
		//如果指定了Bean则取指定的PlatformTransactionManager类型的Bean
		if (StringUtils.hasText(qualifier)) {
			return determineQualifiedTransactionManager(this.beanFactory, qualifier);
		}
		//如果指定了Bean的名称,则根据bean名称获取对应的bean
		else if (StringUtils.hasText(this.transactionManagerBeanName)) {
			return determineQualifiedTransactionManager(this.beanFactory, this.transactionManagerBeanName);
		}
		else {
		// 默认取一个PlatformTransactionManager类型的Bean
			PlatformTransactionManager defaultTransactionManager = getTransactionManager();
			if (defaultTransactionManager == null) {
				defaultTransactionManager = this.transactionManagerCache.get(DEFAULT_TRANSACTION_MANAGER_KEY);
				if (defaultTransactionManager == null) {
					defaultTransactionManager = this.beanFactory.getBean(PlatformTransactionManager.class);
					this.transactionManagerCache.putIfAbsent(
							DEFAULT_TRANSACTION_MANAGER_KEY, defaultTransactionManager);
				}
			}
			return defaultTransactionManager;
		}
	}
可以看到，如果没有指定TransactionManager参数的话，会使用默认的一个实现，所以当程序中有多个数据库时，事务执行最好是指定事务管理器。

## 事务的信息TransactionInfo
TransactionInfo是对当前事务的描述，其中记录了事务的状态等信息。它记录了和一个事务所有的相关信息。它没有什么方法，只是对事务相关对象的一个组合。最关键的对象是TransactionStatus，它代表当前正在运行的是哪个事务。

1.核心属性：事务状态TransactionStatus
2.事务管理器
3.事务属性
4.上一个事务信息oldTransactionInfo，REQUIRE_NEW传播级别时，事务挂起后前一个事务的事务信息
![Nt0vAe.png](https://s1.ax1x.com/2020/06/23/Nt0vAe.png)

## 当前事务状态TransactionStatus

通过TransactionManager的getTransaction方法，获取当前事务的状态。
具体是在AbstractPlatformTransactionManager中实现.
TransactionStatus被用来做什么:TransactionManager对事务进行提交或回滚时需要用到该对象
具体方法有：
![NtBCct.png](https://s1.ax1x.com/2020/06/23/NtBCct.png)

作用：

1. 判断当前事务是否是一个新的事务，否则加入到一个已经存在的事务中。事务传播级别REQUIRED和REQUIRE_NEW有用到。
2. 当前事务是否携带保存点，嵌套事务用到。
setRollbackOnly,isRollbackOnly，当子事务回滚时，并不真正回滚事务，而是对子事务设置一个标志位。
3. 事务是否已经完成，已经提交或者已经回滚。

# 传播级别
## 介绍

Spring事务的传播级别描述的是多个使用了@Transactional注解的方法互相调用时，Spring对事务的处理。包涵的传播级别有：

1. REQUIRED, 如果当前线程已经在一个事务中，则加入该事务，否则新建一个事务。
1. SUPPORT, 如果当前线程已经在一个事务中，则加入该事务，否则不使用事务。
1. MANDATORY(强制的)，如果当前线程已经在一个事务中，则加入该事务，否则抛出异常。
1. REQUIRES_NEW，无论如何都会创建一个新的事务，如果当前线程已经在一个事务中，则挂起当前事务，创建一个新的事务。
1. NOT_SUPPORTED，如果当前线程在一个事务中，则挂起事务。
1. NEVER，如果当前线程在一个事务中则抛出异常。
1. NESTED, 执行一个嵌套事务，有点像REQUIRED，但是有些区别，在Mysql中是采用SAVEPOINT来实现的。

挂起事务，指的是将当前事务的属性如事务名称，隔离级别等属性保存在一个变量中，同时将当前线程中所有和事务相关的ThreadLocal变量设置为从未开启过线程一样。Spring维护着一个当前线程的事务状态，用来判断当前线程是否在一个事务中以及在一个什么样的事务中，挂起事务后，当前线程的事务状态就好像没有事务。
## 现象描述

	@Service
	public class MyTsA {
	    @Resource
	    private UserServiceImpl userService;
	    @Resource
	    private MyTsB myTsB;
	
	    @Transactional(rollbackFor = Exception.class)
	    public void testNewRollback(){
	        User user = new User();
	        user.setId(1);
	        user.setName("张三");
	        userService.save(user);
	        myTsB.save();
	    }
	}
	    
	@Service
	public class MyTsB {
	    @Resource
	    private UserServiceImpl userService;
	    @Transactional(rollbackFor = RuntimeException.class, propagation = ???)
	    public void save(){
	        User user = new User();
	        user.setId(2);
	        user.setName(“李四”)
	        userService.save(user);
	}
	
如上面所示有两个事务A和事务B，事务A方法中调用了事务B方法，区分两种回滚情况。
A提交，B回滚。
A回滚，B提交。

对于不同的传播级别：
1.当传播级别为为REQUIRED时，第一种情况A和B都会回滚。
2.。当传播级别为RQUIRED_NEW时，第一种情况A可以成功提交，B回滚，第二种情况A回滚，B可以正确提交，二者互不影响。（在A事务捕获B事务异常的情况下，保证A事务提交）
3.当传播级别为NESTED时，第一种情况A可以正常提交，B回滚，第二种情况二者都会回滚。
# 原理
我们从两个角度看他们的实现原理，一个是刚进入事务时，针对不同的传播级别，它们的行为有什么区别。另一个角度是当事务提交或者回滚时，传播级别对事务行为的影响。

首先在尝试获取事务信息时，如果当前已经存在一个事务，则会根据传播级别做一些处理:
## 隔离级别对开始事务的影响(获取TransactionStatus)
	@Override
	public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException {
	  // 从当前的transactionManager获取DataSource对象
	  // 然后以该DataSource对象为Key，
	  // 去一个ThreadLocal变量中的map中获取该DataSource的连接
	  // 然后设置到DataSourceTransactionObject中返回。
		Object transaction = doGetTransaction();
		// Cache debug flag to avoid repeated checks.
		boolean debugEnabled = logger.isDebugEnabled();
		if (definition == null) {
			// Use defaults if no transaction definition given.
			definition = new DefaultTransactionDefinition();
		}
	// 如果当前线程已经在一个事务中了，则需要根据事务的传播级别
	//来决定如何处理并获取事务状态对象
		if (isExistingTransaction(transaction)) {
			// Existing transaction found -> check propagation behavior to find out how to behave.
			return handleExistingTransaction(definition, transaction, debugEnabled);
		}
		// Check definition settings for new transaction.
		if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
			throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
		}
		// No existing transaction found -> check propagation behavior to find out how to proceed.
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
			throw new IllegalTransactionStateException(
					"No existing transaction found for transaction marked with propagation 'mandatory'");
		}
		else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
			SuspendedResourcesHolder suspendedResources = suspend(null);
			if (debugEnabled) {
				logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
			}
			try {
			  //如果当前不在一个事务中，则执行事务的准备操作
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
				// 构造事务状态对象,注意这里第三个参数为true,代表是一个新事务
				DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
				//执行begin操作,核心操作是设置隔离级别，执行   conn.setAutoCommit(false); 同时将数据连接绑定到当前线程
				doBegin(transaction, definition);
				// 针对事务相关属性如隔离级别，是否在事务中，设置绑定到当前线程
				prepareSynchronization(status, definition);
				return status;
			}
			catch (RuntimeException | Error ex) {
				resume(null, suspendedResources);
				throw ex;
			}
		}
		else {
			// Create "empty" transaction: no actual transaction, but potentially synchronization.
			if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
				logger.warn("Custom isolation level specified but no actual transaction initiated; " +
						"isolation level will effectively be ignored: " + definition);
			}
			boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
			return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
		}
	}

可以看到，如果当前不存在事务时，创建一个新的TransactionStatus对象，否则进入到handleExistingTransaction。下面来看这个方法

1. 如果是NEVER传播级别则抛出异常
2. 如果是不支持，则挂起事务，将当前事务对象设置为null,newTransaction设置为false，把线程的相关Threadlocal变量改的就像当前不存在事务一样
3. 如果是required_NEW的话，则挂起当前事务，同时创建一个新的事务，执行doBegin操作，设置newTransaction为true
4. 如果是嵌入事务，则创建一个SAVEPOINT
5. 对于REQUIRED传播级别会把newTransaction标志位设置为false

	/**
	 * Create a TransactionStatus for an existing transaction.
	 */
	private TransactionStatus handleExistingTransaction(
			TransactionDefinition definition, Object transaction, boolean debugEnabled)
			throws TransactionException {
	    //如果是NEVER传播级别则抛出异常
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
			throw new IllegalTransactionStateException(
					"Existing transaction found for transaction marked with propagation 'never'");
		}
	    // 如果是不支持，则挂起事务
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
			if (debugEnabled) {
				logger.debug("Suspending current transaction");
			}
			Object suspendedResources = suspend(transaction);
			boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
			//挂起事务同时将当前事务设置为null,newTransaction设置为false，把线程的相关Threadlocal变量改的就像当前不存在事务一样
			return prepareTransactionStatus(
					definition, null, false, newSynchronization, debugEnabled, suspendedResources);
		}
	    //如果是required_NEW的话，则挂起当前事务，同时创建一个新的事务，执行doBegin操作
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
			if (debugEnabled) {
				logger.debug("Suspending current transaction, creating new transaction with name [" +
						definition.getName() + "]");
			}
			SuspendedResourcesHolder suspendedResources = suspend(transaction);
			try {
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
				DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
				doBegin(transaction, definition);
				prepareSynchronization(status, definition);
				return status;
			}
			catch (RuntimeException | Error beginEx) {
				resumeAfterBeginException(transaction, suspendedResources, beginEx);
				throw beginEx;
			}
		}
	    // 如果是嵌入事务，则创建一个SAVEPOINT
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
			if (!isNestedTransactionAllowed()) {
				throw new NestedTransactionNotSupportedException(
						"Transaction manager does not allow nested transactions by default - " +
						"specify 'nestedTransactionAllowed' property with value 'true'");
			}
			if (debugEnabled) {
				logger.debug("Creating nested transaction with name [" + definition.getName() + "]");
			}
			if (useSavepointForNestedTransaction()) {
				// Create savepoint within existing Spring-managed transaction,
				// through the SavepointManager API implemented by TransactionStatus.
				// Usually uses JDBC 3.0 savepoints. Never activates Spring synchronization.
				DefaultTransactionStatus status =
						prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
				status.createAndHoldSavepoint();
				return status;
			}
			else {
				// Nested transaction through nested begin and commit/rollback calls.
				// Usually only for JTA: Spring synchronization might get activated here
				// in case of a pre-existing JTA transaction.
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
				DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, null);
				doBegin(transaction, definition);
				prepareSynchronization(status, definition);
				return status;
			}
		}
	
		// Assumably PROPAGATION_SUPPORTS or PROPAGATION_REQUIRED.
		if (debugEnabled) {
			logger.debug("Participating in existing transaction");
		}
	//这里判断是否需要对已经存在的事务进行校验，这个可以通过AbstractPlatformTransactionManager.setValidateExistingTransaction(boolean)来设置，设置为true后需要校验当前事务的隔离级别和已经存在的事务的隔离级别是否一致 
		if (isValidateExistingTransaction()) {
			if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
				Integer currentIsolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
				if (currentIsolationLevel == null || currentIsolationLevel != definition.getIsolationLevel()) {
					Constants isoConstants = DefaultTransactionDefinition.constants;
					throw new IllegalTransactionStateException("Participating transaction with definition [" +
							definition + "] specifies isolation level which is incompatible with existing transaction: " +
							(currentIsolationLevel != null ?
									isoConstants.toCode(currentIsolationLevel, DefaultTransactionDefinition.PREFIX_ISOLATION) :
									"(unknown)"));
				}
			}
			if (!definition.isReadOnly()) {
				if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
					throw new IllegalTransactionStateException("Participating transaction with definition [" +
							definition + "] is not marked as read-only but existing transaction is");
				}
			}
		}
		// 如果不设置是否校验已经存在的事务，则对于REQUIRED传播级别会走到这里来，这里把newTransaction标志位设置为false,
		// 这里用的definition是当前事务的相关属性，所以隔离级别等依然是当前事务的（子事务），而不是已经存在的事务的隔离级别（父事务）
		boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
		return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
	}

## 隔离级别对回滚事务的影响
1. 如果newTransaction设置为ture，则真正执行回滚。
1. 如果有保存点，则回滚到保存点。
1. 否则不会真正回滚，而是设置回滚标志位。
	private void processRollback(DefaultTransactionStatus status, boolean unexpected) {
			try {
				boolean unexpectedRollback = unexpected;
				try {
					triggerBeforeCompletion(status);
					// 如果有保存点
					if (status.hasSavepoint()) {
						if (status.isDebug()) {
							logger.debug("Rolling back transaction to savepoint");
						}
						status.rollbackToHeldSavepoint();
					}
					// 如果是新的事务，当传播级别为RUQUIRED_NEW时会走到这里来
					else if (status.isNewTransaction()) {
						if (status.isDebug()) {
							logger.debug("Initiating transaction rollback");
						}
						doRollback(status);
					}
					else {
					    // 加入到事务中，设置回滚状态，适用于REQUIRED传播级别
					    // 并不会真的回滚，而是设置回滚标志位
						// Participating in larger transaction
						if (status.hasTransaction()) {
							if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
								if (status.isDebug()) {
									logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
								}
								doSetRollbackOnly(status);
							}
							else {
								if (status.isDebug()) {
									logger.debug("Participating transaction failed - letting transaction originator decide on rollback");
								}
							}
						}
						else {
							logger.debug("Should roll back transaction but cannot - no transaction available");
						}
						// Unexpected rollback only matters here if we're asked to fail early
						if (!isFailEarlyOnGlobalRollbackOnly()) {
							unexpectedRollback = false;
						}
					}
				}
				catch (RuntimeException | Error ex) {
					triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
					throw ex;
				}
	
				triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
	
				// Raise UnexpectedRollbackException if we had a global rollback-only marker
				if (unexpectedRollback) {
					throw new UnexpectedRollbackException(
							"Transaction rolled back because it has been marked as rollback-only");
				}
			}
			finally {
				cleanupAfterCompletion(status);
			}
		}
## 隔离级别对提交事务的影响
1. 就算事务方法没有抛出异常，走到了commit方法中，但是依然有可能回滚事务。
2. 对于REQUIRED传播级别，即使父事务中没有抛出异常，但是子事务中已经设置了回滚标志，那么父事务依然会回滚
3. 只有newTransaction标志位为true的事务才会真正执行commit操作。

	@Override
	public final void commit(TransactionStatus status) throws TransactionException {
		if (status.isCompleted()) {
			throw new IllegalTransactionStateException(
					"Transaction is already completed - do not call commit or rollback more than once per transaction");
		}
	
		DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
		if (defStatus.isLocalRollbackOnly()) {
			if (defStatus.isDebug()) {
				logger.debug("Transactional code has requested rollback");
			}
			processRollback(defStatus, false);
			return;
		}
	 // 对于REQUIRED传播级别，即使父事务中没有抛出异常，但是子事务中已经设置了回滚标志，那么父事务依然会回滚
		if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
			if (defStatus.isDebug()) {
				logger.debug("Global transaction is marked as rollback-only but transactional code requested commit");
			}
			processRollback(defStatus, true);
			return;
		}
	
		processCommit(defStatus);
	}

![NtB5b8.png](https://s1.ax1x.com/2020/06/23/NtB5b8.png)

# 为什么我的事务不生效

1. 如果不是Innodb存储引擎，MyISAM不支持事务。
2. 没有指定rollbackFor参数。
3. 没有指定transactionManager参数，默认的transactionManager并不是我期望的，以及一个事务中涉及到了多个数据库。
4. 如果AOP使用了JDK动态代理，对象内部方法互相调用不会被Spring的AOP拦截，@Transactional注解无效。
5. 如果AOP使用了CGLIB代理，事务方法或者类不是public，无法被外部包访问到，或者是final无法继承，@transactional注解无效。

