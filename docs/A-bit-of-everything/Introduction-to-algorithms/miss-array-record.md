# 啊哈?数组

## **简单聊聊数据**

1. 数组简单易用,在实现上使用的是连续的内存空间,可以借助 CPU 的缓存机制,预读数组中的数据,所以访问效率更高.
2. 链表在内存中并不是连续存储,所以对 CPU 缓存不友好,没办法有效预读.
3. 针对CPU缓存机制指的是什么？为什么用数组效率高？
    在学习操作系统和计算机组成原理的时候了解到
    CPU在从内存读取数据的时候,会先把读取到的数据加载到CPU的缓存中.
    CPU每次从内存读取数据并不是只读取那个特定要访问的地址,而是读取一个数据块(这个大小不太确定..)并保存到CPU缓存中,然后下次访问内存数据的时候就会先从CPU缓存开始查找,如果找到就不需要再从内存中取.这样就实现了比内存访问速度更快的机制,也就是CPU缓存存在的意义:为了弥补内存访问速度过慢与CPU执行速度快之间的差异而引入.
    对于数组来说,存储空间是连续的,所以在加载某个下标的时候可以把以后的几个下标元素也加载到CPU缓存,这样执行速度会快于存储空间不连续的链表存储.
