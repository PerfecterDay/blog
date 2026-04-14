# HotSpot虚拟机对象探秘
{docsify-updated}

## 对象的创建
当Java虚拟机遇到一条字节码new指令时，首先将去检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已被加载、解析和初始化过。如果没有，那必须先执行相应的类加载过程。

在类加载检查通过后，接下来虚拟机将为新生对象分配内存。对象所需内存的大小在类加载完成后便可完全确定，为对象分配空间的任务实际上便等同于把一块确定大小的内存块从Java堆中划分出来。  
### 指针碰撞
假设Java堆中内存是绝对规整的，所有被使用过的内存都被放在一边，空闲的内存被放在另一边，中间放着一个指针作为分界点的指示器，那所分配内存就仅仅是把那个指针向空闲空间方向挪动一段与对象大小相等的距离，这种分配方式称为“指针碰撞”（Bump The Pointer）。

### 空闲列表
但如果Java堆中的内存并不是规整的，已被使用的内存和空闲的内存相互交错在一起，那就没有办法简单地进行指针碰撞了，虚拟机就必须维护一个列表，记录上哪些内存块是可用的，在分配的时候从列表中找到一块足够大的空间划分给对象实例，并更新列表上的记录，这种分配方式称为“空闲列表”（Free List）。

选择哪种分配方式由Java堆是否规整决定，而Java堆是否规整又由所采用的垃圾收集器是否带有空间压缩整理（Compact）的能力决定。因此，当使用 `Serial` 、 `ParNew` 等带压缩整理过程的收集器时，系统采用的分配算法是指针碰撞，既简单又高效；而当使用 `CMS` 这种基于清除（Sweep）算法的收集器时，理论上就只能采用较为复杂的空闲列表来分配内存。

### 同步问题
除如何划分可用空间之外，还有另外一个需要考虑的问题：对象创建在虚拟机中是非常频繁的行为，即使仅仅修改一个指针所指向的位置，在并发情况下也并不是线程安全的，可能出现正在给对象A分配内存，指针还没来得及修改，对象B又同时使用了原来的指针来分配内存的情况。解决这个问题有两种可选方案：
1. 一种是对分配内存空间的动作进行同步处理——实际上虚拟机是采用CAS配上失败重试的方式保证更新操作的原子性
2. 另外一种是把内存分配的动作按照线程划分在不同的空间之中进行，即每个线程在Java堆中预先分配一小块内存，称为本地线程分配缓冲（Thread Local Allocation Buffer，TLAB），哪个线程要分配内存，就在哪个线程的本地缓冲区中分配，只有本地缓冲区用完了，分配新的缓存区时才需要同步锁定。虚拟机是否使用TLAB，可以通过 `-XX：+/-UseTLAB` 参数来设定。

内存分配完成之后，虚拟机必须将分配到的内存空间（但不包括对象头）都初始化为零值，如果使用了TLAB的话，这一项工作也可以提前至TLAB分配时顺便进行。这步操作保证了对象的实例字段在Java代码中可以不赋初始值就直接使用，使程序能访问到这些字段的数据类型所对应的零值。

接下来，Java虚拟机还要对对象进行必要的设置，例如这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码（实际上对象的哈希码会延后到真正调用 `Object::hashCode()` 方法时才计算）、对象的GC分代年龄等信息。这些信息存放在对象的**对象头**（Object Header）之中。根据虚拟机当前运行状态的不同，如是否启用偏向锁等，对象头会有不同的设置方式。

在上面工作都完成之后，从虚拟机的视角来看，一个新的对象已经产生了。但是从Java程序的视角看来，对象创建才刚刚开始——构造函数，即Class文件中的 `<init>()` 方法还没有执行，所有的字段都为默认的零值，对象需要的其他资源和状态信息也还没有按照预定的意图构造好。一般来说（由字节码流中new指令后面是否跟随 `invokespecial` 指令所决定，Java编译器会在遇到new关键字的地方同时生成这两条字节码指令，但如果直接通过其他方式产生的则不一定如此），new指令之后会接着执行 `<init>()` 方法，按照程序员的意愿对对象进行初始化，这样一个真正可用的对象才算完全被构造出来。

```
CASE(_new): {
        u2 index = Bytes::get_Java_u2(pc+1);

        // Attempt TLAB allocation first.
        //
        // To do this, we need to make sure:
        //   - klass is initialized
        //   - klass can be fastpath allocated (e.g. does not have finalizer)
        //   - TLAB accepts the allocation
        ConstantPool* constants = istate->method()->constants();
        if (UseTLAB && !constants->tag_at(index).is_unresolved_klass()) {
          Klass* entry = constants->resolved_klass_at(index);
          InstanceKlass* ik = InstanceKlass::cast(entry);
          if (ik->is_initialized() && ik->can_be_fastpath_allocated()) {
            size_t obj_size = ik->size_helper();
            HeapWord* result = THREAD->tlab().allocate(obj_size);
            if (result != nullptr) {
              // Initialize object field block.
              if (!ZeroTLAB) {
                // The TLAB was not pre-zeroed, we need to clear the memory here.
                size_t hdr_size = oopDesc::header_size();
                Copy::fill_to_words(result + hdr_size, obj_size - hdr_size, 0);
              }

              // Initialize header, mirrors MemAllocator.
              if (UseCompactObjectHeaders) {
                oopDesc::release_set_mark(result, ik->prototype_header());
              } else {
                oopDesc::set_mark(result, markWord::prototype());
                if (oopDesc::has_klass_gap()) {
                  oopDesc::set_klass_gap(result, 0);
                }
                oopDesc::release_set_klass(result, ik);
              }
              oop obj = cast_to_oop(result);

              // Must prevent reordering of stores for object initialization
              // with stores that publish the new object.
              OrderAccess::storestore();
              SET_STACK_OBJECT(obj, 0);
              UPDATE_PC_AND_TOS_AND_CONTINUE(3, 1);
            }
          }
        }
        // Slow case allocation
        CALL_VM(InterpreterRuntime::_new(THREAD, METHOD->constants(), index),
                handle_exception);
        // Must prevent reordering of stores for object initialization
        // with stores that publish the new object.
        OrderAccess::storestore();
        SET_STACK_OBJECT(THREAD->vm_result_oop(), 0);
        THREAD->set_vm_result_oop(nullptr);
        UPDATE_PC_AND_TOS_AND_CONTINUE(3, 1);
}
```