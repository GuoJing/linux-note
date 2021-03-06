---
layout:    post
title:     进程类型和命名空间
category:  进程
description: 进程类型和命名空间...
tags: 进程 命名空间
---
典型的UNIX进程包括：由二进制代码组成的应用程序、单线程、分配给应用程序的一组资源。信进程是使用fork或者exec系统调用产生的。

### fork ###

生成当前进程的一个相同的副本，该副本称之为*子进程*，原进程的所有资源都以适当的方式复制到子进程，因此该系统调用之后，原来的进程就有了两个独立的实例。这两个实例的联系包括：同一组打开文件、同样的工作目录、内存中同样的数据等等，除此之外没有关联。

### exec ###

从一个可执行的二进制文件加载另一个应用程序，来代替当前运行的进程，换句话说就是加载了信程序。因为exec并不创建信进程，所以必须首先使用fork复制一个旧程序，然后调用exec在系统上创建另一个应用程序。

fork和exec在所有的UNIX操作系统上都是可用的，除此之外Linux还提供了clone系统调用。clone本质上与fork相同，但新的进程不是独立于父进程的，而可以与其共享资源，可以指定需要共享和复制的资源种类，例如父进程的内存数据、打开文件或安装的信号处理程序。clone用于实现线程，但仅仅使用clone还做不到，还需要用户空间库才能提供完整的实现。

### 命名空间 ###

命名空间提供了虚拟化的一种轻量级形式，使得我们可以从不同的方面来查看运行系统的全局属性。该机制类似于Solaris中的zone或者FreeBSD中的jail。

传统上，Linux以及其他的UNIX的变体中，许多资源是全局管理的，例如系统中的所有进程都是通过PID标识的，这意味着内核必须管理一个全局的PID列表，而且所有的调用者通过uname系统调用返回系统的相关信息都是相同的，用户的ID管理方式类似，各个用户是通过一个全局唯一的UID号标识。

比如用户号为0的root用户可以有很大的权利，而普通用户没有特别的权利，传统上需要给每个用户提供一台计算机，又或者使用KVM或VMWare提供的虚拟化环境是一种解决问题的方法，但资源分配做的不是非常好。计算机的各个用户需要一个独立的内核，以及一份完全安装好的配套的用户层应用。

命名空间提供了一种不同的解决方案，所需的资源较少。在虚拟化的系统中，一台物理计算机可以运行多个内核，可能是并行的多个不同的操作系统。而命名空间则在一台物理机只使用一个内核。前面说的全局资源都通过命名空间抽象起来。这使得可以将一组进程放置到容器中，各个容器之间彼此隔离，隔离可以使得容器的成员完全独立，但可以通过允许容器进行一定的共享来降低容器之间的分隔[^1]。

[^1]: 例如容器可以设置为使用自身的PID集合，但仍然与其他容器共享部分文件系统。比如最近流行的Docker就是用了命名空间的原理！

命名空间建立了系统的不同视图，如下图。命名空间可以按层次关联起来，每个命名空间都发源于一个父命名空间，一个父命名空间可以有多各子命名空间。

![system](images/namespace.png)
命名空间

命名空间可以组织为层次结构。一个命名空间是父命名空间，衍生了两个子命名空间。假定容器用于虚拟主机配置中，其中的每个容器必须看起来是一台单独的Linux计算机，因此其中每个都有自身的init进程，PID为0，其他进程的PID以递增次序分配。两个子命名空间都有PID为0的init进程，以及PID为2、3的进程。

虽然子容器不了解系统中的其他容器，但父容器知道子命名空间的存在，也可以看到其中执行的所有进程，可以看到图中的父命名空间中的进程实际上是隐射到子命名空间中的进程的。

命名空间也可以是非层次的，如果命名空间包含的内容是比较简单的，这样父子命名空间之间没有关系。

### 命名空间的创建 ###

使用fork或clone系统调用创建新进程，有特定的选项可以控制是与父进程共享命名空间还是建立新的命名空间。

使用unshare系统调用将进程的某些部分从父进程分离，其中也包括命名空间。
