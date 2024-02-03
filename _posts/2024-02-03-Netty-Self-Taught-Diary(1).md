---
layout: post
title:  "Netty Self-taught Diary"
date:   2024-02-03 16:00:55 +0800
categories: jekyll update
---
### Intro
本來很久之前就想自己寫一個聊天程式自己用，但是一直苦於Netty官網上艱澀的User Guide

自己本身學的是java，但之前對Java IO只有些許瞭解，再加上完全沒有使用過原生IO，所以一直不得其門

最近有空想説再來學習一下Netty并且自己建立一個項目學習，如有不對之處望指正！

### What is Netty?
```
Netty is an NIO client server framework which enables quick and easy development of network applications
such as protocol servers and clients. It greatly simplifies and streamlines network programming such
as TCP and UDP socket server.
```
簡單來説Netty可以用在處理高并發的IO項目中，不管是Sevrer/Client都可以適用

### Netty components
首先Netty設計了幾個特殊的、包裝過的類；借由這些類來實現高并發和提升性能:

- Channel: 對應Java API裏的Socket
- ChannelFuture: 在Netty中很多異步的Channel並不會馬上返回，因此需要這樣一個ChannelFuture類來表示返回類型
- ChannelHandler: 是開發者使用Netty時最專注的模塊，因爲幾乎所有BL都是在這邊實現，有點類似于Interceptor，只不過細部概念後續説明，這邊只要知道它是處理ChannelPipeline中流通的events並將處理結果傳給後續的Handler
- ChannelPipeline: 每一個Channel在生成的時候都會指派一個屬於自己的ChannelPipeline用於處理在這個Channel中所有的Handler和Events；注意光看命名其實容易搞混，可以理解爲Channel擁有的Pipeline,換句話説是存在于Channel裏的
- ChannelHandlerContext: 可以理解表示ChannelHandler與ChannelPipeline之間的關係，也可以從中獲取ChannelPipeline依附的Channel，但通常來説這個類用來處理outbound數據
- Inbound/Outbound: 簡單來説就是數據流通方向，若是往外流就是Outbound；往内流就是Inbound。例如在Client端中若要設計一個ChannelHandler用於處理從Server發送的數據，那麽就應該繼承ChannelInboundHandler/Adapter類似的類
- EventLoopGroup: 在Netty中一個EventLoopGroup擁有多個EventLoop，而一個EventLoop永遠對應一個Thread，并且一個Channel在它的整個生命周期中只綁定一個EventLoop；但相反的一個EventLoop可以擁有多個Channel。這點與傳統Java IO不同，一個Channel通常只對應一個Thread，若要使用不堵塞則使用threadPool，但在Java IO中一個thread也只能對應一個channel

### 0. Build Server
通過上面的介紹，應該對其中幾個類都有一定的瞭解。若要建立一個最簡單的Server則需要兩個部分：

- ChannelHandler: 處理流通在ChannelPipeline上的所有事件(event)，例如最簡單的當Client發送訊息時，Server接受到之後想要返回一段特殊字串，那就需要在ChannelHandler裏實現
- Bootstrap: Server端的啓動流程，包括綁定端口、初始化EventLoopGroup，以及各種定義如若是Connection生成后Channel的類/ChannelPipeline上有哪些ChannelHandler以及他們的順序等等

不多說，直接看代碼來學習是最快的方法，下面實現的是一個返回Client端發送訊息的Server:
- EchoServerHandler.java


```
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelFutureListener;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.CharsetUtil;

public class EchoServerHandler extends ChannelInboundHandlerAdapter{
    
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg){
        ByteBuf in = (ByteBuf) msg;
        System.out.println(
            "Server received:" + in.toString(CharsetUtil.UTF_8)
        );
        ctx.write(in);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx){
        ctx.writeAndFlush(Unpooled.EMPTY_BUFFER).addListener(ChannelFutureListener.CLOSE);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause){
        cause.printStackTrace();
        ctx.close();
    }
}
```

- EchoServer.java


```
import java.net.InetSocketAddress;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;

public class EchoServer {
    private static int port = 8080;

    public static void main(String[] args) throws Exception{
        EchoServer.start();
    }

    public static void start() throws Exception {
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(group)
            .channel(NioServerSocketChannel.class)
            .localAddress(new InetSocketAddress(port))
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch){
                    ch.pipeline().addLast(new EchoServerHandler());
                }
            });
            ChannelFuture cf = b.bind().sync();
            cf.channel().closeFuture().sync();

        } finally {
            group.shutdownGracefully().sync();
        }
    }
}
```

首先代碼非常簡單，如同前面所説分爲兩個部分：**ChannelHandler** 實現了一個 **ChannelInboundHandlerAdapter** 類用於幫助生成其他基本的方法，
這裏覆寫了三個方法分別是: 
- **channelRead()** 用於處理接收到的數據
- **channelReadComplete()** 用於表示在 channelRead()方法結束后要做什麽
- **exceptionCaught()** 表示若有異常在這個ChannelHandler中發生要怎麽處理，是要直接關閉這個channel還是如何

再來看到 **Bootstrap** 部分:
- **ServerBootstrap**: 這是Netty提供的初始啓動輔助類，幫助使用者簡化流程，并將許多類挂載到這上面
- **EventLoopGroup group;** & **b.group(group)**: 如同前面所提到的將一個EventLoop分配給Channel
- **.channel(NioServerSocketChannel.class)**: 指定Channel的類型
- **.localAddress(new InetSocketAddress(port))**: 綁定這個Server要在哪個端口監聽請求
- **.childHandler(...)**: 將前面定義的Channel上的所有ChannelHandler並將他們加到自己的ChannelPipeline上。注意這個 **addLast()** 中可
以添加多個 **ChannelHandler**，并且他們之間的順序也是添加的順序。
- **ChannelFuture cf = b.bind().sync()**: 啓動監聽并且將這個步驟設置為同步堵塞(**sync**)，也就是這行代碼沒有執行完之前不會執行後面
的代碼。因爲這個方法並不是馬上傳回值，因此這個方法返回的是 **ChannelFuture**
- 最後若是前面堵塞的代碼結束，表示Channel要關閉，因此執行channel.closeFuture()

## 學習資料
1. [Netty in Action.pdf](https://github.com/zuzeep/book/blob/master/Netty%20in%20Action.pdf)
2. [Netty Ofiicial](https://netty.io/wiki/user-guide-for-4.x.html)
