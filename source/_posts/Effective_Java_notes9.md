---
title: 《Effective Java》学习笔记九——并发
categories:
    - '技术'
    - 'Java'
tags:
    - Java
    - 读书笔记
---

《Effective Java》学习笔记
<!--more-->





# 同步访问共享的可变数据

关键字synchronized可以保证同一时刻，只有一个线程可以执行某一个方法，或者某一个代码块。

Java语言规范保证度或者写一个变量是原子的，除非这个变量的类型为long或者double。

为了在线程之间进行可靠的通信，也为了互斥访问，同步是必要的。

当多个线程共享可变数据的时候，每个读或者写数据的线程都必须执行同步。如果没有同步，就无法保证一个线程所做的修改可以被另一个线程获知。未能同步共享可变数据会造成程序的活性失败和安全性失败。

将可变数据限制在单个线程中，尽量避免在线程间共享可变数据。



# 避免过度同步

依据情况的不同，过度同步可能会导致性能降低、死锁，甚至不确定的行为。

**为了避免活性失败和安全性失败，在一个被同步的方法或者代码块中，永远不要放弃对客户端的控制。**换句话说，在一个被同步的区域内部，不要调用设计成要被覆盖的方法，或者是由客户端以函数对象的形式提供的方法。从包含该同步区域的类的角度来看，这样的方法是外来的。这个类不知道该方法会做什么事情，也无法控制它。根据外来方法的作用，从同步区域中调用他会导致异常、死锁或者数据损坏。

假设当同步区域所保护的约束条件暂时无效时，你要从同步区域中调用一个外来方法。由于Java程序设计语言中的锁是可重入的，这种调用不会死锁。他会产生一个异常，因为调用线程已经有这个锁了，因此当该线程试图再次获得该锁时会成功，尽管概念上不相关的另一项操作正在该锁保护的数据上进行着。这种失败的后果可能是灾难的。从本质上来说，这个锁没有尽到它的职责。可再入的锁简化了多线程的面向对象程序的构造，但是他们可能会将活性失败变成安全性失败。

事实上，要将外来方法的调用移出同步的代码块。还有一种更好地方法。自从Java 1.5发行版本以来，Java类库就提供了一个并发集合，称作CopyOnWriteArrayLIst，这是专门为此定制的。这是ArrayList的一种变种，通过重新拷贝整个底层数组，在这里实现所有的写操作。由于内部数组永远不改动，因此迭代不需要锁定，速度也非常快。如果大量使用，CopyOnWriteArrayList的性能将大受影响，但是对于观察者列表来说却是很好的，因为他们几乎不改动，并且经常被遍历。

在同步区域之外被调用的外来方法被称作“开放调用”。除了可以避免死锁之外，开放调用还可以极大地增加并发性。外来方法的运行时间可能会任意长。如果在同步区域内调用外来方法，其他线程对受保护资源的访问就会遭到不必要的拒绝。

**通常，你应该在同步区域内做尽可能少的工作。**获得锁，检查共享数据，根据需要转换数据，然后放掉锁。如果你必须要执行某个很耗时的动作，则应该设法把这个动作移到同步区域的外面。

虽然自从Java平台早期以来，同步的成本已经下降了，但更重要的是，永远不要过度同步。在这个多核的时代，过度同步的实际成本并不是指获取锁所花费的CPU时间；而是指失去了并行的机会，以及因为需要确保每个核都有一个一致的内存视图而导致的延迟。过度同步的另一项潜在开销在于，他会限制VM优化代码执行的能力。

如果一个可变的类要并发使用，应该使这个类变成是线程安全的，通过内部同步，你还可以获得明显比从外部锁定整个对象更高的开发性。否则，就不要在内部同步。让客户在必要的时候从外部同步。在Java平台出现的早期，许多类都违背了这些指导方针。当你不确定的时候，就不要同步你的类，而是应该建立文档，注明它不是线程安全的。

如果你的内部同步了类，就可以使用不同的方法来实现高并发性，例如分拆锁、分离锁和非阻塞并发控制。

如果方法修改了静态域，那么你也必须同步对这个域的访问，即使他往往只用于单个线程。客户要在这种方法上执行外部同步时不可能的，因为不可能保证其他不想关的客户也会执行外部同步。

简而言之，为了避免死锁和数据破坏。千万不要从同步区域内部调用外来方法。更为一般的讲，要尽量限制同步区域内部的工作量。当你在设计一个可变类的时候，要考虑一下他们是否应该自己完成同步操作。在现在这个多核的时代，这比永远不要过度同步来得更重要。只有当你有足够的理由一定要在内部同步类的时候，才应该这么做，同时还应该将这个决定清楚地写到文档中。



# executor和task优先于线程

在Java 1.5 发行版本中，Java平台中增加了java.util.concurrent。这个包中包含了Executor Framework，这是一个很灵活的基于接口的任务执行工具。它创建了一个在各方面都比工作队列更好，却只需要这一行代码：

ExecutorService executor = Executors.newSingleThreadExecutor();

下面是为执行提交一个runnable的方法：

executor.execute(runnable);

下面是告诉executor如何优雅的终止（如果做不到这一点，虚拟机可能将不会退出）：

executor.shutdown()。

你可以利用executor service完成更多的事情。例如，可以等待完成一项特殊的任务（后台线程），你可以等待一个任务中任何任务或者所有任务完成（利用invokeAny或者invokeAll方法），你可以等待executor service优雅的完成终止（利用awaitTermination方法），你可以在任务完成时逐个地获取这些任务的结果（利用ExecutorCompletionService），等等。

如果想让不止一个线程来处理来自这个队列的请求，只要调用一个不同的静态工厂，这个工厂创建了一种不同的executor service，称作线程池。你可以用固定或者可变数目的线程创建一个线程池。java.util.concurrent.Executors类包含了静态工厂，能为你提供所需的大多数executor。然而，如果你想来点特别的，可以直接使用ThreadPoolExecutor类。这个类允许你控制线程池操作的几乎每个方面。

为特殊的应用程序选择executor service是很有技巧的。如果编写的是小程序，或者轻载的服务器，使用Executors.newCachedThreadPool通常是个不错的选择，因为它不需要配置，并且一般情况下能够正确的完成工作。但是对于大负载的服务器来说，缓存的线程池就不是很好的选择了！在缓存的线程池中，被提交的任务没有排成队列，而是直接交给线程执行。如果没有线程可用，就创建一个新的线程。如果服务器负载的太重，以致它所有的CPU都完全被占用了，当有更多的任务时，就会创建更多的线程，这样只会使情况变得更糟。因此，在大负载的产品服务器中，最好使用Executors.newFixedThreadPool，它为你提供了一个包含固定线程数目的线程池，或者为了最大限度的控制它，就直接使用ThreadPoolExecutor类。

你不仅应该尽量不要编写自己的工作队列，而且还应该尽量不直接使用线程。现在关键的抽象不再是Thread了，它以前可是既充当工作单元，又是执行机制。现在工作单元和执行机制是分开的。现在关键的抽象是工作单元，称作任务（task）。任务有两种：Runnable及其近亲Callable（它与Runnable域类似，但它会返回值）。执行任务的通用机制是executor service。如果从任务的角度来看问题，并让一个executor service替你执行任务，在选择适当地执行策略方面就获得了极大地灵活性。从本质上讲Executor Famework所做的工作是执行，犹如Collections Framework所做的工作是聚集（aggregation）一样。

Executor Framework也有一个可以代替java.util.Timer的东西，即ScheduledThreadPoolExecutor。虽然timer使用起来更加容易，但是被调度的线程池executor更加灵活。timer只用一个线程来执行任务，这在面对长期运行的任务时，会影响到定时的准确性。如果timer唯一的线程抛出未被捕获的异常，timer就会停止执行。被调度的线程池executor支持多个线程，并且优雅的从抛出未受检异常的任务中恢复。



# 并发工具优先于wait和notify

自从Java 1.5 发行版本开始，Java平台就提供了更高级的并发工具，他们可以完成以前必须在wait和notify上手写代码来完成的各项工作。既然正确的使用wait和notify比较困难，就应该用更高级的并发工具来代替。

java.util.concurrent中更高级的工具分成三类：Executor Framework、并发集合（Concurrent Collection）以及同步器（Synchronizer）。

并发集合为标准的集合接口（如List、Queue和Map）提供了高性能的并发实现。为了提供高并发性，这些实现在内部自己管理同步。因此，**并发集合中不可能排除并发活动；将它锁定没有什么作用，只会使程序的速度变慢。**

这意味着客户无法原子的对并发集合进行方法调用。因此有些集合接口已经通过依赖状态的修改操作进行了扩展，它将几个基于操作合并到了单个原子操作中。例如，ConcurrentMap扩展了Map接口，并添加了几个方法，包括putIfAbsent(key, value)，当键没有映射时会替她插入一个映射，并返回与键关联的前一个值，如果没有这样的值，则返回null。

ConcurrentHashMap除了提供卓越的并发性之外，速度也非常快。除非不得已，**否则应该优先使用ConcurrentHashMap，而不是使用Collections.synchronizedMap或者Hashtable。**只要用并发Map替换老式的同步Map，就可以极大地提升并发应用程序的性能。更一般的，应该优先使用并发集合，而不是使用外部同步的集合。

有些集合接口已经通过阻塞操作进行了扩展，他们会一直等待（或者阻塞）到可以成功执行为止。例如，BlockingQueue扩展了Queue接口，并添加了包括take在内的几个方法，它从队列中删除并返回了头元素，如果队列为空，就等待。这样就允许将阻塞队列用于工作队列，也称作生产者-消费者队列，一个或者多个生产者线程在工作队列中添加工作项目，并且当工作项目可用时，一个或者多个消费者线程，则从工作队列中取出队列并处理工作项目。不出所料，大多数ExecutorService实现（包括ThreadPoolExecutor）都使用BlockingQueue。

同步器是一些使线程能够等待另一个线程的对象，允许他们协调动作。最常用的同步器是CountDownLatch和Semaphore。较不常用的是CyclicBarrier和Exchanger。

倒计数锁存器（CountDownLatch）是一次性的障碍，允许一个或者多个线程等待一个或者多个其他线程来做某些事情。CountDownLatch的唯一构造器带有一个int类型的参数，这个int参数是指允许所有在等待的线程被处理之前，必须在锁存器上调用countDown方法的次数。

要在这个简单的基本类型之上构建一些有用的东西，做起来是相当的容易。例如，假设想要构建一个简单地框架，用来给一个工作的并发执行定时。这个框架中包含单个方法，这个方法带有一个执行该动作的executor，一个并发级别（表示要并发执行该动作的次数），以及表示该动作的runnable。所有的工作线程（worker thread）自身都准备好，要在timer线程启动时钟之前运行该动作（为了实现准确地定时，这是必须的）。当最后一个工作线程准备好运行该动作时，timer线程就“发起头炮”，同时允许工作线程执行该动作。一旦最后一个工作线程执行完成该动作，timer线程就立即停止计时。

还有一些细节值得注意。传递给timer方法的executor必须允许创建至少与指定并发级别一样多的线程，否则就永远不会结束。这就是线程饥饿死锁。如果工作线程捕捉到InterruptedException，就会利用习惯用法Thread.currentThread().interrupt()重新断言中断，并从他的run方法中返回。这样就允许executor在必要的时候处理中断，事实上也理当如此。最后，**对于间歇式的定时，始终应该优先使用System.nanoTime，而不是使用System.currentTimeMills。**System.nanoTIme更加准确也更加精确，它不受系统时钟的调整所影响。

虽然你时钟应该优先使用并发工具，而不是使用wait和notify，但可能必须维护使用了wait和notify的遗留代码。wait方法被用来使线程等待某个条件，它必须在同步区域内部被调用，这个同步区域将对象锁定在了调用wait方法的对象上。

**始终应该使用wait循环模式来调用wait方法；永远不要在循环之外调用wait方法。循**环会在等待之前和之后测试条件。

在等待之前测试条件，当条件已经成立时就跳过等待，这对于确保活性是必要的。如果条件已经成立，并且在线程等待之前，notify（或者notifyAll）方法已经被调用，则无法保证该线程将会从等待中苏醒过来。

在等待之后测试条件，如果 条件不成立的话继续等待，这对于确保安全性是必要的。当条件不成立的时候，如果线程继续执行，则可能会破坏被锁保护的约束关系。当条件不成立时，有下面一些理由可使一个线程苏醒过来：

- 另一个线程可能已经得到了锁，并且从一个线程调用notify那一刻起，到等待线程苏醒过来的这段时间中，得到锁的线程已经改变了受保护的状态。
- 条件并不成立，但是另一个线程可能意外的或恶意的调用了notify。在公有可访问的对象上等待，这些类实际上把自己暴露在了这种危险的境地中。公有可访问对象的同步方法中包含的wait都会出现这样的问题。
- 通知线程在唤醒等待线程时可能会过度“大方”。例如，即使只有某一些等待线程的条件已经被满足，但是通知线程可能仍然调用notifyAll。
- 在没有通知的情况下，等待线程也可能（但很少）会苏醒过来。这被称为“伪唤醒”。

一个相关的话题是，为了唤醒正在等待的线程，你应该使用notfiy还是notifyAll。一种常见的说法是，你总是应该使用notifyAll。这是合理而保守的建议。它总会产生正确的结果，因为它可以保证你将会唤醒所有需要被唤醒的线程你可能也会唤醒其他一些线程，但是这不会影响程序的正确性，这些线程醒来之后，会检查他们正在等待的条件如果发现条件并不满足，就会继续等待。

从优化的角度来看，如果处于等待状态的所有线程都在等待同一个条件，而每次只有一个线程可以从这个条件中被唤醒，那么你应该选择调用notify，而不是notifyAll。

即使这些条件都是真的，也许还是有理由使用notifyAll而不是notify。就好像把wait调用放在一个循环中，以避免在公有可访问对象上的意外或恶意的通知一样，与此类似，使用notifyAll代替notify可以避免来自不想关线程的意外或恶意的等待。否则，这样的等待会“吞掉”一个关键的通知，使真正的接收线程无限的等待下去。

简而言之，直接使用wait和notify就像用“并发汇编语言”进行编程一样，而java.util.concurrent则提供了更高级的语言。**没有理由在新代码中使用wait和notify，即使有、也是极少的。**如果你在维护使用wait和notify的代码，务必确保始终是利用标准的模式从while循环内部调用wait。一般情况下，你应该优先使用notifyAll，而不是使用notify。如果使用notify，请一定要小心，以确保程序的活性。



# 线程安全性的文档化

当一个了IDE实例或者静态方法被并发使用的时候，这个类的行为如何，是该类与其客户端程序建立的约定的重要组成部分。如果你没有在一个类的文档中描述其行为的并发性情况，使用这个类的程序员将不得不做出某些假设。如果这些假设是错误的，这样得到的程序就可能缺少足够的同步，或者过度同步。无论属于这其中的哪种情况，都可能会发生严重的错误。

你可能听过这样的说法：通过查看文档中是否出现synchronized修饰符，可以确定一个方法是否是线程安全的。这种说法从几个方面来说都是错误的。在正常的操作中,Javadoc并没有在它的输出中包含synchronized修饰符，这是有理由的。**因为在一个方法声明中出现synchronized修饰符，这是个实现细节，并不是导出的API的一部分。她并不一定表明这个方法是线程安全的。**

而且，“出现了synchronized关键字就足以用文档说明线程安全性”的这种说法隐含了一个错误的观念，即认为线程安全性是一种“要么全有要么全无”的属性。实际上，线程安全性有多种级别。**一个类为了可被多个线程安全的使用，必须在文档中清楚地说明他所支持的线程安全性级别。**

下面的列表概括了线程安全性的几种级别。这份列表并没有涵盖所有的可能，而只是些常见的情形：

- **不可变的（immutable）**——这个类的实例视不变的。所以，不需要外部的同步。这样的例子包括String、Long和BigInteger。
- **无条件的线程安全（unconditionally thread-safe）**——这个类的实例是可变的，但是这个类有足够的内部同步，所以，它的实例可以被并发使用，无需任何外部同步。其例子包括Random和ConcurrentHashMap。
- **有条件的线程安全（conditionally thread-safe）**——除了有些方法为进行安全的并发使用而需要外部同步之外，这种线程安全级别与无条件的线程安全相同。这样的例子包括Collections.synchronized包装返回的集合，他们的迭代器要求外部同步。
- **非线程安全（not thread-safe）**——这个类的实例是可变的。为了并发的使用他们，客户必须利用自己选择的外部同步包围每个方法调用（或者调用序列）。这样的例子包括通用的集合实现，例如ArrayList和HashMap。
- **线程对立的（thread-hostile）**——这个类不能安全的被多个线程并发使用，即使所有方法调用都被外部同步包围。线程对立的根源通常在于，没有同步的修改静态数据。没有人会有意编写一个线程对立的类；这种类是因为没有考虑到并发性而产生的后果。幸运的是，在Java平台类库中，线程对立的类或方法非常少。System.runFinalizersOnExit方法是线程对立的，但已经被废除了。

在文档中描述一个有条件的线程安全类要特别小心。你必须指明哪个调用序列需要外部同步，还要指明为了执行这些序列，必须获得哪一把锁（极少的情况下是指哪几把锁）。通常情况下，这是指作用在实例自身上的那把锁。但也有例外。如果一个对象代表了另一个对象的一个视图（view），客户通常就必须在后台对象上同步，以防止其他线程直接修改后台对象。

类的线程安全说明通常放在他的文档注释中，但是带有特殊线程安全属性的方法则应该在他们自己的文档注释中说明他们的属性。没有必要说明枚举类型的不可变性。除非从返回类型来看已经很明显，否则静态工厂必须在文档中说明被返回对象的线程安全性。

当一个类承诺了“使用一个公有可访问的所对象”时，就意味着允许客户端以原子的方式执行一个方法调用序列，但是，这种灵活性是要付出代价的。并发集合使用的那种并发控制，并不能与高性能的内部并发控制相兼容。客户端还可以发起拒绝服务攻击，它只需超时的保持公有可访问锁即可。这有可能是无意的，也可能是有意的。

为了避免这种拒绝服务攻击，应该使用一个私有锁对象来代替同步的方法（隐含着一个公有可访问锁）。

重申一下，私有锁对象模式只能用在无条件的线程安全类上。有条件的线程安全类不能使用这种模式，因为他们必须在文档中说明：在执行某些方法调用序列时，他们的客户端程序必须获得哪把锁。

私有锁对象模式特别适用于那些专门为继承而设计的类。如果这种类使用它的实例作为锁对象，子类可能很容易在无意中方案基类的操作，反之亦然。出于不同的目的而使用相同的锁，子类和基类可能会“相互绊住对方的脚”。这不只是一个理论意义上的问题。

简而言之，每个类都应该利用字斟句酌的说明或者线程安全注解，清楚地在文档中说明他的线程安全属性。sychronized修饰符与这个文档毫无关系。有条件的线程安全类必须在文档中指明“哪个方法调用序列需要外部同步，以及在执行这些序列的时候要获得哪把锁”。如果你编写的是无条件的线程安全类，就应该考虑使用私有锁对象来代替同步的方法。这样可以防止客户端程序和子类的不同步干扰，让你能够在后续的版本中灵活的对并发控制采用更加复杂的方法。



# 慎用延迟初始化

延迟初始化是延迟到需要域的值时才将它初始化的这种行为。如果永远不需要这个值，这个域就永远不会被初始化。这种方法既适用于静态域，也适用于实例域。虽然延迟初始化主要是一种优化，但它也可以用来打破类和实例初始化中的有害循环。

就像大多数的优化一样，对于延迟初始化，最好建议“除非绝对必要，否则就不要这么做”。延迟初始化就像一把双刃剑。它降低了初始化类或者创建实例的开销，却增加了访问被延迟初始化的域的开销。根据延迟初始化的域最终需要初始化的比例、初始化这些域要多少开销，以及每个域多久被访问一次，延迟初始化实际上降低了性能。

也就是说，延迟初始化有它的好处。如果域只在类的实例部分被访问，并且初始化这个域的开销很高，可能就值得进行延迟初始化。要确定这一点，唯一的办法就是测量类在用和不同延迟初始化时的性能差异。

当有多个线程时，延迟初始化是需要技巧的。如果两个或者多个线程共享一个延迟初始化的域，采用某种形式的同步是很重要的，否则就可能造成严重的Bug。

**在大多数情况下，正常的初始化要优先于延迟初始化。**

**如果利用延迟优化来破坏初始化的循环，就要使用同步访问方法，因为它是最简单、最清楚地替代方法。**

这两种习惯模式（正常的初始化和使用了同步访问方法的延迟初始化）应用到静态域上时保持不变，除了给域和访问方法声明添加了static修饰符之外。

**如果出于性能的考虑而需要对静态域使用延迟初始化，就使用lazy initialization holder class模式。这种模式保证了类要被用到的时候才会被初始化。**

现代的VM讲在初始化该类的时候，同步域的访问。一旦这个类被初始化，VM将修补代码，以便后续对该域的访问不会导致任何测试或者同步。

**如果出于性能的考虑而需要对实例域使用延迟初始化，就是用双重检查模式。**这种模式避免了在域被初始化之后访问这个域时的锁定开销。这种模式背后的思想是：两次检查域的值，第一次检查时没有锁定，看看这个域是否被初始化了；第二次检查时有锁定。只有当第二次检查时表明这个域没有被初始化，才会调用computeFieldValue方法对这个域进行初始化。因为如果域已经被初始化就不会有锁定，域被声明为volatile很重要。

双重检查模式的两个变量值值得一提。有时候，你可能需要延迟初始化一个可以接受重复初始化的实例域。如果处于这种情况，就可以使用双重检查惯用法的一个变形，它省去了第二次检查。没错，它就是单重检查模式。

当双重检查模式或者单重检查模式应用到数值型的基本类型域时，就会用0来检查这个域（这是数值型基本变量的默认值），而不是用null。

如果你不在意是否每个线程都重新计算域的值，并且域的类型为基本类型，而不是long或者double类型，就可以选择从单重检查模式的域声明中删除volatile修饰符。这种变体称之为racy single-check idiom。它加快了某些架构上的域访问，代价是增加了额外的初始化（直到访问）。该域的每个线程都进行一次初始化这显然是一种特殊的方法，不适合于日常的使用。然而，String实例却用它来缓存他们的散列码。

简而言之，大多数的域应该正常进行初始化，而不是延迟初始化。如果为了达到性能目标，或者为了破坏有害的初始化循环，而必须延迟初始化一个域，就可以使用相应的延迟初始化方法。对于实例域，就是用双重检查模式；对于静态域，则使用lazy initialization holder class。对于可以接受重复初始化的实例域，也可以考虑使用单重检查模式。



# 不要依赖于线程调度器

当有多个线程可以运行时，由线程调度器决定哪些线程将会运行，以及运行多长时间。任何一个合理的操作系统在做出这样的决定时，都会努力做到公正，但是所采用的策略却大相径庭。因此，编写良好的程序不应该依赖于这种策略的细节。**任何依赖于线程调度器来达到正确性或者性能要求的程序，很有可能都是不可移植的。**

要编写健壮的、响应良好的、可移植的多线程应用程序，最好的办法是确保可运行线程的平均数量不明显多余处理器的数量。这使得线程调度器没有更多的选择：它只需要运行这些可运行的线程，直到他们不再可运行为止。即使在根本不同的线程调度算法下，这些程序的行为也不会有很大的变化。注意可运行线程的数量并不等于线程的总数量，前者可能更多。在等待的线程并不是可运行的。

保持可运行线程数量尽可能少的主要方法是，让每个线程做些有意义的工作，然后等待更多有意义的工作**。如果线程没有在做有意义的工作，就不应该运行。**根据Executor Framework，这意味着适当地规定了线程池的大小，并且使任务保持适当地小，彼此独立。任务不应该太小，否则分配的开销也会影响到性能。

线程不应该一直处于忙-等的状态，即反复的检查一个共享对象，以等待某些事情发生。除了使程序易受到调度器的变化影响之外，忙-等这种做法也会极大地增加处理器的负担，降低了同一机器上其他进程上其他进程可以完成的有用工作量。

如果某一个程序不能工作，是因为某些线程无法像其他线程那样获得足够的CPU时间，那么，**不要企图通过调用Thead.yield来“修正”该程序。**你可能好不容易成功让程序能够工作，但这样得到的程序仍然是不可移植的。同一个yield调用在一个JVM实现上能提高性能，而在另一个JVM实现上却有可能会更差，在第三个JVM实现上则可能没有影响。Thread.yield没有可测试的语义。更好地解决办法是重新构造应用程序，以减少可并发运行的线程数量。

有一种相关的方法是调整线程优先级，同样有类似的警告**。线程优先级是Java平台上最不可移植的特征了。**通过调整某些线程的优先级来改善应用程序的响应能力，这样做并非不合理，却是不必要的，也是不可移植的。通过调整线程的优先级来解决严重的活性问题是不合理的。在你找到并修正底层的真正原因之前，这个问题可能会再次出现。

应该使用Thread.sleep(1)代替Thead.yield来进行并发测试。千万不要使用Thread.sleep(0)，它会立即返回。

简而言之，不要让应用程序的正确性依赖于线程调度器。否则，结果得到的应用程序将既不健壮，也不具有可移植性。作为推论，不要依赖Thread.yield或者线程优先级。这些设施仅仅对调度器做些暗示。线程优先级可以用来提高一个已经能够正常工作的程序的服务质量，但永远不应该用来“修正”一个原本并不能工作的程序。



# 避免使用线程组

除了线程、锁和监视器之外，线程系统还提供了一个基本的抽象，即线程组。线程组的初衷是作为一种隔离applet（小程序）的机制，当然是出于安全的考虑。但是他们从来没有真正履行这个承诺，他们的安全价值已经差到根本不在Java安全模型的标准工作中提及的地步。

他们允许你同时把Thread的某些基本功能应用到一组线程中。其中有一些基本功能已经被废弃了，剩下的也很少使用。

具有讽刺意味的是，从线程安全性的角度来看，ThreadGroup API非常弱。为了得到一个线程组中的活动线程列表，你必须调用enumerate方法，它有一个数组参数，并且数组的容量必须足够大，以便容纳所有的活动线程。activeCount方法返回一个线程组中活动线程的数量，但是，一旦这个数组进行了分配，并传递给了enumerate方法，就不保证原先得到的活动线程数仍是正确的。如果线程数增加了，而数组太小，enumerate方法就会悄然的忽略掉无法再数组中容纳的线程。

列出线程组汇总子组的API也有类似的缺陷。虽然通过增加新的方法，这些问题有可能得到修正，但是他们目前还没有修正，因为线程组已经过时了，所以实际上根本没有必要修正。

总而言之，线程组并没有提供太多有用的功能，而且他们提供的许多功能还是有缺陷的。我们最好把线程组看做是一个不成功的试验，你可以忽略掉他们。如果你正在设计的一个类需要处理线程的逻辑组，或许就应该使用线程池executor。



