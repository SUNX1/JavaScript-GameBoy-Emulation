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



> 原文链接：[GameBoy Emulation in JavaScript: Sprites](http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-Sprites)


