---
title: "Understanding DNS"
date: 2023-09-16T23:03:19-07:00
draft: true
---

# Understanding DNS

This article is definitely not an authoratiative source of the **Domain Name System (DNS)**, but rather a summary of what I've learned when implementing my very own [DNS recursive resolver](https://github.com/andykhv/recursive_resolver).

## Overview
From a bird's eye view, DNS is very much of a distributed system of servers that interface through the User Datagram Protocal (UDP).

### UDP

UDP is a rather minimal transport protocol that operates in layer 4 of the [OSI Model](https://en.wikipedia.org/wiki/OSI_model). DNS doesn't seem to have a need for application layer protocols like HTTP. This would really just increase overhead of parsing DNS messages. With UDP, DNS servers send/receive DNS messages without the need of establishing connections among one another.

There are some caveats with UDP though. Its non-need of a connection doesn't guarantee a strict order of packets received or delivery of the DNS message at the very least. Despite these implications, DNS messages are independent from one another. In other words, each DNS message is an encapsulation of the query and response. UDP is rather a good choice then. The lack of order in UDP doesn't affect the functionality and requirements of DNS.

What if DNS operated via the **Transmission Control Protocol (TCP)** instead of UDP? TCP guarantees a reliable connection between client and server, in which packets are transmitted in an ordered stream. This connection is established after a three-way handshake. Now, if DNS operated solely through TCP, there wouldn't be much of a change in functionality. An ordered stream of DNS messages is a bit overkill, because these DNS messages are independent from another. Hypothetically in comparison to UDP, TCP's three-way handshake would increase the latency for DNS communication. The internet would be a tad bit slower if TCP was the common transport method.

## Digging through DNS
`dig @8.8.8.8 -p 53 +noedns google.com`

We are using the dig utility to send a DNS packet to address *8.8.8.8:53* to resolve the name *google.com*

8.8.8.8 is google's IPv4 address for their DNS server. We use port 53 because this is the designated port for DNS servers.[^1]

Here is the truncated response given:
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


## References
[^1]: [RFC1035](https://www.ietf.org/rfc/rfc1035.txt)