---
title: 'rCore-ch6'
date: 2024/1/17
permalink: /posts/2024/01/rCore_ch6/
excerpt: '第六章 文件系统'
tags:
  - rCore
  - OS
---
# 第六章 文件系统
文件最早来自于计算机用户需要把数据持久保存在 **持久存储设备** 上的需求。由于放在内存中的数据在计算机关机或掉电后就会消失，所以应用程序要把内存中需要保存的数据放到 持久存储设备的数据块（比如磁盘的扇区等）中存起来。  
随着操作系统功能的增强，在操作系统的管理下，应用程序不用理解持久存储设备的硬件细节，而只需对 **文件** 这种持久存储数据的抽象进行读写就可以了，由操作系统中的文件系统和存储设备驱动程序一起来完成繁琐的持久存储设备的管理与读写。所以本章要完成的操作系统的第一个核心目标是： **让应用能够方便地把数据持久保存起来** 。
## 简易文件系统 easy-fs
`easy-fs` crate 自下而上大致可以分成五个不同的层次：

1. 磁盘块设备接口层：定义了以块大小为单位对磁盘块设备进行读写的trait接口

2. 块缓存层：在内存中缓存磁盘块的数据，避免频繁读写磁盘

3. 磁盘数据结构层：磁盘上的超级块、位图、索引节点、数据块、目录项等核心数据结构和相关处理

4. 磁盘块管理器层：合并了上述核心数据结构和磁盘布局所形成的磁盘文件系统数据结构，以及基于这些结构的创建/打开文件系统的相关处理和磁盘块的分配和回收处理

5. 索引节点层：管理索引节点（即文件控制块）数据结构，并实现文件创建/文件打开/文件读写等成员函数来向上支持文件操作相关的系统调用

### 块设备接口层
在 `easy-fs` 库的最底层声明了一个块设备的抽象接口 `BlockDevice` ：
```rust
// easy-fs/src/block_dev.rs

pub trait BlockDevice : Send + Sync + Any {
    fn read_block(&self, block_id: usize, buf: &mut [u8]);
    fn write_block(&self, block_id: usize, buf: &[u8]);
}
```
它需要实现两个抽象方法：

`read_block` 将编号为 `block_id` 的块从磁盘读入内存中的缓冲区 `buf` ；

`write_block` 将内存中的缓冲区 `buf` 中的数据写入磁盘编号为 `block_id` 的块。

在 `easy-fs` 中并没有一个实现了 `BlockDevice Trait` 的具体类型。因为块设备仅支持以块为单位进行随机读写，所以需要由具体的块设备驱动来实现这两个方法，实际上这是需要由文件系统的使用者（比如操作系统内核或直接测试 `easy-fs` 文件系统的 `easy-fs-fuse` 应用程序）提供并接入到 `easy-fs` 库的。 `easy-fs` 库的块缓存层会调用这两个方法，进行块缓存的管理。这也体现了 `easy-fs` 的泛用性：它可以访问实现了 `BlockDevice Trait` 的块设备驱动程序。
> 实际上，块和扇区是两个不同的概念。 扇区 (Sector) 是块设备随机读写的数据单位，通常每个扇区为 512 字节。而块是文件系统存储文件时的数据单位，每个块的大小等同于一个或多个扇区。之前提到过 Linux 的Ext4文件系统的单个块大小默认为 4096 字节。在我们的 easy-fs 实现中一个块和一个扇区同为 512 字节，因此在后面的讲解中我们不再区分扇区和块的概念。  

### 块缓存层
由于操作系统频繁读写速度缓慢的磁盘块会极大降低系统性能，因此常见的手段是先通过 read_block 将一个块上的数据从磁盘读到内存中的一个缓冲区中，这个缓冲区中的内容是可以直接读写的，那么后续对这个数据块的大部分访问就可以在内存中完成了。如果缓冲区中的内容被修改了，那么后续还需要通过 write_block 将缓冲区中的内容写回到磁盘块中。  
>事实上，无论站在代码实现鲁棒性还是性能的角度，将这些缓冲区合理的管理起来都是很有必要的。一种完全不进行任何管理的模式可能是：每当要对一个磁盘块进行读写的时候，都通过 read_block 将块数据读取到一个 临时 创建的缓冲区，并在进行一些操作之后（可选地）将缓冲区的内容写回到磁盘块。从性能上考虑，我们需要尽可能降低实际块读写（即 read/write_block ）的次数，因为每一次调用它们都会产生大量开销。要做到这一点，关键就在于对块读写操作进行 合并 。例如，如果一个块已经被读到缓冲区中了，那么我们就没有必要再读一遍，直接用已有的缓冲区就行了；同时，对于缓冲区中的同一个块的多次修改没有必要每次都写回磁盘，只需等所有的修改都结束之后统一写回磁盘即可。

当磁盘上的数据结构比较复杂的时候，很难通过应用来合理地规划块读取/写入的时机。这不仅可能涉及到复杂的参数传递，稍有不慎还有可能引入同步性问题(目前可以暂时忽略)：即一个块缓冲区修改后的内容在后续的同一个块读操作中不可见，这很致命但又难以调试。

我们的做法是将缓冲区统一管理起来。当我们要读写一个块的时候，首先就是去全局管理器中查看这个块是否已被缓存到内存缓冲区中。如果是这样，则在一段连续时间内对于一个块进行的所有操作均是在同一个固定的缓冲区中进行的，这解决了同步性问题。此外，通过 read/write_block 进行块实际读写的时机完全交给块缓存层的全局管理器处理，上层子系统无需操心。全局管理器会尽可能将更多的块操作合并起来，并在必要的时机发起真正的块实际读写。

### 块缓存
定义如下：
```rust
// easy-fs/src/lib.rs

pub const BLOCK_SZ: usize = 512;

// easy-fs/src/block_cache.rs

pub struct BlockCache {
    cache: [u8; BLOCK_SZ],
    block_id: usize,
    block_device: Arc<dyn BlockDevice>,
    modified: bool,
}
```
其中：

* cache 是一个 512 字节的数组，表示位于内存中的缓冲区；

* block_id 记录了这个块缓存来自于磁盘中的块的编号；

* block_device 是一个底层块设备的引用，可通过它进行块读写；

* modified 记录这个块从磁盘载入内存缓存之后，它有没有被修改过。

当我们创建一个 BlockCache 的时候，这将触发一次 read_block 将一个块上的数据从磁盘读到缓冲区 cache ：
```rust
// easy-fs/src/block_cache.rs

impl BlockCache {
    /// Load a new BlockCache from disk.
    pub fn new(
        block_id: usize,
        block_device: Arc<dyn BlockDevice>
    ) -> Self {
        let mut cache = [0u8; BLOCK_SZ];
        block_device.read_block(block_id, &mut cache);
        Self {
            cache,
            block_id,
            block_device,
            modified: false,
        }
    }
}
```
一旦磁盘块已经存在于内存缓存中，CPU 就可以直接访问磁盘块数据了：
```rust
// easy-fs/src/block_cache.rs

impl BlockCache {
    fn addr_of_offset(&self, offset: usize) -> usize {
        &self.cache[offset] as *const _ as usize
    }

    pub fn get_ref<T>(&self, offset: usize) -> &T where T: Sized {
        let type_size = core::mem::size_of::<T>();
        assert!(offset + type_size <= BLOCK_SZ);
        let addr = self.addr_of_offset(offset);
        unsafe { &*(addr as *const T) }
    }

    pub fn get_mut<T>(&mut self, offset: usize) -> &mut T where T: Sized {
        let type_size = core::mem::size_of::<T>();
        assert!(offset + type_size <= BLOCK_SZ);
        self.modified = true;
        let addr = self.addr_of_offset(offset);
        unsafe { &mut *(addr as *mut T) }
    }
}
```
* `addr_of_offset` 可以得到一个 `BlockCache` 内部的缓冲区中指定偏移量 `offset` 的字节地址；

* `get_ref` 是一个泛型方法，它可以获取缓冲区中的位于偏移量 `offset` 的一个类型为 T 的磁盘上数据结构的不可变引用。该泛型方法的 `Trait Bound` 限制类型 T 必须是一个编译时已知大小的类型，我们通过 `core::mem::size_of::<T>()` 在编译时获取类型 T 的大小，并确认该数据结构被整个包含在磁盘块及其缓冲区之内。

* `get_mut` 与 `get_ref` 的不同之处在于，`get_mut` 会获取磁盘上数据结构的可变引用，由此可以对数据结构进行修改。由于这些数据结构目前位于内存中的缓冲区中，我们需要将 `BlockCache` 的 `modified` 标记为 `true`表示该缓冲区已经被修改，之后需要将数据写回磁盘块才能真正将修改同步到磁盘。 

可以将 `get_ref/get_mut` 进一步封装为更为易用的形式：
```rust
// easy-fs/src/block_cache.rs

impl BlockCache {
    pub fn read<T, V>(&self, offset: usize, f: impl FnOnce(&T) -> V) -> V {
        f(self.get_ref(offset))
    }

    pub fn modify<T, V>(&mut self, offset:usize, f: impl FnOnce(&mut T) -> V) -> V {
        f(self.get_mut(offset))
    }
}
```

### 块缓存全局管理器
为了避免在块缓存上浪费过多内存，我们希望内存中同时只能驻留有限个磁盘块的缓冲区。 
```rust 
// easy-fs/src/block_cache.rs

const BLOCK_CACHE_SIZE: usize = 16;
```
块缓存全局管理器的功能是：当我们要对一个磁盘块进行读写时，首先看它是否已经被载入到内存缓存中了，如果已经被载入的话则直接返回，否则需要先读取磁盘块的数据到内存缓存中。此时，如果内存中驻留的磁盘块缓冲区的数量已满，则需要遵循某种缓存替换算法将某个块的缓存从内存中移除，再将刚刚读到的块数据加入到内存缓存中。我们这里使用一种类 FIFO 的简单缓存替换算法，因此在管理器中只需维护一个队列：
```rust
// easy-fs/src/block_cache.rs

use alloc::collections::VecDeque;

pub struct BlockCacheManager {
    queue: VecDeque<(usize, Arc<Mutex<BlockCache>>)>,
}

impl BlockCacheManager {
    pub fn new() -> Self {
        Self { queue: VecDeque::new() }
    }
}
```
`get_block_cache` 方法尝试从块缓存管理器中获取一个编号为 `block_id` 的块的块缓存，如果找不到，会从磁盘读取到内存中，还有可能会发生缓存替换：
```rust
// easy-fs/src/block_cache.rs

impl BlockCacheManager {
    pub fn get_block_cache(
        &mut self,
        block_id: usize,
        block_device: Arc<dyn BlockDevice>,
    ) -> Arc<Mutex<BlockCache>> {
        if let Some(pair) = self.queue
            .iter()
            .find(|pair| pair.0 == block_id) {
                Arc::clone(&pair.1)
        } else {
            // substitute
            if self.queue.len() == BLOCK_CACHE_SIZE {
                // from front to tail
                if let Some((idx, _)) = self.queue
                    .iter()
                    .enumerate()
                    .find(|(_, pair)| Arc::strong_count(&pair.1) == 1) {
                    self.queue.drain(idx..=idx);
                } else {
                    panic!("Run out of BlockCache!");
                }
            }
            // load block into mem and push back
            let block_cache = Arc::new(Mutex::new(
                BlockCache::new(block_id, Arc::clone(&block_device))
            ));
            self.queue.push_back((block_id, Arc::clone(&block_cache)));
            block_cache
        }
    }
}
```
这个函数的作用是：首先遍历整个队列试图找到一个编号相同的块缓存，如果找到了，会将块缓存管理器中保存的块缓存的引用复制一份并返回；如果找不到，此时必须将块从磁盘读入内存中的缓冲区。在实际读取之前，需要判断管理器保存的块缓存数量是否已经达到了上限。如果达到了上限（第 15 行）才需要执行缓存替换算法，丢掉某个块缓存并空出一个空位。但此时队头对应的块缓存可能仍在使用：判断的标志是其强引用计数>=2 ，即除了块缓存管理器保留的一份副本之外，在外面还有若干份副本正在使用。因此，我们的做法是从队头遍历到队尾找到第一个强引用计数恰好为 1 的块缓存并将其替换出去。如果队列已满且其中所有的块缓存都正在使用，则panic退出。在最后创建一个新的块缓存，并加入到队尾，返回给用户。  
接下来需要创建 BlockCacheManager 的全局实例：
```rust
// easy-fs/src/block_cache.rs

lazy_static! {
    pub static ref BLOCK_CACHE_MANAGER: Mutex<BlockCacheManager> = Mutex::new(
        BlockCacheManager::new()
    );
}

pub fn get_block_cache(
    block_id: usize,
    block_device: Arc<dyn BlockDevice>
) -> Arc<Mutex<BlockCache>> {
    BLOCK_CACHE_MANAGER.lock().get_block_cache(block_id, block_device)
}
```
这样对于其他模块而言，就可以直接通过 `get_block_cache` 方法来请求块缓存了。这里需要指出的是，它返回的是一个 `Arc<Mutex<BlockCache>>` ，调用者需要通过 `.lock()` 获取里层互斥锁 `Mutex` 才能对最里面的 `BlockCache` 进行操作，比如通过 `read/modify` 访问缓冲区里面的磁盘数据结构。`Mutex` 则避免在后续多核拓展时的访存冲突。

## 磁盘布局及磁盘上数据结构
对于一个文件系统而言，最重要的功能是如何将一个逻辑上的文件目录树结构映射到磁盘上，决定磁盘上的每个块应该存储文件相关的哪些数据。为了更容易进行管理和更新，我们需要将磁盘上的数据组织为若干种不同的磁盘上数据结构，并合理安排它们在磁盘中的位置。

### easy-fs 磁盘布局概述
在 easy-fs 磁盘布局中，按照块编号从小到大顺序地分成 5 个不同属性的连续区域：

* 最开始的区域的长度为一个块，其内容是 easy-fs 超级块 (Super Block)。超级块内以魔数的形式提供了文件系统合法性检查功能，同时还可以定位其他连续区域的位置。  

超级块是一个磁盘上数据结构，它就存放在磁盘上编号为 0 的块的起始处.SuperBlock 的内容如下：  
```rust
// easy-fs/src/layout.rs

#[repr(C)]
pub struct SuperBlock {
    magic: u32,
    pub total_blocks: u32,
    pub inode_bitmap_blocks: u32,
    pub inode_area_blocks: u32,
    pub data_bitmap_blocks: u32,
    pub data_area_blocks: u32,
}
```

* 第二个区域是一个索引节点位图，长度为若干个块。它记录了后面的索引节点区域中有哪些索引节点已经被分配出去使用了，而哪些还尚未被分配出去。

* 第三个区域是索引节点区域，长度为若干个块。其中的每个块都存储了若干个索引节点。

* 第四个区域是一个数据块位图，长度为若干个块。它记录了后面的数据块区域中有哪些数据块已经被分配出去使用了，而哪些还尚未被分配出去。

* 最后的区域则是数据块区域，顾名思义，其中的每一个已经分配出去的块保存了文件或目录中的具体数据内容。  

### 磁盘上的索引节点
每个文件/目录在磁盘上均以一个 DiskInode 的形式存储。其中包含文件/目录的元数据： size 表示文件/目录内容的字节数， type_ 表示索引节点的类型 DiskInodeType ，目前仅支持文件 File 和目录 Directory 两种类型。其余的 direct/indirect1/indirect2 都是存储文件内容/目录内容的数据块的索引，这也是索引节点名字的由来。  
```rust
// easy-fs/src/layout.rs

const INODE_DIRECT_COUNT: usize = 28;

#[repr(C)]
pub struct DiskInode {
    pub size: u32,
    pub direct: [u32; INODE_DIRECT_COUNT],
    pub indirect1: u32,
    pub indirect2: u32,
    type_: DiskInodeType,
}

#[derive(PartialEq)]
pub enum DiskInodeType {
    File,
    Directory,
}
```
为了尽可能节约空间，在进行索引的时候，块的编号用一个 u32 存储。索引方式分成直接索引和间接索引两种：

* 当文件很小的时候，只需用到直接索引， direct 数组中最多可以指向 INODE_DIRECT_COUNT 个数据块，当取值为 28 的时候，通过直接索引可以找到 14KiB 的内容。

* 当文件比较大的时候，不仅直接索引的 direct 数组装满，还需要用到一级间接索引 indirect1 。它指向一个一级索引块，这个块也位于磁盘布局的数据块区域中。这个一级索引块中的每个 u32 都用来指向数据块区域中一个保存该文件内容的数据块，因此，最多能够索引512/4=128 个数据块，对应 64KiB 的内容。

* 当文件大小超过直接索引和一级索引支持的容量上限 78KiB 的时候，就需要用到二级间接索引 indirect2 。它指向一个位于数据块区域中的二级索引块。二级索引块中的每个 u32 指向一个不同的一级索引块，这些一级索引块也位于数据块区域中。因此，通过二级间接索引最多能够索引 <math xmlns="http://www.w3.org/1998/Math/MathML">
  <mn>128</mn>
  <mo>&#xD7;</mo>
  <mn>64</mn>
  <mtext>KiB</mtext>
  <mo>=</mo>
  <mn>8</mn>
  <mtext>MiB</mtext>的内容。

以下是一些方法的简介，代码需要参照源码：
* 通过 initialize 方法可以初始化一个 DiskInode 为一个文件或目录 
需要注意的是， indirect1/2 均被初始化为 0 。因为最开始文件内容的大小为 0 字节，并不会用到一级/二级索引。为了节约空间，内核会按需分配一级/二级索引块。此外，直接索引 direct 也被清零。  
* is_file 和 is_dir 两个方法可以用来确认 DiskInode 的类型为文件还是目录 
* get_block_id 方法体现了 DiskInode 最重要的数据块索引功能，它可以从索引中查到它自身用于保存文件内容的第 block_id 个数据块的块编号，这样后续才能对这个数据块进行访问：
* 在对文件/目录初始化之后，它的 size 均为 0 ，此时并不会索引到任何数据块。它需要通过 increase_size 方法逐步扩充容量。在扩充的时候，自然需要一些新的数据块来作为索引块或是保存内容的数据块。我们需要先编写一些辅助方法来确定在容量扩充的时候额外需要多少块  
* clear_size 方法清空文件的内容并回收所有数据和索引块。
* read_at方法将文件内容从 offset 字节开始的部分读到内存中的缓冲区 buf 中，并返回实际读到的字节数。如果文件剩下的内容还足够多，那么缓冲区会被填满；否则文件剩下的全部内容都会被读到缓冲区中。
* write_at 的实现思路基本上和 read_at 完全相同。但不同的是 write_at 不会出现失败的情况，传入的整个缓冲区的数据都必定会被写入到文件中。当从 offset 开始的区间超出了文件范围的时候，就需要调用者在调用 write_at 之前提前调用 increase_size ，将文件大小扩充到区间的右端，证写入的完整性。

## 数据块与目录项
作为一个文件而言，它的内容在文件系统看来没有任何既定的格式，都只是一个字节序列。因此每个保存内容的数据块都只是一个字节数组：
```rust
// easy-fs/src/layout.rs

type DataBlock = [u8; BLOCK_SZ];
```
然而，目录的内容却需要遵从一种特殊的格式。在我们的实现中，它可以看成一个目录项的序列，每个目录项都是一个二元组，二元组的首个元素是目录下面的一个文件（或子目录）的文件名（或目录名），另一个元素则是文件（或子目录）所在的索引节点编号。目录项相当于目录树结构上的子树节点，我们需要通过它来一级一级的找到实际要访问的文件或目录。我们可以通过 empty 和 new 分别生成一个空的目录项或是一个合法的目录项。目录项 DirEntry 的定义如下：
```rust
// easy-fs/src/layout.rs

const NAME_LENGTH_LIMIT: usize = 27;

#[repr(C)]
pub struct DirEntry {
    name: [u8; NAME_LENGTH_LIMIT + 1],
    inode_number: u32,
}

pub const DIRENT_SZ: usize = 32;
```
此外，通过 name 和 inode_number 方法可以取出目录项中的内容。

## 磁盘块管理器
实现 easy-fs 的整体磁盘布局，将各段区域及上面的磁盘数据结构结构整合起来就是简易文件系统 EasyFileSystem 的职责。它知道每个布局区域所在的位置，即可以从 inode位图 或数据块位图上分配的 bit 编号，来算出各个存储inode和数据块的磁盘块在磁盘上的实际位置。磁盘块的分配和回收也需要经过它才能完成，因此某种意义上讲它还可以看成一个磁盘块管理器。

注意从这一层开始，所有的数据结构就都放在内存上了。  
```rust
// easy-fs/src/efs.rs

pub struct EasyFileSystem {
    pub block_device: Arc<dyn BlockDevice>,
    pub inode_bitmap: Bitmap,
    pub data_bitmap: Bitmap,
    inode_area_start_block: u32,
    data_area_start_block: u32,
}
```
* 通过 create 方法可以在块设备上创建并初始化一个 easy-fs 文件系统。
* 通过 open 方法可以从一个已写入了 easy-fs 镜像的块设备上打开我们的 easy-fs 。
* get_disk_inode_pos和get_data_block_id用于算出各个存储inode和数据块的磁盘块在磁盘上的实际位置。
* inode 和数据块的分配/回收也由 EasyFileSystem 负责

## 索引节点
索引节点 (Inode, Index Node) 是文件系统中的一种重要数据结构。逻辑目录树结构中的每个文件和目录都对应一个 inode ，我们前面提到的文件系统实现中，文件/目录的底层编号实际上就是指 inode 编号。在 inode 中不仅包含了我们通过 stat 工具能够看到的文件/目录的元数据（大小/访问权限/类型等信息），还包含实际保存对应文件/目录数据的数据块（位于最后的数据块区域中）的索引信息，从而能够找到文件/目录的数据被保存在磁盘的哪些块中。从索引方式上看，同时支持直接索引和间接索引。  
Inode 和 DiskInode 的区别从它们的名字中就可以看出： DiskInode 放在磁盘块中比较固定的位置，而 Inode 是放在内存中的记录文件索引节点信息的数据结构。 
```rust 
// easy-fs/src/vfs.rs

pub struct Inode {
    block_id: usize,
    block_offset: usize,
    fs: Arc<Mutex<EasyFileSystem>>,
    block_device: Arc<dyn BlockDevice>,
}
```
block_id 和 block_offset 记录该 Inode 对应的 DiskInode 保存在磁盘上的具体位置方便我们后续对它进行访问。 fs 是指向 EasyFileSystem 的一个指针，因为对 Inode 的种种操作实际上都是要通过底层的文件系统来完成。  
>所有暴露给文件系统的使用者的文件系统操作（还包括接下来将要介绍的几种），全程均需持有 EasyFileSystem 的互斥锁。这能够保证在多核情况下，同时最多只能有一个核在进行文件系统相关操作。这样也许会带来一些不必要的性能损失，但我们目前暂时先这样做。如果我们在这里加锁的话，其实就能够保证块缓存的互斥访问了。

### 获取根目录的inode
文件系统的使用者在通过 EasyFileSystem::open 从装载了 easy-fs 镜像的块设备上打开 easy-fs 之后，要做的第一件事情就是获取根目录的 Inode 。使用root_inode方法来获取。
### 文件索引
EasyFileSystem 是一个扁平化的文件系统，即在目录树上仅有一个目录——那就是作为根节点的根目录。所有的文件都在根目录下面。于是，我们不必实现目录索引。文件索引的查找比较简单，仅需在根目录的目录项中根据文件名找到文件的 inode 编号即可。由于没有子目录的存在，这个过程只会进行一次。  
find 方法只会被根目录 Inode 调用，文件系统中其他文件的 Inode 不会调用这个方法。它首先调用 find_inode_id 方法，尝试从根目录的 DiskInode 上找到要索引的文件名对应的 inode 编号。这就需要将根目录内容中的所有目录项都读到内存进行逐个比对。如果能够找到，则 find 方法会根据查到 inode 编号，对应生成一个 Inode 用于后续对文件的访问。

### 文件列举
ls 方法可以收集根目录下的所有文件的文件名并以向量的形式返回，这个方法只有根目录的 Inode 才会调用
### 文件创建
create 方法可以在根目录下创建一个文件，该方法只有根目录的 Inode 会调用：
* 检查文件是否已经在根目录下，如果找到的话返回 None 
* 为待创建文件分配一个新的 inode 并进行初始化
* 将待创建文件的目录项插入到根目录的内容中，使得之后可以索引到  

### 文件清空
在索引到文件的 Inode 之后，可以调用 clear 方法,这会将该文件占据的索引块和数据块回收。  

### 文件读写
从根目录索引到一个文件之后，可以对它进行读写，使用read_at/write_at方法这里会从 EasyFileSystem 中分配一些用于扩容的数据块并传给 DiskInode::increase_size     
注意：和 DiskInode 一样，这里的读写作用在字节序列的一段区间上。  

## 内核索引节点层
站在用户的角度看来，在一个进程中可以使用多种不同的标志来打开一个文件，这会影响到打开的这个文件可以用何种方式被访问。此外，在连续调用 sys_read/write 读写一个文件的时候，我们知道进程中也存在着一个文件读写的当前偏移量，它也随着文件读写的进行而被不断更新。这些用户视角中的文件系统抽象特征需要内核来实现，与进程有很大的关系，而 easy-fs 文件系统不必涉及这些与进程结合紧密的属性。因此，我们需要将 easy-fs 提供的 Inode 加上上述信息，进一步封装为 OS 中的索引节点 OSInode :  

```rust 
// os/src/fs/inode.rs

pub struct OSInode {
    readable: bool,
    writable: bool,
    inner: Mutex<OSInodeInner>,
}

pub struct OSInodeInner {
    offset: usize,
    inode: Arc<Inode>,
}

impl OSInode {
    pub fn new(
        readable: bool,
        writable: bool,
        inode: Arc<Inode>,
    ) -> Self {
        Self {
            readable,
            writable,
            inner: Mutex::new(OSInodeInner {
                offset: 0,
                inode,
            }),
        }
    }
}
```
OSInode 就表示进程中一个被打开的常规文件或目录。 readable/writable 分别表明该文件是否允许通过 sys_read/write 进行读写。至于在 sys_read/write 期间被维护偏移量 offset 和它在 easy-fs 中的 Inode 则加上一把互斥锁丢到 OSInodeInner 中。这在提供内部可变性的同时，也可以简单应对多个进程同时读写一个文件的情况。
## 文件描述符层
一个进程可以访问的多个文件，所以在操作系统中需要有一个管理进程访问的多个文件的结构，这就是 文件描述符表 (File Descriptor Table) ，其中的每个 文件描述符 (File Descriptor) 代表了一个特定读写属性的I/O资源。  
为简化操作系统设计实现，可以让每个进程都带有一个线性的 文件描述符表 ，记录该进程请求内核打开并读写的那些文件集合。而 文件描述符 (File Descriptor) 则是一个非负整数，表示文件描述符表中一个打开的 文件描述符 所处的位置（可理解为数组下标）。进程通过文件描述符，可以在自身的文件描述符表中找到对应的文件记录信息，从而也就找到了对应的文件，并对文件进行读写。当打开（ open ）或创建（ create ） 一个文件的时候，如果顺利，内核会返回给应用刚刚打开或创建的文件对应的文件描述符；而当应用想关闭（ close ）一个文件的时候，也需要向内核提供对应的文件描述符，以完成对应文件相关资源的回收操作。
### 文件描述符表
为了支持进程对文件的管理，我们需要在进程控制块中加入文件描述符表的相应字段：  

```rust 
// os/src/task/task.rs

pub struct TaskControlBlockInner {
    ...
    pub fd_table: Vec<Option<Arc<dyn File + Send + Sync>>>,
}
```
fd_table 的类型包含多层嵌套，我们从外到里分别说明：

* Vec 的动态长度特性使得我们无需设置一个固定的文件描述符数量上限，我们可以更加灵活的使用内存，而不必操心内存管理问题；

* Option 使得我们可以区分一个文件描述符当前是否空闲，当它是 None 的时候是空闲的，而 Some 则代表它已被占用；

* Arc 首先提供了共享引用能力。后面我们会提到，可能会有多个进程共享同一个文件对它进行读写。此外被它包裹的内容会被放到内核堆而不是栈上，于是它便不需要在编译期有着确定的大小；

* dyn 关键字表明 Arc 里面的类型实现了 File/Send/Sync 三个 Trait ，但是编译期无法知道它具体是哪个类型（可能是任何实现了 File Trait 的类型如 Stdin/Stdout ，故而它所占的空间大小自然也无法确定），需要等到运行时才能知道它的具体类型，对于一些抽象方法的调用也是在那个时候才能找到该类型实现的方法并跳转过去。

## 应用访问文件的内核机制实现
### 文件系统初始化
### 打开与关闭文件
### 基于文件来加载并执行应用
在有了文件系统支持之后，我们在 sys_exec 所需的应用的 ELF 文件格式的数据就不再需要通过应用加载器从内核的数据段获取，而是从文件系统中获取。
```rust 
// os/src/syscall/process.rs

pub fn sys_exec(path: *const u8, mut args: *const usize) -> isize {
    let token = current_user_token();
    let path = translated_str(token, path);
    let mut args_vec: Vec<String> = Vec::new();
    loop {
        let arg_str_ptr = *translated_ref(token, args);
        if arg_str_ptr == 0 {
            break;
        }
        args_vec.push(translated_str(token, arg_str_ptr as *const u8));
        unsafe { args = args.add(1); }
    }
    if let Some(app_inode) = open_file(path.as_str(), OpenFlags::RDONLY) {
        let all_data = app_inode.read_all();
        let task = current_task().unwrap();
        let argc = args_vec.len();
        task.exec(all_data.as_slice(), args_vec);
        // return argc because cx.x[10] will be covered with it later
        argc as isize
    } else {
        -1
    }
}
```
同样的，我们在内核中创建初始进程 initproc 也需要替换为基于文件系统的实现。
### 读写文件
基于文件抽象接口和文件描述符表，我们可以按照无结构的字节流在处理基本的文件读写，这样可以让文件读写系统调用 sys_read/write 变得更加具有普适性，为后续支持把管道等抽象为文件打下了基础

## 编程练习
由于从本章开始需要向上一章兼容，所以上一章实现的mmap,munmap,spawn都要能在本章继续运行。  
mmap和munmap比较简单，直接从上一章复制过来就可以，但是spawn的实现需要进行改动。  
```rust
//os/src/syscall/process.rs

/// YOUR JOB: Implement spawn.
/// HINT: fork + exec =/= spawn
pub fn sys_spawn(path: *const u8) -> isize {
    trace!(
        "kernel:pid[{}] sys_spawn",
        current_task().unwrap().pid.0
    );
    let token=current_user_token();
    let path=translated_str(token, path);
    if let Some(app_inode) = open_file(path.as_str(), OpenFlags::RDONLY) {
        let data = app_inode.read_all();
        let current_task = current_task().unwrap();
        let task = current_task.spawn(data.as_slice());
        let pid = task.getpid() as isize;
        add_task(task);
        pid
    }
    else {
        -1
    }
}
```
在上一章中：
```rust
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
这是由于本章开始引入了文件系统，上一章使用的loader.rs模块不再使用，get_app_data_by_name函数也不再存在，需要使用inode对文件进行访问。

### 硬链接
硬链接要求两个不同的目录项指向同一个文件，在我们的文件系统中也就是两个不同名称目录项指向同一个磁盘块。

本节要求实现三个系统调用 sys_linkat、sys_unlinkat、sys_stat 。

**linkat：**

* syscall ID: 37
* 功能：创建一个文件的一个硬链接， linkat标准接口 。
* Rust 接口： fn linkat(olddirfd: i32, oldpath: *const u8, newdirfd: i32, newpath: *const u8, flags: u32) -> i32

* **参数**：
    * olddirfd，newdirfd: 仅为了兼容性考虑，本次实验中始终为 AT_FDCWD (-100)，可以忽略。

    * flags: 仅为了兼容性考虑，本次实验中始终为 0，可以忽略。

    * oldpath：原有文件路径

    * newpath: 新的链接文件路径。

* **说明**：
    * 为了方便，不考虑新文件路径已经存在的情况（属于未定义行为），除非链接同名文件。

    * 返回值：如果出现了错误则返回 -1，否则返回 0。

* **可能的错误**:  链接同名文件。

**unlinkat:**

* syscall ID: 35

* 功能：取消一个文件路径到文件的链接, unlinkat标准接口 。

* Rust 接口： fn unlinkat(dirfd: i32, path: *const u8, flags: u32) -> i32

* **参数：**
    * dirfd: 仅为了兼容性考虑，本次实验中始终为 AT_FDCWD (-100)，可以忽略。

    * flags: 仅为了兼容性考虑，本次实验中始终为 0，可以忽略。

    * path：文件路径。

* **说明：**
    * 为了方便，不考虑使用 unlink 彻底删除文件的情况。

* 返回值：如果出现了错误则返回 -1，否则返回 0。

* **可能的错误：**
    文件不存在。

```rust
//easy-fs/src/vfs.rs

    ///创建一个文件的一个硬链接
    pub fn create_link(&self, old_name:&str,new_name:&str)->isize{
        let mut fs=self.fs.lock();
        self.modify_disk_inode(|root_inode| {
            if let Some(old_inode_id)=self.find_inode_id(old_name, root_inode){
                let file_count = (root_inode.size as usize) / DIRENT_SZ;
                let new_size = (file_count + 1) * DIRENT_SZ;
                // increase size
                self.increase_size(new_size as u32, root_inode, &mut fs);
                // write dirent
                let dirent = DirEntry::new(new_name, old_inode_id);
                root_inode.write_at(
                    file_count * DIRENT_SZ,
                    dirent.as_bytes(),
                    &self.block_device,
                );
                0
            }
            else{
                -1
            }
        })
    }
    ///取消一个文件路径到文件的链接
    pub fn delete_link(&self,name:&str)->isize{
        let mut _fs=self.fs.lock();
        self.modify_disk_inode(|root_inode| {
            assert!(root_inode.is_dir());
            let file_count = (root_inode.size as usize) / DIRENT_SZ;
            let mut dirent = DirEntry::empty();
            for i in 0..file_count{
                assert_eq!(
                    root_inode.read_at(DIRENT_SZ * i, dirent.as_bytes_mut(), &self.block_device,),
                    DIRENT_SZ,
                );
                if dirent.name()==name{
                    let temp=DirEntry::empty();
                    root_inode.write_at(DIRENT_SZ*i, temp.as_bytes(),&self.block_device);
                    return 0;
                }
            }
            -1
        })
    }

//os/src/fs/inode.rs
///  创建硬连接
pub fn linkat(old_name: &str, new_name: &str) -> isize {
    ROOT_INODE.create_link(old_name, new_name)
}

///  删除硬连接
pub fn unlinkat(name: &str) -> isize {
    ROOT_INODE.delete_link(name)
}
```
```rust
//os/src/syscall/fs.rs
/// YOUR JOB: Implement linkat.
pub fn sys_linkat(old_name: *const u8, new_name: *const u8) -> isize {
    trace!(
        "kernel:pid[{}] sys_linkat",
        current_task().unwrap().pid.0
    );
    let token = current_user_token();
    let old_path = translated_str(token, old_name);
    let new_path = translated_str(token, new_name);
    if old_path.is_empty() || old_path == new_path {
        return -1;
    }
    linkat(old_path.as_str(), new_path.as_str())
}

/// YOUR JOB: Implement unlinkat.
pub fn sys_unlinkat(name: *const u8) -> isize {
    trace!(
        "kernel:pid[{}] sys_unlinkat",
        current_task().unwrap().pid.0
    );
    let token = current_user_token();
    let path=translated_str(token, name);
    if path.is_empty(){
        return -1;
    }
    unlinkat(path.as_str())
}
```

**fstat:**

* syscall ID: 80

* 功能：获取文件状态。
* Rust 接口： fn fstat(fd: i32, st: *mut Stat) -> i32

* 参数：
    * fd: 文件描述符

    * st: 文件状态结构体    


    
```rust
#[repr(C)]
#[derive(Debug)]
pub struct Stat {
    /// 文件所在磁盘驱动器号，该实验中写死为 0 即可
    pub dev: u64,
    /// inode 文件所在 inode 编号
    pub ino: u64,
    /// 文件类型
    pub mode: StatMode,
    /// 硬链接数量，初始为1
    pub nlink: u32,
    /// 无需考虑，为了兼容性设计
    pad: [u64; 7],
}

/// StatMode 定义：
bitflags! {
    pub struct StatMode: u32 {
        const NULL  = 0;
        /// directory
        const DIR   = 0o040000;
        /// ordinary regular file
        const FILE  = 0o100000;
    }
}
```

这个还不是很理解。 Todo