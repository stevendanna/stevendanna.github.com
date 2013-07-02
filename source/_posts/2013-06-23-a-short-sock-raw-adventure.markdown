---
layout: post
title: "A Short SOCK_RAW Adventure"
date: 2013-06-23 18:55
comments: true
categories: linux C networking
---

Last week a colleague and I were talking about custom network
protocols.  As I haven't done much networking programming in Linux, it
piqued my curiosity about how hard it is to send data using IP but not
one of the common transport protocols (TCP or UDP). Since I'm
currently on a plane without wifi, I've written up some of my notes on
this short bout of manpage surfing.

I quickly found that the [raw(7)](http://linux.die.net/man/7/raw) manual page has
documentation on just what I was looking for:

> Raw sockets allow new IPv4 protocols to be implemented in user
  space. A raw socket receives or sends the raw datagram not including
  link level headers.

The following is a minimal example that sends a single message, "Hello
World!\0", to localhost using the Internet Protocol and a raw `AF_INET`
socket:

``` c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdio.h>

int main() {
  int sd;
  struct sockaddr_in dest;
  char msg[] = "Hello, World!";

  dest.sin_family = AF_INET;
  if (inet_pton(AF_INET, "127.0.0.1", &(dest.sin_addr)) != 1) {
      printf("Bad Address!\n");
      return(1);
  }

  if ((sd = socket(AF_INET, SOCK_RAW, 253)) < 0) {
      printf("socket() failed!\n");
      return(1);
  }

  if (sendto(sd, &msg, 14, 0, (struct sockaddr*) &dest, sizeof(struct sockaddr)) < 0)  {
    printf("sendto() failed!\n");
    return(1);
  }

  return(0);
}
```

# inet_pton(), socket(), sendto()

This short program can be broken into three parts:

1. Specifying our destination,
2. Creating a socket, and
3. Sending a message over the socket.

The key system calls used to complete these tasks are `inet_pton`,
`socket()` and `sendto()`, respectively.

## inet_pton()

Our destination is stored in a `sockaddr_in` structure that has the
following definition (from [ip(7)](http://linux.die.net/man/7/ip)):

    struct sockaddr_in {
         sa_family_t    sin_family; /* address family: AF_INET */
         in_port_t      sin_port;   /* port in network byte order */
         struct in_addr sin_addr;   /* internet address */
    };

In our example, `sin_family` will be AF_INET as we are using IPv4.
Since port is a concept used by transport layer protocols such as UDP
and TCP and not used by IP, we'll ignore the port in this case.
Finally, the `sin_addr` field is the destination address in network
byte order.

Rather than figuring out how to turn a human-readable destination such
as "127.0.0.1" into its network byte-order representation, we use
[inet_pton](http://linux.die.net/man/3/inet_pton) to do the dirty work for us:

    inet_pton(AF_INET, "127.0.0.1", &(dest.sin_addr))

## socket()

The [socket](http://linux.die.net/man/2/socket) system call has the following signature:

    socket(int domain, int type, int protocol)

In this case, we want an `AF_INET` socket of type `SOCK_RAW`.  In the
context of an AF_INET socket, the protocol fields specifies which
protocol will be used on top of IP.  Since we don't want to use a
protocol on top of IP, I've used protocol 253 which RFC 3692 reserves
for experimental and testing purposes.

This will return a file descriptor that we can pass to sendto().

## sendto()

Finally, we use sendto() to send the desired message.  [sendto](http://linux.die.net/man/2/sendto) has the following
signature:

    sendto(int sockfd, const void *buf, size_t len, int flags,
           const struct sockaddr *dest_addr, socklen_t addrlen);

In the above example

`sockfd` is `sd`, the socket created by line 17, `*buf` is the
message defined at line 9 and `len` is its length. Flags is set to 0
as we don't need special flags this case.

`*dest_addr` is our destination specified as a `sockadd_in`
structure.  This contains the destination address we packed into it
using `inet_pton`.

# Verification with TCP Dump

If we compile and run this program, we can capture the incoming
packet and verify that we didn't send anything but our IP header and
the payload "Hello, World!\0".

    $ gcc -Wall send_msg.c
    $ bash -c 'sleep 10; sudo ./a.out'
    $ sudo tcpdump -i lo -X
    tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
    listening on lo, link-type EN10MB (Ethernet), capture size 65535 bytes

    20:46:26.470243 IP localhost > localhost:  ip-proto-253 14
            0x0000:  4500 0022 0000 4000 40fd 3bdd 7f00 0001  E.."..@.@.;.....
            0x0010:  7f00 0001 4865 6c6c 6f2c 2057 6f72 6c64  ....Hello,.World
            0x0020:  2100                                     !.

In hex our payload is:

    $ printf 'Hello, World!\0' | hexdump
    0000000 6548 6c6c 2c6f 5720 726f 646c 0021

which we can see at the end of the packet.  Assuming that our packet
starts with the IP header,
[RFC 791](https://www.rfc-editor.org/rfc/rfc791.txt) specifies that
the first 4 bits is the versions field and the second four bits is the
"Internet Header Length".  Here, we can see the version field is set
to 4 and the IHL is 5. According to the RFC

> Internet Header Length is the length of the internet header in 32
  bit words, and thus points to the beginning of the data.

Five 32-bit words is 160 bits.  160 bits is 40 hexadecimal digits.
That accounts for the rest of the packet we receive, confirming that
the packet we sent is nothing more than an IP header and our desired
payload.

# Verification with Socat

If we wanted to see our payload via something other than tcpdump we
can use the swiss-army knife of socket communication, [`socat`](http://www.dest-unreach.org/socat/):

In one shell:

    sudo socat IP4-RECVFROM:253 -

In another:

    sudo ./a.out

Back in our original shell we should see:

    Hello, World!

In fact, we can use socat to send the same payload using only IPv4:

    printf 'Hello, World!\0' | sudo socat - IP4-SENDTO:localhost:253

Using TCP dump, we can see the socat sent the same packet we did:

    21:16:35.729037 IP localhost > localhost:  ip-proto-253 14
        0x0000:  4500 0022 0000 4000 40fd 3bdd 7f00 0001  E.."..@.@.;.....
        0x0010:  7f00 0001 4865 6c6c 6f2c 2057 6f72 6c64  ....Hello,.World
        0x0020:  2100                                     !.


# Further Exploration and Notes

## Digging Deeper

If we wanted to dig deeper, Linux provides a couple of ways to dig
even deeper into the network stack from userspace:

- Using `setsockopt()` we can set the `IP_HDRINCL` option.  When this
  option is set, we can provide our own IP header information for each
  message we want to send.

- Using an AF_PACKET it is possible to send packets at the device
  driver level. See [packet(7)](http://linux.die.net/man/7/packet) for
  more details.

## IPv6

Using the `getaddrinfo()` function we can support both IPv4 and IPv6
addresses. A short, un-elaborated example is below.

``` c
#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <netdb.h>
#include <string.h>
#include <stdio.h>

int main(int argc, char *argv[]) {
  int sd;
  struct addrinfo hints;
  struct addrinfo *res;

  if (argc != 3) {
    fprintf(stderr, "usage: send_msg IP MSG\n");
    return(1);
  }

  memset(&hints, 0, sizeof(struct addrinfo));
  hints.ai_family = AF_UNSPEC;
  hints.ai_socktype = SOCK_RAW;
  hints.ai_protocol = 253;
  if (getaddrinfo(argv[1], NULL, &hints, &res) != 0) {
    fprintf(stderr, "getaddrinfo() failed!\n");
  }

  if ((sd = socket(res->ai_family, res->ai_socktype, res->ai_protocol)) < 0) {
    fprintf(stderr, "socket() failed!\n");
    return(1);
  }

  if (sendto(sd, argv[2], strlen(argv[2]), 0, res->ai_addr, res->ai_addrlen) < 0) {
    fprintf(stderr, "sendto() failed!\n");
  }

  return(0);
}
```
## The real Internet

If you try most of the small examples here with two hosts on a real
network, you may find that you message doesn't reach its destination.
Much of the network equipment it has to traverse is likely doing
transport layer inspection and dropping your message.

## References

- [raw(7) man page](http://linux.die.net/man/7/raw)
- [ip(7) man page](http://linux.die.net/man/7/ip)
- [socket(7) man page](http://linux.die.net/man/7/socket)
- [packet(7) man page](http://linux.die.net/man/7/packet)
- [sendto(2) man page](http://linux.die.net/man/2/sendto)
- [inet_pton(3) man page](http://linux.die.net/man/3/inet_pton)
- [getaddrinfo(2) man page](http://linux.die.net/man/2/getaddrinfo)
- [RFC 791](https://www.rfc-editor.org/rfc/rfc791.txt)
