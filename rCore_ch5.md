---
title: 'rCore-ch5'
date: 2024/1/15
permalink: /posts/2024/01/rCore_ch5/
excerpt: '第五章 进程'
tags:
  - rCore
  - OS
---
# 第五章  进程
## 1. 进程概念
进程 就是操作系统选取某个可执行文件并对其进行一次动态执行的过程， 对于可执行文件中给出的需求能相应对 硬件/虚拟资源 进行 动态绑定和解绑。 进程的引入让开发者能够控制程序的运行， 操作系统能够面向用户进行交互。 rCore 增添了操作系统中最为关键的几个系统调用： sys_fork， sys_waitpid， sys_exec， 以及 sys_read。 同样的， 通过引入 buddy_system_allocator 提供动态内存分配支持， 结合这些系统调用， 在用户态 rCore 提供了一个用户初始程序 initproc， 这让 rCore 具备了与应用层的交互能力， 但其背后则需依托 rCore 的进程管理。
## 2. 进程管理的组成结构
### 2.1 进程控制块(PCB)
在本章中，新增**进程控制块**  (PCB,Process Control Block)，存放每个进程的执行状态、资源控制等元数据。它是内核对进程进行管理的单位，故而是一种极其关键的内核数据结构。  
承接前面的章节，我们仅需对任务控制块 `TaskControlBlock` 进行若干改动并让它直接承担进程控制块的功能：
``` rust
// os/src/task/task.rs

pub struct TaskControlBlock {
    // immutable
    pub pid: PidHandle,
    pub kernel_stack: KernelStack,
    // mutable
    inner: UPSafeCell<TaskControlBlockInner>,
}

pub struct TaskControlBlockInner {
    pub trap_cx_ppn: PhysPageNum,
    pub base_size: usize,
    pub task_cx: TaskContext,
    pub task_status: TaskStatus,
    pub memory_set: MemorySet,
    pub parent: Option<Weak<TaskControlBlock>>,
    pub children: Vec<Arc<TaskControlBlock>>,
    pub exit_code: i32,
}
```
作为对比，在这里附上前面章节的TCB结构：
```rust
pub struct TaskControlBlock {
    /// Save task context
    pub task_cx: TaskContext,
    /// Maintain the execution status of the current process
    pub task_status: TaskStatus,
    /// Application address space
    pub memory_set: MemorySet,
    /// The phys page number of trap context
    pub trap_cx_ppn: PhysPageNum,
    /// The size(top addr) of program which is loaded from elf file
    pub base_size: usize,
    /// Heap bottom
    pub heap_bottom: usize,
    /// Program break
    pub program_brk: usize,
    /// start_time
    pub start_time: usize,
    /// syscall_times
    pub syscall_times: [u32;MAX_SYSCALL_NUM],
}
```

任务控制块中包含两部分：

在初始化之后就不再变化的元数据：直接放在任务控制块中。这里将进程标识符 `PidHandle` 和内核栈 `KernelStack` 放在其中；

在运行过程中可能发生变化的元数据：则放在 `TaskControlBlockInner` 中，将它再包裹上一层 `UPSafeCell<T>` 放在任务控制块中。这是因为在我们的设计中外层只能获取任务控制块的不可变引用，若想修改里面的部分内容的话这需要 `UPSafeCell<T>`所提供的内部可变性。  
注意我们在维护父子进程关系的时候大量用到了引用计数 `Arc/Weak` 。进程控制块的本体是被放到内核堆上面的，对于它的一切访问都是通过智能指针 `Arc/Weak` 来进行的，这样是便于建立父子进程的双向链接关系（避免仅基于 `Arc` 形成环状链接关系）。当且仅当智能指针 `Arc` 的引用计数变为 `0` 的时候，进程控制块以及被绑定到它上面的各类资源才会被回收。子进程的进程控制块并不会被直接放到父进程控制块中，因为子进程完全有可能在父进程退出后仍然存在。    

`TaskControlBlockInner` 提供的方法主要是对于它内部的字段的快捷访问：

```rust
// os/src/task/task.rs

impl TaskControlBlockInner {
    pub fn get_trap_cx(&self) -> &'static mut TrapContext {
        self.trap_cx_ppn.get_mut()
    }
    pub fn get_user_token(&self) -> usize {
        self.memory_set.token()
    }
    fn get_status(&self) -> TaskStatus {
        self.task_status
    }
    pub fn is_zombie(&self) -> bool {
        self.get_status() == TaskStatus::Zombie
    }
}
```
而任务控制块 `TaskControlBlock` 目前提供以下方法：
```rust
// os/src/task/task.rs

impl TaskControlBlock {
    pub fn inner_exclusive_access(&self) -> RefMut<'_, TaskControlBlockInner> {
        self.inner.exclusive_access()
    }
    pub fn getpid(&self) -> usize {
        self.pid.0
    }
    pub fn new(elf_data: &[u8]) -> Self {...}
    pub fn exec(&self, elf_data: &[u8]) {...}
    pub fn fork(self: &Arc<TaskControlBlock>) -> Arc<TaskControlBlock> {...}
}
```

* `inner_exclusive_access` 通过 `UPSafeCell<T>.exclusive_access()` 来得到一个 `RefMut<'_, TaskControlBlockInner>` ，它可以被看成一个内层 `TaskControlBlockInner` 的可变引用并可以对它指向的内容进行修改。

* `getpid` 以 `usize` 的形式返回当前进程的进程标识符。

* `new` 用来创建一个新的进程，目前仅用于内核中手动创建唯一一个初始进程 `initproc` 。

* `exec` 用来实现 `exec` 系统调用，即当前进程加载并执行另一个 `ELF` 格式可执行文件。

* `fork` 用来实现 `fork` 系统调用，即当前进程 `fork` 出来一个与之几乎相同的子进程。

>这里是我比较费解的地方，一开始没有注意到，在编程中复用ch4的mmap和munmap出现了问题，反复对比源码和查阅文档发现是TCB是被修改过的，不能像之前那样访问成员。

```rust
    ///ch5: mmap
    pub fn mmap(&self,start: usize, len: usize, port: usize)->isize{
        let mut inner=self.inner.exclusive_access();
        let start_va=mm::VirtAddr::from(start);
        let end_va=mm::VirtAddr::from(start+len);
        //遍历了虚拟地址范围内的所有虚拟页
        for vpn in mm::VPNRange::new(start_va.floor(),end_va.ceil()){
            if let Some(pte) =  inner.memory_set.translate(vpn){
                if pte.is_valid(){
                    return -1;
                }
            }
        }
        let map_permission: mm::MapPermission = MapPermission::from_bits((port as u8) << 1).unwrap() | MapPermission::U;
        inner.memory_set.insert_framed_area(start_va, end_va, map_permission);
        //检查从起始地址到结束地址中是否有未被映射的内存
        for vpn in mm::VPNRange::new(start_va.floor(),end_va.ceil()){
            match inner.memory_set.translate(vpn) {
                Some(pte)=>{
                    if pte.is_valid()==false{
                        return -1;
                    }
                }
                None => {
                    return -1;
                }
            }
        }
        0
    }
    ///ch5: munmap
    pub fn munmap(&self,start:usize,len:usize)->isize{
        let mut inner=self.inner.exclusive_access();
        let start_va=mm::VirtAddr::from(start);
        let end_va=mm::VirtAddr::from(start+len);
        //检查从起始地址到结束地址中是否有未被映射的内存
        for vpn in mm::VPNRange::new(start_va.floor(),end_va.ceil()){
            match inner.memory_set.translate(vpn) {
                Some(pte)=>{
                    if pte.is_valid()==false{
                        return -1;
                    }
                }
                None => {
                    return -1;
                }
            }
        }
        inner.memory_set.remove_area_with_start_vpn(mm::VirtPageNum::from(start_va));//按照虚页号删除，源代码中提供，不像ch4需要自己实现
        0
    }
```
### 2.2 进程管理器
任务管理器 或称 进程管理器 `TaskManager` 将 `CPU` 的监控职能拆分到处理器管理结构 `Processor` 中， 其自身仅负责管理所有进程， 这种分离有助于后续实现多核环境的拓展。 这里比较关键的设计是将 `TaskControlBlock` 用 `Arc` 智能指针管理， 并将其存储在堆上， 不仅减少开销也方便维护。
```rust
// os/src/task/manager.rs
pub struct TaskManager {
    ready_queue: VecDeque<Arc<TaskControlBlock>>,
}
```
### 2.3 CPU管理器
处理器管理结构 `Processor` 负责维护 `CPU` 相关的状态。 其中 `current` 表示当前处理器上执行的进程任务， 而 `idle_task_cx` 则是 `idle` 控制流的任务上下文。 这个 `idle` 控制流运行在 `CPU` 启动栈上， 它会从 `TaskManager` 中选择一个进程任务放在当前的 `CPU` 核上运行。
```rust
// os/src/task/processor.rs
pub struct Processor {
    current: Option<Arc<TaskControlBlock>>,
    idle_task_cx: TaskContext,
}
```
`idle` 控制流的设计目的在于分离进程调度与 `Trap`， 换入/换出进程仅需要调用 `schedule` 函数， 通过 `__switch` 保存自身进程的上下文， 之后程序流会换回 `idle_task_cx_ptr` 中的内容重新回到 `run_tasks` 的 loop 循环中查找下一个待运行的进程。 这点设计和 `xv6` 如出一辙， 在 `rCore` 中则是会持续运行 `run_tasks` 用以进程的调度。

>`rCore` 让 `run_tasks` 始终保持运行， 可以把这个函数想象成一个游戏机，它的作用就是不断挑选可运行的游戏卡（进程）， 然后放到 `CPU` 这个卡槽中。 一定时间后这个 `CPU` 卡槽要给别的游戏卡（进程）用了就把这个当前的卡弹出来（这里就用了 `schedule` 函数弹出）， 然后游戏机又开始选下一张能用的游戏卡（进程）了。  
游戏卡（进程）只知道自己被插上要运行， 以及到某个点自己要弹出来， 它不知道游戏机的存在， 这样就做到了任务调度的透明。   ---from hangx-ma

## 3. 进程管理机制
### 3.1 初始进程创建
唯一一个通过硬编码创建的进程:`INITPROC`。
### 3.2 进程调度机制
进程调度一方面是 `CPU` 分给当前进程的时间片用尽触发 `SupervisorTimer` 中断需要调度， 另一方面是进程主动让出 `CPU` 的占用权， 都需要用到 `suspend_current_and_run_next` 这个函数。 在引入进程的抽象之后， 调度不需要进程本身更新关于 `__switch` 函数相关的内容， 只需要获取当前进程的控制块 `TCB` 并将其加入 `task` 队列中后使用 `schedule` 即可。 这也是前述引入 `idle` 控制流的好处之一， 对进程调度的解耦让整个代码流都干净了很多。

### 3.3 进程生成机制
在内核中唯一的初始化进程是 `initproc`， 后续的进程都需要通过 `fork/exec` 这两个系统调用提供的进程生成机制实现。

* **fork**  
  `fork` 需要对除了返回值外的父进程所有信息的完全复制， 这甚至要求地址空间的映射方式， 映射页的权限， 映射范围， 以及数据都需要与父进程一致， 不同的是最终映射到的具体的物理地址页 `PPN`。 在 rCore 中管理这部分信息的是 `memory_set.rs`， 为了复制这些信息新增了 `from_another` 函数拷贝一个 逻辑段 的上述数据。  
  这之后真正对整个地址空间进行完全复制的是， `MemorySet` 中新增的`from_existed_user` 函数。 该函数将生成一个新的 `tranmpoline` 并遍历当前用户态虚拟地址空间中所有的逻辑段并拷贝这些数据到目标地址空间。  
  可以看到 `TaskControlBlock::fork` 的代码和 new 基本一致， 只是它的数据来源于它的父进程， 子进程的地址空间通过 `MemorySet::from_existed_user` 建立。 另外， `sys_fork` 的实现中需要更改 `TrapContext` 中的 `a0` 寄存器， `trap_handler` 部分需要配合做出微调， 以保证能够通过返回值区分子进程和父进程。

* **exec**    
  `exec` 系统调用是载入一个新的 ELF 的代码和数据替换当前进程的应用地址空间中的内容并执行。 它要做的事情其实和 `TaskControlBlock::new` 也很像， 但我们不需要重新生成进程的 `PID` 以及分配新的 `KernelStack`， 这些信息原有的进程已经提供了， 仅需要将我们所需要的 `memory_set`， `user_sp`， `entry_point` 这些信息进行更新即可， 之后我们仅需要完善 `sys_exec` 系统调用的实现。  
  最为关键的还是 `translated_str` 这个函数， `sys_exec` 需要对输入的 `path` 参数进行解析， 这个 `path` 的内容来自应用的虚拟地址空间。 因而 `translated_str` 就需要在该应用空间中通过查找 `Page Table` 的方式逐个拷贝字符串信息到当前`kernel` 中的 `string` 变量中。
```rust
/// Translate&Copy a ptr[u8] array end with `\0` to a `String` Vec through page table
pub fn translated_str(token: usize, ptr: *const u8) -> String {
    let page_table = PageTable::from_token(token);
    let mut string = String::new();
    let mut va = ptr as usize;
    loop {
        let ch: u8 = *(page_table
            .translate_va(VirtAddr::from(va))
            .unwrap()
            .get_mut());
        if ch == 0 {
            break;
        } else {
            string.push(ch as char);
            va += 1;
        }
    }
    string
}
```
* **read**  
目前`sys_read`实现还没涉及到文件系统，使用`RustSBI`提供的接口获取用户从键盘的写入。
### 3.4 进程资源回收机制
资源回收涉及到进程的退出， 在此之前 task 中实现此功能的是 exit_current_and_run_next 函数， 但相比之前的章节， ch5 需要该函数传入一个退出码， 这个退出码会写入到当前进程的进程控制块 TCB 中。 这里比较关键的一步操作是将当前进程的所有子进程的父进程更改为 INITPROC， 这样当前进程被回收后其子进程仍能被管理而不至于进入一种无法定义的状态。
```rust
/// Exit the current 'Running' task and run the next task in task list.
pub fn exit_current_and_run_next(exit_code: i32) {
    // take from Processor
    let task = take_current_task().unwrap();

    let pid = task.getpid();
    if pid == IDLE_PID {
        println!(
            "[kernel] Idle process exit with exit_code {} ...",
            exit_code
        );
        panic!("All applications completed!");
    }

    // **** access current TCB exclusively
    let mut inner = task.inner_exclusive_access();
    // Change status to Zombie
    inner.task_status = TaskStatus::Zombie;
    // Record exit code
    inner.exit_code = exit_code;
    // do not move to its parent but under initproc

    // ++++++ access initproc TCB exclusively
    {
        let mut initproc_inner = INITPROC.inner_exclusive_access();
        for child in inner.children.iter() {
            child.inner_exclusive_access().parent = Some(Arc::downgrade(&INITPROC));
            initproc_inner.children.push(child.clone());
        }
    }
    // ++++++ release parent PCB

    inner.children.clear();
    // deallocate user space
    inner.memory_set.recycle_data_pages();
    drop(inner);
    // **** release current PCB
    // drop task manually to maintain rc correctly
    drop(task);
    // we do not have to save task context
    let mut _unused = TaskContext::zero_init();
    schedule(&mut _unused as *mut _);
}
```
父进程的资源回收操作详见 `sys_waitpid` 系统调用。
## 实践作业
## 1.spawn
大家一定好奇过为啥进程创建要用 `fork + execve` 这么一个奇怪的系统调用，就不能直接搞一个新进程吗？思而不学则殆，我们就来试一试！这章的编程练习请大家实现一个完全 DIY 的系统调用 `spawn`，用以创建一个新进程。

`spawn` 系统调用定义

```rust
fn sys_spawn(path: *const u8) -> isize
```
* syscall ID: 400

* 功能：新建子进程，使其执行目标程序。

说明：成功返回子进程id，否则返回 -1。

可能的错误：
* 无效的文件名。

* 进程池满/内存不足等资源错误。

TIPS：虽然测例很简单，但提醒读者 `spawn` **不必** 像 `fork` 一样复制父进程的地址空间。

这里直接缝合fork和exec两个函数。真就fork+exec=spawn 🤣  

```rust
//os/src/task/task.rs
    ///spawn
    pub fn spawn(self:&Arc<TaskControlBlock>,elf_data: &[u8])->Arc<TaskControlBlock>{
        let mut parent_inner=self.inner_exclusive_access();// 来自fork
        let (memory_set,user_sp,entry_point)=MemorySet::from_elf(elf_data); //来自exec
        let trap_cx_ppn=memory_set.translate(VirtAddr::from(TRAP_CONTEXT_BASE).into()).unwrap().ppn();//来自exec
        let pid_handle=pid_alloc();//来自fork
        let kernel_stack=kstack_alloc();//分配新的内核栈
        let kernel_stack_top=kernel_stack.get_top();
        let task_control_block=Arc::new(Self{
            pid:pid_handle,
            kernel_stack,
            inner:unsafe{
                UPSafeCell::new(TaskControlBlockInner { 
                    trap_cx_ppn, 
                    base_size: user_sp, 
                    task_cx: TaskContext::goto_trap_return(kernel_stack_top), 
                    task_status: TaskStatus::Ready, 
                    memory_set, 
                    parent: Some(Arc::downgrade(self)), 
                    children: Vec::new(), 
                    exit_code: 0, 
                    heap_bottom: user_sp, 
                    program_brk: user_sp, 
                    stride:0,
                    priority:16,
                })
            },
        });
        parent_inner.children.push(task_control_block.clone());
        let trap_cx=task_control_block.inner_exclusive_access().get_trap_cx();
        *trap_cx=TrapContext::app_init_context(
          entry_point, 
          user_sp, 
          KERNEL_SPACE.exclusive_access().token(),
          kernel_stack_top, 
          trap_handler as usize
        );

        task_control_block
    }

//os/src/syscall/sys_spawn


/// YOUR JOB: Implement spawn.
/// HINT: fork + exec =/= spawn
pub fn sys_spawn(path: *const u8) -> isize {
    trace!(
        "kernel:pid[{}] sys_spawn NOT IMPLEMENTED",
        current_task().unwrap().pid.0
    );
    let token=current_user_token();
    let path=translated_str(token, path);
    if let Some(data)=get_app_data_by_name(path.as_str()){
        let current_task=current_task().unwrap();
        let task=current_task.spawn(data);
        let pid=task.getpid();
        add_task(task);
        pid as isize
    }
    else {
        -1
    }
}  
```

## 2.stride 调度算法
ch3 中我们实现的调度算法十分简单。现在我们要为我们的 `os` 实现一种带优先级的调度算法：`stride` 调度算法。

算法描述如下:

1. 为每个进程设置一个当前 `stride`，表示该进程当前已经运行的“长度”。另外设置其对应的 `pass` 值（只与进程的优先权有关系），表示对应进程在调度后，stride 需要进行的累加值。

2. 每次需要调度时，从当前 `runnable` 态的进程中选择 `stride` 最小的进程调度。对于获得调度的进程 P，将对应的 `stride` 加上其对应的步长 `pass`。

3. 一个时间片后，回到上一步骤，重新调度当前`stride` 最小的进程。

可以证明，如果令`P.pass = BigStride / P.priority` 其中` P.priority` 表示进程的优先权（大于 1），而 `BigStride` 表示一个预先定义的大常数，则该调度方案为每个进程分配的时间将与其优先级成正比。

其他实验细节：

* `stride` 调度要求进程优先级 >=2 ，所以设定进程优先级<=1会导致错误。

* 进程初始 `stride` 设置为 0 即可。

* 进程初始优先级设置为 16。

为了实现该调度算法，内核还要增加 set_prio 系统调用
```rust
// syscall ID：140
// 设置当前进程优先级为 prio
// 参数：prio 进程优先级，要求 prio >= 2
// 返回值：如果输入合法则返回 prio，否则返回 -1
fn sys_set_priority(prio: isize) -> isize;
```
```rust
//os/src/task/task.rs
    /// 设置优先级
    pub fn set_priority(&self, prio: isize)->isize{
        let mut inner = self.inner_exclusive_access();
        inner.priority = prio as usize;
        inner.priority as isize
    }


//os/src/syscall/process.rs
// YOUR JOB: Set task priority.
pub fn sys_set_priority(prio: isize) -> isize {
    trace!(
        "kernel:pid[{}] sys_set_priority NOT IMPLEMENTED",
        current_task().unwrap().pid.0
    );
    if prio<=1{
        return -1;
    }
    else{
        current_task().unwrap().set_priority(prio)
    }
}
```
在调度算法中使用了比较暴力的方法，即把整个队列用迭代器进行遍历，取出其中具有最小的`stride`的任务进行执行。  
另外的想法是使用优先队列，但是在`no_std`环境下不知道要怎么用优先队列。  
```rust
//os/src/config.rs
///stride
pub const BIG_STRIDE:usize=usize::MAX;

//os/src/task/manage.rs
    /// Take a process out of the ready queue
    /// update: stride
    pub fn fetch(&mut self) -> Option<Arc<TaskControlBlock>> {
        //self.ready_queue.pop_front()  //原来的FIFO
        let mut min_index=0;
        let mut min_stride=0x7FFF_FFFF;
        for (idx,task) in self.ready_queue.iter().enumerate(){
            let inner=task.inner_exclusive_access();
            if inner.task_status==TaskStatus::Ready{
                if inner.stride<min_stride{
                    min_stride=inner.stride;
                    min_index=idx;
                }
            }
        }
        if let Some(task)=self.ready_queue.get(min_index){
            let mut inner=task.inner_exclusive_access();
            inner.stride+=BIG_STRIDE/inner.priority;
        }
        self.ready_queue.remove(min_index)
    }
```
## 测试：
![rCore-ch5-0](/images/rCore/ch5-0.png)

## 本章小结
通过本章的学习加深了对进程的了解。巩固了在`UNIX系统程序设计`课程中的学习。