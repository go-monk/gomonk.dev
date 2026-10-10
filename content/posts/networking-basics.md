+++
date = '2026-10-10T18:10:00+02:00'
draft = true
group = 'net' # no code here, only protocol diagrams
title = 'Networking basics'
+++

This is my cheat sheet on how data gets from one program to another over a network: the layering models, IP, TCP and UDP. It is mostly diagrams and lists, the things I keep looking up, such as what is in a TCP header or which steps a router takes with a packet. I wrote it while reading Wendell Odom's CCNA book.

## Networking models

A networking model splits the job of communicating into layers with standard interfaces between them. The two models you meet are OSI (Open Systems Interconnection), which has seven layers, and TCP/IP, which has four or five. Layering has these benefits:

* higher layers are shielded from the complexity of the lower layers
* an application "thinks" it's communicating directly with another application through the standard sockets API
* one vendor writes the software implementing higher layers (e.g. a web browser), another vendor writes the software implementing lower layers (e.g. the TCP/IP stack built into the operating system)

OSI and TCP/IP models:

```
       OSI              TCP/IP             TCP/IP             Sample
                        original           updated            protocol
  +------------+     +------------+     +------------+     +----------------+ The "language" that
7 |Application |     |            |     |            |     |Telnet HTTP     | apps and servers use
  |------------+     |            |     |            |     |FTP SMTP        | to communicate.
6 |Presentation|     |Application | 5-7 |Application |     |POP3 VoIP       |
  |------------+     |            |     |            |     |SNMP DNS Halo 3 |
5 |Session     |     |            |     |            |     |arp traceroute  |
  |------------+- - -+------------+- - -+------------+- - -+----------------+ Data transmission
4 |Transport   |     |Transport   |  4  |Transport   |     |TCP UDP         | characteristics.
  |------------+- - -+------------+- - -+------------+- - -+----------------+ How to move packets
3 |Network     |     |Internetwork|  3  |Internetwork|     |IP ICMP         | from src to dst.
  |------------+- - -+------------+- - -+------------+- - -+----------------+
2 |Data link   |     |Network     |  2  |Data link   |     |Eth HDLC PPP ARP|
  |------------+     |            |- - -|------------|- - -|----------------| How to send raw data
1 |Physical    |     |access      |  1  |Physical    |     |Eth RJ-45 V.35  | across a physical medium.
  +------------+     +------------+     +------------+     +----------------+
```

To remember the OSI layers from the bottom up: Please Do Not Throw Sausage Pizza Away. Layers are often called by number, so L2 is the data link layer and L3 the network layer.

Each layer provides a service to the layer above it. For example, TCP provides error recovery to HTTP:

![A web server sends three TCP segments, the second is lost, and the browser asks the server to send segment 2 next](/networking-basics/tcp_error_recovery.png)

Each layer also wraps the data from the layer above in its own header (and sometimes a trailer). This is called encapsulation. Here Ethernet carries an IP packet to the router:

![A host encapsulates an IP packet between an Ethernet header and trailer, transmits it, and the router receives and decapsulates it](/networking-basics/network_access_layer.png)

The wrapped data has a different name at each layer: a TCP segment, an IP packet, a data-link frame. The generic name is PDU (protocol data unit), so a packet is an L3 PDU.

![A segment is a TCP header plus data, a packet is an IP header plus data, a frame is a link header plus data plus a link trailer](/networking-basics/data_perspectives.png)

## Binary math

Binary to hexadecimal conversion works four bits (a nibble) at a time:

```
                 01101110   Binary (representation of) byte

                  |    |
                  v    v

  Higher order  0110  1110  Lower order
  nibble                    nibble
                  |    |
                  v    v

                  6    14   Decimal

                  |    |
                  v    v

                  6    E    Hexadecimal byte
```

## IPv4 addressing and routing

IP is the only widely used L3 protocol. The standard 20-byte IPv4 header:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  |   DS Field    |        Packet Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Identification        |Flags|      Fragment Offset    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Time to Live |    Protocol   |         Header Checksum       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source Address                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination Address                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

* IHL -- IP header length (including optional fields)
* DS Field -- differentiated services field, used for QoS (quality of service)
* Packet Length -- the entire packet length, including data
* Identification -- used by the IP fragmentation process; all fragments of a packet have the same ID
* Flags -- used by the IP fragmentation process (3 bits)
* Time to Live (TTL) -- each router decrements it and drops the packet at zero, which prevents routing loops
* Protocol -- the contents of the data portion of the packet (e.g. 6 means a TCP header is the first thing in the data field)

The rule of IP addressing: addresses in the same network must not be separated by a router.

What a router does with each frame it receives:

1. use the frame's checksum (FCS, frame check sequence) to check for errors; if an error occurred, discard the frame
2. discard the data-link header and trailer, leaving the IP packet
3. use the destination IP address to look up the outgoing interface in the routing table
4. encapsulate the IP packet inside a data-link header and trailer appropriate for the outgoing interface and forward the frame

### L3 utilities

DNS (Domain Name System) translates names to IP addresses:

1. What is foo's IP address?
2. Foo's IP is 10.1.1.2

ARP (Address Resolution Protocol) translates IP addresses to MAC addresses, the hardware addresses of network cards:

1. Hey everybody, if you are 10.1.1.2 tell me your MAC address!
2. I'm 10.1.1.2 and my MAC is 0200.2222.2222

[DHCP](https://github.com/jreisinger/blog/blob/master/posts/dhcp.md) (Dynamic Host Configuration Protocol) gives a host its IP address and other network settings:

1. Client -- DHCP discover message (broadcast to the local network)
2. Server -- DHCP offer message (broadcast to the local network)
3. Client -- DHCP request message (to server)
4. Server -- DHCP acknowledgement (to client)

## Transmission Control Protocol (TCP)

TCP is a connection-oriented protocol: it requires an exchange of messages (or a preestablished correlation) before data transfer.

Features:

* multiplexing using ports
* error recovery
* flow control using windowing
* connection establishment and termination
* ordered data transfer and segmentation

TCP header format:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ -----
|          Source Port          |       Destination Port        |    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+    |
|                        Sequence Number                        |    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+    |
|                    Acknowledgment Number                      |    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+  Required
|  Data |           |U|A|P|R|S|F|                               |    |
| Offset| Reserved  |R|C|S|S|Y|I|            Window             |    |
|       |           |G|K|H|T|N|N|                               |    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+    |
|           Checksum            |         Urgent Pointer        |    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ -----
|                    Options                    |    Padding    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             data                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The Window field holds the maximum number of unacknowledged bytes the sender may have outstanding. It starts small, then grows until errors occur (a dynamic or sliding window).

Multiplexing relies on sockets. A socket is three things:

* an IP address (e.g. 10.1.1.2)
* a transport protocol (e.g. TCP)
* a port number (e.g. 80)

TCP connection establishment (the three-way handshake):

```
 +-------+             SEQ=200                 +------+
 |Web    |     SYN, DPORT=80, SPORT=1027       |Web   |
 |browser| ----------------------------------> |server|
 +-------+                                     +------+
                    SEQ=1450, ACK=201
              SYN, ACK, DPORT=1027, SPORT=80
           <----------------------------------

                  SEQ=201, ACK=1451
               ACK, DPORT=80, SPORT=1027
           ---------------------------------->
```

The maximum transmission unit (MTU) is the size of the largest L3 packet that fits inside a frame's data field. It's 1500 bytes for many L2 protocols, including Ethernet.

* because the IP and TCP headers are 20 bytes each, TCP typically segments large data into 1460-byte chunks
* a TCP segment (L4 PDU) = TCP header + data field

## User Datagram Protocol (UDP)

UDP is a connectionless protocol: it does not require an exchange of messages (or a preestablished correlation) before data transfer.

UDP adds just two features to IP:

* port numbers for multiplexing
* a data checksum for error detection

Applications using UDP either tolerate data loss or have their own mechanism for recovering lost data:

* VoIP (voice over IP) -- recovery wouldn't help anyway, it would be too late
* DNS -- retries if the resolution fails
* NFS (Network File System) -- recovery is done by application layer code

UDP header format:

```
 0      7 8     15 16    23 24    31
+--------+--------+--------+--------+
|     Source      |   Destination   |
|      Port       |      Port       |
+--------+--------+--------+--------+
|                 |                 |
|     Length      |    Checksum     |
+--------+--------+--------+--------+
|
|          data octets ...
+---------------- ...
```

## TCP/IP applications

QoS (quality of service) is what an application requires from the network.

Before the mid 1990s voice and video used totally separate networks; today they run over IP. VoIP's QoS demands:

* low delay (< 200 ms)
* low jitter (< 30 ms), i.e. variation in delay
* low loss -- a lost VoIP packet is not retransmitted

HTTP commands and responses:

* GET request -- a request from a client to get a file from a web server
* the server sends a response with code 200 (meaning OK) together with the file contents
* 404 -- file not found

See [HTTP protocol](https://github.com/jreisinger/blog/blob/master/posts/http.md) for more.

## Sources

* W. Odom: CCENT/CCNA ICND1 (2012)
* ULSAH (UNIX and Linux System Administration Handbook), Ch. 14: TCP/IP Networking
* [RFC 791 - IP](https://www.rfc-editor.org/rfc/rfc791)
* [RFC 793 - TCP](https://www.rfc-editor.org/rfc/rfc793)
* [RFC 768 - UDP](https://www.rfc-editor.org/rfc/rfc768)

This post was made from [my old notes](https://github.com/jreisinger/docs/blob/master/notes/net/basics.md).
