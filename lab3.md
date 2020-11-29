## Lab3学习笔记

### 修改内核

​	将内核代码放在虚拟地址空间中以 0xffffffff80200000 开头的一段高地址空间中。并将每个字段设置为4KB对齐（一个4KB的虚拟页不会包含两个段）。

##### boot_page_table

```assembly
    # 计算 boot_page_table 的物理页号
    lui t0, %hi(boot_page_table)	#t0 => 0xffffffff80200
    li t1, 0xffffffff00000000
    sub t0, t0, t1					#t0 => 0x80200
    srli t0, t0, 12					#t0 => 0x80200000
    # 8 << 60 是 satp 中使用 Sv39 模式的记号
    li t1, (8 << 60)
    or t0, t0, t1
    # 写入 satp 并更新 TLB
    csrw satp, t0
    sfence.vma
```

​	boot_page_table中第二项是低地址到低地址的映射，第510项是高地址到低地址的映射。
### 实现页表

![img](https://pic3.zhimg.com/80/v2-0bb54bd378ba63fea38b862c88fbb596_720w.png)	实现Sv39页表，需要三级页表，通过address.rs中level函数，获得三级VPN；

![img](https://rcore-os.github.io/rCore-Tutorial-deploy/docs/lab-3/pics/sv39_pte.jpg)

```rust
pub struct Flags: u8 {
        /// 有效位
        const VALID =       1 << 0;
        /// 可读位
        const READABLE =    1 << 1;
        /// 可写位
        const WRITABLE =    1 << 2;
        /// 可执行位
        const EXECUTABLE =  1 << 3;
        /// 用户位
        const USER =        1 << 4;
        /// 全局位，我们不会使用
        const GLOBAL =      1 << 5;
        /// 已使用位，用于替换算法
        const ACCESSED =    1 << 6;
        /// 已修改位，用于替换算法
        const DIRTY =       1 << 7;
    }
```

​	设置页表项中的标志位

```rust
pub struct PageTable {
    pub entries: [PageTableEntry; PAGE_SIZE / 8],
}
```

​	页表结构体，将512个页表项封装为页表。

```rust
pub struct PageTableTracker(pub FrameTracker);
```

 	将FrameTrack记录的物理页当成PageTable进行操作。

### 函数调用过程

​	    通过main函数调用memory_set.rs中的new_kernel函数，此函数实现内核重映射。首先设置.text、.rodata、.data、.bss以及剩余内存空间（内核结束地址到内存结束地址）的映射关系结构体Segment（其成员包括：映射类型map_type，所映射的虚拟地址range，权限标志flags），存储到Vec中。

​	    字段设置完成后，调用mapping.rs中的new函数，进而调用PageTableTracker的new函数返回创建的空的page_table。并获取page_table的物理页号，返回一个Mapping结构体（此结构体保存线程的内存映射关系，成员包括保存使用到的页表的page_tables，根页表物理页号的root_ppn，所有分配的物理页面映射信息的mapped_pairs），从而创建一个有根节点的映射。

​	    循环将每个字段的semgent在页表中进行映射，因为每个segment的映射类型都是线性映射，初始化数据均为空，所以仅执行map函数中线性映射分支，调用map_one函数实现每个字段虚拟页号和物理页号的映射关系。在map_one函数中，根据虚拟页号查找页表项，如果页表项不存在，就通过page_table_entry.rs中的new函数将物理页号和标志写入一个页表项。因为segment初始化数据为None，所以不需要拷贝数据。

​	    最后new_kernel函数返回一个MemorySet结构体（该结构体包含mapping成员，维护页表和映射关系；segments成员是指字段）

​	    通过activate函数将映射加载到stap寄存器，并刷新TLB
