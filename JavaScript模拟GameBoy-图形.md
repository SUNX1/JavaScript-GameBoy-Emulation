# JavaScript模拟GameBoy: 图形
 
在本系列之前的几篇中，我们已经构建了GameBoy模拟器的轮廓，建立了CPU和图形处理器之间的时序，初始化了一个canvas并准备好由模拟器来绘制图形。现在GPU的模拟已经有了具体的结构，但它仍然无法将图形渲染到帧缓冲器。为了模拟渲染图形，必须简单介绍下GameBoy图形背后的概念。

## 背景

就像那个年代大多数的游戏机一样，GameBoy内存不够大，不能让帧缓冲直接存到内存中。取而代之的是使用图块系统（tile system）：内存中保存了一组很小的位图，使用这些位图的引用来构建map。这个系统的先天优势在于map可以通过图块的引用重复使用同一个图块。

GameBoy的图块图形系统使用8x8像素的tile块，一个map中可以使用256个独立的tile。内存中可以存放两张32x32 tile的map，同一时刻显示其中一张。内存中有384个tile的空间，因此有一半的图块是被两张map共享的：一个map使用0-255编号的tile，而另一个使用编号-128到127的tile。

在图像内存中，图块数据和map的规化如下：

| 区域 | 使用 |
| --- | --- |
| 8000-87FF | Tile set #1: tiles 0-127 |
| 8800-8FFF | Tile set #1: tiles 128-255 Tile set #0: tiles -1 to -128|
| 9000-97FF | Tile set #0: tiles 0-127 |
| 9800-9BFF | Tile map #0 |
| 9C00-9FFF | Tile map #1 |

表1: VRAM规化

定义背景时通过其map和图块数据交互来产生图形显示：

![图1：背景映射](http://imrannazar.com/content/img/jsgb-gpu-bg-map.png)
图1：背景映射
 
如前所示，背景map由32x32的图块构成。总共有256x256像素。GameBoy的显示屏是160x144像素的，所以背景的范围可以相对于屏幕移动。GPU通过在背景中定义一个相对于屏幕左上角的点来实现这点：通过在画面中移动这个点来使背景在屏幕上滚动。为此，GPU的寄存器Scroll X和Scroll Y存储了左上角的相对位置。

![图2：背景滚动寄存器](http://imrannazar.com/content/img/jsgb-gpu-bg-scrl.png)
图2：背景滚动寄存器

## 调色板
人们常说GameBoy是单色设备，只能显示黑色和白色。但这并不正确，GameBoy也可以处理浅灰色和深灰色，总共四种颜色。图块数据保存这四种颜色之一需要两位，因此图块数据集中的每个图块需要8x8x2位或16字节。

`GameBoy背景另一个难题是从图块数据到最终显示之间的调色盘：图块像素四个可能值中的每一个可以对应四种颜色中的任何一种。这主要用于让图块集的颜色更容易变更。例如，如果保存了一组对应英文字母表的图块集，反向显示版本可以通过改变调色盘来实现，而不是占用图块集的另一部分。通过改变背景调色盘GPU寄存器的值，四个调色板条目一次全部更新，显示颜色引用和寄存器结构。`

| 值 | 像素 | 模拟的颜色 |
| --- | --- | --- |
| 0 | Off | [255, 255, 255] |
| 1 | 33% on | [192, 192, 192] |
| 2 | 66% on | [96, 96, 96] |
| 3 | On | [0, 0, 0] |
表2：颜色相关值

![http://imrannazar.com/content/img/jsgb-gpu-bg-pal.png](http://imrannazar.com/content/img/jsgb-gpu-bg-pal.png)
图3:背景调色板寄存器

## 实现：图块数据
如上所述，图块数据集中的每个像素由两个比特表示，当图块在map中被引用时，这些比特被GPU读取，通过调色板并推到屏幕上。`GPU的硬件是wired，这使得图块的一行可以同时被访问，像素通过运行位来循环访问。`唯一的问题是，图块的一行是两个字节，由此产生了比较复杂的存储方案，其中每个像素的低位保存在一个字节中，高位保存在另一个字节中。

![图4：图块数据位图结构](http://imrannazar.com/content/img/jsgb-gpu-bg-tile.png)
图4：图块数据位图结构

由于JavaScript不太适合于位图结构的快速处理，处理图块数据集时间效率最高的方式是在图像内存旁边维护一个内部数据集，并用一个更大的视图来预先计算每个像素的值。为了准确映射图块数据集，对图像内存的任何写入都必须调用更新GPU内部图块数据的函数。

    GPU.js: 内部图块数据
        _tileset: [],
    
        reset: function()
        {
            // 除了之前重置以外的代码
    	GPU._tileset = [];
    	for(var i = 0; i < 384; i++)
    	{
    	    GPU._tileset[i] = [];
    	    for(var j = 0; j < 8; j++)
    	    {
    	        GPU._tileset[i][j] = [0,0,0,0,0,0,0,0];
    	    }
    	}
        },
    
        // 将值写入VRAM然后更新内部图块数据集
        
        updatetile: function(addr, val)
        {
            // 获取这个图块行的基地址
    	addr &= 0x1FFE;
    
    	// 确定哪个图块和行已更新
    	var tile = (addr >> 4) & 511;
    	var y = (addr >> 1) & 7;
    
    	var sx;
    	for(var x = 0; x < 8; x++)
    	{
    	    // 找到这个像素的位索引
    	    sx = 1 << (7-x);
    
    	    // 更新图块集
    	    GPU._tileset[tile][y][x] =
    	        ((GPU._vram[addr] & sx)   ? 1 : 0) +
    	        ((GPU._vram[addr+1] & sx) ? 2 : 0);
        }
    }
        
    MMU.js: 图块更新触发器
        wb: function(addr, val)
        {
            switch(addr & 0xF000)
    	{
            // 只显示VRAM情况：
    	    case 0x8000:
    	    case 0x9000:
    		GPU._vram[addr & 0x1FFF] = val;
    		GPU.updatetile(addr, val);
    		break;
    	}
    }

## 实现：扫描渲染

有了这些组件，就可以开始渲染GameBoy屏幕了。由于是逐行完成的，因此在渲染扫描线之前，第3部分中提到的渲染扫描函数必须找出它在屏幕上的位置。这涉及使用滚动寄存器和当前扫描线计数器计算背景map中位置的X和Y坐标。一旦确定了这个位置，扫描渲染器就能扫过map中一行中的每个图块，每遇到一个图块就读入新的图块数据。

    GPU.js: 扫描渲染
        renderscan: function()
        {
        
	// VRAM offset for the tile map
    	var mapoffs = GPU._bgmap ? 0x1C00 : 0x1800;
    
	// Which line of tiles to use in the map
    	mapoffs += ((GPU._line + GPU._scy) & 255) >> 3;
    	
    	// Which tile to start with in the map line
    	var lineoffs = (GPU._scx >> 3);
    
    	// Which line of pixels to use in the tiles
    	var y = (GPU._line + GPU._scy) & 7;
    
    	// Where in the tileline to start
    	var x = GPU._scx & 7;
    
            // Where to render on the canvas
    	var canvasoffs = GPU._line * 160 * 4;
    
    	// Read tile index from the background map
    	var colour;
    	var tile = GPU._vram[mapoffs + lineoffs];
    
    	// If the tile data set in use is #1, the
    	// indices are signed; calculate a real tile offset
    	if(GPU._bgtile == 1 && tile < 128) tile += 256;
    
    	for(var i = 0; i < 160; i++)
    	{
    	    // Re-map the tile pixel through the palette
    	    colour = GPU._pal[GPU._tileset[tile][y][x]];
    
    	    // Plot the pixel to canvas
    	    GPU._scrn.data[canvasoffs+0] = colour[0];
    	    GPU._scrn.data[canvasoffs+1] = colour[1];
    	    GPU._scrn.data[canvasoffs+2] = colour[2];
    	    GPU._scrn.data[canvasoffs+3] = colour[3];
    	    canvasoffs += 4;
    
    	    // When this tile ends, read another
    	    x++;
    	    if(x == 8)
    	    {
    		x = 0;
    		lineoffs = (lineoffs + 1) & 31;
    		tile = GPU._vram[mapoffs + lineoffs];
    		if(GPU._bgtile == 1 && tile < 128) tile += 256;
    	    }
    	}
    }

## 下一步: 输出
有了CPU、内存处理和图形子系统，模拟器已经几乎可以输出了。在第5篇中，我们将研究如何将不同的模块文件整合到一个整体的系统，加载并运行一个简单的ROM文件，将图形寄存器绑定到MMU，并设计一个简单的接口来控制模拟的运行。

> 原文链接：[GameBoy Emulation in JavaScript: Graphics](http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-Graphics)


