# Lab Checkpoint 0: networking warmup

我还在想怎么用一句话来表示我对无知的恐惧但又很想霸气地去面对它(就像古人那样子)哈哈哈哈哈哈哈😅

值得吐槽的一点是，我从以前开始看英文材料什么的，都是一堆密密麻麻的字，看的好难受

冲冲冲👊~

### 1 环境配置

环境配置一搜一大堆懒得写了，傻瓜式输入指令就好了。

Set up GNU/Linux on your computer：我在这次的学习中是直接选择使用cs144官方给的虚拟机（实在不想在自己那个乱七八糟的wsl上再配置什么东西了QWQ），在vmware或者virtualbox上直接导入就好（可能需要注意一下最好给到15G的磁盘空间，别问我为什么要提醒这个😭）



### 2 Networking by hand

这一部分让我们体验一下~~黑窗下~~如何去使用network: retrieving a Web page (just like a Web browser) and sending an email message (like an email client).

这两个任务依赖于一个叫做**可靠的双向字节流**的网络抽象：你会在终端输入一串字节，这串字节最终会以相同的顺序传送到另一台计算机上运行的程序（服务器）。服务器会以自己的字节序列进行响应，并将它传回到你的终端。

#### 2-1 Fetch a Web page

```shell
telnet cs144.keithw.org http
#然后键入下面的东西
GET /hello HTTP/1.1 #  tells the server the path part of the URL.
Host: cs144.keithw.org # tells the server the host part of the URL. 
Connection: close # tells the server that you are finished making requests, and it should close the connection as soon as it 	                     finishes replying.
# 再次键入'\n'
```

- response是这样的

```shell
HTTP/1.1 200 OK
Date: Thu, 19 Sep 2024 07:05:34 GMT
Server: Apache
Last-Modified: Thu, 13 Dec 2018 15:45:29 GMT
Accept-Ranges: bytes
Content-Length: 14
Connection: close
Content-Type: text/plain

Hello, CS144!
Connection closed by foreign host.
```

- nc（netcat）在本地的 2333 端口上等待来自其他主机的连接，类似一个简单的服务器。当其他主机通过相应的 IP 地址和端口号连接到这个端口时，数据可以在两者之间传输。
  - `-l`：表示 Netcat 进入监听模式，会等待一个连接请求，而不是主动去连接。
  - `-v`：启用详细输出模式，显示更多调试信息（比如连接成功或失败的消息）。
  - `-p 2333`：指定要监听的端口号 2333。

```shell
nc -lvp 2333
```

- 这个命令会让 Telnet 尝试连接到本地的 2333 端口。如果这个端口上有服务在运行，比如前面用 Netcat 监听的服务，Telnet 可以连接并进行数据交互。Telnet 是一个传统的网络协议工具，常用于测试和简单的网络调试。

```shell
telnet localhost 2333
```

那个写email的小实验可以自己试试看，我懒得写了QWQ



### 3 Writing a network program using an OS stream socket

#### 3-0 一些前置

1. **Stream Socket**

在实验文档中提到的 "stream socket"是 Linux 内核和大多数操作系统提供的一个特性，它允许在两个程序之间创建一个可靠的双向字节流。这个字节流连接可以在同一台计算机上运行的程序之间，也可以在两台通过网络连接的计算机之间（比如客户端和一个Web服务器，或者Netcat程序）。
对程序和Web服务器来说，流套接字看起来就像一个普通的**文件描述符**，比如像文件、标准输入（stdin）或标准输出（stdout）那样，通过读写操作传输数据。

![image-20240919162424301](https://cdn.jsdelivr.net/gh/NGSJCBF/img@main/img/202409191624332.png)

2. **互联网的基础服务：最佳努力交付**

虽然操作系统通过 TCP 提供了可靠的字节流，但实际上互联网本身并不保证提供这样的服务。互联网只是尽力将数据包（称为 "互联网数据报" 或 "IP数据报"）传递到目的地。每个数据报包含一些元数据，比如源地址、目标地址等信息，以及最多1500字节的数据。

3. **互联网的不可靠性**

在传输过程中，互联网数据报可能会出现以下问题：

- **(1) 丢失**：数据报可能在网络中消失。
- **(2) 顺序错乱**：数据可能以错误的顺序到达。
- **(3) 数据篡改**：数据内容可能在传输途中被修改。
- **(4) 重复发送**：数据报可能被重复传输。

因此，互联网提供的传输服务本质上是不可靠的。

4. **操作系统的作用：将不可靠的服务变为可靠的字节流**

为了实现可靠的字节流传输，两个计算机的操作系统需要合作。它们需要确保每个字节都按照正确的顺序被传递到对方的流套接字中。它们还需要通过某种方式相互告知自己可以接收的数据量，以防止一方发送的数据超过另一方的承受能力。

5. **TCP 协议**

这些可靠性功能是通过一种被称为 **传输控制协议（TCP）** 的协议来实现的。TCP 协议在1981年被制定，它规定了一种标准化的方式，使得操作系统能够将互联网的 "最佳努力" 数据报转化为可靠的字节流。通过 TCP，计算机可以确保数据可靠地传输，数据顺序不乱，不丢失，并且不会超出接收方的接收能力。



#### **3-1** Let’s get started—fetching and building the starter code

搭建环境，看文档就好😁



#### 3-2 Modern C++: mostly safe but still fast and low-level

一些该注意的点（纯搬运，如有侵权，我将删除QWQ，我这个至少格式比那个pdf好看了吧🤭）

> • Use the language documentation at https://en.cppreference.com as a resource. (We’d recommend you avoid cplusplus.com which is more likely to be out-of-date.) 
>
> • Never use malloc() or free(). 
>
> • Never use new or delete. 
>
> • Essentially never use raw pointers (*), and use “smart” pointers (unique ptr or shared ptr) only when necessary. (You will not need to use these in CS144.) 
>
> • Avoid templates, threads, locks, and virtual functions. (You will not need to use these in CS144.) • Avoid C-style strings (char *str) or string functions (strlen(), strcpy()). These are pretty error-prone. Use a std::string instead. 
>
> • Never use C-style casts (e.g., (FILE *)x). Use a C++ static cast if you have to (you generally will not need this in CS144). • Prefer passing function arguments by const reference (e.g.: const Address & address). 
>
> • Make every variable const unless it needs to be mutated. 
>
> • Make every method const unless it needs to mutate the object. 
>
> • Avoid global variables, and give every variable the smallest scope possible. 
>
> • Before handing in an assignment, run cmake --build build --target tidy for suggestions on how to improve the code related to C++ programming practices, and cmake --build build --target format to format the code consistently.

- a Socket is a type of FileDescriptor, and a TCPSocket is a type of Socket. 【感谢huangrt01师傅提供了这个[网站](https://cs144.github.io/doc/lab0/)，让我茅塞顿开】



#### 3-3 Reading the Minnow support code

待会再读

>  (Please note that a Socket is a type of FileDescriptor, and a TCPSocket is a type of Socket.)



#### 3-4 webget()

Hints

> • Please note that in HTTP, each line must be ended with “\r\n” (it’s not sufficient to use just “\n” or endl). 
>
> • Don’t forget to include the “Connection: close” line in your client’s request. This tells the server that it shouldn’t wait around for your client to send any more requests after this one. Instead, the server will send one reply and then will immediately end its outgoing bytestream (the one from the server’s socket to your socket). You’ll discover that your incoming byte stream has ended because your socket will reach “EOF” (end of file) when you have read the entire byte stream coming from the server. That’s how your client will know that the server has finished its reply. 
>
> • Make sure to read and print all the output from the server until the socket reaches “EOF” (end of file)—a single call to read is not enough. 
>
> • We expect you’ll need to write about ten lines of code.

10行代码，整出个小浏览器就好，相当于熟悉一下socket里面有啥

```c++
void get_URL(const string &host, const string &path) {
    TCPSocket socket{};
    socket.connect(Address(host,"http"));
    string input("GET "+path+" HTTP/1.1\r\nHost: "+host+"\r\n\r\n");
    socket.write(input);
    socket.shutdown(SHUT_WR);
    while(!socket.eof()){
        string data;
        socket.read(data);
        cout<<data;  }
    sock.close();
}
```



### 4 An in-memory reliable byte stream

这里我们需要实现在内存中的一个字节流，字节从input中写入，从output中被读出（管道），遇到EOF（0）的时候就可以停下来了。

![image-20240919193529434](https://cdn.jsdelivr.net/gh/NGSJCBF/img@main/img/202409191935520.png)



![](https://cdn.jsdelivr.net/gh/NGSJCBF/img@main/img/202409191935520.png)

byte_stream.hh

```c++
#pragma once

#include <cstdint>
#include <string>
#include <string_view>

class Reader;
class Writer;

class ByteStream
{
public:
  explicit ByteStream( uint64_t capacity );

  // Helper functions (provided) to access the ByteStream's Reader and Writer interfaces
  Reader& reader();
  const Reader& reader() const;
  Writer& writer();
  const Writer& writer() const;

  void set_error() { error_ = true; };       // Signal that the stream suffered an error.
  bool has_error() const { return error_; }; // Has the stream had an error?

protected:
  // Please add any additional state to the ByteStream here, and not to the Writer and Reader interfaces.
  uint64_t capacity_;
  bool error_ {};
  bool is_closed_ {};
   
  std::string buffer_ {};
  std::deque<char>buffer_temp_{};
  uint64_t bytes_pushed_ {};
  uint64_t bytes_popped_ {};
    
};

class Writer : public ByteStream
{
public:
  void push( std::string data ); // Push data to stream, but only as much as available capacity allows.
  void close();                  // Signal that the stream has reached its ending. Nothing more will be written.

  bool is_closed() const;              // Has the stream been closed?
  uint64_t available_capacity() const; // How many bytes can be pushed to the stream right now?
  uint64_t bytes_pushed() const;       // Total number of bytes cumulatively pushed to the stream
};

class Reader : public ByteStream
{
public:
  std::string_view peek() const; // Peek at the next bytes in the buffer
  void pop( uint64_t len );      // Remove `len` bytes from the buffer

  bool is_finished() const;        // Is the stream finished (closed and fully popped)?
  uint64_t bytes_buffered() const; // Number of bytes currently buffered (pushed and not popped)
  uint64_t bytes_popped() const;   // Total number of bytes cumulatively popped from stream
};

/*
 * read: A (provided) helper function thats peeks and pops up to `len` bytes
 * from a ByteStream Reader into a string;
 */
void read( Reader& reader, uint64_t len, std::string& out );
```

byte_stream_helper.cc

```c++
#include "byte_stream.hh"

using namespace std;

ByteStream::ByteStream( uint64_t capacity ) : capacity_( capacity ) { 
  buffer_.reserve( capacity );
}

bool Writer::is_closed() const
{
  return this->is_closed_;
}

void Writer::push( string data )
{
  size_t available_capacity = this->available_capacity();
  size_t bytes_to_copy = min(data.size(), available_capacity);

  buffer_ += data.substr(0, bytes_to_copy);
  bytes_pushed_ += bytes_to_copy;
    
  //下面是deque版本的代码内容
  //size_t available_capacity=this->available_capacity();
  //size_t bytes_to_copy = min(data.size(),available_capacity);
  //for(char c:data.substr(0,bytes_to_copy)){
  //	bytes_temp_.push_back(c);
  //}
  bytes_pushed+= bytes_to_copy; 
}

void Writer::close()
{
  is_closed_ = true;
}

uint64_t Writer::available_capacity() const
{
  return capacity_ - buffer_.size();
  //deque版本
  //return capacity_ - deque.size();
}

uint64_t Writer::bytes_pushed() const
{
  return bytes_pushed_;

}

bool Reader::is_finished() const
{
  return ((is_closed_ == true) && (this->bytes_buffered() == 0));
  
}

uint64_t Reader::bytes_popped() const
{
  return bytes_popped_;
}

string_view Reader::peek() const
{
  string_view sv( buffer_.data() , min(static_cast<uint64_t>(1000),this->bytes_buffered()));
  return sv;
  //deque版本
  //string_view sv(buffer_temp_,buffer_temp_.size());
    return sv;
}

void Reader::pop( uint64_t len )
{
  buffer_.erase( buffer_.begin() , buffer_.begin() + len );
  bytes_popped_ += len;
  
  if(len>bytes_temp_.size())len=bytes_temp_.size();
  //deque版本
  //for(int i=0;i<len;i++){
  //    buffer_temp_.pop_front();
  //}
  bytes_popped_+=len;
}

uint64_t Reader::bytes_buffered() const
{
  return buffer_.size();
  //deque版本
  return buffer_temp_.size();
}
```

在后面浏览了其他人对这部分的做法后，发现自己的编程能力异常地差嘞(deque版本还没改好🤣)，好好加油/(ㄒoㄒ)/~~。



### Referencing

[这个师傅的学习笔记做得很详细，我一些编程上不懂的就会看看他的做法，Don't be so serious~~~](https://tzr.icu/)    ~~臭大三的编程能力还这么哭死了😭（指我）~~

还有一个就是GPT啦嘻嘻嘻

![image-20240927174321200](https://cdn.jsdelivr.net/gh/NGSJCBF/img@main/img/202409271743160.png)
