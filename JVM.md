# JVM

# 1.简介

- JVM是【规范】，最常用的是Oracle的HotSpot

- HotSpot的Java线程与OS线程【直接映射】：Java线程准备完毕 --> 创建OS线程 --> OS线程准备完毕 --> 调用run() --> Java线程结束 --> OS线程回收

- **JVM的组成**

  ![image-20220527100121805](C:\Users\91494\AppData\Roaming\Typora\typora-user-images\image-20220527100121805.png)

- 程序计数器、虚拟机栈、本地方法栈**线程私有**，堆、方法区、直接内存**线程共享**，前者生命周期**与线程相同**，后者生命周期**与虚拟机相同**

- 直接内存并不是 JVM 运行时数据区的一部分, 但也会被频繁的使用: 在 JDK 1.4 引入的 NIO 提

  供了基于 Channel 与 Buffer 的 IO 方式, 它可以使用 Native 函数库直接分配堆外内存, 然后使用

  DirectByteBuffer 对象作为这块内存的引用进行操作(详见: Java I/O 扩展), 这样就避免了在 Java

  堆和 Native 堆中来回复制数据, 因此在一些场景中可以显著提高性能



# 2.内存结构

## 2.1 程序计数器

- 记录正在执行的字节码指令地址，物理上通过【CPU寄存器】实现，**唯一不会内存溢出**的区域

## 2.2 虚拟机栈

- **线程运行时**的内存空间，一个线程一个栈
- 一次方法执行对应一个栈帧，存储局部变量表、操作数栈、动态链接、方法出口等，栈顶为【活动栈帧】
- 通过`-Xss`设置大小，默认1M，Windows不一定
- 局部变量不安全：引用外部对象 or 作为返回结果
- 栈内存溢出：栈帧过多 or 栈帧过大(少见) or 循环引用
- 线程运行诊断
  - CPU占用过多
    - 通过`top`定位进程
    - 通过`ps H -eo pid,tid,%cpu | grep 进程id`定位线程
    - `jstack 进程id`列出进程的所有线程，找到对应线程可定位出问题的代码位置
  - 程序运行很久得不到结果，如死锁
    - `jstack 进程id`输出的最后会显示死锁信息，可定位出问题的代码位置

## 2.3 本地方法栈

- 为Native方法服务

## 2.4 堆

- new的对象和数组存放的位置，需要考虑线程安全
- 需要GC，从GC角度可将堆分为新生代（Eden、From Survivor区和To Survivor区）和老年代
- 通过`-Xmx`设置大小，默认4G，可能会出现堆内存溢出
- 相关工具
  - jps工具：查看当前系统的java进程
  - jmap工具：查看某时刻堆内存占用情况，`jmap -heap 进程id`
  - jconsole工具：图形界面持续监测工具
  - jvisualvm工具

## 2.5 方法区

- 用来存储加载的类信息、常量、静态变量、JIT编译后的代码等
- 方法区是【概念】，HotSpot使用Java堆的永久代实现，因此也可进行GC（针对常量池的回收和类型的卸载）
- 通过`-XX:MaxMetaspaceSize/MaxPermSize=`设置大小
- ，1.7之前通过【永久代】实现，其中包括【CLass、ClassLoader和常量池】
  - 1.7后通过【元空间】实现，存储的内容相同（常量池中的StringTable存在heap中），但不占用JVM内存，而使用【本地内存】
- 出现场景：Spring和MyBatis的cglib动态类加载

### 常量池

- 二进制字节码中包括类基本信息、常量池、类方法定义(包含了虚拟机指令)
- 通过`javap -c xxx.class`反编译
- 每条指令后都有`#数字`，代表的就是常量池（一张表）中的位置，根据其找到要执行的类名、方法名、参数类型、字面量等信息
- 常量池存在.class文件中，当类被加载后常量池就会被放入【运行时常量池】，符号地址也变为【真实地址】

### StringTable

- 符号与懒惰加载：常量池中的【符号】只有在被用到时才会被放入StringTable变成Java【对象】
- StringTable为HashTable存储，不可扩容
- 字符串拼接
  - 变量拼接：new一个StringBuilder进行拼接后再toString()转为String，存储于堆中而非StringTable中
  - 常量拼接：直接去Table中找，==结果为true，通过javac【编译期优化】实现，因为都是常量，所以拼接结果也是确定的
- intern()会将字符串放入串池并返回
- 存储位置：1.7前存在JVM的永久代的常量池中，1.7后单独存在堆中，常量池存在元空间中，前者在full GC时回收，后者在minor GC时回收，减轻了字符串对内存的占用
- StringTable也会进行GC
- 性能调优：`XX:StringTableSize`调整桶个数以减少碰撞或者使用intern()不将重复字符串对象入池



# 3.直接内存

- 常用在NIO操作中作为数据缓冲区
- 分配回收成本高，但读写性能高
- 不受JVM内存回收管理
- 使用ByteBuffer读写效率高：CPU需要在用户态和内核态间转换，而磁盘文件先加载至系统内存的系统缓存区中，再加载至Java堆内存的Java缓冲区byte[]中，而通过ByteBuffer.allocateDirect()可以在【系统内存】中划分一块缓冲区，**该区域可以被Java代码访问**
- 直接内存释放原理：通过**Unsafe对象**分配和释放内存(ByteBuffer构造方法中就是用的Unsafe)，而System.gc()是通过回收Java对象引起直接内存回收的
  - ByteBuffer.allocateDirect()会构造一个DirectByteBuffer对象，其构造方法中会调用unsafe.allocateMemory()，同时会创建一个【虚引用】对象Cleaner = Cleaner.create(this, new Deallocator(base, size, cap))，当this关联的Java对象被JVM回收后，就会调用Deallocator中的unsafe.freeMemory()回收直接内存
- `XX:+DIsableExplicitGC`禁用System.gc()，以防Full GC导致程序暂停时间过长，但这会影响直接内存的回收，这种情况下就可以手动使用Unsafe释放直接内存



# 4.垃圾回收

## 4.1 判断可回收对象

- 引用计数法：计数值为0时就可回收，但会存在【循环引用】问题

- **可达性分析算法**：首先确定一系列【根对象】，即肯定不能当成垃圾被回收的对象，在GC前会对堆中所有对象扫描判断是否被GC roots对象【直接或间接引用】，【堆外的结构中对堆内对象进行引用】的都可作为GC Root，如虚拟机栈、本地方法栈、方法区、串池等，如果【一个指针保存了堆中对象但自己又不在堆中】，那他就是一个Root

- 四种引用

  1. 强引用：new的对象，只有引用【全部断开】才能被回收

  2. 软引用：被强引用间接引用的对象，当GC完后内存不够时才会回收，可以回到引用队列中

     ```java
     List<SoftReference<byte[]>> list = new Arraylist<>();
     SoftReference<byte[]> ref = new SoftReference(new byte[]);
     
     // 引用队列用于清理软引用对象本身
     ReferenceQueue<byte[]> queue = new ReferenceQueue<>();
     // 当软引用关联的对象被回收时，对象自身会进入队列中
     SoftReference<byte[]> ref = new SoftReference(new byte[], queue);
     while (queue.poll() != null) {
         list.remove(poll);
         poll.queue.poll();
     }
     ```

  3. 弱引用WeakReference：被强引用间接引用的对象，只要GC就会回收，可以回到引用队列中

  4. 虚引用：必须配合引用队列使用，当引用的对象被回收时会进入队列，由ReferenceHandler定期检查队列释放关联的直接内存

  5. 终结器引用：必须配合引用队列使用，强引用断开后进入队列，必须重写Object.finallize()且由优先级很低的线程负责检查队列并回收

## 4.2 GC算法

1. 标记清除：将没有GC root引用的对象先标记，下次分配空间时【直接覆盖】，优点是【快】，缺点是会出现【内存碎片】
2. 标记整理：先标记，并【移动】可用对象
3. 复制：将内存划分为相等大小的2块区域FROM与TO，将FROM中存活的对象复制到TO中，清空FROM并交换TO，没碎片但需要双倍空间

## 4.3 分代GC

- 实际的JVM会采用多种GC算法，通过分代实现，将堆内存划分为新生代和老年代(存储长时间使用对象)，新生代又分为Eden、From和To

  1. 新创建的对象先存在Eden中
  2. 当Eden满了之后就会通过可达性分析算法进行Minor GC，此时除垃圾回收线程之外的线程【全部暂停】，如果存活就复制到To中，寿命+1，并交换From与To指针
  3. 下次Eden又满了时候会一起判断Eden和From中的对象是否有引用
  4. 如果寿命超过阈值(默认15)就会去老年代
  5. Minor GC后老年代也放不下了就会进行Full GC，也会stop the world
  6. 如果此时仍不够内存就会抛出heap out of memory error

- 大对象直接晋升：当新生代内存不够存放大对象时不会触发GC而是直接放入老年代中

- 当一个线程抛出OOM异常后，其所占据的内存会全部释放，因此不会影响其他线程执行

- 相关参数

  ![image-20220528144908230](C:\Users\91494\AppData\Roaming\Typora\typora-user-images\image-20220528144908230.png)

## 4.4 垃圾回收器

### 1.串行Serial

- 使用单个GC线程，【暂停其他线程】
- 适合堆内存较小（额外内存消耗低）、CPU核少（无线程交互开销）的情况，适合客户端
- 新生代采用【复制】算法，老年代采用【标记整理】算法
- 改进：ParNewGC，新生代采用多线程并行收集，老年代还是单线程

### 2.吞吐量优先Parallel Scavenge

- 吞吐量 = 用户代码运行时间 / (用户代码+GC)，高吞吐量和低暂停时间是相悖的

- 采用多线程，尽可能少地进行GC以提高回收效率

- 新生代为【复制】算法，老年代为【标记整理】算法

### 3.响应时间优先(Concurrent Mark Sweep, CMS)

- 追求最短回收停顿时间，适用于关注响应速度的应用
- 基于【标记清除】算法
- 步骤
  1. 初始标记：标记GC Roots【直接关联】的对象，速度很快，【需要STW】
  2. 并发标记：标记【所有】GC Roots关联的对象，耗时较长但不停止用户线程
  3. 重新标记：修正被用户线程修改的对象标记，【需要STW】
  4. 并发清除：清除已死亡对象，不用停止用户线程

- 缺点
  1. 无法处理浮动垃圾，即本次GC时用户运行产生的垃圾
  2. 需要预留部分空间供用户线程在标记时使用
  3. 标记清除算法会产生内存碎片

### 4.Garbage First(G1)

- 之前的GC思想都是针对【整个区域】，新生代、老年代或是整个堆，而G1只看哪块内存垃圾最多、回收收益最大

- 通过Region实现，每个Region【都可】作为Eden，Survivor或Old，不再是固定的连续区域，而是个【动态集合】

- 9默认开启，同时注重吞吐量和低延时，适合超大堆内存，整体上是【标记整理】算法，Region间是【复制】算法

- JDK8u20字符串去重：将所有新分配字符串放入队列，当新生代回收时G1并发检查是否重复，如果值一样就让其引用**同一个char[]**

- 8u40并发标记类卸载：对象经过并发标记后就能知道哪些类不再使用，当一个类加载器的所有类都不再使用就卸载其加载的所有类

- 8u60回收巨型对象：当一个对象大于Region一半时就是巨型对象，存储在连续Region中，视为老年代，不会拷贝且优先回收

- JDK9调整并发标记起始时间：并发标记必须在堆空间占满前完成，不然会退化为Full GC，9可以动态调整，9还做了很多功能增强

- 跨代引用问题：GC Roots很可能存在老年代中并引用新生代，而老年代空间大遍历时间长

  - 解决思路：在新生代中建立一个全局的【记忆集】Remembered Set，将【老年代】划分为【若干块】，标记哪一块会存在跨代引用
  - 卡精度精确到一块内存区域是否含有跨代指针，称为卡页，通过卡表实现，只要卡页内有对象存在跨代指针就将对应卡表的数组元素标识为1
  - 何时变脏，如何变脏：发生引用时，通过写屏障维护卡表状态，在引用赋值时增加【环绕通知】

- 三色标记：降低用户线程的停顿

  - 白色：对象尚未被GC访问
  - 黑色：对象已被GC访问过且其所有引用都已访问过，是安全存活的，如果其他对象指向黑色则无须扫描，黑色不能直接指向白色
  - 灰色：对象已被GC访问过但至少有一个引用没被扫描
  - 当以下2个条件同时满足时会产生【对象消失】问题
    - 赋值器插入了一条或多条从黑色对象到白色对象的新引用 -- 【增量更新】方法解决 -- 重新扫描一次插入新引用的黑色节点
    - 赋值器删除了全部从灰色对象到该白色对象的直接或间接引用 -- 【原始快照】方法解决 -- 记录删除的记录重新扫描一次

- 步骤

  1. 初始标记：标记GC Roots直接关联的对象，需要STW但耗时很短且在Minor GC时同步完成

     <img src="C:\Users\91494\AppData\Roaming\Typora\typora-user-images\image-20220528155113562.png" alt="image-20220528155113562" style="zoom:33%;" /><img src="C:\Users\91494\AppData\Roaming\Typora\typora-user-images\image-20220528155017452.png" alt="image-20220528155017452" style="zoom:33%;" /><img src="C:\Users\91494\AppData\Roaming\Typora\typora-user-images\image-20220528155042210.png" alt="image-20220528155042210" style="zoom:33%;" />

  2. 并发标记：扫描整个堆中对象图，可并发执行

  <img src="C:\Users\91494\AppData\Roaming\Typora\typora-user-images\image-20220528155151046.png" alt="image-20220528155151046" style="zoom:33%;" />

  3. 最终标记：短暂暂停用户线程，处理遗留记录
  3. 筛选回收：根据用户设置时间选择高价值区域进行回收，将该Region中存活对象复制到空Region中，再清理整个Region，STW

  <img src="C:\Users\91494\AppData\Roaming\Typora\typora-user-images\image-20220528155314306.png" alt="image-20220528155314306" style="zoom:33%;" />

1. 4种GC器在新生代内存不足时都是Minor GC，前2种在老年代不足时是Full GC，CMS并发收集失败后变成Full GC，当G1收集垃圾速度跟不上产生速度时就会退化成串行GC变成Full GC

## 4.5 GC调优

- 调优目标：低延迟(G1)还是高吞吐量(ParallelGC)
- 最快的GC就是不发生GC：查看Full GC前后内存占用，考虑
  - 数据是否太多
  - 数据表示是否太臃肿：对象图、对象大小
  - 是否存在内存泄漏
- 建议从新生代开始调优
  - Eden从new对象分配内存时减少内存冲突：TLAB thread-local allocation buffer
  - 死亡对象回收代价为0；大部分对象用过即死；MinorGC远小于FullGC
  - 调优方式：调大新生代，但不是越大越好，因为会导致老年代变小引发FullGC，建议在1/4-1/2之间
- 幸存区要大到能保留当前活跃对象+需要晋升对象，如果太小会动态调整晋升时间放入老年代触发Full GC，又希望长时间存活对象尽快晋升
- CMS的老年代内存越大越好，如果没有Full GC就不需要调
- 案例1：FullGC和MinorGC频繁，原因是新生代太小，很多该回收的进入老年代触发FullGC，方法是调大新生代大小以及晋升区空间和晋升阈值
- 案例2：高峰期发生FullGC，单次暂停时间长，使用的是CMS，原因是重新标记时间长，方法是在重新标记前先清理一次