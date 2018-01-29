# JavaScript模拟GameBoy: 图形
 
在本系列之前的几篇中，我们已经构建了GameBoy模拟器的轮廓，建立了CPU和图形处理器之间的时序，初始化了一个canvas并准备好由模拟器来绘制图形。现在GPU的模拟已经有了具体的结构，但它仍然无法将图形渲染到帧缓冲器。为了模拟渲染图形，必须简单介绍下GameBoy图形背后的概念。

## 背景

就像那个年代大多数的游戏机一样，GameBoy内存不够大，不能让帧缓冲直接存到内存中。取而代之的是使用瓦片系统（tile system）：内存中保存了一组很小的位图，使用这些位图的引用来构建map。这个系统的先天优势在于map可以通过瓦片的引用重复使用同一个瓦片。

GameBoy的瓦片图形系统使用8x8像素的tile块，一个map中可以使用256个独立的tile。内存中可以存放两张32x32 tile的map，同一时刻显示其中一张。内存中有384个tile的空间，因此有一半的瓦片是被两张map共享的：一个map使用0-255编号的tile，而另一个使用编号-128到127的tile。

在图像内存中，瓦片数据和map的规化如下：

| 区域 | 使用 |
| --- | --- |
| 8000-87FF | Tile set #1: tiles 0-127 |
| 8800-8FFF | Tile set #1: tiles 128-255 Tile set #0: tiles -1 to -128|
| 9000-97FF | Tile set #0: tiles 0-127 |
| 9800-9BFF | Tile map #0 |
| 9C00-9FFF | Tile map #1 |

表1: VRAM规化

定义背景时通过其map和瓦片数据交互来产生图形显示：

![图1：背景映射](http://imrannazar.com/content/img/jsgb-gpu-bg-map.png)
图1：背景映射
 
如前所示，背景map由32x32的瓦片构成。总共有256x256像素。GameBoy的显示屏是160x144像素的，所以背景的范围可以相对于屏幕移动。GPU通过在背景中定义一个相对于屏幕左上角的点来实现这点：通过在画面中移动这个点来使背景在屏幕上滚动。为此，GPU的寄存器Scroll X和Scroll Y存储了左上角的相对位置。

![图2：背景滚动寄存器](http://imrannazar.com/content/img/jsgb-gpu-bg-scrl.png)
图2：背景滚动寄存器

## 调色板
人们常说GameBoy是单色设备，只能显示黑色和白色。但这并不正确，GameBoy也可以处理浅灰色和深灰色，总共四种颜色。瓦片数据保存这四种颜色之一需要两位，因此瓦片数据集中的每个瓦片需要8x8x2位或16字节。



> 原文链接：[GameBoy Emulation in JavaScript: Graphics](http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-Graphics)


