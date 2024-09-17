---
title: DNS Record Types | Hands on DNS authoritative server configuration
description: >-
  Get started with Chirpy basics in this comprehensive overview.
  You will learn how to install, configure, and use your first Chirpy-based website, as well as deploy it to a web server.
author: nisharai
date: 2023-08-04 20:55:00 +0800
categories: [F5 Networks, DNS]
tags: [integration, getting-started]
---

Before we jump into the dns record types, there are few terminology to be familiar with:

# Resource Records (RR)
A resource record (RR) is a data structure in DNS that contains information about a specific resource associated with a domain name, such as an IP address, mail server, or text. Resource records are used by DNS servers to resolve domain names into IP addresses and other types of data. In short, a grouping of a domain name, type, class, TTL and the associated resource data.

# Resource Records Sets (RRset)
A set of RRs with the same name and type where the rdata (resource data) associated with each RR in the set must be distinct. RR sets are treated atomically when returning responses.

The border line denotes the resource records and the whole highlighted denotes the resource record sets. On the above illustrated image, there are 8 resource records and 4 RRsets.

<br>

# Resource Record Types
There are several DNS resource record types which are enlisted below and described below:

## SOA: Start of Authority
A type of resource record in the DNS that specifies authoritative information about t a DNS zone, such as the email address of the administrator, when the domain was last updated, and others. SOA defines the start of a new Zone and always appears at the apex of the zone.
The serial number should be incremented on zone content updates.


## NS: Name Server Record
A type of resource record used to delegate a DNS zone to a set of authorative name servers. An NS record specifies the DNS servers that are authoritative for a particular domain or subdomain. Delegates a DNS subtree from the parent (i.e. creates a new zone)
It appears in both parent and child zones where rdata contains hostname of the DNS server.

There is no inherent concept of primary and secondary name server associated with NS records instead follows the order in which NS records are listed in the zone file for a domain determines the priority of the name servers.


## A: IPv4 Address Record
A type of resource record used to map a domain name to the IPv4 address of a server hosting the domain’s content. An A record contains the IP address of a specific server associated with a domain name.
rdata contains an IPv4 address.


## AAAA: IPv6 Address Record
A type of resource record used to map a domain name to the IPv6 address of a server hosting the domain’s content.


## CNAME: Canonical Name (Alias)
Used to alias one domain name to another. This allows multiple domains names to be associated with the same IP address or server. CNAME records can be used to create aliases for subdomains or for entirely different domain name.
Note:
CNAME records cannot coexist with other types of records for the same domain name. In other words, if a domain name has a CNAME record, it cannot also have an A or AAAA record for the same name.


When a name server fails to find a desired RR, it looks for a CNAME. If found, the CNAME is included in the response and the server restarts querying for the domain name specified in the RDATA of the CNAME record.


## MX: Mail Exchanger Record (IP to host)
To specify the mail server responsible for accepting email messages on behalf of a domain. An MX record contains the name and priority of one or more mail servers for a specific domain.
rdata consists of a preference field and the hostname of the mail receiver.


## PTR: Pointer (Reverse DNS info)
Used to map an IP address to a domain name and commonly used in reverse DNS (rDNS) lookups, which are used to associate an IP address with a domain name. Unlike other types of DNS records, which map domain names to IP addresses, a PTR record maps an IP address to a domain name.
IPv4 PTR record uses “in-addr.arpa.” subtree. The left hand side of the PTR record denotes the owner name constructed in the following format:
Reverse all onctets int e IPv4 address and make each octet a DNS label then append “in-addr.arpa.” to the domain name.


In case of IPv6 PTR record, it uses “ip6.arpa” subtree where the left hand side of the PTR record – owner name is constructed by the following method:
Expand and reverse all the hex digits, make each hex digit a DNS label then append “ip6.arpa.” to the domain name.



## TXT
A type of resource record used to store arbitrary text information about a domain that contains a text string for various purposes like providing information about the domain’s owner, verifying the domain ownership for use with certain services, or storing data associated with a specific service or application.

TXT records are commonly used to implement Sender Policy Framework (SPF) and DomainKeys Identified Mail (DKIM), which are email authentication mechanisms that help prevent spam and phishing.


## SRV: Service Location Record (host + port)
Used to specify the location of a specific service or server. SRV record contains information about a service or server, such as the hostname, port, number and priority that can be used by clients to locate the service or server.

SRV records are commonly used in conjunction with other types of DNS records such as  and AAAA records to provide more detailed information about a particular service or server. For instance, a web server with the domain name example.com, one can create an SRV record in the DNS zone file for _http._tcp.example.com that specifies the hostname and port number of the web server. When a client needs to connect to the web server, it can perform a DNS query for the SRV record to obtain the necessary information.
rdata contains priority, weight, port and server hostname.
Lower preference = high priority and higher weight = more often used


> rdata, known as “resource record data” refers to the data portion of a DNS resource record. The rdata field contain the information associated with a particular record type such as the IP address associated with a A record, the mail server name associated with an MX record or the text string associated with a TXT record.

The below diagram illustrates the architecture of the typical DNS authoraitative DNS server where each DNS authoraitative server are placed in different geographical area for redundancy and disaster recovery purpose. The server are categorized into primary and secondary mode where each category is clustered into a group and defined with an anycast IP address.

<br>

# Configuration of the authoritative DNS server
This guide will cover up the configuration of the authoritative DNS server as primary and secondary where all the relevant details like the used version of the dns-utils, environment and others are discussed in the following segment. Instance for Authoritative DNS server

- Instance for Authoritative DNS server
- Installation of Bind package on the instance
- Illustration of Config and zone file configuration
- Configuration as Primary authoritative DNS server
- Configuration on the /etc/named.conf
- Configuration as a Secondary authoritative DNS server
- Verification of both authoritative DNS server
- Bonus

Each section consists of the required details essential for the proper configuration of the authoritative DNS server.
<br>

## Instance for Authoritative DNS server
I’ve used the Azure service for two virtual instances and selected Rocky Linux 9 for the instance. One can follow the following steps:

Main Dashboard of Azure > Create Resource > Type Rocky Linux 9 on Search Button > Create

(You can either one network security group, public ip and NIC to be associated for the new Azure instance or create while creating the new azure instance)

I’ve created two Azure instance for two authoritative DNS server.

On the network security group, you need to enable both TCP and UDP port 53 where the regular DNS query uses DNS port 53 and the communication between the primary and secondary server requires TCP port 53.


## Installation of Bind package on the instance
The information of used Rocky Linux 9 Azure instance:

After performing the update on both of the new Azure instance, the bind dns-utils was installed by executing the command:

```Linux

sudo dnf update -y
sudo dnf install bind dnsutils -y
sudo systemctl start named
```

## Illustration of Config and zone file configuration
The below diagram illustrates the directory of the configuration files and its usage.


As the diagram illustrates the directory on the system where the configuration file related to the dns server are located:

- /etc/named.rfc1912zonesThis file is used by the BIND DNS server software to define the standard DNS zones, including the IN-ADDR.ARPA reverse lookup zone, the IP6.ARPA reverse lookup zone for IPv6 addressses and the local zone for configuring local hostnames. The file typically contains zone definitions that specify the domain name, the type of zone, the filename for the zone data and other configuration options.

- /etc/named.conf
This file contains configuration directives that specify how the DNS server should operate including information about the zones that the server is authoritative for, the location of zone data files and options that control the behavior of the server.

- /var/named
This directory consists the zone file (domain) including different DNS record type like NS, MX, PTR, and others.

- /var/named/data/named.run
This file consists of the logs relevant to the DNS server (kind of an audit log)

<br>
## Configuration as Primary authoritative DNS server
Once you’re logged in on the instance and as per the above illustration, browse to the /var/named and create a new directory – (domain namespace would be better) then create a new zone file for it.

Once the zone file has been created, the configuration will be done on the basis of the master zone file format:



```DNS

$TTL 86399 @  
IN  SOA     ns1.nitratic.com. root.nitratic.com. (         2023070800   ;Serial          3600        ;Refresh          1800        ;Retry          604800      ;Expire          86400       ;Minimum TTL )  

; Set your Name Servers here         
IN  NS      ns1.nitratic.com.         
IN  NS      ns2.nitratic.com.    

 ; Set each IP address of a hostname. Sample A records.
ns1       IN       A        20.235.87.247
ns2       IN       A        4.224.20.229
@         IN       A        202.166.197.69
www     IN       CNAME    @
mail      IN       CNAME    nitratic.com
@       IN      MX 10 mail.nitratic.com.
ftp       IN       CNAME    www.nitratic.com.
```

A little description of the above illustrated configuration zone file:

- $TTL 86399:
This sets the Time to Live (TTL) value for the zone to 86399 seconds (23 hours, 59 minutes, and 59 seconds). This value determines how long DNS resolvers and clients should cache the zone data before requesting updated information from the DNS server.

- @ IN SOA ns1.nitratic.com. root.nitratic.com. (2023070800 3600 1800 604800 86400):
This line defines the Start of Authority (SOA) record for the zone. The SOA record specifies the primary authoritative DNS server for the zone, along with various other parameters such as the serial number, refresh time, retry time, expire time, and minimum TTL.

- IN NS ns1.nitratic.com. and IN NS ns2.nitratic.com.:
These lines define the authoritative name servers for the zone. The NS record specifies the hostname of each authoritative name server for the zone.

- ns1 IN A 20.235.87.247 and ns2 IN A 4.224.20.229:
These lines define the IP addresses for the ns1 and ns2 hostnames, respectively. The A record specifies the IPv4 address for a hostname.

- @ IN A 202.166.197.69:
This line defines the IP address for the apex (or root) hostname of the zone, which in this case is nitratic.com. The @ symbol is a shorthand for the zone apex.

- www IN CNAME @:
This line defines a canonical name (CNAME) record for the www hostname, which specifies that www.nitratic.com is an alias for the zone apex nitratic.com.

- mail IN CNAME nitratic.com and @ IN MX 10 mail.nitratic.com.:
These lines define a CNAME record and a mail exchange (MX) record for the mail hostname. The CNAME record specifies that mail.nitratic.com is an alias for the zone apex nitratic.com, and the MX record specifies that mail for the nitratic.com domain should be delivered to mail.nitratic.com.

Then open the /etc/named.rfc1912zones file then add a few lines at the end:

`vi /etc/named.rfc1912zones`

```
Add the line:
zone “example.com” IN {
type master;
file “/var/named/<domain>/<domain>.forward”;

or,

file “<domain>/<domain>.forward”;
allow-update { none; };
allow-transfer { any; };
};
```

Then save it.

The used directive denotes the following:

- zone “<domain>.com” IN:
This directive specifies the domain name of the zone that is being configured, in this case and the IN keyword indicates that this is an Internet zone.

- type master:
This directive specifies that this DNS server is the master, or primary, source for the zone data and the zone data is stored in a file on the local server and is not copied from another DNS server.

- file “/var/named/<domain>/<domain>.forward”;:
This directive specifies the location of the zone data file for the “<domain>.com” zone. The zone data file contains the resource records that define the DNS information for the zone.

- allow-update { none; };:
This directive specifies which hosts or networks are allowed to submit dynamic updates for the “nitratic.com” zone. Here – none keyword indicates that no hosts are allowed to submit dynamic updates for the zone.

- allow-transfer { any; };:
This directive specifies which hosts or networks are allowed to transfer the zone data from this DNS server to another DNS server. Here – any keyword indicates that any host is allowed to transfer the zone data, which can be a security risk if the zone contains sensitive information.


In case of the recursive zone file, each record type requires a recursive zone file and follows the similar configuration steps:
create recursive zone file called <record-type-domain.reverse> >> Include on the /etc/named.rfc1912zones >> Reload the named service.

cat nitratic.reverse

```
$TTL 86400
@   IN  SOA     ns1.nitratic.com. root.ns1.nitratic.com. (          2023070800  ;Serial          3600        ;Refresh          1800        ;Retry          604800      ;Expire          86400       ;Minimum TTL  )         

; Set Name Server        
IN  NS      ns1.nitratic.com.        
IN  NS      ns2.nitratic.com.  

; Set each IP address of a hostname. Sample PTR records.  

247   IN  PTR     ns1.nitratic.com.
69    IN  PTR     www.nitratic.com.      
IN  PTR     mail.nitratic.com.
```

One need to add those recursive zone file on the /etc/named.rfc1912zones

## Configuration on the /etc/named.conf
The options section in the /etc/named.conf configuration file specifies global options and settings for the BIND DNS server.


- listen-on port 53 { 127.0.0.1; any; };:
This directive specifies the IP addresses and network interfaces on which the DNS server should listen for incoming DNS queries. Here, the DNS server is configured to listen on port 53 for queries from the loopback address 127.0.0.1 and any other IP address.

- listen-on-v6 port 53 { ::1; };:
This directive specifies the IPv6 addresses and network interfaces on which the DNS server should listen for incoming DNS queries over IPv6. Here, the DNS server is configured to listen on port 53 for queries from the IPv6 loopback address ::1.

- directory “/var/named”;:
This directive specifies the directory where BIND should store its zone data files, configuration files, and other data files. By default, BIND stores its data files in the /var/named directory.

- dump-file “/var/named/data/cache_dump.db”;:
This directive specifies the location of the file where BIND should dump its cache data in a binary format. The cache data contains recently resolved DNS queries and their corresponding responses.

- statistics-file “/var/named/data/named_stats.txt”;:
This directive specifies the location of the file where BIND should write its runtime statistics in a human-readable format.

- memstatistics-file “/var/named/data/named_mem_stats.txt”;:
This directive specifies the location of the file where BIND should write its memory usage statistics in a human-readable format.

- secroots-file “/var/named/data/named.secroots”;:
This directive specifies the location of the file where BIND should store the trusted root keys for DNSSEC validation.

- recursing-file “/var/named/data/named.recursing”;:
This directive specifies the location of the file where BIND should log information about recursive DNS queries.

- allow-query { localhost; any; };:
This directive specifies which IP addresses and networks are allowed to send DNS queries to the DNS server. In this case, the 

- allow-query directive allows queries from the localhost address 127.0.0.1 and any other IP address.
The same configuration is for the secondary authoritative DNS server, too.


## Configuration as a Secondary authoritative DNS server
Most of the configuration is quite similar to the primary authoritative DNS server with only few changes on the secondary authoritative DNS server.

The zone file will be stored on the slave directory of the /var/named/slaves/<domain-zone>

> The user and group ownership of the /var/named/slaves/<domain> directory is set to root group and user when a new directory or file is created as you’re logged in as root user. This can cause the issue as the permission denied.


This is the old screenshot of permission denied issue due to the ownership.

One need to assign named group and user as the ownership which is the default user and group account used by the BIND DNS server software.
So change the ownership of the <zone> directory to named.

`sudo  chown named:named /var/named/slaves/<domain-directory>`

The zone file is same as the primary DNS authoritative DNS server (since all the updates should reflect on the secondary DNS authoritative DNS server)

And same goes for the reverse DNS zone file:

Then certain changes on the /etc/named.rfc1912zones needs to be done:

the masters directive is used to specify the IP addresses of the master DNS servers for a particular zone in a DNS slave configuration.

The line masters { 20.235.87.247; }; specifies that the IP address 20.235.87.247 is the master DNS server for the “nitratic.com” zone. This means that the DNS server running the slave zone configuration will periodically check with the master server to update its zone data.

> After changes restart the named service to have the changes on the DNS server to come into effect.


### Verification of both authoritative DNS server
Once all the configuration has been completed, execute:

`tail -f /var/named/data/named.run` then wait for few minutes.


**Bonus**
To verify whether the service is DNS service is providing correct output or not. Use of dig command to specify the ip address of the DNS server and view the output.





