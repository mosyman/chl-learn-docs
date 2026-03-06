
## 目录
- [notify(),notifyAll()](#notifynotifyall)
    - [1](#1)
    - [2](#2)



## notify(),notifyAll()

# 1
这两个方法 `notify()` 和 `notifyAll()` 是 Java 并发编程中**等待/通知机制（Wait/Notify Pattern）**的核心，它们定义在 `java.lang.Object` 类中。这意味着**任何 Java 对象**都可以作为线程间通信的媒介。

要深入理解它们，我们需要从**API 行为语义**、**JVM 底层实现原理**、**操作系统交互**以及**常见误区**四个维度进行详细剖析。

---

### 1. 核心语义与行为对比

#### `notify()`
*   **作用**：随机唤醒**一个**正在该对象监视器（Monitor）上等待（`wait`）的线程。
*   **选择策略**：**任意（Arbitrary）**。JVM 规范没有规定唤醒哪一个，完全由具体实现决定（通常取决于操作系统的调度策略或等待队列的结构）。
*   **执行时机**：被唤醒的线程**不会立即执行**。它必须等待当前持有锁的线程退出同步块（释放锁）后，与其他试图获取该锁的线程（包括新来的线程）重新竞争锁。
*   **风险**：如果唤醒了错误的线程（例如，该线程等待的条件并未满足），而其它真正需要处理的线程还在沉睡，可能导致**死锁**或**逻辑停滞**。

#### `notifyAll()`
*   **作用**：唤醒**所有**正在该对象监视器上等待的线程。
*   **执行时机**：同样，所有`被唤醒的线程都不会立即执行`，而是`进入锁的竞争队列`（Entry Set），等待当前线程释放锁后，逐个获取锁并继续执行。
*   **优势**：安全性高。无论有多少种不同的等待条件，唤醒所有线程让它们重新检查条件（`while(!condition) wait()`），总能保证正确的线程被处理。
*   **代价**：性能开销略大（上下文切换多），但在现代 JVM 优化下，这种开销通常可以忽略不计，除非是在极高并发场景。

---

### 2. 底层原理：HotSpot JVM 实现机制

Java 的 `synchronized` 和 `wait/notify` 机制在 HotSpot JVM 中是基于 **ObjectMonitor** 实现的。这是一个 C++ 类，每个 Java 对象头（Object Header）中都包含一个指向 `ObjectMonitor` 的指针（当对象处于`重量级锁`状态时）。

#### 2.1 关键数据结构：ObjectMonitor
在 HotSpot 源码 (`src/hotspot/share/runtime/objectMonitor.hpp`) 中，`ObjectMonitor` 主要维护了三个关键队列/集合：

1.  **Owner**：当前持有锁的线程。
2.  **EntryList**（入口队列）：
    *   存放那些**试图获取锁但失败**，正在阻塞等待锁的线程。
    *   这些线程处于 `BLOCKED` 状态。
3.  **WaitSet**（等待集合）：
    *   存放那些调用了 `wait()` 方法，主动释放了锁并进入等待状态的线程。
    *   这些线程处于 `WAITING` 或 `TIMED_WAITING` 状态。
4.  **Cxq** (Contention Queue)：
    *   在某些版本的 HotSpot 中，用于优化锁竞争的高速缓存友好型队列，线程可能先入这里，再迁移到 EntryList。

#### 2.2 `notify()` 的底层流程
当线程 A 调用 `obj.notify()` 时（假设 A 已经持有 `obj` 的锁）：

1.  **权限检查**：JVM 首先检查当前线程是否是 `ObjectMonitor` 的 **Owner**。如果不是，直接抛出 `IllegalMonitorStateException`。
2.  **查找 WaitSet**：JVM 遍历 `WaitSet` 链表。
3.  **挑选线程**：
    *   如果是 `notify()`：从 `WaitSet` 中**随机**（或按某种内部顺序，如 FIFO/LIFO，取决于实现版本）挑选**一个**线程 B。
    *   如果是 `notifyAll()`：遍历 `WaitSet`，挑选**所有**线程。
4.  **状态迁移**：
    *   将选中的线程 B 从 `WaitSet` 移除。
    *   将线程 B 放入 **EntryList**（或 Cxq）。
    *   线程 B 的状态从 `WAITING` 变为 `BLOCKED`。
    *   **注意**：此时线程 B **并没有** 获得锁，它只是获得了“竞争锁的资格”。
5.  **返回**：`notify()` 方法执行结束，线程 A 继续执行后续代码，直到退出 synchronized 块。

#### 2.3 锁释放与重新竞争
只有当线程 A 执行完 synchronized 代码块，调用 `monitorexit` 指令释放锁时：

1.  JVM 会从 **EntryList** 中选取一个线程（通常是等待最久的，或者是根据优先级/自适应自旋策略选取）。
2.  将该线程设置为新的 **Owner**。
3.  该线程从 `BLOCKED` 状态变为 `RUNNABLE`，开始执行 `wait()` 方法之后的代码。

> **关键点解释**：为什么被唤醒的线程不能立即执行？
> 因为 `notify()` 调用时，当前线程 A 依然持有锁。Java 的监视器模型规定，同一时刻只有一个线程能持有锁。如果允许被唤醒的线程 B 立即执行，就会破坏互斥性。因此，B 必须等到 A 释放锁。

---

### 3. 操作系统层面的交互

Java 线程映射到操作系统内核线程（Kernel Thread, 如 Linux 上的 pthread）。

*   **wait() 时**：
    *   JVM 调用操作系统的条件变量接口（如 Linux 的 `pthread_cond_wait`）。
    *   操作系统将该线程挂起（Park），放入内核的等待队列，线程不再占用 CPU 时间片，状态表现为操作系统的 `Sleeping` 或 `Waiting`。
*   **notify() 时**：
    *   JVM 调用操作系统的唤醒接口（如 `pthread_cond_signal` 或 `pthread_cond_broadcast`）。
    *   操作系统将对应的内核线程状态标记为 `Runnable`，并将其移入操作系统的运行就绪队列。
*   **锁竞争时**：
    *   当锁释放，操作系统调度器根据调度算法（CFS 等）决定哪个线程真正获得 CPU 时间片运行。

这就是为什么文档中说 "The choice is arbitrary"（选择是任意的）且 "enjoys no reliable privilege"（没有特权），因为这最终依赖于操作系统的线程调度器，而 OS 调度器为了公平性和吞吐量，通常不会保证严格的 FIFO 顺序（尽管很多实现近似 FIFO）。

---

### 4. 为什么推荐使用 `notifyAll()`？（经典陷阱）

虽然 `notify()` 看起来效率更高（只唤醒一个），但在实际开发中，** Effective Java (Item 84)** 强烈建议优先使用 `notifyAll()`。

#### 场景：多条件等待
假设一个缓冲区 `Buffer`，既有生产者（满时等待），也有消费者（空时等待）。它们都在同一个对象锁上 `wait()`。

```java
// 错误的示范
public void put(Item item) {
    synchronized (this) {
        while (isFull()) {
            wait(); // 生产者等待
        }
        // ... 添加元素 ...
        notify(); // 危险！可能唤醒了另一个生产者，而不是消费者
    }
}

public void take() {
    synchronized (this) {
        while (isEmpty()) {
            wait(); // 消费者等待
        }
        // ... 移除元素 ...
        notify(); // 危险！可能唤醒了另一个消费者，而不是生产者
    }
}
```

**问题分析**：
1.  缓冲区满了，生产者 P1 调用 `wait()` 进入 WaitSet。
2.  缓冲区也空了（极端情况或逻辑错误），消费者 C1 调用 `wait()` 进入 WaitSet。
3.  一个新的生产者 P2 进来，放入了一个元素，调用 `notify()`。
4.  **灾难发生**：JVM 随机唤醒了 P1（另一个生产者）。
5.  P1 醒来，获取锁，检查 `isFull()`，发现还是满的（因为刚才放入元素的 P2 还没让出锁，或者逻辑上依然满），于是 P1 再次 `wait()`。
6.  真正的消费者 C1 依然在沉睡，没人去消费数据。
7.  如果之后没有新的生产者进来触发 `notify`，系统就**死锁**了（所有生产者都在等空间，消费者在等数据但没被唤醒）。

**正确使用 `notifyAll()`**：
如果使用 `notifyAll()`，P2 放入元素后唤醒所有人。
1.  P1 醒来，检查 `isFull()` -> true -> 继续 `wait()`。
2.  C1 醒来，检查 `isEmpty()` -> false -> 执行消费逻辑。
3.  问题解决。

**结论**：使用 `notifyAll()` 配合 `while` 循环检查条件，是解决“虚假唤醒”和“多条件等待”最安全、最通用的模式。性能损耗在现代系统中通常是可以接受的。

---

### 5. 总结与关键点回顾

| 特性 | `notify()` | `notifyAll()` |
| :--- | :--- | :--- |
| **唤醒数量** | 1 个（随机） | 所有 |
| **安全性** | 低（在多条件等待易死锁） | 高（通用模式） |
| **性能** | 理论上稍好（少唤醒） | 稍差（多上下文切换），但通常可忽略 |
| **底层动作** | WaitSet -> EntryList (1个) | WaitSet -> EntryList (全部) |
| **锁状态** | 调用者必须持有锁 | 调用者必须持有锁 |
| **执行流** | 被唤醒者需重新竞争锁 | 被唤醒者需重新竞争锁 |
| **推荐度** | 仅在单一等待条件且性能极度敏感时使用 | **默认推荐** |

**底层核心记忆点**：
1.  **Monitor 模型**：基于 ObjectMonitor，涉及 Owner, EntryList, WaitSet。
2.  **状态流转**：`WAITING` (WaitSet) --(notify)--> `BLOCKED` (EntryList) --(lock released)--> `RUNNABLE`。
3.  **非即时性**：notify 不等于 unlock，也不等于立即运行，只是“入场券”。
4.  **Native 实现**：方法是 `native` 的，直接桥接到 C++ 运行时库，利用 OS 的原生同步原语。

理解这些原理，能帮助你在编写多线程代码时，避免死锁，正确选择通知策略，并理解为何有时候线程“看起来”被唤醒了却没动静（因为它还在 EntryList 里排队抢锁）。


[目录](#目录)


# 2

你想深入理解 Java 中 `Object` 类的 `notify()` 和 `notifyAll()` 方法的详细语义、使用规则以及背后的底层实现原理，我会从**功能解释**、**核心规则**、**底层原理**、**使用差异**四个维度全面拆解。

### 一、核心功能与语义解释
首先明确这两个方法的核心定位：它们是 Java 内置的**线程间通信机制**，用于唤醒因调用 `wait()` 方法而阻塞在某个对象监视器（monitor）上的线程，属于「等待-通知」模型的核心 API。

#### 1. notify() 方法
- **字面语义**：唤醒**任意一个**正在当前对象监视器上等待的线程。
- **关键细节**：
    - 「任意性」：被唤醒的线程由 JVM 实现决定（无固定规则，如不按等待顺序、优先级等），开发者无法指定。
    - 「无法立即执行」：被唤醒的线程不会立刻获得对象锁，必须等待当前调用 `notify()` 的线程**释放锁**（退出 `synchronized` 代码块/方法）后，才能参与锁的竞争。
    - 「调用前提」：当前线程必须是该对象监视器的「所有者」（持有锁），否则抛出 `IllegalMonitorStateException`。

#### 2. notifyAll() 方法
- **字面语义**：唤醒**所有**正在当前对象监视器上等待的线程。
- **关键细节**：
    - 「全部唤醒」：所有等待线程都会被唤醒，但最终只有一个线程能竞争到锁，其余线程会重新进入「阻塞状态」等待锁释放。
    - 「竞争规则」：被唤醒的线程遵循公平/非公平规则（由锁类型决定）竞争锁，无任何特权（比如先等的线程不一定先拿到锁）。
    - 「调用前提」：与 `notify()` 完全一致，必须持有对象锁。

### 二、核心规则（必须遵守）
从注释和 Java 规范中提炼的硬性规则：

| 规则 | 说明 |
|------|------|
| 锁持有要求 | 调用 `notify()`/`notifyAll()` 的线程必须是对象监视器的所有者（持有锁），否则抛 `IllegalMonitorStateException` |
| 监视器所有权的获取方式 | 1. 执行对象的 `synchronized` 实例方法；<br>2. 执行 `synchronized (对象)` 代码块；<br>3. 执行 `Class` 对象的 `synchronized` 静态方法（针对类锁） |
| 监视器排他性 | 同一时间只有一个线程能持有对象的监视器（锁） |
| 唤醒后行为 | 被唤醒线程需重新竞争锁，竞争失败则继续阻塞，直到拿到锁后才能从 `wait()` 方法返回 |

### 三、底层原理（JVM + OS 层面）
要理解这两个方法的底层，必须先理解「对象监视器（Monitor）」和「线程状态流转」：

#### 1. 核心概念：对象监视器（Monitor）
每个 Java 对象都关联一个「监视器」（可理解为「锁 + 等待队列 + 入口队列」的组合体），其结构简化如下：
```mermaid
graph TD
    A[对象Monitor] --> B[持有锁的线程（owner）]
    A --> C[等待队列（Wait Set）：调用wait()的线程]
    A --> D[入口队列（Entry Set）：竞争锁的线程]
```
- **Owner**：当前持有锁的线程（最多1个）。
- **Wait Set**：调用 `wait()` 后释放锁、进入「等待状态」的线程集合（核心：线程会释放锁，且不在 Entry Set 中）。
- **Entry Set**：未拿到锁、处于「阻塞状态」（BLOCKED）的线程集合（比如进入 `synchronized` 但锁被占用）。

#### 2. notify()/notifyAll() 的底层执行流程
##### （1）notify() 底层流程
```mermaid
graph LR
    Step1[当前线程调用notify()] --> Step2[JVM从Wait Set中随机选1个线程]
    Step2 --> Step3[将该线程从Wait Set移到Entry Set]
    Step3 --> Step4[线程状态从WAITING变为BLOCKED]
    Step4 --> Step5[当前线程释放锁后，Entry Set的线程竞争锁]
```
- 关键：仅移动**一个**线程到入口队列，其余仍在等待队列。

##### （2）notifyAll() 底层流程
```mermaid
graph LR
    Step1[当前线程调用notifyAll()] --> Step2[JVM将Wait Set中所有线程移到Entry Set]
    Step2 --> Step3[所有线程状态从WAITING变为BLOCKED]
    Step3 --> Step4[当前线程释放锁后，Entry Set的所有线程竞争锁]
```
- 关键：清空 Wait Set，所有等待线程进入锁竞争。

#### 3. 底层实现的核心依赖
- **native 方法**：两个方法均被 `native` 修饰，说明其实现不在 Java 层，而是由 JVM 调用操作系统的「线程调度」接口（如 Linux 下的 `pthread_cond_signal()`/`pthread_cond_broadcast()`）。
- **IntrinsicCandidate 注解**：标记该方法为「固有方法」，JVM 会对其做特殊优化（如直接编译为机器码，而非通过 JNI 调用），提升性能。
- **操作系统层面**：
    - `notify()` 对应操作系统的「条件变量单播唤醒」（`pthread_cond_signal`）；
    - `notifyAll()` 对应「条件变量广播唤醒」（`pthread_cond_broadcast`）；
    - 监视器的锁机制底层依赖操作系统的「互斥量（mutex）」实现。

### 四、notify() vs notifyAll() 核心差异与使用场景
| 维度 | notify() | notifyAll() |
|------|----------|-------------|
| 唤醒数量 | 1个（任意） | 全部 |
| 性能 | 开销小（仅唤醒1个） | 开销大（唤醒全部，竞争激烈） |
| 风险 | 可能导致「线程饥饿」（某些线程一直没被唤醒） | 无饥饿风险，但竞争成本高 |
| 使用场景 | 等待条件满足时，**只有一个线程能处理任务**（如生产者-消费者中，生产1个产品只需唤醒1个消费者） | 等待条件满足时，**多个线程需重新检查条件**（如多个线程等待不同条件，需全部唤醒后自行判断） |

#### 典型错误场景（新手必避）
```java
// 错误示例：用notify()唤醒，但多个线程等待同一条件，可能导致线程永久等待
synchronized (obj) {
    while (条件不满足) {
        obj.wait(); // 多个线程进入Wait Set
    }
    obj.notify(); // 仅唤醒1个，其余线程永远等待
}

// 正确示例：用notifyAll()确保所有线程都能检查条件
synchronized (obj) {
    while (条件不满足) {
        obj.wait();
    }
    obj.notifyAll(); // 唤醒全部，线程重新检查条件
}
```

### 五、底层原理补充：wait() + notify() 的完整协作流程
理解 notify 的底层，必须结合 wait()，完整流程如下：
1. 线程A持有对象锁，调用 `obj.wait()` → 释放锁 → 进入 Wait Set → 状态变为 WAITING。
2. 线程B获取对象锁，执行业务逻辑后调用 `obj.notify()` → JVM 将 Wait Set 中一个线程（如A）移到 Entry Set → A的状态变为 BLOCKED。
3. 线程B释放锁（退出 synchronized）→ Entry Set 中的线程竞争锁 → 线程A拿到锁后，从 `wait()` 方法返回，继续执行后续代码。

### 总结
1. **核心定位**：`notify()`/`notifyAll()` 是 Java 线程通信的核心 API，用于唤醒阻塞在对象监视器上的线程，必须与 `wait()` 配合使用。
2. **底层核心**：依赖对象监视器（Monitor）的「等待队列/入口队列」机制，底层调用操作系统的条件变量和互斥量，`native` 方法由 JVM 优化实现。
3. **使用关键**：
    - 必须在 `synchronized` 中调用（持有锁）；
    - `notify()` 适合「单线程处理」场景，`notifyAll()` 适合「多线程需检查条件」场景；
    - 被唤醒线程需重新竞争锁，且无执行特权。

