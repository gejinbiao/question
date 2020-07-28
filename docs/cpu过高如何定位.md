-----


# Java问题定位：CPU占用过高分析

一般在开发Java的时候，为防止占用过多的资源，对CPU和内存的占用，都会有一个要求，例如CPU不能超过70%，内存不能超过4G等，那在一般问题定位的过程中，如何和定位这些问题呢，下面简单介绍一下CPU占用过高问题的定位方法。

# Linux环境
在Linux上面，可以借助丰富的命令行工具来进行定位

- 查找占用过高的进程
执行top命令查看CPU占用高的进程

![image21079659c6d35b4d.png](https://wx2.sbimg.cn/2020/06/19/image21079659c6d35b4d.png)

例如我们认为上面红框的部分为显示的CPU占用过多的进程，可以看到，其进程号为93937，下面会多次使用这个进程号定文。


- 查找对应的线程
执行ps -mp 93937-o THREAD,tid,time如下

![image95a4769745296e7c.png](https://wx1.sbimg.cn/2020/06/19/image95a4769745296e7c.png)

最下面的三个线程，显示占用较高，几乎占满的单个CPU的时间



- 查找对应的方法栈
这边利用了Java自带的jstack命令，执行一下jstack 93937，如下

![image9907d3c48b7262ce.png](https://wx2.sbimg.cn/2020/06/19/image9907d3c48b7262ce.png)

如上，我们看到这几个线程，当然真实场景会更为复杂一些，红框的部分，是线程号，我们看到这些都是以0x打头的，是16进制的表示，所以需要将步骤二定位出来的线程号转换一下，可以直接在Linux命令行输入如下命令来定位上面框出来的三个线程

printf "%x\n" 93975 16f17
printf "%x\n" 93976 16f18
printf "%x\n" 93977 16f19

正好定位到三个16进制表示的线程号，至此就找到了对应的方法栈，下面就可以直接去代码里面查找问题了。

当然，对于此步骤，还可以一步执行如下命令定位，如下

jstack 93937 | grep $(printf "%x\n" 93975) -A 10

93937 表示进程号， 93975 表示有问题的线程号

![imaged13e3ac149de3dde.png](https://wx2.sbimg.cn/2020/06/19/imaged13e3ac149de3dde.png)

-A 10表示多打印10行，这个可以根据不同情况设置

# Windows环境
推荐使用JDK自带的工具Java Visual VM，一般就是这货，在JDK的bin目录下

![image7a7fc62dbb5ebea3.png](https://wx2.sbimg.cn/2020/06/19/image7a7fc62dbb5ebea3.png)

点开之后

![image12ba6b63525e09fe.png](https://wx1.sbimg.cn/2020/06/19/image12ba6b63525e09fe.png)

左侧会显示现在正在运行的Java进程，我们随便点开一个

可以看到此进程的一些运行信息，下面关注我们的问题，直接查看CPU的使用情况

可以清晰的看到CPU时间占用最多的线程，现在就还是利用jstack来定位具体是哪个方法
jstack.exe -l 6964 >dump.stack
然后就在输出文件中查找对应线程名即可。