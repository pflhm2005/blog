### 前言
上一篇文件简单讲了一下excel、word等文件的导出原理，其中有一块很重要的部分是将所有xml文件、文件夹整合成一个压缩文件。
> 有了单个的文件，那么最后需要将其全部整合成一个zip文件，这里用的是另外一个插件jszip。

在当时，我用了这个名为jszip的插件，这一篇就来讲这个zip插件的生成原理。

这里需要重新申明一下，所有的工具插件核心原理其实都非常简单，复杂度都来源于兼容、多样化参数配置处理、功能解耦、代码预留等。这样可以让代码维护与合作开发变的简单，但是对于阅读源码学习的人来说就比较痛苦，毕竟我不关心每一个类都有什么作用。

当时用了一个名为jszip的插件，这次就来讲zip文件的原理（[代码](https://github.com/pflhm2005/XLSX/blob/master/lib/dep/jszip/index.js))

话不多说，开始。

### 总览
与W3C一样，对于zip或类zip的压缩文件，都有一个官方的格式规范，具体的实现并不统一。

[官方的地址](https://pkware.cachefly.net/webdocs/casestudies/APPNOTE.TXT)

文档对zip文件的结构做了详细的规定，其中有些部分不是必须的，一图总览如下：
```
  [local file header 1]
  [encryption header 1](加密)
  [file data 1]
  [data descriptor 1](压缩)
  . 
  .
  .
  [local file header n]
  [encryption header n](加密)
  [file data n]
  [data descriptor n]
  [archive decryption header] (加密)
  [archive extra data record] (加密)
  [central directory header 1]
  .
  .
  .
  [central directory header n]
  [zip64 end of central directory record](64位)
  [zip64 end of central directory locator](64位)
  [end of central directory record]
```
如上所示，文件的结构非常简单，按照先文件再文件夹的顺序将数据加入压缩文件中，其中部分结构是可选的。再具体一点，整体结构如下：

文件1头部 - 文件1内容 - ... - 文件n头部 - 文件n内容 - 文件夹1头部 - ... - 文件夹n头部 - 文件夹尾部

### 文件
文件头部一般都包含了一些描述信息，比如内容长度、时间、宿主平台等等，官方文档都有详细的定义，规范如下：
```
  local file header signature     4 bytes  (0x04034b50)
  version needed to extract       2 bytes
  general purpose bit flag        2 bytes
  compression method              2 bytes
  last mod file time              2 bytes
  last mod file date              2 bytes
  crc-32                          4 bytes
  compressed size                 4 bytes
  uncompressed size               4 bytes
  file name length                2 bytes
  extra field length              2 bytes

  file name (variable size)
  extra field (variable size)
```
依次解释上述内容：
- local file header signature
这是一个固定标识(0x04034b50)，翻译成字符串就是'\x04\x03KP'，但是由于这里的读取方式是小端，所以在实际输入时是'PK\x03\x04'

- version needed to extract 代表待处理压缩文件的级别，比如说有文件夹、内容用XX方式压缩、用XX方式加密等等都是不同的值
- general purpose bit flag 内容的编码格式、是否加密等描述信息
- compression method 压缩方式，默认不压缩
- last mod file time/date 文件的时间日期信息，默认是new Date()。关于格式，在DOS平台有规范：

  time => 16位中 15-11存hours 10-5存minutes 4-0存second/2

  假设时间为12时34分56秒 => 换算后为 01100(12点) 100010(34分) 11100(28秒)

  date => 16位中 15-9存year-1980 8-5存month 4-0存day

  假设时间为2022年6月29日 => 换算后为 0101010(42) 0110(6) 11101(29)

- crc-32 完整性校验，关于crc32这里暂时不做展开
- compressed size/uncompressed size 文件压缩长度信息，如果不做压缩，这里的size均为数据的长度
- file name length/extra field length 字面意思
- file name 文件名，可能被encode过
- extra field 略

头部后面，接的是file data，指的就是文件内容，可以选择原始内容，也可以选择一个压缩方式进行压缩

### 文件夹
文件夹就更简单了，只有描述信息和一个尾巴，规范如下：

头部
```
  central file header signature   4 bytes  (0x02014b50)
  version made by                 2 bytes
  version needed to extract       2 bytes
  general purpose bit flag        2 bytes
  compression method              2 bytes
  last mod file time              2 bytes
  last mod file date              2 bytes
  crc-32                          4 bytes
  compressed size                 4 bytes
  uncompressed size               4 bytes
  file name length                2 bytes
  extra field length              2 bytes
  file comment length             2 bytes
  disk number start               2 bytes
  internal file attributes        2 bytes
  external file attributes        4 bytes
  relative offset of local header 4 bytes

  file name (variable size)
  extra field (variable size)
  file comment (variable size)
```
尾巴
```
  end of central dir signature    4 bytes  (0x06054b50)
  number of this disk             2 bytes
  number of the disk with the
  start of the central directory  2 bytes
  total number of entries in the
  central directory on this disk  2 bytes
  total number of entries in
  the central directory           2 bytes
  size of the central directory   4 bytes
  offset of start of central
  directory with respect to
  the starting disk number        4 bytes
  .ZIP file comment length        2 bytes
  .ZIP file comment       (variable size)
```
这里多了几个参数，挑出来解释一下。
- version made by 操作系统及编码zip文件的版本

  高位 => 操作系统 00是DOS 03是UNIX

  低位 => 编码文件的软件版本 0x0014 => 14 => 1.4版本
- disk number start/internal file attributes 默认为\x00\x00
- external file attributes 平台相关的值
- total number of ... 压缩的文件数/压缩的文件夹数
- size of the central directory ... 文件夹头部总长度
- offset of start of ... 文件头部+内容总长度

按照如上的结构依次将数据整合，最后就可以生成一个zip文件了。
### 代码实现
有了规范，就可以来实现一个最简单的压缩文件，无压缩，无加密，无编码。

假设有一个文件名为Hello.txt，内容是Hello World\n，放在默认的文件夹下，需要将其处理成一个zip文件。

首先看头部，其中需要实现的有时间、crc32、长度等，其余全部设为默认值。

按照2022年7月25日，早上10点整来算:

time: 01010(10点) 000000(0分) 00000(0秒) => \x50\x00

date: 0101010(42) 0111(7) 11001(25) => \x54\xf9

crc32去网上找个工具：b095e5e3

文件名长度为9，文件内容长度为12，文件头部+数据长度为51，文件夹头部长度为55

对应的数据组合如下：
- Local file header

['PK\x03\x04', '\x0A\x00', '\x00\x00', '\x00\x00', '\x00\x50', '\xF9\x54', '\xe3\xe5\x95\xb0', '\x0c\x00\x00\x00', '\x0c\x00\x00\x00', '\x09\x00', '\x00\x00', 'Hello.txt']

加粗的字体的是header，在文件夹中也要用到

- file data

['Hello World\n']

- central directory header

['PK\x01\x02', '\x14\x00', '\x0A\x00', '\x00\x00', '\x00\x00', '\x00\x50', '\xF9\x54', '\xe3\xe5\x95\xb0', '\x0c\x00\x00\x00', '\x0c\x00\x00\x00', '\x09\x00', '\x00\x00', '\x00\x00', '\x00\x00', '\x00\x00', '\x00\x00\x00\x00', '\x00\x00\x00\x00', 'Hello.txt']

加粗的部分与文件的header相同

- end of central directory record

['PK\x05\x06', '\x00\x00', '\x00\x00', '\x01\x00', '\x01\x00', '\x37\x00\x00\x00', '\x33\x00\x00\x00', '\x00\x00']

数组中的每一个值，都对应了文档中一个元素，合并上述数据，得到一长串字符串，用老方式输出到浏览器。
```
function s2ab(s) {
  let buf = new ArrayBuffer(s.length);
  let view = new Uint8Array(buf);
  for (let i = 0; i != s.length; ++i) view[i] = s.charCodeAt(i) & 0xFF;
  return buf;
}
const bytes = [
  'PK\x03\x04', '\x0A\x00', '\x00\x00', '\x00\x00', '\x00\x50', '\xF9\x54', '\xe3\xe5\x95\xb0', '\x0c\x00\x00\x00', '\x0c\x00\x00\x00', '\x09\x00', '\x00\x00', 'Hello.txt',
  'Hello World\n',
  'PK\x01\x02', '\x14\x00', '\x0A\x00', '\x00\x00', '\x00\x00', '\x00\x50', '\xF9\x54', '\xe3\xe5\x95\xb0', '\x0c\x00\x00\x00', '\x0c\x00\x00\x00', '\x09\x00', '\x00\x00', '\x00\x00', '\x00\x00', '\x00\x00', '\x00\x00\x00\x00', '\x00\x00\x00\x00', 'Hello.txt',
  'PK\x05\x06', '\x00\x00', '\x00\x00', '\x01\x00', '\x01\x00', '\x37\x00\x00\x00', '\x33\x00\x00\x00', '\x00\x00'
].join('');
let blob = new Blob([s2ab(bytes)], { type: "application/octet-stream" });
let url = URL.createObjectURL(blob);
const a = document.createElement('a');
a.href = url;
a.download = '123.zip';
a.click();
setTimeout(() => {
  URL.revokeObjectURL(url);
}, 10000);
```
得到一个123.zip的压缩文件，解压后得到一个Hello.txt的文本文件，查看修改时间，正好是2022年7月25日10点整
![Alt](https://pic4.zhimg.com/80/v2-1d534ef0d2c0e85a833ea8396119474f_720w.jpg)

文件内容也是我们写入的文案：
![Alt](https://pic1.zhimg.com/80/v2-e274299ba826ef340e36db5ec085ea40_720w.jpg)

此时，按照规范已经实现了一个最简单的zip文件，关于加密、压缩、crc32等知识点有兴趣的可以自行探索，不久的将来我也可能会简单的科普。
