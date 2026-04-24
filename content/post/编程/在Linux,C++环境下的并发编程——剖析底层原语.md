---
title: "在Linux/C++环境下的并发编程——剖析底层原语"
categories:
    - "编程"
tags:
    - "并发编程"
    - "C++"
    - "Linux"
date: 2026-04-20T10:05:38+08:00
comments: false
draft: false
build:
    list: always    # Change to "never" to hide the page from the list
---

### 前言

本系列将以Linux/C++的视角讲解并发编程，文章将注重底层实现与编程方式，如遇到不了解的API或者概念等烦请自行查阅资料。



## 一、为什么会有并发冲突？

### 1.1 基本概念

我不是很擅长说文解字，所以在此仅仅是罗列概念，如果你想要对于这些概念更清晰生动的讲解，请自行查阅其他资料。

**共享资源**：多个进程/线程可以共同访问和操作的资源，例如全局变量、文件、内存中的一块数据等等。

**竞态条件**：当两个或多个线程（或进程）访问同一个共享资源，并且最终的执行结果取决于这些线程（或进程）执行的时序（即它们执行的精确顺序）时，就发生了**竞态条件**。

**临界区**：为了避免竞态条件，我们需要识别出代码中访问共享资源的部分。这部分代码就成为**临界区**。

**互斥**：解决竞态问题的核心思想是，确保在任何时刻，最多只有一个线程（或进程）能够进入临界区执行。这种机制就叫做**互斥**。

**同步**：线程间的执行需要遵循一定的先后顺序。一个线程的执行可能依赖于另一个线程产生的某个结果。**同步**就是用于实现这个的。

---

### 1.2 执行局限——语义的非原子性

试想一行简单的C++代码。

```c++
a += 1;
```

无论你熟不熟悉汇编，你只需要知道这条语句**一般**会被编译成**三条汇编指令**，而汇编指令会一一映射为硬件操作，同时我们需要知道的是，**CPU在执行指令的时候仅保证硬件操作的原子性**，以及**操作系统有时会切换CPU的执行上下文**。

那么问题就来了，如果一个CPU同时执行两段代码，然后在执行先前所讲的三条指令过程的中间态时（比如只执行了前一条或者前两条，但无论如何现在的 a 都会处于混乱的状态），碰巧CPU切换到另一段代码，碰巧这段代码将操作同一个内存地址的 a ，不难推演，此时 a 的错误很有可能引发一系列的错误。

事实上，针对共享资源，我们很多情况都不会仅仅是`a += 1`这样简单的代码，而是更加复杂的处理，那么就更容易产生并发冲突了，这时，保证对于共享资源操作的原子性就尤为重要。

甚至，以上还没有针对多CPU的并发进行探讨。那将更加复杂。



## 二、内存序

### 2.1 前提——硬件的原子指令

首先我们要明确一点，纯软件很难写出（或者，即使写出成本也非常高）硬件通用的互斥实现（不仅是前面所讲的执行局限，还有CPU的内存序问题，后续会讲解）。所以，我们把这个重任交给硬件，让硬件提供原子指令（比如 `TSL`和 `XCHG`）。当然，原子操作可以设计非常多，我们一般仅要求硬件提供基础的原子操作即可，也就是RISC。

接着，就让我们从底层实现一步步到达上层的原子接口。

---

### 2.2 CPU的缓存一致性

在多核 CPU 中，每个核心都有自己的 L1/L2 缓存，只有 L3 缓存是共享的，那么为了保证缓存一致性，一般都会使用`MESI`或类似的协议来保证。

简单说下`MESI`协议，就是把缓存行分成四种状态：

1. **M (Modified)**：表示这份数据只存在于当前这一核心的 Cache 中且**已经被修改**了（也就是”脏“了），和主内存中的数据是不一致的。这意味着当前核心对这份数据是独占写入的，且之后应该把修改后的数据写回主内存。
2. **E (Exclusive)**：表示这份数据只存在于当前核心的 Cache 中，但**还没有被修改**，与主内存一致。不过，当前是独占，随时可以进入 M 状态而不用通知其他核心。
3. **S (Shared)**：表示这份数据可能同时存在于**多个核心**的 Cache 中，并且它没有被修改过，与主内存一致。那么，所有持有者都只有读取权。同时，任何一个持有者想要写入这份数据的时候，都要先广播，想要修改其他核心该缓存行的状态为 I，然后自己才能进入 M 状态修改。
4. **I (Invalid)**：表示这份数据是**无效且不可信**的，那么任何对当前缓存行的读写**都会触发 Cache Miss** 并强制从主内存或者其他核心的 Cache 中加载。

这样，当读写操作涉及到修改其他核心缓存行状态的时候，CPU 就会进行广播来保证缓存一致性。

但是，目前广播的时候**必须等待其他核心的 Ack** 。为了极致的响应性能，CPU又引入了 `Store Buffer` 和 `Invalidate Queue`，同时，把其带来的内存一致性的问题，交给了软件。

---

### 2.3 内存一致性与内存序

我们先来讲解前面提到的两个 CPU 的新组件以及其带来的问题。

1. **Store Buffer**：当执行写入（Store）操作时，CPU 将不再等待其他核心的 Ack，而是**直接把数据放进 Store Buffer，然后执行下一条指令。**
   + 导致的问题：**Store - Load 重排**。因为当前写入对其他核心**不可见**，那么其他核心紧接着读取（Load）的时候就很可能读到旧值。此时我们需要使用 `seq_cst` 内存序来避免该问题。

2. **Invalidate Queue**：当 Core B 收到 Core A 的 Invalidate 消息时，如果它正忙，它不会立刻去清空自己的缓存，而是**把消息塞进 Invalidate Queue，然后立刻回复 Ack**。
   + 导致的问题：**Load-Load / Load-Store 重排**。Core B 承诺了会清空缓存，但还没清空。如果 Core B 此时执行读取（Load），它会读到自己缓存里的**过期脏数据**。

#### 关于 x86 和 ARM

从计算机的发展历史来讲，相关领域的专家总是喜欢独揽任务而不信任其他领域。硬件总希望处理很多脏活累活，同理操作系统和系统软件也是。只有业务程序员才算是享受相对完美轻松的抽象层。

那么，对 x86 来说，硬件带来的问题自然不想暴露给上层，所以 x86 硬件工程师们承诺了 TSO(强内存模型)，x86 在内部引入了复杂的总线监听和流水线回滚来保证了**绝对不会发生非 Store - Load 重排**（你可能会质疑为什么不把 Store - Load 重排问题也包办了，那就要考虑到成本了）。

同时，ARM 就不一样了，如果说 x86 是保姆（通过硬件层面添加晶体管和功耗来兜底内存序），那么 ARM 就是信任程序员的战友，这份信任很沉重，ARM 只管往前冲（快速），而你要看住他的后背（保证内存同步）。如果搭配好的话，收益还是不错的。

总的来说，x86 在单核下既简单又强悍（单核下很多不必要的同步开销硬件会自动省略，但是 ARM 所依赖的软件指令依然会导致开销，毕竟你不可能只在单核运行，对吗？），而 ARM 则省下了晶体管和电费，用省出的芯片面积优化其他功能。

那么，如果你是企业，你会挑选哪种 CPU 呢？我想你或许已经知道 ARM 架构的火热，但如果想起，不妨感谢一下程序员为了保证 ARM 的安全所消耗的头发。



#### **内存序**

我们借用C++标准的内存序来一一讲解：

**x86架构**

1. **memory_order_relaxed**: 编译成普通的 MOV 指令。

2. **memory_order_acquire**: 编译成普通的 MOV 指令。

3. **memory_order_release**: 编译成普通的 MOV 指令。

4. **memory_order_seq_cst**: 编译成 LOCK XCHG 或者普通的 MOV 加上一条 **MFENCE (内存屏障)** 指令。
   + MFENCE 的作用是：**锁死当前 CPU，强制把 Store Buffer 里的所有数据排空（Flush）到 L1 缓存中，让全局可见后，才允许执行下一条指令。**

**ARM架构（ARMv8）**

1. **memory_order_relaxed**: 编译成普通的 LDR (Load) / STR (Store)。没有任何重排限制，除了地址依赖（比如读指针再读指针指向的值，实际上C++曾经有提出consume内存序来表示仅地址依赖的情况，但是过于复杂，我建议你忘掉它的存在）。
   - STR：CPU 把地址和数据扔进 Store Buffer，然后立刻执行下一条指令。什么时候进 L1 缓存？随缘（取决于 Store Buffer 拥挤程度和 MESI 协议的响应速度）。
   - LDR：CPU 先查 Store Buffer，再查 L1 缓存。如果有 Invalidate 消息躺在 Invalidate Queue 里还没处理，它会毫无顾忌地读出 L1 缓存里的过期脏数据。

2. **memory_order_acquire**: 编译成特殊的 **LDAR (Load-Acquire)** 指令。
   + 在 LDAR 之后的任何读写指令，**绝对不允许**在流水线中越过 LDAR 提前执行。
   + CPU 会确保在执行后续指令前，当前核心的 Invalidate Queue 中与后续读取相关的失效消息被处理，从而保证后续读取能拿到最新值。

3. **memory_order_release**: 编译成特殊的 **STLR (Store-Release)** 指令。
   + 在 STLR 之前的任何读写指令，**必须**在 STLR 真正执行（即数据准备排入 L1 缓存）之前（注意并不代表立即刷新到L1缓存，仅保证前后顺序），全部完成并达到全局可见。
   + CPU 会等待前面的所有 Store 操作都拿到缓存行的独占权（收到所有核心的 Ack），然后再把 STLR 的数据排入 L1 缓存

4. **memory_order_seq_cst**: 编译成 LDAR / STLR，并且在某些情况下，编译器会插入极其沉重的 **DMB ISH (Data Memory Barrier)** 指令，**强制刷新Store Buffer并检查Invalidate Queue**。
   + ARMv8规则：**如果一个** **STLR** **指令后面跟着一个** **LDAR** **指令，硬件强制保证它们之间绝对不会发生 Store-Load 重排**
   + 插入 DMB ISH 的主要情况：
     + **显式内存屏障** ：`std::atomic_thread_fence(std::memory_order_seq_cst);`
     + 旧架构兼容，如**ARMv7**是必然加DMB ISH指令的

仅仅基于ARMv8，如果你仔细思考的话很容易发现 seq_cst 可以完全替代 acquire 和 release ，毕竟硬件指令都一样。ARM 的工程师很快意识到一点，使用 acquire 和 release 是不需要防御 Store-Load 重排的，这导致了语义的过度强化（RISC经常这样），所以ARMv8.3就引入了新的指令来映射 acquire 和 release ，其实就是剥离的 Store-Load 重排防御的 LDAR / STLR。

关于内存屏障，在ARM架构下，我们希望尽可能使用原子变量的内存序来实现，而不是单纯的fence内存屏障。fence内存屏障仅建议在防御无原子变量的读写环境下使用。



#### **CAS操作**

CAS（Compare-And-Swap）是最重要的RMW操作，是所有无锁数据结构的基石。

**x86实现（LOCK CMPXCHG单指令）**

1. **锁缓存行**：当 CPU 遇到 LOCK CMPXCHG 时，它会通过 MESI 协议，向其他核心发送信号，**强行独占**包含该变量的缓存行。在早期的老 CPU 上，这甚至会锁住整个内存总线（Bus Locking），导致所有核心停摆。
2. **原子执行**：在独占缓存行期间，CPU 从缓存中读出值，与寄存器中的 expected 比较。如果相等，将 desired 写入缓存。
3. **释放缓存行**：操作完成，释放独占权。

首先LOCK表示自带全屏障，然后注意到内部完全原子，所以没有伪失败。你会发现这里没有对应不同内存序，所以一定使用全屏障，不过你也不要因此乱填内存序，因为内存序还会影响编译器指令重排。

**ARM实现（LL/SC机制）**

作为RISC，ARM选择拆成两条指令LDXR和STXR。

1. **LDXR (读并监视)**：CPU 从内存读取当前值到寄存器，同时在硬件层面设置一个 **独占监视器**，盯住这个内存地址。
2. **本地比较**：CPU 在自己的寄存器里，用普通的 CMP 指令比较读到的值和 expected。如果不等，直接跳出，CAS 失败。
3. **STXR (条件写)**：如果相等，CPU 尝试执行 STXR 写入新值。**关键点**：STXR 会去问独占监视器：“从我刚才读到现在，有人碰过这个地址吗？”如果监视器说“没有”，写入成功。如果监视器说“有”（监视器被破坏），**写入失败，放弃修改**。

因为是两条指令，所以可能会发生伪失败。

幸运的是，ARM知道trade-off，没有死板的固执于极致的RISC。实际上，在高并发下，CAS操作一旦不严格原子，很容易导致活锁（互相破坏彼此的监视器，谁也写不进去），因此ARM设计师还是选择了单CAS指令：**CASAL**

**CASAL（从独占监视器到远端原子操作）**

所有的RMW操作在ARM架构下最初都是使用带‘X’的两个原子指令，通过专门的监视器来实现整体的原子性。

但是，正如前面所说的，本质类似于乐观锁，所以在高并发下容易导致伪失败。而且，频繁跨核修改会导致其在各个核心的L1缓存中疯狂移动，会导致总线拥堵。而 CASAL 成功解决了以上问题，终止了伪失败。以下为具体操作：

1. **打包发送**：CPU 核心**不再**把内存里的值读到寄存器里。它直接把 expected、desired 和内存地址打包成一个特殊的微指令包。
2. **下放给缓存控制器**：这个包被发送给 L3 缓存控制器（或者内存控制器）。
3. **远端执行**：缓存控制器在内部**锁住**这个缓存行，直接在缓存控制器内部的微型 ALU 中，原子地完成“读取、比较、写入”这三个动作。
4. **返回结果**：缓存控制器把执行结果（成功或失败，以及旧值）返回给 CPU 核心。



关于`memory_order_acq_rel`，这是针对RMW操作特有的内存序，语义是：

+ **读的部分 (Load)** 具有 acquire 语义（阻止后面的指令跑到前面）。使用 LDXR。

- **写的部分 (Store)** 具有 release 语义（阻止前面的指令跑到后面）。使用 STXR。
- 当执行 **LDXR** 时，硬件不仅把值读出来，还在 CPU 内部的“独占监视器”里记录下这个物理地址，标记为“我正在监视它”。
- 当执行 **STXR** 时，硬件会去查监视器：“这个地址被别人碰过吗？”如果没有，写入成功；如果有，写入失败。

+ 当然，正如前面所说，现在已经优化为 CASAL 指令。

**C++层面**

在C++的CAS操作中，编译器都是直接映射为硬件指令，这里来讲一下两个参数success_order和failure_order。

我们前面讲解CAS的ARM指令的时候特地没有讲内存序的区别，这里将展开。

首先，在单CAS指令的情况下，success_order的内存序就是整体的内存序。包含以下四个指令：

1. **CAS (Compare and Swap)**：**没有任何屏障语义**。纯粹的原子读写。对应 C++：memory_order_relaxed
2. **CASA (Compare and Swap, Acquire)**：**只有 Acquire 屏障**。对应 C++：memory_order_acquire
3. **CASL (Compare and Swap, Release)**：**只有 Release 屏障**。对应 C++：memory_order_release
4. **CASAL (Compare and Swap, Acquire and Release)**：**双向屏障**。对应 C++：memory_order_acq_rel 或 memory_order_seq_cst

当然，考虑到向后兼容，C++当然保留了failure_order来处理 LL/SC，以下为cas(..., success = release, failure = relaxed) 在 LL/SC 下的真实汇编逻辑：

```assembly
.loop:
    LDXR  R1, [ADDR]      // 1. 裸的读取 (Relaxed)
    CMP   R1, EXPECTED    // 2. 比较
    B.NE  .fail           // 3. 如果不等，直接跳到失败分支！

    // --- 只有成功才会走到这里 ---
    STLXR R2, DESIRED, [ADDR] // 4. 带有 Release 屏障的写入！
    CBNZ  R2, .loop       // 5. 如果伪失败，重试

.fail:
    // 失败分支，什么屏障都没有！直接返回。
```



#### **指令重排**

当代码没按预期顺序执行的时候，往往是指令重排导致的，同时也意味着你要做好费劲心力找Bug的心理准备。

**编译器层面**

C++之父`Bjarne`说过：`What you do use, you couldn't hand code any better.`

编译器是这样的，它会尽可能地让生成的汇编代码做到足够的优秀。除非你相当有汇编造诣（就像`ffmpeg`的作者那般），不然还是选择尽可能地配合编译器。

很可惜，编译器只懂单线程的逻辑，只要单线程下结果不变，它可以自由打乱代码顺序，只要生成的汇编代码更紧凑、更好地利用 CPU 寄存器、更少的内存访问。

**硬件层面**

为了压榨 CPU 的每一个执行单元，如果指令 A 需要等内存（比如Cache Miss），CPU 不会傻等，它会把后面的指令 B 提前拉出来执行。

CPU 的乱序执行引擎会在内部维护一个重排序缓冲区（ROB），保证**最终提交**的结果与单线程顺序一致。但在这个过程中，内存的读写（通过 Store Buffer 和 Invalidate Queue）会对外呈现出乱序。

**内存序的影响**

**对编译器**：内存序充当了**编译器屏障**。比如 acquire 告诉编译器：“在生成汇编时，绝对不准把这行代码后面的指令挪到它前面去。”

**对硬件**：编译器会在生成的汇编中，插入特定的**硬件屏障指令**（如 LDAR, DMB, MFENCE）。这些指令告诉 CPU：“在执行时，必须排空 Store Buffer 或检查失效队列，绝对不准乱序。”



### 2.4 使用恰当的内存序

以下的内容，我们采用 ARM 架构来讲述，x86 场景简单，我想身为读者的你很容易举一反三。

#### **强大的硬件RMW指令**

通过前面对硬件操作的讲解，我想你能感受到硬件RMW指令的强大，无论是 LL/SC 还是 CAS 单指令，**无论选择什么内存序（哪怕是 relaxed），硬件都绝对保证该变量本身的修改是串行化、不冲突、不丢失的。**

你指定的内存序，本质上还是一道非fence内存屏障。不像 load/store，这些操作本身是需要内存序来保护自身的正确性的，而RMW硬件指令已经能够保证执行指令变量本身不会冲突，丢失。

所以，在无普通数据（线程不安全的数据）负载的情况下，我们一般都是选择relaxed，比如计数器（fetch_add）。

CAS使用acquire/release的一个常见场景是锁的实现，毕竟锁就是为了保护线程不安全的数据嘛。而且，这里的保护是单向的。

当然，有时我们会遇到需要双向保护的场景。最常见的就是`shared_ptr`的析构，引用计数将`fetch_sub(1);`，一般来说我们析构之前只要 release 保证内部保护的数据进行的操作成功发布，但是如果引用计数归0，它需要确保其他 `shared_ptr` 的操作已经执行（可见），所以还需要加上 acquire 语义，综合为 acq_rel。

少数情况下，我们可能还要防御Store-Load重排，比如一个经典逻辑“我写了变量 X，然后去读变量 Y；同时别人写了变量 Y，来读变量 X”，那么就需要使用seq_cst。

补充一点，在现代ARM/x86下，acq_rel自带Store-Load重排防御，LL/SC的话就不会了，有是否加 DMB ISH 的区别。不过，就C++的内存模型而言，acq_rel就是不防御 Store-Load 重排的，请不要直接依赖硬件。



#### **安全的数据发布——生产/消费**

这应该是最常见的情况。在无锁队列中，无论静态还是动态内存，往往对应位置都得进行成员构造或者其他的内存操作。生产者和消费者分别对内存进行填充和消费（或者统一为修改），幸运的是，这个数据流向是单向的，所以我们采用 acquire/release 来保证正确性。

以一个无锁的MPMC队列来说：

```c++
struct Slot {
        std::atomic<size_t> sequence;
        alignas(T) std::byte data[sizeof(T)];
    };

    alignas(kCacheLineSize) std::atomic<size_t> _head;
    alignas(kCacheLineSize) std::atomic<size_t> _tail;
    alignas(kCacheLineSize) Slot _buffer[Capacity];
```

生产者仅需要操作tail，消费者仅需要操作head。我们对于head和tail的所有操作使用relaxed内存序，把剩下的内存同步任务交给序列号sequence。

生产者填充完Slot后，需要release修改sequence，这样保证了sequence可见，对应的data一定已经操作完成且可见。

消费者在获取Slot后，需要acquire读取序列号，这保证了读取到最新的sequence，并且作为屏障，后续对data的操作一定是基于最新读到的这个sequence。

```C++
template <typename... Args>
    bool enqueue(Args&&... args) noexcept {
        static_assert(std::is_constructible_v<T, Args...>, "T cannot be constructed from these arguments");

        size_t tail = _tail.load(std::memory_order_relaxed);
        while (true) {
            Slot& slot = _buffer[tail & Mask];
            size_t seq = slot.sequence.load(std::memory_order_acquire);
            if (seq == tail) {
                if (_tail.compare_exchange_weak(tail, tail + 1, std::memory_order_relaxed)) {
                    T* ptr = std::launder(reinterpret_cast<T*>(&slot.data));
                    new (ptr) T(std::forward<Args>(args)...);
                    slot.sequence.store(tail + 1, std::memory_order_release);
                    return true;
                }
            }
            else if (seq < tail) {
                return false;
            }
            else {
                tail = _tail.load(std::memory_order_relaxed);
            }
        }
    }

bool dequeue(T& result) noexcept {
        size_t head = _head.load(std::memory_order_relaxed);

        while (true) {
            Slot& slot = _buffer[head & Mask];
            size_t seq = slot.sequence.load(std::memory_order_acquire);
            if (seq == head + 1) {
                if (_head.compare_exchange_weak(head, head + 1, std::memory_order_relaxed)) {
                    T* ptr = std::launder(reinterpret_cast<T*>(&slot.data));
                    result = std::move(*ptr);
                    ptr->~T();
                    
                    slot.sequence.store(head + Capacity, std::memory_order_release);
                    return true;
                }
            }
            else if (seq < head + 1) {
                return false;
            }
            else {
                head = _head.load(std::memory_order_relaxed);
            }
        }
    }
```

顺便，我们来讲解一下这个序列号机制。

首先，所有槽位的序列号为其在\_buffer中的下标，表示待生产。tail & Mask是我们希望当前生产的槽位，head & Mask则是我们希望当前消费的槽位，并且消费完后需要把序列号变成下一次生产所需的序列号。

以槽位1来说，它的序列号应该是如下变化：初始值为1——>生产后为2——>消费后为1 + Capacity——>生产后为2 + Capacity......

**enqueue**：

如果 seq < tail，那么就说明当前槽位还未被消费，直接返回即可。
如果 seq > tail，就说明当前槽位已经被其他生产者抢走且已经生产完了，那么我们重新获取\_tail的最新值。
如果seq == tail，先别急着说这个槽位是我的了，我们需要先CAS抢占获取。

**dequeue**：

如果 seq < head + 1，那么就说明当前槽位还未生产，直接返回即可。
如果 seq > head + 1，就说明当前槽位已经被其他消费者抢走且已经消费完了，那么我们重新获取\_head的最新值。
如果seq == head + 1，先别急着说这个槽位是我的了，我们需要先CAS抢占获取。



#### **状态互相依赖——防御Store-Load重排**

前面的\_head和\_tail互相并不依赖，所以我们不用采取更严格的内存序。但是，当队列的首尾（不同的原子变量）互相依赖的时候，我们就需要加入seq_cst了。

以一个Chase-Lev队列来说：

```C++
struct Slot {
        alignas(T) std::byte data[sizeof(T)];
        // 1 字节的状态标志，用于防御 Wrap-around Bug
        std::atomic<bool> is_extracted;
    };
    alignas(kCacheLineSize) std::atomic<size_t> _head;
    alignas(kCacheLineSize) std::atomic<size_t> _tail;
    alignas(kCacheLineSize) Slot _buffer[Capacity];
```

现在，我们允许任务窃取。生产者仅有本地线程且操作\_tail，本地线程从\_tail获取数据，而窃取线程从\_head来获取数据。所以，\_tail就可以纯 load/store 而不需要CAS操作。

```C++
template <typename... Args>
    bool push(Args&&... args) noexcept {
        static_assert(std::is_constructible_v<T, Args...>, "T cannot be constructed from these arguments");

        size_t tail = _tail.load(std::memory_order_relaxed);
        size_t head = _head.load(std::memory_order_relaxed);
        
        if (tail - head >= Capacity) [[unlikely]] {
            return false;
        }

        Slot& slot = _buffer[tail & Mask];
        
        // 防御 Wrap-around Bug
        // 等待可能被挂起的 Stealer 完成数据提取。
        while (!slot.is_extracted.load(std::memory_order_acquire)) [[unlikely]] {
            #if defined(_MSC_VER)
            _mm_pause();
            #elif defined(__GNUC__) || defined(__clang__)
            __builtin_ia32_pause();
            #endif
        }

        T* ptr = std::launder(reinterpret_cast<T*>(&slot.data));
        new (ptr) T(std::forward<Args>(args)...);
        
        slot.is_extracted.store(false, std::memory_order_relaxed);
        
        _tail.store(tail + 1, std::memory_order_release);
        
        return true;
    }


bool pop(T& result) noexcept {
        size_t tail = _tail.load(std::memory_order_relaxed);
        if (tail == 0) return false;  // 防无符号数下溢，但不防御空队列
        
        tail = tail - 1;
        _tail.store(tail, std::memory_order_seq_cst);

        size_t head = _head.load(std::memory_order_seq_cst);

        if (head <= tail) {
            Slot& slot = _buffer[tail & Mask];
            T* ptr = std::launder(reinterpret_cast<T*>(&slot.data));

            if (head == tail) {
                // 队列只剩最后一个元素，可能与 Stealer 发生竞争
                if (!_head.compare_exchange_strong(head, head + 1, 
                                                  std::memory_order_relaxed)) {
                    // 竞争失败，Stealer 抢走了任务
                    _tail.store(tail + 1, std::memory_order_relaxed); // 恢复 _tail
                    return false;
                }
                // 竞争成功，我们拿到了最后一个任务
                result = std::move(*ptr);
                ptr->~T();
                slot.is_extracted.store(true, std::memory_order_release);
                
                _tail.store(tail + 1, std::memory_order_relaxed); // 队列彻底空了
                return true;
            }

            // 正常情况：队列元素 > 1，绝对安全，直接拿走
            result = std::move(*ptr);
            ptr->~T();
            // 保证先写入result然后标志位才可见
            slot.is_extracted.store(true, std::memory_order_release);
            return true;
        } else {
            // 队列为空 (在我们减 _tail 之前，Stealer 已经把队列偷空了)
            _tail.store(tail + 1, std::memory_order_relaxed); // 恢复 _tail
            return false;
        }
    }

bool steal(T& result) noexcept {
        size_t head = _head.load(std::memory_order_acquire);
        
        size_t tail = _tail.load(std::memory_order_acquire);

        if (head < tail) {
            // 尝试抢占队头
            if (_head.compare_exchange_strong(head, head + 1, 
                                             std::memory_order_relaxed)) {
                // 抢占成功
                Slot& slot = _buffer[head & Mask];
                T* ptr = std::launder(reinterpret_cast<T*>(&slot.data));
                
                result = std::move(*ptr);
                ptr->~T();
                
                // 释放 Slot，允许覆盖写入
                slot.is_extracted.store(true, std::memory_order_release);
                return true;
            }
        }
        return false;
    }

```

\_tail更新可以保证slot已经构造完成，但是_head更新仅仅是抢占槽位，这使得必须引入标志位表示数据已经消费。is_extracted 的使用是非常典型的 acquire/release，这里不再赘述。

**push**：\_head 使用 relaxed 读取，因为没有数据负载，哪怕和实际冲突也只是“伪满”，不会影响正确性，故进作为启发式判满。\_tail使用 release 发布数据，标准的数据发布语义。

**pop**：_tail Store时使用 seq_cst，这是为了新的窃取线程能够立刻获得最新的 _tail（哪怕在 Store Buffer 中），这样新来的窃取线程就绝对不会和本地线程竞争当前槽位。

**那么，如果窃取线程已经 acquire 读取了 \_tail 呢？**

如果当前的任务数量大于 1，那么本地线程直接原子操作获取尾部任务。而其他窃取线程CAS抢占头部任务，谁抢到就给谁。

如果当前任务数量等于 1，那么本地线程也参与到CAS竞争中，如果本地线程没抢到，那么 relaxed 恢复 \_tail（因为此时新的窃取线程只会直接返回，直到本地线程通过push()修改\_tail）。

值得一提的是，CAS 逻辑完全是 relaxed，虽然前面我已经有讲过 CAS 的强大，这里再补充一遍：
首先我们不需要release，因为CAS之前的操作没有内存写，即使有，也是\_tail（但是是seq_cst内存序所以不用担心），其次我们不需要acquire，acquire是担心数据是否已经写完，但是只要\_tail可见，对应的槽位就必然已经生产完了的，本地线程的\_tail一定可见，而窃取线程的\_tail已经用acquire保证可见了。

**steal**：读取最新的\_head和\_tail，然后尝试抢占任务。_head的acquire读用于保证先读\_head再读\_tail（原因在下面），而\_tail的acquire则是前面讲到的安全数据发布。

这里**补充一下**：如果我们考虑x86的话，其实\_head是不需要严格内存序的，因为前面_tail自带全屏障且之后的CAS操作已经做好了一切。但是如果考虑ARM架构的话，它的seq_cst是一对指令LDAR / STLR，需要一前一后，这样硬件不需要全屏障而是通过其他操作避免Store-Load重排。

**注意**：如果你觉得我前面的解释不够完备，甚至有些想当然，那么你大概有很敏锐的洞察力，人脑推演的话对于多线程是较为困难的，这里我们来讲解最极端的问题。

当任务数为2的时候，也就是最危险的时候，假设\_head为0，\_tail为2。我们来推演一下：

为什么是2？因为任务数为 1 时本地线程也会CAS，你可以理解为退化为先前的Mpmc队列，都使用CAS抢占是绝对没问题的。

有一堆窃取线程（threads_a）只读到了\_head为0，另一堆窃取线程（threads_b）则读到了\_head=0，\_tail=2。现在他们都挂起在CAS前。
接着进入本地线程，本地线程修改\_tail使其减1并且立刻广播，后续线程就必然读到\_tail=1。本地线程线程仅知道\_tail = 1。然后挂起。
前面的所有窃取线程，最多只会把\_head增加1，这不会影响本地线程还有任务。而且后续窃取线程读到的只会是\_head=1和\_tail=1，直接退出。
如果前面的窃取线程还在挂起，那么接着，窃取线程（threads_c）读到了\_head为0，窃取线程（threads_d）读到了\_head为0，\_tail为1。
此时目前这些窃取线程也最多把\_head加1，不影响本地线程还有任务。
现在，无论本地线程读到的\_head是0还是1，第二个任务都只会被本地线程获取，因为所有窃取线程都只会抢第一个任务。
**这就是为什么steal()一定要先读\_head再读\_tail，因为只读\_tail并且卡在这里，后续的读到的\_head就不确定了。_head不确定就会导致窃取的任务不确定。**

至此，我们总算是用了最轻松的内存序最严格地保证了内存一致性。



### 2.5 小结

这章花费了我不少的精力，结合了大量我的血泪教训和经验，以及大量的资料查阅。

虽然整体有很多基础概念可能没有讲清，以及或许会有一定的跳跃性，但我想如果你有不错的并发编程基础，应该还是能够看懂且获益匪浅的。



## 三、Futex

**Futex**并不是一种独立的锁类型，而是一种由Linux内核提供的、用于构建更高级锁（如互斥锁、信号量）的底层同步原语。它的设计思想是极致的优化。

在它出现之前，传统的 IPC 机制（如 System V 信号量）无论是否发生竞争，每次加锁/解锁都必须陷入内核（执行系统调用），开销极大。

Futex 的核心哲学只有一句话：**“无竞争时，用户态解决（极速）；有竞争时，内核态仲裁（睡眠）。”**



### 3.1 Futex的底层实现

**核心数据结构**

- **futex_hash_bucket（哈希桶）：** 全局哈希表由多个桶组成。每个桶里有一把内核自旋锁（保护这个桶）和一个双向链表（等待队列）。
- **futex_q（等待节点）：** 当一个线程调用 FUTEX_WAIT 时，内核会为它创建一个 futex_q 节点，挂在对应的哈希桶的链表上。这个节点里记录了：
  - 等待的 Key（见下文）。
  - 指向该线程 task_struct 的指针。

**Futex Key 的生成**

用户仅仅传入了一个虚拟地址，我们需要针对该虚拟地址对应变量找到唯一标识符。

如果该变量仅在单进程出现，那么Futex Key可以直接使用当前进程的 mm_struct 指针 + 虚拟地址，绝对唯一标识。

但是我们不能假设单进程环境，故必须找到物理地址作为Key，具体操作如下：

1. 内核查当前进程的页表，找到 uaddr 对应的**物理页帧**。
2. 内核提取出该物理页的标识（如果是匿名页，就是物理地址；如果是文件映射页，就是 inode + offset）。
3. 内核将这个**绝对唯一的物理标识**作为 Key，进行哈希计算，找到对应的 futex_hash_bucket。



### 3.2 Futex的使用

Futex是Linux内核暴露给用户态的一个**图灵完备的并发微内核**。通过直接操作Futex，我们可以实现所有的同步/互斥语义。可惜的是，C++直到C++20才开始暴露基本的Futex功能（如std::atomic::wait），所以，针对Linux，如果你希望根据使用场景最大化优化同步/互斥开销，请直接使用Futex。

```C++
long syscall(SYS_futex, uint32_t *uaddr, int futex_op, uint32_t val,
             const struct timespec *timeout, uint32_t *uaddr2, uint32_t val3);
```

我们针对 futex_op（命令+标志位）来分别讲解，当然考虑篇幅只讲常用的，如果要全面的讲解你为什么不去查官方文档呢？

#### **标志位**

**FUTEX_PRIVATE_FLAG（私有标志）**

- 默认情况下，Futex 是支持跨进程（IPC）的。为了保证不同进程映射同一块物理内存时能找到同一把锁，内核必须锁住进程的内存描述符（mmap_sem），遍历页表，计算出物理页的唯一标识作为 Hash Key。这个过程在多核高并发下极其昂贵。
- 如果你加上这个 Flag，你就是在向内核发誓：“这把锁只在我这一个进程的多个线程间使用！”
- 内核直接跳过页表查询，直接用 **当前进程的 mm_struct 指针 + uaddr 的虚拟地址** 作为 Hash Key。**这不仅省去了查页表的开销，更重要的是彻底避开了内核内存管理子系统的锁竞争。** 
- 一般来说，很少会跨进程并发，所以使用用途挺广的。

**FUTEX_CLOCK_REALTIME（时钟标志）**

- 默认情况下，timeout 参数使用的是 CLOCK_MONOTONIC（单调时钟，不受系统时间修改影响）。加上这个 Flag，可以切换为墙上时钟（受 NTP 同步或手动改时间影响），通常用于实现 pthread_cond_timedwait。



#### **命令**

**FUTEX_WAIT (对应 std::atomic::wait)**

- **语义：** “请把当前线程挂起睡眠，**前提是**用户态地址 uaddr 里的值现在依然等于 val。”
- **为什么必须传入期望值 val？（防丢失唤醒机制）**
  假设没有 val 参数。线程 A 发现锁是状态 2，准备调用 futex_wait(uaddr) 睡眠。就在它刚准备陷入内核的瞬间，线程 B 释放了锁，把状态改回 0，并调用了 futex_wake(uaddr)。
  因为线程 A 还没真正睡着，这个 wake 信号就**丢失**了。随后线程 A 陷入内核睡死，再也没有人唤醒它。
  **传入 val 完美解决了这个问题：** 内核在真正把线程挂起之前，会**再次读取** uaddr 的值。如果发现 *uaddr != val（说明状态已经被别人改了），内核会直接返回 EWOULDBLOCK，拒绝睡眠。线程 A 回到用户态重新自旋检查即可。

**FUTEX_WAKE (对应 std::atomic::notify_one/all)**

- **语义：** “请唤醒正在等待用户态地址 uaddr 的最多 val 个线程。”
- 如果 val = 1，就是 notify_one；如果 val = INT_MAX，就是 notify_all。

**FUTEX_CMP_REQUEUE (条件重排队)**

+ **语义：** “唤醒 uaddr 上的 val 个线程，然后**把剩下的所有等待线程，直接在内核态转移（Requeue）到 uaddr2（即 Mutex 的地址）的等待队列中**。”

- **解决惊群效应**。
  假设你用条件变量实现了一个生产者-消费者模型。100 个消费者在等待条件变量。
  生产者生产了数据，调用 pthread_cond_broadcast。
  如果底层只是简单地 FUTEX_WAKE 唤醒这 100 个线程，灾难就发生了：100 个线程同时醒来，从内核态返回用户态，然后**立刻去争抢与之绑定的那把 Mutex 互斥锁**。结果只有 1 个线程抢到，剩下 99 个线程又得立刻调用 FUTEX_WAIT 重新陷入内核睡觉。这 99 次上下文切换纯属浪费 CPU。
- **结果：** 100 个线程中，只有 1 个被真正唤醒到用户态去拿锁。剩下的 99 个线程在睡梦中被内核偷偷“搬了家”，直接挂在了 Mutex 的等待队列上。没有任何无意义的唤醒和上下文切换。

**FUTEX_LOCK_PI / FUTEX_UNLOCK_PI (优先级继承)**

- **解决优先级反转**。
  假设有三个线程：高优先级(H)、中优先级(M)、低优先级(L)。
  L 拿到了锁。H 想要锁，被阻塞。此时 M 开始运行，因为 M 优先级比 L 高，M 一直抢占 CPU，导致 L 永远无法运行去释放锁。结果就是：最高优先级的 H 被中优先级的 M 间接饿死了。
- 当你使用 FUTEX_LOCK_PI 时，用户态的锁变量（uaddr）里不仅存了锁状态，还存了**当前持有锁的线程 TID**。
  当 H 线程在内核中等待时，内核会读取这个 TID，发现是 L 线程拿着锁。内核会**临时将 L 线程的调度优先级提升到和 H 一样高。**
  这样 L 就能立刻抢占 M 的 CPU，迅速执行完临界区并释放锁（FUTEX_UNLOCK_PI），随后内核将 L 的优先级恢复原状，H 成功拿到锁。

**FUTEX_WAKE_OP (唤醒并修改)**

- **减少系统调用次数**。
  在某些复杂的同步原语中，你可能需要唤醒一个 Futex，同时修改另一个 Futex 的值，然后再唤醒它。如果分两次系统调用，开销太大。
- 这个命令允许你在**内核态**执行一段极其简单的“微指令”。
  参数 val3 被编码成了一个指令集（包含操作码、比较符、操作数）。
  内核会先唤醒 uaddr 上的线程，然后根据 val3 的指令，原子地修改 uaddr2 的值，如果满足条件，再唤醒 uaddr2 上的线程。这一切都在一次系统调用、一次内核锁的保护下完成。

**FUTEX_WAIT_BITSET / FUTEX_WAKE_BITSET (位图过滤)**

- **事件多路复用与分组唤醒**。
  假设有 10 个线程等在同一个 Futex 上，但它们等待的“事件类型”不同（比如 5 个等读事件，5 个等写事件）。传统的 WAKE 只能盲目唤醒，或者全部唤醒。
- 引入了 val3 作为 Bitset（位图掩码）。线程调用 WAIT_BITSET 时，传入一个掩码（比如 0x01 代表读，0x02 代表写）。内核会把这个掩码存在 futex_q 节点里。唤醒者调用 WAKE_BITSET 时，也传入一个掩码。内核在遍历等待队列时，会将唤醒掩码与等待掩码进行按位与（&）操作。**只有结果不为 0 的线程才会被唤醒。**这极其适合实现高效的读写锁或复杂的事件分发器。



### 3.3 小结

一般来说，我应该使用Futex实现一个易用的并发组件，但是限于篇幅，后续更新再说吧。









