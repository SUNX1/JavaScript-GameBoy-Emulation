#JavaScript模拟GameBoy: 存储器

在本系列的前一部分中，计算机被描述为从存储器中获取指令的处理单元。绝大部分情况下，一个计算机的存储器并不是一个简单的连续区域，GameBoy在这方面同样如此。由于GameBoy可以访问其地址总线上的65,536个单独的位置，因此可以绘制出CPU可以访问的所有存储器区域的映射。
![GameBoy地址总线的存储器映射](http://imrannazar.com/content/img/jsgb-mmu-map.png)
图1: GameBoy地址总线的存储器映射

更详细的存储器区域如下所示：

* [0000-3FFF] 卡带ROM，存储区0：卡带存储器程序的前16384字节在存储器映射中始终可用。特殊情况下：
  * [0000-00FF] BIOS：当CPU启动时，PC（程序计数器）从0000h开始计数，0000h是GameBoy BIOS代码的开始位置。BIOS运行后，将被从存储器映射中移除，然后这部分卡带ROM变成可寻址的。
  * [0100-014F] 卡带头部：卡带的这部分 包含了卡带名称以及制造商的数据，必须按照特定格式书写。（更多关于卡带头部的细节可以阅读[The Cartridge Header](http://gbdev.gg8.se/wiki/articles/The_Cartridge_Header)）
* [4000-7FFF] 卡带ROM，其他存储区：卡带程序之后的每个16k存储区可以在这里被CPU逐一获取。卡带上的芯片通常用来在存储区间切换，使得特定的区域可访问。最小的程序32k，这意味着不需要存储区选择芯片。
* [8000-9FFF] 图形RAM：图形子系统使用的背景和sprites数据保存在这里，可以通过卡带程序进行更改。这个区域将在本系列的第3部分中进一步详细讨论。
* [A000-BFFF] 卡带（External）RAM：GameBoy中有少量可写的内存。如果制作的游戏硬件内存不足，可以在这里可以使用额外的8k内存。
* [C000-DFFF] 工作RAM：GameBoy内部可以由CPU读取或写入的8k内存。
* [E000-FDFF] 工作RAM（隐藏）：由于GameBoy硬件的接线方式，工作内存的精确副本比在存储器映射中高出8k。直到其他区域访问到映射的最后512字节这个部分才变成可获取的。
* [FE00-FE9F]图形：sprite信息：由图形芯片渲染的sprites的数据被保存在这里，其中包括sprite的位置和属性。
* [FF00-FF7F] 存储器映射I/O：GameBoy的每个子系统（图形、声音等）都有控制值，用来让程序制造效果并使用硬件。在这块区域CPU可以直接在地址总线上使用这些值。
* [FF80-FFFF] 内存顶部有128字节RAM高速区域。奇怪的是，虽然它是整个存储器的末尾，但由于大部分程序和GameBoy硬件之间的交互通过使用这块内存，所以它被称为Zero-page。


##连接到CPU

为了让模拟的CPU能够独立访问这些区域，每一个区域必须在内存管理单元中被处理为一个特殊的case。前面文章已经部分提及了这部分的代码，并且为MMU对象描述了一个基本的接口。实现这个接口只需要一个简单的switch语句。


    MMU.js: Mapped read
    MMU = {
        // Flag indicating BIOS is mapped in
        // BIOS is unmapped with the first instruction above 0x00FF
        // 
        _inbios: 1,
    
        // Memory regions (initialised at reset time)
        _bios: [],
        _rom: [],
        _wram: [],
        _eram: [],
        _zram: [],
    
        // Read a byte from memory
        rb: function(addr)
        {
    	switch(addr & 0xF000)
    	{
    	    // BIOS (256b)/ROM0
    	    case 0x0000:
    	        if(MMU._inbios)
    		{
    		    if(addr < 0x0100)
    		        return MMU._bios[addr];
    		    else if(Z80._r.pc == 0x0100)
    		        MMU._inbios = 0;
    		}
    
    		return MMU._rom[addr];
    
    	    // ROM0
    	    case 0x1000:
    	    case 0x2000:
    	    case 0x3000:
    	        return MMU._rom[addr];
    
    	    // ROM1 (unbanked) (16k)
    	    case 0x4000:
    	    case 0x5000:
    	    case 0x6000:
    	    case 0x7000:
    	        return MMU._rom[addr];
    
    	    // Graphics: VRAM (8k)
    	    case 0x8000:
    	    case 0x9000:
    	        return GPU._vram[addr & 0x1FFF];
    
    	    // External RAM (8k)
    	    case 0xA000:
    	    case 0xB000:
    	        return MMU._eram[addr & 0x1FFF];
    
    	    // Working RAM (8k)
    	    case 0xC000:
    	    case 0xD000:
    	        return MMU._wram[addr & 0x1FFF];
    
    	    // Working RAM shadow
    	    case 0xE000:
    	        return MMU._wram[addr & 0x1FFF];
    
    	    // Working RAM shadow, I/O, Zero-page RAM
    	    case 0xF000:
    	        switch(addr & 0x0F00)
    		{
    		    // Working RAM shadow
    		    case 0x000: case 0x100: case 0x200: case 0x300:
    		    case 0x400: case 0x500: case 0x600: case 0x700:
    		    case 0x800: case 0x900: case 0xA00: case 0xB00:
    		    case 0xC00: case 0xD00:
    		        return MMU._wram[addr & 0x1FFF];
    
    		    // Graphics: object attribute memory
    		    // OAM is 160 bytes, remaining bytes read as 0
    		    case 0xE00:
    		        if(addr < 0xFEA0)
    			    return GPU._oam[addr & 0xFF];
    			else
    			    return 0;
    
    		    // Zero-page
    		    case 0xF00:
    		        if(addr >= 0xFF80)
    			{
    			    return MMU._zram[addr & 0x7F];
    			}
    			else
    			{
    			    // I/O control handling
    			    // Currently unhandled
    			    return 0;
    			}
    		}
    	}
        },
    
        Read a 16-bit word
        rw: function(addr)
        {
            return MMU.rb(addr) + (MMU.rb(addr+1) << 8);
        }
    };
    
在上面的代码中，应该注意0xFF00到0xFF7F之间的内存区域是未处理的。这些位置被用作提供I/O的各种芯片的内存映射I/O，其含义将在后面部分涉及到时介绍。

写字节的处理方式非常相似。每个操作都是相反的，并且值被写入内存的各种区域，而不是从函数中被返回。

##加载ROM
就像CPU模拟如果没有内存访问、图形等基本部分的支持就毫无意义一样，不能加载程序的话，从内存中读取程序同样没什么用。将程序装载到模拟器主要由两种方法：将其直接硬编码到模拟器代码中，或者允许从其他位置加载ROM文件。硬编码进程序明显的缺点是ROM是固定的，不容易更换。

在我们的JavaScript模拟器中，因为GameBoy的BIOS不会改变，所以BIOS被硬编码在MMU中。程序文件是在模拟器初始化之后异步加载的，这一步可以通过XMLHTTP或者Andy Na's BinFileReader之类的二进制文件读取器来完成，最终得到一个包含ROM文件的字符串。

    MMU.js: 加载ROM文件
    MMU.load = function(file)
    {
        var b = new BinFileReader(file);
        MMU._rom = b.readString(b.getFileSize(), 0);
    };

由于ROM文件被保存为字符串而不是整型数组，rb和wb函数必须改成字符串索引：

    MMU.js: ROM文件索引
    	    case 0x1000:
    	    case 0x2000:
    	    case 0x3000:
    	        return MMU._rom.charCodeAt(addr);

##下一步
当CPU和MMU完成后，我们可以看到程序一步步的被执行：获得一个模拟器，并且在正确的寄存器中生成预期的值。现在还缺少的是图像输出，在本系列的下一篇将讨论图形的问题，包括GameBoy如何构建其图形输出，以及如何将图形渲染到屏幕上。


> 原文链接：[GameBoy Emulation in JavaScript: Memory](http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-Memory)


