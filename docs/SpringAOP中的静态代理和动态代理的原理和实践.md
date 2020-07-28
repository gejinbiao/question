# Spring AOP中的静态代理和动态代理的原理和实践
为什么会有面向切面编程（AOP）？我们知道java是一个面向对象(OOP)的语言，但它有一些弊端,比如当我们需要为多个不具有继承关系的对象引入一个公共行为，例如日志,权限验证,事务等功能时，只能在在每个对象里引用公共行为，这样做不便于维护，而且有大量重复代码。AOP的出现弥补了OOP的这点不足。

为了阐述清楚spring AOP，我们从将以下方面进行讨论：

1. 代理模式。
1. 静态代理原理及实践。
1. 动态代理原理及实践。
1. spring AOP原理及实战。

# 1.代理模式。
代理模式：为其他对象提供一种代理以控制对这个对象的访问。这段话比较官方，但我更倾向于用自己的语言理解：比如A对象要做一件事情，在没有代理前，自己来做，在对A代理后，由A的代理类B来做。代理其实是在原实例前后加了一层处理，这也是AOP的初级轮廓。





# 2.静态代理原理及实践。
静态代理模式：静态代理说白了就是在程序运行前就已经存在代理类的字节码文件，代理类和原始类的关系在运行前就已经确定。废话不多说，我们看一下代码，为了方便阅读，博主把单独的class文件合并到接口中，读者可以直接复制代码运行：

	package test.staticProxy;
	// 接口
	public interface IUserDao {
		void save();
		void find();
	}
	//目标对象
	class UserDao implements IUserDao{
		@Override
		public void save() {
			System.out.println("模拟：保存用户！");
		}
		@Override
		public void find() {
			System.out.println("模拟：查询用户");
		}
	}
	/**
	    静态代理
	          特点：
		1. 目标对象必须要实现接口
		2. 代理对象，要实现与目标对象一样的接口
	 */
	class UserDaoProxy implements IUserDao{
		// 代理对象，需要维护一个目标对象
		private IUserDao target = new UserDao();
		@Override
		public void save() {
			System.out.println("代理操作： 开启事务...");
			target.save();   // 执行目标对象的方法
			System.out.println("代理操作：提交事务...");
		}
		@Override
		public void find() {
			target.find();
		}
	}

![Ntykxs.png](https://s1.ax1x.com/2020/06/23/Ntykxs.png)

静态代理虽然保证了业务类只需关注逻辑本身，代理对象的一个接口只服务于一种类型的对象，如果要代理的方法很多，势必要为每一种方法都进行代理。再者，如果增加一个方法，除了实现类需要实现这个方法外，所有的代理类也要实现此方法。增加了代码的维护成本。那么要如何解决呢?答案是使用动态代理。

# 3.动态代理原理及实践
动态代理模式:动态代理类的源码是在程序运行期间通过JVM反射等机制动态生成，代理类和委托类的关系是运行时才确定的。实例如下:

	package test.dynamicProxy;
	 
	import java.lang.reflect.InvocationHandler;
	import java.lang.reflect.Method;
	import java.lang.reflect.Proxy;
	// 接口
	public interface IUserDao {
		void save();
		void find();
	}
	//目标对象
	 class UserDao implements IUserDao{
		@Override
		public void save() {
			System.out.println("模拟： 保存用户！");
		}
		@Override
		public void find() {
			System.out.println("查询");
		}
	}
	/**
	 * 动态代理：
	 *    代理工厂，给多个目标对象生成代理对象！
	 *
	 */
	class ProxyFactory {
		// 接收一个目标对象
		private Object target;
		public ProxyFactory(Object target) {
			this.target = target;
		}
		// 返回对目标对象(target)代理后的对象(proxy)
		public Object getProxyInstance() {
			Object proxy = Proxy.newProxyInstance(
				target.getClass().getClassLoader(),  // 目标对象使用的类加载器
				target.getClass().getInterfaces(),   // 目标对象实现的所有接口
				new InvocationHandler() {			// 执行代理对象方法时候触发
					@Override
					public Object invoke(Object proxy, Method method, Object[] args)
							throws Throwable {
	 
						// 获取当前执行的方法的方法名
						String methodName = method.getName();
						// 方法返回值
						Object result = null;
						if ("find".equals(methodName)) {
							// 直接调用目标对象方法
							result = method.invoke(target, args);
						} else {
							System.out.println("开启事务...");
							// 执行目标对象方法
							result = method.invoke(target, args);
							System.out.println("提交事务...");
						}
						return result;
					}
				}
			);
			return proxy;
		}
	}

![NtylRJ.png](https://s1.ax1x.com/2020/06/23/NtylRJ.png)

在运行测试类中创建测试类对象代码中

	IUserDao proxy = (IUserDao)new ProxyFactory(target).getProxyInstance();
其实是jdk动态生成了一个类去实现接口,隐藏了这个过程:

	class $jdkProxy implements IUserDao{}
使用jdk生成的动态代理的前提是目标类必须有实现的接口。但这里又引入一个问题,如果某个类没有实现接口,就不能使用jdk动态代理,所以Cglib代理就是解决这个问题的。

Cglib是以动态生成的子类继承目标的方式实现，在运行期动态的在内存中构建一个子类，如下:

	public class UserDao{}
	//Cglib是以动态生成的子类继承目标的方式实现,程序执行时,隐藏了下面的过程
	public class $Cglib_Proxy_class  extends UserDao{}

**Cglib使用的前提是目标类不能为final修饰。因为final修饰的类不能被继承。**

现在，我们可以看看AOP的定义：面向切面编程，核心原理是使用动态代理模式在方法执行前后或出现异常时加入相关逻辑。

通过定义和前面代码我们可以发现3点：

1. AOP是基于动态代理模式。
1. AOP是方法级别的（要测试的方法不能为static修饰，因为接口中不能存在静态方法，编译就会报错）。
1. AOP可以分离业务代码和关注点代码（重复代码），在执行业务代码时，动态的注入关注点代码。切面就是关注点代码形成的类。

# 4.spring AOP原理及实战

前文提到jdk代理和cglib代理两种动态代理，优秀的spring框架把两种方式在底层都集成了进去,我们无需担心自己去实现动态生成代理。那么，spring是如何生成代理对象的？

1. 创建容器对象的时候，根据切入点表达式拦截的类，生成代理对象。
2. 如果目标对象有实现接口，使用jdk代理。如果目标对象没有实现接口，则使用cglib代理。然后从容器获取代理后的对象，在运行期植入"切面"类的方法。通过查看spring源码，我们在DefaultAopProxyFactory类中，找到这样一段话。

![NtyceP.png](https://s1.ax1x.com/2020/06/23/NtyceP.png)

简单的从字面意思看出,如果有接口,则使用Jdk代理,反之使用Cglib，这刚好印证了前文所阐述的内容。spring AOP综合两种代理方式的使用前提有会如下结论：<span style="color:red">如果目标类没有实现接口，且class为final修饰的，则不能进行spring AOP编程！</span>

知道了原理，现在我们将自己手动实现spring的AOP：

	package test.spring_aop_anno;
	 
	import org.aspectj.lang.ProceedingJoinPoint;
	 
	public interface IUserDao {
		void save();
	}
	//用于测试Cglib动态代理
	class OrderDao {
		public void save() {
			//int i =1/0;用于测试异常通知
			System.out.println("保存订单...");
		}
	}
	//用于测试jdk动态代理
	class UserDao implements IUserDao {
		public void save() {
			//int i =1/0;用于测试异常通知
			System.out.println("保存用户...");
		}
	}
	//切面类
	class TransactionAop {
		public void beginTransaction() {
			System.out.println("[前置通知]  开启事务..");
		}
		public void commit() {
			System.out.println("[后置通知] 提交事务..");
		}
		public void afterReturing(){
			System.out.println("[返回后通知]");
		}
		public void afterThrowing(){
			System.out.println("[异常通知]");
		}
		public void arroud(ProceedingJoinPoint pjp) throws Throwable{
			System.out.println("[环绕前：]");
			pjp.proceed();    			   // 执行目标方法
			System.out.println("[环绕后：]");
		}
	}

spring的xml配置文件:

	<?xml version="1.0" encoding="UTF-8"?>
	<beans xmlns="http://www.springframework.org/schema/beans"
	    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	    xmlns:context="http://www.springframework.org/schema/context"
	    xmlns:aop="http://www.springframework.org/schema/aop"
	    xsi:schemaLocation="
	        http://www.springframework.org/schema/beans
	        http://www.springframework.org/schema/beans/spring-beans.xsd
	        http://www.springframework.org/schema/context
	        http://www.springframework.org/schema/context/spring-context.xsd
	        http://www.springframework.org/schema/aop
	        http://www.springframework.org/schema/aop/spring-aop.xsd">
		<!-- dao实例加入容器 -->
		<bean id="userDao" class="test.spring_aop_anno.UserDao"></bean>
	 
		<!-- dao实例加入容器 -->
		<bean id="orderDao" class="test.spring_aop_anno.OrderDao"></bean>
	 
		<!-- 实例化切面类 -->
		<bean id="transactionAop" class="test.spring_aop_anno.TransactionAop"></bean>
	 
		<!-- Aop相关配置 -->
		<aop:config>
			<!-- 切入点表达式定义 -->
			<aop:pointcut expression="execution(* test.spring_aop_anno.*Dao.*(..))" id="transactionPointcut"/>
			<!-- 切面配置 -->
			<aop:aspect ref="transactionAop">
				<!-- 【环绕通知】 -->
				<aop:around method="arroud" pointcut-ref="transactionPointcut"/>
				<!-- 【前置通知】 在目标方法之前执行 -->
				<aop:before method="beginTransaction" pointcut-ref="transactionPointcut" />
				<!-- 【后置通知】 -->
				<aop:after method="commit" pointcut-ref="transactionPointcut"/>
				<!-- 【返回后通知】 -->
				<aop:after-returning method="afterReturing" pointcut-ref="transactionPointcut"/>
				<!-- 异常通知 -->
				<aop:after-throwing method="afterThrowing" pointcut-ref="transactionPointcut"/>
			</aop:aspect>
		</aop:config>
	</beans>      
切入点表达式不在这里介绍。ref:Spring AOP 切入点表达式

代码的测试结果如下:

![Nt6JpQ.png](https://s1.ax1x.com/2020/06/23/Nt6JpQ.png)

到这里,我们已经全部介绍完spring AOP，回到开篇的问题，我们拿它做什么？

1. Spring声明式事务管理配置，博主的另一篇文章：分布式系统架构实战demo:ssm+dubbo
1. Controller层的参数校验。ref：spring aop拦截Controller做参数校验
1. 使用Spring AOP实现MySQL数据库读写分离案例分析
1. 在执行方法前,判断是否具有权限。
1. 对部分函数的调用进行日志记录。监控部分重要函数，若抛出指定的异常，可以以短信或邮件方式通知相关人员。
1. 信息过滤，页面转发等等功能,博主一个人的力量有限,只能列举这么多,欢迎评论区对文章做补充。
spring AOP还能做什么，实现什么魔幻功能，就在于我们每一个平凡而又睿智的程序猿！