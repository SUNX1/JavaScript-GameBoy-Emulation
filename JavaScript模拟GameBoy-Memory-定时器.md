# JavaScript模拟GameBoy: 定时器
计算机出现以来，记录时间就是其基本功能之一，即根据计时器协调动作。即使是最简单的游戏也包含时间部分。例如《乒乓球》需要以特定的速度在屏幕上移动球。为了处理这些时间问题，游戏主机都有某种形式的计时器，来让事件在特定时刻或以特定速率发生。

GameBoy同样也包含了一组基于可编程时间表自动递增的寄存器。在本系列的这部分中，我们将研究定时器的结构和操作，以及如何将它用于伪随机数生成器，例如俄罗斯方块中的伪随机数生成器。下面演示了一个使用定时器来挑选随机棋子的俄罗斯方块例子。

## 定时器结构
如本系列第一部分所述，GameBoy的CPU运行时钟频率为4194304Hz，其中以每个时钟步长递增的T时钟以及以四分之一的速度（1048576Hz）递增的M时钟两个内部指标用于执行每条指令。定时器基于这些时钟，以M时钟速率的四分之一（262144Hz
）递增。在本文中，我们把这个最终值作为定时器的基础速度。

GameBoy的定时器硬件提供两个独立的定时器寄存器，系统通过以预定速率递增每个寄存器中的值来工作。分频器定时器永久设置为16384Hz，即基础速度的十六分之一。由于它是一个8位寄存器，所以它的值在达到255后会回到零。

定时器具有更高的可编程性，它可以设置为四种速度中的一种（基数除以1、4、16或64），并且可以设置回到非零值，溢出超过255。此外，如第8部分所述，只要计数器定时器溢出，定时器硬件将向CPU发送中断。

有四个寄存器用于定时器。就像图形和中断寄存器一样，这些寄存器可供系统作为I / O页面的一部分使用。

| 地址 | 寄存器 | 细节 |
| --- | --- | --- |
| 0xFF04 | 分频器 | 计数固定在16384Hz;无论何时写入，都重置为0 |
| 0xFF05 | 计数器 | 按指定的速率计算;当255-> 0时触发INT 0x50 |
| 0xFF06 | 模运算 | 当计数器溢出到0时，它重置为从模运算开始 |
| 0xFF07 | 控制 | 见表2 |

表1：定时器寄存器

| 位 | 功能 | 细节 |
| --- | --- | --- |
| 0-1 | 速度 | 00: 4096Hz,01: 262144Hz,10: 65536Hz,11: 16384Hz |
| 2 | 运行 | 1运行定时器，0停止 |
| 3-7 | 未使用 |  |

表2：定时器控制寄存器细节

由于计数器定时器在溢出时会触发中断，因此如果游戏需要定期发生某些事情，那么它会特别有用。然而Gameboy游戏通常可以使用垂直消隐来达到相同的效果，因为它以近乎60Hz的固定速度发生，垂直消隐处理器不仅可用于刷新屏幕内容，还可用于检查键盘和更新游戏状态。因此，尽管计时器可以在图形演示中用于更大的效果，在传统的Gameboy游戏中几乎没有要求使用计时器。

## 实现定时器模拟
本系列文章中开发的模拟器使用CPU时钟作为时间的基本单位，因此维护与CPU时钟同步的定时器的时钟最简单，并且定时器由调度功能进行更新。在这个阶段，将DIV寄存器作为可控定时器的独立实体很方便，并以最快定时器步骤的1/16速率递增：

    Timer.js: 时钟递增
    TIMER = {
        _clock: {
            main: 0,
    	sub:  0,
    	div:  0
        },
    
        _reg: {
            div:  0,
    	tima: 0,
    	tma:  0,
    	tac:  0
        },
    
        inc: function()
        {
            // 增加最后一个操作码的时间
            TIMER._clock.sub += Z80._r.m;
    
    	// 没有操作码需要超过4M的时间，所以我们只需要检查一次溢出
    	if(TIMER._clock.sub >= 4)
    	{
    	    TIMER._clock.main++;
    	    TIMER._clock.sub -= 4;
    
    	    // DIV寄存器以1/16的速率递增，因此保留一个计数
    	    TIMER._clock.div++;
    	    if(TIMER._clock.div == 16)
    	    {
    	        TIMER._reg.div = (TIMER._reg.div+1) & 255;
    		TIMER._clock.div = 0;
    	    }
    	}
    
    	// 检查是否需要在定时器中执行一个步骤
    	TIMER.check();
        }
    };
    Z80.js: 调度器
    while(true)
    {
        // 为此指令执行
        var op = MMU.rc(Z80._r.pc++);
        Z80._map[op]();
        Z80._r.pc &= 65535;
        Z80._clock.m += Z80._r.m;
        Z80._clock.t += Z80._r.t;
    
        // 更新定时器
        TIMER.inc();
    
        Z80._r.m = 0;
        Z80._r.t = 0;
    
        // 如果IME处于打开状态，并且某些中断在IE中启用，并且设置了中断标志，则处理该中断
        if(Z80._r.ime && MMU._ie && MMU._if)
        {
            // Mask未启用的整数
            var ifired = MMU._ie & MMU._if;
    
    	if(ifired & 0x01)
    	{
    	    MMU._if &= (255 - 0x01);
    	    Z80._ops.RST40();
    	}
        }
    
        Z80._clock.m += Z80._r.m;
        Z80._clock.t += Z80._r.t;
    
        // 再次更新定时器，以防发生RST
        TIMER.inc();
    }

从这里开始，可控计时器由基本速度的不同分区组成，这使得检查计时器值是否需要加速以及将寄存器提供为内存I/O页面的一部分变得相对简单。以下代码和MMU、I/O页面处理程序之间的接口作为读者的练习。

    Timer.js: 寄存器检查和更新
        check: function()
        {
            if(TIMER._reg.tac & 4)
    	{
    	    switch(TIMER._reg.tac & 3)
    	    {
    	        case 0: threshold = 64; break;		// 4K
    		case 1: threshold =  1; break;		// 256K
    		case 2: threshold =  4; break;		// 64K
    		case 3: threshold = 16; break;		// 16K
    	    }
    
    	    if(TIMER._clock.main >= threshold) TIMER.step();
    	}
        },
    
        step: function()
        {
            // 计时器+1
    	TIMER._clock.main = 0;
        	TIMER._reg.tima++;
    
    	if(TIMER._reg.tima > 255)
    	{
    	    // 溢出时，用Modulo重新填充
    	    TIMER._reg.tima = TIMER._reg.tma;
    
    	    // 将定时器中断标记到调度程序
    	    MMU._if |= 4;
    	}
        },
    
        rb: function(addr)
        {
    	switch(addr)
    	{
    	    case 0xFF04: return TIMER._reg.div;
    	    case 0xFF05: return TIMER._reg.tima;
    	    case 0xFF06: return TIMER._reg.tma;
    	    case 0xFF07: return TIMER._reg.tac;
    	}
        },
    
        wb: function(addr, val)
        {
    	switch(addr)
    	{
    	    case 0xFF04: TIMER._reg.div = 0; break;
    	    case 0xFF05: TIMER._reg.tima = val; break;
    	    case 0xFF06: TIMER._reg.tma = val; break;
    	    case 0xFF07: TIMER._reg.tac = val & 7; break;
    	}
    }

## 生成伪随机数发生器
许多游戏的一个重要组成部分是随机性，例如俄罗斯方块会落下一个未知的方块。理想情况下，计算机通过生成随机数字来提供随机机制，但计算机不能提供真正的随机数字。有各种算法来产生表面看似随机的数字序列，这些算法被称为伪随机数生成（PRNG）算法。

PRNG通常作为一个公式来实现，即给定一个特定的输入数字，产生与输入几乎没有关系的另一个数字。对于俄罗斯方块而言不需要那么复杂，下面的代码用来产生一个看似随机的块。

    Tetris.asm: Select new block
    BLK_NEXT = 0xC203
    BLK_CURR = 0xC213
    REG_DIV  = 0x04
    
    NBLOCK: ld hl, BLK_CURR		; Bring the next block
    	ld a, (BLK_NEXT)	; forward to current
    	ld (hl),a
    	and 0xFC		; Clear out any rotations
    	ld c,a			; and hold onto previous
    
    	ld h,3			; Try the following 3 times
    
    .seed:	ldh a, (REG_DIV)	; Get a "random" seed
    	ld b,a
    .loop:	xor a			; Step down in sevens
    .seven:	dec b			; until zero is reached
    	jr z, .next		; This loop is equivalent
    	inc a			; to (a%7)*4
    	inc a
    	inc a
    	inc a
    	cp 28
    	jr z, .loop
    	jr .seven
    
    .next:	ld e,a			; Copy the new value
    	dec h			; If this is the
    	jr z, .end		; last try, just use this
    	or c			; Otherwise check
    	and 0xFC		; against the previous block
    	cp c			; If it's the same again,
    	jr z, .seed		; try another random number
    .end:	ld a,e			; Get the copy back
    	ld (BLK_NEXT), a	; This is our next block

俄罗斯方块块选择器的基础是DIV寄存器，由于选择例程每隔几秒才运行一次，因此寄存器在任何给定的运行中都会有一个未知值，因此它可以很好地作为随机数源。

> 原文链接：[GameBoy Emulation in JavaScript: Timers](http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-Timers)


