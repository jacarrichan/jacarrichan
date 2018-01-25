title: dubbo provider 暴露错误的IP
date: 2017-9-13 11:42:56
tags:  [dubbo,Zookeeper,ping,hostname]
---

最近运维童鞋在部署服务的时候发现dubbo consumer调用服务的时候总是失败，根据异常看，大致是dubbo consum 调用服务的时候，访问一个生产环境不存在IP。

<!-- more -->

# 表现的异常信息如下：

```
com.alibaba.dubbo.remoting.RemotingException: client(url: dubbo://116.251.209.183:60010/com.xxx.xxx?anyhost=true&application=xx-provider&check=false&codec=dubbo&default.check=false&default.retries=0&default.timeout=20000&dubbo=2.5.3&heartbeat=60000&interface=com.xxx.xxx&methods=handler&pid=16882&side=consumer&timeout=5000&timestamp=1502874179375) failed to connect to server /116.251.209.183:60010 client-side timeout 3000ms (elapsed: 3003ms) from netty client 172.16.200.4 using dubbo version 2.4.9
at com.alibaba.dubbo.remoting.transport.netty.NettyClient.doConnect(NettyClient.java:127)
at com.alibaba.dubbo.remoting.transport.AbstractClient.connect(AbstractClient.java:280)
at com.alibaba.dubbo.remoting.transport.AbstractClient.(AbstractClient.java:103)
at com.alibaba.dubbo.remoting.transport.netty.NettyClient.(NettyClient.java:61)
at com.alibaba.dubbo.remoting.transport.netty.NettyTransporter.connect(NettyTransporter.java:37)
at com.alibaba.dubbo.remoting.Transporter$Adpative.connect(Transporter$Adpative.java)
at com.alibaba.dubbo.remoting.Transporters.connect(Transporters.java:67)
....

```

# 排错过程

开始以为是系统被攻击或者中病毒了导致，各种PS、 各种分析日志、各种查看机器负载情况后，排除了这个问题。

找到provider注册到Zookeeper所在的节点，发现注册在Zookeeper里面IP就是这个有问题的IP。也就是说provider机器启动的时候获取的本地IP是有问题。

一个同事提出尝试ping provider机器hostname，出现了这个问题IP。这时联想到dubbo  启动的时候 **会不会有类似根据hostname获取IP的机制呢？**
翻看dubbo，确实是有。

 [ServiceConfig.java]
 
```
...

    private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
        String name = protocolConfig.getName();
        if (name == null || name.length() == 0) {
            name = "dubbo";
        }

        String host = protocolConfig.getHost();
        if (provider != null && (host == null || host.length() == 0)) {
            host = provider.getHost();
        }
        boolean anyhost = false;
        if (NetUtils.isInvalidLocalHost(host)) {
            anyhost = true;
            try {
                host = InetAddress.getLocalHost().getHostAddress();
            } catch (UnknownHostException e) {
                logger.warn(e.getMessage(), e);
            }
            if (NetUtils.isInvalidLocalHost(host)) {
                if (registryURLs != null && registryURLs.size() > 0) {
                    for (URL registryURL : registryURLs) {
                        try {
                            Socket socket = new Socket();
                            try {
                                SocketAddress addr = new InetSocketAddress(registryURL.getHost(), registryURL.getPort());
                                socket.connect(addr, 1000);
                                host = socket.getLocalAddress().getHostAddress();
                                break;
                            } finally {
                                try {
                                    socket.close();
                                } catch (Throwable e) {
                                }
                            }
                        } catch (Exception e) {
                            logger.warn(e.getMessage(), e);
                        }
                    }
                }
                if (NetUtils.isInvalidLocalHost(host)) {
                    host = NetUtils.getLocalHost();
                }
            }
        }

...

```
* 代码地址参见： [ServiceConfig.java](https://github.com/alibaba/dubbo/blob/master/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ServiceConfig.java) 

# 分析问题出现的原因

出现问题的代码就是***InetAddress.getLocalHost().getHostAddress();***，此代码的执行的结果和ping hostname的结果是一样的。

翻看JDK中对应的代码，确认了它有使用*sun.net.spi.nameservice.dns.DNSNameService*;



# 解决办法

配置hostname的回环地址，使ping hostname得到的结果是127.0.0.1。

# 附注

以下为运维童鞋整理的：

这两台有问题的主机，是我们后期加的，gateway-client需要和很多外部交互，所以有很多解析上的配置，新加的机器直接复制之前主机的hosts文件，导致hosts文件中主机名和自身主机名不一致，当时并不曾想得到，hostname这里还有这种坑，所以就没管hostname。
另一方面，hostname解析过程中云析还会将xx.cmbcloud.com 加在hostname后面去解析，这是更没有想到，然后就得到了这个116.251.209.183 IP地址。


```
F:\temp\mvn\jacarrichan.github.io>ping cmbcloud.com

正在 Ping cmbcloud.com [116.251.209.183] 具有 32 字节的数据:
来自 116.251.209.183 的回复: 字节=32 时间=83ms TTL=53
来自 116.251.209.183 的回复: 字节=32 时间=106ms TTL=53
来自 116.251.209.183 的回复: 字节=32 时间=76ms TTL=53
来自 116.251.209.183 的回复: 字节=32 时间=68ms TTL=53

116.251.209.183 的 Ping 统计信息:
    数据包: 已发送 = 4，已接收 = 4，丢失 = 0 (0% 丢失)，
往返行程的估计时间(以毫秒为单位):
    最短 = 68ms，最长 = 106ms，平均 = 83ms

```