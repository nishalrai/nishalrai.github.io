---
title: Introduction to DNS | How it works
author: nishalrai
date: 2023-08-1 20:55:00 +0800
categories: [F5 Networks, DNS]
tags: [getting started, dns]
---

This would be a series of DNS service to provide a holistic view of DNS working, configuration practices, security threats, and mitigation. As one should be aware, DNS has been a really engaging topic to explore, and I will be documenting the useful findings about it.

One of the best approaches to digging into the topic is the Golden Circle rule: what, How, and Why. Before we jump into the usual way of learning DNS, we need to know why we even need it in the first place.

We may be aware that computers or electronic devices communicate in binary form, this addresses the use of IP addresses to distinguish computers in a network consisting of various computers. The Internet is a simple group of computer networks integrated with various routing protocols and computers to facilitate communication between end users. Let’s skip the different routing protocols, network architecture, and other relevant details of networks for now.

Let’s consider a scenario where there are multiple networks that can communicate with each other, as illustrated in the below diagram. So, when a user tries to communicate with another user via an electronic device on the network, if they know the IP address of each other, they can communicate with ease, but what if they need to communicate with different users that reside on different networks?

This can end up being a tedious job to remember and manually type the ip address of each user with whom they want to communicate. Also, one needs to know the IP address of the intended user to be able to communicate with them and even if they, it would be very difficult to manage those records by an individual. This really obscures the actual usage of the internet instead of ease of use, and it brings several challenges for the user in terms of communication. The proliferation of computers network will exhaust the trivial option to establish communication between the users, and a new solution — DNS was proposed.

DNS solved the two major challenges for the user: centralized entity to record the new IP addresses in a dynamic networks and easily navigating to those intended IP addresses by associating them with a human-readable format. This helped to avoid the multiple challenges due to the initial trivial practice of using the IP address to communicate with the user. So, this is a general overview of the origin of the DNS in the computer network.


# What is DNS?
DNS refers to the Domain Name System and acts as the phonebook of the Internet. The user accesses the information online through domain names and retrieves the IP address associated with those domain names, which will be used to navigate and communicate between the users. Each device connected to the internet has a unique IP address, which other devices use to find the device in a computer network—the Internet. A domain or domain name is the location of the resource — website or service — microsoft.com that points to the IP address “20.76.201.171”

## General DNS terms
It’s important to be familiar with few general terms for better understanding of the DNS. DNS works in a hierarchical manner where Root (“.”) resides at the top and domain names as leaves underneath. The administration is shared where authority is delegated where no single entity is in charge.

There are 13 root servers (a-m.root-servers.net) and more than a thousand instances around the world. Those instances has been clustered and configured anycast across diverse geographical position. Next level of names are called Top Level Domains (TLSDs).

The general types are:

- TLD: Top Level Domains (.com, .net, .edu, .org, etc)
- ccTLD: Country Code TLD (2 letter country codes: .us, .bd, etc)
- Infrastrucuture: .arpa (usage: reverse DNS)
- IDN: Internationalized Domain name
- The new gTLD: Generic TLD (.tourism, .museum. .dubai, etc) newgtlds.icann.org

Illustration of the Parent-Child Relationship of the Domain

An administrator of a domain can delegate responsibility for managing a subdomain to someone else. The parent domain retains links to the delegated subdomain where the parent domain “remembers” who it delegated the subdomain to.

A domain is a hierarchical naming structure used to identify computers, services, or resources connected to the Internet (or an entire subtree), while a DNS zone is a portion of the DNS namespace that is administered by a single entity and contains a collection of resource records defining the DNS mappings for the domain names within the zone. “ Delegation is the boundary between zones and also refers to Zone Cuts.”


The major DNS components can be categorized into two:

<br>
## Server Side
**Authoritative Servers**
The definitive source of information for a domain name’s DNS records. It is responsible for providing accurate and up-to-date answers to DNS queries for that domain. When a DNS resolver sends a query to resolve a domain name, the query is passed from one DNS server to another until it reaches the authoritative nameserver for the domain. The authoritative nameserver is responsible for returning the correct answer to the query, which includes the IP address for the requested hostname if it has access to the requested record.
There are two times of Authoritative servers – Primary and Secondary Authoritative Servers.

**Resolvers (Recursive Resolvers)**
A server is designed to receive DNS queries from client machines and is responsible for making additional requests to other DNS servers in order to resolve the queries and provide the requested information. DNS recursive resolvers help to speed up the DNS resolution process by caching frequently requested DNS information and reusing it for subsequent requests. This can help to reduce the overall time it takes to resolve DNS queries and improve the performance of applications that rely on DNS, such as web browsers.
DNS response data includes a TTL values, defined in seconds and each second, the TTL is decremented by 1. When TTL reaches to 0 value then the record is expired and the immediate DNS request for that data will require a new query be performed.


## Client Side
Stub resolvers
A software component that is responsible for initiating DNS queries on behalf of a client application and relies on a DNS resolver to provide the requested information. Stub resolvers is typically part of an operating system’s networking stack and is used by applications to resolve domain names into IP addresses. Recursive Resolvers are configured on stub resolver that is uses to lookup “resolve” domain names.  Almost every stub resolver caches results to reduce lookup time, modern browsers also cache results for the same reason.

When the web browser requests the OS to know whether the requested domain is known and if not then checks their cache before asking the OS. If still not found then the OS looks in the resolver config for the server to send the DNS query.

When a user enters a domain name into their web browser then the browser first checks its own cache to see if it has the corresponding IP address. If the browser does find its corresponding IP address, then it sends a request to a DNS resolver which is typically provided by the user’s Internet Service Provider (ISP) or a third-party DNS provider. The DNS resolver then sends a query to a DNS root server, which responds with a referral to a Top-Level Domain (TLD) server.

For an instance, if the domain name is www.google.com, the DNS resolver will query the root server for the .com TLD. The TLD server then responds with a referral to an authoritative DNS server for the domain name. In the case of www.google.com, the authoritative DNS server would be operated by Google. The authoritative DNS server responds to the query with the IP address for the requested domain name, which is then returned to the user’s browser and used to initiate a connection to the server hosting the website.

A simple analogy of the above illustrated DNS working mechanism:

Client
When the user enters the domain on the url of the browser, it checks on its stub resolver and then to its cache. Then ultimately sends the query to its recursive server.
Recursive Resolver
It acts as the person, who is asked to go find a particular item in a store by the customer–client. So, the recursive resolver is the receiver of the client queries via the web browser and is responsible for making additional requests to fulfill the client’s DNS query.
Root nameserver
This can be regarded as an index in a store that points to different categories (food, clothes, others) of items. The initial point responsible to translate or resolve human-readable format hostnames into IP addresses.
TLD nameserver
Denotes the specific category of items in a store. This nameserver looks for a specific IP address and it hosts the last portion of a hostname – google.com, the TLD server is .com
Authoritative nameserver
The searched item of the specific category. It is the final stop in the nameserver query. If the nameserver has access to the requested record then it will return the IP address for the requested hostname back to the DNS recursive server that made the initial request.
The detail of the root nameserver:


```dns
.                                   3600000           IN         NS        a.root-servers.net.
.                                   3600000           IN         NS        b.root-servers.net.
.                                   3600000           IN         NS        c.root-servers.net.
.                                   3600000           IN         NS        d.root-servers.net.
.                                   3600000           IN         NS        e.root-servers.net.
.                                   3600000           IN         NS        f.root-servers.net.
.                                   3600000           IN         NS        g.root-servers.net.
.                                   3600000           IN         NS        h.root-servers.net.
.                                   3600000           IN         NS        i.root-servers.net.
.                                   3600000           IN         NS        j.root-servers.net.
.                                   3600000           IN         NS        k.root-servers.net.
.                                   3600000           IN         NS        l.root-servers.net.
.                                   3600000           IN         NS        m.root-servers.net.
a.root-servers.net.       3600000           IN         A          198.41.0.4
a.root-servers.net.       3600000           IN         AAAA    2001:503:ba3e:0:0:0:2:30
b.root-servers.net.       3600000           IN         A          199.9.14.201
b.root-servers.net.       3600000           IN         AAAA     2001:500:200:0:0:0:0:b
c.root-servers.net.       3600000           IN         A          192.33.4.12
c.root-servers.net.       3600000           IN         AAAA     2001:500:2:0:0:0:0:c
d.root-servers.net.       3600000           IN         A          199.7.91.13
d.root-servers.net.       3600000           IN         AAAA      2001:500:2d:0:0:0:0:d
e.root-servers.net.       3600000           IN         A          192.203.230.10
e.root-servers.net.       3600000           IN         AAAA       2001:500:a8:0:0:0:0:e
f.root-servers.net.        3600000           IN         A          192.5.5.241
f.root-servers.net.        3600000           IN         AAAA       2001:500:2f:0:0:0:0:f
g.root-servers.net.       3600000           IN         A          192.112.36.4
g.root-servers.net.       3600000           IN         AAAA       2001:500:12:0:0:0:0:d0d
h.root-servers.net.       3600000           IN         A          198.97.190.53
h.root-servers.net.       3600000           IN         AAAA       2001:500:1:0:0:0:0:53
i.root-servers.net.         3600000           IN         A          192.36.148.17
i.root-servers.net.         3600000           IN         AAAA       2001:7fe:0:0:0:0:0:53
j.root-servers.net.         3600000           IN         A          192.58.128.30
j.root-servers.net.         3600000           IN         AAAA       2001:503:c27:0:0:0:2:30
k.root-servers.net.        3600000           IN         A          193.0.14.129
k.root-servers.net.        3600000           IN         AAAA       2001:7fd:0:0:0:0:0:1
l.root-servers.net.         3600000           IN         A          199.7.83.42
l.root-servers.net.         3600000           IN         AAAA       2001:500:9f:0:0:0:0:42
m.root-servers.net.      3600000           IN         A          202.12.27.33
m.root-servers.net.      3600000           IN         AAAA        2001:dc3:0:0:0:0:0:35

Reference link:
https://www.internic.net/domain/named.root
```

The above illustration (discussed in ipv4 format, as it works in the same way) is described in the following steps:

The user enters the domain name www.example.com into their browser and query.
The browser checks its own cache to see if it has the corresponding IP address for www.example.com. If it does, it uses that IP address to initiate a connection to the server hosting the website. If it does not have the IP address, it sends a request to the recursive resolver.
Recursive resolver receives the query for www.example.com from stub resolver and checks its cache to see if it has the corresponding IP address. If it does not have the IP address, it starts the process of resolving the query and forwards to the root nameserver.
Root nameserver provides a referral to a top-level domain (TLD) nameserver based on the TLD of the domain name being queried. In this case, the TLD is “.com”
The recursive resolver sends a query to the TLD nameserver for “.com”.
The TLD nameserver is responsible for providing a referral to the authoritative nameserver for the domain www.example.com.
Recursive resolver sends a query of the domain www.example.com to the received referral of the authoritative nameserver.
The authoritative nameserver provides the IP address for www.example.com to the recursive resolver.
The recursive resolver caches the IP address and returns it to the stub resolver.
The stub resolver caches the IP address and returns it to the browser. The browser uses the IP address to initiate a connection to the server hosting the website.


## Reference
- https://www.cloudflare.com/learning/dns/what-is-dns/
- https://www.youtube.com/watch?v=WkXSx-Tuzlo&t=43s
- https://learn.microsoft.com/en-us/windows-server/networking/dns/dns-top

> Most of the images used have been taken from the APNIC – DNS training session by Philip Paeps