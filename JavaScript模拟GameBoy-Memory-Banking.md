# JavaScript模拟GameBoy: Memory-Banking
在本系列之前的部分中，我们一直在处理GameBoy的简单的内存映射的加载和模拟，将整个游戏Rom装入内存的下半部分。很少有游戏可以整个装入内存（俄罗斯方块是少数之一），大部分游戏都比这个大，必须要采用独立的机制来将游戏的存储区交换到GameBoy的CPU视图中。

GameBoy库中最早的一些游戏在卡带中设计了一个存储体控制器（MBC），用于在卡带ROM存储区间切换进入视图。多年以来，各种版本的卡带MBC都是为日益大型化的游戏而打造的。本篇实例中的MBC第一版本用于处理64KB ROM的加载。

## Banking and memory expansion
多年以来，许多计算机系统不得不面对太多程序要加载到内存中的问题。一般来说有两种解决方案。

* 增加地址空间：构建拥有更大地址线的CPU，这让CPU能支持更多的内存。这是首选解决方案，但需要大量的时间来重新开发有问题的计算机系统，并且可能需要CPU的芯片组上进行更多的修改。
* 虚拟内存：这指的是在磁盘上保存大块RAM，并在需要时进行交换；或者需要时交换预先写入的ROM块。这两种方式下，系统硬件都只需要很小的扩展，但系统的每个软件都必须知道paging/banking系统才能使用。

GameBoy时一个广泛分布的固定硬件设备，因此在大型游戏生产时无法增加地址空间，但内置于卡带的MBC提供了一种将16KB ROM存储区切换到视图中的方案。除此之外，MBC1还支持卡带内可写的32KB外部内存。如果可用，可将其存储到存储器映射的[0xA000-0xBFFF]空间中。

为了让使用MBC1的软件更容易使用，ROM的第一个16KB存储区（bank 0）固定在地址0x0000。ROM空间的后半部分可以在1到127之间的任何存储区中形成一个窗口，最大ROM大小为2048KB。MBC1的一个特别之处在于在#32存储区内进行的内部处理，存储区#32、#64和#96不可访问，因为它们在存储系统内被视为存储区#0。这意味着除了股东存储区#0以外的125个存储区都可以使用。

MBC1芯片内有四个寄存器来让ROM和RAM交换存储区，可以通过在ROM空间一定范围内写入数据来改变这些寄存器的值。细节如下表所示：


| 位置 | 寄存器 | 细节 |
| --- | --- | --- |
| 0x0000-0x1FFF | Enable external RAM  | 4 bits wide; value of 0x0A enables RAM,any other value disables |
| 0x2000-0x3FFF | ROM bank (low 5 bits) | Switch between banks 1-31 (value 0 is seen as 1) |
| 0x4000-0x5FFF | ROM bank (high 2 bits) RAM bank | ROM mode: switch ROM bank "set" {1-31}-{97-127} RAM mode: switch RAM bank 0-3|
| 0x6000-0x7FFF | Mode |  0: ROM mode (no RAM banks, up to 2MB ROM)1: RAM mode (4 RAM banks, up to 512kB ROM)|

表1：MBC1寄存器集

## MBC和卡带头
由于banking有多种类型的控制器，所以所有游戏都必须在卡带头数据中指明使用哪种MBC。这是卡带ROM的第一块数据，遵循特定的格式。


| 位置 | 值 | 大小（字节） | 细节 |
| --- | --- | --- | --- |
| 0100-0103h | Entry point | 4 | Where the game starts.Usually "NOP; JP 0150h" |
| 0104-0133h | Nintendo logo | 48 | Used by the BIOS to verify checksum |
| 0134-0143h | Title | 16 | Uppercase, padded with 0 |
| 0144-0145h | Publisher | 2 | Used by newer GameBoy games |
| 0146h | Super GameBoy flag | 1 | Value of 3 indicates SGB support |
| 0147h | Cartridge type | 1 | MBC type/extras |
| 0148h | ROM size | 1 | Usually between 0 and 7,Size = 32kB << [0148h] |
| 0149h | RAM size | 1 | Size of external RAM |
| 014Ah | Destination | 1 | 0 for Japan market, 1 otherwise |
| 014Bh | Publisher | 1 | Used by older GameBoy games |
| 014Ch | ROM version | 1 | Version of the game, usually 0 |
| 014Dh | Header checksum | 1 | Checked by BIOS before loading |
| 014E-014Fh | Global checksum | 2 | Simple summation, not checked |
| 0150h | Start of game |  |  |

表2：卡带头部格式

如果MBC1安装在卡带上，卡带类型可以是以下值：


| 值 | 描述 |
| --- | --- |
| 00h | No MBC |
| 01h | MBC1 |
| 02h | MBC1 with external RAM |
| 03h | MBC1 with battery-backed external RAM |

表3：MBC1相关的卡带类型值

就本文而言，外部RAM不会执行备用电池系统，这个特性经常被游戏用来保存它们的状态以备以后使用，这在后面的部分会详细介绍。

## MBC1的实现
存储体控制器狠命是内存操作，所以可以很好的整合到MMU中。由于第一个ROM存储区（bank #0）是固定的，所以只需要保存MBC的偏移量来指示正在读取的第二个存储区的位置。为了稍后添加更多的MBC处理，可以使用一组数据来保存控制器的状态：

    MMU.js: MBC state and reset
    MMU = {
        // MBC states
        _mbc: [],
    
        // Offset for second ROM bank
        _romoffs: 0x4000,
    
        // Offset for RAM bank
        _ramoffs: 0x0000,
    
        // Copy of the ROM's cartridge-type value
        _carttype: 0,
    
        reset: function()
        {
            ...
    
    	// In addition to previous reset code,
    	// initialise MBC internal data
    	MMU._mbc[0] = {};
    	MMU._mbc[1] = {
    	    rombank: 0,		// Selected ROM bank
    	    rambank: 0,		// Selected RAM bank
    	    ramon: 0,		// RAM enable switch
    	    mode: 0		// ROM/RAM expansion mode
    	};
    
    	MMU._romoffs = 0x4000;
    	MMU._ramoffs = 0x0000;
        },
    
        load: function(file)
        {
            ...
    	MMU._carttype = MMU._rom.charCodeAt(0x0147);
        }
    }

从上面的代码可以看出，MBC1的四个寄存器的内部状态由MMU中的一个对象表示，与MBC类型1相关联。当这些被更改时，可以修改ROM和RAM偏移量来指向适当的内存存储区，一旦设置了指针，就快可以正常访问内存了。

    MMU.js: MBC1-based access
    MMU = {
        rb: function(addr)
        {
        	switch(addr & 0xF000)
    	{
    	    ...
    
    	    // ROM (switched bank)
    	    case 0x4000:
    	    case 0x5000:
    	    case 0x6000:
    	    case 0x7000:
    	        return MMU._rom.charCodeAt(MMU._romoffs +
    		                           (addr & 0x3FFF));
    
    	    // External RAM
    	    case 0xA000:
    	    case 0xB000:
    	        return MMU._eram[MMU._ramoffs +
    		                 (addr & 0x1FFF)];
    	}
        }
    };

这些指针偏移的计算是在写入MBC寄存器时执行的，如下所示。

    MMU.js: MBC1 control
        wb: function(addr, val)
        {
            switch(addr & 0xF000)
    	{
    	    // MBC1: External RAM switch
    	    case 0x0000:
    	    case 0x1000:
    	        switch(MMU._carttype)
    		{
    		    case 2:
    		    case 3:
    			MMU._mbc[1].ramon =
    			    ((val & 0x0F) == 0x0A) ? 1 : 0;
    			break;
    		}
    		break;
    
    	    // MBC1: ROM bank
    	    case 0x2000:
    	    case 0x3000:
    	        switch(MMU._carttype)
    		{
    		    case 1:
    		    case 2:
    		    case 3:
    		        // Set lower 5 bits of ROM bank (skipping #0)
    			val &= 0x1F;
    			if(!val) val = 1;
    			MMU._mbc[1].rombank =
    			    (MMU._mbc[1].rombank & 0x60) + val;
    
    			// Calculate ROM offset from bank
    			MMU._romoffs = MMU._mbc[1].rombank * 0x4000;
    			break;
    		}
    		break;
    
    	    // MBC1: RAM bank
    	    case 0x4000:
    	    case 0x5000:
    	        switch(MMU._carttype)
    		{
    		    case 1:
    		    case 2:
    		    case 3:
    		    	if(MMU._mbc[1].mode)
    			{
    			    // RAM mode: Set bank
    			    MMU._mbc[1].rambank = val & 3;
    			    MMU._ramoffs = MMU._mbc[1].rambank * 0x2000;
    			}
    			else
    			{
    			    // ROM mode: Set high bits of bank
    			    MMU._mbc[1].rombank =
    			    	(MMU._mbc[1].rombank & 0x1F) +
    				((val & 3) << 5);
    			
    			    MMU._romoffs = MMU._mbc[1].rombank * 0x4000;
    			}
    			break;
    		}
    		break;
    
    	    // MBC1: Mode switch
    	    case 0x6000:
    	    case 0x7000:
    	        switch(MMU._carttype)
    		{
    		    case 2:
    		    case 3:
    		    	MMU._mbc[1].mode = val & 1;
    			break;
    		}
    		break;
    
    	    ...
    
    	    // External RAM
    	    case 0xA000:
    	    case 0xB000:
    	        MMU._eram[MMU._ramoffs + (addr & 0x1FFF)] = val;
    		break;
    	}
        }

`In the above control code, instances of MBC1 that are stated as having external RAM attached are the ones which have RAM banking.`有了这个代码，演示程序就可以正常加载和运行; 如果没有MBC1处理程序，代码将在试图访问显示器的精灵和背景数据时崩溃。

## 接下来
除了能够将更大的游戏装入内存之外，游戏更重要的一个方面就是保持时间的能力。例如基于时钟的游戏如果没有某种基于时钟的时钟机制就毫无意义。 如前所述，许多游戏使用垂直消隐中断来计时，但有些游戏需要更细粒度的时间结构。这是由GameBoy提供的一个基于CPU时钟的硬件定时器。

定时器还提供了一种检查CPU时钟的方法，这使得它可以用作随机数发生器的种子。例如俄罗斯方块使用硬件定时器的这个功能来选择方块。 在下一部分中，我将详细介绍计时器的工作原理以及如何实现。

> 原文链接：[GameBoy Emulation in JavaScript: Memory-Banking](http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-Memory-Banking)


