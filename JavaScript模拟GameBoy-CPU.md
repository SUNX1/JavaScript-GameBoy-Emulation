#JavaScript模拟GameBoy: CPU

JavaScript通常被描述为专门为实现网站动态交互而设计的语言，其实JavaScript是一个完整的面相对象编程语言，除了Web以外还应用于其他领域，比如Windows和MacOS最新版本的Widget就是JavaScript实现的，Mozilla应用程序套件的GUI同样如此。

我们知道桌面应用可以模拟C64（1982年Commodore公司推出的8位PC）、GameBoy（你懂的）以及其他游戏主机，随着HTML引入了 <canvas>标签，JavaScript程序是否也能模拟一个系统呢？检验是否可行最简单的办法就是尝试用JavaScript写一个这样的模拟器。

本文着手实现GameBoy模拟的基础，为模拟物理机器的每个部分奠定基础，而起点就是CPU。

##模型


计算机的传统模型是一个通过指令程序被告知要做什么的处理单元。该程序可能位于自己的特殊内存，也可能位于正常内存区域。指令逐条被执行，每条指令只消耗很短的时间。从CPU的角度来看，只要打开计算机，一个循环就会启动，从内存中获取指令，计算出它的内容并执行它。

为了跟踪CPU在程序中的位置，CPU保存了一个叫做程序计数器的数字（PC）。一条指令被从内存中取出后，程序计数器会增加指令的字节数。

![The fetch-decode-execute loop](http://imrannazar.com/content/img/jsgb-cpu-fetchloop.png)
图1: 取指令-解码-执行 循环

原版GBA的CPU是经过修改的Zilog Z80，所以有下面几点是相同的：

* Z80是8位的芯片，所有的内部工作一次只能操作一个字节
* 内存接口最多可以寻址65536字节（16位地址总线）
* 程序通过与普通内存相同的地址总线进行访问
* 指令可以在1到3个字节之间的任何地方

除了PC之外，CPU内部还有一些可用于计算的数字，它们被称为寄存器A、B、C、D、E、H和L。它们全都是一个字节，所以可以保存0到255之间的值。Z80中的大部分指令被用来处理这些寄存器中的值，例如将内存中的值加载到寄存器中，增加或减少值等等。

因为指令的第一个字节中有256个可能的值，所以在基本表中就有256个可能的指令。该表在本站发布的[GameBoy Z80操作码映射表](/Gameboy-Z80操作码映射表.md)中有详细说明。每个指令都可以通过JavaScript函数来模拟，该函数对寄存器的内模进行操作，并将得到的结果交给内存接口的内模。

Z80中还有其他一些标志寄存器（F）用于处理保持状态，其操作在下面会详细说明。堆栈指针（SP）与PUSH和POP指令一起用于基本的LIFO值处理。

因此基本的Z80模拟模型需要一下组件：

* 内部状态
    * 保存寄存器当前状态的结构
    * 执行最后一条指令的时间量
    * CPU总运行的时间量
* 模拟每条指令的函数
* 将所述函数映射到操作码映射表的表格
* 用于和模拟的内存交互的公开接口

内部的状态可以这样存储：

    Z80.js: 内部状态值
    Z80 = {
        // 时钟: Z80有两种时钟(m和t)
        _clock: {m:0, t:0},
        
        // 寄存器集合
        _r: {
            a:0, b:0, c:0, d:0, e:0, h:0, l:0, f:0,    // 8位寄存器
            pc:0, sp:0,                                // 16位寄存器
            m:0, t:0                                   // 最后一条指令的Clock
        }
    };    

标志寄存器（F）对于处理器的运转很重要，它会根据上一次操作的结果自动计算某些位或标志。Gameboy Z80有四个标志：

* Zero（0x80）：当最后一次操作产生的结果为0时的标志
* Operation（0x40）：当最后一次操作是减法时的标志
* Half-carry （0x20）：当最后一次操作的结果中低位字节溢出超过15时的标志
* Carry（0x10）：当最后一个操作产生的结果超过255（加法）或小于0（减法）时的标志

由于基本计算寄存器是8位的, Carry（进位标志）允许软件在计算结果溢出寄存器时计算出发生了什么。下面给出了一些考虑了标志处理问题的指令模拟例子。这些例子被简化了，而且没有考虑半进位标志。

    Z80.js: 指令模拟
    Z80 = {
        // 内部状态
        _clock: {m:0, t:0},
        _r: {a:0, b:0, c:0, d:0, e:0, h:0, l:0, f:0, pc:0, sp:0, m:0, t:0},
    
        // E+A，把计算存储到A （ADD A, E）
        ADDr_e: function() {
            Z80._r.a += Z80._r.e;                      // 执行加法
            Z80._r.f = 0;                              // 清除标志
            if(!(Z80._r.a & 255)) Z80._r.f |= 0x80;    // 检查Zero标志
            if(Z80._r.a > 255) Z80._r.f |= 0x10;       // 检查Carry标志
            Z80._r.a &= 255;                           // mask 8位
            Z80._r.m = 1; Z80._r.t = 4;                // m时钟置1
        }
    
        // 比较B和A，设置标志 (CP A, B)
        CPr_b: function() {
            var i = Z80._r.a;                          // A的临时副本
            i -= Z80._r.b;                             // 减B
            Z80._r.f |= 0x40;                          // 设置Operation标志
            if(!(i & 255)) Z80._r.f |= 0x80;           // 检查Zero标志
            if(i < 0) Z80._r.f |= 0x10;                // 检查下溢
            Z80._r.m = 1; Z80._r.t = 4;                // m时钟置1
        }
    
        // 空指令 (NOP)
        NOP: function() {
            Z80._r.m = 1; Z80._r.t = 4;                // m时钟置1
        }
    };


##内存接口
一个可以操纵其内部的寄存器的处理器当然很好，但它必须能够把结果放到内存中才有用。同样，上面的CPU模拟需要一个模拟内存的接口，这可以由内存管理单元（MMU）提供。因为Gameboy本身不包含复杂的MMU，所以模拟组件可以非常简单。

现在CPU只需要知道有一个接口存在，而Gameboy如何将内存和硬件库映射到地址总线上的细节对于处理器的操作是无关紧要的。CPU只需要四个操作：

    MMU.js: 内存接口
    MMU = {
        rb: function(addr) { /* 从传入的地址中读8位字节（byte） */ },
        rw: function(addr) { /* 从传入的地址中读16位字（word） */ },
    
        wb: function(addr, val) { /* 向传入的地址中写8位字节（byte） */ },
        ww: function(addr, val) { /* 向传入的地址中写16位字（word） */ }
    };

有了这些，就可以模拟其余的CPU指令了。下面是另外几个例子：

    Z80.js: 内存处理模拟
    // 寄存器B和C入栈 (PUSH BC)
    PUSHBC: function() {
        Z80._r.sp--;                               // 清栈顶
        MMU.wb(Z80._r.sp, Z80._r.b);               // 写B
        Z80._r.sp--;                               // 清栈顶
        MMU.wb(Z80._r.sp, Z80._r.c);               // 写C
        Z80._r.m = 3; Z80._r.t = 12;               // m时钟置3
    },
    
    // 寄存器H和L出栈 (POP HL)
    POPHL: function() {
        Z80._r.l = MMU.rb(Z80._r.sp);              // 读L
        Z80._r.sp++;                               // 移回栈
        Z80._r.h = MMU.rb(Z80._r.sp);              // 读H
        Z80._r.sp++;                               // 移回栈
        Z80._r.m = 3; Z80._r.t = 12;               // m时钟置3
    }
    
    // 从绝对地址读1个字节到A (LD A, addr)
    LDAmm: function() {
        var addr = MMU.rw(Z80._r.pc);              // 从指令中获取地址
        Z80._r.pc += 2;                            // 程序计数器+2
        Z80._r.a = MMU.rb(addr);                   // 从地址读取值
        Z80._r.m = 4; Z80._r.t = 16;               // m时钟置4
    }

##度和重置

当指令准备就绪后，剩下的问题就是启动时重置CPU，并给模拟程序提供指令。有一个重置程序允许CPU停止并恢复到执行起始点。下面是一个例子：

    Z80.js: 重置
    reset: function() {
    	Z80._r.a = 0; Z80._r.b = 0; Z80._r.c = 0; Z80._r.d = 0;
    	Z80._r.e = 0; Z80._r.h = 0; Z80._r.l = 0; Z80._r.f = 0;
    	Z80._r.sp = 0;
    	Z80._r.pc = 0;      // 从0开始执行
    
    	Z80._clock.m = 0; Z80._clock.t = 0;
    }

为了能够模拟Z80运行，程序必须完整的模拟前面提到的读取、解码、执行这一序列操作。“执行”指令由指令模拟函数负责，读取和解码指令需要专门的代码，我们称之为调度循环。这个循环接受每条指令，解码出它必须被送往哪里执行，并将其分配给相关函数。

    Z80.js: 分配器
    while(true)
    {
        var op = MMU.rb(Z80._r.pc++);              // 获取命令
        Z80._map[op]();                            // 分配
        Z80._r.pc &= 65535;                        // mask PC 16位
        Z80._clock.m += Z80._r.m;                  // 将时间加到CPU时钟
        Z80._clock.t += Z80._r.t;
    }
    
    Z80._map = [
        Z80._ops.NOP,
        Z80._ops.LDBCnn,
        Z80._ops.LDBCmA,
        Z80._ops.INCBC,
        Z80._ops.INCr_b,
        ...
    ];


##Usage in a system emulation
实现了Z80模拟核心却没有模拟器来运行它是毫无作用的。在本系列的下一部分将开始GameBoy的模拟工作：我将思考GameBoy的内存映射问题，以及如何通过Web将游戏图像加载到模拟器中。

完整的Z80内核可在以下网址获取：[http：//imrannazar.com/content/files/jsgb.z80.js](http：//imrannazar.com/content/files/jsgb.z80.js) 

如果你在实践过程中遇到任何困扰，请随时联系我。
Imran Nazar <tf@imrannazar.com>，2010年7月。


> 原文链接：[GameBoy Emulation in JavaScript: the CPU](http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-The-CPU)

