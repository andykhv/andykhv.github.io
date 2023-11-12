---
title: "Resolving Domain Names"
date: 2023-11-11T12:00:00-07:00
draft: false
---

# Resolving Domain Names
This article is rather a summary of what I've learned when implementing my very own [DNS recursive resolver](https://github.com/andykhv/recursive_resolver). Albeit implementation is quite rudimentary and lacks many of the features described in RFC1035[^2]

From a bird's eye view, DNS manages namespaces with a hierarchical structure. These namespaces are provided throughout a distributed system architecture.[^1] There's not one single server that owns all records of every known domain and sub-domain out there. Instead there are numerous name servers that hold their individual authoritative information. I hope that from reading this article, we'll understand a bit more about DNS and its host address resolution.

### UDP

DNS interfaces through the User Datagram Protocal (UDP).[^1] UDP is a rather minimal transport protocol that operates in layer 4 of the [OSI Model](https://en.wikipedia.org/wiki/OSI_model). With UDP, DNS servers send/receive DNS messages without the need of establishing connections among one another.

There are some caveats with UDP though. Its non-need of a connection doesn't guarantee a strict order of packets received or delivery of the DNS message at the very least. Despite these implications, DNS messages are independent from one another, so order isn't needed at all. Each DNS message is an encapsulation of a query and response. UDP is rather a good choice then. The lack of order in UDP doesn't necessarily affect the functionality of DNS address resolution. Well, there is one caveat; DNS messages over UDP were originally limited to 512mb. There are some RFCs that explain solutions to resolve this issue. Such as DNS over TLS[^6] and EDNS[^3], but we're focused mainly on **RFC1034**[^1] and **RFC1035**[^2] here.

## Digging through DNS

Let's start figuring out the host address for *google.com*! Luckily MacOS comes with the **dig** cli tool:

`dig @8.8.8.8 -p 53 +noedns google.com`

We are using the **dig** utility to send a DNS packet to address *8.8.8.8:53* to resolve *google.com* for its host address. *8.8.8.8* is google's IPv4 address for their DNS server. We use **port 53** because this is the designated port for DNS servers.[^2] **Extension DNS** is out of scope for this article, since it is used to extend functionality for DNS messages.[^3] Hence we set the **+noedns** flag.

After entering the above command, we get this truncated response:
```
  ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 6741
  flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0 
  ;; QUESTION SECTION:
  ;google.com.			IN	A
  ;; ANSWER SECTION:
  google.com.		257	IN	A	142.250.189.14
```

Let's slowly digest this response. This response can be broken down into three parts:
1. Header
2. Questions
3. Answers

### Header
```
  ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 6741
  flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0 
```

The header of a DNS message contains data about the overall message. For this instance, the header states: 
- **opcode: QUERY**, the DNS message is a standard query for a domain name.
  - More information about the query itself is contained in the **QUESTION** section.
- **status: NOERROR**, the DNS message contains no errors.
- **flags: qr rd ra**
  - **qr** flag is set because this message is a query response.
  - **rd** flag is set during creation of the query message. It is set when the requester asks the target server to answer the query recursively.
  - **ra** flag is set during the response of the query. It denotes that recursion is available from the server. 
- **QUERY: 1**, there is one record in the **QUESTION** section.
- **ANSWER: 1**, there is one record in the **ANSWER** section.

### Questions 
```
  ;; QUESTION SECTION:
  ;google.com.			IN	A
```

This section of the message is rather simple. We have one question record to detail our query for *google.com*.

- **IN** represents that this question record relates to the internet class.
  - There are other classes that represent the **CHAOS** class (CH) and Hesiod (HS), but these are not relevant for this article.
- **A** represents that this question record is a query for a host address.

Overall, this question record is a query for the internet host address of *google.com*.

### Answers 
```
  ;; ANSWER SECTION:
  google.com.		257	IN	A	142.250.189.14
```

The answer section contains one answer record for the query for *google.com*. This record conveys **142.250.189.14** is the IPv4 host address for *google.com*. The Time-To-Live (TTL) is also conveyed in the record -- **257** seconds. Meaning, the name server that owns this resource record will refresh the information in 257 seconds.

## Is DNS Resolution that simple?
This article previously mentions DNS is a distributed system of name servers. We used **dig** to send a DNS query packet to *8.8.8.8:53* and received a response with an answer record. This seems a bit too simple for a "distributed system", right? As a matter of fact, *8.8.8.8* is not necessarily a name server. It is Google's public DNS resolution service. *8.8.8.8* is doing all of the heavy lifting for us!

So, what was *8.8.8.8* really doing? If we assume no previous knowledge or cache existed, *8.8.8.8* was resolving our query recursively. It first sent the query to a **root server**. There are 13 root servers in the Internet managed by different entities.[^4] Recursive resolution of a domain name starts with the root server. The root server sends us a response with name servers that can be used for the next query to get closer to our answer. This step occurs repeatedly until we get an answer record in the response packet.

## Resolving Recursively
Example query to a root server: `dig @a.root-servers.net +noedns google.com`

Truncated response:
```
  ;; Truncated, retrying in TCP mode.
  ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 13422
  ;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 13, ADDITIONAL: 26
  ;; QUESTION SECTION:
  ;google.com.			IN	A
  ;; AUTHORITY SECTION:
  com.			172800	IN	NS	e.gtld-servers.net.
  com.			172800	IN	NS	b.gtld-servers.net.
  ;; ADDITIONAL SECTION:
  e.gtld-servers.net.	172800	IN	A	192.12.94.30
  e.gtld-servers.net.	172800	IN	AAAA	2001:502:1ca1::30
  b.gtld-servers.net.	172800	IN	A	192.33.14.30
  b.gtld-servers.net.	172800	IN	AAAA	2001:503:231d::2:30
;; MSG SIZE  rcvd: 824
```

The first thing to notice from the response is that dig retried the query with a TCP connection instead.[^5] This is due to the response size being 824MB. As mentioned previously, the maximum packet size for DNS UDP is 512mb.

Comparatively to the response we received from *8.8.8.8*, there is no Answer section. Instead, there are 26 Additional resource records to accompany the 13 Authority resource records. The response shown is truncated. The Authority section contains records of Name Servers pertaining to the *com* domain. In other words *e.gtld-servers.net* and *b.gtld-servers.net* are name servers that contain records of domains that end with *com*, such as *google.com*. The additional section describes the **IPv4 (A)** and **IPv6 (AAAA)** addresses of the name servers.

Querying *e.gtld-servers.net* with `dig @e.gtld-servers.net +noedns +norecurse google.com`.

This is the truncated response:
```
  ;; AUTHORITY SECTION:
  google.com.		172800	IN	NS	ns2.google.com.
  ;; MSG SIZE  rcvd: 276
```

We received a resource record indicating that *ns2.google.com* is a Name Server that holds sub-domain information for *google.com*. Let's query this server next: `dig @ns2.google.com +noedns +norecurse google.com`.

This is the truncated response:

```
  ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 46238
  ;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
  ;; ANSWER SECTION:
  google.com.		300	IN	A	142.250.176.14
```

We finally received an Answer resource record! At the time of this query, *google.com* is mapped to *142.250.176.14*. Resolving *google.com* took 3 queries to complete. A query to a root server, *com* server, and finally a *google.com* server.

## Learnings
Resolvement of a domain name requires recursive queries to different name servers, one after another. Each name server responding to the query with either Authority records or the final answer record. *a.root-servers.net* is one of the initial root servers to query. Usually these name servers will respond with Authority resource records pertaining to the top-level domain of the domain name in question. *e.gtld.servers.net* and *b.gtld.servers.net* are authorities of com. *ns2.google.com* is an authority of google.com.

The diagram below depicts the hierarchical structure of domain names.

![diagram of domain names](/domainnames.png)

It was also quite fun to implement a recursive resolver in Rust! Check out my [recursive resolver here!](https://github.com/andykhv/recursive_resolver)

[^1]: [RFC 1034: DNS Concepts](https://www.ietf.org/rfc/rfc1034.txt)
[^2]: [RFC 1035: DNS Specification](https://www.ietf.org/rfc/rfc1035.txt)
[^3]: [RFC 2671: Extensions for DNS](https://www.ietf.org/rfc/rfc2671.txt)
[^4]: [RFC 7858: DNS over TCP](https://www.ietf.org/rfc/rfc7766.txt)
[^5]: [Root Servers](https://root-servers.org/)
[^6]: [TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol)
