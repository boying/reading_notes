---
title: java nio
tags: java,io,nio
grammar_cjkRuby: true
---

## 概述

### 特点

TODO

### 与标准io的差别

#### 基于Chanel和Buffer操作

标准IO基于字节流和字符流进行操作，NIO基于通道（Chanel）和缓冲区（Buffer）进行操作，数据从通道读取到缓冲区中，或者从缓冲区写入到通道中

#### Non-blocking IO 非阻塞IO


#### 引入Selector选择器

选择器用于监听多个通道的事件（connect可连接、accept可接受、read可读、write可写），**单个线程**可以监听**多个**数据通道

## 核心组件

* Chanel
* Selector
* Buffer

### Chanel

Chanel类似于标准IO中的流，又有区别：
* 既可以从通道中读数据，也可以向通道中写数据。流的读写通常是单向的
* 通道可异步读写
* 通道中的数据总是要先读到一个Buffer，或者从Buffer中写入

#### Chanel的实现

* FileChanel 文件读写
* DatagramChanel 通过UDP读写网络数据
* SocketChanel 通过TCP读写网络数据
* ServerSocketChanel 可监听新进来的TCP连接。对每个新进来的连接都会创建一个SocketChanel

#### FileChanel

```
channel.force(true);
```

FileChannel.force()方法将通道里尚未写入磁盘的数据强制写到磁盘上。于性能方面的考虑，操作系统会将数据缓存在内存中，所以无法保证写入到FileChannel里的数据一定会即时写到磁盘上。要保证这一点，需要调用force()方法。

#### pip

Java NIO 管道是2个线程之间的单向数据连接。Pipe有一个source通道和一个sink通道。数据会被写到sink通道，从source通道读取。

![enter description here][1]

```
Pipe pipe = Pipe.open();
Pipe.SinkChannel sinkChannel = pipe.sink();
sinkChannel.write(buf);

Pipe.SourceChannel sourceChannel = pipe.source();
int bytesRead = sourceChannel.read(buf);

```

### Selector

Selector能够检测多个NIO通道，并能知道通道是否为读写时间做好准备。这样，一个单独的线程可以管理多个chanel。

Selector + Chanel的好处是，减少线程的使用。线程占用资源，线程之间的上下文切换开销大。Selector可以让一个线程来处理多个通道，减少了线程的使用个数。因而，系统的能够处理更多的客户端连接。

#### SelectionKey

当向Selector注册Channel时，register()方法会返回一个SelectionKey对象。

**intrest集合**

```
// 关注哪些事件
int interestSet = selectionKey.interestOps();

// 修改关注事件
```

**ready集合**

```
int readySet = selectionKey.readyOps();

selectionKey.isAcceptable();
selectionKey.isConnectable();
selectionKey.isReadable();
selectionKey.isWritable();

```

**附加对象**

在向Selector注册Chanel时候，可以附上Attachment，这样在从Chanel对应SelectionKey获取该Attachment。例如，可以讲Buffer与Chanel关联起来。

```
chanel.register(selector, SelectionKey.OP_READ, ByteBuffer.allocate(1024));
```

### Buffer

Buffer用于和通道进行交互

#### capacity position limit

![capacity postion limit][2]

#### 常用方法

**flip() 翻转**

将Buffer从写模式切换到读模式。limit = position; position = 0

**rewind() 倒回**

可重读buffer中的数据。postion = 0； limit = limit

**clear()**

清空所有。 postion = 0； limit = capacity

**compact()**

清空已读。未读的数据拷贝到Buffer起始处。然后将position设到最后一个未读元素正后面。



## demo

### echo server

```
package local;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;

/**
 * Created by jiangzhiwen on 17/1/14.
 */
public class EchoServer {
    public static void main(String[] args) throws IOException {
        Selector selector = Selector.open();

        ServerSocketChannel channel = ServerSocketChannel.open();
        channel.socket().bind(new InetSocketAddress(9999));
        channel.configureBlocking(false);
        channel.register(selector, SelectionKey.OP_ACCEPT);

        while (true) {
            if (selector.select() == 0) {
                continue;
            }

            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                iterator.remove();

                if (key.isAcceptable()) {
                    System.out.println("acceptble");
                    ServerSocketChannel c = (ServerSocketChannel) key.channel();
                    SocketChannel accept = c.accept();
                    accept.configureBlocking(false);
                    accept.register(selector, SelectionKey.OP_READ, ByteBuffer.allocate(1024));

                } else if (key.isReadable()) {
                    System.out.println("readable");

                    SocketChannel c = (SocketChannel) key.channel();
                    ByteBuffer buffer = (ByteBuffer) key.attachment();

                    int n = c.read(buffer);
                    if(n == -1){ // end of stream
                        key.cancel();
                        //c.close(); TODO
                        continue;
                    }
                    while (n > 0) {
                        n = c.read(buffer);
                    }
                    key.interestOps(SelectionKey.OP_WRITE);

                } else if (key.isWritable()) {
                    System.out.println("wirtable");

                    ByteBuffer buffer = (ByteBuffer) key.attachment();
                    buffer.flip();
                    SocketChannel c = (SocketChannel) key.channel();
                    c.write(buffer);
                    buffer.clear();

                    key.interestOps(SelectionKey.OP_READ);

                } else if (key.isConnectable()) {
                    System.out.println("connectable");
                }

            }

        }


    }
}

```

### echo client

```
package local;

import java.io.IOException;
import java.net.InetAddress;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.CharBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;
import java.nio.charset.Charset;
import java.util.Iterator;
import java.util.Scanner;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingDeque;

/**
 * Created by jiangzhiwen on 17/1/14.
 */
public class EchoClient {

    static class InputThread extends Thread {
        private BlockingQueue<String> queue;

        public InputThread(BlockingQueue<String> queue) {
            this.queue = queue;
        }

        @Override
        public void run() {
            super.run();

            Scanner sc = new Scanner(System.in);
            while (true) {
                try {
                    String str = sc.nextLine();
                    System.out.println("std in " + str);
                    queue.put(str);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public static void main(String[] args) throws IOException, InterruptedException {
        BlockingQueue<String> queue = new LinkedBlockingDeque<String>();
        InputThread inputThread = new InputThread(queue);
        inputThread.start();

        Selector selector = Selector.open();

        SocketChannel socketChannel = SocketChannel.open();
        socketChannel.configureBlocking(false);

        socketChannel.register(selector, SelectionKey.OP_CONNECT);

        socketChannel.connect(new InetSocketAddress(InetAddress.getLocalHost(), 9999));


        while (true) {
            int select = selector.select();
            System.out.println(select);

            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            while (iterator.hasNext()) {

                SelectionKey key = iterator.next();
                SocketChannel chanel = (SocketChannel) key.channel();
                if (key.isConnectable()) {
                    if (chanel.isConnectionPending()) {
                        chanel.finishConnect();

                        key.interestOps(SelectionKey.OP_WRITE);
                    }
                } else if (key.isWritable()) {

                    String str = queue.take();
                    if(str.equals("")){
                        continue;
                    }
                    System.out.println("queue str is " + str);
                    ByteBuffer buffer = ByteBuffer.allocate(1024);
                    buffer.put(str.getBytes(Charset.forName("UTF-8")));
                    buffer.flip();
                    chanel.write(buffer);

                    key.interestOps(SelectionKey.OP_READ);
                } else if (key.isReadable()) {
                    ByteBuffer buffer = ByteBuffer.allocate(1024);
                    chanel.read(buffer);
                    buffer.flip();
                    CharBuffer charbuffer = Charset.forName("UTF-8").newDecoder().decode(buffer);
                    String s = charbuffer.toString();
                    System.out.println(s);


                    key.interestOps(SelectionKey.OP_WRITE);
                }

                iterator.remove();
            }

        }
    }
}

```


  [1]: ./assets/1484503043720.jpg "1484503043720.jpg"
  [2]: ./assets/buffers-modes.png