<!-- TOC -->

- [Java中锁的详解](#java中锁的详解)
  - [背景](#背景)
    - [i--/++分析](#i--分析)
      - [字节码指令](#字节码指令)
    - [临界区 Critical Section](#临界区-critical-section)
    - [竞态条件 Race Condition](#竞态条件-race-condition)
    - [解决方式](#解决方式)
  - [锁分类](#锁分类)
    - [宏观分类](#宏观分类)
      - [乐观锁 （宏观分类）](#乐观锁-宏观分类)
      - [悲观锁（宏观分类）](#悲观锁宏观分类)
    - [公平分类](#公平分类)
      - [非公平锁 （Nonfair）](#非公平锁-nonfair)
      - [公平锁 （Fair）](#公平锁-fair)
    - [共享锁和独占锁](#共享锁和独占锁)
      - [独占锁](#独占锁)
      - [共享锁](#共享锁)
    - [可重入锁（递归锁）](#可重入锁递归锁)
  - [锁优化](#锁优化)
    - [锁升级](#锁升级)
      - [偏向锁（偏向第一个获取它的线程，优化锁机制）](#偏向锁偏向第一个获取它的线程优化锁机制)
        - [偏向锁获取过程](#偏向锁获取过程)
        - [偏向锁的撤销](#偏向锁的撤销)
        - [HashCode](#hashcode)
        - [使用场景](#使用场景)
        - [查看停顿–安全点停顿日志](#查看停顿安全点停顿日志)
          - [安全点日志分析](#安全点日志分析)
      - [轻量级锁](#轻量级锁)
        - [轻量级锁加锁过程](#轻量级锁加锁过程)
        - [轻量级锁的释放](#轻量级锁的释放)
          - [释放锁线程视角](#释放锁线程视角)
          - [尝试获取锁线程视角](#尝试获取锁线程视角)
      - [自旋锁](#自旋锁)
        - [自旋锁的优缺点](#自旋锁的优缺点)
        - [自旋锁时间阈值（1.6引入了适应性自旋锁）](#自旋锁时间阈值16引入了适应性自旋锁)
        - [自旋锁的开启](#自旋锁的开启)
      - [重量级锁（Mutex Lock）](#重量级锁mutex-lock)
    - [锁分离](#锁分离)
      - [读写锁](#读写锁)
        - [读锁（并发读）](#读锁并发读)
        - [写锁（单独写）](#写锁单独写)
        - [Java中的实现](#java中的实现)
    - [锁粗化（减小锁的粒度）](#锁粗化减小锁的粒度)
      - [分段锁-ConcurrentHashMap](#分段锁-concurrenthashmap)
      - [LongAdder](#longadder)
      - [LinkedBlockingQueue](#linkedblockingqueue)
  - [其他](#其他)
    - [同步锁与死锁](#同步锁与死锁)
      - [同步锁](#同步锁)
      - [死锁](#死锁)
    - [互斥同步](#互斥同步)
  - [参考](#参考)

<!-- /TOC -->

# Java中锁的详解

## 背景

在多线程同时访问同一个资源时，如果需要考虑对共享资源的竞争问题以及数据的正确性，这需要引入锁的概念，这就是所谓的线程安全。最常见的场景就是i--操作。

### i--/++分析

#### 字节码指令

```java
getstatic i // 获取静态变量i的值
iconst_1    // 准备常量1
isub        // 自减
putstatic i // 将修改后的值存入静态变量i

getstatic i // 获取静态变量i的值
iconst_1    // 准备常量1
iadd        // 自增
putstatic i // 将修改后的值存入静态变量i
```

<div align=center>

![1590651451164.png](..\images\1590651451164.png)

</div>

### 临界区 Critical Section

1. 一个程序运行多个线程本身是没有问题的
2. 问题出在多个线程访问共享资源
   - 多个线程读共享资源其实也没有问题
   - 在多个线程对共享资源读写操作时发生指令交错，就会出现问题
3. 一段代码块内如果存在对共享资源的多线程读写操作，称这段代码块为临界区

### 竞态条件 Race Condition

多个线程在临界区内执行，由于代码的执行序列不同而导致结果无法预测，称之为发生了竞态条件

### 解决方式

1. 阻塞：锁（synchronized、lock）
2. 非阻塞：原子化、cas

## 锁分类

### 宏观分类

#### 乐观锁 （宏观分类）

1. 场景：多读少写
2. 思想：读数据时认为数据不会被修改，不加锁；写数据时需要判断数据是否被修改过
3. Java实现：版本控制和CAS
4. CAS说明：一种更新的原子操作，比较当前值跟传入值是否一样，一样则更新，否则失败）

#### 悲观锁（宏观分类）

1. 场景：写多读少
2. 每次获取数据时都需要加锁，防止被别人修改
3. Java实现：Synchronized,AQS框架下的锁则是先尝试cas乐观锁去获取锁，获取不到，才会转换为悲观锁，如 RetreenLock。 

### 公平分类

#### 非公平锁 （Nonfair）

1. 原理：JVM 按随机、就近原则分配锁
2. ReentrantLock：默认非公平，可以配置为公平
3. 常用：非公平执行效率高于公平锁
4. 加锁时不考虑排队等待问题，直接尝试获取锁，获取不到自动到队尾等待 
   - 非公平锁性能比公平锁高 5~10 倍，因为公平锁需要在多核的情况下维护一个队列 
5. Java 中的 synchronized 是非公平锁，ReentrantLock 默认的 lock()方法采用的是非公平锁。 

#### 公平锁 （Fair）

1. 先对锁提出获取请求的线程会先被分配到锁
2. ReentrantLock 在构造函数中提供了是否公平锁的初始化方式来定义公平锁
3. 加锁前检查是否有排队等待的线程，优先排队等待的线程，先来先得

### 共享锁和独占锁 

java 并发包提供的加锁模式分为独占锁和共享锁。 

#### 独占锁 

独占锁模式下，每次只能有一个线程能持有锁，ReentrantLock 就是以独占方式实现的互斥锁。独占锁是一种悲观保守的加锁策略，它避免了读/读冲突，如果某个只读线程获取锁，则其他读线程都只能等待，这种情况下就限制了不必要的并发性，因为读操作并不会影响数据的一致性。 

#### 共享锁 

共享锁则允许多个线程同时获取锁，并发访问 共享资源，如：ReadWriteLock。共享锁则是一种乐观锁，它放宽了加锁策略，允许多个执行读操作的线程同时访问共享资源。 
1.	AQS 的内部类 Node 定义了两个常量 SHARED 和 EXCLUSIVE，他们分别标识 AQS 队列中等待线程的锁获取模式。 
2.	java 的并发包中提供了 ReadWriteLock，读-写锁。它允许一个资源可以被多个读操作访问，或者被一个 写操作访问，但两者不能同时进行。 

### 可重入锁（递归锁）

重入锁，也叫做递归锁，指的是同一线程 外层函数获得锁之后 ，内层递归函数仍然有获取该锁的代码，但不受影响。在 JAVA 环境下 ReentrantLock 和 synchronized 都是 可重入锁。

## 锁优化

**对象头相关知识点，参考[HotSpot虚拟机对象探秘](book/jvm-hotspot-object.md)**

高效并发是从JDK 5升级到JDK 6后一项重要的改进项，HotSpot虚拟机开发团队在这个版本上花费了大量的资源去实现各种锁优化技术，如**适应性自旋（Adaptive Spinning）、锁消除（Lock Elimination）、锁膨胀（LockCoarsening）、轻量级锁（Lightweight Locking）、偏向锁（Biased Locking）** 等，这些技术都是为了在线程之间更高效地共享数据及解决竞争问题，从而提高程序的执行效率。
**锁的状态总共有四种：无锁状态、偏向锁、轻量级锁和重量级锁。** 

1. 锁升级：随着锁的竞争，锁可以从偏向锁升级到轻量级锁，再升级的重量级锁（但是锁的升级是单向的，也就是说只能从低到高升级，不会出现锁的降级）。 
2. 优化方式
   - 减少锁的持有时间：只用在有线程安全要求的线程上加锁
   - 减小锁粒度 ：偏向锁、分段锁
   - 将大对象（这个对象可能会被很多线程访问），拆成小对象，大大增加并行度，降低锁竞争。
   - 降低了锁的竞争，偏向锁，轻量级锁成功率才会提高。最最典型的减小锁粒度的案例就是ConcurrentHashMap。 
3. 锁分离
   - 读写锁分离，根据功能进行分离成读锁和写锁，这样读读不互斥，读写互斥，写写互斥，即保证了线程安全，又提高了性能
4. 锁粗化 
   - 使用锁的时间尽量短，即时进行释放
   - 防止锁的多次请求、同步和释放
5. 锁消除 锁消除是在编译器级别的事情。在即时编译器时，如果发现不可能被共享的对象，则可以消除这些对象的锁操作，多数是因为程序员编码不规范引起。 

### 锁升级

#### 偏向锁（偏向第一个获取它的线程，优化锁机制）

1. 背景：锁在大部分情况下不仅不存在多线程竞争，并且可能会被同一个线程访问多次
2. 目的：消除这个线程锁冲入的开销
3. 优点：减少不必要轻量级锁执行路径（轻量级锁的获取及释放依赖多次 CAS 原子指令，而偏向锁只需要在置换 ThreadID的时候依赖一次 CAS 原子指令（由于一旦出现多线程竞争的情况就必须撤销偏向锁，所以偏向锁的撤销操作的性能损耗必须小于节省下来的 CAS 原子指令的性能消耗））
4. 轻量级锁是为了在线程交替执行同步块时提高性能，而偏向锁则是在只有一个线程执行同步块时进一步提高性能
5. 偏向锁也是JDK 6中引入的一项锁优化措施，它的目的是消除数据在无竞争情况下的同步原语，进一步提高程序的运行性能。如果说轻量级锁是在无竞争的情况下使用CAS操作去消除同步使用的互斥量，那偏向锁就是在无竞争的情况下把整个同步都消除掉，连CAS操作都不去做了
6. 启动方式：启用参数-XX：+UseBiased Locking，这是自JDK 6起HotSpot虚拟机的默认值

##### 偏向锁获取过程

<div align=center>

![1589109028095.png](..\images\1589109028095.png)

![1590754383821.png](..\images\1590754383821.png)

</div>

1. 初始化创建对象时（上图1：**未锁定，不偏向**）
2. 第一次调用（上图2：**未锁定，已偏向**），进入偏向模式。同时使用CAS操作把获取到这个锁的线程的ID记录在对象的Mark Word之中
3. 第二次调用：通过Markword判断是否可偏向（偏向锁标识为1且锁标志为01）
   1. 相同：可偏向，判断线程ID是否与当前线程相同，相同则执行代码块
   2. 不相同，通过CAS操作竞争锁，成功则设置Markword中threadID为当前线程，并执行代码块
4. 竞争失败（锁升级为轻量级锁，上图3：），当到达全局安全点（safepoint）时获得偏向锁的线程被挂起，偏向锁升级为轻量级锁，然后被阻塞在安全点的线程继续往下执行同步代码。（撤销偏向锁的时候会导致stop the word，时间很短

##### 偏向锁的撤销

**一旦出现另外一个线程去尝试获取这个锁的情况，偏向模式就马上宣告结束。** 
根据锁对象目前是否处于被锁定的状态决定是否撤销偏向（偏向模式设置为“0”），撤销后标志位恢复到未锁定（标志位为“01”）或轻量级锁定（标志位为“00”）的状态

##### HashCode

1. 当一个对象已经计算过一致性哈希码后，它就再也无法进入偏向锁状态了
2. 而当一个对象当前正处于偏向锁状态，又收到需要计算其一致性哈希码请求时，它的偏向状态会被立即撤销，并且锁会膨胀为重量级锁。
3. 在重量级锁的实现中，对象头指向了重量级锁的位置，代表重量级锁的ObjectMonitor类里有字段可以记录非加锁状态（标志位为“01”）下的Mark Word，其中自然可以存储原来的哈希码。

##### 使用场景

始终只有一个线程在执行同步块，在它没有执行完释放锁之前，没有其它线程去执行同步块，在锁无竞争的情况下使用，一旦有了竞争就升级为轻量级锁，升级为轻量级锁的时候需要撤销偏向锁，撤销偏向锁的时候会导致stop the word操作；在有锁的竞争时，偏向锁会多做很多额外操作，尤其是撤销偏向所的时候会导致进入安全点，安全点会导致stw，导致性能下降，这种情况下应当禁用；

##### 查看停顿–安全点停顿日志

1. 打开安全点日志（不能这一会打开）
   - -XX:+PrintGCApplicationStoppedTime 会打出系统停止的时间，
   - -XX:+PrintSafepointStatistics -XX:PrintSafepointStatisticsCount=1 这两个参数会打印出详细信息，可以查看到使用偏向锁导致的停顿，时间非常短暂，但是争用严重的情况下，停顿次数也会非常多
2. 安全点日志的缺点
   - 安全点日志默认输出到stdout，一是stdout日志的整洁性，二是stdout所重定向的文件如果不在/dev/shm，可能被锁。 
   - 对于一些很短的停顿，比如取消偏向锁，打印的消耗比停顿本身还大。 
   - 安全点日志是在安全点内打印的，本身加大了安全点的停顿时间。
3. 如果在生产系统上要打开，再增加下面四个参数： 
   - -XX:+UnlockDiagnosticVMOptions 
   - -XX: -DisplayVMOutput 
   - -XX:+LogVMOutput 
   - -XX:LogFile=/dev/shm/vm.log 
4. 打开Diagnostic（只是开放了更多的flag可选，不会主动激活某个flag），关掉输出VM日志到stdout，输出到独立文件,/dev/shm目录（内存文件系统）。

<div align=center>

![1589109071755.png](..\images\1589109071755.png)

</div>

###### 安全点日志分析

1. 第一部分是时间戳，VM Operation的类型 
2. 第二部分是线程概况，被中括号括起来 
   - total: 安全点里的总线程数 
   - initially_running: 安全点开始时正在运行状态的线程数 
   - wait_to_block: 在VM Operation开始前需要等待其暂停的线程数
3. 第三部分是到达安全点时的各个阶段以及执行操作所花的时间，其中最重要的是vmop
   - spin: 等待线程响应safepoint号召的时间；
   - block: 暂停所有线程所用的时间；
   - sync: 等于 spin+block，这是从开始到进入安全点所耗的时间，可用于判断进入安全点耗时；
   - cleanup: 清理所用时间；
   - vmop: 真正执行VM Operation的时间。

**可见，那些很多但又很短的安全点，全都是RevokeBias， 高并发的应用会禁用掉偏向锁。**

#### 轻量级锁

1. 目的：轻量级锁并不是用来代替重量级锁的，轻量级锁的初衷是在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗
2. 限制：依赖于Java对象的头信息
3. 场景：线程交替执行同步块的情况，如果存在同一时间访问同一锁的情况，就会导致轻量级锁膨胀为重量级锁。 
4. 由来：轻量级锁是由偏向所升级来的，偏向锁运行在一个线程进入同步块的情况下，当第二个线程加入锁争用的时候，偏向锁就会升级为轻量级锁； 

##### 轻量级锁加锁过程

当对象锁状态为无锁状态（锁标志位为“01”状态，是否为偏向锁为“0”），虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝，官方称之为 Displaced Mark Word。这时候线程堆栈与对象头的状态如图所示： 
<div align=center>

![1589109098671.png](..\images\1589109098671.png)

</div>


1. 拷贝对象头中的Mark Word复制到锁记录中；
2. 拷贝成功后，虚拟机将使用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指针，并将Lock record里的owner指针指向object mark word
   - 如果成功，那么这个线程就拥有了该对象的锁，并且对象Mark Word的锁标志位设置为“00”，即表示此对象处于轻量级锁定状态，如图所示：
      <div align=center>

      ![1589109123275.png](..\images\1589109123275.png)

      </div>
   - 如果失败
     - 会检查对象的Mark Word是否指向当前线程的栈帧，如果是就说明当前线程已经拥有了这个对象的锁，那就可以直接进入同步块继续执行
     - 否则说明多个线程竞争锁，轻量级锁就要膨胀为重量级锁，锁标志的状态值变为“10”，Mark Word中存储的就是指向重量级锁（互斥量）的指针，后面等待锁的线程也要进入阻塞状态。 而当前线程便尝试使用自旋来获取锁，自旋就是为了不让线程阻塞，而采用循环去获取锁的过程。

##### 轻量级锁的释放

###### 释放锁线程视角

1. 轻量锁切换重量锁是发生在轻量锁释放锁的期间，
2. 比对：在释放锁的时候使用复制的Markword与对象中的Markword的比对，如果不一致则切换到重量级锁（说明在这期间有线程修改了Markword）
3. 无需mutex的场景：确认该markword是否被其他线程持有，如果否说明线程已经释放了markword，通过CAS后就可以直接进入线程，无需进入mutex

###### 尝试获取锁线程视角

1. 线程尝试获取锁，如果轻量锁正在被其他线程占有，则修改markwod为重量级锁
2. 等待轻量锁的线程不会阻塞，它会一直自旋等待锁，并如上所说修改markword。这就是自旋锁，尝试获取锁的线程，在没有获得锁的时候，不被挂起，而转而去执行一个空循环，即自旋。在若干个自旋后，如果还没有获得锁，则才被挂起，获得锁，则执行代码。

#### 自旋锁 

1. 原理：消耗CPU自旋，用来获取锁
2. 目的：减少内核态和用户态之间的切换，防止进入阻塞挂起状态
3. 退出：获取到了锁，或者超过了自旋最大时间

##### 自旋锁的优缺点

1. 减少线程的阻塞，
2. 锁竞争不激烈：可以提高性能
3. 锁竞争激烈：浪费资源
4. 持有锁的线程长时间不释放锁：浪费资源

##### 自旋锁时间阈值（1.6引入了适应性自旋锁）
 
JVM 对于自旋周期的选择，jdk1.5 这个限度是写死的，在 1.6 引入了适应性自旋锁，适应性自旋锁意味着自旋的时间不在是固定的了，而是由前一次在同一个锁上的自旋时间以及锁的拥有者的状态来决定，基本认为一个线程上下文切换的时间是最佳的一个时间，同时 JVM 还针对当前 CPU 的负荷情况做了较多的优化：

1. 如果平均负载小于 CPUs 则一直自旋，
2. 如果有超过(CPUs/2) 个线程正在自旋，则后来线程直接阻塞，
3. 如果正在自旋的线程发现 Owner 发生了变化则延迟自旋时间（自旋计数）或进入阻塞，
4. 如果 CPU 处于节电模式则停止自旋，自旋时间的最坏情况是 CPU 的存储延迟（CPU A 存储了一个数据，到 CPU B 得知这个数据直接的时间差），
5. 自旋时会适当放弃线程优先级之间的差异。 

##### 自旋锁的开启

JDK1.6 中-XX:+UseSpinning 开启；  
-XX:PreBlockSpin=10 为自旋次数；  
JDK1.7 后，去掉此参数，由 jvm 控制； 

#### 重量级锁（Mutex Lock） 

1. 定义：依赖于操作系统 Mutex Lock 所实现的锁（线程之间的切换需要从用户态转换成核心态，成本非常高）
2. Synchronized 是通过对象内部的一个叫做监视器锁（monitor）来实现的
3. 监视器锁本质又是依赖于底层的操作系统的 Mutex Lock 来实现的

### 锁分离

#### 读写锁  

为了提高性能，Java 提供了读写锁，在读的地方使用读锁，在写的地方使用写锁，灵活控制，如果没有写锁的情况下，读是无阻塞的,在一定程度上提高了程序的执行效率。读写锁分为读锁和写锁，多个读锁不互斥，读锁与写锁互斥，这是由 jvm 自己控制的，你只要上好相应的锁即可。 

##### 读锁（并发读）

如果你的代码只读数据，可以很多人同时读，但不能同时写，那就上读锁 

##### 写锁（单独写）

如果你的代码修改数据，只能有一个人在写，且不能同时读取，那就上写锁。总之，读的时候上读锁，写的时候上写锁！ 

##### Java中的实现

1. Java中读写锁有个接口ReadWriteLock实现类ReentrantReadWriteLock；可以用来实现TreeMap的线程安全使用。
2. CopyOnWriteArrayList 、CopyOnWriteArraySet
3. CopyOnWrite容器即写时复制的容器。通俗的理解是当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器。这样做的好处是我们可以对CopyOnWrite容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以CopyOnWrite容器也是一种读写分离的思想，读和写不同的容器。CopyOnWrite并发容器用于读多写少的并发场景，因为，读的时候没有锁，但是对其进行更改的时候是会加锁的，否则会导致多个线程同时复制出多个副本，各自修改各自的；

### 锁粗化（减小锁的粒度）

减小锁粒度是指缩小锁定对象的范围，从而减小锁冲突的可能性，从而提高系统的并发能力。减小锁粒度是一种削弱多线程锁竞争的有效手段。

#### 分段锁-ConcurrentHashMap

java中的ConcurrentHashMap在jdk1.8之前的版本，使用一个Segment 数组

```java
Segment< K,V >[] segments
```

Segment继承自ReenTrantLock，所以每个Segment就是个可重入锁，每个Segment 有一个HashEntry< K,V >数组用来存放数据，put操作时，先确定往哪个Segment放数据，只需要锁定这个Segment，执行put，其它的Segment不会被锁定；所以数组中有多少个Segment就允许同一时刻多少个线程存放数据，这样增加了并发能力。

ConcurrentHashMap，它内部细分了若干个小的 HashMap，称之为段(Segment)。默认情况下一个 ConcurrentHashMap 被进一步细分为 16 个段，既就是锁的并发度。 
如果需要在 ConcurrentHashMap 中添加一个新的表项，并不是将整个 HashMap 加锁，而是首先根据hashcode得到该表项应该存放在哪个段中，然后对该段加锁，并完成put操作。在多线程环境中，如果多个线程同时进行put操作，只要被加入的表项不存放在同一个段中，则线程间可以做到真正的并行。 
ConcurrentHashMap 是由Segment 数组结构和HashEntry 数组结构组成 
ConcurrentHashMap 是由 Segment 数组结构和 HashEntry 数组结构组成。Segment 是一种可重入锁 ReentrantLock，在 ConcurrentHashMap 里扮演锁的角色，HashEntry 则用于存储键值对数据。一个 ConcurrentHashMap 里包含一个 Segment 数组，Segment 的结构和 HashMap 类似，是一种数组和链表结构， 一个 Segment 里包含一个 HashEntry 数组，每个 HashEntry 是一个链表结构的元素， 每个 Segment 守护一个 HashEntry 数组里的元素,当对 HashEntry 数组的数据进行修改时，必须首先获得它对应的 Segment 锁。 
<div align=center>

![1589108702724.png](..\images\1589108702724.png)

</div>

#### LongAdder

LongAdder 实现思路也类似ConcurrentHashMap，LongAdder有一个根据当前并发状况动态改变的Cell数组，Cell对象里面有一个long类型的value用来存储值; 
开始没有并发争用的时候或者是cells数组正在初始化的时候，会使用cas来将值累加到成员变量的base上，在并发争用的情况下，LongAdder会初始化cells数组，在Cell数组中选定一个Cell加锁，数组有多少个cell，就允许同时有多少线程进行修改，最后将数组中每个Cell中的value相加，在加上base的值，就是最终的值；cell数组还能根据当前线程争用情况进行扩容，初始长度为2，每次扩容会增长一倍，直到扩容到大于等于cpu数量就不再扩容，这也就是为什么LongAdder比cas和AtomicInteger效率要高的原因，后面两者都是volatile+cas实现的，他们的竞争维度是1，LongAdder的竞争维度为“Cell个数+1”为什么要+1？因为它还有一个base，如果竞争不到锁还会尝试将数值加到base上；

#### LinkedBlockingQueue

LinkedBlockingQueue也体现了这样的思想，在队列头入队，在队列尾出队，入队和出队使用不同的锁，相对于LinkedBlockingArray只有一个锁效率要高；
拆锁的粒度不能无限拆，最多可以将一个锁拆为当前cup数量个锁即可；

## 其他

### 同步锁与死锁 

#### 同步锁 

当多个线程同时访问同一个数据时，很容易出现问题。为了避免这种情况出现，我们要保证线程同步互斥，就是指并发执行的多个线程，在同一时间内只允许一个线程访问共享数据。 Java 中可以使用 synchronized 关键字来取得一个对象的同步锁。 

#### 死锁 

何为死锁，就是多个线程同时被阻塞，它们中的一个或者全部都在等待某个资源被释放。  

### 互斥同步

互斥同步（Mutual Exclusion & Synchronization）是一种最常见也是最主要的并发正确性保障手段。同步是指在多个线程并发访问共享数据时，保证共享数据在同一个时刻只被一条（或者是一些，当使用信号量的时候）线程使用。而互斥是实现同步的一种手段，临界区（CriticalSection）、互斥量（Mutex）和信号量（Semaphore）都是常见的互斥实现方式。因此在“互斥同步”这四个字里面，互斥是因，同步是果；互斥是方法，同步是目的。

## 参考

1. [Java锁 - 导读](https://www.jianshu.com/p/39628e1180a9)
2. [java 中的锁 -- 偏向锁、轻量级锁、自旋锁、重量级锁](https://blog.csdn.net/zqz_zqz/article/details/70233767)