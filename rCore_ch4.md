---
title: 'rCore-ch4'
date: 2024/1/13
permalink: /posts/2024/01/rCore_ch4/
excerpt: '第四章 地址空间'
tags:
  - rCore
  - OS
---

# 第四章 地址空间
## SV39多级页表
在本章中使用了SV39多级页表，多级页表机制类似字典树(Trie)的实现，大大减小了页表的内存占用。  
![rCore-ch4-0](/images/rCore/ch4-0.png)
rCore 和 xv6 都使用了 SV39 模式， 这意味着 64-bit 的 VA (Virtual Address) 仅有低端的 39 bits 是被使用的。 而这 39 位又被划分为两部分， 被称为 VPN (Virtual Page Number) 的高 27 位用于索引 PTE (Page Table Entry)， PTE 是存放在每个应用的 Page Table 中， 由 44 位的 PPN (Physical Page Number) 与 10 位标志位组成， 这 PTE 实际构成了虚拟地址与物理地址的索引关系。 另外低 12 位是表示页内的偏移量， 这是因为我们使用 分页内存管理， 最小的内存单位是 页， 这低 12 位最终会与 PPN 组合成为实际的 56 位物理地址。

    需要明确的是 2^12=4096， 因而虚拟地址的低 12 位才被称作是页内的偏移量， 因为 12 位正好构成了一个页的大小。

以物理页号为单位进行物理页帧的分配和回收，实现一种最简单的栈式物理页帧管理策略：  
```rust
// os/src/mm/frame_allocator.rs

pub struct StackFrameAllocator {
    current: usize,  //空闲内存的起始物理页号
    end: usize,      //空闲内存的结束物理页号
    recycled: Vec<usize>,
}
```
其中各字段的含义是：物理页号区间 `[current , end)`此前均**从未**被分配出去过，而向量`recycled`以后入先出的方式保存了被回收的物理页号（注：我们已经自然的将内核堆用起来了）。  
初始化非常简单。在通过 `FrameAllocator` 的 `new` 方法创建实例的时候，只需将区间两端均设为 
，然后创建一个新的向量；而在它真正被使用起来之前，需要调用 `init` 方法将自身的 `[current , end)`
初始化为可用物理页号区间：
```rust
// os/src/mm/frame_allocator.rs

impl FrameAllocator for StackFrameAllocator {
    fn new() -> Self {
        Self {
            current: 0,
            end: 0,
            recycled: Vec::new(),
        }
    }
}

impl StackFrameAllocator {
    pub fn init(&mut self, l: PhysPageNum, r: PhysPageNum) {
        self.current = l.0;
        self.end = r.0;
    }
}
```
**接下来是核心的物理页帧分配和回收的实现机制：**
```rust
// os/src/mm/frame_allocator.rs

impl FrameAllocator for StackFrameAllocator {
    fn alloc(&mut self) -> Option<PhysPageNum> {
        if let Some(ppn) = self.recycled.pop() {
            Some(ppn.into())
        } else {
            if self.current == self.end {
                None
            } else {
                self.current += 1;
                Some((self.current - 1).into())
            }
        }
    }
    fn dealloc(&mut self, ppn: PhysPageNum) {
        let ppn = ppn.0;
        // validity check
        if ppn >= self.current || self.recycled
            .iter()
            .find(|&v| {*v == ppn})
            .is_some() {
            panic!("Frame ppn={:#x} has not been allocated!", ppn);
        }
        // recycle
        self.recycled.push(ppn);
    }
}
```
* 在分配 `alloc` 的时候，首先会检查栈 `recycled` 内有没有之前回收的物理页号，如果有的话直接弹出栈顶并返回；否则的话我们只能从之前从未分配过的物理页号区间 `[ current , end )` 上进行分配，我们分配它的左端点 `current` ，同时将管理器内部维护的 `current` 加 `1` 代表 `current` 已被分配了。在即将返回的时候，我们使用 `into` 方法将 `usize` 转换成了物理页号 `PhysPageNum` 。

* 注意极端情况下可能出现内存耗尽分配失败的情况：即 `recycled` 为空且 `current == end` 。为了涵盖这种情况， `alloc` 的返回值被 `Option` 包裹，我们返回 `None` 即可。

* 在回收 `dealloc` 的时候，我们需要检查回收页面的合法性，然后将其压入 `recycled` 栈中。回收页面合法有两个条件：

    * 该页面之前一定被分配出去过，因此它的物理页号一定 小于`current` ；

    * 该页面没有正处在回收状态，即它的物理页号不能在栈 `recycled` 中找到。  
我们通过 `recycled.iter()` 获取栈上内容的迭代器，然后通过迭代器的 `find` 方法试图寻找一个与输入物理页号相同的元素。其返回值是一个 `Option` ，如果找到了就会是一个 `Option::Some` ，这种情况说明我们内核其他部分实现有误，直接报错退出。  

## 多级页表管理
SV39 多级页表是以节点为单位进行管理的。每个节点恰好存储在一个物理页帧中，它的位置可以用一个物理页号来表示。
```rust
// os/src/mm/page_table.rs

pub struct PageTable {
    root_ppn: PhysPageNum,
    frames: Vec<FrameTracker>,
}

impl PageTable {
    pub fn new() -> Self {
        let frame = frame_alloc().unwrap();
        PageTable {
            root_ppn: frame.ppn,
            frames: vec![frame],
        }
    }
}
```
每个应用的地址空间都对应一个不同的多级页表，这也就意味这不同页表的起始地址（即页表根节点的地址）是不一样的。因此 `PageTable` 要保存它根节点的物理页号 `root_ppn` 作为页表唯一的区分标志。向量 `frames` 以 `FrameTracker` 的形式保存了页表所有的节点（包括根节点）所在的物理页帧。  
当我们通过 `new` 方法新建一个 `PageTable` 的时候，它只需有一个根节点。为此我们需要分配一个物理页帧 `FrameTracker` 并挂在向量 `frames` 下，然后更新根节点的物理页号 `root_ppn` 。

多级页表并不是被创建出来之后就不再变化的，为了 MMU 能够通过地址转换正确找到应用地址空间中的数据实际被内核放在内存中位置，操作系统需要动态维护一个虚拟页号到页表项的映射，支持插入/删除键值对，其方法如下：
```rust
// os/src/mm/page_table.rs

impl PageTable {
    pub fn map(&mut self, vpn: VirtPageNum, ppn: PhysPageNum, flags: PTEFlags) {
        let pte = self.find_pte_create(vpn).unwrap();
        assert!(!pte.is_valid(), "vpn {:?} is mapped before mapping", vpn);
        *pte = PageTableEntry::new(ppn, flags | PTEFlags::V);
    }
    pub fn unmap(&mut self, vpn: VirtPageNum) {
        let pte = self.find_pte(vpn).unwrap();
        assert!(pte.is_valid(), "vpn {:?} is invalid before unmapping", vpn);
        *pte = PageTableEntry::empty();
    }
}
```
只需根据虚拟页号找到页表项，然后修改或者直接清空其内容即可。  
通过 `map` 方法来在多级页表中插入一个键值对，注意这里将物理页号 `ppn` 和页表项标志位 `flags` 作为不同的参数传入;  
通过 `unmap` 方法来删除一个键值对，在调用时仅需给出作为索引的虚拟页号即可。      
`find_pte`和`find_pte_create`的区别是当找不到合法叶子节点的时候不会新建叶子节点而是直接返回 `None` 即查找失败。因此，它不会尝试对页表本身进行修改，但是注意它返回的参数类型是页表项的可变引用，也即它允许我们修改页表项。从 `find_pte` 的实现还可以看出，即使找到的页表项不合法，还是会将其返回回去而不是返回 `None` 。这说明在目前的实现中，页表和页表项是相对解耦合的。    
在上述操作的过程中，内核需要能访问或修改多级页表节点的内容。即在操作某个多级页表或管理物理页帧的时候，操作系统要能够读写与一个给定的物理页号对应的物理页帧上的数据。这是因为，在多级页表的架构中，每个节点都被保存在一个物理页帧中，一个节点所在物理页帧的物理页号其实就是指向该节点的“指针”。   

<h3>为了方便后面的实现，我们还需要 PageTable 提供一种类似 MMU 操作的手动查页表的方法：</h3>  

```rust
// os/src/mm/page_table.rs

impl PageTable {
    /// Temporarily used to get arguments from user space.
    pub fn from_token(satp: usize) -> Self {
        Self {
            root_ppn: PhysPageNum::from(satp & ((1usize << 44) - 1)),
            frames: Vec::new(),
        }
    }
    pub fn translate(&self, vpn: VirtPageNum) -> Option<PageTableEntry> {
        self.find_pte(vpn)
            .map(|pte| {pte.clone()})
    }
}
```
* 第 5 行的 `from_token` 可以临时创建一个专用来手动查页表的 `PageTable` ，它仅有一个从传入的 `satp token` 中得到的多级页表根节点的物理页号，它的 `frames` 字段为空，也即不实际控制任何资源；

* 第 11 行的 `translate` 调用 `find_pte` 来实现，如果能够找到页表项，那么它会将页表项拷贝一份并返回，否则就返回一个 `None` 。

之后，当遇到需要查一个特定页表（非当前正处在的地址空间的页表时），便可先通过 `PageTable::from_token` 新建一个页表，再调用它的 `translate` 方法查页表。  

## 内核访问物理页帧
为了让内核能访问实际的物理地址， rCore 设计了三种粒度的访问方式： 基于 PTE， 基于 Bytes， 基于变量类型。 至此， 物理地址空间的分配以及访问的框架已经建成， 后续需要做的就是构建虚拟地址与物理地址映射的 页表(Page Table)。
```rust
// os/src/mm/address.rs

impl PhysPageNum {
    pub fn get_pte_array(&self) -> &'static mut [PageTableEntry];
    pub fn get_bytes_array(&self) -> &'static mut [u8];
    pub fn get_mut<T>(&self) -> &'static mut T;
}
```
## 改进SYSCALL
本章更改了 `sys_write` 系统调用， 通过 `ranslated_byte_buffer` 在内核空间中开辟了一个可以访问应用数据的区域（实际上是将应用数据从用户态虚拟地址空间拷贝到了内核态虚拟地址空间以供内核访问）。 相应的， 前述章节需要访问应用态数据的系统调用均需要通过这种间接的方式进行数据的访问与更改。 这也就是 `sys_get_time` 以及 `sys_task_info` 在第四章失效的原因。

## 编程作业
<h3>1.重写 sys_get_time 和 sys_task_info  </h3>

引入虚存机制后，原来内核的 sys_get_time 和 sys_task_info 函数实现就无效了。请你重写这个函数，恢复其正常功能。  

### sys_get_time:
```rust
// os/src/syscall/process.rs

/// YOUR JOB: get time with second and microsecond
/// HINT: You might reimplement it with virtual memory management.
/// HINT: What if [`TimeVal`] is splitted by two pages ?
pub fn sys_get_time(ts: *mut TimeVal, _tz: usize) -> isize {
    trace!("kernel: sys_get_time");
    let us=get_time_us();
    let p_ts=translated_mut_ptr(current_user_token(), ts);
    *p_ts = TimeVal {
        sec: us / 1_000_000,
        usec: us % 1_000_000,
    };
    0
}

//current_user_token()的定义如下：
/// Get the current 'Running' task's token.
//pub fn current_user_token() -> usize {
//    TASK_MANAGER.get_current_token()
//}
```
这里首先要使用`current_user_token()`获取当前正在运行的任务的`token`,用这个token来查页表,`tranlated_mut_ptr()`获得在物理页帧中的类型为`TimeVal`的可变引用，然后对这个`TimeVal`进行修改。  
思路：`token->手动查页表->PTE->PhysPageNum->PhysAddr-> get_mut()访问物理地址`
```rust
//os/src/mm/page_table.rs

/// ch4:从虚拟地址转换为物理地址
pub fn virt_to_pyhs(token: usize,va: VirtAddr) -> PhysAddr{   
    let pagetable = PageTable::from_token(token);//从token查页表
    let vpn = va.floor();  //获取虚拟地址所处的虚拟页的页号
    //页表+虚拟页号->三级页表项(指向物理页的页表项)->物理页号->物理地址的前半部分
    let mut pa: PhysAddr = pagetable.translate(vpn).unwrap().ppn().into(); 
    pa.0 += va.page_offset();  //前半部分与页内偏移组合为完整物理地址
    pa  
}

/// ch4: return a physical pointer in memory set of the pagetable token 
pub fn translated_mut_ptr<T>(token: usize, ptr: *mut T) -> &'static mut T{
    let ptr_va = VirtAddr::from(ptr as usize);
    let ptr_pa = virt_to_pyhs(token, ptr_va);
    ptr_pa.get_mut()  //rCore的三种粒度访问物理页帧的方式之三:基于变量类型
}
```  

### sys_task_info:  

这里时间会涉及到精度的问题（我也想知道为什么），在ch3中直接使用`get_time_ms()`与`tcb`中的`start_time`相减即可，但是在ch4中`get_time_ms()`的精度不够，需要从`get_time_us()`获得的微秒时间进行转换。同时需要把`tcb`中的`ms`时间转换为`us`。    
关于`get_current_task()`的实现参考ch3，但是需要进行修改，原因是在ch3中，`TaskManagerInner`的数据结构为普通的数组，而在ch4中被更新为`Vec`，其中包含不支持`clone`的成员，在多次尝试后一直无法绕过，所以进行如此修改。  
关于查页表操作可参考`sys_get_time()`中的说明。
```rust
//区别在这里
/// Inner of Task Manager
ch3:
pub struct TaskManagerInner {
    /// task list
    tasks: [TaskControlBlock; MAX_APP_NUM],
    /// id of current `Running` task
    current_task: usize,
}
ch4:
struct TaskManagerInner {
    /// task list
    tasks: Vec<TaskControlBlock>,
    /// id of current `Running` task
    current_task: usize,
}
```
```rust
//os/src/task/mod.rs
    ///当前任务的TCB
    fn get_current_task(&self)-> (TaskStatus, [u32; MAX_SYSCALL_NUM],usize){
        let inner=self.inner.exclusive_access();
        let current=inner.current_task;
        let tcb=&inner.tasks[current];
        let time=get_time_ms();
        let mut syscall_times_clone:[u32;MAX_SYSCALL_NUM]=[0;MAX_SYSCALL_NUM];
        for i in 0..tcb.syscall_times.len(){
            syscall_times_clone[i]=tcb.syscall_times[i];
        }
        (
            TaskStatus::Running,
            syscall_times_clone,
            time-tcb.start_time,
        )
    }
    ///增加系统调用次数
    fn inc_syscall_times(&self,syscall_id:usize){
        let mut inner=self.inner.exclusive_access();
        let current=inner.current_task;
        inner.tasks[current].syscall_times[syscall_id]+=1;
    }
```
```rust
//os/src/syscall/process.rs

/// YOUR JOB: Finish sys_task_info to pass testcases
/// HINT: You might reimplement it with virtual memory management.
/// HINT: What if [`TaskInfo`] is splitted by two pages ?
pub fn sys_task_info(ti: *mut TaskInfo) -> isize {
    trace!("kernel: sys_task_info NOT IMPLEMENTED YET!");
    let (status,syscall_times,start_time)=get_current_task();
    let p_ti=translated_mut_ptr(current_user_token(), ti);
    let time_now = get_time_us();
    let time_now_ms = ((time_now / 1_000_000) & 0xffff) * 1000 + (time_now % 1_000_000 ) / 1000;
    let time_start_ms = ((start_time / 1_000_000) & 0xffff) * 1000 + (start_time % 1_000_000 ) / 1000;
    let time = time_now_ms - time_start_ms;
    *p_ti=TaskInfo{
            status,
            syscall_times,
            time,
        };
    0
}
```
<h3>2.mmap 和 munmap 匿名映射</h3>

mmap 在 Linux 中主要用于在内存中映射文件， 本次实验简化它的功能，仅用于申请内存。

请实现 `mmap` 和 `munmap` 系统调用，`mmap` 定义如下：
```rust
fn sys_mmap(start: usize, len: usize, port: usize) -> isize
```
* syscall ID：222

* 申请长度为 `len` 字节的物理内存（不要求实际物理内存位置，可以随便找一块），将其映射到 `start` 开始的虚存，内存页属性为 `port`

* 参数：  
    * `start` 需要映射的虚存起始地址，要求按页对齐

    * `len` 映射字节长度，可以为 0

    * `port`：第 0 位表示是否可读，第 1 位表示是否可写，第 2 位表示是否可执行。其他位无效且必须为 0

* 返回值：执行成功则返回 0，错误返回 -1

* 说明：
    为了简单，目标虚存区间要求按页对齐，len 可直接按页向上取整，不考虑分配失败时的页回收。

* 可能的错误：
    * `start `没有按页大小对齐  
    * `port & !0x7 != 0` (port 其余位必须为0)
    * `port & 0x7 = 0` (这样的内存无意义)
    * `[start, start + len)` 中存在已经被映射的页
    * 物理内存不足

munmap 定义如下：
```rust
fn sys_munmap(start: usize, len: usize) -> isize
```
* syscall ID：215

* 取消到 `[start, start + len)` 虚存的映射  

* 参数和返回值请参考 mmap

* 说明：
为了简单，参数错误时不考虑内存的恢复和回收。

* 可能的错误：
`[start, start + len)` 中存在未被映射的虚存。

tips:

* 一定要注意 `mmap` 是对页表项，注意 `riscv` 页表项的格式与 `port` 的区别。

* 你增加 PTE_U 了吗？

### mmap:
用于申请内存从虚拟内存映射到物理内存。输入参数`start`为需要映射的虚存起始地址，`len`为虚拟内存长度。首先使用`.floor()`和`.ceil()`方法找到虚拟内存对应的虚拟页的起始和结束地址，然后对其中的所有虚拟页进行遍历，如果其中的虚拟页无效，则返回-1。  
然后用`from_bits()`方法把输入的`por`t转化为`MapPermission`，注意添加`PTE_U`。  
最后调用`insert_framed_area()`向物理内存中添加页框。
添加完毕后对成功性进行检查。  
```rust
//os/src/syscall/process.rs

// YOUR JOB: Implement mmap.
pub fn sys_mmap(start: usize, len: usize, port: usize) -> isize {
    trace!("kernel: sys_mmap NOT IMPLEMENTED YET!");
    //对合法性进行检查
    if start %  PAGE_SIZE != 0 || port & !0x7 != 0 || port & 0x7 == 0{
        return -1;
    }
    mmap(start,len,port)
}

//os/src/task/mod.rs
...
///申请内存
    fn mmap(&self,start: usize, len: usize, port: usize)->isize{
        let mut inner=self.inner.exclusive_access();
        let current=inner.current_task;
        let tcb=&mut inner.tasks[current];
        let start_va=mm::VirtAddr::from(start);
        let end_va=mm::VirtAddr::from(start+len);
        //遍历了虚拟地址范围内的所有虚拟页
        for vpn in mm::VPNRange::new(start_va.floor(),end_va.ceil()){
            if let Some(pte) =  tcb.memory_set.translate(vpn){
                if pte.is_valid(){
                    return -1;
                }
            }
        }
        let map_permission: mm::MapPermission = MapPermission::from_bits((port as u8) << 1).unwrap() | MapPermission::U;
        tcb.memory_set.insert_framed_area(start_va, end_va, map_permission);
        //检查从起始地址到结束地址中是否有未被映射的内存
        for vpn in mm::VPNRange::new(start_va.floor(),end_va.ceil()){
            match tcb.memory_set.translate(vpn) {
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
```

### munmap
用于删除从物理内存中删除一段虚拟地址的映射。  
主要是一个delete_frame_area()方法，和insert_frame_area()是一个对应关系，但是插入方法原来的代码中已经给出，删除代码需要仿照其去实现。
```rust
//os/src/mm/memory_set.rs

    /// Assume that no conflicts.
    ///新增页框  原环境自带
    pub fn insert_framed_area(
        &mut self,
        start_va: VirtAddr,
        end_va: VirtAddr,
        permission: MapPermission,
    ) {
        self.push(
            MapArea::new(start_va, end_va, MapType::Framed, permission),
            None,
        );
    }
    /// ch4:删除页框
    pub fn delete_frame_area(
        &mut self,
        start_va: VirtAddr,
        end_va: VirtAddr,
    ){
        let start_vpn = start_va.floor();
        let end_vpn = end_va.ceil();  //虚拟地址转为虚拟页号
        let map_area = self.areas.iter_mut().filter(|a| {
            a.vpn_range.get_start() == start_vpn && 
            a.vpn_range.get_end() == end_vpn
        }).next().unwrap(); //使用迭代器和过滤器，遍历 self 中的区域集合，找到与指定虚拟地址范围匹配的区域

        map_area.unmap(&mut self.page_table);
    }
```
```rust
//os/src/syscall/process.rs
// YOUR JOB: Implement munmap.
pub fn sys_munmap(start: usize, len: usize) -> isize {
    trace!("kernel: sys_munmap NOT IMPLEMENTED YET!");
    if start %  PAGE_SIZE != 0 {
        return -1;
    }
    munmap(start, len)
}

//os/src/task/mod.rs
    ///回收内存
    fn munmap(&self,start:usize,len:usize)->isize{
        let mut inner=self.inner.exclusive_access();
        let current=inner.current_task;
        let tcb=&mut inner.tasks[current];
        let start_va=mm::VirtAddr::from(start);
        let end_va=mm::VirtAddr::from(start+len);
        //检查从起始地址到结束地址中是否有未被映射的内存
        for vpn in mm::VPNRange::new(start_va.floor(),end_va.ceil()){
            match tcb.memory_set.translate(vpn) {
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
        tcb.memory_set.delete_frame_area(start_va, end_va);//按照虚拟地址从物理内存中删除页框
        0
    }
```
### 运行测试：
![rCore-ch4](/images/rCore/ch4-1.png)
