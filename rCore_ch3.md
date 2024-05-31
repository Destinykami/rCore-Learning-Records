---
title: 'rCore-ch3'
date: 2024/1/12
permalink: /posts/2024/01/rCore_ch3/
excerpt: '第三章 多道程序与分时多应用 '
tags:
  - rCore
  - OS
---


# 第三章 多道程序与分时多应用
## 编程作业
### 获取任务信息

ch3 中，我们的系统已经能够支持多个任务分时轮流运行，我们希望引入一个新的系统调用 `sys_task_info` 以获取当前任务的信息，定义如下：

```rust
fn sys_task_info(ti: mut TaskInfo) -> isize
```

- syscall ID: 410
- 查询当前正在执行的任务信息，任务信息包括任务控制块相关信息（任务状态）、任务使用的系统调用及调用次数、系统调用时刻距离任务第一次被调度时刻的时长（单位ms）。

```rust
struct TaskInfo {
    status: TaskStatus,
    syscall_times: [u32; MAX_SYSCALL_NUM],
    time: usize
}
```

- **参数：** ti: 待查询任务信息
- **返回值**：执行成功返回0，错误返回-1
- **说明：**
    - 相关结构已在框架中给出，只需添加逻辑实现功能需求即可。
    - 在我们的实验中，系统调用号一定小于 500，所以直接使用一个长为 `MAX_SYSCALL_NUM=500` 的数组做桶计数。
    - 运行时间 time 返回系统调用时刻距离任务第一次被调度时刻的时长，也就是说这个时长可能包含该任务被其他任务抢占后的等待重新调度的时间。
    - 由于查询的是当前任务的状态，因此 TaskStatus 一定是 Running。（助教起初想设计根据任务 id 查询，但是既不好定义任务 id 也不好写测例，遂放弃 QAQ）
    - 调用 `sys_task_info` 也会对本次调用计数。
- **提示：**
    - 大胆修改已有框架！除了配置文件，你几乎可以随意修改已有框架的内容。
    - 程序运行时间可以通过调用 `get_time()` 获取，注意任务运行总时长的单位是 ms。
    - 系统调用次数可以考虑在进入内核态系统调用异常处理函数之后，进入具体系统调用函数之前维护。
    - 阅读 TaskManager 的实现，思考如何维护内核控制块信息（可以在控制块可变部分加入需要的信息）。
    - 虽然系统调用接口采用桶计数，但是内核采用相同的方法进行维护会遇到什么问题？是不是可以用其他结构计数？
    

代码：

```rust
pub struct TaskControlBlock {
    /// The task status in it's lifecycle
    pub task_status: TaskStatus,
    /// The task context
    pub task_cx: TaskContext,
    /// 系统调用次数
    pub syscall_times:[u32;MAX_SYSCALL_NUM],
    /// 任务开始时间
    pub start_time:usize,
}
```

```rust
   //另外在lazy_static中需要对syscall_times以及start_time进行初始化
...
    ///当前任务的TCB
    fn get_current_task(&self)->TaskControlBlock{
        let inner=self.inner.exclusive_access();
        let current=inner.current_task;
        inner.tasks[current]
    }
    ///增加系统调用次数
    fn inc_syscall_times(&self,syscall_id:usize){
        let mut inner=self.inner.exclusive_access();
        let current=inner.current_task;
        inner.tasks[current].syscall_times[syscall_id]+=1;
    }
}

...
///get tcb
pub fn get_current_task()->TaskControlBlock{
    TASK_MANAGER.get_current_task()
}
///inc 
pub fn inc_syscall_times(syscall_id:usize){
    TASK_MANAGER.inc_syscall_times(syscall_id);
}
```

以下是获取当前的任务状态的函数

```rust
/// YOUR JOB: Finish sys_task_info to pass testcases
pub fn sys_task_info(ti: *mut TaskInfo) -> isize {
    trace!("kernel: sys_task_info");
    let tcb=get_current_task();
    unsafe {
        (*ti).time=get_time_ms()-tcb.start_time;
        (*ti).syscall_times=tcb.syscall_times;
        (*ti).status=tcb.task_status;
    }
    0
}
```

在os/src/syscall/mod.rs中，在syscall调用中需要调用inc_syscall_times来增加系统调用次数。

运行ci测试

![pic1](/images/rCore/ch3-1.png)