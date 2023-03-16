参考： https://docs.sel4.systems/Tutorials/capabilities.html

## What's a capability

> A _capability_ is a unique, unforgeable token that gives the possessor permission to access an entity or object in system.

按照教程定义，可以理解为 `capability` 是访问系统中一切实体或对象的令牌，这个令牌包含了某些访问权力，只有持有这个令牌，才能按照特定的方式访问系统中的某个对象。教程中还提到了：

> One way to think of a capability is as a pointer with access rights

可以将 `capability` 视为一个拥有访问权限的指针，而在C语言中，指针常常可以共享，因此个人理解这里隐含着一层 `capability`  可以传递和分享的意思。

`seL4` 中将 `capability`  分做三种：
- 内核对象（如 线程控制块）的访问控制的 `capability`。
- 抽象资源（如 `IRQControl` ）的访问控制的 `capability` 。（中断控制块是干啥的，以后会提到）
- `untyped capabilities` ，负责内存分配的 `capabilities` 。`seL4` 不支持动态内存分配，在内核加载阶段将所有的空闲内存绑定到 `untyped capabilities` 上，后续的内核对象申请新的内存，都通过系统调用和 `untyped caability` 来申请。

在 `seL4` 内核初始化时，指向内核控制资源的所有的 `capability` 全部被授权给特殊的任务 `root task` ，用户代码向修改某个资源的状态，必须使用 `kernel API` 在指定的 `capability` 上请求操作。

> 关于 Root Task，有几个概念上的疑惑问答，参考：[About Root Task in seL4](https://sel4.discourse.group/t/about-root-task-in-sel4/624)

例如，`root task` 拥有控制它自己的进程控制块（TCB）的 `capability` ，这个 `capability` 被定义为一个常量 `seL4_CapInitThreadTCB` ，下面的一个例子就是通过这个 `capability` 来改变当前当前进程任务的栈顶指针（当需要一个很大的用户栈时，这个操作非常有用）：
```C
    seL4_UserContext registers;
    seL4_Word num_registers = sizeof(seL4_UserContext)/sizeof(seL4_Word);

    /* Read the registers of the TCB that the capability in seL4_CapInitThreadTCB grants access to. */
    seL4_Error error = seL4_TCB_ReadRegisters(seL4_CapInitThreadTCB, 0, 0, num_registers, &registers);
    assert(error == seL4_NoError);

    /* set new register values */
    registers.sp = new_sp; // the new stack pointer, derived by prior code.

    /* Write new values */
    error = seL4_TCB_WriteRegisters(seL4_CapInitThreadTCB, 0, 0, num_registers, &registers);
    assert(error == seL4_NoError);
```

主要是 `seL4_TCB_ReadRegisters` , `seL4_TCB_WriteRegisters` 两个接口。下面将介绍 `capability` 的主要概念。

## CNodes and CSlots
	
教程给出的定义：

> A _CNode_ (capability-node) is an object full of capabilities: you can think of a CNode as an array of capabilities.

简单来说， `CNode` 也是一个对象，但这个对象存的是一个数组，数组里的元素是 `capability` 。数组中的各个位置我们称为 `CSlots` ，在上面的例子中， `seL4_CapInitThreadTCB` 实际上就是一个 `CSlot` ，这个 `CSlot` 中存的是控制 `TCB` 的 `capability` 。

每个 `Slot` 有两种状态：
- `empty`：没有存 `capability`。
- `full`：存了 `capability`。

出于习惯，0号 `Slot` 一直为 `empty`。

一个 `CNode` 有 `1 << CNodeSizeBits` 个 `Slot`，一个 `Slot` 占 `1 << seL4_SlotBits` 字节。

## CSpaces

教程给出的定义：

> A _CSpace_ (capability-space) is the full range of capabilities accessible to a thread, which may be formed of one or more CNodes.

值得注意的是， `CSpace` 是一个 `线程` 的能力空间，教程里只讨论了一个 `CSpace` 只有一个 `CNode` 的普遍情况。

## CSpace addressing

要在 `capability` 上进行相关操作，在之前需要先指定对应的 `capability`。 `seL4` 提供了两种指定 `capability` 的方式： `Invocation` 和 `Direct addressing` 。

### Invocation

每个线程有在 `TCB` 中装载了一个特殊的 `CNode` 作为它 `CSpace` 的 `root`。这个 `root` 可以为空（代表这个线程没有被赋予任何 `capability` ）。

在 `Invocation` 方式中，我们通过隐式地调用线程的 `CSpace root` 来寻址 `CSlot` 。例如：我们使用对 `seL4_CapInitThreadTCB` `CSlot` 的调用来读取和写入由该特定 `CSlot` 中的功能表示的 `TCB` 的寄存器。

```c
seL4_TCB_WriteRegisters(seL4_CapInitThreadTCB, 0, 0, num_registers, &registers);
```

这个模式会在 `CSpace root` 中隐式地查找 `seL4_CapInitThreadTCB Slot` 。

### Direct CSpace addressing

与 `Invocation` 默认在 `CSpace root` 中查找不同，你可以指定你要在哪个 `CNode` 中查找。这种操作主要用于构建和操作 `CSpace` 的形状（可能是另一个线程的 `CSpace` ）。

`Direct Addressing` 一般需要以下几个参数：
- `_server/root`  需要操作的 `capability` 所在的 `CNode`。
- `index`  需要操作的 `Slot` 在 `CNode` 中的序号。
- `depth`  在定位到 `Slot` 之前遍历 `CNode` 的距离（？）。

单层的 `CSpace` 的 `depth` 一直是 `seL4_WordBits` 。

下面的例子中直接定位了 `root task` 的 `TCB` ，然后在 `CSpace root` 的第 0 个 `Slot` 中复制它。

```c
    seL4_Error error = seL4_CNode_Copy(
        seL4_CapInitThreadCNode, 0, seL4_WordBits, // destination root, slot, and depth
        seL4_CapInitThreadCNode, seL4_CapInitThreadTCB, seL4_WordBits, // source root, slot, and depth
        seL4_AllRights);
    assert(error == seL4_NoError);
```

在上面的例子中， `seL4_CapInitThreadCNode` 既是源 `CNode`， 也是目标 `CNode`，`seL4_CapInitThreadTCB` 指向了控制 `TCB` 的 `capability` ，上面的函数就是把 `TCB` 的 `capability` 复制到 0 号 `Slot`。

## Initial CSpace

`root task` 也有一个 `CSpace`， 在 `boot` 阶段被设置，包含了被 `seL4` 管理的所有资源的 `capability` 。 我们之前用过的控制 `TCB` 的 `seL4_CapInitThreadTCB` 和 `CSpace root` 对应的 `seL4_CapInitThreadCNode` ，都在 `libsel4` 中被静态定义为常量，但并不是所有的 `capability` 都被静态指定，其他 `capability` 有 `seL4_BootInfo` 这个数据结构来描述。 `seL4_BootInof` 描述了初始的所有 `capability`的范围，包括了初始的 `CSpace` 中的可用的 `Slot` ：

```c
typedef struct seL4_BootInfo {
	seL4_Word extraLen; /* length of any additional bootinfo information */
	seL4_NodeId nodeID; /* ID [0..numNodes-1] of the seL4 node (0 if uniprocessor) */
	seL4_Word numNodes; /* number of seL4 nodes (1 if uniprocessor) */
	seL4_Word numIOPTLevels; /* number of IOMMU PT levels (0 if no IOMMU support) */
	seL4_IPCBuffer *ipcBuffer; /* pointer to initial thread's IPC buffer */
	seL4_SlotRegion empty; /* empty slots (null caps) */
	seL4_SlotRegion sharedFrames; /* shared-frame caps (shared between seL4 nodes) */
	seL4_SlotRegion userImageFrames; /* userland-image frame caps */
	seL4_SlotRegion userImagePaging; /* userland-image paging structure caps */
	seL4_SlotRegion ioSpaceCaps; /* IOSpace caps for ARM SMMU */	
	seL4_SlotRegion extraBIPages; /* caps for any pages used to back the additional bootinfo information */
	seL4_Word initThreadCNodeSizeBits; /* initial thread's root CNode size (2^n slots) */
	seL4_Domain initThreadDomain; /* Initial thread's domain ID */
#ifdef CONFIG_KERNEL_MCS
	seL4_SlotRegion schedcontrol; /* Caps to sched_control for each node */
#endif
	seL4_SlotRegion untyped; /* untyped-object caps (untyped caps) */
	seL4_UntypedDesc untypedList[CONFIG_MAX_NUM_BOOTINFO_UNTYPED_CAPS]; /* information about each untyped */
	/* the untypedList should be the last entry in this struct, in order
	* to make this struct easier to represent in other languages */
} seL4_BootInfo;
```

## Exercises

在实验中，`main.c` 被编译进 `seL4` 项目中，生成 `root task`。我们需要在 `main.c` 中对 `root task` 的 `capability` 进行一些操作练习。

### How big is your CSpace?

计算出初始的 `CSpace` 需要多少个字节。

初始的 `CSpace` 只有一个 `CNode`，我们只需要算出 `CNode` 中有多少个 `Slot` ，然后一个 `Slot` 占几个字节就可以了。

```c
// 从默认crt0.S设置的环境变量中解析出seL4_BootInfo数据结构的位置
seL4_BootInfo *info = platsupport_get_bootinfo();

// 计算出初始的 cnode 有多少个 slot
size_t initial_cnode_object_size = BIT(info->initThreadCNodeSizeBits)

// 计算这些 slot 的总共的大小就是 cnode 的大小
size_t initial_cnode_object_size_byte = initial_cnode_object_size * (1u << seL4_SlotBits);
```

### Copy a capability between CSlots

将 `root task` 的操作 `TCB` 的 `capability` 复制到 最后一个空闲的 `Slot` 上。

我们借助 `seL4_CNode_Copy` 接口，从 `BootInfo` 中拿到最后一个空闲 `Slot` 的位置：
```c
// 拿到最后一个空闲的 Slot 的位置
seL4_CPtr last_slot = info->empty.end - 1;

// 复制 capability
error = seL4_CNode_Copy(seL4_CapInitThreadCNode, last_slot, seL4_WordBits,
						seL4_CapInitThreadCNode, seL4_CapInitThreadTCB, seL4_WordBits, seL4_AllRights);
```

### How do you delete capabilities?

将之前拷贝的 `Slot` 清空。

这里提供了两个接口：
- `seL4_CNode_Revoke(seL4_CNode _service, seL4_Word index, seL4_Uint8 depth)` 
- `seL4_CNode_Delete(seL4_CNode _service, seL4_Word index, seL4_Uint8 depth)`

这两个接口的函数参数相同，但用法不同。 `seL4_CNode_Revoke` 作用于原始的 `capability` ，删除从原始派生出去的其他 `capability` 。而 `seL4_CNode_Delete` 作用于需要删除的 `capability` 。

如果要使用 `seL4_CNode_Revoke` 的话：
```c
seL4_CNode_Revoke(seL4_CapInitThreadCNode, seL4_CapInitThreadTCB, seL4_WordBits);
```

如果要使用 `seL4_CNode_Delete` 的话：
```c
seL4_CNode_Delete(seL4_CapInitThreadCNode, first_free_slot, seL4_WordBits);
seL4_CNode_Delete(seL4_CapInitThreadCNode, last_slot, seL4_WordBits);
```

可以看出，如果原始的 `capability` 派生复制了很多的 `capability`，如果 `seL4_CNode_Delete` 删除的话，要作用于每一个复制的 `cap`，会十分不便，因此 `seL4_CNode_Revoke` 是更好的选择。

### Invoking capabilities

使用 `seL4_TCB_Suspend` 来暂停当前线程。

这里考察我们使用 `Invocation` 的方式来操作 `capability` ，在 `seL4_TCB_Suspend` 接口中，会默认使用 `CSpace root` 作为查找的节点，因此只需要传入对应的 `Slot` 即可：

```c
seL4_TCB_Suspend(seL4_CapInitThreadTCB);
```
