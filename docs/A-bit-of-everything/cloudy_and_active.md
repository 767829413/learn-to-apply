# 公司多云多活架构小结

## 设计思路

目前在阿里云和移动云都有服务对外提供,考虑到目前云服务商可能宕机或者故障,想利用两个云的服务互为备份,数据层通过MYSQL之间相互同步进行互相备份,保证上层的业务状态通过数据库的数据同步切换云后不丢失,但是对象存储的数据没有做到两边云的同步,目前认为对象存储业务稳定不挂

## 业务适配问题

### 数据库同步业务导致的主键id更改

使用了阿里云的DTS进行数据双向的同步,我们所有类似redis和ES这种中间件的数据也是通过MYSQL这层做同步中转的,在业务的库表中,使用了自增主键,阿里云的DTS业务在同步过程中如果主键冲突插入或者唯一键冲突插入会导致同步失败,为了保证同步的稳定性,针对自增主键需要去掉自增属性并且保证多云下唯一

方案无非是使用 uuid objectid 雪花id,最后定下为业界通用方案雪花id,具体为啥用这个就不解释了(凡人复杂的博弈),主要是使用雪花id后遇到以下问题:

1. 雪花id分配算法中机器码的分配,原本打算使用 hash(服务器机器码+podname+containername),有大佬说可能会出现冲突(hash碰撞,概率再小你也不能保证不发生吧,嘿嘿嘿),
    * 解决方案: 通过redis或者全局的mysql表添加记录使用唯一键

2. 前端的number类型超最大长度,导致数据失真
    * 解决方案: 前端用bignumber或者string类型(这个服务端要兼容string的解析)

3. 已有数据的id修正
    * 解决方案: 脚本刷呗,不过只有两个云的环境,只要刷一边就行了,简单操作就是一个云找最大的id分配值,另一个云所有id + 这个值向上取整的数值

### 适配多云的对象存储

多云切换后,需要保证原有的服务的对象存储业务都可用(每个云都是各自用各自的),在启用多云多活的备份方案前,移动云全部使用的oss,不是移动云自己的对象存储,所以业务代码中充斥着大量的固定配置项目,需要全部重构(那啥山就不多说,开挖),基于每个使用对象存储的记录都记录对应的对象存储提供商和用途

### 切云流量的分配

这里没啥说的SLB这层做域名ip绑定,然后每个云使用不同的域名,只要更换对应云的网关ip就ok,不过细节肯定很复杂了,都是大佬们开动就行