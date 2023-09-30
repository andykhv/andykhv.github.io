---
title: "Understanding DNS"
date: 2023-09-16T23:03:19-07:00
draft: true
---

# Understanding DNS
This article is definitely not an authoratiative source of the **Domain Name System (DNS)**, but rather a summary of what I've learned when implementing my very own [DNS recursive resolver](https://github.com/andykhv/recursive_resolver).

From a bird's eye view, DNS manages namespaces with a hierarchical structure. These namespaces are provided throughout a distributed system architecture.[^1] There's not one single server that owns all records of every known domain name out there. Instead there are numerous name servers that hold their authoritative information. I hope that from reading this article, we'll understand a bit about host address resolution from DNS.

### UDP

DNS interfaces through the User Datagram Protocal (UDP). UDP is a rather minimal transport protocol that operates in layer 4 of the [OSI Model](https://en.wikipedia.org/wiki/OSI_model). DNS doesn't seem to have a need for application layer protocols like HTTP. This would really just increase overhead of parsing DNS messages. With UDP, DNS servers send/receive DNS messages without the need of establishing connections among one another.

There are some caveats with UDP though. Its non-need of a connection doesn't guarantee a strict order of packets received or delivery of the DNS message at the very least. Despite these implications, DNS messages are independent from one another. In other words, each DNS message is an encapsulation of the query and response. UDP is rather a good choice then. The lack of order in UDP doesn't affect the functionality and requirements of DNS.

Another caveat is the packet size limit of 512MB.[^5] So, does this mean DNS packets are limited to 512MB? Even with truncation, there will be scenarios where a DNS packet's size is greather than 512MB. If this scenario occurs, DNS servers establish a TCP connection instead. The connection is established after a three-way handshake. TCP guarantees a reliable connection between client and server, in which packets are transmitted in an ordered stream. With this transport protocol, a DNS response can be sent through multiple broken-down packets through an ordered stream to the requesting client.

## Digging through DNS

Let's start with figuring out the host address for *google.com*! Luckily MacOS comes with the **dig** cli tool:

`dig @8.8.8.8 -p 53 +noedns google.com`

We are using the **dig** utility to send a DNS packet to address *8.8.8.8:53* to resolve *google.com* for its host address. *8.8.8.8* is google's IPv4 address for their DNS server. We use port 53 because this is the designated port for DNS servers.[^2] **Extension DNS** is out of scope for this article, since it is used to extend the size of several parameters in the DNS packet.[^3] Hence we set the `+noedns` flag.

After entering the above command, we get this response (truncated):
><TRUNCATED>
>->>HEADER<<- opcode: QUERY, status: NOERROR, id: 6741
>flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0 
>
>;; QUESTION SECTION:
>;google.com.			IN	A
>
>;; ANSWER SECTION:
>google.com.		257	IN	A	142.250.189.14
><TRUNCATED>

Let's slowly digest this response. This response can be broken down into three parts:
1. Header
2. Questions
3. Answers

### Header
>->>HEADER<<- opcode: QUERY, status: NOERROR, id: 6741
>flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0 

The header of a DNS message contains data about the overall message. For this instance, the header states: 
- `opcode: QUERY`, the DNS message is a standard query for a domain name.
  - More information about the query itself is contained in the **QUESTION** section.
- `status: NOERROR`, the DNS message contains no errors.
- `flags: qr rd ra`
  - `qr` flag is set because this message is a query response.
  - `rd` flag is set during creation of the query message. It is set when the requester asks the target server to answer the query recursively.
  - `ra` flag is set during the response of the query. It denotes that recursion is available from the server. 
- `QUERY: 1`, there is one record in the **QUESTION** section.
- `ANSWER: 1`, there is one record in the **ANSWER** section.

### Questions 
>;; QUESTION SECTION:
>;google.com.			IN	A

This section of the message is rather simple. We have one question record to detail our query for *google.com*.

- **IN** represents that this question record relates to the internet class.
  - There are other classes that represent the **CHAOS** class (CH) and Hesiod (HS), but these are not relevant for this article.
- **A** represents that this question record is a query for a host address.

Overall, this question record is a query for the internet host address of *google.com*.

### Answers 
>;; ANSWER SECTION:
>google.com.		257	IN	A	142.250.189.14

The answer section contains one answer record for the query for *google.com*. This record conveys **142.250.189.14** is the IPv4 host address for *google.com*. The Time-To-Live (TTL) is also conveyed in the record -- **257** seconds. Meaning, the name server that owns this resource record will refresh the information in 257 seconds.

## Is DNS Resolution that simple?
This article previously mentions DNS is a distributed system of name servers. We used **dig** to send a DNS query packet to *8.8.8.8:53* and received a response with an answer record. This seems a bit too simple for a "distributed system", right? As a matter of fact, *8.8.8.8* is not a name server. It is Google's public DNS resolution service. *8.8.8.8* is doing all of the heavy lifting for us!

So, what was *8.8.8.8* really doing? From my own understanding, it was resolving our query recursively. Assuming no previous knowledge or cache existed, *8.8.8.8* first sent the query to a **root server**. There are 13 root servers in the Internet managed by different entities.[^4] Recursive resolution of a domain name starts with the root server. The root server sends us a response with name servers that can be used for the next query to get closer to our answer. This step occurs repeatedly until we get an answer record in the response packet.

Example query to a root server:
`dig @a.root-servers.net +noedns google.com`

Example response from a root server:

[^1]: [RFC 1034](https://www.ietf.org/rfc/rfc1034.txt)
[^2]: [RFC 1035](https://www.ietf.org/rfc/rfc1035.txt)
[^3]: [RFC 2671](https://www.ietf.org/rfc/rfc2671.txt)
[^4]: [Root Servers](https://root-servers.org/)
[^5]: [Microsoft Docs on UDP and TCP](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/dns-works-on-tcp-and-udp)
