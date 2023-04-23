# 关于对 Kubernetes 原子调度单位Pod的思考

## **问题产生**

最近了解了一下关于[Kubernetes](https://kubernetes.io/)的相关概念,有点疑问.

## **引发思考**

老是在想为啥在[Kubernetes](https://kubernetes.io/)里,原子调度单位是[POD](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/),而不是容器本身?

[Kubernetes](https://kubernetes.io/)里不是有容器了吗?

[POD](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)存在的意义呢?

## **进行总结**

1. 容器的本质是进程,,无非就是通过Namespace做隔离,Cgroups 做限制,rootfs做文件系统,来约束和修改进程的动态表现，从而创造出一个边界,那么顺理成章,
2. [Kubernetes](https://kubernetes.io/)可以理解为一个操作系统,容器是其在上表现为运行的一个个进程,而在操作系统里,进程不是孤立的,而是以进程组的方式的来相互协作的.以进程组的方式调度资源就能合理的组织进程
3. [POD](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)就是表示了一组共享了某些资源的容器。在[POD](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)里的所有容器，共享的是同一个[Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)，并且可以声明共享同一个[Volumes](https://kubernetes.io/docs/concepts/storage/volumes/),那么[Kubernetes](https://kubernetes.io/)以[POD](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)为最小调度单位也是为了更好的解决成组调度所面临的问题.
4. [POD](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)，实际上是在扮演传统基础设施里类似"虚拟机"的角色；而对容器而言，则是这个"虚拟机"里运行的用户程序。
5. [POD](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)是一个概念，抽象出来的理论，提供的是一种编排思想，而不是具体的技术方案,也可以使用虚拟机来作为[POD](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)的实现，然后把用户容器都运行在这个虚拟机里。
比如，Mirantis 公司的[VIRTLET](https://github.com/Mirantis/virtlet)项目就在干这个事情。甚至还可以去实现一个带有 Init 进程的容器项目，来模拟传统应用的运行方式。这些工作，在[Kubernetes](https://kubernetes.io/)中都是非常轻松的，特别是现在使用的[CRI](https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/)(Container Runtime Interface)。
6. 不要强行把整个应用塞到一个容器里，甚至不惜在生产环境使用Docker In Docker这种糟糕的方案,带来的隐患是令人难以接受的.

## **备注:**

Linux 支持7种namespace:

1. **cgroup用于隔离cgroup根目录**
2. **IPC用于隔离系统消息队列**
3. **Network隔离网络**
4. **Mount隔离挂载点**
5. **PID隔离进程**
6. **User隔离用户和用户组**
7. **UTS隔离主机名nis域名**
