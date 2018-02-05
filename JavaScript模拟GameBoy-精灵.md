# JavaScript模拟GameBoy: 精灵

在本系列之前的部分中，模拟器完成了按键输入，这意味着已经可以玩井字棋游戏了。剩下的问题是只能盲目地玩游戏，因为没有下一步行动将在何处进行的标志，也不知道游戏里按键将会把你移动到哪里。二维游戏程序通常使用精灵来解决这个问题。精灵是可以独立于背景放置的可移动对象块，其包含了独立于背景的数据。GameBoy在这方面也不例外， 它可以将精灵放置在背景之上或之下，并同时在屏幕上显示多个精灵。

## 介绍：GameBoy精灵
GameBoy精灵是图新图块，跟用于背景的图块一样，每个精灵8x8像素。如上所述，精灵可以放置在屏幕的任意位置上，部分或全部放在屏幕外也可以，并且可以放在背景之上或之下。这在技术上意味着背景之下的精灵将显示颜色值0。

![图1：精灵优先级](http://imrannazar.com/content/img/jsgb-gpu-sprite-prio.png)

图1：精灵优先级

在上图中，背景穿过精灵的中心部分显示出来，因为背景之上的精灵这些像素设置为乐颜色0。同样的，背景下面的精灵在背景颜色为0的部分显示出来。在模拟器中模拟这个机制，最简单的方式是先渲染背景之下的精灵，然后渲染背景本身，最后渲染背景之上的精灵。然而这是个算法有点简陋了，它重复了精灵的渲染过程。更简单的办法是先绘制背景，然后根据优先级和背景颜色确定精灵中的给定的像素是否应该出现。


    精灵渲染伪代码
    For each row in sprite
        If this row is on screen
            For each pixel in row
                If this pixel is on screen
                    If this pixel is transparent
                        * Do nothing
                    Else 
                        If the sprite has priority
                            Draw pixel
                        Else if this pixel in the background is 0
                            Draw pixel
                        Else
                            * Do nothing
                        End If
                    End If
                End If
            End For
        End If
    End For


GameBoy精灵系统的另一个难点是精灵可以在渲染时被硬件水平或垂直翻转。这可以节省游戏空间，例如向后飞行的太空船可以用经过适当的翻转的向前的精灵来表示。

## 精灵数据：对象属性内存

GameBoy可以在名为对象属性内存（OAM）的专用内存区域内存储40个精灵的信息。40个精灵中每个精灵在OAM中都有4字节数据，如下：


| 字节 | 描述 |
| --- | --- |
| 0 | 左上角的Y坐标（存储的值是Y坐标减去16） |
| 1 | 左上角的X坐标（存储的值是X坐标减去8） |
| 2 | 数据块编号 |
| 3 | 选项 |

表1：精灵的OAM数据（选项见表2）

| 位 | 描述 | 为0时 | 为1时 |
| --- | --- | --- | --- |
| 7 | 精灵/背景 优先级 | 在背景之上 | 在背景之下（除颜色0以外） |
| 6 | X翻转 | 默认 | 垂直翻转 |
| 5 | Y翻转 | 默认 | 水平翻转 |
| 4 | 调色盘 | OBJ 调色板 #0 | OBJ 调色板 #1 |

表2：精灵的OAM数据选项

为了渲染扫描线时更方便访问这些信息，建立一个根据OAM来设值的结构来保存精灵数据时非常有用的。当数据被写入OAM，图形模拟配合的MMU可以更新此结构以备之后使用。实现如下所示：

    MMU.js: OAM访问
        rb: function(addr)
        {
    	switch(addr & 0xF000)
    	{
    	    ...
    	    case 0xF000:
    	        switch(addr & 0x0F00)
    		{
    		    ...
    		    // OAM
    		    case 0xE00:
    		        return (addr < 0xFEA0) ? GPU._oam[addr & 0xFF] : 0;
    		}
    	}
        },
    
        wb: function(addr)
        {
    	switch(addr & 0xF000)
    	{
    	    ...
    	    case 0xF000:
    	        switch(addr & 0x0F00)
    		{
    		    ...
    		    // OAM
    		    case 0xE00:
    		        if(addr < 0xFEA0) GPU._oam[addr & 0xFF] = val;
    			GPU.buildobjdata(addr - 0xFE00, val);
    			break;
    		}
    	}
        }
    GPU.js: 精灵结构
        _oam: [],
        _objdata: [],
    
        reset: function()
        {
            // 添加到之前的重制代码：
    	for(var i=0, n=0; i < 40; i++, n+=4)
    	{
    	    GPU._oam[n + 0] = 0;
    	    GPU._oam[n + 1] = 0;
    	    GPU._oam[n + 2] = 0;
    	    GPU._oam[n + 3] = 0;
    	    GPU._objdata[i] = {
    	        'y': -16, 'x': -8,
    		'tile': 0, 'palette': 0,
    		'xflip': 0, 'yflip': 0, 'prio': 0, 'num': i
    	    };
    	}
        },
    
        buildobjdata: function(addr, val)
        {
    	var obj = addr >> 2;
    	if(obj < 40)
    	{
    	    switch(addr & 3)
    	    {
    	        // Y坐标
    	        case 0: GPU._objdata[obj].y = val-16; break;
    		
    		// X坐标
    		case 1: GPU._objdata[obj].x = val-8; break;
    
    		// 数据块
    		case 2: GPU._objdata[obj].tile = val; break;
    
    		// 选项
    		case 3:
    		    GPU._objdata[obj].palette = (val & 0x10) ? 1 : 0;
    		    GPU._objdata[obj].xflip   = (val & 0x20) ? 1 : 0;
    		    GPU._objdata[obj].yflip   = (val & 0x40) ? 1 : 0;
    		    GPU._objdata[obj].prio    = (val & 0x80) ? 1 : 0;
    		    break;
    	    }
    	}
        }

## 渲染精灵
GameBoy图形系统逐行渲染屏幕，其中不仅包括背景，还包括其下的精灵。换句话说，绘制背景之后，必须将渲染精灵添加到扫描线渲染器。和背景一样，LCDC寄存器中有一个开关来激活精灵，这必须添加到GPU的I/O处理中。

因为精灵可以在包括屏幕外的任何位置，渲染器必须检查有哪些精灵位于当前扫描线内。最简单的算法是检查每一个精灵的位置，如果它落在扫描线范围内，则渲染适当的精灵的线。通过预先计算的图块集，可以使用与背景相同的方式来检索精灵数据。下面这个例子囊括了上面提到的过程的实现：

    GPU.js: 渲染一个有精灵的扫描线
        renderscan: function()
        {

            // 供精灵渲染器使用的扫描线数据
    	var scanrow = [];
    
            // 如果它打开则渲染背景
            if(GPU._switchbg)
    	{
    	    var mapoffs = GPU._bgmap ? 0x1C00 : 0x1800;
    	    mapoffs += ((GPU._line + GPU._scy) & 255) >> 3;
    	    var lineoffs = (GPU._scx >> 3);
    	    var y = (GPU._line + GPU._scy) & 7;
    	    var x = GPU._scx & 7;
    	    var canvasoffs = GPU._line * 160 * 4;
    	    var colour;
    	    var tile = GPU._vram[mapoffs + lineoffs];
    
    	    // 如果使用的是图块数据集 #1，标记索引
    	    // 计算真实的图块偏移量
    	    if(GPU._bgtile == 1 && tile < 128) tile += 256;
    
    	    for(var i = 0; i < 160; i++)
    	    {

    	        // 通过调色板重映射图块像素
    	        colour = GPU._pal.bg[GPU._tileset[tile][y][x]];
    
    	        // 绘制像素到canvas
    	        GPU._scrn.data[canvasoffs+0] = colour[0];
    	        GPU._scrn.data[canvasoffs+1] = colour[1];
    	        GPU._scrn.data[canvasoffs+2] = colour[2];
    	        GPU._scrn.data[canvasoffs+3] = colour[3];
    	        canvasoffs += 4;
    
    		// 存储像素以供稍后检查
    		scanrow[i] = GPU._tileset[tile][y][x];
    
    	        // 当前图块结束时，读取另一个图块
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
    
    	// 如果它们打开则渲染精灵
    	if(GPU._switchobj)
    	{
    	    for(var i = 0; i < 40; i++)
    	    {
    	        var obj = GPU._objdata[i];
    

    		// 检查这个精灵是否落在这条扫描线上
    		if(obj.y <= GPU._line && (obj.y + 8) > GPU._line)
    		{
    		    // 用于这个精灵的调色板
    		    var pal = obj.pal ? GPU._pal.obj1 : GPU._pal.obj0;
    
            // 在canvas上渲染哪里
    		    var canvasoffs = (GPU._line * 160 + obj.x) * 4;
    
    		    // 当前精灵的线的数据
    		    var tilerow;
    
    		    // 如果精灵进行了Y翻转，使用图块的另一面
    		    if(obj.yflip)
    		    {
    		        tilerow = GPU._tileset[obj.tile]
    			                      [7 - (GPU._line - obj.y)];
    		    }
    		    else
    		    {
    		        tilerow = GPU._tileset[obj.tile]
    			                      [GPU._line - obj.y];
    		    }
    
    		    var colour;
    		    var x;
    
    		    for(var x = 0; x < 8; x++)
    		    {
    		   // 如果这个像素还在屏幕内，且颜色不是0（透明），且精灵有优先级或显示在背景之下，则渲染像素
    			if((obj.x + x) >= 0 && (obj.x + x) < 160 &&
    			   tilerow[x] &&
    			   (obj.prio || !scanrow[obj.x + x]))
    			{
    			    // 如果精灵经过X翻转，以相反的顺序写入像素
    			    colour = pal[tilerow[obj.xflip ? (7-x) : x]];
    
    			    GPU._scrn.data[canvasoffs+0] = colour[0];
    			    GPU._scrn.data[canvasoffs+1] = colour[1];
    			    GPU._scrn.data[canvasoffs+2] = colour[2];
    			    GPU._scrn.data[canvasoffs+3] = colour[3];
    
    			    canvasoffs += 4;
    			}
    		    }
    		}
    	    }
    	}
        },
        
        rb: function(addr)
        {
            switch(addr)
    	{
    	    // LCD控制
    	    case 0xFF40:
    	        return (GPU._switchbg  ? 0x01 : 0x00) |
    		       (GPU._switchobj ? 0x02 : 0x00) |
    		       (GPU._bgmap     ? 0x08 : 0x00) |
    		       (GPU._bgtile    ? 0x10 : 0x00) |
    		       (GPU._switchlcd ? 0x80 : 0x00);
    
    	    // ...
    	}
        },
    
        wb: function(addr, val)
        {
            switch(addr)
    	{
    	    // LCD控制
    	    case 0xFF40:
        	   GPU._switchbg  = (val & 0x01) ? 1 : 0;
        		GPU._switchobj = (val & 0x02) ? 1 : 0;
        		GPU._bgmap     = (val & 0x08) ? 1 : 0;
        		GPU._bgtile    = (val & 0x10) ? 1 : 0;
        		GPU._switchlcd = (val & 0x80) ? 1 : 0;
        		break;
    	    // ...
    	}
        }

## 接下来
实现精灵后，像井字棋这样的简单的游戏已经可以完整运行了，然而许多游戏需要确定屏幕合适可以重新绘制的方法才能运行。因为直到下一次GPU画出一帧时屏幕的改变才会显示出来，所以几乎所有游戏都会在屏幕垂直消隐时刷新屏幕数据。

简单的游戏和demo有时是通过检查GPU的绘制过程是否达到了第144行来实现这一机制的，但这在重复循环中占用了很多处理能力。更常见的方式是当事件发生时通知游戏，这个消息被称为中断。在下一篇中，我们将关注垂直消隐中断，以及如何模拟将消息传递给模拟的游戏。

> 原文链接：[GameBoy Emulation in JavaScript: Sprites](http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-Sprites)


