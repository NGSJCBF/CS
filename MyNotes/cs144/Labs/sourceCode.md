source code这里感觉直接看他给的源码网站就好了，这里只做点标记

### writing a network program using an OS stream socket

注意点（c文件直接看gpt的描述吧，一点一点翻译好了QWQ）

在socket.hh头文件中，主要声明了几种socket

`Socket` Class (Base Class for Network Sockets)

**Key Methods and Members:**

- **`get_address()`**: Retrieves either the local or peer address of a connected socket. It accepts a function that interacts with low-level system calls like `getsockname` and `getpeername`.
- **`bind()`**: Binds the socket to a local address (useful for listening or accepting connections).
- **`connect()`**: Connects the socket to a peer address.
- **`shutdown()`**: Shuts down the socket in one or more directions (read/write).
- **`local_address()`**: Returns the local address of the socket using `getsockname`.
- **`peer_address()`**: Returns the peer address using `getpeername`.
- **`set_reuseaddr()`**: Sets the `SO_REUSEADDR` option on the socket, allowing the local address to be reused quickly.
- **`throw_if_error()`**: Checks for socket errors and throws an exception if any are detected.

 **`DatagramSocket` Class**

- Inherits from `Socket` and is used for sockets that communicate using datagrams (like UDP).
- **`recv()`**: Receives a datagram, storing the sender's address and the payload.
- **`sendto()`**: Sends a datagram to a specific destination address.
- **`send()`**: Sends a datagram to the connected address (after a call to `connect()`).
- `UDPSocket` Class
- Inherits from `DatagramSocket` and provides a wrapper around UDP sockets (`SOCK_DGRAM`).
- **`UDPSocket()`**: Default constructor for an unbound, unconnected UDP socket.

 **`TCPSocket` Class**

- Inherits from `Socket` and provides a wrapper around TCP sockets (`SOCK_STREAM`).
- **`listen()`**: Marks the socket as a listener for incoming connections.
- **`accept()`**: Accepts an incoming connection, returning a new `TCPSocket` representing the connection.

**`PacketSocket` Class**

- Inherits from `DatagramSocket` and is designed for raw packet sockets (used for lower-level network protocols).
- **`set_promiscuous()`**: Likely sets the socket to promiscuous mode, allowing it to capture all network traffic on the interface.

`LocalStreamSocket` and `LocalDatagramSocket` Classes

- These are used for Unix domain sockets, which allow communication between processes on the same machine.
- **`LocalStreamSocket`**: Wrapper around Unix-domain stream sockets (`SOCK_STREAM`).
- **`LocalDatagramSocket`**: Wrapper around Unix-domain datagram sockets (`SOCK_DGRAM`).





在filedescription.hh中，

**`FDWrapper` Class**

- **`fd_`**: Stores the file descriptor number (an integer returned by system calls like `open()` or `socket()`).
- **`eof_`**: A flag that indicates whether the end of the file (EOF) has been reached.
- **`closed_`**: A flag indicating whether the file descriptor has been closed.
- **`non_blocking_`**: A flag indicating whether the file descriptor is in non-blocking mode.
- **`read_count_` & `write_count_`**: Track how many times the descriptor has been read from or written to.

keymethod

- **Constructor (`FDWrapper(int fd)`)**: Initializes the wrapper with a file descriptor.
- **Destructor (`~FDWrapper()`)**: Closes the file descriptor when the `FDWrapper` object is destroyed.
- **`close()`**: Closes the file descriptor explicitly.
- **`CheckSystemCall`**: A template function that checks the return value of a system call and ensures it completed successfully, or handles errors appropriately. `s_attempt` is a string that describes the operation being attempted, and `return_value` is the result of the system call.

`**FileDescriptor` Class**

- **`internal_fd_`**: A `std::shared_ptr<FDWrapper>`, which allows for shared ownership of the `FDWrapper` object. When the last `FileDescriptor` goes out of scope, the file descriptor is automatically closed.

- **`FileDescriptor(int fd)`**: Initializes the file descriptor with a given file descriptor number.
- **`FileDescriptor(std::shared_ptr<FDWrapper>)`**: Private constructor used internally to duplicate the `FileDescriptor` (i.e., increase the reference count).

**Key Methods:**

- **`read(std::string& buffer)`**: Reads data from the file descriptor into the provided string buffer.
- **`read(std::vector<std::string>& buffers)`**: Reads data into multiple buffers (useful for more complex I/O operations).
- **`write(std::string_view buffer)`**: Attempts to write the contents of a string buffer to the file descriptor and returns the number of bytes written.
- **`write(const std::vector<std::string_view>& buffers)`**: Writes multiple buffers, useful for sending multiple chunks of data.
- **`close()`**: Closes the underlying file descriptor by calling `close()` on `FDWrapper`.
- **`duplicate()`**: Explicitly duplicates the `FileDescriptor`, increasing the reference count and sharing the file descriptor.
- **`set_blocking(bool blocking)`**: Sets the file descriptor to either blocking or non-blocking mode.

- **`size()`**: Returns the size of the file associated with the file descriptor

**Utility Methods:**

- **`fd_num()`**: Returns the actual file descriptor number.
- **`eof()`**: Returns whether the file descriptor is at the end of the file.
- **`closed()`**: Returns whether the file descriptor has been closed.
- **`read_count()`**: Returns how many times the file descriptor has been read from.
- **`write_count()`**: Returns how many times the file descriptor has been written to.

**Reference Counting and Lifetime Management:**

- The file descriptor is managed using a `std::shared_ptr`, meaning multiple `FileDescriptor` objects can refer to the same file descriptor, and the resource is released when the last reference is destroyed.
- **Move semantics**: The `FileDescriptor` can be moved but cannot be copied (copy constructors and copy assignment operators are deleted).

**Protected Members:**

- **`kReadBufferSize`**: A constant representing the size of the buffer used for reading data (set to 16 KB).

**Template Methods:**

- **`CheckSystemCall()`**: A utility method to check the result of a system call, similar to the one in `FDWrapper`.

![img](https://cdn.jsdelivr.net/gh/NGSJCBF/img@main/img/202409191518269.png)

*~~偷图~~*

> FileDescriptor: 在 Unix 内核的系统中，程序每打开或者创建一个文件，内核就会向进程返回一个文件描述符（通常为正整数）用来索引、指向这个文件。FileDescriptor 则封装了文件的读写等操作。
>
> Socket: 我粗浅的认为套接字就是两个程序通信的管道，一个程序写，另一个程序就可读。当然，两个程序可以在本地（LOCAL），也可以在不同的地方（即通过网络通信）。而 Socket 则封装了若干函数，这个地方要注意一下 `shutdown` 函数：
>
> 1. `shutdown(SHUT_RD)` 关闭当前 Socket 的读入，但是还是可以向 `connect` 的另一端写入数据。
> 2. `shutdown(SHUT_WR)` 关闭当前 Socket 的写入，但是还是可以读入 `connect` 的另一端写入的数据。
> 3. `shutdown(SHUT_RDWR)` 关闭读写。
>
> TCPSocket: 两个程序建立的通信的链接的协议为 TCP 协议，即 `Socket(AF_INET, SOCK_STREAM)`，`AF_INET` 表示网络连接（IP 协议），`SOCK_STREAM` 表示稳定的数据传输。（SOCK_STREAM provides sequenced, reliable, two-way, connection-based byte streams. An out-of-band data transmission mechanism may be supported）



```c++
void get_URL(const string &host,const string &path){
	TCPSocket sock;
    sock.connect(Address(host,"http"));
    sock.write("GET "+path+" HTTP/1.1\r\nHOST: "+host+"\r\n\r\n\r\n");
    sock.shutdown(SHUT_WR);
    while(!socket.eof())
        std::cout<<sock.read();
    sock.close();
    return;
}
```

利用TCPSocket与目标host建立连接并发送一个请求头即可。

1、\r\n：\r 将光标回到当前行的行首，之后的输出会把之前的输出覆盖

2、不要忘了添加“Connection: close”: 告诉服务器连接已经噶了【~~还是看文档好了~~】

Instead, the server will send one reply and then will immediately end its outgoing bytestream (the one from the server’s socket to your socket). You’ll discover that your incoming byte stream has ended because your socket will reach “EOF” (end of file) when you have read the entire byte stream coming from the server. That’s how your client will know that the server has finished its reply.

3、a single call to read is not enough.

4, we except you'll need to write about ten lines of code.

### An in-memony reliable byte stream

![image-20240912201527550](https://cdn.jsdelivr.net/gh/NGSJCBF/img@main/img/202409122015601.png)



**src/byte stream.hh src/byte stream.cc files**





conclusion: take more and more small step. 