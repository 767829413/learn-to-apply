# 看看书本上的linux系列-6

##  **内存管理**

1. 虚拟内存地址到物理内存地址的映射
    * 分段
        * 虚拟地址 = 段选择子(段寄存器) + 段内偏移量
        * 段选择子 = 段号(段表索引) + 标识位
        * 段表 = 物理基地址 + 段界限(偏移量范围) + 特权等级
        * Linux 分段实现
            * 段表称为段描述符表, 放在全局标识符表中
            * Linux 将段基地址都初始化为 0, 不用于地址映射
            * Linux 分段功能主要用于权限检查
    * 分页
        * 页表 = 虚拟页号 + 物理页号; 用于定位页
        * 虚拟地址 = 虚拟页号 + 页内偏移
        * 若采用单页表, 32位系统中一个页表将有 1M 页表项, 占用 4MB(每项 4B)
        * Linux 通过分页实现映射
            * 物理内存被换分为大小固定(4KB)的页, 物理页可在内存与硬盘间换出/换入
        * Linux 32位系统采用两级页表: 页表目录(1K项, 10bit) + 页表(1K项, 10bit)(页大小(4KB, 12bit))
        * 映射 4GB 内存理论需要 1K 个页表目录项 + 1K*1K=1M 页表项, 将占用 4KB+4MB 空间
        * 完整的页表目录可以满足所有地址的查询, 因此页表只需在对应地址有内存分配时才生成;
        * 64 为系统采用 4 级页表,基本实现原理和32位类似
