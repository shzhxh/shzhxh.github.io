---
title: fat32文件系统的磁盘布局
date: 2023-09-05 10:47:00
categories: OS
---

FAT文件系统把存储介质看成是由簇(cluster)组成的数组。

注：**簇**是FAT和NTFS文件系统里的最小单位。**块**是EXT2和EXT3文件系统里的最小单位。**扇区**是磁盘里的最小单位。

<!-- more -->

#### fat32的整体结构

FAT文件系统=保留扇区 + 文件分配表 + 目录和数据区

#### 保留扇区

##### 引导记录

引导记录大小是1个扇区，位置是所在分区的0号扇区。

引导记录=BIOS参数块(0x00-0x23) + 扩展引导记录(0x24-0x1ff)

```rust
pub(crate) struct BootSector {
    bootjmp: [u8; 3],		// offset:0x00，跳转指令，用于跳过数据区，即：BiosParameterBlock
    oem_name: [u8; 8],		// offset:0x03
    pub(crate) bpb: BiosParameterBlock,	// 数据部分
    boot_code: [u8; 448],	// offset:0x5a
    boot_sig: [u8; 2],		// offset:0x1fe, 启动分区的签名"0xAA55"
}
pub(crate) struct BiosParameterBlock {
    pub(crate) bytes_per_sector: u16,	// offset:0x0b，扇区的大小
    pub(crate) sectors_per_cluster: u8,	// offset:0x0d，簇的大小
    pub(crate) reserved_sectors: u16,	// offset:0x0e，保留的扇区个数(保留扇区包含了boot record sectors)
    pub(crate) fats: u8,	// offset:0x10, 文件分配表的个数，一般是2
    pub(crate) root_entries: u16,		// offset:0x11, 根目录包含多少个条目
    pub(crate) total_sectors_16: u16,	// offset:0x13，当前分区包含的扇区个数。为0表示超过了65535,则扇区个数记录在0x20处
    pub(crate) media: u8,				// offset:0x15，介质的类型
    pub(crate) sectors_per_fat_16: u16,	// offset:0x16，每个FAT占用的扇区数。仅用于FAT12/FAT16
    pub(crate) sectors_per_track: u16,	// offset:0x18，每个磁道包含的扇区数。
    pub(crate) heads: u16,				// offset:0x1a，磁头的数量
    pub(crate) hidden_sectors: u32,		// offset:0x1c，隐藏扇区的数量
    pub(crate) total_sectors_32: u32,	// offset:0x20，当前分区包含的扇区个数。当0x13记录不下，用当前字段来记录

    // Extended BIOS Parameter Block
    pub(crate) sectors_per_fat_32: u32,	// offset:0x24，分区表的大小
    pub(crate) extended_flags: u16,		// offset:0x28，标志(flags)
    pub(crate) fs_version: u16,			// offset:0x2a，FAT版本号
    pub(crate) root_dir_first_cluster: u32,	// offset:0x2c, 根目录的簇号，一般设置为2
    pub(crate) fs_info_sector: u16,		// offset:0x30, FSInfo的扇区号
    pub(crate) backup_boot_sector: u16,	// offset:0x32，boot sector备份的扇区号
    pub(crate) reserved_0: [u8; 12],	// offset:0x34
    pub(crate) drive_num: u8,			// offset:0x40，驱动器编号。等于BIOS执行完后DL寄存器的内容。比如，0x80代表硬盘。
    pub(crate) reserved_1: u8,			// offset:0x41
    pub(crate) ext_sig: u8,				// offset:0x42, 签名。只能是0x28或0x29
    pub(crate) volume_id: u32,			// offset:0x43，卷ID的序列号。可忽略。
    pub(crate) volume_label: [u8; 11],	// offset:0x47
    pub(crate) fs_type_label: [u8; 8],	// offset:0x52, 总是"FAT32"
}


```

##### FSInfo扇区

```rust
// FSInfo结构体占用512字节，扇区号在扩展引导记录里指定。
// 注：offset:0x4~0x1e3, 0x1f0~0x1fb是保留空间
struct FsInfoSector {
    free_cluster_count: Option<u32>,	// offset:0x1e8，卷上空闲簇的数量。如为0xFFFF_FFFF,则空闲簇数量未知，必须计算求得。
    next_free_cluster: Option<u32>,		// offset:0x1ec，应该从哪个簇号开始查找空闲簇。
    dirty: bool,
}
impl FsInfoSector {
    const LEAD_SIG: u32 = 0x4161_5252;	// 先导签名的值，offset:0x0
    const STRUC_SIG: u32 = 0x6141_7272;	// 中间签名的值，offset:0x1e4
    const TRAIL_SIG: u32 = 0xAA55_0000;	// 结尾签名的值，offset:0x1fc
    
    ...
}
```



#### 文件分配表

文件分配表里的一个表项就是一个32位的簇地址，表项的数量就是当前分区里的簇数量。

FAT32使用32位地址来给簇编号，但高4位被保留了，只有低28位是有效的。

- 0号表项是被保留的。其值必须是`0xf8ff_ff0f`。
- 1号表项是被保留的，留做将来使用。其值必须是`0xffff_ffff`或`0xffff_ff0f`。
- 目录和数据区是从**2**开始编号的。

文件分配表里元素值的含义：

- 大于等于`0x0fff_fff8`，代表没有多余的簇了，即整个文件已读取完成。
- 等于`0x0fff_fff7`，代表这个簇是坏的。
- 等于`0`，代表这个簇是空闲的。
- 等于`1`，没有意义。
- 其它值，代表了当前文件下一个簇的簇号。



#### 目录和数据区

##### 根目录

根目录在目录和数据区的第0号簇上，记录了整个磁盘的结构。是由目录条目组成的数组，数组元素为32字节大小。

##### 目录和文件的条目

```rust
// 标准8.3格式。大小32字节。
pub(crate) struct DirFileEntryData {
    name: [u8; SFN_SIZE],	// offset:0, SFN_SIZE值为11。前8名字，后3后缀。
    attrs: FileAttributes,	// offset:11, u8类型。文件属性，可以决定当前条目是目录还是文件。
    reserved_0: u8,			// offset:12
    create_time_0: u8,		// offset:13，创建时间，以0.1秒为单位。
    create_time_1: u16,		// offset:14，创建时间，前5小时，中6分钟，后5秒。注：秒要乘2.
    create_date: u16,		// offset:16，创建日期。前7年，中4月，后5日。
    access_date: u16,		// offset:18，最后访问日期。前7年，中4月，后5日。
    first_cluster_hi: u16,	// offset:20, 当前文件第一个簇的簇号的高16位。
    modify_time: u16,		// offset:22，最后修改时间。前5小时，中6分钟，后5秒。注：秒要乘2.
    modify_date: u16,		// offset:24，最后修改日期。前7年，中4月，后5日。
    first_cluster_lo: u16,	// offset:26, 当前文件第一个簇的簇号的低16位。
    size: u32,				// offset:28, 当前文件占用了多少字节。
}
// 长文件名条目。大小32字节。位置是在标准8.3格式的前面。
pub(crate) struct DirLfnEntryData {
    order: u8,			// offset:0，当前条目在长文件名条目序列里的次序。这样就可以知道当前条目的字符在文件名中的位置。
    name_0: [u16; 5],	// offset:1，前5个双字节字符。
    attrs: FileAttributes,	// offset:11，文件属性，固定为0x0F。
    entry_type: u8,		// offset:12。条目类型。0代表名字条目。
    checksum: u8,		// offset:13。创建文件时对短文件名生成的检验和。
    name_1: [u16; 6],	// offset:14。接下来6个双字节字符。
    reserved_0: u16,	// osffset:26。保留字段，总是设置为0.
    name_2: [u16; 2],	// offset:28。最后2个双字节字符。
}

bitflags! {
    /// A FAT file attributes.
    #[derive(Default)]
    pub struct FileAttributes: u8 {
        const READ_ONLY  = 0x01;
        const HIDDEN     = 0x02;
        const SYSTEM     = 0x04;
        const VOLUME_ID  = 0x08;
        const DIRECTORY  = 0x10;	// 目录
        const ARCHIVE    = 0x20;	// 普通文件
        // LFN：值为0x0F，表示当前条目是长文件名条目
        const LFN        = Self::READ_ONLY.bits | Self::HIDDEN.bits
                         | Self::SYSTEM.bits | Self::VOLUME_ID.bits;
    }
}
```

#### 参考资料

- [osdev.org/fat](https://wiki.osdev.org/FAT)
- [crates.io/fatfs](https://crates.io/crates/fatfs)
- 《操作系统的设计与实现 - 5.3.2实现文件》
