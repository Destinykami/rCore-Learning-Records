---
title: 'rCore-ch7'
date: 2024/5/7
permalink: /posts/2024/05/rCore_ch7/
excerpt: '第七章 进程间通信与I/O重定向'
tags:
  - rCore
  - OS
---
# 第七章 进程间通信与I/O重定向
在本章之前，进程被操作系统彻底隔离了。进程间无法方便地“沟通”，导致进程不能一起协作干“大事”。如果能让不同进程实现数据共享与交互，就能把不同程序的功能组合在一起，实现更加强大和灵活的功能。为了让简单的应用程序能够组合在一起形成各种强大和复杂的功能，本章要完成的操作系统的核心目标是： **让不同应用通过进程间通信的方式组合在一起运行** 。
## 基于文件的标准输入/输出
本节我们介绍为何要把标准输入/输出用文件来进行抽象，以及如何以文件和文件描述符概念来重新定义标准输入/输出，并在进程中把各种文件描述符组织到文件描述符表中，同时将进程对于标准输入输出的访问修改为基于文件抽象的接口实现。
### 一切皆是文件
在UNIX操作系统之前，大多数的操作系统提供了各种复杂且不规则的设计实现来处理各种I/O设备（也可称为I/O资源），如键盘、显示器、以磁盘为代表的存储介质、以串口为代表的通信设备等，使得应用程序开发繁琐且很难统一表示和处理I/O设备。随着UNIX的诞生，一个简洁优雅的I/O设备抽象出现了，这就是 文件 。在 UNIX 操作系统中，”一切皆文件“ (Everything is a file) 是一种重要的设计思想，这种设计思想继承于 Multics 操作系统的 通用性 文件的设计理念，并进行了进一步的简化。应用程序访问的 文件 (File) 就是一系列的字节组合。操作系统管理文件，但操作系统不关心文件内容，只关心如何对文件按字节流进行读写的机制，这就意味着任何程序可以读写任何文件（即字节流），对文件具体内容的解析是应用程序的任务，操作系统对此不做任何干涉。例如，一个Rust编译器可以读取一个C语言源程序并进行编译，操作系统并并不会阻止这样的事情发生。  

有了文件这样的抽象后，操作系统内核就可把能读写的I/O资源按文件来进行管理，并把文件分配给进程，让进程以统一的文件访问接口与I/O 资源进行交互。在目前和后续可能涉及到的I/O硬件设备中，大致可以分成以下几种：

* **键盘设备** 是程序获得字符输入的一种设备，也可抽象为一种只读性质的文件，可以从这个文件中读出一系列的字节序列；

* **屏幕设备** 是展示程序的字符输出结果的一种字符显示设备，可抽象为一种只写性质的文件，可以向这个文件中写入一系列的字节序列，在显示屏上可以直接呈现出来；

* **串口设备** 是获得字符输入和展示程序的字符输出结果的一种字符通信设备，可抽象为一种可读写性质的文件，可以向这个文件中写入一系列的字节序列传给程序，也可把程序要显示的字符传输出去；还可以把这个串口设备拆分成两个文件，一个用于获取输入字符的只读文件和一个传出输出字符的只写文件。  

### 标准输入/输出对 File trait 的实现
我们把标准输出设备在文件描述符表中的文件描述符的值规定为 1 ，用 Stdout 表示；把标准输入设备在文件描述符表中的文件描述符的值规定为 0，用 Stdin 表示 。现在，我们可以重构操作系统，为标准输入和标准输出实现 File Trait，使得进程可以按文件接口与I/O外设进行交互:  
```rust  
// os/src/fs/stdio.rs

pub struct Stdin;

pub struct Stdout;

impl File for Stdin {
    fn read(&self, mut user_buf: UserBuffer) -> usize {
        assert_eq!(user_buf.len(), 1);
        // busy loop
        let mut c: usize;
        loop {
            c = console_getchar();
            if c == 0 {
                suspend_current_and_run_next();
                continue;
            } else {
                break;
            }
        }
        let ch = c as u8;
        unsafe { user_buf.buffers[0].as_mut_ptr().write_volatile(ch); }
        1
    }
    fn write(&self, _user_buf: UserBuffer) -> usize {
        panic!("Cannot write to stdin!");
    }
}

impl File for Stdout {
    fn read(&self, _user_buf: UserBuffer) -> usize{
        panic!("Cannot read from stdout!");
    }
    fn write(&self, user_buf: UserBuffer) -> usize {
        for buffer in user_buf.buffers.iter() {
            print!("{}", core::str::from_utf8(*buffer).unwrap());
        }
        user_buf.len()
    }
}
```
### 对标准输入/输出的管理
当一个进程被创建的时候，内核会默认为其打开三个缺省就存在的文件：

* 文件描述符为 0 的标准输入

* 文件描述符为 1 的标准输出

* 文件描述符为 2 的标准错误输出

在我们的实现中并不区分标准输出和标准错误输出，而是会将文件描述符 1 和 2 均对应到标准输出。  

这里隐含着有关文件描述符的一条重要的规则：即进程打开一个文件的时候，内核总是会将文件分配到该进程文件描述符表中 **最小的** 空闲位置。比如，当一个进程被创建以后立即打开一个文件，则内核总是会返回文件描述符 3 （0~2号文件描述符已被缺省打开了）。当我们关闭一个打开的文件之后，它对应的文件描述符将会变得空闲并在后面可以被分配出去。  
## 管道
管道是一种进程间通信机制，由操作系统提供，并可通过直接编程或在shell程序的帮助下轻松地把不同进程（目前是父子进程之间或子子进程之间）的输入和输出对接起来。我们也可以将管道看成一个有一定缓冲区大小的字节队列，它分为读和写两端，需要通过不同的文件描述符来访问。读端只能用来从管道中读取，而写端只能用来将数据写入管道。由于管道是一个队列，读取数据的时候会从队头读取并弹出数据，而写入数据的时候则会把数据写入到队列的队尾。由于管道的缓冲区大小是有限的，一旦整个缓冲区都被填满就不能再继续写入，就需要等到读端读取并从队列中弹出一些数据之后才能继续写入。当缓冲区为空的时候，读端自然也不能继续从里面读取数据，需要等到写端写入了一些数据之后才能继续读取。
### 基于文件的管道
我们将管道的一端（读端或写端）抽象为 Pipe 类型：
```rust
// os/src/fs/pipe.rs

pub struct Pipe {
    readable: bool,
    writable: bool,
    buffer: Arc<Mutex<PipeRingBuffer>>,
}
```
`readable` 和 `writable` 分别指出该管道端可否支持读取/写入，通过 `buffer` 字段还可以找到该管道端所在的管道自身。后续我们将为它实现 `File Trait` ，之后它便可以通过文件描述符来访问。

而管道自身，也就是那个带有一定大小缓冲区的字节队列，我们抽象为 `PipeRingBuffer` 类型：
```rust
// os/src/fs/pipe.rs

const RING_BUFFER_SIZE: usize = 32;

#[derive(Copy, Clone, PartialEq)]
enum RingBufferStatus {
    FULL,
    EMPTY,
    NORMAL,
}

pub struct PipeRingBuffer {
    arr: [u8; RING_BUFFER_SIZE],
    head: usize,
    tail: usize,
    status: RingBufferStatus,
    write_end: Option<Weak<Pipe>>,
}
```
### 管道读写
首先来看如何为 `Pipe` 实现 `File Trait` 的 `read `方法，即从管道的读端读取数据。在此之前，我们需要对于管道循环队列进行封装来让它更易于使用：
```rust
// os/src/fs/pipe.rs

impl PipeRingBuffer {
    pub fn read_byte(&mut self) -> u8 {
        self.status = RingBufferStatus::NORMAL;
        let c = self.arr[self.head];
        self.head = (self.head + 1) % RING_BUFFER_SIZE;
        if self.head == self.tail {
            self.status = RingBufferStatus::EMPTY;
        }
        c
    }
    pub fn available_read(&self) -> usize {
        if self.status == RingBufferStatus::EMPTY {
            0
        } else {
            if self.tail > self.head {
                self.tail - self.head
            } else {
                self.tail + RING_BUFFER_SIZE - self.head
            }
        }
    }
    pub fn all_write_ends_closed(&self) -> bool {
        self.write_end.as_ref().unwrap().upgrade().is_none()
    }
}
```
`PipeRingBuffer::read_byte` 方法可以从管道中读取一个字节，注意在调用它之前需要确保管道缓冲区中不是空的。它会更新循环队列队头的位置，并比较队头和队尾是否相同，如果相同的话则说明管道的状态变为空 `EMPTY` 。仅仅通过比较队头和队尾是否相同不能确定循环队列是否为空，因为它既有可能表示队列为空，也有可能表示队列已满。因此我们需要在 `read_byte` 的同时进行状态更新。

`PipeRingBuffer::available_read` 可以计算管道中还有多少个字符可以读取。我们首先需要判断队列是否为空，因为队头和队尾相等可能表示队列为空或队列已满，两种情况 `available_read` 的返回值截然不同。如果队列为空的话直接返回 0，否则根据队头和队尾的相对位置进行计算。

`PipeRingBuffer::all_write_ends_closed` 可以判断管道的所有写端是否都被关闭了，这是通过尝试将管道中保存的写端的弱引用计数升级为强引用计数来实现的。如果升级失败的话，说明管道写端的强引用计数为 0 ，也就意味着管道所有写端都被关闭了，从而管道中的数据不会再得到补充，待管道中仅剩的数据被读取完毕之后，管道就可以被销毁了。

下面是 `Pipe` 的 `read` 方法的实现：
```rust
// os/src/fs/pipe.rs

impl File for Pipe {
    fn read(&self, buf: UserBuffer) -> usize {
        assert!(self.readable());
        let want_to_read = buf.len();
        let mut buf_iter = buf.into_iter();
        let mut already_read = 0usize;
        loop {
            let mut ring_buffer = self.buffer.exclusive_access();
            let loop_read = ring_buffer.available_read();
            if loop_read == 0 {
                if ring_buffer.all_write_ends_closed() {
                    return already_read;
                }
                drop(ring_buffer);
                suspend_current_and_run_next();
                continue;
            }
            for _ in 0..loop_read {
                if let Some(byte_ref) = buf_iter.next() {
                    unsafe {
                        *byte_ref = ring_buffer.read_byte();
                    }
                    already_read += 1;
                    if already_read == want_to_read {
                        return want_to_read;
                    }
                } else {
                    return already_read;
                }
            }
        }
    }
}
```
第 7 行的 `buf_iter` 将传入的应用缓冲区 `buf` 转化为一个能够逐字节对于缓冲区进行访问的迭代器，每次调用 `buf_iter.next()` 即可按顺序取出用于访问缓冲区中一个字节的裸指针。

第 8 行的 `already_read` 用来维护实际有多少字节从管道读入应用的缓冲区。

`File::read` 的语义是要从文件中最多读取应用缓冲区大小那么多字符。这可能超出了循环队列的大小，或者由于尚未有进程从管道的写端写入足够的字符，因此我们需要将整个读取的过程放在一个循环中，当循环队列中不存在足够字符的时候暂时进行任务切换，等待循环队列中的字符得到补充之后再继续读取。

这个循环从第 9 行开始，第 11 行我们用 `loop_read` 来表示循环这一轮次中可以从管道循环队列中读取多少字符。如果管道为空则会检查管道的所有写端是否都已经被关闭，如果是的话，说明我们已经没有任何字符可以读取了，这时可以直接返回；否则我们需要等管道的字符得到填充之后再继续读取，因此我们调用 `suspend_current_and_run_next` 切换到其他任务，等到切换回来之后回到循环开头再看一下管道中是否有字符了。在调用之前我们需要手动释放管道自身的锁，因为切换任务时候的 `__switch` 跨越了正常函数调用的边界。

如果 `loop_read` 不为 0 ，在这一轮次中管道中就有`loop_read` 个字节可以读取。我们可以迭代应用缓冲区中的每个字节指针，并调用 `PipeRingBuffer::read_byte` 方法来从管道中进行读取。如果这 `loop_read` 个字节均被读取之后还没有填满应用缓冲区，就需要进入循环的下一个轮次，否则就可以直接返回了。在具体实现的时候注意边界条件的判断。

## 命令行参数
 argv
## 标准 I/O 重定向

## 编程作业
### 进程通信：邮箱
这一章我们实现了基于 pipe 的进程间通信，但是看测例就知道了，管道不太自由，我们来实现一套乍一看更靠谱的通信 syscall吧！本节要求实现邮箱机制，以及对应的 syscall。

邮箱说明：每个进程拥有唯一一个邮箱，基于“数据报”收发字节信息，利用环形buffer存储，读写顺序为 FIFO，不记录来源进程。每次读写单位必须为一个报文，如果用于接收的缓冲区长度不够，舍弃超出的部分（截断报文）。为了简单，邮箱中最多拥有16条报文，每条报文最大长度256字节。当邮箱满时，发送邮件（也就是写邮箱）会失败。不考虑读写邮箱的权限，也就是所有进程都能够随意给其他进程的邮箱发报。

**mailread:**

* syscall ID：401

* Rust接口: fn mailread(buf: *mut u8, len: usize)

* 功能：读取一个报文，如果成功返回报文长度.

* 参数：
  * buf: 缓冲区头。

  * len：缓冲区长度。

* 说明：
  * len > 256 按 256 处理，len < 队首报文长度且不为0，则截断报文。

  * len = 0，则不进行读取，如果没有报文读取，返回-1，否则返回0，这是用来测试是否有报文可读。

* 可能的错误：
  *   邮箱空。

  * buf 无效。

**mailwrite:**

* syscall ID：402

* Rust接口: fn mailwrite(pid: usize, buf: *mut u8, len: usize)

* 功能：向对应进程邮箱插入一条报文.

* 参数：
  *  pid: 目标进程id。

  * buf: 缓冲区头。

  * len：缓冲区长度。

* 说明：
  * len > 256 按 256 处理，

  * len = 0，则不进行写入，如果邮箱满，返回-1，否则返回0，这是用来测试是否可以发报。

  * 可以向自己的邮箱写入报文。

* 可能的错误：
  * 邮箱满。

  * buf 无效。

首先在config.rs里加上
```rust
/// 邮箱中的最大报文数
pub const MAX_MESSAGE_NUM: usize = 16;
/// 每条报文的最大长度
pub const MAX_MAIL_LENGTH: usize = 256;
```

在os/src/fs中仿照pipe.rs新建mailbox.rs，定义了MailBox结构体，将来要加入进程的PCB中。
```rust
//! mail message between different processes

use crate::sync::UPSafeCell;
use crate::config::{MAX_MAIL_LENGTH, MAX_MESSAGE_NUM};


#[derive(Clone, Copy)]
pub struct Mail {
    pub data: [u8; MAX_MAIL_LENGTH],
    pub len: usize,
}

impl Mail {
    fn new() -> Self {
        Self { data: [0; MAX_MAIL_LENGTH], len: 0 }
    }
}

pub struct MailBox {
    pub buffer: UPSafeCell<MailBoxInner>,
}

impl MailBox {
    pub fn new() -> Self {
        unsafe {
            Self { buffer: UPSafeCell::new(MailBoxInner::new()) }
        }
    }
}

#[derive(Clone, Copy, PartialEq)]
pub enum MailBoxStatus {
    Full,
    Empty,
    Normal,
}

pub struct MailBoxInner {
    pub arr: [Mail; MAX_MESSAGE_NUM],
    pub head: usize,
    pub tail: usize,
    pub status: MailBoxStatus,
}

impl MailBoxInner {
    pub fn new() -> Self {
        Self {
            arr: [Mail::new(); MAX_MESSAGE_NUM],
            head: 0,
            tail: 0,
            status: MailBoxStatus::Empty,
        }
    }

    pub fn is_full(&self) -> bool {
        self.status == MailBoxStatus::Full
    }

    pub fn is_empty(&self) -> bool {
        self.status == MailBoxStatus::Empty
    }

    pub fn available_read(&self) -> usize {
        if self.status == MailBoxStatus::Empty {
            0
        } else if self.tail > self.head {
            self.tail - self.head
        } else {
            self.tail + MAX_MAIL_LENGTH - self.head
        }
    }
    pub fn available_write(&self) -> usize {
        if self.status == MailBoxStatus::Full {
            0
        } else {
            MAX_MAIL_LENGTH - self.available_read()
        }
    }
}
```
在os/src/syscall/mod.rs中新增系统调用
```rust
        SYSCALL_MAIL_READ => sys_mailread(args[0] as *mut u8,args[1] as usize),
        SYSCALL_MAIL_WRITE => sys_mailwrite(args[0] as usize,args[1] as *mut u8,args[2] as usize),
```
在os/src/syscall/fs.rs中完善:
```rust
pub fn sys_mailread(buf: *mut u8, len: usize)->isize{
    let task = current_task().unwrap();
    let inner = task.inner_exclusive_access();
    let token = inner.get_user_token();
    let mut mailbox_inner = inner.mailbox.buffer.exclusive_access();
    if len == 0 {
        if mailbox_inner.is_empty(){
            println!("Len=0, The MailBox is empty!");
            return -1;
        }
        println!("Len=0, The MailBox is not empty!");
        return 0;
    }
    if mailbox_inner.is_empty() {
        println!("Can't Read, The MailBox is empty!");
        return -1;
    }
    let mailbox_head = mailbox_inner.head; 
    // the truncated mail length
    let mlen = len.min(mailbox_inner.arr[mailbox_head].len);
    let dst_vec = translated_byte_buffer(token, buf, mlen);
    let src_ptr = mailbox_inner.arr[mailbox_head].data.as_ptr();

    for (idx, dst) in dst_vec.into_iter().enumerate() {
        unsafe {
            dst.copy_from_slice(
                core::slice::from_raw_parts(
                    src_ptr.wrapping_add(idx) as *const u8,
                    core::mem::size_of::<u8>() *mlen
                    )
            );
        }
    }
    mailbox_inner.status = MailBoxStatus::Normal;
    mailbox_inner.head = (mailbox_head + 1) % MAX_MAIL_LENGTH;
    if mailbox_inner.head == mailbox_inner.tail {
        mailbox_inner.status = MailBoxStatus::Empty;
    }
    println!("Read a mail");
    mlen as isize
}
pub fn sys_mailwrite(pid:usize,buf: *mut u8, len: usize)->isize{
    if core::ptr::null() == buf {
        return -1;
    }
    if let Some(target_task) = pid2task(pid) {
        let target_task_ref = target_task.inner_exclusive_access();
        let token = target_task_ref.get_user_token();
        let mut mailbox_inner = target_task_ref.mailbox.buffer.exclusive_access();
        if len == 0 {
            if mailbox_inner.is_full() {
                println!("Len=0, The MailBox is full!");
                return -1;
            }
            println!("Len=0, The MailBox is not full!");
            return 0;
        }
        if mailbox_inner.is_full() {
            return -1;
        }
        let mailbox_tail = mailbox_inner.tail;
        mailbox_inner.status = MailBoxStatus::Normal;
        // the truncated mail length
        let mlen = len.min(MAX_MAIL_LENGTH);
        // prepare source data
        let src_vec: alloc::vec::Vec<&mut [u8]> = translated_byte_buffer(token, buf, mlen);
        
        //copy from source to dst
        for (idx, src) in src_vec.into_iter().enumerate() {
            let slice_length=src.len();
            let destination=&mut mailbox_inner.arr[mailbox_tail].data[idx*slice_length..(idx+1)*slice_length];
            destination.copy_from_slice(src);
            
        }
        // store the mail length
        mailbox_inner.arr[mailbox_tail].len = mlen;
        mailbox_inner.tail = (mailbox_tail + 1) % MAX_MESSAGE_NUM;
        if mailbox_inner.tail == mailbox_inner.head {
            mailbox_inner.status = MailBoxStatus::Full;
        }
        println!("Write a mail");
        return 0;
    }
    -1
}
```
思路参照`pipe`循环队列的实现，`writemail`按照`pid`取出进程(`pid2task`)，然后获得进程的`MailBox`的可变引用，向其中新增要投递的邮件，注意arr数组是一个循环队列，要对队列的头指针和尾指针进行判断。  
`readmail`获得当前任务的可变引用，读取一条信息，指针移动。

* `test_mail0`:测试邮箱基本功能  
* `test_mail1`:测试邮箱容量  
* `test_mail2`:双进程邮箱测试 (还未完善，会卡住，不知道是什么问题 Todo)     
* `test_mail3`:邮箱错误参数测试  
```text
Rust user shell
>> ch7b_test_mail0
Write a mail
Read a mail
mail0 test OK!
Shell: Process 2 exited with code 0
>> ch7b_test_mail1
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Read a mail
Write a mail
mail1 test OK!
Shell: Process 2 exited with code 0
>> ch7b_test_mail3
Len=0, The MailBox is not full!
Len=0, The MailBox is empty!
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Write a mail
Len=0, The MailBox is full!
Len=0, The MailBox is not empty!
Len=0, The MailBox is full!
mail3 test OK!
Shell: Process 2 exited with code 0
```