### linux内核知识点记录

####1.memery barried(内存屏蔽)

首先需要了解乱序执行。乱序是指cpu最后执行代码的顺序不是按照程序中代码书写的顺序执行。

```
// 代码书写顺序
int a = 0; 
int c = 1; 

// cpu执行顺序可能为
int c = 1;
int a = 0;
```

乱序执行的原因:
* 1.编译器优化导致代码顺序改变。
* 2.cpu乱序执行代码。(cpu流水线导致指令被分为几个并行阶段)

当然这并不是我们就应该担忧乱序的问题。毫无关联代码的乱序执行，并不会影响最后的结果。其次在大多数情况下，存在显性因果关系的代码。编译器和cpu都会处理，不会让这种乱序发生。

```
// 只有a被赋值，b才会被赋值！
// 不会被乱序执行！
int *a = &p;
int *b = &a;
```

但如果是隐性的因果关系了？

```
// 看起来没有任何关系的两句代码。但如果addr 代表了一个设备的端口
// data代表了写入该设备的数据。可想而知。这两句代码是不能乱序执行的
int *addr = &p;
int *data = &d;
```

所以才会有内存屏蔽出现，来避免这种情况的发生。代码中出现内存屏蔽。表示之前的代码一定优先于之后的代码执行。但内存屏蔽同侧则顺序不保证

```
// mb()之前的代码，一定先于mb()之后的执行。但tmp和addr的赋值顺序不保证！
int tmp = a;
int *addr = &p;
mb();
int *data = &d;
``` 

内存屏蔽分为读屏蔽，写屏蔽，通用屏蔽，优化屏蔽等。通用屏蔽包括了读写屏蔽。

我们再看SMP,对称多处理器，需不需要内存屏蔽！前面已经说了单个cpu需要用内存屏蔽来保证。那么SMP中，如果CPU-1使用了内存屏蔽。那么CPU-2看到CPU-1的内存操作一定是有序的。那么CPU-2是不是就不需要再使用内存屏蔽了？对于大多数系统来说。确实不必要！但某些cpu是需要的。

```
CPU-1 						CPU-2
int data = 2; 				if(ok) {
int ok = false; 				int a = data;
mb() 						}
bool ok = false;
```

CPU-1按照顺序发出了写data和写ok的指令。但如果CPU使用了分列cache。导致cache可以并行操作。则虽然CPU-1的指令有序的发出。但实际中CPU-2.因为data缓存比较繁忙，而先更新了ok缓存。所以CPU-2 也需要使用匹配的内存屏蔽.

```
CPU-1 						CPU-2
int data = 2; 				if(ok) {
int ok = false; 				rmb();
wmb() 							int a = data;	
bool ok = false; 			}
```