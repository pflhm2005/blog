### 前言
关于excel的纯前端导出，之前一直做的是普通文本的生成，尚未尝试过插入图片这种新型的方式

通过展开插入图片的excel文件，可以发现比起普通的目录，多了4个文件，分别是
```
xl/drawings/drawing1.xml  // 描述图片类型、位置等信息
xl/drawings/_rels/drawing1.xml.rels // 图片位置的映射
xl/worksheets/_rels/sheet.xml.rels  // 对drawing.xml文件的映射
xl/media/image1.png // 图片源文件 可能不止一个 按照序号依次命名
```
除去2个映射文件，剩下的就是图片和图片展示的具体描述，还是比较好理解的

在对应的sheet.xml中，只是多了一行``<drawing r:id="rId1"/>``表示这个sheet中有图形资源

那么现在就有两个问题

1. 如何将png图片插入压缩文件中
2. 图片描述文件的内容解析

首先看第二个问题，对于该文件的大概内容如下:
```xml
// ...xml头部与前缀bound文件
<xdr:twoCellAnchor editAs="oneCell">
  <xdr:from>
    <xdr:col>3</xdr:col>
    <xdr:colOff>9525</xdr:colOff>
    <xdr:row>7</xdr:row>
    <xdr:rowOff>9525</xdr:rowOff>
  </xdr:from>
  <xdr:to>
    <xdr:col>17</xdr:col>
    <xdr:colOff>466725</xdr:colOff>
    <xdr:row>40</xdr:row>
    <xdr:rowOff>9525</xdr:rowOff>
  </xdr:to>
  <xdr:pic>
    // ...其余标签
    <xdr:blipFill>
      <a:blip r:embed="rId1"/>
// ...其余标签
```
通过查阅Office文档，可以知道from、to分别代表了图片的起始和结束位置，col和row很明显就是对应的行列数，但是这个colOff就很抽象。

关于colOff，字面意思非常好理解，就是列的偏移量，图片不总会刚好是一个整格，会有一定的超出，所以需要一个值，问题是这个数字太大了，比如下面的466725。文档中给出了这个值的定义：``The possible values for this element are defined by the ST_Coordinate simple type (§20.1.10.16).``这个ST_Coordinate我查过，看不懂，有比较通俗和简单的文章可以告诉我一下。

值看不懂，但是可以猜。于是我把图片拉满，然后通过最大值，来算1的宽度或高度可以换算成多少，于是有了源码中的10305和9588，经过简单测试，生成的图片尺寸误差小于2px，可以接受了。

除了位置定义，剩下的有效信息就是那个embed，代表插入的是什么东西，rId1在映射文件中Target指向media文件夹中的图片源文件，所以最后渲染的就是给定起点、终点的一张图片。

再回头看第一个问题，由于是纯前端，所以fs肯定用不了，如何写入一个图片，还是需要靠Blob来操作。

经过多番查阅，目前有了2个方向
1. canvas + img

通过对img标签设置src，可以获取到给定图片的宽高。接下来通过canvas的drawImage + toBlob可以将img转换成数据，从而写入文件并导出。

但这里遇到了一个问题，即``Tainted canvases may not be exported.``
```js
  const canvas = document.getElementById('canvas');
  const ctx = canvas.getContext('2d');
  const img = document.createElement('img');
  img.src='./1.jpg';
  img.onload = () => {
    ctx.drawImage(img, 0, 0);
    canvas.toBlob(b => b); // error
  } 
```
通过查阅资料，得出这里的img需要设置``crossOrigin``属性，但是设置后又会出现跨域问题。最后得出的结论就是，这个方法仅限于线上 + 服务器设置了access-control-allow-origin: '*'的图片。

<del>

2. File System Access API

这个方式可以支持本地的图片，但是非常难用，因为不是0帧起手，需要``showOpenFilePicker``。这个方式或许很多人没用过，但是``<input type='file'>``应该都知道，调用这个方法相当于点了那个上传按钮弹出的文件选择。

该方法返回一个文件句柄，通过句柄可以获取、操作文件。于是，导出的逻辑变成了如下代码
```js
// 这个方法必须通过用户交互触发
DOM.addEventListener('click', async () => {
  const [fileHandle] = await window.showOpenFilePicker({
    types: [
      {
        description: '图片',
        accept: {
          "image/*": [".png", ".gif", ".jpeg", ".jpg"],
        }
      }
    ],
    multiple: false,
  });
  let file = await fileHandle.getFile();
  exportExcel(file);
});
```
虽说可以插入本地图片，而且excel里面插入图片好像也要走这个步骤(- -)，不过总感觉很奇怪，我的代码不好写了。

</del>

关于问题1真是瞎搞了，之前一直是本地直接打开html引入dist在调试，感觉不对劲。后来随便起了一个服务，在node下调试图片引入，发现没有任何问题，画布污染跨域什么的通通没有，此问题终结。

## 后记

如果有别的纯前端方法可以获取图片的数据，可以告诉我！