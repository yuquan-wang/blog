---
layout: post
title:  "从头写一个bittorrent(1)-Bencode"
date:   2019-01-12 05:49:45 +0900
categories: bittorrent
---
从本篇文章开始，打算写一系列文章来说明，如何从头开始实现一个bittorrent.具体代码在[build-your-own-bittorrent](https://github.com/yuquan-wang/build-your-own-bittorrent)中

torrent是我们经常用到的下载工具，利用torrent我们可以进行非常快速的下载，尤其我们经常会听到，越多人做种，你下载的就越快，那么它背后到底是什么原理？从本文开始，我们将逐渐揭露它背后的工作原理。

第一步我们需要对torrent文件有一个认识，我从[Ubuntu官网](https://www.ubuntu.com/download/alternative-downloads)下载了Ubuntu 18.10 Desktop (64-bit)的种子，并且用文件编辑器打开，如下展示了该种子文件的前若干行：
> d8:announce39:http://torrent.ubuntu.com:6969/announce13:announce-listll39:http://torrent.ubuntu.com:6969/announceel44:http://ipv6.torrent.ubuntu.com:6969/announceee7:comment29:Ubuntu CD releases.ubuntu.com13:creation datei1539860537e4:infod6:lengthi1999503360e4:name30:ubuntu-18.10-desktop-amd64.iso12:piece lengthi524288e6:pieces76280:&o

上面的内容比较难看懂，因为该内容是被编码过了的，这里就涉及到本文主要介绍的Bencode了。

Bencode的主要介绍可以看[wiki](https://zh.wikipedia.org/wiki/Bencode), 这里我把主要的列一下：
>
Bencode（发音为Bee-Encode）是BitTorrent用在传输数据结构的编码方式。这种编码方式支持四种数据类型：
- 字符串(一个字节的字符串（只是一个字节的字符串，不一定是一个方块字）会以（长度）:（内容）编码，长度的值和数字编码方法一样，只是不允许负数；内容就是字符串的内容，如字符串“spam”就会编码为“4:spam”，本规则不能处理ASCII以外的字符串，为了解决这个问题，一些BitTorrent程序会以非标准的方式将ASCII以外的字符以UTF-8转化后再编码。)
- 整数(一个整型数会以十进制数编码并括在i和e之间，不允许前导零（但0依然为整数0），负数在编码后直接加前导负号，不允许负零。如整型数“42”编码为“i42e”，数字“0”编码为“i0e”，“-42”编码为“i-42e”。)
- 串列(线性表会以l和e括住来编码，其中的内容为Bencode四种编码格式所组成的编码字符串，如包含和字符串“spam”数字“42”的线性表会被编码为“l4:spami42ee”，注意分隔符要对应配对。)
- 字典表(字典表会以d和e括住来编码，字典元素的键和值必须紧跟在一起，而且所有键为字符串类型并按字典顺序排好。如键为“bar”值为字符串“spam”和键为“foo”值为整数“42”的字典表会被编码为“d3:bar4:spam3:fooi42ee”。)

Bencode还是比较简单的，支持4种类型，每种都有特殊的开始字符，如以i开头我们就知道是int，以l开始我们就知道是list,以d开头我们就知道是dict了，剩下的就只是string了，那么以这个规则来看，我们就能读懂Ubuntu torrent的内容了，差不多如下：
{% highlight xml %}
{
  "announce": "http://torrent.ubuntu.com:6969/announce",
  "announce-list": [
    [
      "http://torrent.ubuntu.com:6969/announce"
    ],
    [
      "http://ipv6.torrent.ubuntu.com:6969/announce"
    ]
  ],
  "comment": "Ubuntu CD releases.ubuntu.com",
  "creationdate": 1539860537,
  "info": {
    "length": 1999503360,
    "name": "ubuntu-18.10-desktop-amd64.iso",
    "piece length": 524288,
    "pieces": "XXXXXXX"
  }
}
{% endhighlight %}
是不是就比较好懂了呢？ 在本篇文章中，主要讲解如何实现这个编码和解码。

首先介绍一下环境的搭建，我使用Intellij创建了一个gradle项目，并向其中添加junit依赖用来做单元测试。

来看一下如何对对象进行解码，其实也比较直观，假设一个input过来，我们人为的会怎么分析，那就是看第一个字符，如果是i，那么就是一个int,把接下来的输入读取，直到读到结尾符号e为止，如果是个数字类型，那么就是个string,先读取长度，在读取string内容，如果是l开头，那么我们知道是个list类型，然后读取list元素，元素怎么读取呢，那么同样的也是看元素的第一个字符判断类型进行读取（这里就需要用到递归了），直到读到结尾符e为止，如果是d开头，那么我们知道是个dict类型，然后分别读取key和value,具体同list,知道读到结尾符e为止。

具体代码如下：
{% highlight java %}
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/*
    Algorithm introduction: https://en.wikipedia.org/wiki/Bencode
 */
public class BEncoder {
    private static final String INVALID = "Invalid input";

    private String data = "";
    private int index = 0;

    void setInput(String input) {
        data = input;
        index = 0;
    }

    public Object decode(String input) throws Exception {
        setInput(input);
        Object ret = internal_decode();
        if (index == data.length()) {
            return ret;
        } else {
            throw new Exception(INVALID);
        }
    }

    // decode data as an object recursively
    private Object internal_decode() throws Exception {
        char c = data.charAt(index);
        if (c == 'i') {
            //if it is type of integer
            index++; // skip ':'
            String s = "";
            while (data.charAt(index) != 'e') {
                s += data.charAt(index);
                index++;
            }
            index++;// skip 'e'
            int output = 0;
            try {
                output = Integer.valueOf(s);
            } catch (NumberFormatException e) {
                throw new Exception(INVALID);
            }
            if (isValidInteger(output, s)) {
                return output;
            } else {
                throw new Exception(INVALID);
            }
        } else if (c == 'l') {
            //if it is type of list
            List<Object> output = new ArrayList<>();
            index++; // skip 'l'
            while (data.charAt(index) != 'e') {
                output.add(internal_decode());
            }
            index++; // skip 'e'
            return output;
        } else if (c == 'd') {
            //if it is type of dictionary
            Map<Object, Object> output = new HashMap<>();
            index++; // skip 'd'
            while (data.charAt(index) != 'e') {
                Object key = internal_decode();
                Object value = internal_decode();
                output.put(key, value);
            }
            index++; // skip 'e'
            return output;
        } else {
            //it should be type of string
            //for performance we could use StringBuilder later
            String lenS = "";
            while (data.charAt(index) != ':') {
                lenS += data.charAt(index);
                index++;
            }
            int len = 0;
            try {
                len = Integer.valueOf(lenS); // when length is too large, better to user long
            } catch (NumberFormatException e) {
                throw new Exception(INVALID);
            }
            String output = "";
            index++; // ignore char ':'
            for (int i = 0; i < len; i++) {
                output += data.charAt(index++);
            }
            return output;
        }
    }

    private boolean isValidInteger(int n, String s) {
        if (n == 0) {
            if (s.equals("0")) { // only i0e is valid
                return true;
            } else {
                return false;
            }
        } else {
            if (String.valueOf(n).length() != s.length()) { //contains leading zero like i03e
                return false;
            } else {
                return true;
            }
        }
    }
}
{% endhighlight %}

返回来要把一个对象进行encode的话就更加简单了，直接根据类型和规则进行转换即可：
{% highlight java %}
public String encode(Object input) throws Exception {
        if (input instanceof String) {
            return ((String) input).length() + ":" + input;
        } else if (input instanceof Integer) {
            return "i" + input + "e";
        } else if (input instanceof Iterable) {
            String output = "l";
            for (Object element : (Iterable) input) {
                output += encode(element);
            }
            output += "e";
            return output;
        } else if (input instanceof Map) {
            String output = "d";
            for (Object key : ((Map) input).keySet()) {
                Object value = ((Map) input).get(key);
                output += encode(key);
                output += encode(value);
            }
            output += "e";
            return output;
        } else {
            throw new Exception(INVALID);
        }
    }
{% endhighlight %}

当然我们可以用一系列的单元测试来看看实现是否正确，在这里不加细讲，详细可以看[测试用力](https://github.com/yuquan-wang/build-your-own-bittorrent/blob/master/src/test/java/BEncoderTest.java)

了解Bencode是最最基础的，接下来将讲解如果根据内容构建一个种子文件。
