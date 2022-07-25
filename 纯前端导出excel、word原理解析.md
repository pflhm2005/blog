### 前言
插件是几年前写的，拿出来复习一下，原理来源于一个开源插件sheet.js。
![Alt](https://pic1.zhimg.com/80/v2-ec737413659df0abd5304bcd792dfe65_720w.png)

经过改良后，加入了自己的[github](https://github.com/pflhm2005/XLSX)，如下：

- 抽离核心代码，去掉大量兼容、无用、老代码，用最新语法重写，从正宗大肥猪瘦身成白骨精！
- 添加样式功能，支持宽高、字体大小、对齐等少量但实用的样式制定
- 额外支持了word文档的导出，若有兴趣，读者可自信学习扩展到pdf、ppt等office文档，核心原理一致
- 将相关的依赖重写并通过内部引入，保证没有node_modules

插件主要API的名字延续了sheetJs，如aoa_to_sheet

### 主要原理
每一个office文件，比如excel，本质上都是一个压缩文件，集成了各类子文件，系统打开的时候会解析并打开成对应类型的文件。

![Alt](https://pic1.zhimg.com/80/v2-ab8ad4e50404611104bde9d2a36584d6_720w.png)

上图即是一个excel文件的实际内容，包含了文件的引用、配置信息、文件内容等等一系列文件。  

明白了相关的文件构成，那么如何生成一个文件呢？这里就要用到两个API：Blob、URL.createObjectURL

其中Blob负责生成一个原始文件对象，URL.createObjectURL负责将文件对象转换成URL对象，以便我们下载

使用方式如下：
```
let blob = new Blob(['abbbb'], { type: "application/octet-stream" });
let url = URL.createObjectURL(blob);
const a = document.createElement('a');
a.href = url;
a.download = 'abc.txt';
a.click();
setTimeout(() => {
 URL.revokeObjectURL(url);
}, 10000);
```
将上述代码复制到浏览器运行，会得到一个txt文件，内容正是我们写入的abbbb

有了上面两个，就可以着手开始生成excel文件了。

首先分解每一个excel文件，查看其中内容了，可以发现大部分是xml文件，如
```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Types xmlns="http://schemas.openxmlformats.org/package/2006/content-types">
...more
```
可以写一个工具方法来生成对应的XML文件，下面是我写的其中一个方法
```
writeTag(o) {
  if(!o) return '';
  let properties = '';
  if(o.p) properties = Object.keys(o.p).map(key => ` ${key}="${o.p[key]}"`).join('');
  if(o.t === undefined && !o.c) return `<${o.n}${properties}/>`;
  else if(o.c !== undefined) return `<${o.n}${properties}>${(o.c || []).map(v => this.writeTag(v)).join('')}</${o.n}>`;
  else return `<${o.n}${properties}>${o.t}</${o.n}>`;
}

writeTag({n:'Properties'}) => <Properties/>
writeTag({n:'Properties', t: 'Microsoft Macintosh Excel'}) => <Properties>Microsoft Macintosh Excel</Properties>
...more
```
通过一个ast可以生成所需的xml文件，详情可见源码中的lib/docx(xlsx)/ast.js

有了单个的文件，那么最后需要将其全部整合成一个zip文件，这里用的是另外一个插件jszip。

具体的原理有时间可以去研究一下，大概就是在文件头尾加一些标识符，中间放压缩文件的内容。

最后，我们的excel文件就成功出来了，流程如下：

输入对应的EXCEL表格数组 => 转换成一系列XML字符串 => 合并成一个zip文件流 => 通过a标签的事件下载

觉得可以就给我个Star~