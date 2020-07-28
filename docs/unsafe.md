-------


# Java-Unsafe
Unsafe 是 sun.misc 包下的一个类，可以直接操作堆外内存，可以随意查看及修改 JVM 中运行时的数据，使 Java 语言拥有了类似 C 语言指针一样操作内存空间的能力。

Unsafe 的操作粒度不是类，而是内存地址和所对应的数据，增强了 Java 语言操作底层资源的能力。

 

# 一、获得 Unsafe 实例
查看 Unsafe.java 源码：https://hg.openjdk.java.net/jdk8u/jdk8u/jdk/file/3ef3348195ff/src/share/classes/sun/misc/Unsafe.java


    public final class Unsafe {
    private Unsafe() {}
    
    // 单例对象
    private static final Unsafe theUnsafe = new Unsafe();
    
    @CallerSensitive
    public static Unsafe getUnsafe() {
    Class<?> caller = Reflection.getCallerClass();
    // 仅在引导类加载器 BootstrapClassLoader 加载时才合法
    if (!VM.isSystemDomainLoader(caller.getClassLoader()))
    throw new SecurityException("Unsafe");
    return theUnsafe;
    }
    }


由此可以看出自己写的类即不能 new Unsafe() 对象也不能调用其 getUnsafe() 方法。

若想使用这个类只能通过其它方法，有如下两个可行方案。

1.让自己写的类使用 BootstrapClassLoader 加载
# unix 使用:号，windows 使用;号，这里以 windows 为例，使用了 Unsafe 类的 jar 包路径为 /hone/myUnsafe.jar
java -Xbootclasspath/a;/home/myUnsafe.jar com.unsafeTest
2.通过反射获取单例对象 theUnsafe
    
    private static Unsafe reflectGetUnsafe() {
    try {
    // 获得 theUnsafe 属性对象
    Field field = Unsafe.class.getDeclaredField("theUnsafe");
    // 取消权限控制检查，让其可获得 private 修饰属性
    field.setAccessible(true);
    // 获得 Unsafe.class 的 theUnsafe 属性（如果底层字段是一个静态字段，则忽略 obj 参数；它可能为 null）
    return (Unsafe) field.get(null);
    } catch (Exception e) {
    e.printStackTrace();
    return null;
    }
    }
    
 

二、Unsafe 常用 API 介绍
Unsafe 类大部分都是 native 方法，具体实现由 JVM 完成：https://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/7576bbd5a03c/src/share/vm/prims/unsafe.cpp
![image.png](https://i.loli.net/2020/06/18/Op8YDvhIK9mXrUM.png)


## 1.内存操作（堆外内存）

    // 分配内存, 相当于 C++ 的 malloc 函数
    public native long allocateMemory(long bytes);
    
    // 扩充内存
    public native long reallocateMemory(long address, long bytes);
    
    // 释放内存
    public native void freeMemory(long address);
    
    // 在给定的内存块中设置值
    public native void setMemory(Object o, long offset, long bytes, byte value);
    
    // 内存拷贝
    public native void copyMemory(Object srcBase, long srcOffset, Object destBase, long destOffset, long bytes);
    
    // 获取给定地址值，忽略修饰限定符的访问限制。与此类似操作还有: getInt，getDouble，getLong，getChar 等
    public native Object getObject(Object o, long offset);
    
    // 为给定地址设置值，忽略修饰限定符的访问限制，与此类似操作还有: putInt,putDouble，putLong，putChar 等
    public native void putObject(Object o, long offset, Object x);
    
    // 获取给定地址的 byte 类型的值（当且仅当该内存地址为 allocateMemory 分配时，此方法结果为确定的）
    public native byte getByte(long address);
    
    // 为给定地址设置 byte 类型的值（当且仅当该内存地址为 allocateMemory 分配时，此方法结果才是确定的）
    public native void putByte(long address, byte x);

## 2.CAS

    /**
     * CAS
     *
     * @param o包含要修改field的对象
     * @param offset   对象中某field的偏移量
     * @param expected 期望值
     * @param update   更新值
     * @return true | false
     */
    public final native boolean compareAndSwapObject(Object o, long offset, Object expected, Object update);
    
    public final native boolean compareAndSwapInt(Object o, long offset, int expected, int update);
    
    public final native boolean compareAndSwapLong(Object o, long offset, long expected, long update);

![image.png](https://i.loli.net/2020/06/18/ToBlR69MsJya1hW.png)

## 3.线程调度

    // 阻塞线程
    public native void park(boolean isAbsolute, long time);
    
    // 取消阻塞线程
    public native void unpark(Object thread);
    
    // 获得对象锁（可重入锁）
    @Deprecated
    public native void monitorEnter(Object o);
    
    // 释放对象锁
    @Deprecated
    public native void monitorExit(Object o);
    
    // 尝试获取对象锁
    @Deprecated
    public native boolean tryMonitorEnter(Object o);

## 4.Class 相关

    // 获取给定静态字段的内存地址偏移量，这个值对于给定的字段是唯一且固定不变的
    public native long staticFieldOffset(Field f);
    
    // 获取一个静态类中给定字段的对象指针
    public native Object staticFieldBase(Field f);
    
    // 判断是否需要初始化一个类，通常在获取一个类的静态属性的时候（因为一个类如果没初始化，它的静态属性也不会初始化）使用。 当且仅当 ensureClassInitialized 方法不生效时返回 false。
    public native boolean shouldBeInitialized(Class<?> c);
    
    // 检测给定的类是否已经初始化。通常在获取一个类的静态属性的时候（因为一个类如果没初始化，它的静态属性也不会初始化）使用。
    public native void ensureClassInitialized(Class<?> c);
    
    // 定义一个类，此方法会跳过 JVM 的所有安全检查，默认情况下，ClassLoader（类加载器）和 ProtectionDomain（保护域）实例来源于调用者
    public native Class<?> defineClass(String name, byte[] b, int off, int len, ClassLoader loader, ProtectionDomain protectionDomain);
    
    // 定义一个匿名类
    public native Class<?> defineAnonymousClass(Class<?> hostClass, byte[] data, Object[] cpPatches);

## 5.对象操作

    // 返回对象成员属性在内存地址相对于此对象的内存地址的偏移量
    public native long objectFieldOffset(Field f);
    
    // 获得给定对象的指定地址偏移量的值，与此类似操作还有：getInt，getDouble，getLong，getChar等
    public native Object getObject(Object o, long offset);
    
    // 给定对象的指定地址偏移量设值，与此类似操作还有：putInt，putDouble，putLong，putChar等
    public native void putObject(Object o, long offset, Object x);
    
    // 从对象的指定偏移量处获取变量的引用，使用volatile的加载语义
    public native Object getObjectVolatile(Object o, long offset);
    
    // 存储变量的引用到对象的指定的偏移量处，使用 volatile 的存储语义
    public native void putObjectVolatile(Object o, long offset, Object x);
    
    // 有序、延迟版本的 putObjectVolatile 方法，不保证值的改变被其他线程立即看到。只有在 field 被 volatile 修饰符修饰时有效
    public native void putOrderedObject(Object o, long offset, Object x);
    
    // 绕过构造方法、初始化代码来创建对象
    public native Object allocateInstance(Class<?> cls) throws InstantiationException;

## 6.数组相关
    // 返回数组中第一个元素的偏移地址
    public native int arrayBaseOffset(Class<?> arrayClass);
    
    // 返回数组中一个元素占用的大小
    public native int arrayIndexScale(Class<?> arrayClass);

## 7.内存屏障

    // 内存屏障，禁止 load 操作重排序。屏障前的 load 操作不能被重排序到屏障后，屏障后的 load 操作不能被重排序到屏障前
    public native void loadFence();
    
    // 内存屏障，禁止 store 操作重排序。屏障前的 store 操作不能被重排序到屏障后，屏障后的 store 操作不能被重排序到屏障前
    public native void storeFence();
    
    // 内存屏障，禁止 load、store 操作重排序
    public native void fullFence();

##8.系统相关
    // 返回系统指针的大小。返回值为 4（32位系统）或 8（64位系统）。
    public native int addressSize();
    
    // 内存页的大小，此值为 2 的幂次方。
    public native int pageSize();
 

# 三、简单使用
## 创建对象并修改其属性
跳过对象初始化阶段，或绕过构造器的安全检查，或实例化一个没有任何公共构造器的类。

反射可以实现相同的功能。但值得关注的是，我们可以修改任何对象，甚至没有这些对象的引用。

    
    class User {
    private String name;
    private int age;
    
    public User(String name, int age) {
    this.name = name;
    this.age = age;
    }
    
    public String getName() {
    return name;
    }
    
    public void setName(String name) {
    this.name = name;
    }
    
    public int getAge() {
    return age;
    }
    
    public void setAge(int age) {
    this.age = age;
    }
    
    @Override
    public String toString() {
    return "User{" + "name='" + name + '\'' + ", age=" + age + '}';
    }
    }
    
    
    public static void main(String[] args) throws Exception {
    Class userClass = User.class;
    // 避开构造方法初始化对象
    User user = (User) unsafe.allocateInstance(userClass);
    
    user.setAge(20);
    System.out.println(user);
    
    // 修改对象成员值
    Field field = User.class.getDeclaredField("age");
    unsafe.putInt(user, unsafe.objectFieldOffset(field), 8);
    System.out.println(user);
    }
    
    private static Unsafe unsafe;
    
    static {
    unsafe = reflectGetUnsafe();
    }
    
    private static Unsafe reflectGetUnsafe() {
    try {
    Field field = Unsafe.class.getDeclaredField("theUnsafe");
    field.setAccessible(true);
    return (Unsafe) field.get(null);
    } catch (Exception e) {
    e.printStackTrace();
    return null;
    }
    }

## 大数组
Java 数组大小的最大值为 Integer.MAX_VALUE。使用直接内存分配，我们创建的数组大小受限于堆大小。

使用堆外内存（off-heap memory）技术，这种方式的内存分配不在堆上，且不受 GC 管理，所以必须小心 Unsafe.freeMemory() 的使用。它也不执行任何边界检查，所以任何非法访问可能会导致 JVM 崩溃。


    class SuperArray {
    private final static int BYTE = 1;
    private long size;
    private long address;
    private static Unsafe unsafe;
    
    static {
    unsafe = reflectGetUnsafe();
    }
    
    public static Unsafe getUnsafe() {
    return unsafe;
    }
    
    public long getAddress() {
    return address;
    }
    
    private static Unsafe reflectGetUnsafe() {
    try {
    Field field = Unsafe.class.getDeclaredField("theUnsafe");
    field.setAccessible(true);
    return (Unsafe) field.get(null);
    } catch (Exception e) {
    e.printStackTrace();
    return null;
    }
    }
    
    public SuperArray(long size) {
    this.size = size;
    address = unsafe.allocateMemory(size * BYTE);
    }
    
    public void set(long i, byte value) {
    unsafe.putByte(address + i * BYTE, value);
    }
    
    public int get(long idx) {
    return unsafe.getByte(address + idx * BYTE);
    }
    
    public long size() {
    return size;
    }
    }
    
    public static void main(String[] args) {
    int sum = 0;
    long SUPER_SIZE = (long) Integer.MAX_VALUE * 2;
    SuperArray array = new SuperArray(SUPER_SIZE);
    System.out.println("Array size:" + array.size()); // 4294967294
    for (int i = 0; i < 100; i++) {
    array.set((long) Integer.MAX_VALUE + i, (byte) 3);
    sum += array.get((long) Integer.MAX_VALUE + i);
    }
    System.out.println("Sum of 100 elements:" + sum);  // 300
    SuperArray.getUnsafe().freeMemory(array.getAddress());
    }