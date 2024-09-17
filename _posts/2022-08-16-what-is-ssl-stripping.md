---
title: what is SSL stripping method?
author: Nishal Rai
date: 2022-08-16 20:55:00 +0800
categories: [F5 Networks, Visaulization]
tags: [getting started]
---

# ssl stripping, is it a potential threat? How does it work and what are the precaution to avoid such an attack?
SSL Stripping or SSL Downgrade Attack as the name implies is an attack that uses the SSL Strip tool or other related techniques to strip away the protection provided by the SSL/TLS protocol and HTTPS. SSL Stripping can be considered as a form of Man-in-the-middle attack (MitM) that takes over the advantage of the TLS protocol and the way it begins the connection. SSL stripping can be used for eavesdropping and data manipulation to the victim

## History of SSL & its limitations
SSL (Secure Socket Layer) protocol was initially developed for a secure connection i.e. www over the internet. SSL made connections via a secure channel, port 443 and layered encryption over the connection at the application level. This was known as HTTP over SSL or HTTPS.
However, SSL was found vulnerable and the privacy was compromised. Then SSL 3.0 was replaced with transport layer security (TLS) 1.0 in 1999. One of the major differences between the SSL and TLS was a hello via an insecure channel before being redirected to a secure one. Since both use port 443 and encrypt HTTP connection but, the security and encryption of those connection differed.

## What MitM does in ssl stripping?

Since when the user connects to a website or any network on the internet then, the connection has to be routed through dozens of other points on its way to the destination. So, if the user is not using encryption or HTTP, all the data transmitted over that connection passes through each one of those points in plaintext.
Here, Man-in-the-Middle (MitM) can intersect connection at any point between the user and the destinated server. That means the MitM (attacker) can be eavesdropping at one of those points and also can effectively intercept, read and even manipulate everything being communicated between them.

## What really is ssl stripping & How does it work?
As we mentioned earlier that TLS begins with an insecure hello before it’s redirected to a secure channel. If a Man-in-the-Middle attacker can redirect the client to an HTTP version of the website, it can steal information and manipulate the connection.

To understand in a simple language:

When the user requests for a vault to transport their message in a secure way to the server. Then, the server confirms the identity of the requested user through certain procedures and provides an unlocked vault only to each and every certified user who had requested.

Since the vault was initially unlocked to the user and they put their message inside the vault and close the door then spin the knob to send back to the server. The user doesn’t know the combination of the vault to unlock it and has no information related to the combination of the locked vault. As a whole, this process is considered as a secure communication over the internet.

But, when the MitM involves in such process as intercepter. MitM (attacker) receives the request from the user which was originally sent to the server. The MitM (attacker) receives the request of the user (victim), then it requests a need of a vault to the server which was actually asked by the user (victim). After that, the MitM responds to the user, an unlocked vault of the attacker not the requested server to put their message. Since the user is unaware of the fact and puts their messages inside the attacker vault then closes the door and spin the knob as mentioned earlier. When the user sends back to the server, the MitM intercepts in between and unlocks the vault and collects the messages then, sends back the messages to the server unlocked vault. In this way, all the messages of the user (victim) is gained by the MitM (attacker) by performing the SSL stripping method.

But, before we dive inside the technical part of the ssl stripping technique. We need to understand the working of SSL / TLS handshake then only things starts to make sense.Since the initial hello and HTTP redirection request is unencrypted, a MITM can alter it and send the user to an insecure version of the page. In order to execute the ssl stripping technique, it just need three entity – a victim (user), a secure communication & Mitm (who behaves as proxy server).
So, here are all the things behind the ssl stripping technique.

Consider 

Tom (victim) wants to establish a secure connection to 

the server via HTTPS-enabled website. But, 

John (attacker) wants to intercept the communication and see all the credentials of Tom. In order to accomplish this task, John establish a connection with the Tom where Tom will disconnect from the secure server.When Tom requests to visit the destinated secure site on his browser, John gets the request and respond it to the destinated server as Tom. However, all the communication taking place between John and the website is SSL protected.
Then the server responds to John which was actually the request of Tom in HTTPS-enabled URL form and John will use his precarious and coding skills to downgrade this secure HTTPS URL to insecure HTTP URL which is forward to the Tom. Here, Tom has no clue about the fact of actual connection with the server i.e. insecure HTTP URL website
Since the connection is over stripped HTTP protocol by John. This will expose all the credentials such as password, banking details, credit card details, and everything, Tom sends through an insecure HTTP URL. Due to its nature of attack where the connection is downgraded from HTTPS to HTTP, SSL stripping attack is also called HTTP downgrade attack.The SSL stripping techniques can be performed mainly in three ways as mentioned below:

- Using proxy server
- ARP spoofing
- Using hotspot

How should I protect myself from ssl stripping?
Since, every attack has its loop hole and certain limitation which can be advantage to the user. There are certain measures to tackle from such attacks:

- Use only the latest TLS version protocol and enable HTTPS on pages of your website.
- Add the visited website to HSTS Preload List – A strict policy where the browser will deny the website which has HTTP enabled protocol in it.
- Do not connect in unknown free wifi or hotspot network especially in public places. 