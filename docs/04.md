# 让我们建立一个 JPEG 解码器:文件结构

> 原文：<http://web.archive.org/web/20220810161336/https://imrannazar.com/Let>

本文是探索 JPEG 解码的概念和实现的系列文章的第二部分；目前有四个部分可用，其他部分预计将随后推出。

*   [第一部分:概念](http://web.archive.org/web/20220810161352/http://imrannazar.com/Let%27s-Build-a-JPEG-Decoder%3A-Concepts)
*   [第二部分:文件结构](http://web.archive.org/web/20220810161352/http://imrannazar.com/Let%27s-Build-a-JPEG-Decoder%3A-File-Structure)
*   第三部分:霍夫曼表
*   [第四部分:帧和比特流](http://web.archive.org/web/20220810161352/http://imrannazar.com/Let%27s-Build-a-JPEG-Decoder%3A-Frames-and-Bitstreams)

*下载代码:[http://imrannazar.com/content/files/jpegparse.zip](http://web.archive.org/web/20220810161352/http://imrannazar.com/content/files/jpegparse.zip)*

在前一部分中，我简要介绍了 JPEG 压缩图像的技术。在研究这些技术的详细实现之前，查看 JPEG 文件的整体结构是很有用的，原因有两个:

*   编码器使用的一些过程采用值表，这些值表与图像信息本身一起存储，因此在需要它们之前将它们检索到内存中是明智的。
*   我们还需要一个地方来放置图像解压缩算法的实现，并且有一个合适的框架来促进这一点。

本系列文章中开发的实现将用 C++编写，但是这些结构可以移植到您选择的语言中，而不会增加额外的复杂性。

### JPEG 图像的类型

这里应该说明的是，这里开发的实现只适用于所有可能的 JPEG 图像类型的一个公共子集。首先，该标准支持四种类型的压缩:

*   **基线**:最常见的压缩类型，所有的图像信息都包含在一系列 8×8 的块中。
*   **扩展序列**:最常用于医学成像，这种类型允许每个像素有更多的级别。
*   **Progressive** :来自频域的信息被写入一系列扫描中，每个块的最重要值首先出现在文件中。这允许以低分辨率渲染整个图像，并在图像下载时填充细节。
*   **无损**:一种基于目标像素与其周围环境之间预测差异的稀有编码。

此外，有两种形式的编码应用于图像压缩之上，以进一步压缩文件数据:

*   **基于霍夫曼的熵**:图像被制成比特流，最常见的值被编码为短(2 或 3 比特)流条目，不太常见的值被记录为更长的比特串。
*   **算术**:一种以前获得专利的编码方法，将图像数据表示为一系列值出现的概率，并组合成一个分数。

这个系列将实现一个**熵编码的基线** JPEG 解码器。

### 文件段

JPEG 文件由不同长度的段组成，每个段都以一个“标记”开始，以表示它是哪种段。有 254 种可能的片段类型，但在我们将要解码的图像类型中只找到了少数几种:

| 名字 | 简称 | 标记 | 描述 | 长度(字节) |
| --- | --- | --- | --- | --- |
| 图像开始 | 通信操作指令 | FF D8 | 分隔文件的开始 | Two |
| 定义量化表 | DQT | FF DB | 解码器使用的值 | sixty-nine |
| 定义霍夫曼表 | DHT | C4 法郎 | 解压缩器使用的值 | 可变的 |
| 帧的开始 | 影片录音(sound on film) | C0 法郎 | 熵编码的基线帧的信息 | Ten |
| 扫描开始 | 无线电紧急呼救信号 | FF DA | 编码和压缩的图像比特流 | 可变的 |
| 图像结束 | EOI | FF D9 | 分隔文件的结尾 | Two |

*Table 1: Segments present in an entropy-coded baseline JPEG file*

大多数不同类型的数据段在标记后面都有一个“长度”值，它表示数据段的长度(包括长度值)；这可以用来跳过解码器不知道的段。这条一般规则有三个例外:

*   **SOI** 和 **EOI** :因为这些更多的是定界符而不是标记，所以它们只包含标记值。
*   **SOS** :扫描的是比特流，图像完全编码后自动“结束”。因此，SOS 段没有写入文件的长度。有两种策略来处理这个问题:或者我们可以假设文件的其余部分是扫描的一部分，或者我们可以通读文件，寻找表示新段开始的标记。

对于本文，如果我们遇到 SOS 段，我将假设文件的其余部分是扫描的一部分，并直接跳到 EOI。

### 实现:在 JPEG 文件中列出片段

作为第一步，编写一个程序来打开一个 JPEG 文件，并运行它来寻找段标记是有意义的。这种程序的结构可以用处理不同种类的段的实现来扩展，并且在给定其大小的情况下跳过段的机制可以在以后用于跳过文件中对解码过程不重要的部分。

由于 JPEG 文件中值的大小是以字节数的绝对值来指定的，因此将基本的整数类型抽象成与大小相关的类型是一个好主意。为此，我们将使用一个短的头文件。

#### inttypes.h:独立于架构的整数大小定义

```
#ifndef __INTTYPES_H_
#define __INTTYPES_H_

typedef unsigned char  u8;
typedef unsigned short u16;
typedef unsigned int   u32;

typedef signed char  s8;
typedef signed short s16;
typedef signed int   s32;

#endif//__INTTYPES_H_
```

上面的文件是为 32 位编译设置的，但是如果需要 64 位或 16 位代码也可以修改。这样做的好处是，在 JPEG 解码器实现中引用整数本身可以是架构不可知的，只需引用`u16`和这里定义的其他类型。

有了这些抽象，段列表的实现就非常简单了。因为我们将把解码功能构建到一个类中，所以此时定义类本身是值得的。

#### jpeg.h: JPEG 解码器类定义

```
#ifndef __JPEG_H_
#define __JPEG_H_

#include "inttypes.h"
#include <string>
#include <vector>
#include <map>
#include <stdio.h>

// Macro to read a 16-bit word from file
#define READ_WORD() ((fgetc(fp) << 8) | fgetc(fp))

// Segment parsing error codes
#define JPEG_SEG_ERR  0
#define JPEG_SEG_OK   1
#define JPEG_SEG_EOF -1

class JPEG {
    private:
        // Names of the possible segments
        std::string segNames[64];

        // The file to be read from, opened by constructor
        FILE *fp;

        // Segment parsing dispatcher
        int parseSeg();

    public:
        // Construct a JPEG object given a filename
        JPEG(std::string);
};

#endif//__JPEG_H_
```

#### jpeg.cpp: JPEG 段列表实现

```
#include "jpeg.h"
#include <stdlib.h>
#include <string.h>
#include <math.h>

//-------------------------------------------------------------------------
// Function: Parse JPEG file segment (parseSeg)
// Purpose: Retrieves 16-bit block ID from file, shows name

int JPEG::parseSeg()
{
    if (!fp) {
        printf("File failed to open.\n");
        return JPEG_SEG_ERR;
    }

    u32 fpos = ftell(fp);
    u16 id = READ_WORD(), size;
    if (id < 0xFFC0)
    {   
        printf("Segment ID expected, not found.\n");
        return JPEG_SEG_ERR;
    }

    printf(
        "Found segment at file position %d: %s\n",
        fpos, segNames[id-0xFFC0].c_str());

    switch (id) {
        // The SOI and EOI segments are the only ones not to have
        // a length, and are always a fixed two bytes long; do
        // nothing to advance the file position
        case 0xFFD9:
            return JPEG_SEG_EOF;
        case 0xFFD8:
            break;

        // An SOS segment has a length determined only by the
        // length of the bitstream; for now, assume it's the rest
        // of the file less the two-byte EOI segment
        case 0xFFDA:
            fseek(fp, -2, SEEK_END);
            break;

        // Any other segment has a length specified at its start,
        // so skip over that many bytes of file
        default:
            size = READ_WORD();
            fseek(fp, size-2, SEEK_CUR);
            break;
    }

    return JPEG_SEG_OK;
}

//-------------------------------------------------------------------------
// Function: Array initialisation (constructor)
// Purpose: Fill in arrays used by the decoder, decode a file
// Parameters: filename (string) - File to decode

JPEG::JPEG(std::string filename)
{   
    // Debug messages used by parseSeg to tell us which segment we're at
    segNames[0x00] = std::string("Baseline DCT; Huffman");
    segNames[0x01] = std::string("Extended sequential DCT; Huffman");
    segNames[0x02] = std::string("Progressive DCT; Huffman");
    segNames[0x03] = std::string("Spatial lossless; Huffman");
    segNames[0x04] = std::string("Huffman table");
    segNames[0x05] = std::string("Differential sequential DCT; Huffman");
    segNames[0x06] = std::string("Differential progressive DCT; Huffman");
    segNames[0x07] = std::string("Differential spatial; Huffman");
    segNames[0x08] = std::string("[Reserved: JPEG extension]");
    segNames[0x09] = std::string("Extended sequential DCT; Arithmetic");
    segNames[0x0A] = std::string("Progressive DCT; Arithmetic");
    segNames[0x0B] = std::string("Spatial lossless; Arithmetic");
    segNames[0x0C] = std::string("Arithmetic coding conditioning");
    segNames[0x0D] = std::string("Differential sequential DCT; Arithmetic");
    segNames[0x0E] = std::string("Differential progressive DCT; Arithmetic");
    segNames[0x0F] = std::string("Differential spatial; Arithmetic");
    segNames[0x10] = std::string("Restart");
    segNames[0x11] = std::string("Restart");
    segNames[0x12] = std::string("Restart");
    segNames[0x13] = std::string("Restart");
    segNames[0x14] = std::string("Restart");
    segNames[0x15] = std::string("Restart");
    segNames[0x16] = std::string("Restart");
    segNames[0x17] = std::string("Restart");
    segNames[0x18] = std::string("Start of image");
    segNames[0x19] = std::string("End of image");
    segNames[0x1A] = std::string("Start of scan");
    segNames[0x1B] = std::string("Quantisation table");
    segNames[0x1C] = std::string("Number of lines");
    segNames[0x1D] = std::string("Restart interval");
    segNames[0x1E] = std::string("Hierarchical progression");
    segNames[0x1F] = std::string("Expand reference components");
    segNames[0x20] = std::string("JFIF header");
    segNames[0x21] = std::string("[Reserved: application extension]");
    segNames[0x22] = std::string("[Reserved: application extension]");
    segNames[0x23] = std::string("[Reserved: application extension]");
    segNames[0x24] = std::string("[Reserved: application extension]");
    segNames[0x25] = std::string("[Reserved: application extension]");
    segNames[0x26] = std::string("[Reserved: application extension]");
    segNames[0x27] = std::string("[Reserved: application extension]");
    segNames[0x28] = std::string("[Reserved: application extension]");
    segNames[0x29] = std::string("[Reserved: application extension]");
    segNames[0x2A] = std::string("[Reserved: application extension]");
    segNames[0x2B] = std::string("[Reserved: application extension]");
    segNames[0x2C] = std::string("[Reserved: application extension]");
    segNames[0x2D] = std::string("[Reserved: application extension]");
    segNames[0x2E] = std::string("[Reserved: application extension]");
    segNames[0x2F] = std::string("[Reserved: application extension]");
    segNames[0x30] = std::string("[Reserved: JPEG extension]");
    segNames[0x31] = std::string("[Reserved: JPEG extension]");
    segNames[0x32] = std::string("[Reserved: JPEG extension]");
    segNames[0x33] = std::string("[Reserved: JPEG extension]");
    segNames[0x34] = std::string("[Reserved: JPEG extension]");
    segNames[0x35] = std::string("[Reserved: JPEG extension]");
    segNames[0x36] = std::string("[Reserved: JPEG extension]");
    segNames[0x37] = std::string("[Reserved: JPEG extension]");
    segNames[0x38] = std::string("[Reserved: JPEG extension]");
    segNames[0x39] = std::string("[Reserved: JPEG extension]");
    segNames[0x3A] = std::string("[Reserved: JPEG extension]");
    segNames[0x3B] = std::string("[Reserved: JPEG extension]");
    segNames[0x3C] = std::string("[Reserved: JPEG extension]");
    segNames[0x3D] = std::string("[Reserved: JPEG extension]");
    segNames[0x3E] = std::string("Comment");
    segNames[0x3F] = std::string("[Invalid]");

    // Open the requested file, keep parsing blocks until we run
    // out of file, then close it.
    fp = fopen(filename.c_str(), "rb");
    if (fp) {
        while(parseSeg() == JPEG_SEG_OK);
        fclose(fp);
    }
    else {
        perror("JPEG");
    }
}
```

当用文件构造时，这个 JPEG 类的对象将提供类似如下的输出。

#### JPEG 段列表的输出

```
Found segment at file position 0: Start of image
Found segment at file position 2: JFIF header
Found segment at file position 20: Quantisation table
Found segment at file position 89: Quantisation table
Found segment at file position 158: Baseline DCT; Huffman
Found segment at file position 177: Huffman table
Found segment at file position 208: Huffman table
Found segment at file position 289: Huffman table
Found segment at file position 318: Huffman table
Found segment at file position 371: Start of scan
Found segment at file position 32675: End of image
```

### 下次:检查扫描的编码

从上面可以看出,“扫描”构成了熵编码的基线 JPEG 的大部分；由于整个图像数据在扫描中被编码，这是有意义的。熵编码是基于 Huffman 压缩算法的，所以在下一篇文章中，我将研究 JPEG 文件中提供所需信息的部分，这些信息用于将比特流中的扫描解码为可用于进一步处理的内容。

2013 年 1 月，伊姆兰·纳扎尔<>。

*文章日期:2013 年 1 月 20 日*