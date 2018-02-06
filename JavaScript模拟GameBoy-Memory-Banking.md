# JavaScript模拟GameBoy: Memory-Banking
在本系列之前的部分中，我们一直在处理GameBoy的简单的内存映射的加载和模拟，将整个游戏Rom装入内存的下半部分。很少有游戏可以整个装入内存（俄罗斯方块是少数之一），大部分游戏都比这个大，必须要采用独立的机制来将游戏的存储区交换到GameBoy的CPU视图中。

GameBoy库中最早的一些游戏在卡带中设计了一个存储体控制器（MBC），用于在卡带ROM存储区间切换进入视图。多年以来，各种版本的卡带MBC都是为日益大型化的游戏而打造的。本篇实例中的MBC第一版本用于处理64KB ROM的加载。

## Banking and memory expansion
多年以来，许多计算机系统不得不面对太多程序要加载到内存中的问题。一般来说有两种解决方案。

* 增加地址空间：构建拥有更大地址线的CPU，这让CPU能支持更多的内存。这是首选解决方案，但需要大量的时间来重新开发有问题的计算机系统，并且可能需要CPU的芯片组上进行更多的修改。
* 虚拟内存：这指的是在磁盘上保存大块RAM，并在需要时进行交换；或者需要时交换预先写入的ROM块。这两种方式下，系统硬件都只需要很小的扩展，但系统的每个软件都必须知道paging/banking系统才能使用。

GameBoy时一个广泛分布的固定硬件设备，因此在大型游戏生产时无法增加地址空间，但内置于卡带的MBC提供了一种将16KB ROM存储区切换到视图中的方案。除此之外，MBC1还支持卡带内可写的32KB外部内存。


> 原文链接：[GameBoy Emulation in JavaScript: Memory-Banking](http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-Memory-Banking)


