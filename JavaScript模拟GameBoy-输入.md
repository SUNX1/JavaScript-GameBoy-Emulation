# JavaScript模拟GameBoy: 输入

通过前面五篇中开发的的模拟器和接口，模拟系统已经能够运行基本的测试ROM并产生图形输出了，但现在的模拟器还不能以按键作为按键输入，并将输入反馈给测试ROM。为此，必须模拟按键对I/O寄存器的影响。

## 按键
GameBoy只有可以按下任意数量键的八键键盘这一种输入方式。对于大多数键盘，键排列在行和列的网格中。当一列被激活时，连接到此列的行也将被激活，硬件能够检测活动行以确定当前按下的键。GameBoy的键盘网格有两列四行，这样设计的好处是可以在一个8位I/O寄存器中完成所有处理。

![图1：键盘输入](http://imrannazar.com/content/img/jsgb-key-wires.png)

图1：键盘输入

由于全部六行都绑定在同一个寄存器上，因此读取键的GameBoy过程绍微有点复杂。

* 将0x10或0x20写入JOYP，这将激活4或5位，一列的行
* 等待行连接传递到JOYP（几个周期）
* 检查JOYP的低4位，找到这一列哪一行激活了

## 按键的实现
编写代码模拟按键按下相对简单，但有两个因素使问题复杂了：允许在读取行之前设置网格中的列，以及使用JavaScript实现按键代码。
为了解决这两个问题，模拟程序中保存两个代表该列和行之间交点的值。另一个需要考虑的问题是键盘的数值是相反的，默认情况下一行保持高电压状态，当与列相交时变为低电压状态。这在I/O寄存器上表现为行不按键为1，按键为0。

JavaScript的`keydown`和`keyup`事件可以用来找出何时键被按下或松开。可以按照以下方式将它们绑定到键盘处理程序。

    Key.js: 对象接口
    KEY = {
        _rows: [0x0F, 0x0F],
        _column: 0,
    
        reset: function()
        {
            KEY._rows = [0x0F, 0x0F];
    	KEY._column = 0;
        },
    
        rb: function(addr)
        {
            switch(KEY._column)
    	{
    	    case 0x10: return KEY._rows[0];
    	    case 0x20: return KEY._rows[1];
    	    default: return 0;
    	}
        },
    
        wb: function(addr, val)
        {
            KEY._column = val & 0x30;
        },
    
        kdown: function(e)
        {
            // 重置适当的位
        },
    
        kup: function(e)
        {
            // 设置适当的位
        }
    };
    
    window.onkeydown = KEY.kdown;
    window.onkeyup = KEY.kup;


除此之外，处理zero-page的同时还要扩展MMU来处理按键I/O寄存器。下面给出一个例子：

    MMU.js: Keypad I/O interface
        rb: function(addr)
        {
    	switch(addr & 0xF000)
    	{
    	    ...
    	    case 0xF000:
    	        switch(addr & 0x0F00)
    		{
    		    ...
    		    // Zero-page
    		    case 0xF00:
    		        if(addr >= 0xFF80)
    			{
    			    return MMU._zram[addr & 0x7F];
    			}
    			else if(addr >= 0xFF40)
    			{
    			    // GPU (64 registers)
    			    return GPU.rb(addr);
    			}
    			else switch(addr & 0x3F)
    			{
    			    case 0x00: return KEY.rb();
    			    default: return 0;
    			}
    		}
    	}
    }

键盘处理程序启动后，剩下的问题是按键的处理以及键盘代码区分不同按下的键，这可以通过JavaScript事件对象实现。如果需要的话，任何浏览器运行时候触发的事件（例如鼠标点击或案件事件）以及描述事件的对象将被传递给代码。按键时，事件对象包含了`character code`和`key scan code`，这两者都描述了按下的键。

Through testing by Peter-Paul Koch, it has been determined that the character code passed by browsers to JavaScript code is unreliable, and will change depending on which browser is used. 

经过[Peter-Paul Koch](https://www.quirksmode.org/js/keys.html)测试，已经确定浏览器传递给JavaScript代码的`character code`是不可靠的，它会随着使用的浏览器而改变。所有浏览器都一致的是为`keyup`和`keydown`事件生成的`key-scan code`。在任何浏览器中，按下给定的键将产生一个特定的值。为了实现模拟，按键代码需要处理8个键：

| Scan code | Key | Mapping |
| --- | --- | --- |
| 13 | Enter | Start |
| 32 | Space | Select |
| 37 | Left arrow | Left |
| 38 | Up arrow | Up |
| 39 | Right arrow | Right |
| 40 | Down arrow | Down |
| 88 | X | B |
| 90 | Z | A |

如上所述，当按下一个按键时，必须重置相应的位，并在释放按键时进行设置。这可以实现如下：

    Key.js: 按键处理
        kdown: function(e)
        {
        	switch(e.keyCode)
    	{
                case 39: KEY._keys[1] &= 0xE; break;
                case 37: KEY._keys[1] &= 0xD; break;
                case 38: KEY._keys[1] &= 0xB; break;
                case 40: KEY._keys[1] &= 0x7; break;
                case 90: KEY._keys[0] &= 0xE; break;
                case 88: KEY._keys[0] &= 0xD; break;
                case 32: KEY._keys[0] &= 0xB; break;
                case 13: KEY._keys[0] &= 0x7; break;
    	}
        },
    
        kup: function(e)
        {
        	switch(e.keyCode)
    	{
                case 39: KEY._keys[1] |= 0x1; break;
                case 37: KEY._keys[1] |= 0x2; break;
                case 38: KEY._keys[1] |= 0x4; break;
                case 40: KEY._keys[1] |= 0x8; break;
                case 90: KEY._keys[0] |= 0x1; break;
                case 88: KEY._keys[0] |= 0x2; break;
                case 32: KEY._keys[0] |= 0x4; break;
                case 13: KEY._keys[0] |= 0x8; break;
    	}
}

## 测试以及下一步
![](https://ws3.sinaimg.cn/large/006tNc79ly1fo25vzdq8rj304y05sglm.jpg)

图1：按键输入实现

上图显示了添加了输入之后的模拟器运行一个简单的井字游戏。在这个例子中，按下Start键（映射到回车键）之后，初始化画面跳到开发组画面，再次按下Start键弹出游戏画面。游戏中玩家是一方，电脑是另一方，按下GameBoy的A键（映射到Z）玩家将放置一个十字或圆圈。

现在，玩上面的游戏时会感觉找不到头绪，因为没有指示把棋子放哪里的标志。游戏通过精灵（sprite）来实现这个标示，精灵是一个可以被图形芯片放置在背景上的图块，可以独立移动。大多数游戏通过使用精灵来实现他们的游戏玩法，因此将精灵构建到模拟中是本系列的下一个重要步骤。下次，我们将看看GameBoy提供的用于渲染精灵的功能，以及如何使用JavaScript实现它们。

> 原文链接：[GameBoy Emulation in JavaScript: Input](http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-Input)


