---
title: ext2文件系统的磁盘布局
date: 2023-09-02 11:00:20
categories: OS
---

Ext2是Linux在90年代事实上使用的文件系统。它原生支持Unix的所有权机制，符号链接和硬链接。Ext3和Ext4是对Ext2的增强，它们的核心设计都是一样的。

<!-- more -->

#### 整体结构

- 从宏观上看，ext2文件系统由若干**块组**(Block Group)构成。

- 每个块组里块的数量是相同的，但最后一个块组内块的数量可能小于这个值。

- 块组＝超级块＋块组描述＋保留GDT块＋块位图＋索引结点位图＋索引结点表＋数据块区

  - 超级块 ：记录了整个文件系统的信息，在0号块组上。其它块组上也可能有超级块，但都是0号块组上超级块的备份。

    超级块上记录的重要信息有：总索引结点数，总块数，。。。

  - 块组描述：记录了所有块组的信息，在0号块组上。块组描述在超级块后面出现在其它块组内，也像超级块一样是0号块组上块组描述的备份。

    块组描述上记录了每个块组的：块组号，起始块号，超级块的块号(-1表示不存在)，块组描述的块号范围(-1表示不存在)，块位图的块号，索引结点位图的块号，索引结点表的块号

  - 块位图：记录当前块组内每个块是否空闲。

  - 索引结点位图：记录当前块组内每个索引结点是否分配。

  - 索引结点表：由索引结点组成的表。

  - 数据块：存放文件的内容。

- 块(Block)是ext2文件系统的最小单元，块大小相同，从0开始编号。

- 块的大小可以通过`sudo dumpe2fs <device> | grep "Block size:"`查到。这是在`/etc/mke2fs.conf`指定的，也可以在执行`mke2fs`使用`-b <block-size>`指定。

- 块的大小和数量在格式化后不能改变。

- 每个块最多只能保存1个文件的数据。

#### 超级块

超级块在当前分区的1024字节处，其长度是1024字节。

```rust
pub struct Superblock {
    pub inodes_count: u32,	// offset:0, 总索引结点数
    pub blocks_count: u32,	// offset:4, 总块数
    pub r_blocks_count: u32,	// offset:8, 超级用户保留的块数(见offset:80)
    pub free_blocks_count: u32,	// offset:0xc，未分配块的数量
    pub free_inodes_count: u32,	// offset:0x10, 未分配索引结点的数量
    pub first_data_block: u32,	// offset:0x14, 超级块所在块的块号
    pub log_block_size: u32,	// offset:0x18，(log2 (block size) - 10)
    pub log_frag_size: i32,	// offset:0x1c, (log2 (fragment size) - 10)
    pub blocks_per_group: u32,	// offset:0x20, 每个块组内块的数量
    pub frags_per_group: u32,	// offset:0x24，每个块组内fragment的数量
    pub inodes_per_group: u32,	// offset:0x28，每个块组内索引结点的数量
    pub mtime: u32,	// offset:0x2c，上次挂载的时间
    pub wtime: u32,	// offset:0x30，上次写的时间
    pub mnt_count: u16,	// offset:0x34，上次fsck以来挂载的次数
    pub max_mnt_count: i16,	// offset:0x36，fsck之前允许挂载的数量
    pub magic: u16,	// offset:0x38，Ext2签名 (0xef53)
    pub state: u16,	// offset:0x3a, 1:文件系统是干净的，2：文件系统出错了
    pub errors: u16,	// offset:0x3c, 检测到错误后做什么：1忽略，2重新挂载为只读，3内核恐慌
    pub rev_minor: u16,	// offset:0x3e, 副版本号
    pub lastcheck: u32,	// offset:0x40, 上次fsck的时间
    pub checkinterval: u32,	// offset:0x44, fsck的时间间隔
    pub creator_os: u32,	// offset:0x48，本文件系统的创建者：0Linux, 1GNU HURD, 2MASIX, 3FreeBSD, 4其它BSD4.4-Lite衍生操作系统
    pub rev_major: u32,	// offset: 0x4c, 主版本号
    pub block_uid: u16,	// offset:0x50, 可以使用保留块的用户id
    pub block_gid: u16,	// offset:0x52, 可以使用保留块的组id 

    /// 扩展的超级块，当主版本号大于等于1时如下字段存在
    pub first_inode: u32,	// offset:0x54, 第一个非保留的索引结点
    pub inode_size: u16,	// offset:0x58, 索引结点的大小，以字节为单位
    pub block_group: u16,	// offset:0x5a, 当前超级块所在的块组
    pub features_opt: FeaturesOptional,	// offset:0x5c, 可选属性。
    pub features_req: FeaturesRequired,	// offset:0x60, 要求的属性。
    pub features_ronly: FeaturesROnly,	// offset:0x64, 若不支持此属性，分区只能挂载为只读
    pub fs_id: [u8; 16],	// offset:0x68，文件系统ID
    pub volume_name: [u8; 16],	// offset:0x78，分区名
    pub last_mnt_path: [u8; 64],	// offset:0xc7，上次挂载路径
    pub compression: u32,	// offset:c8，使用了压缩算法
    pub prealloc_blocks_files: u8,	// offset:0xcc，为文件预先分配的块数
    pub prealloc_blocks_dirs: u8,	// offset:0xcd, 为目录预先分配的块数
    #[doc(hidden)]
    _unused: [u8; 2],
    pub journal_id: [u8; 16],	// offset:0xd0，日志ID
    pub journal_inode: u32,	// offset:0xe0，日志索引结点
    pub journal_dev: u32,	// offset:0xe4，日志设备
    pub journal_orphan_head: u32,	// offset:0xe8，孤立索引结点列表的head
    #[doc(hidden)]
    _reserved: [u8; 788],	// offset:0xec
}

bitflags! {
    /// 可选属性，可用于提升性能。见超级块的0x5c偏移。
    pub struct FeaturesOptional: u32 {
        const PREALLOCATE = 0x0001;	// 创建目录时为它预先分配一定数量的块。参考超级块的0xcd偏移
        const AFS = 0x0002;		// AFS服务索引结点存在
        const JOURNAL = 0x0004;	// 文件系统有日志(Ext3)
        const EXTENDED_INODE = 0x0008;	// 索引结点有扩展属性
        const SELF_RESIZE = 0x0010;	// 文件系统可以占据更大的分区
        const HASH_INDEX = 0x0020;	// 目录使用哈希索引
    }
}
bitflags! {
    /// 要求的属性。见超级块的0x60偏移。
    pub struct FeaturesRequired: u32 {
        const REQ_COMPRESSION = 0x0001;	// 使用压缩
        const REQ_DIRECTORY_TYPE = 0x0002;	// 目录条目包含类型字段
        const REQ_REPLAY_JOURNAL = 0x0004;	// 文件系统需要replay它的日志
        const REQ_JOURNAL_DEVICE = 0x0008;	// 文件系统使用了一个日志设备
    }
}
bitflags! {
    /// 若不支持此属性，分区只能挂载为只读。见超级块的0x64偏移。
    pub struct FeaturesROnly: u32 {
        const RONLY_SPARSE = 0x0001;	// 稀疏超级块和块组描述符表
        const RONLY_FILE_SIZE_64 = 0x0002;	// 文件系统使用64位文件大小
        const RONLY_BTREE_DIRECTORY = 0x0004;	// 目录内容以二叉树的形式存储
    }
}
```

#### 块组描述

块组描述符表紧跟在超级块之后。

```rust
pub struct BlockGroupDescriptor {
    pub block_usage_addr: u32,	// 块位图的块号
    pub inode_usage_addr: u32,	// 索引结点位图的块号
    pub inode_table_block: u32,	// 索引结点表的起始块号
    pub free_blocks_count: u16,	// 块组内未分配块的数量
    pub free_inodes_count: u16,	// 块组内未分配索引结点的数量
    pub dirs_count: u16,	// 块组内目录的数量
    #[doc(hidden)]
    _reserved: [u8; 14],
}
```

#### 索引结点

索引结点是从1开始编号的。而块是从0开始编号的。

从Ext2的版本1开始，第一个非保留的索引结点记录在超级块的`0x54`偏移处。在保留的索引结点里最重要的应该是用于根目录的2号索引结点。

从Ext2的版本1开始，索引结点的大小记录在超级块的`0x58`偏移处。查找索引结点的过程先是看它属于哪个块组，然后在该块组的索引结点表里对它进行索引。

##### Inode数据结构

```rust
pub struct Inode {
    pub type_perm: TypePerm,	// offset:0, 类型与权限
    pub uid: u16,	// offset:2, 用户ID
    pub size_low: u32,	// offset:4, 文件大小的低32位，单位字节
    pub atime: u32,	// offset:0x8，上次访问时间
    pub ctime: u32,	// offset:0xc，创建时间
    pub mtime: u32,	// offset:0x10，上次修改时间
    pub dtime: u32,	// offset:0x14，删除时间
    pub gid: u16,	// offset:0x18，组ID
    pub hard_links: u16,	// offset:0x1a，当前索引结点上的硬链接数
    pub sectors_count: u32,	// offset:0x1c，当前索引结点使用的扇区数
    pub flags: Flags,	// offset:0x20，标志
    pub _os_specific_1: [u8; 4],	// offset:0x24，操作系统特定值#1
    pub direct_pointer: [u32; 12],	// offset:0x28，直接块的指针数组，记录了12个直接块的地址
    pub indirect_pointer: u32,	// offset:0x58，单级间接块的指针，单级间接块里记录的是直接块的地址
    pub doubly_indirect: u32,	// offset:0x5c，二级间接块的指针，二级间接块里记录的是单级间接块的地址
    pub triply_indirect: u32,	// offset:0x60，三级间接块的指针，三级间接块里记录的是二级间接块的地址
    pub gen_number: u32,	// offset:0x64，生成号(主要用于NFS)
    pub ext_attribute_block: u32,	// offset:0x68，扩展属性块(ACL)
    pub size_high: u32,	// offset:0x6c，如果是文件则为文件大小的高32位，如果是目录则为目录的ACL
    pub frag_block_addr: u32,	// offset:0x70，fragment的块地址
    pub _os_specific_2: [u8; 12],	// offset:0x74，操作系统特定值#2
}

bitflags! {
    /// 见Inode的0偏移
    pub struct TypePerm: u16 {
        const FIFO = 0x1000;	// FIFO
        const CHAR_DEVICE = 0x2000;	// 字符设备
        const DIRECTORY = 0x4000;	// 目录
        const BLOCK_DEVICE = 0x6000;	// 块设备
        const FILE = 0x8000;	// 常规文件
        const SYMLINK = 0xA000;	// 符号链接
        const SOCKET = 0xC000;	// Unix网络插座
        const O_EXEC = 0x001;	// other可执行
        const O_WRITE = 0x002;	// other写
        const O_READ = 0x004;	// other读
        const G_EXEC = 0x008;	// group可执行
        const G_WRITE = 0x010;	// group写
        const G_READ = 0x020;	// group读
        const U_EXEC = 0x040;	// user可执行
        const U_WRITE = 0x080;	// user写
        const U_READ = 0x100;	// user读
        const STICKY = 0x200;	// 粘滞位
        const SET_GID = 0x400;	// 设置组ID
        const SET_UID = 0x800;	// 设置用户ID
    }
}
bitflags! {
    /// 见Inode的0x20偏移
    pub struct Flags: u32 {
        const SECURE_DEL = 0x00000001;	// 安全删除
        const KEEP_COPY = 0x00000002;	// 删除后保持数据的备份
        const COMPRESSION = 0x00000004;	// 文件压缩
        const SYNC_UPDATE = 0x00000008;	// 同步数据就是直接写硬盘
        const IMMUTABLE = 0x00000010;	// 文件内容不可改变
        const APPEND_ONLY = 0x00000020;	// 仅能追加
        const NODUMP = 0x00000040;	// 执行dump命令不包含此文件
        const DONT_ATIME = 0x00000080;	// 不更新上次访问时间
        const HASH_DIR = 0x00010000;	// 对目录使用哈希索引
        const AFS_DIR = 0x00020000;	// ASF目录
        const JOURNAL_DATA = 0x00040000;	// 日志文件数据
    }
}
```

##### 目录条目

目录属于一种索引结点，它的内容是**目录条目**组成的数组。

目录条目：

| 偏移 | 大小 | 字段描述                                                     |
| ---- | ---- | ------------------------------------------------------------ |
| 0    | 4    | 索引结点                                                     |
| 4    | 2    | 当前条目的大小                                               |
| 6    | 1    | 名字的长度的最低8个有效位                                    |
| 7    | 1    | 若超级块的`0x60`偏移指定了“目录条目包含文件类型字节”，则此字节代表类型指示符；否则代表名字的长度的最高8个有效位 |
| 8    | N    | 名字                                                         |

目录条目的类型指示符：

| 值   | 类型描述 |
| ---- | -------- |
| 0    | 未知类型 |
| 1    | 常规文件 |
| 2    | 目录     |
| 3    | 字符设备 |
| 4    | 块设备   |
| 5    | FIFO     |
| 6    | 网络插座 |
| 7    | 符号链接 |

##### 根目录

根目录在2号索引结点。

#### 参考资料

- [osdev.org/ext2](https://wiki.osdev.org/Ext2)
- [github.com/pi-pi3/ext2-rs](https://github.com/pi-pi3/ext2-rs)
- [Linux EXT2文件系统详解](https://zhuanlan.zhihu.com/p/438532666)
