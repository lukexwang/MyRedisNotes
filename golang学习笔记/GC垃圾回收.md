### Go语言GC原理

文章:

- [Go语言GC实现原理及源码分析](https://www.luozhiyun.com/archives/475)
- [图文解析Golang GC、三色标记、混合写屏障](https://juejin.cn/post/7040737998014513183)

**什么是GC?**

GC(Garbage Collection,缩写为GC),是一种自动内存管理机制。

即我们在程序中定义一个变量后，会在内存中开辟相应空间进行存储。当不需要此变量后，需要手动销毁此对象，并释放内存。而这种对不再使用的内存资源进行自动回收的功能即为垃圾回收。

**GC相关术语**

- **赋值器**: 在程序的执行过程中，可能会改变对象的引用关系，或者创建新的引用;

- **回收器**: 垃圾回收器的责任就是去干掉那些程序中不再被引用得对象;

- **STW**: 全称为 stop the word, GC期间某个阶段会停止所有的赋值器，中断你的程序逻辑，以确定引用关系。

- ROOT对象

  : 根对象是指赋值器不需要通过其他对象就可以直接访问到的对象，通过Root对象, 可以追踪到其他存活的对象。常见root对象:

  - **全局变量**: 程序在编译期就能确定的那些存在于程序整个生命周期的变量;
  - **执行栈**: 每个goroutine (包括main函数)都拥有自己的执行栈，**这些执行栈上包含栈上的变量及堆内存指针;**

**三色标记法**

1. 初始时，所有对象被标记为白色;
2. GC开始，遍历ROOT对象，将可达的对象标记为灰色;
3. 遍历灰色对象，将直接可达对象标记为灰色，并将自身标记为黑色；
4. 重复第3步，直到标记完所有对象；
5. 将剩余依然是白色的对象当做 垃圾回收掉；

**三色标记法的问题：**

- 多标-浮动垃圾问题

E/F/G应该被护手的，但是因为E已经变成了灰色，其仍会被当做存活对象继续遍历下去。所以这部分对象在本轮GC中不会被回收。

![image-20220315164235488](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220315164235488.png)

- 漏标-悬挂指针问题

E变成了灰色，D变成了黑色，灰色E断开白色G的引用，黑色D引用了白色G。

此时GC线程继续跑，因为E已经没有对G的引用了，所以不会将G放到灰色集合。

尽管因为D重新引用了G，但因为D是黑色，不会再做重新遍历处理。

G一直停留在白色集合中，组合会被当做垃圾进行清除。

![image-20220315164300322](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220315164300322.png)

**内存屏障**

**内存屏障，是一种屏障指令，它能使CPU 或 编译器对该屏障指令之前和之后发出的内存操作强制执行排序约束，在内存屏障前执行的操作一定会先于内存屏障后执行。**

为了在标记算法中保证正确性，那么我们需要达成下面任意一个条件：

- **强三色不变性（strong tri-color invariant）：黑色对象不会指向白色对象，只会指向灰色对象或者黑色对象；**
- **弱三色不变性（weak tri-color invariant）：即便黑色对象指向白色对象，那么从灰色对象出发，总存在一条可以找到该白色对象的路径。白色对象的上游必须存在灰色对象;**

![image-20220315164543560](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220315164543560.png)

**屏障机制**

Golang团队遵循上述两种不变式提到的原则，分别提出了两种实现机制: **插入写屏障 和 删除写屏障。**

**插入写屏障:**

规则: 当一个对象引用另外一个对象时，将另一个对象标记为灰色。(如果黑色对象引用白色对象，白色对象马上会变成灰色)

注意: 插入屏障仅会在对内存中生效，不对栈内存空间生效。因为go在并发运行时，大部分的操作都在栈上，函数调用非常频繁。goroutine的栈都进行屏障保护自然会有性能问题。

![image-20220315164638989](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220315164638989.png)

![image-20220315164714204](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220315164714204.png)

对于插入写屏障来讲，道友们需记住，插入写屏障最大的弊端就是，在一次正常的三色标记流程结束后，需要对栈上重新进行一次stw，然后再rescan一次。

**删除写屏障**

规则: 删除引用时，如果被删除引用的对象自身是灰色 或者 白色，那么被标记为灰色。

> 满足弱三色不变式。灰色对象到白色对象的路径不会断

![image-20220315165151479](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220315165151479.png)

**但是引入删除写屏障，有一个弊端，就是一个对象的引用被删除后，即使没有其他存活的对象引用它，它仍然会活到下一轮。如此一来，会产生很多的冗余扫描成本。**

#### V1.8 混合写屏障机制

- **GC刚开始的时候，会将栈上的可达对象全部标记为黑色**;

- **GC期间，任何在栈上新创建的对象，均为黑色**;

  >上面两点只有一个目的，将栈上的可达对象全部标黑，最后无需对栈进行STW，就可以保证栈上的对象不会丢失。有人说，一直是黑色的对象，那么不就永远清除不掉了么，这里强调一下，标记为黑色的是可达对象，不可达的对象一直会是白色，直到最后被回收。

- **堆上被删除的对象标记为灰色**

- **堆上新添加的对象标记为灰色**