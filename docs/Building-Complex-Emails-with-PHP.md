# 用 PHP 构建复杂的电子邮件

> 原文：<http://web.archive.org/web/20220810161336/https://imrannazar.com/Building-Complex-Emails-with-PHP>

PHP 新手首先要学习的事情之一是如何发送一封简单的电子邮件:使用`mail`函数将几段文本发送到一个电子邮件地址。这是发送状态信息和其他小邮件的一种简单实用的方式，但也有缺点:

Unformatted text:

With plain text emails, only the most rudimentary structure can be given to a message; there's no inherent way to insert a heading or a bullet-point list. If HTML were allowed in the email, for example, this formatting could be provided to the message.

No attachments:

Because the email is a single plain-text message, there is no way to provide additional documents or other files in the body of the message. A compromise is to place the files in question on the public Web, and provide links to the files, but this also compromises the security of the documents.

这些问题不仅仅是 PHP 邮件例程的局限性:它们也是电子邮件传输机制的局限性。为了绕过他们，一个狡猾的计划在 20 世纪 90 年代被标准化了。

### 多用途互联网邮件扩展

MIME 标准被设计成在现有的电子邮件传输系统中工作；因此，它不需要任何特殊的连接方法，开发者也不需要复杂的网络。相反，MIME 通过将所有部分插入到一封纯文本电子邮件中，并用边界分隔各个部分，来允许多部分消息。

#### 结构:基本边界

```
Text outside the boundary (part #0)
--BOUNDARY
Part #1
--BOUNDARY
Part #2
--BOUNDARY--
```

从上面的例子中可以看出，两个连字符位于边界的所有实例之前，一个边界构成了一个部分的结束和下一个部分的开始。最后一部分的结尾用两个连字符表示，在关闭该部分的边界之后。

上面概述的基本结构允许部分分离，但是它所能提供的只是将多个纯文本消息合并成一个。为了允许编码更复杂的信息，必须提供与每个部分相关联的头。

### MIME Headers

边界结构出现了一个问题:电子邮件阅读客户端如何知道哪些行表示部分的边界，哪些只是消息的一部分？通过提供消息的总标题，可以通知客户端正在使用哪个边界。报头通常用于表示消息的始发者、发送服务器的软件版本以及其他可能与客户端相关的信息；MIME 边界可以添加到这个。

#### 标题:MIME 邮件边界

```
From: "Imran Nazar" <tf@oopsilon.com>
MIME-Version: 1.0
Content-type: multipart/mixed; boundary="BOUNDARY"
*<blank line>*
Message body
```

`Content-type`报头告诉电子邮件客户端在消息中提供哪种数据；`Content-type`后面的文本被称为 MIME 类型。MIME 类型的概念已经扩展到电子邮件之外，现在通常由 Web 和文件服务器响应数据请求而提供。

与数据块一起提供的 MIME 类型可用于标识所讨论的数据。有各种与 MIME 类型相关联的数据类，以及每个类的子定义。数据的类和子类以`major/minor`格式给出；下面提供了几个例子。

| 重要的 | 较小的 | 完整类型 | 数据 |
| --- | --- | --- | --- |
| **文本文档** |
| 文本 | 平原 | 文本/纯文本 | 纯文本文档 |
| 文本 | 超文本标记语言 | 文本/html | HTML 文档 |
| 文本 | 战斗支援车 | text/csv | 逗号分隔的数据文件 |
| **图像** |
| 图像 | 联合图象专家组 | 图像/jpeg | JPEG 格式的图像 |
| 图像 | png | 图片/png | PNG 格式的图像 |
| **特定应用类型** |
| 应用 | 可移植文档格式文件的扩展名（portable document format 的缩写） | 应用程序/pdf | 可移植文档格式(PDF) |
| 应用 | 活力 | 应用程序/zip | PKZIP 压缩档案 |
| 应用 | ms word | 应用程序/msword | MS Word 文档 |
| **多组分类型** |
| 几部分的 | 表单数据 | 多部分/表单数据 | 带有上传文件的 Web 表单 |
| 几部分的 | 混合的 | 多部分/混合 | 包含多种类型组件的消息 |

*表 1:示例 MIME 类型*

从上面可以看出，`multipart/mixed` MIME 类型告诉电子邮件读者消息的每个部分可以是不同的类型。就像消息一样，每个部分都可以有标题和正文。考虑到这一点，可以构建更完整的符合 MIME 的消息。

#### 带标题的多部分电子邮件

```
This is part #0.
--BOUNDARY
Content-type: text/html

This is part #1.
--BOUNDARY
Content-type: text/csv

id,content,date
"1","This is part #2.","2008-08-10"
--BOUNDARY--
```

### 附件和内容标题

我们已经了解了如何将多种类型的消息放入一封电子邮件中，但是这不足以将文档和其他文件附加到一封电子邮件中。将文档插入电子邮件有两个主要问题:

Naming:

As can be seen above, files can be inserted into an email as a MIME part, but they are not given a filename, and are not treated as attachments. This problem is solved by inserting another header along with the part's `Content-type`, called `Content-disposition`.

Encoding:

An email message has to be readable in its entirety by any mailserver that happens across it. Because mailservers may run in many places, under many languages and character sets, a binary data file is not guaranteed to make it to the destination intact: it has to be encoded into a more basic character set, and the email client has to be told how to decode the resultant email part. This is done with a third header, called `Content-transfer-encoding`.

附加到消息部分的`Content-disposition`可以是两种类型之一:`inline`，表示这种类型将作为电子邮件的一部分显示，以及`attachment`，表示附加供下载的文件。如果是一个`attachment`，一个`filename`可以作为参数提供给`Content-disposition`头。使用这个头文件，我们可以将上面例子中的 CSV 数据文件制作成一个附件:

#### 具有处置的多部分电子邮件

```
This is part #0.
--BOUNDARY
Content-type: text/html
Content-disposition: inline

This is part #1.
--BOUNDARY
Content-type: text/csv
Content-disposition: attachment; filename="data.csv"

id,content,date
"1","This is part #2.","2008-08-10"
--BOUNDARY--
```

这解决了将文件附加到电子邮件的第一个问题，但是第二个问题仍然存在:将附件编码成可传输的格式。MIME 标准允许两种主要的编码方法:

*   `quoted-printable`:区别编码，允许标准文本通过而不编码，但将非标准字符翻译成它们的十六进制序数值；
*   `base64`:无差别编码，将整个数据流作为一个数字，一次翻译一个组块，3 个字节翻译成 4 个字符的块。

`base64`编码通常更容易产生，因为`quoted-printable`编码需要专门的翻译表。使用`base64`，数据被分解成 48 字节的“行”，并在插入电子邮件之前编码成 64 个字符的行。

一旦选择了编码，就应该在消息部分的头中提供它，如下例所示。

#### 附加 base64 编码的二进制文件

```
Content-type: image/gif
Content-disposition: attachment; filename="text-icon.gif"
Content-transfer-encoding: base64

R0lGODlhIAAgAKIEAISEhMbGxgAAAP///////wAAAAAAAAAAACH5BAEAAAQALAAA
AAAgACAAAAOaSKoi08/BKeW6Cgyg+e7gJwICRmjOM6hs6q5kUF7o+rZ2vgkypq3A
oHA4kPVoxCTROFv8lNAir5mxNa7ESorpi0a5yMg15QU7vVBzFZ1Un9jtaVeMRbuf
8OA9P9zTx4CAK358QH6BiIJSR2eFhnJhiZJbkI2Oi1Rvf5N1hI6ehYeKZZVrl6Jj
bKB8q3luJwGxsrO0taUXnLkXCQA7
```

现在我们有了拼图的所有部分:创建包含多个部分的电子邮件消息的能力，以及对电子邮件进行编码和附加文件的方法。只是执行的问题。

### 使用 PHP 发送符合 MIME 的电子邮件

有了以上信息，实施就不成问题了。唯一的问题是如何定义任意附件的 MIME 类型。幸运的是，UNIX 系统提供了`file`命令，它可以读取任何文件并计算出其内容的 MIME 类型。在 Windows 服务器上，不存在这样的模拟，但是可以通过微软 Unix 服务或 UnxUtils 获得`file`。

下面提供了一个符合 MIME 的电子邮件解决方案，利用了这种策略和本文中提供的信息。

#### PHP:PHP 的邮件构建类

```
define('MIMEMAIL_HTML', 1);
define('MIMEMAIL_ATTACH', 2);
define('MIMEMAIL_TEXT', 3);

class MIMEMail
{
    private $plaintext;
    private $output;
    private $headers;
    private $boundary;

    public function __construct()
    {
        $this->output = '';
        $this->headers = '';
        $this->boundary = md5(microtime());
        $this->plaintext = 0;
    }

    // add: Add a part to the email
    // Parameters: type (Constant) - MIMEMAIL_TEXT, MIMEMAIL_HTML, MIMEMAIL_ATTACH
    //             name (String)   - Contents of email part if TEXT or HTML
    //                             - Attached name of file if ATTACH
    //             value (String)  - Source name of file if ATTACH
    public function add($type, $name, $value='')
    {
        switch($type)
        {
            case MIMEMAIL_TEXT:
                $this->plaintext = (strlen($this->output))?0:1;
                $this->output = "{$name}\r\n" . $this->output;
                break;

            case MIMEMAIL_HTML:
                $this->plaintext = 0;
                $this->writePartHeader($type, "text/html");
                $this->output .= "{$name}\r\n";
                break;

            case MIMEMAIL_ATTACH:
                $this->plaintext = 0;
                if(is_file($value))
                {
		    // If the file exists, get its MIME type from `file`
		    // NOTE: This will only work on systems which provide `file`: Unix, Windows/SFU
                    $mime = trim(exec('file -bi '.escapeshellarg($value)));
                    if($mime) $this->writePartHeader($type, $name, $mime);
                    else $this->writePartHeader($type, $name);

                    $b64 = base64_encode(file_get_contents($value));

                    // Cut up the encoded file into 64-character pieces
                    $i = 0;
                    while($i < strlen($b64))
                    {
                        $this->output .= substr($b64, $i, 64);
                        $this->output .= "\r\n";
                        $i += 64;
                    }
                }
                break;
        }
    }

    // addHeader: Provide additional message headers (Cc, Bcc)
    public function addHeader($name, $value)
    {
        $this->headers .= "{$name}: {$value}\r\n";
    }

    // send: Complete and send the message
    public function send($from, $to, $subject)
    {
        $this->endMessage($from);
        return mail($to, $subject, $this->output, $this->headers);
    }

    // writePartHeader: Helper function to add part headers
    private function writePartHeader($type, $name, $mime='application/octet-stream')
    {
        $this->output .= "--{$this->boundary}\r\n";
        switch($type)
        {
            case MIMEMAIL_HTML:
                $this->output .= "Content-type: {$name}; charset=\"iso8859-1\"\r\n";
                break;

            case MIMEMAIL_ATTACH:
                $this->output .= "Content-type: {$mime}\r\n";
                $this->output .= "Content-disposition: attachment; filename=\"{$name}\"\r\n";
                $this->output .= "Content-transfer-encoding: base64\r\n";
                break;
        }

        $this->output .= "\r\n";
    }

    // endMessage: Helper function to build message headers
    private function endMessage($from)
    {
        if(!$this->plaintext)
        {
            $this->output .= "--{$this->boundary}--\r\n";

            $this->headers .= "MIME-Version: 1.0\r\n";
            $this->headers .= "Content-type: multipart/mixed; boundary=\"{$this->boundary}\"\r\n";
            $this->headers .= "Content-length: ".strlen($this->output)."\r\n";
        }

        $this->headers .= "From: {$from}\r\n";
        $this->headers .= "X-Mailer: MIME-Mail v0.03, 20070419\r\n\r\n";
    }
}
```

#### mimemail 的用法示例

```
include('mimemail.php');

$m = new MIMEMail();

// Provide the message body
$m->add(MIMEMAIL_TEXT, 'An example email message.');

// Attach file 'icons/txt.gif', and call it 'text-icon.gif' in the email
$m->add(MIMEMAIL_ATTACH, 'text-icon.gif', '/var/www/icons/txt.gif');

// Send to the author
$m->send('noreply@oopsilon.com', '"Imran Nazar" <tf@oopsilon.com>', 'Test message');
```

下载脚本:[mimemail.php](http://web.archive.org/web/20220810161353/http://oopsilon.com/html/mimemail-20070419.phps)

2008 年[tf@oopsilon.com](http://web.archive.org/web/20220810161353/mailto:tf@oopsilon.com)>

*文章日期:2008 年 8 月 10 日*