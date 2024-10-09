## lecture-1: some key concepts in the network layer



### A day in in the life of a app

在[大课](https://www.youtube.com/watch?v=r2WZNaFyrbQ&list=PL6RdenZrxrw9inR-IJv-erlOKRHjymxMN)中用了三个具体的例子来说明：

WWW(http)

相互直接通过Internet进行通讯

![image-20240919155737055](https://cdn.jsdelivr.net/gh/NGSJCBF/img@main/img/202409191557195.png)

BitTorrent



![image-20240919155812760](https://cdn.jsdelivr.net/gh/NGSJCBF/img@main/img/202409191600552.png)

Skype：通过中转站的方式

![image-20240919155852371](https://cdn.jsdelivr.net/gh/NGSJCBF/img@main/img/202409191600406.png)

![image-20240919155924636](https://cdn.jsdelivr.net/gh/NGSJCBF/img@main/img/202409191600314.png)







### 读读源码

网络的三个关键概念：datagrams、encapsulation、multiplexing

- Datagrams 数据报
  - a short piece of data (aka a "packet") that includes:
    - a "destination" address with meaning in some context
    - other data(the "payload")
  - the service abstraction is:
    - the network will make "best efforts" to deliver the datagram to its ultimate destination (what's marked in the datagram)
    - **no matter where the datagram is currently!**
      - the destination address is sufficient to find the **ultimate** destination. It's not just the "next hop" -- it's the final destination of that datagram.
      - Any part of the network knows how to deal with it to get it closer to its final destination
      - 
- Encapsulation 封装
  - assigning an object to be the **payload of another object,** where the outer object is oblivious to the meaning or contents of its payload
  - **How Encapsulation Works(GPT tells)**:
    1. **Application Layer** (e.g., HTTP, FTP):
       - The user data, such as a file or web request, originates here. No encapsulation occurs at this point, but the data is prepared for transport.
    2. **Transport Layer** (e.g., TCP, UDP):
       - The application data is encapsulated into a **segment** (TCP) or **datagram** (UDP). The transport layer adds headers that contain information such as source and destination ports, sequencing, and error checking mechanisms (checksum).
    3. **Network Layer** (e.g., IP):
       - The transport segment is encapsulated into a **packet** or **datagram**. This layer adds an **IP header**, which includes the source and destination IP addresses, Time to Live (TTL), and other routing information.
    4. **Data Link Layer** (e.g., Ethernet, Wi-Fi):
       - The network packet is encapsulated into a **frame**. A data link layer header is added, which contains the **source and destination MAC addresses**, as well as other information for ensuring proper communication on the physical network. Sometimes a **trailer** is added for error detection (e.g., a frame check sequence).
    5. **Physical Layer** (e.g., Ethernet cable, fiber optics, radio waves):
       - The frame is broken down into electrical signals, light pulses, or radio waves, which are transmitted over the physical medium.
  - **Decapsulation(Reverse Process)**
- Multiplexing 多路复用
  - **sharing** a resource by dispatching according to a **runtime identifier**
  - Time Division Multiplexing (TDM), Frequency Division Multiplexing (FDM), Wavelength Division Multiplexing (WDM), Code Division Multiplexing (CDM). And there will be some problem in understanding the principle of [CDM][https://en.wikipedia.org/wiki/Code-division_multiple_access]. 

```c++
#include "socket.hh"
#include <cstdlib>
#include <iostream>

using namespace std;

// NOLINTBEGIN(*-casting)

class RawSocket : public DatagramSocket
{
public:
  RawSocket() : DatagramSocket( AF_INET, SOCK_RAW, IPPROTO_RAW ) {}
};

auto zeroes( auto n )
{
  return string( n, 0 );
}

//building an IPv4 datagram
void send_internet_datagram( const string& payload )
{
  RawSocket sock;

  string datagram;
  datagram += char( 0b0100'0101 ); // v4(IPV4), and headerlength 5 words
  datagram += zeroes( 7 ); //These could correspond to fields like the Type of Service, Total Length, and Identification.
  //Type of Service (1 byte).
  //Total Length (2 bytes) – likely zeroed for now, though this would normally hold the size of the datagram.
  //Identification (2 bytes) – also zeroed.
  //Flags and Fragment Offset (2 bytes) – also zeroed.
    
  datagram += char( 64 );  // TTL: Time to live; decremented by each router the packet passes through
  datagram += char( 1 );   // protocol: using ICMP (Internet Control Message Protocol).
  datagram += zeroes( 6 ); // checksum + src address 

  ////Destination IP Address
  datagram += char( 34 );  
  datagram += char( 93 );
  datagram += char( 94 );
  datagram += char( 131 );//34.93.94.131

  datagram += payload;

  sock.sendto( Address { "1" }, datagram );
}

void send_icmp_message( const string& payload )
{
  send_internet_datagram( "\x08" + payload );
}

void program_body()
{
  string payload;
  while ( cin.good() ) {
	getline( cin, payload );
	send_icmp_message( payload + "\n" );
  }
}

int main()
{
  try {
	program_body();
  } catch ( const exception& e ) {
	cerr << e.what() << "\n";
	return EXIT_FAILURE;
  }

  return EXIT_SUCCESS;
}

// NOLINTEND(*-casting)

```

**beautified version**

![Lightbox](https://cdn.jsdelivr.net/gh/NGSJCBF/img@main/img/202410091040067.jpeg)

**original version**

![image-20240910202349715](https://cdn.jsdelivr.net/gh/NGSJCBF/img@main/img/202409102023829.png)

Yeah! Very intuitive! 
