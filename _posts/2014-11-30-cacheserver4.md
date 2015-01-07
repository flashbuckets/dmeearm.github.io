---
layout: post
title: 缓存服务器设计与实现(四) 
tags: [dev]
---

这里我们聚焦一个问题，就是缓存满的情况。一般的cache都会配置容量，无论是内存缓存还是磁盘缓存，都不能无节制的去使用他们。这里以磁盘缓存为例，如果配置的限额已用完，该如何处理呢？

对于nginx，如果你开启了cache功能，那么你通过ps命令看到这样的进程：**cache manager process**。其实这个进程的作用主要是在文件失效或者磁盘空间不足的时候，删除对象。那么怎么删，什么标准？

由于每个缓存对象都有一个最长有效期，过了这个时间对象就认为无效了，需要删掉。在nginx的缓存系统中有个lru队列，里面的元素其实是对象在rbtree中的一个node。每次hit一个对象，对应的node就从队列中脱离，然后重新插入到队列头部，这样最近被访问的node就被移到队列开头，冷的对象则逐渐推向队尾，一个最基本的LRU思想(当然这样的处理存在公认的问题，即不能准确判断一个文件的冷热程度，在队尾的也不一定就是冷的)。而这个cache manager进程就是从队尾开始，查看是否有失效的文件。很显然，如果队尾的节点没有失效，那么整个队列上也就都没有问题。后面nginx就会设置定时器，过一段时间再来做检查。但是如果此时缓存满了，又没有文件失效，那么就要做lru来删一些冷的文件，这个也是没有办法的事情。在删的时候，也是从lru队列的尾部开始，如果当前文件已经没有请求在使用，那么就可以直接删了。如果还有请求在用，那么它就从后往前找下一个可以删的对象。一般它尝试找20次，如果找20次还没有找到一个可删的目标，那就先歇一会再处理。删到什么程度结束呢？一般删掉到磁盘使用量低于配额就可。当然你可以指定一些标准，例如删掉总量的10%等等。还有一点要说明一下：因为每个缓存对象都要对应一个内存结构，这些空间来自共享内存，在内存不足的时候分配可能会失败，那么这时也需要采取一些挽救措施。nginx的处理就是去做lru，虽然lru是来删文件，显然在删的同时，内存中的控制结构也会被释放，这样就可以起到缓解内存紧张的局面。

这块的东西基本就是这样，剩下的就是一些代码细节，有兴趣的朋友可以去翻看nginx代码。

来看第二个问题：一个缓存服务器异常停机或者要进行维护关机了，当机器再次启动缓存服务的时候，会遇到一个令人难以接受的状况。因为你之前的机器跑了很久，已经缓存了成千上万的文件，体积高达数百G甚至上T。如果现在这个机器要提供服务，那么会由于之前的缓存不可用(其实是不可见)，而导致大量的回源请求。要是把客户源站给打死了，就等着客户找你算账吧。

所以关键的问题，就是如何重建之前的缓存。其实文件都还在磁盘上，缺失的只是内存中的控制信息，没有这些信息就不能打通整个缓存系统跟实际磁盘文件的联系。在nginx里面，重建缓存这项艰巨的任务由一个叫“cache loader process”的进程来完成，同样的通过ps指令就可以看到。其实原理很简单，我们先看以前文章中的那个例子：
>/cache/0/8d/8ef9229f02c5672c747dc7a324d658d0

cache loader process在执行时会去缓存目录下遍历所有的子目录和文件，对于文件需要做一些额外的验证，判读该文件是否符合一个正常缓存文件的特征，例如文件名是否是32字符(例子中的hash串表示的文件名)，文件最少长度(因为在nginx里面，缓存文件开头总是含有特定大小的控制信息)等。对于nginx认为的非合法文件，nginx将会予以删除。假设nginx发现了一个合法的文件，那么就需要将它注册到缓存系统中，具体的动作包括申请结构，加到rbtree和lru队列中等。特别是在插入rbtree的时候，文件名正是原来缓存中用来插入和查找的关键字，所以很方便的就可以完成插入。当cache loader process完成了重建缓存的任务后，就可以exit，因为它已经没有什么存在的价值了。

这里还有一个有意思的地方：如果文件比较多，重建过程可能会很慢，为此nginx使用了三个个变量来控制速度，loader_threshold，loader_files和loader_sleep。如果loader进程持续工作时间达到loader_threshold，那么就去歇一会。如果重建的文件数量到达loader_files，作为奖励，也可以去歇一会，歇息的时间为loader_sleep。这三个变量都是可以通过配置指定的。

**这里有个重要的事情要强调一下:**
>在loader进程重建缓存的时候，其实nginx的worker也在同时提供服务，那么这个loader是否是会影响正常的服务？其实这个loader进程与普通的worker都是由master进程通过fork派生的，所以两者都可以看到共享内存中的一切，由于整个系统的缓存信息都在共享内存中，两者对共享资源的访问就会存在互斥。loader进程中，将一个对象插入到缓存的过程跟worker中是一样的，都是先抢锁，然后查找，没有找到再插入。所以一个对象到底是由谁插入到缓存系统中，完全靠loader和worker公平竞争锁。而那个竞争中的失败者，当它后面拿到锁的时候，就发现让人捷足先登了，那么就跳过插入的工作了。反过来也是一样。

总的看来，nginx在很多机制上处理都非常的简单明了，并且继承了一直以来的高效。虽然站在一个专业cache的角度看，它很多功能还不完善，但是依然有很多值得我们参考和借鉴的东西。如果你想设计和实现一个自己cache，又对功能要求不是那么高，那去参考nginx的这个模型倒是一个不错的选择。

关于cache的设计，还有一个重要的机制就是过期控制。这个至关重要，大家如果对cache有个大概的认识，稍微想一下就能体会它的重要性。这块后面再拿出篇幅来讨论吧。