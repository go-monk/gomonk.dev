+++
date = '2026-10-12T09:00:00+02:00'
title = 'Watching the TCP handshake from Go'
+++

Every TCP connection starts with three segments[^1] going back and forth before any data is sent. `net.Dial` hides them: you call it and get a connection. This post is a 74-line Go program, standard library only, that connects to itself and prints those three segments as the kernel sends them. It needs root on Linux, or Docker.

## The handshake

TCP numbers the bytes it sends, so the receiver can put them in order and say which ones arrived. Before sending data, each side has to tell the other the number it starts counting from[^2]. That takes three segments, known as the three-way handshake:

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

1. the client sends a segment with the SYN (synchronize) flag and its starting sequence number
2. the server replies with its own SYN and starting number, and sets the ACK (acknowledge) flag to confirm the client's
3. the client confirms the server's number with an ACK

The acknowledgment number is the next sequence number the sender expects, which is why 200 is answered with 201.

## Getting at the segments

The kernel does the handshake, a program normally sees only the result. To see the segments I open a raw socket. On Linux, a raw socket for protocol `tcp` gets a copy of every TCP segment arriving at its address, header included. In Go that's `net.ListenIP("ip4:tcp", ...)`, and it's the part that needs root.

Then the program needs a connection to watch, so it listens on a free port on 127.0.0.1. The program also dials it, so it's running both a server and a client. Both ends are on the loopback interface, where every segment is an arriving one, so the raw socket sees both directions.

What it reads is the TCP header. The fields I'm interested in sit at fixed offsets, all in big-endian byte order[^3]:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |  bytes 0-3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |  bytes 4-7
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |  bytes 8-11
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |           |U|A|P|R|S|F|                               |
| Offset| Reserved  |R|C|S|S|Y|I|            Window             |  bytes 12-15
|       |           |G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The six flags are the low bits of byte 13, with FIN as the lowest.

## The program

```go
// Handshake connects to itself over TCP and prints the segments it sees.
package main

import (
	"encoding/binary"
	"fmt"
	"log"
	"net"
	"strings"
)

func main() {
	// A raw socket gets a copy of every
	// TCP segment arriving at the address.
	raw, err := net.ListenIP(
		"ip4:tcp",
		&net.IPAddr{IP: net.IPv4(127, 0, 0, 1)},
	)
	if err != nil {
		log.Fatal(err)
	}
	defer raw.Close()

	server, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		log.Fatal(err)
	}
	defer server.Close()
	// Get the port chosen by the kernel.
	port := uint16(server.Addr().(*net.TCPAddr).Port)

	client, err := net.Dial("tcp", server.Addr().String())
	if err != nil {
		log.Fatal(err)
	}
	defer client.Close()

	// Plenty: a handshake segment is at most 60 bytes.
	buf := make([]byte, 1500)
	for seen := 0; seen < 3; {
		n, _, err := raw.ReadFrom(buf)
		if err != nil {
			log.Fatal(err)
		}
		if n < 20 {
			continue // shorter than a TCP header
		}
		seg := buf[:n]
		sport := binary.BigEndian.Uint16(seg[0:2])
		dport := binary.BigEndian.Uint16(seg[2:4])
		if sport != port && dport != port {
			continue // somebody else's connection
		}
		seq := binary.BigEndian.Uint32(seg[4:8])
		ack := binary.BigEndian.Uint32(seg[8:12])
		fmt.Printf(
			"%5d -> %-5d %-8s SEQ=%d ACK=%d\n",
			sport, dport, flags(seg[13]), seq, ack)
		seen++
	}
}

// flags gives human-readable names to the bits
// set in the flags byte of a TCP header.
func flags(b byte) string {
	var names []string
	all := []string{"FIN", "SYN", "RST", "PSH", "ACK", "URG"}
	for i, name := range all {
		if b&(1<<i) != 0 {
			names = append(names, name)
		}
	}
	return strings.Join(names, ",")
}
```

Three things that aren't obvious from the code:

* The raw socket is opened before the dial. The kernel queues the segments on it, so they are still there to read after `net.Dial` has returned.
* The server never calls `Accept`. The kernel completes the handshake on its own and keeps the connection in a queue until somebody accepts it.
* Port 0 in `127.0.0.1:0` asks the kernel for any free port.

## Running it

On Linux build it and run the binary with `sudo`. On a Mac, or to avoid `sudo`, run it in a container, where you are root already:

```sh
$ vi handshake.go # copy code from above
$ docker run --rm -v $PWD:/src -w /src golang go run handshake.go
47462 -> 34347 SYN      SEQ=3175403979 ACK=0
34347 -> 47462 SYN,ACK  SEQ=713211396  ACK=3175403980
47462 -> 34347 ACK      SEQ=3175403980 ACK=713211397
```

The client (port 47462) starts counting at 3175403979 and the server acknowledges 3175403980. The server starts at 713211396 and the client acknowledges 713211397. The acknowledgment field of the first segment is zero because nothing has been received yet, and without the ACK flag it's ignored anyway.

The starting numbers aren't 200 and 1450, and they differ on every run. The kernel picks them unpredictably, so that somebody who can't see the connection can't guess the numbers and inject segments into it.

## Limits

This works only when the program talks to itself. A raw socket gets the segments that arrive, not the ones that leave. To watch both directions of a real connection you need a packet capture tool such as `tcpdump`.

Without root the first call fails:

```
listen ip4:tcp 127.0.0.1: socket: operation not permitted
```

## Sources

* [RFC 793 - TCP](https://www.rfc-editor.org/rfc/rfc793), section 3.4 on establishing a connection
* [raw(7)](https://man7.org/linux/man-pages/man7/raw.7.html), the Linux manual page on raw sockets

[^1]: A segment is a TCP header plus data, TCP's unit of transfer. The handshake segments carry no data.
[^2]: On a real network the starting numbers are much bigger than the 200 and 1450 used below.
[^3]: Big-endian means the most significant byte comes first, the same left-to-right order you read the number in. So the 16-bit port 443 (0x01BB) goes over the wire as 0x01 then 0xBB. Network protocols use big-endian order, which is why it is also called network byte order. x86 and most ARM systems are little-endian: they store the least significant byte first.
