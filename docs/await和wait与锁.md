#await/wait与锁

![UKOxyt.png](https://s1.ax1x.com/2020/07/10/UKOxyt.png)

在使用Lock之前，我们都使用Object 的wait和notify实现同步的。举例来说，一个producer和consumer，consumer发现没有东西了，等待，produer生成东西了，唤醒。

![UKXuwT.png](https://s1.ax1x.com/2020/07/10/UKXuwT.png)

有了lock后，世道变了，现在是：

![UKXlY4.png](https://s1.ax1x.com/2020/07/10/UKXlY4.png)

为了突出区别，省略了若干细节。区别有三点：

1. lock不再用synchronize把同步代码包装起来；
2. 阻塞需要另外一个对象condition；
3. 同步和唤醒的对象是condition而不是lock，对应的方法是await和signal，而不是wait和notify。
为什么需要使用condition呢？简单一句话，lock更灵活。以前的方式只能有一个等待队列，在实际应用时可能需要多个，比如读和写。为了这个灵活性，lock将同步互斥控制和等待队列分离开来，互斥保证在某个时刻只有一个线程访问临界区（lock自己完成），等待队列负责保存被阻塞的线程（condition完成）。

通过查看ReentrantLock的源代码发现，condition其实是等待队列的一个管理者，condition确保阻塞的对象按顺序被唤醒。

1.object.wait()

使用方法:

	线程A:
	
	synchronized(obj){
	
	obj.wait(); //此时当前线程释放obj锁，进入[等待状态]，等待其他线程执行obj.notify()时才有可能执行（有可能执行的意思可能有多个线程执行了wait）
	
	A do something
	
	}

	线程C:
	
	synchronized(obj){
	
	obj.wait(); //此时当前线程释放obj锁，进入[等待状态]，等待其他线程执行obj.notify()时才有可能执行（有可能执行的意思可能有多个线程执行了wait）
	
	C do something
	
	}

	线程B:
	
	synchronized(obj){
	
	obj.notify(); //此时当前线程释放obj锁，随机唤醒一个处于等待状态的线程，继续执行wait后面的程序。
	
	}

假设三个线程执行顺序

线程A-->线程C-->线程B //没毛病，因为wait后是释放了锁的

所以问题来了：等待的线程中有A和C, B notify后，只会唤醒其中一个执行(notifyAll同样只有一个执行)；假如我们的需求是想让A线程执行，那么这种object的方式是无法控制的

2.所以condition来了

使用方法：注意condition是依赖ReentrantLock

	ReentrantLock lock = new ReentrantLock(true);
	
	Condition aCondition = reentrantLock.newCondition();
	
	Condition cCondition = reentrantLock.newCondition();
	
	线程A:
	
	{
	
	lock.lock();
	
	aCondition.await(); //此时当前线程释放lock锁，进入[等待状态]，等待其他线程执行aCondition.signal()时才有可能执行
	
	A do something
	
	lock.unlock();
	
	}
	
	线程C:
	
	{
	
	lock.lock();
	
	cCondition.await();
	
	do something
	
	lock.unlock();
	
	}
	
	线程B:
	
	{
	
	lock.lock();
	
	aCondition.signal(); //此时当前线程释放lock锁，随机唤醒一个处于等待状态等待aCondition的线程，继续执行await后面的程序。
	
	//cCondition.signal(); ////此时当前线程释放lock锁，随机唤醒一个处于等待状态等待cCondition的线程，继续执行await后面的程序。
	
	lock.unlock();
	
	}

所以通过condition与object进行线程通信的区别已经很明显了，condition更加灵活。

个人理解，本质上来讲：

多线程环境的下，线程直接的互斥[执行]依靠的应该是锁Lock，线程的之间的[通信]依靠的应该是条件Condition/信号，一般情况下lock确实可以同时满足做这两个事情，所以在Object的方式满足了这个一般情况，但是肯定会有复杂的场景比如刚才例子中，需要让满足一定条件的线程执行，仅仅依靠锁是不能完美解决的。所以condition实际上分离了执行和通信。