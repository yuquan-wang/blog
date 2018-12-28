---
layout: post
title:  "利用Zookeeper进行主备切换"
date:   2018-12-23 11:49:45 +0200
categories: ZooKeeper
---
众所周知，现在为了提高可用性，经常使用master-slave来解决单点问题，即用master节点来服务，然后slave节点来做备份，当slave节点监测到master节点挂了之后，slave节点会摇身一变变成master节点继续服务。

这样的功能我们可以用zookeeper来实现。以下是对zookeeper介绍的引用：
>“ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services.”

首先我们可以利用zookeeper来进行选主，主要用到的原理是zookeeper内的目录具有唯一性，有且仅有一个客户端能够创建该目录，若该目录已经存在，则会抛出异常，那么自然而然的，我们可以让第一个创建该目录的作为master(即leader), 剩余的作为slave.

简单模拟代码如下：
{% highlight java %}
import org.I0Itec.zkclient.ZkClient;
import org.I0Itec.zkclient.exception.ZkException;

public class ZKClient implements Runnable {
    final private int clientNumber;
    ZKClient(final int clientNumber) {
        this.clientNumber = clientNumber;
    }
    @Override
    public void run() {
        final ZkClient zkClient = new ZkClient("127.0.0.1:2181");
        System.out.println("client " + clientNumber + " try to be the leader");
        try {
            zkClient.createEphemeral("/leader");
        } catch( final ZkException e) {
            System.out.println("Exception for client:" + clientNumber + " " + e.getMessage());
            return;
        }
        System.out.println("client " + clientNumber + " is the leader!");
    }
}
{% endhighlight %}

{% highlight java %}
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ZKMain {

    public static void main(String[] args) {
        final ExecutorService executorService = Executors.newFixedThreadPool(5);
        for(int i=0;i < 5;i++) {
            executorService.submit(new ZKClient(i));
        }
    }
}
{% endhighlight %}

执行前先看一个zookeeper 根目录：
{% highlight bash %}
[zk: localhost:2181(CONNECTED) 26] ls /
[zk-book, zookeeper]
{% endhighlight %}

然后执行main函数，可以看到：
{% highlight bash %}
client 0 try to be the leader
client 4 try to be the leader
client 3 try to be the leader
client 2 try to be the leader
client 1 try to be the leader
client 0 is the leader!
Exception for client:2 org.apache.zookeeper.KeeperException$NodeExistsException: KeeperErrorCode = NodeExists for /leader
Exception for client:4 org.apache.zookeeper.KeeperException$NodeExistsException: KeeperErrorCode = NodeExists for /leader
Exception for client:3 org.apache.zookeeper.KeeperException$NodeExistsException: KeeperErrorCode = NodeExists for /leader
Exception for client:1 org.apache.zookeeper.KeeperException$NodeExistsException: KeeperErrorCode = NodeExists for /leader
{% endhighlight %}

这样client 0 就变成了 leader ！
再看一下zookeeper里的内容：
{% highlight bash %}
[zk: localhost:2181(CONNECTED) 28] ls /
[leader, zk-book, zookeeper]
{% endhighlight %}
可见该目录已经存在zookeeper里面了。

到目前为止我们只利用zookeeper实现了选主功能，还没有实现主备切换。实现主备切换需要利用zookeeper临时节点的一个特性：当创建临时节点的那个客户端断开连接时，zookeeper会自动删除那个临时节点，那么当其他客户端watch这个临时节点的时候，会收到一个 data deleted事件，我们再handle这个事件的时候可以再次进行选主，这样就实现了主备切换。

代码如下，我们让一开始是leader的客户端等待5秒后自动断开连接，以模拟该进程挂掉。只需要对zkclient代码进行一些改动：
{% highlight java %}
import org.I0Itec.zkclient.IZkDataListener;
import org.I0Itec.zkclient.ZkClient;
import org.I0Itec.zkclient.exception.ZkException;

public class ZKClient implements Runnable {
    final private int clientNumber;
    final private ZkClient zkClient;

    ZKClient(final int clientNumber) {
        this.clientNumber = clientNumber;
        zkClient = new ZkClient("127.0.0.1:2181");
    }

    @Override
    public void run() {
        final String leaderPath = "/leader";
        leaderElection(leaderPath);
        zkClient.subscribeDataChanges(leaderPath, new IZkDataListener() {
            @Override
            public void handleDataChange(String dataPath, Object data) throws Exception {
                //do nothing
            }

            public void handleDataDeleted(String path) throws Exception {
                leaderElection(path);
            }
        });

    }

    private void leaderElection(final String path) {
        System.out.println("client " + clientNumber + " try to be the leader");
        try {
            zkClient.createEphemeral(path);
        } catch (final ZkException e) {
            System.out.println("Exception for client:" + clientNumber + " " + e.getMessage());
            return;
        }
        System.out.println("client " + clientNumber + " is the leader!");
        try {
            for (int i = 0; i < 5; i++) {
                Thread.sleep(1000);
                System.out.println("Sleep for one second ................");
            }
        } catch (final InterruptedException e) {
            //do nothing
        }
        zkClient.close();
        System.out.println("client " + clientNumber + " close connection");
    }
}
{% endhighlight %}

执行main函数，可以看到如下输出：
{% highlight bash %}
client 0 try to be the leader
client 1 try to be the leader
client 2 try to be the leader
client 3 try to be the leader
client 1 is the leader!
client 4 try to be the leader
Exception for client:2 org.apache.zookeeper.KeeperException$NodeExistsException: KeeperErrorCode = NodeExists for /leader
Exception for client:4 org.apache.zookeeper.KeeperException$NodeExistsException: KeeperErrorCode = NodeExists for /leader
Exception for client:3 org.apache.zookeeper.KeeperException$NodeExistsException: KeeperErrorCode = NodeExists for /leader
Exception for client:0 org.apache.zookeeper.KeeperException$NodeExistsException: KeeperErrorCode = NodeExists for /leader
Sleep for one second ................
Sleep for one second ................
Sleep for one second ................
Sleep for one second ................
Sleep for one second ................
client 1 close connection
client 4 try to be the leader
client 2 try to be the leader
client 3 try to be the leader
client 0 try to be the leader
client 3 is the leader!
Exception for client:0 org.apache.zookeeper.KeeperException$NodeExistsException: KeeperErrorCode = NodeExists for /leader
Exception for client:4 org.apache.zookeeper.KeeperException$NodeExistsException: KeeperErrorCode = NodeExists for /leader
Exception for client:2 org.apache.zookeeper.KeeperException$NodeExistsException: KeeperErrorCode = NodeExists for /leader
{% endhighlight %}

可以看到，在client 1断掉连接之后，client 3立马成为了新leader，同时client 0,2,4也尝试去变成leader，虽然并没有成功。
