# 让我们建立一个 JPEG 解码器:帧和比特流

> 原文：<http://web.archive.org/web/20220810161336/https://imrannazar.com/Let>

本文是探索 JPEG 解码的概念和实现的系列文章的第四部分；目前有四个部分可用，其他部分预计将随后推出。

*   [第一部分:概念](http://web.archive.org/web/20220810161352/http://imrannazar.com/Let%27s-Build-a-JPEG-Decoder%3A-Concepts)
*   [第二部分:文件结构](http://web.archive.org/web/20220810161352/http://imrannazar.com/Let%27s-Build-a-JPEG-Decoder%3A-File-Structure)
*   第三部分:霍夫曼表
*   [第四部分:帧和比特流](http://web.archive.org/web/20220810161352/http://imrannazar.com/Let%27s-Build-a-JPEG-Decoder%3A-Frames-and-Bitstreams)

*下载代码:[http://imrannazar.com/content/files/jpegparse.zip](http://web.archive.org/web/20220810161352/http://imrannazar.com/content/files/jpegparse.zip)*

之前，我讨论了由 JPEG 实现的 Huffman 压缩算法，以及 JPEG 编码器选择用于压缩的替换值的机制。在一个基本的熵编码的 JPEG 文件中，图像本身在一次“扫描”中被存储为霍夫曼代码流；代码长度可变，不一定在偶数字节边界。

出于这个原因，需要在文件的字节和 JPEG 解码器看到的值之间引入某种队列:这允许位从文件的字节被推到队列上，并由解码器以不同的量取出。因此，单个例程可以成为任何可变宽度符号请求的中心路由点。

当以这种方式使用时，事物队列通常有两种操作模式:或者队列中有足够的事物供请求处理，或者在队列中有足够的事物处理请求之前必须进行额外的读取。在从文件读取的位级队列的情况下，正如我们在这里需要的，请求的位数是决定因素。在下面的第一个示例中，一个包含五位的队列被要求返回前三位，并且能够毫无问题地处理请求。

![Request handling with a full queue](img/c7b15f2ff9da912b94abf4d8191a34cc.png) *Figure 1: Request handling with sufficient content in the queue*

如果有一个 4 位的请求进来，队列没有足够的位来处理这个请求，必须首先从 JPEG 文件中获取另一个完整的字节(这里用红色显示)。

![Request handling with an empty queue](img/5c4bfba0827acb30c72404d6df9b14a0.png) *Figure 2: Request handling when the queue is empty*

### 实现位队列

当试图对如上所示的位队列的实现进行编码时，出现了一个复杂性:队列的输出端在左边，如果一个连续值用于存储队列，则左边是具有最高值位的最高有效端。如果我们将队列划分为 32 位长，并将一个 32 位值定义为保存其全部内容，则上述场景分解如下:

Pulling three bits

*   该请求短于队列长度 5，不进行文件读取。
*   输出值是队列的前三位，下移(32 - 3)位成为输出值的后三位；
*   队列左移三位，左移长度为 2。

Pulling another four bits

*   请求长于队列长度 2:读入一个字节，并向上移位(24 - 2)位以加入队列；
*   队列现在是 10 位长，没有更多的字节需要读取；
*   输出值是队列的前四位，向下移位(32-4)；
*   队列左移 4 位，长度为 6。

在队列操作期间，下一个可用位始终位于连续队列变量的高值端。它的实现可能如下所示。

#### jpeg.h:位分辨率文件阅读器的定义

```
class JPEG {
    private:
        // Read a number of bits from file
        u16 readBits(int);
};
```

#### jpeg.cpp:位分辨率文件阅读器

```
u16 JPEG::readBits(int len)
{
    // The number of bits left in the queue, and the value represented
    static int queueLen = 0;
    static u32 queueVal = 0;

    u8 readByte;
    u16 output;

    if (len > queueLen) {
        do {
            // Read a byte in, shift it up to join the queue
            readByte = fgetc(fp);
            queueVal = queueVal | (readByte << (24 - queueLen));
            queueLen += 8;
        } while (len > queueLen);
    }

    // Shift the requested number of bytes down to the other end
    output = ((queueVal >> (32 - len)) & ((1 << len) - 1));

    queueLen -= len;
    queueVal <<= len;

    return output;
}
```

### JPEG SOF 片段

正如在本系列的前一部分中提到的，一个 JPEG 文件可以定义多达 32 个 Huffman 码表，每个都在它们自己的`DHT`段中。JPEG 文件将对应于图像本身的数据保存在“帧”中，由“帧开始”段头表示。SOF 报头包含解码帧所需的部分信息，其结构如下表所示。

| 田 | 价值 | 大小(字节) |
| --- | --- | --- |
| 精度(JPEG 块中的像素数) | eight | one |
| 图像高度 | 高达 65535 | Two |
| 图像宽度 | 高达 65535 | Two |
| 成分 | 颜色成分的数量 | one |
| 对于每个组件(在 YUV 颜色文件中，三个) |
| 身份证明 | 供以后使用的标识符 | one |
| 采样分辨率 | 供以后检查 | one |
| 量化表 | 供以后检查 | one |

*Table 1: Structure of the SOF header*

从上表中可以看出，有些字段涉及我们尚未检查的操作(每个分量的采样分辨率和量化表)。为了完成 s of 段处理程序，我们可以保留这些信息供以后使用。

一个“帧”由 s of 报头和多个“扫描”组成；顾名思义，每次扫描都是对图像的整个矩形进行扫描。例如，在隔行扫描的 JPEG 文件中，单个帧会出现多次扫描，每次扫描的分辨率都比上一次高；在渐进式 JPEG 文件中，只有一次扫描包含图像的所有信息。由于本系列关注的是为渐进式 JPEG 文件构建一个解码器，所以我们将着重于处理帧中的单次扫描。

原来，霍夫曼解码所需的信息，特别是使用哪一个`DHT`表，是由帧中的每次扫描而不是帧本身定义的。我们将在本系列的下一部分更详细地查看扫描级信息；现在，将图像中颜色组件的表示从`SOF`数据中分离出来就足够了，稍后再定义组件的确切元数据。

### 结构定义

有了详细描述`SOF`头结构的信息，构建一个插入现有代码的段解析器就变得相对简单了。唯一复杂的是 JPEG 中的多字节值是以 big-endian 格式存储的，这种格式不一定是大整数的宿主格式。拥有一组透明地处理大端值的定义是很有用的，如下所示。

#### byteswap.h:大端值处理宏

```
/**
* Let's Build a JPEG Decoder
* Big-endian value handling macros
* Imran Nazar, May 2013
*/

#ifndef __BYTESWAP_H_
#define __BYTESWAP_H_

#if __SYS_BIG_ENDIAN == 1
#   define htoms(x) (x)
#   define htoml(x) (x)
#   define mtohs(x) (x)
#   define mtohl(x) (x)
#else
#   define htoms(x) (((x)>>8)|((x)<<8))
#   define htoml(x) (((x)<<24)|(((x)&0xFF00)<<8)|(((x)&0xFF0000)>>8)|((x)>>24))
#   define mtohs(x) (((x)>>8)|((x)<<8))
#   define mtohl(x) (((x)<<24)|(((x)&0xFF00)<<8)|(((x)&0xFF0000)>>8)|((x)>>24))
#endif

#endif//__BYTESWAP_H_
```

#### jpeg.h: SOF 头结构

```
// Prevent padding bytes from creeping into structures
#define PACKED __attribute__((packed))

class JPEG {
    private:
        // Information in the SOF header
        struct PACKED {
            u8  precision;
            u16 height;
            u16 width;
            u8  component_count;
        } sofHead;

        typedef struct PACKED {
            u8 id;
            u8 sampling;
            u8 q_table;
        } sofComponent;

        // Internal information about a colour component
        typedef struct PACKED {
            u8 id;
            // There is likely to be more data here...
        } Component;

        // The set of colour components in the image
        std::vector components;

        // The SOF segment handler
        int SOF();
};
```

#### jpeg.cpp:将控制传递给段处理程序

```
int JPEG::parseSeg()
{
    ...

    switch (id) {
        // The SOF segment defines the components and resolution
        // of the JPEG frame for a baseline Huffman-coded image
        case 0xFFC0:
            size = READ_WORD() - 2;
            if (SOF() != size) {
                printf("Unexpected end of SOF segment\n");
                return JPEG_SEG_ERR;
            }
            break;

        ...
    }

    return JPEG_SEG_OK;
}
```

#### jpeg.cpp: SOF 段处理

```
int JPEG::SOF()
{
    int ctr = 0, i;

    fread(&sofHead, sizeof(sofHead), 1, fp);
    ctr += sizeof(sofHead);

    sofHead.width   = mtohs(sofHead.width);
    sofHead.height  = mtohs(sofHead.height);

    printf("Image resolution: %dx%d\n", sofHead.width, sofHead.height);

    for (i = 0; i < sofHead.component_count; i++) {
        sofComponent s;
        fread(&s, sizeof(sofComponent), 1, fp);
        ctr += sizeof(sofComponent);

        Component c;
        c.id = s.id;

        components.push_back(c);
    }

    return ctr;
}
```

### 下一次:最小编码单位

如上所述，渐进 JPEG 文件中的图像帧被编码为扫描，由一系列块组成；根据图像中组件的采样分辨率，这些块可能大于 JPEG 算法的 8×8 像素基本块大小。在本系列的下一部分，我将研究这些较大的单位和图像的颜色成分之间的关系。

2013 年 5 月，*伊姆兰·纳扎尔<tf@imrannazar.comT5。*

*文章日期:2013 年 5 月 5 日*