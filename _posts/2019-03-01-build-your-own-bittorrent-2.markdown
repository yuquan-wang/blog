---
layout: post
title:  "从头写一个bittorrent(2)-TorrentCreator"
date:   2019-03-01 05:49:45 +0900
categories: bittorrent
---
从[第一篇文章](https://yuquan-wang.github.io/blog/2019/01/11/build-your-own-bittorrent-1.html)开始，打算写一系列文章来说明，如何从头开始实现一个bittorrent.具体代码在[build-your-own-bittorrent](https://github.com/yuquan-wang/build-your-own-bittorrent)中

BitTorrent完整的协议在这里：[BitTorrent protocal specification](https://wiki.theory.org/index.php/BitTorrentSpecification),差不多这个教程就是跟着协议一步步写下来，做一个简单版本的bittorrent系统。

上文已经介绍了torrent中的编码和解码协议，这篇文章主要介绍程序上如何根据做种文件来制作一个torrent文件，为了简单起见，我们只关注一个文件，当然协议肯定支持多个文件的，只是为了示例更简介，我们就focus on如何处理一个文件的逻辑上。

下面首先来看一下torrent文件里面都需要有哪些元素（只挑选了一个必须要的且只关注一个文件的）：
>
Metainfo File Structure
All data in a metainfo file is bencoded. The specification for bencoding is defined above.
The content of a metainfo file (the file ending in ".torrent") is a bencoded dictionary, containing the keys listed below. All character string values are UTF-8 encoded.

> info: a dictionary that describes the file(s) of the torrent. There are two possible forms: one for the case of a 'single-file' torrent with no directory structure, and one for the case of a 'multi-file' torrent (see below for details)

> announce: The announce URL of the tracker (string)

torrent 文件里主要包含两个字段，一个是announce,一个是info,其中announce告诉torrent以后要和哪里的tracker（以后的文章会提到，是一个保存bittorrent心跳的应用），而info字段则是具体和要被做成种子的问题有关的各个信息， 可以看一下info 里面的具体字段：

> piece length: number of bytes in each piece (integer)，torrent会把整个文件分块，这个变量定义了每个块的大小

> pieces: string consisting of the concatenation of all 20-byte SHA1 hash values, one per piece (byte string, i.e. not urlencoded)， 分块后每个块计算SHA1,最后把这些SHA1拼接起来

> name: the filename. This is purely advisory. (string) torrent的名字，和输入的文件同名

> length: length of the file in bytes (integer) 输入的文件的大小

其实就基本上是上文中的ubuntu torrent文件的内容了，我们可以再来看一下：
{% highlight xml %}
{
  "announce": "http://torrent.ubuntu.com:6969/announce",
  "info": {
    "length": 1999503360,
    "name": "ubuntu-18.10-desktop-amd64.iso",
    "piece length": 524288,
    "pieces": "XXXXXXX"
  }
}
{% endhighlight %}
我把一些可有可无的选项删了以后，torrent文件便如上面所示。 所以我们可以先定义Torrent文件如下：
{% highlight java %}
public class Torrent {
    static String ANNOUNCE_FIELD = "announce";
    static String NAME_FIELD = "name";
    static String PIECE_LENGTH_FIELD = "pieceLength";
    static String LENGTH_FIELD = "length";
    static String PIECES_FILED = "pieces";
    static String INFO_FIELD = "info";

    private String announce;
    private String name;
    private int pieceLength;
    private int length;
    private String pieces;

    public Torrent(String announce, String name, int pieceLength, int length, String pieces) {
        this.announce = announce;
        this.name = name;
        this.pieceLength = pieceLength;
        this.length = length;
        this.pieces = pieces;
    }

    Map<String, Object> torrentToMap() {
        Map<String, Object> m = new HashMap<>();
        m.put(ANNOUNCE_FIELD, announce);
        Map<String, Object> info = new HashMap();
        info.put(NAME_FIELD, name);
        info.put(PIECE_LENGTH_FIELD, pieceLength);
        info.put(LENGTH_FIELD, length);
        info.put(PIECES_FILED, pieces);
        m.put(INFO_FIELD, info);
        return m;
    }

    //ignore setter and getter

}

{% endhighlight %}
这里比较直接的把所有字段都直接放在Torrent下，是为了方便使用，其实像name字段是在info字段下的，但如果每次去取name，都要先从info入手就显得很麻烦，所以添加了一个torrentToMap方法，只在在写入torrent文件或者读取torrent文件的时候，我们才会把name等字段放在info内部，不然大部分情况下我们可以假设torrent就直接有name这个字段。

接下来我们就需要一个TorrentUtil了，这个类可以帮助我们把抽象出来的torrent写到torrent文件里，或者从torrent文件里读取出来这个torrent，可以看成是序列化和反序列化吧。
{% highlight java %}
public class TorrentUtil {
    private final static String ANNOUNCE = "localhost:6881"; //暂时固定tracker的地址
    private final static int PIECE_LENGTH = 1048576; // 2^20, 1MB by default

    /**
     * @param file the source file to make a torrent, current we only support single file
     * @param dest create a torrent file under dest, torrent name will be same as file's name
     * @throws Exception
     */
    public static void createTorrent(File file, File dest) throws Exception {
        Assert.assertTrue("Torrent source should be a file", file.isFile());
        Assert.assertTrue("Torrent destination should be a directory", dest.isDirectory());

        String name = file.getName();
        int length = (int) file.length();
        int len = length;
        String pieces = "";
        FileInputStream fileInputStream = new FileInputStream(file);
        while (len > 0) {
            long nextPieceLength = len >= PIECE_LENGTH ? PIECE_LENGTH : len;

            // read content of size of nextPieceLength
            byte[] contents = IOUtils.toByteArray(fileInputStream, nextPieceLength);
            // calculate SHA1
            byte[] sha1 = DigestUtils.sha1(contents);
            pieces += new String(sha1, Charset.defaultCharset());

            len -= nextPieceLength;
        }

        File torrentFile = new File(dest, name + ".torrent");
        torrentFile.createNewFile();
        Bencode bencode = new Bencode();
        //需要将map 进行编码
        String torrentContent = bencode.encode(new Torrent(ANNOUNCE, name, PIECE_LENGTH, length, pieces).torrentToMap());
        FileUtils.writeStringToFile(torrentFile, torrentContent, Charset.defaultCharset());
    }

    public static Torrent loadTorrentFromFile(File torrentLocation) throws Exception {
        Assert.assertTrue(torrentLocation.exists());

        String torrentContent = FileUtils.readFileToString(torrentLocation, Charset.defaultCharset());
        Bencode bencode = new Bencode();
        Map<String, Object> contentMap = (Map<String, Object>) bencode.decode(torrentContent);
        return new Torrent(
                (String) contentMap.get(Torrent.ANNOUNCE_FIELD),
                (String) ((Map<String, Object>) contentMap.get(Torrent.INFO_FIELD)).get(Torrent.NAME_FIELD),
                (Integer) ((Map<String, Object>) contentMap.get(Torrent.INFO_FIELD)).get(Torrent.PIECE_LENGTH_FIELD),
                (Integer) ((Map<String, Object>) contentMap.get(Torrent.INFO_FIELD)).get(Torrent.LENGTH_FIELD),
                (String) ((Map<String, Object>) contentMap.get(Torrent.INFO_FIELD)).get(Torrent.PIECES_FILED));
    }

}

{% endhighlight %}
提供了两个方法，一个是根据输入文件制作出一个种子文件，另一个则是从种子文件中直接加载种子。逻辑是比较简单的。

同样，我们可以在单元测试中验证代码的正确性，这里我们用了一首歌（青花瓷）来做测试，之后的博文中也会接着中这个文件来展示如何做种及下载。
{% highlight Java %}
public class TorrentUtilTest {
    @Rule
    public TemporaryFolder folder = new TemporaryFolder();

    @Test
    public void testCreateTorrentAndLoadTorrent() throws Exception {
        ClassLoader classloader = Thread.currentThread().getContextClassLoader();
        Path source = Paths.get(classloader.getResource("青花瓷.mp3").toURI());
        TorrentUtil.createTorrent(source.toFile(), folder.getRoot());

        Torrent t = TorrentUtil.loadTorrentFromFile(new File(folder.getRoot().getAbsolutePath() + "/青花瓷.mp3.torrent"));
        assertEquals("青花瓷.mp3", t.getName());
        assertEquals("localhost:6881", t.getAnnounce());
        assertEquals(3843256, t.getLength());
        assertEquals(1048576, t.getPieceLength());
    }
}
{% endhighlight %}

下篇文章主要介绍tracker，是一个用来维护所有torrent 心跳的一个组件。
