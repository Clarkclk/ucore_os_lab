实验3
====

练习1
----
1.1 给未被映射的地址映射上物理页,简要说明你的设计实现过程。

根据地址和页目录表找到对应页表项，若未分配则新建一个，并调用alloc_page来填写表项。


1.2 请描述页目录项（Pag Director Entry）和页表（Page Table Entry）中组成部分对ucore实现页替换算法的潜在用处。

在lab2中已经回答。页表项中的PTE_P,PTE_W,PTE_U等标志位可以用来识别触发缺页异常的类型。而在实现页替换算法时，PTE_A,PTE_D等可以用来记录时候被访问过，是否被改写过，可以用于选择换出的页。

1.3 如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

CPU在内核中保存现场，将EFLAGS、CS、EIP等进行压栈处理，然后进行页访问异常的服务例程。

练习2
----

2.1 完成vmm.c中的do_pgfault函数，并且在实现FIFO算法的swap_fifo.c中完成map_swappable和swap_out_vistim函数。简要说明你的设计实现过程。

- 首先接练习1,如果页表项已存在，则该页已被换出到磁盘，将该页换入内存，并在内存管理器中添加该页信息，最后标注该页可换出；
- 而对于swap_fifo_c文件，map_swappable函数中把新添加的页添加到链表的末端即可；
- 对于swap_out_vistim函数，选择链表的第一个页为要换出的页，并删除。

2.2 如果要在ucore上实现"extended clock页替换算法"请给你的设计方案，现有的swap_manager框架是否足以支持在ucore中实现此算法？如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。并需要回答如下问题

- 需要被换出的页的特征是什么？
- 在ucore中如何判断具有这样特征的页？
- 何时进行换入和换出操作？

现有的框架足以支持在ucore中实现extended clock算法，因为页表项中有访问和脏位，实现较为复杂，需在swap_out_victim中直接替换链表尾部的页，改为从当前链表指针开始遍历，遍历过的页需按照一定规则将脏位或访问位位置0,直到找到脏位和访问位均为0的页。

- 被换出的页的特征：脏位和访问位均为0，且靠近表头。
- 利用页表项的访问位和脏位的值进行遍历查找
- 发生页访问异常的时候进行换入操作，换入时发现内存分配页面已满时进行换出操作。

