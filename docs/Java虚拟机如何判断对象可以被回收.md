# Java虚拟机如何判断对象可以被回收
垃圾收集器如何判断一个对象已经“死去”，能够回收这块内存呢？通常有引用计数法和可达性算法。

# （1）引用计数法
        简单的说就是给对象添加一个计数器，每当有一个地方引用它时，计数器就加1；当引用失效，计数器就减1；任何时刻计数器为0的对象，就是不可能再使用的。

优点：效率高，实现简单

缺点：无法解决对象之间循环引用的问题

# （2）可达性算法
        算法的基本思想是通过一系列的成为“GC Roots”的对象作为起点，从这些起点开始向下搜索，搜索的路径就成为引用链（Reference Chain），当一个对象到GC Roots没有任何的引用链相连的话，也就是该对象不可达，则证明该对象是不可用的

![UA9YsU.png](https://s1.ax1x.com/2020/07/07/UA9YsU.png)

在Java中GC Roots对象包括以下几个：

- 虚拟机栈（栈帧的本地变量表）中引用的对象
- 方法区中类静态属性引用的对象
- 方法区中常量引用的对象
- 本地方法栈JNI引用的对象

  在可达性的算法中，至少要被标记两次，才会真正的宣告一个对象死亡：如果对象那在进行可达性分析发现没有GC Roots相连的引用链，那么将会被第一次标记并且进行一次筛选，筛选的条件是此对象那个是否有必要执行finalize()方法。当对象没有覆盖finalize()方法，或者finalize()方法已经被虚拟机调用过，虚拟机将这两种情况都视为“没有必要执行”。

   如果这个对象被判定位有必要执行finalize()方法，那么这个对象将会被放置在一个叫做F-Queue的队列中，并在稍后有一个虚拟机自动建立的、低优先级的Finalizer线程去执行它。这里所谓的“执行”是指虚拟机会触发这个方法，并不会承诺会等待它允许结束，这样的原因是，如果一个对象的finalize()方法执行缓慢，或者发生死循环，可能会到导致F-Queue队列中的其他对象永远处于等待，甚至导致整个内存回收系统崩溃。finalize()方法是对象逃脱死亡命运的最后一次机会，如果对象要在finalize()方法中拯救自己--只需要重新与引用链上的任何一个对象建立关联即可（这种自救的机会只能有一次），譬如把自己的this关键字赋值给某各类的变量或者类的成员变量即可，那在第二次标记时，就会被移除回收集合。

   如果对象在这个时候还没有逃脱，基本上就要被回收了。

下面给出示例，展示对象在被回收前自救：

	/**
	 * Created by wzj on 2018/3/21.
	 */
	public class TestEscapeGC
	{
	    public static TestEscapeGC SAVE = null;
	 
	    public void isAlive()
	    {
	        System.out.println(" Yes I am alive");
	    }
	 
	 
	    @Override
	    protected void finalize() throws Throwable
	    {
	        super.finalize();
	 
	        System.out.println("Finalize method executed");
	 
	        //自救
	        TestEscapeGC.SAVE = this;
	    }
	 
	    public static void main(String[] args) throws InterruptedException
	    {
	        SAVE = new TestEscapeGC();
	 
	        //第一次自救
	        SAVE = null;
	        System.gc();
	        Thread.sleep(500);
	 
	        if (SAVE != null)
	        {
	            SAVE.isAlive();
	        }
	        else
	        {
	            System.out.println("Oh I am dead");
	        }
	 
	        //第一次自救
	        SAVE = null;
	        System.gc();
	        Thread.sleep(500);
	 
	        if (SAVE != null)
	        {
	            SAVE.isAlive();
	        }
	        else
	        {
	            System.out.println("Oh I am dead");
	        }
	    }
	}
程序打印结果为：

![UA90iR.png](https://s1.ax1x.com/2020/07/07/UA90iR.png)

从结果可以看出，SAVE对象的finalize()方法确实被GC收集器触发过，并且在收集前逃脱了。

任何一个对象的finalize()方法都只会被系统自动调用过一次，如果对象面临下一次的回收，该方法不会被再次执行，因此第二段代码自救失败了。