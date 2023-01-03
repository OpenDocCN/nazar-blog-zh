# 让我们建立一个 JPEG 解码器:霍夫曼表

> 原文：<http://web.archive.org/web/20220810161336/https://imrannazar.com/Let>

本文是探索 JPEG 解码的概念和实现的系列文章的第三部分；目前有四个部分可用，其他部分预计将随后推出。

*   [第一部分:概念](http://web.archive.org/web/20220810161352/http://imrannazar.com/Let%27s-Build-a-JPEG-Decoder%3A-Concepts)
*   [第二部分:文件结构](http://web.archive.org/web/20220810161352/http://imrannazar.com/Let%27s-Build-a-JPEG-Decoder%3A-File-Structure)
*   第三部分:霍夫曼表
*   [第四部分:帧和比特流](http://web.archive.org/web/20220810161352/http://imrannazar.com/Let%27s-Build-a-JPEG-Decoder%3A-Frames-and-Bitstreams)

*下载代码:[http://imrannazar.com/content/files/jpegparse.zip](http://web.archive.org/web/20220810161352/http://imrannazar.com/content/files/jpegparse.zip)*

在上一部分中，我提到大多数 JPEG 文件在图像压缩的基础上采用了一种编码技术，试图从图像中删除任何冗余信息的痕迹。最常见的 JPEG 编码所使用的技术是对数据压缩世界中常见的一种技术的改编，称为霍夫曼编码，因此详细研究霍夫曼解码器的结构和实现是很有用的。

因为 Huffman 编码是 JPEG 编码器在保存图像文件时执行的最后一件事情，所以它需要是我们的解码器首先要做的事情。这可以通过两种方式实现:

*   **重建**:全图像扫描从霍夫曼编码状态解码成临时占位符，并对临时副本进行进一步处理。
*   **即时**:编码图像扫描一次处理一个代码，当有足够的信息可用时，处理每个 JPEG 块。

本文将采用第二种方法，以节省内存和牺牲时间；完全重构可以使用下面构建的代码以非常相似的方式实现。

### 霍夫曼算法

霍夫曼编码和其他基于熵的方案背后的概念类似于替代密码背后的概念:输入中的每个唯一字符都被转换成唯一的输出字符。最简单的例子是凯撒替换，它可以用表格形式表示如下:

```
A => D
B => E
C => F
...
Y => B
Z => C

This is an example of a Caesar cipher
Wklv lv dq hadpsoh ri d Fdhvdu flskhu
```

通过记录输入中字符的相对频率，并设计一个包含较短代码的表来替代这些字符，而不是较少的字符，可以对标准替代密码进行改进。看一下上例中字母的频率，包括它们的 ASCII 表示，我们可以生成一个递增的唯一代码表，如下所示:

| 性格；角色；字母 | 美国信息交换标准代码 | 频率 | 密码 |
| --- | --- | --- | --- |
| 空间 | 00100000 | seven | 00 |
| a | 01100001 | five | 01 |
| e | 01100101 | four | One hundred |
| 我 | 01101001 | three | One thousand and ten |
| s | 01110011 | three | One thousand and eleven |
| h | 01101000 | Two | Eleven thousand |
| p | 01110000 | Two | Eleven thousand and one |
| r | 01110010 | Two | Eleven thousand and ten |
| C | 01000011 | one | One hundred and ten thousand one hundred and ten |
| T | 01010100 | one | One hundred and ten thousand one hundred and eleven |
| c | 01100011 | one | One hundred and eleven thousand |
| f | 01100110 | one | One hundred and eleven thousand and one |
| l | 01101100 | one | One hundred and eleven thousand and ten |
| m | 01101101 | one | One hundred and eleven thousand and eleven |
| n | 01101110 | one | One hundred and eleven thousand one hundred |
| o | 01101111 | one | One hundred and eleven thousand one hundred and one |
| x | 01110111 | one | One hundred and eleven thousand one hundred and ten |

*Table 1: Frequency of characters in the string "This is an example of a Caesar cipher"*

用这些代码代替原文中的字符，可以看出编码后的数据是如何比原来小得多的。

```
This is an example of a Caesar cipher
01010100 01101000 01101001 01110011 00100000 01101001 01110011 00100000
01100001 01101110 00100000 01100101 01110111 01100001 01101101 01110000
01101100 01100101 00100000 01101111 01100110 00100000 01100001 00100000
01000011 01100001 01100101 01110011 01100001 01110010 00100000 01100011
01101001 01110000 01101000 01100101 01110010

110111 11000 1010 1011 00 1010 1011 00 01 111100 00 100 111110 01 111011
11011 111010 100 00 111101 111001 00 01 00 110110 01 100 1011 01 11010
00 111000 1010 11001 11000 100 11010

54 68 69 73 20 69 73 20 61 6E 20 65 77 61 6D 70 6C 65 20 6F 66 20 61 20
43 61 65 73 61 72 20 63 69 70 68 65 72
DF 15 65 58 F8 4F 9E F7 D4 3D E4 4D 99 6E 8E 2B 38 9A
```

Huffman 编码方法的主要缺点是代码表需要与压缩数据一起存储:在上面的例子中，如果没有相应的频率表，编码字节的红色字符串将是没有意义的。代码表及其对应的字符可以完整记录，但是如果注意代码出现的模式，有一种更节省空间的方法来保存代码。这里要注意两件事:首先，代码的长度增加，而且在一组相同长度的代码中，代码是连续的。这意味着代码表可以写成:

```
2 codes of length two  , starting at 00
1 code  of length three, starting at 100
2 codes of length four , starting at 1010
3 codes of length five , starting at 11000
9 codes of length six  , starting at 110110
```

仔细观察代码本身可以进一步改进记录编码表所需的空间。如果我们结合上面的代码长度列表来看代码，我们可以按如下方式开始计数。

```
00 (zero)
01 (one)
Next code would be 10 (two)
100 (four)
Next code would be 101 (five)
1010 (ten)
1011 (eleven)
Next code would be 1100 (twelve)
11000 (twenty four)
11001 (twenty five)
11010 (twenty six)
Next code would be 11011 (twenty seven)
110110 (fifty four)
110111 (fifty five)
111000 (fifty six)
111001 (fifty seven)
111010 (fifty eight)
111011 (fifty nine)
111100 (sixty)
111101 (sixty one)
111110 (sixty two)
```

在每种情况下，当对于给定的代码长度已经计数了必要的代码数量时，所需要的是将计数器加倍，并且继续下一个代码长度。换句话说，没有必要记录上面代码长度列表中的“起始于”部分，因为它可以从零开始推断。因此，最终的代码列表如下所示。

```
2 codes of length two
1 code  of length three
2 codes of length four
3 codes of length five
9 codes of length six

The above codes correspond to the following characters, in this order:
Space,a,e,i,s,h,p,r,C,T,c,f,l,m,n,o,x
```

### JPEG DHT 段

JPEG 文件的霍夫曼表是以这样的方式记录的:一个给定长度(1 到 16)的代码列表，后面是代码的含义。该信息保存在文件的“定义霍夫曼表”(DHT)段中，根据 JPEG 标准，该段最多可有 32 个。

如上所述，由霍夫曼算法编码的数据最终被记录为一系列代码，楔入比特流中；这也适用于 JPEG 文件中的图像扫描。从比特流中读取代码的简单例程可能如下所示:

#### 用于读取霍夫曼编码值的伪代码

```
Code = 0
Length = 0
Found = False

Do
    Code = Code << 1
    Code = Code | (The next bit in the stream)
    Length = Length + 1
    If ((Length, Code) is in the Huffman list) Then
        Found = True
    End If
While Found = False
```

为了便于这种算法，霍夫曼代码应该以一种允许我们确定一个代码是否在给定长度的映射中的方式存储。表示霍夫曼代码列表的规范方式是二叉树，其中分支的序列定义代码，树的深度告诉我们代码有多长。C++ STL 为我们将它抽象成`map`结构。

由于在一个 JPEG 文件中可以定义多达 32 个可能的霍夫曼表，我们的实现将需要 32 个`map`可用。在这一点上，定义 DHT 段处理程序如何被本系列前一部分开发的`parseSeg`方法调用也是值得的。

#### jpeg.h: DHT 数据定义

```
class JPEG {
    private:
        // Defines a tuple of length and code, for use in the Huffman maps
        typedef std::pair<int, u16> huffKey;

        // The array of Huffman maps: (length, code) -> value
        std::map<huffKey, u8> huffData[32];

        // DHT segment handler
        int DHT();
};
```

#### jpeg.cpp:将控制传递给段处理程序

```
int JPEG::parseSeg()
{
    ...

    switch (id) {
        // The DHT segment defines a Huffman table. The handler should
        // read exactly as many bytes from the file as are in the
        // segment; if not, something's gone wrong
        case 0xFFC4:
            size = READ_WORD() - 2;
            if (DHT() != size) {
                printf("Unexpected end of DHT segment\n");
                return JPEG_SEG_ERR;
            }
            break;

        ...
    }

    return JPEG_SEG_OK;
}
```

#### DHT 段处理程序，构建一个霍夫曼图

```
int JPEG::DHT()
{
    int i, j;

    // A counter of how many bytes have been read
    int ctr = 0;

    // The incrementing code to be used to build the map
    u16 code = 0;

    // First byte of a DHT segment is the table ID, between 0 and 31
    u8 table = fgetc(fp);
    ctr++;

    // Next sixteen bytes are the counts for each code length
    u8 counts[16];
    for (i = 0; i < 16; i++) {
        counts[i] = fgetc(fp);
        ctr++;
    }

    // Remaining bytes are the data values to be mapped
    // Build the Huffman map of (length, code) -> value
    for (i = 0; i < 16; i++) {
        for (j = 0; j < counts[i]; j++) {
            huffData[table][huffKey(i + 1, code)] = fgetc(fp);
            code++;
            ctr++;
        }
        code <<= 1;
    }

    // Once the map has been built, print it out
    printf("Huffman table #%02X:\n", table);

    std::map<huffKey, u8>::iterator iter;
    for (iter = huffData[table].begin();
         iter != huffData[table].end();
         iter++
    ) {
        printf("    %04X at length %d = %02X\n",
            iter->first.second, iter->first.first, iter->second);
    }

    return ctr;
}
```

与前一部分一样，JPEG 类可以用文件名实例化；如果这样做了，上面的代码将产生如下输出:

```
Found segment at file position 177: Huffman table
Huffman table #00:
    0000 at length 2 = 04
    0002 at length 3 = 02
    0003 at length 3 = 03
    0004 at length 3 = 05
    0005 at length 3 = 06
    0006 at length 3 = 07
    000E at length 4 = 01
    001E at length 5 = 00
    003E at length 6 = 08
    007E at length 7 = 09
```

### 下一次:读取比特流

一旦为 JPEG 文件构建了霍夫曼图，就可以对图像扫描进行解码，以便进一步处理。在下一部分中，我将在从图像中读取块并检查它们被转换的过程的更广泛的上下文中，看一看扫描的霍夫曼解码。

2013 年 2 月，伊姆兰·纳扎尔<>。

*文章日期:2013 年 2 月 24 日*