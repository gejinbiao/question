# JVM调优--常用JVM监控工具使用
在实际工作中，在进行jvm调优或者分析内存泄露、溢出等问题时，熟练掌握JVM常用的监控工具能够帮助更快地定位问题所在，目前记录一下使用过的常用的jvm监控工具以及其使用、和对应分析过程。

# 查看jvm使用的垃圾回收器

在开始了jvm调优或者内存问题分析时，我们首先要了解的就是JVM使用垃圾回收器是什么，才能结合监控工具，做出正确的结果分析。那么如何查看JVM运行时使用的垃圾回收器是什么？

- 如果在程序的jvm启动参数中没有指定使用什么垃圾回收器，我们可以直接在命令行执行以下命令

	java -XX:+PrintCommandLineFlags -version

得到以下输出，此命令默认打印出jvm运行使用的参数以及jvm版本信息，默认情况下不指定垃圾回收器类型的话，从虚拟机参数是没办法知道垃圾回收器类型的。

但是我们可以从虚拟机版本得知默认垃圾回收器，64位server的虚拟机默认垃圾回收器是ParallerGC吞吐量优先

	-XX:InitialHeapSize=16264576 -XX:MaxHeapSize=260233216 -XX:+PrintCommandLineFlags -XX:+UseCompressedClassPointers -XX:+UseCompressedOops 
	java version "1.8.0_171"
	Java(TM) SE Runtime Environment (build 1.8.0_171-b11)
	Java HotSpot(TM) 64-Bit Server VM (build 25.171-b11, mixed mode)

如果程序启动的jvm参数有指定使用的垃圾回收器类型的话，更加简单，直接查看jvm启动参数是什么垃圾回收器即可，常用的启动参数与垃圾回收器对应关系如下。

![Uk44Gn.png](https://s1.ax1x.com/2020/07/07/Uk44Gn.png)

# jps
执行jps命令可以获取到当前虚拟机中正在运行的java程序，以及对应的进程号，输出格式如下
2865 jvm-demo1-0.0.1-SNAPSHOT.jar
3627 Jps

前面的数字就是进程号，后续分析肯定用到。

#jmap

jmap使用主要是打印某一时刻，某个进行的jvm堆信息，使用步骤如下

- 获取进行pid ps -ef|grep 对应进程
或者使用jps获取

- 执行以下命令，例如我要获取进行jvm-demo1-0.0.1-SNAPSHOT.jar的堆使用信息，可以执行

	jmap -heap 2865

获得输出如下

	Attaching to process ID 2865, please wait...
	Debugger attached successfully.
	Server compiler detected.
	JVM version is 25.171-b11
	
	using thread-local object allocation.
	Mark Sweep Compact GC
	#堆相关的配置信息
	Heap Configuration:
	   MinHeapFreeRatio         = 40
	   MaxHeapFreeRatio         = 70
	   MaxHeapSize              = 268435456 (256.0MB)
	   NewSize                  = 44695552 (42.625MB)
	   MaxNewSize               = 89456640 (85.3125MB)
	   OldSize                  = 89522176 (85.375MB)
	   NewRatio                 = 2
	   SurvivorRatio            = 8
	   MetaspaceSize            = 134217728 (128.0MB)
	   CompressedClassSpaceSize = 528482304 (504.0MB)
	   MaxMetaspaceSize         = 536870912 (512.0MB)
	   G1HeapRegionSize         = 0 (0.0MB)
	#堆相关的占用信息
	Heap Usage:
	New Generation (Eden + 1 Survivor Space):
	   capacity = 40239104 (38.375MB)
	   used     = 5541792 (5.285064697265625MB)
	   free     = 34697312 (33.089935302734375MB)
	   13.772155562907166% used
	Eden Space:
	   capacity = 35782656 (34.125MB)
	   used     = 2550440 (2.4322891235351562MB)
	   free     = 33232216 (31.692710876464844MB)
	   7.127587175194597% used
	From Space:
	   capacity = 4456448 (4.25MB)
	   used     = 2991352 (2.8527755737304688MB)
	   free     = 1465096 (1.3972244262695312MB)
	   67.12413114659927% used
	To Space:
	   capacity = 4456448 (4.25MB)
	   used     = 0 (0.0MB)
	   free     = 4456448 (4.25MB)
	   0.0% used
	tenured generation:
	   capacity = 89522176 (85.375MB)
	   used     = 16718752 (15.944244384765625MB)
	   free     = 72803424 (69.43075561523438MB)
	   18.675542471174964% used
	
	16179 interned Strings occupying 2172080 bytes.
	

# jstat

jstat用的最多的是用来统计gc相关信息，先看使用步骤如下。

- 获取进程号

- 例如我想统计jvm-demo1-0.0.1-SNAPSHOT.jar（进程号2865）的gc信息，并且每隔3秒统计一次，可以执行以下命令

 	jstat -gcutil 2865 3000

经过一阵时间监控，得到以下输出

	  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT   
	 67.12   0.00   7.13  18.68  97.37  96.73     20    0.181     0    0.000    0.181
	 67.12   0.00   7.13  18.68  97.37  96.73     20    0.181     0    0.000    0.181
	 67.12   0.00   7.13  18.68  97.37  96.73     20    0.181     0    0.000    0.181
	 67.12   0.00   7.13  18.68  97.37  96.73     20    0.181     0    0.000    0.181
	 67.12   0.00   7.13  18.68  97.37  96.73     20    0.181     0    0.000    0.181
	 67.12   0.00   7.13  18.68  97.37  96.73     20    0.181     0    0.000    0.181
	 67.12   0.00   7.13  18.68  97.37  96.73     20    0.181     0    0.000    0.181
	 67.12   0.00   7.13  18.68  97.37  96.73     20    0.181     0    0.000    0.181
	 67.12   0.00   7.13  18.68  97.37  96.73     20    0.181     0    0.000    0.181
	

主要关注以下几个。

- s0、s1:表示两个survior区域的使用百分比
- e：eden区域使用百分比
- o：老年代使用百分比
- m：metaspace（元数据空间）使用百分比
- ygc：新生代gc次数
- ygct：新生代gc累计总时间
- fgc：full gc次数
- fgct：full gc累计总时间
- gct：gc累计总时间
- 假如我想知道jvm-demo1-0.0.1-SNAPSHOT.jar上一次gc的原因，并且持续监控,3秒输出一次，可以执行以下命令

- jstat -gccause 2865  3000

监控一阵子后，得到输出

	 S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT    LGCC                 GCC                 
	  0.00  93.94  25.84  28.97  97.27  96.95     21    0.206     0    0.000    0.206 Allocation Failure   No GC               
	  0.00  93.94  31.70  28.97  97.27  96.95     21    0.206     0    0.000    0.206 Allocation Failure   No GC               
	  0.00  93.94  37.56  28.97  97.27  96.95     21    0.206     0    0.000    0.206 Allocation Failure   No GC  


大部分根gcutil差不多

- LGCC：上一次gc的原因，Allocation Failure就是新生代满了，进行gc
- gcc：当前gc的原因，如果当前没有gc就no gc

# jconsole

可视化监控jvm，基本上看到啥就是啥，但是生产比较少会用到

# gc log

gc log顾名思义就是在gc时打印出来的日志，关于这一块的分析，对于问题定位也是非常有用的。这里gclog的垃圾回收器是ParallerGC

- 在jvm启动参数加上以下，可以开启gc log,配置gclog的输出位置

		-XX:+PrintGCDateStamps -XX:+PrintGCDetails -Xloggc:/usr/local/project/jvmtest/gc.log

- gc log分析，借用网上的一张图片

![Uk5UyV.png](https://s1.ax1x.com/2020/07/07/Uk5UyV.png)

​ 所有的gc日志输出，基本上大同小异，下面看一段gc日志结合分析。

	2019-01-23T22:18:10.534+0800: 442.398: [GC (Allocation Failure) 2019-01-23T22:18:10.534+0800: 442.398: [DefNew: 77
	654K->77654K(78656K), 0.0000730 secs]2019-01-23T22:18:10.534+0800: 442.398: [Tenured: 174240K->174240K(174784K), 0
	.1493292 secs] 251895K->250131K(253440K), [Metaspace: 30461K->30461K(1077248K)], 0.1495998 secs] [Times: user=0.07
	 sys=0.00, real=0.15 secs]

上面是一段gc日志。

GC (Allocation Failure)这种明显就是新生代满，导致无法分配的gc类型，minor gc。

[DefNew: 36848K->2878K(39296K), 0.0039304 secs]:虽然跟图中有所出入，但是可以分析得知是表示新生代，占用在垃圾回收前后的大小变化，括号表示新生代总大小，gc日志中，新生代总大小表示**（eden+1个survior）**。可以通过-Xmn大小m -XX:SurvivorRatio=比率来控制新生代的eden、survior大小。

[Tenured: 174240K->174240K(174784K), 0.1493292 secs]:表示老年代gc前后占用大小变化，括号表示老年代总大小。

下面再来看一段full gc日志。

	2019-01-23T22:18:22.793+0800: 454.657: [Full GC (Allocation Failure) 2019-01-23T22:18:22.793+0800: 454.657: [Tenur
	ed: 174783K->174783K(174784K), 0.0351375 secs] 253439K->252909K(253440K), [Metaspace: 30949K->30949K(1077248K)], 0
	.0352028 secs] [Times: user=0.04 sys=0.00, real=0.03 secs] 
	
跟minor gc基本大同小异。

# jstack

jstack主要的用途是打印出Thread dump，使用步骤如下。

- 执行以下命令,输出dump 到stack.log中。

	jstack pid  > ./stack.log

查看log，截取下面一部分,可以发现，如果出现死锁，log中在末尾部分会提示出来。

	Found one Java-level deadlock:
	=============================
	"Thread-5":
	  waiting to lock monitor 0x00007f36882e5808 (object 0x00000000fff1f3c0, a java.lang.Object),
	  which is held by "Thread-4"
	"Thread-4":
	  waiting to lock monitor 0x00007f36882e4fc8 (object 0x00000000fff1f3d0, a java.lang.Object),
	  which is held by "Thread-5"
	
	Java stack information for the threads listed above:
	===================================================

根据两个线程id，在日志内查找，得到以下信息,下面的信息记录了Thread-5 锁住了Thread-4等待的资源，反过来也是，从而导致死锁。这就是一个简单的Thread dump分析。

补充一些概念：

- “Thread-5” 这个位置表示线程id
- daemon ：表示线程类型
- prio：表示优先级
- nid：表示线程pid的十六进制

	"Thread-5" #32 daemon prio=5 os_prio=0 tid=0x00007f3678f01800 nid=0x156f waiting for monitor entry [0x00007f36a8c99000]
	   java.lang.Thread.State: BLOCKED (on object monitor)
		at com.garine.controllers.JvmTestController$DeadLock.rightLeft(JvmTestController.java:144)
		- waiting to lock <0x00000000fff1f3c0> (a java.lang.Object)
		- locked <0x00000000fff1f3d0> (a java.lang.Object)
		at com.garine.controllers.JvmTestController$Thread1.run(JvmTestController.java:102)
	
	"Thread-4" #31 daemon prio=5 os_prio=0 tid=0x00007f367851b800 nid=0x156e waiting for monitor entry [0x00007f36a8598000]
	   java.lang.Thread.State: BLOCKED (on object monitor)
		at com.garine.controllers.JvmTestController$DeadLock.leftRight(JvmTestController.java:134)
		- waiting to lock <0x00000000fff1f3d0> (a java.lang.Object)
		- locked <0x00000000fff1f3c0> (a java.lang.Object)
		at com.garine.controllers.JvmTestController$Thread0.run(JvmTestController.java:118)


如果我们分析Thread dump目的是寻找死锁，等待的线程，可以直接搜索BLOCKED，deadlock等字眼

mat,全程内存分析工具，专门分析内存溢出时，转存的快照，从而定位问题。

- 想要分析内存快照，需要配置内存dump，在jvm启动参数中加入以下参数,配置好dump存储路径方便查找。


    1. -XX:+HeapDumpOnOutOfMemoryError 
    2. -XX:HeapDumpPath=/usr/local/dump/error.hprof


必要的[mat工具下载地址](http://help.eclipse.org/oxygen/index.jsp?topic=/org.eclipse.mat.ui.help/welcome.html)

在jvm-demo1-0.0.1-SNAPSHOT.jar中，提供一个接口，调用之后疯狂新建对象，最终out of memory，demo如下.在我们配置好的路径下面，会保存一个error.hprof文件，我们使用mat打开该文件。

	@RestController
	public class JvmTestController {
	
	
	    private static List<int[]> bigObj = new ArrayList();
	    private static List<char[]> bigCharObj = new ArrayList();
	
	    public static int[] generate1M() {
	        return new int[524288];
	    }
	
	    public static char[] generate1MChar() {
	        return new char[1048576];
	    }
	
	    @GetMapping("/genObject")
	    public void error() throws InterruptedException {
	        for (int i = 0; i < 1000; ++i) {
	            if (i == 0) {
	                Thread.sleep(500L);
	                System.out.println("start = [" + new Date() + "]");
	            } else {
	                Thread.sleep(4000L);
	            }
	            bigObj.add(generate1M());
	        }
	
	    }
	    
	}

# MAT使用
	1. 打开首页，进入以下界面，选择第一项，'怀疑内存泄露’的分析报告，点击finish
	
![UkoEvt.png](https://s1.ax1x.com/2020/07/07/UkoEvt.png)

	2. 打开error.hprof文件，进入以下界面，进来就可以看到，mat帮助我们分析出疑似是内存泄漏的对象，我们可以直接根据mat的分析作为线索，开展排查。


![UkoKUg.png](https://s1.ax1x.com/2020/07/07/UkoKUg.png)

点击进入detail ，然后翻到以下位置。可以定位疑似内存泄漏的对象与对象代码所在位置。在这例子中，是com.garine.controllers.JvmTestController疯狂创建对象，加入到ArrayList中，导致内存泄漏，对照下面的图片分析，就可以定位问题代码。

![UkolCj.png](https://s1.ax1x.com/2020/07/07/UkolCj.png)

	3. 或者我们可以自己去定位问题所在，不使用mat提供的线索，我们进入OverView-》Historgram视图，这个视图是列出每个类都多少个实例，如下图所示。

可以看出是int[]对象实例最多，可以点击表头进行排序。其中shalldown heap表示对象的实际大小，retained heap表示回收这个对象可以释放的大小，也就是仅仅以这个对象作为gc root，所能关联到的对象大小也算进去。

同时，还可以看到左侧面板，能看到我们选定的对象是否是gc root。

![UkoBG9.png](https://s1.ax1x.com/2020/07/07/UkoBG9.png)

 根据当前所见，我们暂且怀疑int[]是内存泄漏的对象，但是int[] 并不是gc root，为了更好定位具体内存泄漏的位置，我们需要定位int[]的gc root。操作方法如下所示。右键目标对象，寻早gc root。

![UkoD2R.png](https://s1.ax1x.com/2020/07/07/UkoD2R.png)

最终得到结果如下。如此操作，也能得到分析出疑似内存泄漏 的代码位置所在，这种方式可以让我们一个个怀疑点都去分析一遍，更加全面。

![UkoyKx.png](https://s1.ax1x.com/2020/07/07/UkoyKx.png)

