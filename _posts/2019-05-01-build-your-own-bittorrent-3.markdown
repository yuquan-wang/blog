---
layout: post
title:  "从头写一个bittorrent(3)-Tracker"
date:   2019-05-01 05:49:45 +0900
categories: bittorrent
---
从[第一篇文章](https://yuquan-wang.github.io/blog/2019/01/11/build-your-own-bittorrent-1.html)开始，打算写一系列文章来说明，如何从头开始实现一个bittorrent.具体代码在[build-your-own-bittorrent](https://github.com/yuquan-wang/build-your-own-bittorrent)中

BitTorrent完整的协议在这里：[BitTorrent protocal specification](https://wiki.theory.org/index.php/BitTorrentSpecification),差不多这个教程就是跟着协议一步步写下来，做一个简单版本的bittorrent系统。

上文已经介绍了如何根据文件来制作一个torrent文件，本篇文章主要介绍一个全新的组件，名叫Tracker, 顾名思义，就是追踪器， 众所周知bittorrent是一个分布式下载系统，不会因为某个做种的机器坏掉而影响整个下载，这其中的关键就在于这个追踪器。

来看一个Tracker的定义：
> The tracker is an HTTP/HTTPS service which responds to HTTP GET requests. The requests include metrics from clients that help the tracker keep overall statistics about the torrent. The response includes a peer list that helps the client participate in the torrent.

简要说明一下Tracker的作用，TorrentClient(不管是seeding还是download),都会定期向Tracker汇报行踪， 如
  1. TorrentClient1 对文件进行做种， 然后汇报给Tracker,我有文件的所有数据（假设piece 1-10）
  2. TorrentClient2 已经下载了一部分文件， 于是汇报给Tracker,我有文件的部分数据(假设piece 2-5)
  3. TorrentClient3 开始要下载piece5, 此时它会问Tracker,要我piece5,我该问谁去要，那么Tracker就会回复它，client1和client2可能有该数据，你可以问他们去要。
  4. client3首先尝试去问client1去要数据，成功了则该piece的下载结束，若client1因为莫名原因挂掉了，则client3会尝试去问client2下载该数据，这样就避免了集中式下载的单点问题。
  5. 要注意，Tracker只是一个追踪器，只负责告诉客户端哪些Peer有数据，但是并不负责数据的下载。


下面我们来实现该Tracker, 它实际上只需要实现一个接口，即Announce函数，官方定义中定义了request的各种参数和response的各个field,这里我们只选取一些非常必要的进行实现，必要的请求参数如下：
  1. info_hash:代表这是对应哪个torrent
  2. port: 该peer的哪个端口供数据下载(ip可以直接从请求中推测出来)

返回的内容我们也进行了精简，只返回哪些peer含有当前请求数据，peer主要指ip和port, 这样torrentclient就可以向它请求数据了。


下面就是Tracker的实现，在代码中我已经进行了详细的注释，当然也乘这个机会练习了一下使用Netty(所以TrackerServer是基于Netty的)


{% highlight java %}
public class TrackerServer {

    static final int PORT = 6881;

    // key is info_hash(每个torrent都不同), value is list peers, containing key ip and port
    // 注意Tracker是把整个集群的信息都保存在内存中的，也不用担心内存的不可靠性，因为torrentclient是每隔10/15秒就会进行announce,然后tracker就可以在一个进行内存更新
    private static final Map<String, List<Map<String, String>>> TORRENT_MAP = new HashMap<>();

    public void start() throws InterruptedException {
        // Configure the server.
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.option(ChannelOption.SO_BACKLOG, 1024);
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .handler(new LoggingHandler(LogLevel.INFO))
                    .childHandler(new HttpTrackerServerInitializer());

            Channel ch = b.bind(PORT).sync().channel();

            ch.closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    static class HttpTrackerServerInitializer extends ChannelInitializer<SocketChannel> {

        @Override
        protected void initChannel(SocketChannel ch) throws Exception {
            ChannelPipeline p = ch.pipeline();
            p.addLast(new HttpServerCodec());
            p.addLast(new HttpServerExpectContinueHandler());
            p.addLast(new HttpTrackerServerHandler());
        }
    }

    static class HttpTrackerServerHandler extends SimpleChannelInboundHandler<HttpObject> {

        private static final AsciiString CONTENT_TYPE = AsciiString.cached("Content-Type");
        private static final AsciiString CONTENT_LENGTH = AsciiString.cached("Content-Length");

        @Override
        public void channelReadComplete(ChannelHandlerContext ctx){
            ctx.flush();
        }

        @Override
        protected void channelRead0(ChannelHandlerContext ctx, HttpObject msg) throws Exception {
            if(msg instanceof HttpRequest){
                HttpRequest req = (HttpRequest)msg;
                String requestUri = req.uri();
                FullHttpResponse response;
                if(requestUri.startsWith("/announce")){  //只处理announce请求
                    String ipAddress = getIpAddress(ctx);
                    System.out.println("request from: " + ipAddress );
                    List<NameValuePair> params = URLEncodedUtils.parse(new URI(requestUri), Charset.forName("UTF-8"));
                    String port = "";
                    String info_hash = "";
                    for (NameValuePair param : params) {
                        System.out.println(param.getName() + " : " + param.getValue());
                        if(param.getName().equals("port")){
                            port = param.getValue(); //拿到请求中的port参数
                        }else if(param.getName().equals("info_hash")) {
                            info_hash = param.getValue(); //拿到请求中的info_hash参数
                        }
                    }
                    if(port.isEmpty()||info_hash.isEmpty()) {
                        response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.BAD_REQUEST,
                                Unpooled.wrappedBuffer("Please pass info_hash or port".getBytes()));
                    }else{
                        List<Map<String,String> > peers = new ArrayList<>();
                        if(TORRENT_MAP.containsKey(info_hash)){
                            peers = TORRENT_MAP.get(info_hash); //返回当前有哪些peer函数这个数据
                        }
                        Bencode bencode = new Bencode();
                        response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.OK,
                                Unpooled.wrappedBuffer(bencode.encode(peers).getBytes()));
                        addToTorrentMap(info_hash, ipAddress, port); //最后要把当前peer也加入到Tracker的追踪map中
                    }
                }else{
                    response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.NOT_FOUND,
                            Unpooled.wrappedBuffer("Not supported".getBytes()));
                }

                response.headers().set(CONTENT_TYPE, "text/plain");
                response.headers().setInt(CONTENT_LENGTH, response.content().readableBytes());
                ctx.write(response).addListener(ChannelFutureListener.CLOSE);
            }
        }

        private void addToTorrentMap(String info_hash, String ipAddress, String port) {
            if(!TORRENT_MAP.containsKey(info_hash)) {
                TORRENT_MAP.put(info_hash, new ArrayList<>());
            }
            List<Map<String, String>> peers = TORRENT_MAP.get(info_hash);
            boolean exist = false;
            for(Map<String, String> peer: peers){
                if(peer.get("ip").equals(ipAddress) && peer.get("port").equals(port)) {
                    exist = true;
                }
            }
            if(!exist){
                Map<String,String> newPeer = new HashMap<>();
                newPeer.put("ip", ipAddress);
                newPeer.put("port", port);
                peers.add(newPeer);
            }
        }

        private String getIpAddress(ChannelHandlerContext ctx){
            InetSocketAddress socketAddress = (InetSocketAddress) ctx.channel().remoteAddress();
            InetAddress inetaddress = socketAddress.getAddress();
            String ipAddress = inetaddress.getHostAddress(); // IP address of client
            return ipAddress;
        }

    }

    public static void main(String[] args) throws InterruptedException {
        TrackerServer trackerServer = new TrackerServer();
        trackerServer.start();
    }
}
{% endhighlight %}

有了TrackerServer,那么自然而然需要有TrackerClient,这样对任何torrentClient他们就可以轻松的发送信息到TrackerServer了，上文已经说过Announce的时候只需要hash_info和port这两个关键信息，那么我们可以简单的实现出下面的TrackerClient.
{% highlight java %}
public class TrackerClient {
    final static String TRACKER_URI = "http://localhost:6881/announce";
    final static CloseableHttpClient httpClient = HttpClients.createDefault();
    final static String PORT = "port";
    final static String INFO_HASH = "info_hash";

    public List<Map<String, String>> announceToTracker(String infoHash, int port) throws Exception {
        HttpGet httpget = new HttpGet(TRACKER_URI + "?" + PORT + "=" + port + "&" + INFO_HASH + "=" + infoHash);
        CloseableHttpResponse response = httpClient.execute(httpget);
        String responseString = IOUtils.toString(response.getEntity().getContent());
        Bencode bencode = new Bencode();
        List<Map<String, String>> peers = (List<Map<String, String>>) bencode.decode(responseString);
        return peers;
    }
}
{% endhighlight %}
