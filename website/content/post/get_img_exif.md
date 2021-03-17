---
title: "如何获取图片的版权信息"
date: 2017-06-18T10:53:37+08:00
tags: [JavaScript]
draft: false
---

图片版权藏在哪儿
<!--more-->
### 1.图片水印
一般见于图片左下角或右下角的水印logo

### 2.图片元数据

数码图片由一组数字数据组成,记录着图片的像素、色彩、灰度等，除此之外，数码图片还有一组隐藏于幕后的数 字，这就是元数据（Data about Data，描述数据的数据）。图片元数据中记录着创作源设备、日期、位置、作者、版权等等。某些格式的图片如TIFF(tagged image file format)，JPG，PNG(portable network graphics)，EXIF (exchangeable image file)支持在文件中保存标 题、关键词、类别、作者、版权、URL等元数据，不影响图片的正常显示。可用于识别图片版权信息。使用Acdsee，Photoshop等图像软件都可加入这些信息，程序员还可以通过编程接口灵活添加这些信息。不过，利用图片元数据空间保存的图片版权信息，任何人都可以浏览修改，使它只能是一种辅助性或者隐蔽性的版权保护方法。

图 片 文 件 最 常 用 到 的 元 数 据 包 括 EXIF（ Exchangeable Image File Format，即可交换图像文件格式）和IPTC（Inter National Press Telecommunications Council，即国际出版 电讯委员会）。EXIF信息由图片生成设备产生，主要记录图片 生成设备的参数，比如光圈、快门、曝光时间、ISO感光值、焦 长等等。而IPTC信息包含了创作者姓名、版权信息、拍摄地 点、关键字等内容。图片创作者则可使用Acdsee，Photoshop等 图像编辑软件在EXIF或IPTC中添加相关信息以保护作品版权。 不过，图片元数据容量有限不适宜添加较多的信息内容，而且 使用元数据添加的信息很容易为其他用户修改，因此只能作为 一种辅助性的版权保护方法。

### 3.数字图像水印

数字水印技术是将一些 标识信息直接嵌入多媒体、文档、软件等数字载 体中，不影响原载体的使用价值，也不容易被人的知觉系统发现。通过这些隐藏在载体中的信息，可以达到确认内容创建者、购买者、传送隐秘 信息或者判断载体是否被篡改等目的。数字水印具有隐蔽性高、可以抵抗篡改、复制、压缩、转换、 滤波、重复添加等各种攻击的优点。目前已应用 于数字信息版权保护、印刷品防伪、信息来源追溯等领域。


## 获取图片的元数据

使用[exif.js](https://github.com/2xiao/getExif/blob/master/exif.js)提供的方法`EXIF.getData(img, callback)`和`EXIF.getAllIptcTags(img)`，如：


```
document.getElementById("file-input").onchange = function(e) {
    EXIF.getData(e.target.files[0], function() {
        var IptcObject = EXIF.getAllIptcTags(this);
        var IptcString = "";
        for(var tag in IptcObject){
        	var IptcText = IptcObject[tag];
        	IptcString += tag + " = " + IptcText + "<br>";
        }
        document.getElementById("output").innerHTML = IptcString;
    });
}

```
如图，要读取图片的IPTC信息：
![](http://ww4.sinaimg.cn/large/006tNc79ly1fftwu9cvqfj30p00iyn1x.jpg)

**提取出的版权信息为：**
>captionWriter = 李琰
>
>keywords = 中国
>
>headline = （一带一路·高峰论坛）习近平会见波兰总理希德沃
>
>dateCreated = 20170512
>
>copyright = 新华通讯社
>
>category = 18
>
>caption = 新华社照片，北京，2017年5月12日 习近平会见波兰总理希德沃 5月12日，国家主席习近平在北京人民大会堂会见来华出席“一带一路”国际合作高峰论坛的波兰总理希德沃。 新华社记者 谢环驰 摄
>
>byline = 谢环驰

## 中文编码乱码
在[exif.js](https://github.com/2xiao/getExif/blob/master/exif.js)中，方法`readIPTCData()`中，返回值`fieldValue`由方法`getFinalStringFromDB()`得到，这个方法的作用是获取到图片中存储IPTC信息的位置信息后，对字节信息进行编码，得到字符信息，但是编码时需要对中英文字符分开处理。

```
function getFinalStringFromDB(buffer, start, length) {
    var outstr = "";
    for (n = start; n < start+length; ) {
        if(buffer.getUint8(n)<127){
            outstr += String.fromCharCode(buffer.getUint8(n));
            n++;
        }else{
            var buffer10 = buffer.getUint16(n);
            var buffer16 = buffer10.toString(16).toUpperCase();
            var unicode10 = gbk2unicode(buffer16);
            var str = String.fromCharCode(unicode10);
            outstr += str;
            n += 2;
        }
    }
    return outstr
}

```
如果直接getUint8()，getUint8()每8个比特提取为一个字，非汉字是正确的，但是直接将汉字分为两个Unicode，则图片中的汉字被编译成拉丁文

```
function getStringFromDB(buffer, start, length) {
    var outstr = "";
    for (n = start; n < start+length; n++) {
        outstr += String.fromCharCode(buffer.getUint8(n));
    }
    return outstr;
}

// captionWriter = Àîçü
// keywords = ÖÐ¹ú
// headline = £¨Ò»´øÒ»Â·¡¤¸ß·åÂÛÌ³£©Ï°½üÆ½»á¼û²¨À¼×ÜÀíÏ£µÂÎÖ
// dateCreated = 20170512
// copyright = ÐÂ»ªÍ¨Ñ¶Éç
// category = 18
// caption = ÐÂ»ªÉçÕÕÆ¬£¬±±¾©£¬2017Äê5ÔÂ12ÈÕ Ï°½üÆ½»á¼û²¨À¼×ÜÀíÏ£µÂÎÖ 5ÔÂ12ÈÕ£¬¹ú¼ÒÖ÷Ï¯Ï°½üÆ½ÔÚ±±¾©ÈËÃñ´ó»áÌÃ»á¼ûÀ´»ª³öÏ¯¡°Ò»´øÒ»Â·¡±¹ú¼ÊºÏ×÷¸ß·åÂÛÌ³µÄ²¨À¼×ÜÀíÏ£µÂÎÖ¡£ ÐÂ»ªÉç¼ÇÕß Ð»»·³Û Éã
// byline = Ð»»·³Û

```
如果直接getUint16()，则每16个比特提取为一个字，能提取出汉字，但是对应关系错误，因为此时还没有修改`n`，相当于每次还只向后挪动8bytes，所以还要修改`n`和`length`：

```
function getFinalStringFromDB(buffer, start, length) {
    var outstr = "";
    length = length/2;
    for (n = start; n < start+length; ) {
        var buffer10 = buffer.getUint16(n);
        var buffer16 = buffer10.toString(16).toUpperCase();
        var unicode10 = gbk2unicode(buffer16);
        outstr += String.fromCharCode(unicode10);
    }
    return outstr
}

// captionWriter = 李铉
// keywords = 中泄
// headline = 一淮带一宦路贰高叻峰迓论厶坛常习
// dateCreated =
// copyright = 新禄华通
// category =
// caption = 新禄华社缯照掌片北本京年月日习敖近平交会峒见波兰甲总芾理硐希德挛沃月日眨国家抑主飨席习敖近平皆在诒北本京
// byline = 谢换环
```
修改n和length，因为n的位置应该向后移动两个比特，所以作出修改，此时汉字部分正确，但是非汉字部分解码错误

```
function getFinalStringFromDB(buffer, start, length) {
    var outstr = "";
    for (n = start; n < start+length; ) {
        var buffer10 = buffer.getUint16(n);
        var buffer16 = buffer10.toString(16).toUpperCase();
        var unicode10 = gbk2unicode(buffer16);
        outstr += String.fromCharCode(unicode10);
        n += 2;
    }
    return outstr
}

// captionWriter = 李琰
// keywords = 中国
// headline = 一带一路高峰论坛习近平会见波兰总理希德沃
// dateCreated =
// copyright = 新华通讯社
// category =
// caption = 新华社照片北京年习近平会见波兰总理希德沃月日国家主席习近平在北京人民大会堂会见来华出席一带一路国际合作高峰论坛的波兰总理希德沃禄缂钦谢环驰
// byline = 谢环驰
```
修改if判断，对应关系是两个拉丁文字符对应一个汉字，一个汉字占16比特，一个非汉字只占8比特，所以要进行判断，当`Unicode>127`时，代表此处和后面的8bytes组成一个汉字，当`Unicode<127`时，代表此处是英文字符。

```
function getFinalStringFromDB(buffer, start, length) {
    var outstr = "";
    for (n = start; n < start+length; ) {
        if(buffer.getUint8(n)<127){
            outstr += String.fromCharCode(buffer.getUint8(n));
            n++;
        }else{
            var buffer10 = buffer.getUint16(n);
            var buffer16 = buffer10.toString(16).toUpperCase();
            var unicode10 = gbk2unicode(buffer16);
            var str = String.fromCharCode(unicode10);
            outstr += str;
            n += 2;
        }
    }
    return outstr
}
// captionWriter = 李琰
// keywords = 中国
// headline = 一带一路高峰论坛习近平会见波兰总理希德沃
// dateCreated = 20170512
// copyright = 新华通讯社
// category = 18
// caption = 新华社照片北京2017年5月12日 习近平会见波兰总理希德沃 5月12日国家主席习近平在北京人民大会堂会见来华出席一带一路国际合作高峰论坛的波兰总理希德沃 新华社记者 谢环驰 摄
// byline = 谢环驰

```
至此，成功解决了中文编码乱码的问题。



