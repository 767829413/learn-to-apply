# 排序的时候别虚......临时笔记

## **简单聊聊排序**

很多数据结构和算法课程,在讲排序的时候,都是用整数来举例,但在真正软件开发中,我们要排序的往往不是单纯的整数,而是一组对象,我们需要按照对象的某个 key 来排序.

比如说,我们现在要给电商交易系统中的“订单”排序.订单有两个属性,一个是下单时间,另一个是订单金额.如果我们现在有 10 万条订单数据,我们希望按照金额从小到大对订单数据排序.对于金额相同的订单,我们希望按照下单时间从早到晚有序.对于这样一个排序需求,我们怎么来做呢？

最先想到的方法是：我们先按照金额对订单数据进行排序,然后,再遍历排序之后的订单数据,对于每个金额相同的小区间再按照下单时间排序.这种排序思路理解起来不难,但是实现起来会很复杂.

借助稳定排序算法,这个问题可以非常简洁地解决.解决思路是这样的：我们先按照下单时间给订单排序,注意是按照下单时间,不是金额.排序完成之后,我们用稳定排序算法,按照订单金额重新排序.两遍排序之后,我们得到的订单数据就是按照金额从小到大排序,金额相同的订单按照下单时间从早到晚排序的.

为什么呢？稳定排序算法可以保持金额相同的两个对象,在排序之后的前后顺序不变.第一次排序之后,所有的订单按照下单时间从早到晚有序了.在第二次排序中,我们用的是稳定的排序算法,所以经过第二次排序之后,相同金额的订单仍然保持下单时间从早到晚有序.