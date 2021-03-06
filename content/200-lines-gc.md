有的人觉得垃圾回收很复杂。其实实现一个基础的垃圾回收算法一点都不难的。正好刚刚写过，就顺便说一下，如何在 200 行代码以内实现垃圾回收。

实现垃圾回收的第一个难点是，如何确定 root 区域。从 root 引用着的对象，都是活跃对象，不应该被回收掉。root 一般包括当前的程序的栈，全局变量等。确认 root 的做法，一种是对编译器有侵入的，就是语言和编译器结合考虑的话，比如在编译器层做过活跃对象分析。另一种是语言设计在一层抽象虚拟机之上的，通过虚拟机层确定 root 区域，这都脱离了本文主题。那么剩下的唯一可行的，就只有一种较笨拙的方式：扫描整个栈区域。

栈的布局大致上是这样子的：

```
caller stack base addr
    local1
    local2
    ...
    args1
    args2
callee stack base addr
    local1
    local2
    ...
```

栈的增长方向向下。我们可以通过取函数的参数的地址，或者是局部变量地址，来获取函数的当前的栈的位置：

```C
void f() {
    uintptr_t dummp;
    void* stackbase = &dummp;
}
```

在 main 里面获取一次，再地调用到 GC 的函数里面再取一次，就可以得到整个程序的栈的区域了(暂不考虑多线程)。

```C
static void* stacktop;
int main(int argc, char *argv) {
    uintptr_t dummy;
    stacktop = &dummy;
    f();
}
void f() {
    gc();
}
void gc() {
    void *stackbase = &stackbase;
    // 栈区域就是 [stackbase, stacktop]，因为栈增长方向是向下的
}
```

用扫描栈的方式去获取 root，则会遇到这个栈里面并不是每个对象都是从托管的内存中分配出来的。
我们把栈上的每一个对象都假设成指向堆的指针，一个大整数也会被误当作指针，那样就不是一个精确的垃圾回收。
如何实现精确垃圾回收的另一个难点。这里有些策略但是先不展开了。还有对象可能在寄存器上面也会有影响。

有一个小细节是注意分配器分配出来的内存做对齐，比如按 64 位对齐过。对齐过我们在做栈扫描的时候，就只需要按机器字长一个一个扫描栈。如果没做对齐，那就得按 byte 扫描栈了。这个效率上是有许多倍的差异的。

原本我是准备实现 mark-sweep 的垃圾回收器的，不过如果用 mark-sweep 实现会遇到几个难处理的地方。

一个是 mark-sweep 长期运行后会有内存碎片问题。

另外，为了不误报栈扫描的信息，至少需要确认一个 uintptr_t 是：

1. 堆上分配出去的
2. 并且是由我们托管的内存堆上分配出去的

因为如果不是从堆分配出去的，那么指针解引用时可能会直接 panic 了。解决这个有一个方式是用一个 map 存所有从托管内存分配的对象的指针。这个做法太丑了，空间利用率非常低，并且每次判断是不是精确指针，性能也有问题。

无论是判断是否由托管堆分配出去的内存，还是处理碎片问题，mark-sweep 最后解决方式必须自己定制内存分配器，内存分配和垃圾回收强耦合，由内存分配器判断一个指针是不是从自己分配出来的，以及让内存分配器做更好的优化去处理碎片问题。

自己实现内存分配器复杂度过高了，会超过 200 行代码，并且内存分配器和垃圾回收是两件事情，最好不要强耦合，所以这个方法就脱离本文的话题了。

所以我要写的是基于 copy 回收算法，而不是 mark-sweep。

copy 的回收算法思想很简单，有两块内存区域，一块是使用中的，一块是备用的。分配的时候直接从使用中的区域里面划出去，让 offset 加加就可以了。执行垃圾回收的时候，通过扫描，将使用中的活跃对象全部挪到另一块内存中，然后对调一下，让新内存块区域成为使用中的，而回收完的那块老内存区域就成为备用的。

```C
struct Area {
  int offset;
  char *data;
};

struct GC {
  struct Area area1;
  struct Area area2;
  struct Area *curr;
};
```

因为对象之间的引用关系，从一个地方挪到另一个地方，是要更新引用的。copy 算法的实现难点，就变成了另一个问题：如何将一个图数据结构，复制到另一个位置。我们可以在节点里面设置一个 forwarding 指针。

1. 先把节点自身拷贝到新的位置
2. 然后更新原节点的 forwarding 指针，让它指向新的节点位置，意思是说，这个节点已经在拷贝过程中了
3. 然后，递归的把节点的 children 拷贝到新的位置
4. 更新新节点的 children 字段，让它指向新的 children 节点


```C
void*
gcCopy(struct GC *gc, scmHead* p) {
  // Copy the data of itself to the new place.
  void* to = gcAlloc(gc, p->size);
  memcpy(to, p, p->size);

  // Update the forwarding.
  p->forwarding = to;

  // Copy the children to the new place.
  // And update the reference of the new object.
  gcCopyFunc gcCopyChildren = getGCFunc(p->type);
  if (gcCopyChildren != NULL) {
    gcCopyChildren(p, to, gc);
  }

  // Return the new addr for node p.
  return to;
}
```

在 forwarding 的处理上面，需要注意一下成环问题。在递归拷一个节点的时候，遇到 forwarding 指针不为 NULL，可以跳过这个节点。如果在更新 children 的时候，可以直接更新为 forwarding 指向的新节点。

另外，递归写法其实要注意一下递归深度，可能会有栈的 OOM 问题。

核心的算法讲完了，剩下是一些零零碎碎的。比如说接口

```
void* gcAlloc(struct GC* gc, int size);
void gcRun()
void gcRegist(struct GC *gc, *uint8_t type, gcFunc fn);
```

alloc 接口不用多解释，从托管内存里面分配都走这里。这里有一个 gcRegist 接口是干嘛的呢？

我们可以通过 `gcRegist` 注册需要 gc 的对象类型，参数是一个类型的 id，以及一个这个类型的对象的回收方法，也就是前面 `gcCopy` 函数里面的 `gcCopyChildren`。一个对象只要提供了把自己的 children 挪到新的那块内存的方法，垃圾回收的时候就知道怎么样处理这一类的对象。

比如，Cons，就是把 children 拷过去，再更新一下引用关系：

```
struct Cons {
    Obj car;
    Obj cdr;
};

static void
consGCFunc(struct GC *gc, void* f, void* t) {
  struct Cons *from = f;
  struct Cons *to = t;
  to->car = gcCopy(gc, from->car);
  to->cdr = gcCopy(gc, from->cdr);
}
```

最后，无码言屌，来来来，[完整的地址](https://github.com/tiancaiamao/cora/blob/ad0daa17c020ed2f4358629b4ca420f32fdd516b/runtime/gc.c#L154)。

哦，出一道小题目。**一个对象，它知道它引用了哪些对象**，所以它可以递归的把它引用的对象挪到新的位置，然后更新自己的引用，这种场景可以处理。
**但是一个对象，它并不知道哪些对象引用了它**，那么如果它自己挪走了，怎么更新那些指向它的引用呢？
