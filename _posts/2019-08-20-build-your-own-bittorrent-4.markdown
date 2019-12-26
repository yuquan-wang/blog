---
layout: post
title:  "从头写一个bittorrent(4)-TorrentClient"
date:   2019-08-20 05:49:45 +0900
categories: bittorrent
---
从[第一篇文章](https://yuquan-wang.github.io/blog/2019/01/11/build-your-own-bittorrent-1.html)开始，打算写一系列文章来说明，如何从头开始实现一个bittorrent.具体代码在[build-your-own-bittorrent](https://github.com/yuquan-wang/build-your-own-bittorrent)中

BitTorrent完整的协议在这里：[BitTorrent protocal specification](https://wiki.theory.org/index.php/BitTorrentSpecification),差不多这个教程就是跟着协议一步步写下来，做一个简单版本的bittorrent系统。

前一篇文章已经介绍了Tracker是怎么工作的，这篇文章将要介绍最重要的TorrentClient,即具体基于torrent的下载是怎么一步一步工作的，上面的协议其实介绍了很多peer和peer之间是怎么工作的，但本文只关注最核心的下载部分，其余优化暂时略过。

重新回顾下前一篇文章提到的流程：
  1. TorrentClient1 对文件进行做种， 然后汇报给Tracker,我有文件的所有数据（假设piece 1-10）
  2. TorrentClient2 已经下载了一部分文件， 于是汇报给Tracker,我有文件的部分数据(假设piece 2-5)
  3. TorrentClient3 开始要下载piece5, 此时它会问Tracker,要我piece5,我该问谁去要，那么Tracker就会回复它，client1和client2可能有该数据，你可以问他们去要。
  4. client3首先尝试去问client1去要数据，成功了则该piece的下载结束，若client1因为莫名原因挂掉了，则client3会尝试去问client2下载该数据，这样就避免了集中式下载的单点问题。

从这个流程中我们可以看到，TorrentClient主要有两种功能：
  1. 做种(seeding),然后当别的client来询问下载数据的时候，负责把数据发送给其他peer
  2. 下载(download),从Tracker问来哪些peer有当前想要的数据，然后单独找那些peer要数据。


我们可以先定义如下的一个类， 来表示TorrentClient:
{% highlight java %}
public class TorrentClient {
    private Torrent torrent; //对应的torrent
    private TrackerClient trackerClient = new TrackerClient(); // 用来汇报给tracker
    private File location; //数据所在位置,ifFile()==true代表是seed状态，若传入一个文件夹则表示要下载到该文件夹下
    private State state; //代表当前状态是seeding还是downloading
    private BitSet downloaded; //用来记录哪些piece被下载了
    private List<Map<String, String>> peers = new ArrayList<>();//记录Tracker的回复，也就是该问哪些peer去要数据
    private int port = new Random().nextInt(1000) + 3000; //当前做种的端口
    final static CloseableHttpClient httpClient = HttpClients.createDefault(); //问其他peer要数据用

    public TorrentClient(Torrent torrent, File location) {
        this.torrent = torrent;
        this.location = location;
        if (location.isFile()) {
            state = State.seeding; //若传进来一文件，我们认为是直接做种
        } else {
            state = State.downloading;//否则我们认为是需要下载到当前文件夹下(更好的办法是对数据进行分析，如果完全符合torrent则是seeding,不符合则是downloading)
            try {
                File targetFile = new File(location, torrent.getName());
                targetFile.createNewFile();
                RandomAccessFile raf = new RandomAccessFile(targetFile, "rw");
                raf.setLength(torrent.getLength());
            } catch (IOException e) {
                System.err.println("Failed to create target file for torrent");
            }
        }
        downloaded = new BitSet(torrent.getPieceLength());
    }

    private void startAnnounceThread() {
        //不管是下载还是做种，都需要进行announce
        AnnounceThread announceThread = new AnnounceThread();
        announceThread.start();
    }

    public void start() {
        startAnnounceThread();
        if (state == State.seeding) {
            seed();
        } else {
            download();
        }
    }

    private void seed() {
        SeedThread seedThread = new SeedThread();
        seedThread.start();
    }

    //blocking implementation, of course we could change it to non blocking and add call back after download
    private void download() {
        DownloadThread downloadThread = new DownloadThread();
        downloadThread.start();

    }
}
{% endhighlight %}

上面的框架应该也比较容易懂，即如果是seeding状态，我们就开启SeedThread, 否则就开始DownloadThread,当然AnnouceThread是两种状态都需要的。

首先我们来看两种状态都需要的AnnounceThread,功能很简单，即定时向Tracker发送心跳数据：
{% highlight java %}
public class AnnounceThread extends Thread {

    @Override
    public void run() {
        while (true) {
            try {
                peers = trackerClient.announceToTracker(torrent.getInfoHash(), port);
                Thread.sleep(15 * 1000);
            } catch (InterruptedException e) {
                Thread.interrupted();
                e.printStackTrace();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
{% endhighlight %}

下面是SeedThread的实现，功能是根据请求返回相应的piece数据，一样这里我们用了Netty.
{% highlight java %}
public class SeedThread extends Thread {
        @Override
        public void run() {
            EventLoopGroup bossGroup = new NioEventLoopGroup(1);
            EventLoopGroup workerGroup = new NioEventLoopGroup();
            try {
                ServerBootstrap b = new ServerBootstrap();
                b.option(ChannelOption.SO_BACKLOG, 1024);
                b.group(bossGroup, workerGroup)
                        .channel(NioServerSocketChannel.class)
                        .handler(new LoggingHandler(LogLevel.INFO))
                        .childHandler(new SeedingServerInitializer());

                Channel ch = b.bind(port).sync().channel();

                ch.closeFuture().sync();
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                bossGroup.shutdownGracefully();
                workerGroup.shutdownGracefully();
            }
        }


        class SeedingServerInitializer extends ChannelInitializer<SocketChannel> {

            @Override
            protected void initChannel(SocketChannel ch) throws Exception {
                ChannelPipeline p = ch.pipeline();
                p.addLast(new HttpServerCodec());
                p.addLast(new HttpServerExpectContinueHandler());
                p.addLast(new SeedingServerHandler());
            }
        }

        class SeedingServerHandler extends SimpleChannelInboundHandler<HttpObject> {

            private final AsciiString CONTENT_TYPE = AsciiString.cached("Content-Type");
            private final AsciiString CONTENT_LENGTH = AsciiString.cached("Content-Length");
            private final Map<String, List<Pair<String, String>>> torrentMap = new HashMap<>(); // key info_hash, value is list peers, containing key ip and port


            @Override
            public void channelReadComplete(ChannelHandlerContext ctx) {
                ctx.flush();
            }

            @Override
            protected void channelRead0(ChannelHandlerContext ctx, HttpObject msg) throws Exception {
                if (msg instanceof HttpRequest) {
                    HttpRequest req = (HttpRequest) msg;
                    String requestUri = req.uri();
                    FullHttpResponse response;
                    if (requestUri.startsWith("/piece")) {
                        List<NameValuePair> params = URLEncodedUtils.parse(new URI(requestUri), Charset.forName("UTF-8"));
                        String requestedPiece = "";
                        for (NameValuePair param : params) {
                            if (param.getName().equals("q")) { //获取请求的piece
                                requestedPiece = param.getValue();
                            }
                        }
                        if (requestedPiece.isEmpty()) {
                            response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.OK,
                                    Unpooled.wrappedBuffer("Please pass info_hash or port".getBytes()));
                        } else {
                            int pieceIndex = Integer.valueOf(requestedPiece);
                            int pieceLength = torrent.getPieceLength();
                            FileInputStream fileInputStream = new FileInputStream(location);
                            fileInputStream.skip(pieceLength * pieceIndex); //从数据文件中定位请求的数据，然后以byte array发送回去
                            byte[] contents = IOUtils.toByteArray(fileInputStream, pieceLength * (pieceIndex + 1) > torrent.getLength() ? torrent.getLength() - pieceLength * pieceIndex : pieceLength);
                            response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.OK,
                                    Unpooled.wrappedBuffer(contents));

                        }
                    } else {
                        response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.NOT_FOUND,
                                Unpooled.wrappedBuffer("Not supported".getBytes()));
                    }

                    response.headers().set(CONTENT_TYPE, "text/plain");
                    response.headers().setInt(CONTENT_LENGTH, response.content().readableBytes());
                    ctx.write(response).addListener(ChannelFutureListener.CLOSE);
                }
            }
        }

    }
{% endhighlight %}


最后则是相对最复杂的DownloadState了，主要功能如下：
  1. 检测当前状态，如果所有piece都下载完了，则结束，否则进入第二步
  2. 选取当前没下载的piece
  3. 问其他的peer要该piece数据，成功了则更新该piece的数据，否则就轮寻其他peer

以下是简要实现：
{% highlight java %}
public class DownloadThread extends Thread{
        @Override
        public void run(){
            int failCnt = 0; // return false when fail more than 10 times
            while (!isDownloadFinished()) {
                int undownloadedPiece = pickUndownloadedPiece();
                Pair<String, String> peer = pickPeerForDownload();
                if (peer != null && downloadPiece(peer, undownloadedPiece)) {
                    downloaded.set(undownloadedPiece);
                    failCnt = 0;
                } else {
                    failCnt++;
                }
                if (failCnt > 10) {
                    System.err.println("Failed to download. Exit");
                    return;
                }
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            System.out.println("Download finished...Yeah!");
        }
    }


    private Boolean downloadPiece(Pair<String, String> peer, int undownloadedPiece){
        try {
            String host = peer.getKey();
            String port = peer.getValue();
            HttpGet httpget = new HttpGet("http://"+ host + ":" + port + "/piece?q=" + undownloadedPiece);
            CloseableHttpResponse response = httpClient.execute(httpget);
            HttpEntity entity = response.getEntity();
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            entity.writeTo(baos);
            byte[] contents = baos.toByteArray();
            byte[] sha1 = DigestUtils.sha1(contents);
            String downloadedPiece = new String(sha1, Charset.forName("UTF-16"));
            int skippedLength = 10* undownloadedPiece ; // each piece responding to 10 bytes
            String pieceInTorrent = torrent.getPieces().substring(skippedLength, skippedLength+Math.min(torrent.getPieces().length() - skippedLength, 10));
            if (pieceInTorrent.equals(downloadedPiece)) { //下载完要进行hash验证，以防做种方出错或者网络出错
                //successful download
                try(RandomAccessFile raf = new RandomAccessFile( new File(location, torrent.getName()), "rw");) {
                    raf.skipBytes(torrent.getPieceLength()*undownloadedPiece);
                    raf.write(contents);
                }
                System.out.println("Successfully downloaded piece "  + undownloadedPiece + " from peer " + peer);
                return true;
            } else {
                //downloaded content is not what we want
                return false;
            }
        }catch(IOException e){
            return false;
        }
    }

    private Boolean isDownloadFinished() {
        int pieceNumber = (int)Math.ceil(1.0* torrent.getLength()/torrent.getPieceLength());
        for (int i = 0; i < pieceNumber; i++) {
            if (!downloaded.get(i)) {
                return false;
            }
        }
        return true;
    }

    private int pickUndownloadedPiece() {
        for (int i = 0; i < torrent.getPieceLength(); i++) {
            if (!downloaded.get(i))
                return i;
        }
        return -1;
    }
{% endhighlight %}



最后我们来跑一下程序，验证程序的可行性，主要需要提前把TrackerServer跑起来， 下面是TorrentClientTest的代码：
{% highlight java %}
public class TorrentClientTest {

    public static void main(String[] args) throws Exception {
        ClassLoader classloader = Thread.currentThread().getContextClassLoader();
        Path source = Paths.get(classloader.getResource("青花瓷.mp3").toURI());
        TorrentUtil.createTorrent(source.toFile(), source.getParent().toFile());

        Torrent t = TorrentUtil.loadTorrentFromFile(new File(source.getParent().toFile().getAbsolutePath() + "/青花瓷.mp3.torrent"));

        File target = new File("/tmp/testClient");
        target.mkdir();

        Thread trackerThread = new Thread(){
            public void run(){
                TrackerServer trackerServer = new TrackerServer();
                try {
                    trackerServer.start();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };
        trackerThread.start();
        Thread.sleep(5000);

        TorrentClient torrentClient1 = new TorrentClient(t, source.toFile());
        TorrentClient torrentClient2 = new TorrentClient(t, target);
        torrentClient1.start();
        torrentClient2.start();
        Thread.sleep(1000000);
    }
}
{% endhighlight %}

可以看到运行时的日志如下：
{% highlight java %}
Successfully downloaded piece 0 from peer 127.0.0.1=3093
Successfully downloaded piece 1 from peer 127.0.0.1=3093
Successfully downloaded piece 2 from peer 127.0.0.1=3093
Successfully downloaded piece 3 from peer 127.0.0.1=3093
{% endhighlight %}

因为我们只有一个TorrentClient在做种，所以下载都是问该peer要数据，若有多个做种，则可以看到它会问多个peer要数据，即避免了单点问题。

可以做的一些优化：
  1. 随机下载piece,不需要顺序下载。
  2. 并行下载，多个线程分别下载不同piece.
  3. 如果数据量大，优先去下载那些做种少的piece.
  4. 负载平衡，避免多个peer都问同一个peer要数据。
