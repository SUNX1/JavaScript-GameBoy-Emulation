# JavaScript模拟GameBoy: GPU时序
 
在本系列的前几部分中，我们提出了一个GameBoy模拟器的架构，实现了从加载游戏ROM，使用模拟CPU来执行命令。通过模拟处理器连接到内存映射结构，现在可以将外设连接进系统了。GameBoy和主机游戏主要用到的一个外设是图形处理器（GPU）。GPU是游戏机输出的主要手段，大部分处理器的工作都是为GPU生成图像。

## 模拟屏幕

GameBoy在任天堂的内部名称叫做“点矩阵游戏”，它的显示器是一个160x144像素的LCD屏幕。如果将LCD中的每个像素都视为HTML <canvas>中的一个像素，那么GameBoy的LCD可以直接映射为一个宽160高144的canvas。为了直接操作LCD的每个像素，可以将canvas内容作为“帧缓冲器”来操作：包含整个canvas的内存块、一系列4字节RGBA值。

    index.html: <canvas>
    <canvas id="screen" width="160" height="144"></canvas>
    
    GPU.js: Canvas初始化
    GPU = {
        _canvas: {},
        _scrn: {},
    
        reset: function()
        {
            var c = document.getElementById('screen');
    	if(c && c.getContext)
    	{
    	    GPU._canvas = c.getContext('2d');
    	    if(GPU._canvas)
    	    {
    		if(GPU._canvas.createImageData)
    		    GPU._scrn = GPU._canvas.createImageData(160, 144);
    
    		else if(GPU._canvas.getImageData)
    		    GPU._scrn = GPU._canvas.getImageData(0,0, 160,144);
    
    		else
    		    GPU._scrn = {
    		        'width': 160,
    			'height': 144,
    			'data': new Array(160*144*4)
    		    };
    
    		// 将canvas初始化为白色
    		for(var i=0; i<160*144*4; i++)
    		    GPU._scrn.data[i] = 255;
    
    		GPU._canvas.putImageData(GPU._scrn, 0, 0);
    	    }
    	   }
        }
    }
    
一旦为屏幕数据分配了一块内存，就可以将像素的RGBA存为内存块对应位置中的四个值来设置每个像素的颜色，像素的位置可以由公示 `y * 160 + x.`确定。

## 光栅图形

有了一个可以接收GameBoy图形输出的画布，下一步是模拟图形的产生。原版GameBoy硬件在其timings中模拟阴极射线管（CRT）：在CRT中，屏幕被电子束逐行扫描，一次扫描结束后从屏幕顶部重新开始下一次扫描。

![图1：扫描线和消隐期](http://imrannazar.com/content/img/jsgb-gpu-blank.png)
图1：扫描线和消隐期

从上面可以看出，由于CRT需要一个水平消隐期（HBlank）让光束从一行的末尾移到下一行的开始，与简单的遍历像素相比CRT需要消耗更多的时间来绘制扫描线。类似地，每个画面结束后需要垂直消隐期来让光束返回左上角。由于电子束在垂直消隐期中移动的更远，所以垂直消隐期时间通常比水平消隐期时间长得多。

同样地，GameBoy显示器有水平消隐期和垂直消隐期。另外，扫描线本身花费的时间分为两部分：在绘制扫描线时，GPU在图像内存（VRAM）和sprite属性内存（OAM）间切换访问。


为了模拟这个，这两部分是截然不同的，并且互相关联。下表列出了当CPU的T时钟以4194304Hz运行时，GPU在每个周期这种停留多长时间。

| 时期 | GPU模式编号 | 花费时间 (时钟) |
| --- | --- | --- |
| 扫描线 (访问OAM) | 2 | 80 |
| 扫描线 (访问VRAM) | 3 | 172 |
| 水平消隐期 | 0 | 204 |
| 一条线 (扫描和消隐期) |  | 456 |
| 垂直消隐期 | 1 | 4560 (10行) |
| 整个画面 (扫描和垂直消隐期) |  | 70224 |

表1: GPU 画面时序

In order to maintain these timings relative to the emulated CPU, 
为了保持这些时序能与模拟的CPU相对应，需要一个在每个指令执行后调用的定时更新函数。我们可以扩展第1篇中介绍的调度过程来实现：


    Z80.js: 调度器
    while(true)
    {
        Z80._map[MMU.rb(Z80._r.pc++)]();
        Z80._r.pc &= 65535;
        Z80._clock.m += Z80._r.m;
        Z80._clock.t += Z80._r.t;
    
        GPU.step();
    }
    
    GPU.js: 时钟step
        _mode: 0,
        _modeclock: 0,
        _line: 0,
    
        step: function()
        {
            GPU._modeclock += Z80._r.t;
    
    	switch(GPU._mode)
    	{
    	    // 读OAM模式，扫描线激活
    	    case 2:
    	        if(GPU._modeclock >= 80)
    		{
    		    // 进入扫描线模式3
    		    GPU._modeclock = 0;
    		    GPU._mode = 3;
    		}
    		break;
    
    	    // 读VRAM模式，扫描线激活
    	    // 将模式3结束视为扫描线结束
    	    case 3:
    	        if(GPU._modeclock >= 172)
    		{
    		    // 进入水平水平消隐期
    		    GPU._modeclock = 0;
    		    GPU._mode = 0;
    
    		    // 将扫描线写入帧缓冲器
    		    GPU.renderscan();
    		}
    		break;
    
    	    // 水平消隐期
    	    // 水平消隐期结束后将屏幕数据放入canvas
    	    case 0:
    	        if(GPU._modeclock >= 204)
    		{
    		    GPU._modeclock = 0;
    		    GPU._line++;
    
    		    if(GPU._line == 143)
    		    {
    		        // 进入垂直消隐期
    			GPU._mode = 1;
    			GPU._canvas.putImageData(GPU._scrn, 0, 0);
    		    }
    		    else
    		    {
    		    	GPU._mode = 2;
    		    }
    		}
    		break;
    
    	    // 水平消隐期（10行）
    	    case 1:
    	        if(GPU._modeclock >= 456)
    		{
    		    GPU._modeclock = 0;
    		    GPU._line++;
    
    		    if(GPU._line > 153)
    		    {
    		        // 重新启动扫描模式
    			GPU._mode = 2;
    			GPU._line = 0;
    		    }
    		}
    		break;
    	   }
        }

## 下一篇： 背景图和调色板
在上面的代码中，GPU的时序已经建立，但GPU进行渲染扫描工作的部分还没有完成。在本系列的下一篇中，将介绍GameBoy背景图系统背后的概念，并将代码放入渲染函数中进行模拟。

> 原文链接：[GameBoy Emulation in JavaScript: Timings](http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-GPU-Timings)


