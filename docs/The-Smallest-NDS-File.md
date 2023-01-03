# 最小的 NDS 文件

> 原文：<http://web.archive.org/web/20220810161336/https://imrannazar.com/The-Smallest-NDS-File>

你有没有想过任天堂 DS ROM 文件里面到底有什么，为什么简单的 DS 演示要比 GBA 版大那么多？有些人做到了，这一页记录了他们的功绩。

### 介绍

开始的时候很天真。我在找一个小的 ROM，用来测试 DSemu 的帧缓冲显示模式，一个任天堂 DS 模拟器。LiraNuna 同意做一个小的 C 演示程序，用红色填充主屏幕，演示 framebuffer 的用法。编译和拼接时。nds 最终在 7.5KB 左右。

利拉努纳认为，对于像他的演示那样做得如此之少的东西来说，这个数字有点大。在使用 DSemu 的调试器时，我注意到运行了大量不是严格要求的代码:设置缓存参数和堆栈，清除内存区域，等等。这个代码被称为 crt0，它被插入到每个项目中，以保护执行环境。

此外，标准的 ARM7 代码也插入了。nds 文件，它可以做诸如设置触摸屏之类的事情。我们认为，对于一个实际上几乎什么都不做的演示来说，所有这些都有点过头了。于是，砍伐开始了。

### 早期阶段:砍手臂 7

首先，LiraNuna 认为 ARM7 的功能并不是这个演示所特别需要的。所以，思考的过程是，为什么不简单地告诉那个“子”CPU 进入一个无限循环，什么也不做？理由是合理的，因此 ARM7 源文件被替换为一个简单的汇编文件，如下所示。

#### ARM7 裁减:无限循环

```
main: b main
```

一旦放在一起，就减少了整体的大小。nds 文件通过相当的方式；低至大约 5KB。然而，我仍然认为这有点大。快速浏览一下。nds 文件说明了原因:与主 CPU 一样，子 CPU 也有一个由构建过程自动插入的 crt0，这构成了的 ARM7 部分的绝大部分。nds 文件。

因此，LiraNuna 采取了颠覆部分构建过程的步骤，删除了 ARM7 编译的结果，并用一个直接的二进制文件替换它，对无限循环操作码进行编码。

#### ARM7 缩减:最终二进制

```
0000   FE FF FF EA
```

这使得整个二进制文件大约为 4KB。我想，还有很大的改进空间。

### 下一步:组装

主要代码仍然是 C 语言，编译成 Thumb 二进制。在 DSemu 的调试器中，我注意到编译器引入了一些奇怪的东西，这些东西似乎做得很少；值先左移，然后再右移，没有整体效果，以及类似的奇怪现象。因此，下一个合乎逻辑的步骤是在没有编译器干预的情况下，用汇编语言编写这一部分。

LiraNuna 第一次尝试了汇编版本的程序，如下所示。

#### ARM9 削减:第一轮

```
main:
		@ sets POWER_CR
	mov r0, #0x4000000
	orr r0, r0, #0x300
	orr r0, r0, #0x4
	mov r1, #0x3
	str r1, [r0]

		@ sets mode
	mov r0, #0x04000000
	mov r1, #0x00020000
	str r1, [r0]

		@ sets VRAM bank a
	mov r0, #0x04000000
	add r0, r0, #0x240
	mov r1, #0x80
	strb r1, [r0]

		@ loop
	mov r0, #0x06800000
	mov r1, #0x1F
	orr r1, r1, #0x8000
	mov r2, #0x18000

	filloop:
		strh r1, [r0], #0x1
		subs r2, r2, #0x1
		bne filloop

	lforever:
      b lforever

```

当汇编起来的时候，那绝对是不同的；整个 ROM 的大小下降到大约 1.5KB。然而，我开始有一个线索，我们可以做得更好。这时，百事公司提出了一个建议:将代码*放在*中。nds 标题。

### 更深入:在。nds 文件

百事公司这么说是什么意思？为了理解这一点，了解 a .nds ROM 的内部是什么样子是很重要的。

| 文件偏移量 | 成分 |
| --- | --- |
| 0000 | NDS ROM 标题(512 字节) |
| 0200 | ARM9 二进制 |
| 0200+ARM9 | ARM7 二进制 |
| 0200+两者都有 | 可选文件表 |

传统的布局规定主 CPU 的二进制文件放在头文件之后，子 CPU 的二进制文件放在头文件之后。然而，这并不总是正确的；顺序可以交换，可以在二进制文件之间或之后插入空格。

这一切都很好，但是标题里面的*？为了理解这一点，需要查看文件的顶部:ROM 头。*

 *#### 标题结构:ndstool 示例输出

```
0x00  Game title                 
0x0C  Game code                  ####
0x10  Maker code                 
0x12  Unit code                  0x00
0x13  Device type                0x00
0x14  Device capacity            0x00 (1 Mbit)
0x15  (8 bytes blank space)
0x1E  ROM version                0x00
0x1F  reserved                   0x04
0x20  ARM9 ROM offset            0x200
0x24  ARM9 entry address         0x2000000
0x28  ARM9 RAM address           0x2000000
0x2C  ARM9 code size             0x3A0
0x30  ARM7 ROM offset            0x600
0x34  ARM7 entry address         0x3800000
0x38  ARM7 RAM address           0x3800000
0x3C  ARM7 code size             0x8
0x40  File name table offset     0x608
0x44  File name table size       0x9
0x48  FAT offset                 0x614
0x4C  FAT size                   0x0
0x50  ARM9 overlay offset        0x0
0x54  ARM9 overlay size          0x0
0x58  ARM7 overlay offset        0x0
0x5C  ARM7 overlay size          0x0
0x60  ROM control info 1         0x00586000
0x64  ROM control info 2         0x001808F8
0x68  Icon/title offset          0x0
0x6C  Secure area CRC            0x0000 (-, homebrew)
0x6E  ROM control info 3         0x0000
0x70  (16 bytes blank space)
0x80  Application end offset     0x00000000
0x84  ROM header size            0x00000200
0x88  (36 bytes blank space)
0xAC  PassMe autoboot detect     0x53534150 ("PASS")
0xB0  (16 bytes blank space)
0xC0  Nintendo Logo (156 bytes)
0x15C Logo CRC                   0x9E1A (OK)
0x15E Header CRC                 0xC9D3 (OK)
0x160 (160 bytes blank space)
```

突出显示为红色的条目表示标题结构中的空白区域。这些通常在格式的构造过程中被留下，以允许扩展。然而，在这种情况下，可以利用头部的空白区域来保存代码。

从上面的输出中，很容易看出。nds 文件作为一个整体是由这个标题中的条目决定的。ARM9 二进制文件跟在文件头之后的原因很简单，就是将“ARM9 ROM offset”设置为`0x200`，这是文件中文件头之后的第一个字节。类似地，ARM9 之后的 ARM7 代码是“ARM7 ROM 偏移量”设置为`0x600`的简单效果，对应于文件中 1.5KB 的偏移量。

只需改变这个头文件中的“ROM 偏移量”值，就可以改变 CPU 代码的加载点，从头文件之后的默认位置到头文件内部的某个位置；用 ARM 操作码覆盖那个位置的零，并从那里加载。百事公司认为这是一个好主意，而且可行。

LiraNuna 的 ARM9 代码看起来很短，但我认为我可以做得更好，进一步缩短代码。

#### ARM9 缩减:第二轮

```
main:
    mov r0,#0x04000000      ; I/O space offset
    mov r1,#0x3             ; Both screens on
    mov r2,#0x00020000      ; Framebuffer mode
    mov r3,#0x80            ; VRAM bank A enabled, LCD

    str r1,[r0, #0x304]     ; Set POWERCNT
    str r2,[r0]             ;     DISPCNT 
    str r3,[r0, #0x240]     ;     VRAMCNT_A

    mov r0,#0x06800000      ; VRAM offset
    mov r1,#31              ; Writing red pixels
    mov r2,#0xC000          ; 96k of them

lp: strh r1,[r0],#2         ; Write a pixel
    subs r2,r2,#1           ; Move along one
    bne lp                  ; And loop back if not done

nf: b nf                    ; Sit in an infinite loop to finish

```

组装后，这段代码看起来像下面这样。

#### ARM9 精简版:汇编二进制

```
0000   01 03 A0 E3   03 10 A0 E3   02 28 A0 E3   80 30 A0 E3
0010   04 13 80 E5   00 20 80 E5   40 32 80 E5   1A 05 A0 E3
0020   1F 10 A0 E3   03 29 A0 E3   B2 10 C0 E0   01 20 52 E2
0030   FC FF FF 1A   FE FF FF EA
```

肯定小一点；现在剩下的问题是把它和 ARM7 二进制代码(`EAFFFFFE`)放在哪里。ARM7 非常简单:第一个空白区域(8 字节)有足够的空间来放置这个操作码。ARM7 失调发生了变化，尺寸变为 4，这部分已经完成。

ARM9 代码的放置也同样简单:头文件末尾的 160 字节空闲空间似乎足够存放二进制文件，剩下的就是修改 ARM9 ROM 的偏移量和大小。

这似乎是。所有的代码都可以轻松地放入标题和最后的。nds 只有 512 字节大小。这肯定是唯一能做的了吗？不完全是。

### 核心:重新定位

事实证明，并非所有 512 字节的报头都被使用。按照惯例，末尾的 160 个字节位于报头中；有人可能会说。nds 文件由 352 字节的头、160 字节的填充和两个 CPU 二进制文件组成。有没有可能将 56 字节的 ARM9 二进制文件放在报头中的其他地方，并消除这种填充？

我首先改变了`0x84`处的“报头大小”字段，以反映报头的新大小，这将是`0x160`字节。然后，我开始插入操作码，直到我有这样的东西。

#### ARM9 位置:割台内

```
0070   01 03 A0 E3   03 10 A0 E3   02 28 A0 E3   80 30 A0 E3
0080   00 00 00 00   A0 01 00 00   04 13 80 E5   00 20 80 E5
0090   40 32 80 E5   1A 05 A0 E3   1F 10 A0 E3   03 29 A0 E3
00A0   B2 10 C0 E0   01 20 52 E2   FC FF FF 1A   50 41 53 53
00B0   FE FF FF EA   00 00 00 00   00 00 00 00   00 00 00 00
```

可以看到报头中 0x80、0x84 和 0xAC 处的字段，位于 ARM9 代码内。这是一个相当大的问题。如果这些值对应于有效的操作码，它们可能会被执行，这可能会证明对程序的状态是灾难性的。

要求进行拆卸。我在 DSemu 中加载了新的二进制文件，调试器给出了以下输出:

#### ARM9 删减:插入后的代码

```
    mov r0,#0x04000000     
    mov r1,#0x3            
    mov r2,#0x00020000     
    mov r3,#0x80
    andeq r0, r0, r0
    andeq r0, r0, r0, lsr #3
    str r1,[r0, #0x304]    
    str r2,[r0]            
    str r3,[r0, #0x240]    
    mov r0,#0x06800000     
    mov r1,#31             
    mov r2,#0xC000
lp: strh r1,[r0],#2        
    subs r2,r2,#1          
    bne lp                 
    cmppls r3, #0x14
nf: b nf                   

```

看来我很幸运。前两个 AND 语句永远不会被执行，因为它们依赖于被设置的零标志，而所述标志不是由上述指令设置的。至于 CMP，它在 VRAM 写循环之后就位，这确实是幸运的；如果 CMP 在 BNE 之前失败，循环可能会永远执行下去，最终耗尽 VRAM 以进行写入。

出乎意料的幸运，我想；我没有计划这样的后果，它只是由于代码的大小和结构而发生的。不管怎样，我都不会抱怨。

### 结论

所以，我们有它。最小的。nds 文件，它仍然可以做一些事情。ARM7 坚持自己进入无限循环，ARM9 在进入自己的无限循环之前用红色填充主核帧缓冲区。我最终实现了我的愿望，一个小的帧缓冲测试演示，但这很有趣。

#### 最终二进制:352 字节

```
0000   4E 44 53 2E   54 69 6E 79   46 42 00 00   23 23 23 23  NDS.TinyFB..####
0010   00 00 00 00   00 00 FE FF   FF EA 00 00   00 00 00 04  ................
0020   70 00 00 00   00 00 00 02   00 00 00 02   44 00 00 00  p...........D...
0030   16 00 00 00   00 00 80 03   00 00 80 03   04 00 00 00  ................
0040   A0 01 00 00   00 00 00 00   A0 01 00 00   00 00 00 00  ................
0050   00 00 00 00   00 00 00 00   00 00 00 00   00 00 00 00  ................
0060   00 60 58 00   F8 08 18 00   00 00 00 00   00 00 00 00  .`X.............
0070   01 03 A0 E3   03 10 A0 E3   02 28 A0 E3   80 30 A0 E3  .........(...0..
0080   00 00 00 00   A0 01 00 00   04 13 80 E5   00 20 80 E5  ............. ..
0090   40 32 80 E5   1A 05 A0 E3   1F 10 A0 E3   03 29 A0 E3  @2...........)..
00A0   B2 10 C0 E0   01 20 52 E2   FC FF FF 1A   50 41 53 53  ..... R.....PASS
00B0   FE FF FF EA   00 00 00 00   00 00 00 00   00 00 00 00  ................

00C0   C8 60 4F E2   01 70 8F E2   17 FF 2F E1   12 4F 11 48  .`O..p..../..O.H
00D0   12 4C 20 60   64 60 7C 62   30 1C 39 1C   10 4A 00 F0  .L `d`|b0.9..J..
00E0   14 F8 30 6A   80 19 B1 6A   F2 6A 00 F0   0B F8 30 6B  ..0j...j.j....0k
00F0   80 19 B1 6B   F2 6B 00 F0   08 F8 70 6A   77 6B 07 4C  ...k.k....pjwk.L
0100   60 60 38 47   07 4B D2 18   9A 43 07 4B   92 08 D2 18  ``8G.K...C.K....
0110   0C DF F7 46   04 F0 1F E5   00 FE 7F 02   F0 FF 7F 02  ...F............
0120   F0 01 00 00   FF 01 00 00   00 00 00 04   00 00 00 00  ................
0130   00 00 00 00   00 00 00 00   00 00 00 00   00 00 00 00  ................
0140   00 00 00 00   00 00 00 00   00 00 00 00   00 00 00 00  ................
0150   00 00 00 00   00 00 00 00   00 00 00 00   1A 9E 7B EB  ..............{.

```

[http://imrannazar.com/content/files/TinyFB.nds](http://web.archive.org/web/20220810161352/http://imrannazar.com/content/files/TinyFB.nds)

感谢利拉努纳和百事曼

文章日期:2006 年 9 月 22 日*