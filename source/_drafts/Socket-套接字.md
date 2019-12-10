---
title: Socket 套接字
tags:
---


基于 TCP 和 UDP 协议的 Socket 编程。


Socket 编程是基于端到端的通信，意识不到中间经历了多少局域网、路由，因而能够设置的参数只能是端到端协议之上的网络层和传输层。


## 基于 TCP 协议的 Socket 程序函数调用过程


TCP 的服务端首先监听一个端口，一般先调用 bind 函数，给 Socket 对象赋值 IP 地址和端口号(网络包来时，内核需要通过 TCP 头中端口号，找到应用程序)。需要IP地址的原因，在一台服务器上可能有多个网卡，同时也会多个 IP，那么如果指定 IP 地址，那么就只会发给这个网卡数据包。


```
ServerSocket serverSocket = new ServerSocket(port);
clientSocket = serverSocket.accept();
out = new PrintWriter(clientSocket.getOutputStream(), true);
in = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
String greeting = in.readLine();
```

接下来，服务端调用 listener 函数进行监听，服务器就可以发起连接了。

在内核中，为 Socket 维护两个队列，一个是已经建立连接的队列，这时候三次连接已完成；另一个为还没有完全建立连接的队列，三次握手的过程还没有完成。



客户端可以通过 connect 函数发起连接，在参数中指定 IP 地址和端口号，然后发起三次握手，内核会给客户端临时分配一个端口，一旦握手成功，服务端的 accept 就会返回 **另外一个** Socket。

服务端监听的 Socket 和真正用来数据传输的 Socket 为两个不同的 Socket，一个叫做 监听 Socket，一个叫做已连接 Socket。


服务端和客户端建立连接后，双方可以使用 read 和 write 函数进行读写数据，就像往一个文件流中写数据一样。

基于 TCP 协议的 Socket 的函数调用过程：
![](https://static001.geekbang.org/resource/image/77/92/77d5eeb659d5347874bda5e8f711f692.jpg)


一个基于 TCP 协议的 Socket 程序：

服务端：

```
// 1.创建一个ServerSocket对象
        ServerSocket serverSocket = new ServerSocket(8888);
        // 2.调用accept()方法接受客户端请求
        Socket socket = serverSocket.accept();
        System.out.println(socket.getInetAddress().getHostAddress() + "连接成功");
        // 3.获取socket对象的输入输出流
        BufferedReader br = new BufferedReader(new InputStreamReader(
                socket.getInputStream()));
        PrintWriter pw = new PrintWriter(socket.getOutputStream(), true);
        String line = null;
        // 读取客户端传过来的数据
        while ((line = br.readLine()) != null) {
            if (line.equals("over")) {
                break;
            }
            System.out.println(line);
            pw.println(line.toUpperCase());
        }
        pw.close();
        br.close();
        socket.close();
        System.out.println(socket.getInetAddress().getHostAddress() + "断开连接");
```


**客户端**：


```
Socket socket = new Socket("127.0.0.1", 8888);
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));

        PrintWriter pw = new PrintWriter(socket.getOutputStream(), true);
        BufferedReader reader = new BufferedReader(new InputStreamReader(
                socket.getInputStream()));
        while (true) {
            String line = br.readLine();// 获取键盘所输入的字符串
            pw.println(line);
            if (line.equals("over")) {
                break;
            }
            System.out.println(reader.readLine());// 获取服务端传过来的大写字符串
        }
        reader.close();
        br.close();
        pw.close();
        socket.close();
```


## 基于 UDP 协议的 Socket 程序函数调用过程


UDP 是无连接的，所以不需要进行三次握手，这就以意味着不需要调用 listener 和 connect 函数，但是 UDP 依然是需要 IP 和 端口号，所以需要调用 bind 函数。

UDP 是无连接的，所以不需要为每对连接建立一组 Soket，而只需要有一个 Socket，就能够和多个客户端进行通信。因为为无连接的，所以每次通信的时候都需要调用 sendTo 和 recvFrom 函数，并且都可以传入 IP 地址和端口号。


![](https://static001.geekbang.org/resource/image/77/ef/778687d1a02ffc0c24078c33be2ac1ef.jpg)



一个基于 UDP 协议的 Socket 程序：

发送端：

```
DatagramSocket socket = new DatagramSocket();
        String str = "i love you";
        // 把数据进行封装到数据报包中
        DatagramPacket packet = new DatagramPacket(str.getBytes(),
                str.length(), InetAddress.getByName("localhost"), 6666);
        socket.send(packet);// 发送

        byte[] buff = new byte[100];
        DatagramPacket packet2 = new DatagramPacket(buff, 100);
        socket.receive(packet2);
        System.out.println(new String(buff, 0, packet2.getLength()));
        socket.close();
```

接收端：
```
DatagramSocket socket = new DatagramSocket();
        String str = "i love you";
        // 把数据进行封装到数据报包中
        DatagramPacket packet = new DatagramPacket(str.getBytes(),
                str.length(), InetAddress.getByName("localhost"), 6666);
        socket.send(packet);// 发送

        byte[] buff = new byte[100];
        DatagramPacket packet2 = new DatagramPacket(buff, 100);
        socket.receive(packet2);
        System.out.println(new String(buff, 0, packet2.getLength()));
        socket.close();
```
##


Socket 为 TCP/UDP 协议的上层使用。


---

[socket编程入门：1天玩转socket通信技术（非常详细）
](http://c.biancheng.net/socket/)

[网络编程——基于TCP协议的Socket编程，基于UDP协议的Socket编程](https://www.cnblogs.com/wzy330782/p/5479833.html)