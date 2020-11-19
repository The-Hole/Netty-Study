# Netty学习（一）

###  1 、 初识什么是BIO NIO 

#### 1.1 什么是BIO

Java的BIO是一种**同步并阻塞**的IO模型 每次客户端的请求过来 服务器都会开启一个线程去处理 

如果有请求没有进行读写操作    会造成不必要的线程的开销

![](E:\blog\netty\bio.jpg)

#### 1.2 什么是NIO

NIO是一种**同步非阻塞**的一种IO的模型  所有的客户端连接会被注册到一个多路复用器上 这个多路复用器会进行轮询操作 当多路复用器轮询到该客户端连接有相应的IO操作是  就会处理 

![](E:\blog\netty\nio.jpg)

  #### 1.3 理解什么是BIO的阻塞

BIO的阻塞会发生在两个地方

①  BIO的阻塞会发生在accept这里 如果没有连接 就会在这里阻塞

②  BIO的阻塞同时会发生在read这里  如果读取不到数据 同样也会发生阻塞

看具体的代码示例

```java
package com.zyk;


import java.io.IOException;
import java.io.InputStream;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class BIOServer {
    public static void main(String[] args) throws IOException {
        //准备一个线程池
        ExecutorService cachedThreadPool = Executors.newCachedThreadPool();
        //准备一个服务器端
        ServerSocket serverSocket=new ServerSocket(6666);
        //服务器循环的处理客户端的请求
        while (true){
            System.out.println("在accept这里  得不到连接"+Thread.currentThread().getName()+"会发生阻塞");
            final Socket socket = serverSocket.accept();
            System.out.println("建立了一个连接 阻塞结束 后面代码执行");
            cachedThreadPool.execute(new Runnable() {
                public void run() {
                    //执行响应的  服务器端的逻辑的方法
                    try {
                        handler(socket);
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
    }

    //执行响应的服务器端逻辑的方法
    public static void handler(Socket socket) throws IOException {
        InputStream is = socket.getInputStream();
        byte[] bytes=new byte[1024];
        int len=-1;
        //循环的读取客户端发来的数据
        while (true){
            System.out.println("当read不到数据  也会发生阻塞");
            while((len = is.read(bytes)) != -1){
                System.out.println(Thread.currentThread().getName()+"处理了"+new String(bytes,0,len));
                break;
            }
        }
    }
}

```

①accept阻塞

![](E:\blog\netty\accept阻塞.jpg)

② read阻塞

![](E:\blog\netty\read阻塞.jpg)

#### 1.4  BIO会出现的问题 

1）当并发较大时  会创建大量的线程

2）当read读取不到数据时  会一直阻塞  造成服务器资源的浪费（当前线程只能阻塞 不能处理其他事情）

### 2、再看NIO

#### 2.1 简介NIO

**NIO是一种同步非阻塞的I/O模型  它提供了Channel  Buffer  Selector 三种抽象**

**NIO是面向缓冲区 或者面向块编程的**

#### 2.2  NIO的三大核心组件之间的关系（Channel Buffer Selector的关系）

![](E:\blog\netty\channel buffer selector之间的关系.png)

1)   一个线程对应一个Selector  每一个Channel对应一个Buffer 即一个线程对应多个Channel。

2） Channel的切换是基于事件的   即Selector会根据事件进行通道Channel的切换。

3） 数据的读写是通过Buffer来实现的  不同于bio的方式  bio中的流是单向的 而nio中Buffer可读也可写，

​       数据的读写要用flip方法来切换。

4） Channel也是双向的 （可以同时进行读写 而传统的bio的流只能读或者写）。

#### 2.3  利用Channel将本地文件读取到控制台的小案例

```java
package com.zyk.nio;


import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

public class ChannelDemo {
    public static void main(String[] args) throws IOException {
        //读取本地文件到控制台打印

        //获取输入流
        File file=new File("e:\\aaa.txt");
        FileInputStream fileInputStream=new FileInputStream(file);
        //用管道包装输入流
        FileChannel channel = fileInputStream.getChannel();
        //建立一个缓冲区
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
        //将数据写入缓冲区(从管道往缓冲区读取数据)
        channel.read(byteBuffer);

        //从缓冲区把数据get出来  打印到控制台上面
        byte[] array = byteBuffer.array();
        String string=new String(array);
        System.out.println(string);
    }
}

```

#### 2.4 使用一个Buffer完成文件的拷贝

要求  将本地的aaa.txt文件拷贝到本地的bbb.txt当中  这个过程当中只使用一个Buffer

```java
package com.zyk.nio;

import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

public class BufferDemo02 {
    public static void main(String[] args) throws Exception {
        File file=new File("e:\\aaa.txt");
        FileInputStream fis=new FileInputStream(file);
        FileChannel channel = fis.getChannel();
        
        FileOutputStream fos=new FileOutputStream("e:\\ddd.txt");
        FileChannel channel1 = fos.getChannel();
        //创建Buffer
        ByteBuffer byteBuffer = ByteBuffer.allocate(512);
        //循环的拷贝数据
        while (true){
            //将position和limit重置
            byteBuffer.clear();
            int read = channel.read(byteBuffer);
            System.out.println(read);
            if(read==-1){
                break;
            }
            byteBuffer.flip();
            channel1.write(byteBuffer);
        }   
    }
}

```

#### 2.5 Selector选择器的基本介绍

- Selector可以检测多个注册的通道上是否有事件发生

- 当线程在某一个socket连接没有读写的请求时  该线程可以切换去做其他任务

  线程在非阻塞的I/O空闲时间  可以去处理其他通道的I/O请求  实现了单独的线程处理多个通道的请求

- 避免了频繁的I/O操作导致的线程挂起 减少了线程的上下文的切换

#### 2.6 NIO非阻塞网络编程的基本流程

①获取ServerSocketChannel   将serverSocketChannel注册到到selector选择器上

②选择器调用select()方法  开始选择有事件发生的通道 

③对应要与服务器端建立连接的客户端的请求 将这个socketChannel注册到选择器上 并且分配一个buffer

④对于发来数据的客户端的请求 通过selectionKey得到对应的Channel  然后通过管道和buffer进行数据的读取

#### 2.7 NIO 实现服务器和客户端之间的简单通信

NIO服务器端代码

```java
package com.zyk.nio;

import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.Iterator;
import java.util.Set;

public class NIOServer {
    public static void main(String[] args) throws  Exception {
        //获得ServerSocketChannel
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        //获得选择器
        Selector selector = Selector.open();

        //获得ServerSocketChannel绑定端口并且注册到选择器上
        serverSocketChannel.socket().bind(new InetSocketAddress(6666));
        serverSocketChannel.configureBlocking(false);
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        //服务器端进行循环的操作处理客户端的请求
       while (true){
           int select = selector.select(1000);
           if(select==0){
               //在这段时间之内没有获取到任何的事件请求
               System.out.println("这段时间内没有任何的网络通信产生");
               continue;
           }

           Set<SelectionKey> selectionKeys = selector.selectedKeys();
           Iterator<SelectionKey> iterator = selectionKeys.iterator();
           while (iterator.hasNext()){
               SelectionKey selectionKey = iterator.next();

               //该事件是需要建立连接
               if(selectionKey.isAcceptable()){
                   SocketChannel socketChannel = serverSocketChannel.accept();
                   socketChannel.configureBlocking(false);
                   //注册
                   socketChannel.register(selector,SelectionKey.OP_READ,ByteBuffer.allocate(1024));
                   System.out.println("有客户端连接注册到了选择器上面  "+socketChannel.hashCode());

               }

               //该事件是客户端发送来了数据 处于可读的状态
               if (selectionKey.isReadable()){
                   //得到一个SocketChannel
                   SocketChannel socketChannel =(SocketChannel) selectionKey.channel();

                   //把数据写入到buffer当中 应该读取通道
                   ByteBuffer buffer = (ByteBuffer)selectionKey.attachment();
                   socketChannel.read(buffer);

                   //控制台打印一下从客户端读取到的数据
                   byte[] array = buffer.array();
                   System.out.println("from客户端的数据  "+socketChannel.hashCode()+new String(array));

               }

               //Selector不会自己从已选择的键集中移除selectorKey
               //得用迭代器移除 下次有事件发生的时候再次被选入到了集合中
               iterator.remove();
           }

       }
    }
}

```

NIO 客户端代码

```java
package com.zyk.nio;

import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;
import java.util.Date;

public class NIOClient {
    public static void main(String[] args) throws Exception {
        SocketChannel socketChannel = SocketChannel.open();
        socketChannel.configureBlocking(false);
        InetSocketAddress inetSocketAddress = new InetSocketAddress("127.0.0.1", 6666);
        //连接服务器端
        if (!socketChannel.connect(inetSocketAddress)) {
            while (!socketChannel.finishConnect()){
                System.out.println("没有连接上服务器");
            }
        }

        //连接成功之后
        String str="hello"+new Date();
        //发送给服务器
        ByteBuffer buffer = ByteBuffer.wrap(str.getBytes());
        socketChannel.write(buffer);
        System.in.read();
    }
}
```

#### 2.8 ServerSocketChannel和socketChannel区别

①ServerSocketChannel用来监听客户端连接  每次有一个客户端建立连接就建立一个socketChannel

②socketChannel是具体的网络I/O的通道  负责具体的读写操作

### 3、NIO实现的一个群聊系统

NIO群聊系统的服务器端代码

```java
package com.zyk.chatroom;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.Iterator;
import java.util.Set;

public class NIOServer {
    public static void main(String[] args) throws Exception {
        //获得ServerSocketChannel
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        //获得选择器
        Selector selector = Selector.open();

        //获得ServerSocketChannel绑定端口并且注册到选择器上
        serverSocketChannel.socket().bind(new InetSocketAddress(6666));
        serverSocketChannel.configureBlocking(false);
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);


        //服务器端一直循环处理客户端请求的状态
        while (true){
            int select = selector.select(1000);
            if(select==0){
                System.out.println("此阶段没有任何客户端有事件发生");
                continue;
            }else {
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> iterator = selectionKeys.iterator();
                while (iterator.hasNext()){
                    //获得当前这个事件
                    SelectionKey selectionKey = iterator.next();

                    if (selectionKey.isAcceptable()){
                        SocketChannel socketChannel = serverSocketChannel.accept();
                        socketChannel.configureBlocking(false);
                        //完成这个socketChannel在多路复用器上的注册
                        socketChannel.register(selector,SelectionKey.OP_READ);
                        System.out.println(socketChannel.getRemoteAddress()+   "这个客户端用户上线了");

                    }else if (selectionKey.isReadable()){
                        SocketChannel socketChannel=null;

                        try {
                            socketChannel =(SocketChannel) selectionKey.channel();
                            ByteBuffer buffer = ByteBuffer.allocate(1024);
                            //把管道数据读取到buffer里 然后写到服务器
                            int read = socketChannel.read(buffer);
                            if (read>0){
                                byte[] array = buffer.array();
                                String msg=new String(array);
                                System.out.println("服务器收到了从 "+socketChannel.getRemoteAddress()+"发过来的内容 "
                                        +"具体是"+msg);



                                //服务器收到内容之后  把这个内容转发给其他所有客户端
                                //keys()方法返所有注册到了多路复用器上的键集
                                Set<SelectionKey> keys = selector.keys();
                                Iterator<SelectionKey> keyIterator = keys.iterator();
                                while (keyIterator.hasNext()){
                                    SelectionKey key = keyIterator.next();
                                    if (key.channel() instanceof SocketChannel&&key.channel()!=socketChannel){
                                        SocketChannel channel = (SocketChannel)key.channel();
                                        ByteBuffer byteBuffer = ByteBuffer.wrap(msg.getBytes());
                                        channel.write(byteBuffer);
                                    }
                                }
                            }
                        } catch (IOException e) {
                            System.out.println(socketChannel.getRemoteAddress()+"下线了");
                            //取消注册 关闭通道
                            selectionKey.cancel();
                            socketChannel.close();
                        }


                    }else if (selectionKey.isWritable()){
                        //服务器收到了某一个客户端发来的数据  目前这个selectionKey处于可写的状态

                    }

                    //遍历结束之后  在集合中删除这个键
                    iterator.remove();
                }
            }


        }
    }
}

```

NIO群聊系统的客户端代码

```java
package com.zyk.chatroom;



import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectableChannel;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Scanner;
import java.util.Set;

public class NIOClient {
    private final String HOST="127.0.0.1";
    private final int PORT=6666;
    private SocketChannel socketChannel;
    private Selector selector;

    public NIOClient() throws Exception {
        selector=Selector.open();
        socketChannel = SocketChannel.open();
        socketChannel.configureBlocking(false);
        InetSocketAddress inetSocketAddress = new InetSocketAddress(HOST,PORT);
        //连接服务器端
        socketChannel.register(selector, SelectionKey.OP_READ);
        socketChannel.connect(inetSocketAddress);
        if (socketChannel.finishConnect()){
            System.out.println(socketChannel.getLocalAddress()+"上线了");
        }



    }

    public  void sendMsg(String msg) throws IOException {
        ByteBuffer buffer = ByteBuffer.wrap(msg.getBytes());
        socketChannel.write(buffer);
    }

    public  void receiveMsg() throws IOException {
        int select = selector.select(500);
        if (select!=0){
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = selectionKeys.iterator();
            while (iterator.hasNext()){
                SelectionKey selectionKey = iterator.next();
                if (selectionKey.isReadable()){
                    ByteBuffer buffer = ByteBuffer.allocate(1024);
                    SocketChannel channel =(SocketChannel) selectionKey.channel();
                    channel.read(buffer);
                    String s = new String(buffer.array());
                    System.out.println("接收到的内容"+s);
                }
                iterator.remove();
            }
        }
    }

    public static void main(String[] args) throws Exception {
        NIOClient nioClient=new NIOClient();
        //启动另外一个线程一直从服务器端读取数据
        new Thread(()->{
            while (true){
                try {
                    nioClient.receiveMsg();
                    try {
                        Thread.currentThread().sleep(100);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }).start();

        Scanner scanner=new Scanner(System.in);
        while (scanner.hasNextLine()){
            String msg = scanner.nextLine();
            nioClient.sendMsg(msg);
        }



    }
}

```

### 4、NIO与零拷贝

#### 4.1 什么是零拷贝

1)传统的I/O 要经过4次拷贝，2次DMA拷贝，2次cpu拷贝 ，要经过3次操作系统用户态和内核态的转化

![](E:\blog\netty\传统io拷贝.jpg)

2）通过mmap的方式 用户态可以直接共享kernel 空间中的数据  kernel空间中的数据可以直接拷贝到了socket buffer当中 所以减少了一次cpu拷贝

![](E:\blog\netty\mmap拷贝.jpg)

3）Linux2.1中 ，增加了sendFile函数，数据不用经过用户空间 直接从内核空间拷贝到socket buffer当中，减少了一次操作系统用户态和内核态的切换。

![](E:\blog\netty\sendfile1.jpg)

4）Linux2.4中，彻底减少了从内核空间到socket buffer的拷贝，直接从内核缓冲区拷贝到了协议栈，整个过程只有2次DMA拷贝，完全没有cpu的拷贝的发生 又叫做零拷贝。

![](E:\blog\netty\sendfile2.jpg)









