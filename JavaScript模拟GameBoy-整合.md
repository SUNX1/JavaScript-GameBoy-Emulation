# JavaScript模拟GameBoy: 整合
  
在第4篇中我们详细地探讨了GameBoy的图形子系统，并进行了模拟。如果没有一组能被软件处理的GPU映射寄存器，模拟器就不能使用图形子系统。一旦这些寄存器准备妥当，基本上就可以使用模拟器的一些基础的功能了。

## GPU寄存器
GameBoy的图形单元有一系列寄存器映射到存储器映射的I/O内存空间中。为了运行具有背景图片的模拟，GPU需要以下寄存器（其他寄存器也可以用于GPU，这部分将在之后的文章中涉及）：

| 地址 | 寄存器 | 状态 |
| --- | --- | --- |
| 0xFF40 | LCD和GPU控制 | Read/write |
| 0xFF42 | Scroll-Y | Read/write |
| 0xFF43 | Scroll-X | Read/write|
| 0xFF44 | 当前扫描线 | Read only |
| 0xFF47 | 背景调色板 | Write only |

表1：基本GPU寄存器

背景调色板寄存器在之前的部分中已经介绍过了，它由四个2比特的调色板组成。滚动寄存器和扫描线计数器是全字节值。剩下的LCD控制寄存器由8个独立的标志组成，控制GPU的各个部分。

| 比特 | 功能 | 为0时 | 为1时 |
| --- | --- | --- | --- |
| 0 | 背景：on/off | Off | On |
| 1 | Sprites：on/off | Off  | On  |
| 2 | Sprites：大小（像素） | 8x8 | 8x16 |
| 3 | 背景：图块图 | #0 | #1 |
| 4 | 背景：图块集 | #0 | #1 |
| 5 | 窗口：on/off | Off | On |
| 6 | 窗口：图块图 | #0 | #1 |
| 7 | 显示: on/off | Off | On |

表2：GPU控制寄存器

在上面的表中，出现了GPU的新特征：可以出现在背景上方的窗口层，以及可以在背景和窗口中移动的sprite对象，这些新特征将在涉及到时介绍。正如我们看到的，背景标志对于基本的渲染功能是最重要的，翻转寄存器0xFF40中的位就可以改变背景的图块图和图块集。

##实现：GPU寄存器

知道了GPU寄存器逻辑设计，就可以通过将这些地址处理程序添加到MMU轻松实现模拟。处理程序可以将GPU更新硬编码到MMU中，也可以定义从MMU调用GPU寄存器的范围并在那里完成专门的处理过程。因为模块化可以带来很多好处，这里采用了第二种方案。

    MMU.js: Zero-page I/O: GPU
    rb: function(addr) {
        switch (addr & 0xF000) {
    	    ...
            case 0xF000:
                switch (addr & 0x0F00) {
    		    ...
                    // Zero-page
                    case 0xF00:
                        if (addr >= 0xFF80) {
                            return MMU._zram[addr & 0x7F];
                        }
                        else {
                            // I/O控制处理
                            switch (addr & 0x00F0) {
                                // GPU (64寄存器)
                                case 0x40: case 0x50: case 0x60: case 0x70:
                                    return GPU.rb(addr);
                            }
                            return 0;
                        }
                }
        }
    },
    
    wb: function(addr, val) {
        switch (addr & 0xF000) {
    	    ...
            case 0xF000:
                switch (addr & 0x0F00) {
    		    ...
                    // Zero-page
                    case 0xF00:
                        if (addr >= 0xFF80) {
                            MMU._zram[addr & 0x7F] = val;
                        }
                        else {
                            // I/O
                            switch (addr & 0x00F0) {
                                // GPU
                                case 0x40: case 0x50: case 0x60: case 0x70:
                                    GPU.wb(addr, val);
                                    break;
                            }
                        }
                        break;
                }
                break;
        }
    }
    
    GPU.js: 寄存器处理
    rb: function(addr) {
        switch (addr) {
            // LCD控制
            case 0xFF40:
                return (GPU._switchbg ? 0x01 : 0x00) |
                    (GPU._bgmap ? 0x08 : 0x00) |
                    (GPU._bgtile ? 0x10 : 0x00) |
                    (GPU._switchlcd ? 0x80 : 0x00);
    
            // Scroll Y
            case 0xFF42:
                return GPU._scy;
    
            // Scroll X
            case 0xFF43:
                return GPU._scx;
    
            // 当前扫描线
            case 0xFF44:
                return GPU._line;
        }
    },
    
    wb: function(addr, val) {
        switch (addr) {
            // LCD控制
            case 0xFF40:
                GPU._switchbg = (val & 0x01) ? 1 : 0;
                GPU._bgmap = (val & 0x08) ? 1 : 0;
                GPU._bgtile = (val & 0x10) ? 1 : 0;
                GPU._switchlcd = (val & 0x80) ? 1 : 0;
                break;
    
            // Scroll Y
            case 0xFF42:
                GPU._scy = val;
                break;
    
            // Scroll X
            case 0xFF43:
                GPU._scx = val;
                break;
    
            // 背景调色板
            case 0xFF47:
                for (var i = 0; i < 4; i++) {
                    switch ((val >> (i * 2)) & 3) {
                        case 0: GPU._pal[i] = [255, 255, 255, 255]; break;
                        case 1: GPU._pal[i] = [192, 192, 192, 255]; break;
                        case 2: GPU._pal[i] = [96, 96, 96, 255]; break;
                        case 3: GPU._pal[i] = [0, 0, 0, 255]; break;
                    }
                }
                break;
        }
    }


## 显示画面

现在模拟器的CPU调度循环将一直运行下去，不会暂停。模拟器最基本的接口应该能让模拟重置或停止。为了实现这个功能，必须使用已知的时间量作为模拟器接口的基本单位。在这有三种时间单位可以选择：

* 指令：在每条CPU指令之后提供暂停的机会。这会导致大量的开销，因为CPU的每一步都必须调用调度函数。CPU的频率是4.19MHz，这将会产生大量的调用。
* 扫描线：在GPU渲染完每行之后暂停。这种方案产生的开销较少，但调度器同样必须每秒调用几千次。
* 画面：让模拟过程在整个画面被模拟、渲染、放入canvas后停止。这是时序精度与最佳速度间的最佳折衷方案，同时还确保了模拟canvas与GPU状态一致。

每幅画面由144条扫描线和一个10行的垂直消隐期组成，每条扫描线需要456时钟周期才能运行完成，所以一幅画面的时长为70224个时钟。结合在开始模拟时初始化每个子系统的模拟器级别的重置函数，就能够运行模拟器并提供基本的接口。

    index.html: 模拟器接口
    <canvas id="screen" width="160" height="144"></canvas>
    <a id="reset">Reset</a> | <a id="run">Run</a>
    
    jsGB.js: 重置和调度
    jsGB = {
        reset: function()
        {
        	GPU.reset();
    	MMU.reset();
    	Z80.reset();
    
    	MMU.load('test.gb');
        },
    
        frame: function()
        {
            var fclk = Z80._clock.t + 70224;
    	do
    	{
    	    Z80._map[MMU.rb(Z80._r.pc++)]();
    	    Z80._r.pc &= 65535;
    	    Z80._clock.m += Z80._r.m;
    	    Z80._clock.t += Z80._r.t;
    	    GPU.step();
    	} while(Z80._clock.t < fclk);
        },
    
        _interval: null,
    
        run: function()
        {
            if(!jsGB._interval)
    	{
    	    jsGB._interval = setTimeout(jsGB.frame, 1);
    	    document.getElementById('run').innerHTML = 'Pause';
    	}
    	else
    	{
    	    clearInterval(jsGB._interval);
    	    jsGB._interval = null;
    	    document.getElementById('run').innerHTML = 'Run';
    	}
        }
    };
    
    window.onload = function()
    {
        document.getElementById('reset').onclick = jsGB.reset;
        document.getElementById('run').onclick = jsGB.run;
        jsGB.reset();
    };

## 测试

添加了GPU寄存器和模拟器基本控制接口后，结果如下：

![](https://ws1.sinaimg.cn/large/006tKfTcly1fo0skeruh3j305k06hweh.jpg)

图1： jsGB图形实现

图1显示的是将代码结合在一起的成果，模拟器已经能够加载和运行基于图形的demo了。这个demo加载的测试ROM是由[Doug Lanford](http://www.opusgames.com)编写的滚动测试，当按下方向键时，显示的背景将滚动。因为还没有模拟键盘，所以现在这种情况下显示的是静态背景。

在接下来部分中，我们将完成模拟键盘的部分，它可以给模拟程序提供恰当的输入。我们将看到键盘是如何工作的，以及输入是如何映射到内存中的。

> 原文链接：[GameBoy Emulation in JavaScript: Integration](http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-Integration)


