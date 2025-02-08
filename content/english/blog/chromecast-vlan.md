---
draft: false
author: "David Lakatos"
title: Streaming Plex content across VLANs to Google Chromecast
description: Case study how to implement L2 network separation between Chromecast clients and devices
publishDate: 2024-07-21
date: 2024-12-13
Image_webp: images/blog/chromecast-vlan-cover.webp
image: images/blog/chromecast-vlan-cover.png
tags:
  - firewall
  - pfSense
  - VLAN
  - 802.1q
  - streaming
  - Chromecast
  - networking
  - TCP
  - UDP
  - multicast
  - mDNS
  - DNS
  - NAT
  - TLS
  - DNS over TLS
  - DoT
  - DNSsec
---

> Sometimes you just have to do things the hard way.

This was my first impression when I stumbled upon the computer network exploratory journey which I'd like to share with you.


## Network topology

My data link layer is sectioned into several VLANs, based on data confidentiality and connected device security concerns. There is a lot of complexity in the network (IEEE 802.1q trunks, Proxmox virtualization, VLAN-aware bridging, etc). The complex and irrelevant details are abstracted away in the drawings. The relevant VLAN topology is as follows:

![Network topology](/images/blog/chromecast-vlan-overview.svg)

`VLAN100` is `HOME`, and `VLAN300` is the `IOT` data link network. pfSense provides routing and firewall functionality between the two network segments living in the different VLANs. Long story short, laptop may contact the Chromecast device via the following path and vice versa:

![Laptop data connection](/images/blog/chromecast-vlan-laptop-data-connection.svg)

There is a similar VLAN traffic path between Chromecast and Plex:

![Plex data connection](/images/blog/chromecast-vlan-plex-data-connection.svg)

## Device discovery

Chromecast spreads the word on the network about its capabilities via `mDNS`, `SSDP` or `DIAL`. We are going to use mDNS for this purpose. The protocol is restricted to work in one broadcast domain by design. If we want to see the little Chromecast icon in Chrome browser or any YouTube application, we have to make `mDNS` work across broadcast domains and hence their container VLANs.

Our solution is mDNS reflection. The concept is that mDNS queries and responses are repeated on different network segments by the router. Not the best solution from an IT security perspective, but good enough for a home network. pfSense‘s Avahi package is ready to be installed from the package repository to do the work for us.

1. `System` > `Package Manager` > `Available Packages`
1. Search term: `Avahi`
1. `Install`

Let’s configure Avahi so that mDNS messages are reflected on both networks.

Go to `Services` > `Avahi` and configure the service.

1. `Enable`: Check
1. `Interface Action`: Allow Interfaces
1. `Interfaces`: Select both `HOME` and `IOT` (CTRL+lclick)
1. `Enable reflection`: Check
1. `Save`
1. `Restart Service` in the top-right corner

Reflection is now enabled, that is, when a cast-ready client device searches for Chromecast destinations on the `HOME` network, the mDNS query is reflected into the `IOT` network and the response from the IOT network is also reflected into the `HOME` network. Here’s how an mDNS query and its response looks like in Wireshark:

```yaml
Frame 794: 82 bytes on wire (656 bits), 82 bytes captured (656 bits) on interface wlp0s20f3, id 0
Ethernet II, Src: laptop.home.internal.example.com (70:d8:**:**:**:97), Dst: IPv4mcast_fb (01:00:5e:00:00:fb)
Internet Protocol Version 4, Src: laptop.home.internal.example.com (10.0.1.***), Dst: 224.0.0.251 (224.0.0.251)
User Datagram Protocol, Src Port: 5353, Dst Port: 5353
Multicast Domain Name System (query)
  Transaction ID: 0x0000
  Flags: 0x0000 Standard query
  Questions: 1
  Answer RRs: 0
  Authority RRs: 0
  Additional RRs: 0
  Queries
    _googlecast._tcp.local: type PTR, class IN, "QM" question
```

```yaml
Frame 3863: 408 bytes on wire (3264 bits), 408 bytes captured (3264 bits) on interface ***, id 0
Ethernet II, Src: pfsense.home.internal.example.com (fa:ed:**:**:**:b4), Dst: IPv4mcast_fb (01:00:**:**:**:fb)
Internet Protocol Version 4, Src: pfsense.home.internal.example.com (10.0.1.***), Dst: mdns.mcast.net (224.0.0.251)
User Datagram Protocol, Src Port: 5353, Dst Port: 5353
Multicast Domain Name System (response)
  Transaction ID: 0x0000
  Flags: 0x8400 Standard query response, No error
  Questions: 0
  Answer RRs: 4
  Authority RRs: 0
  Additional RRs: 0
  Answers
    _googlecast._tcp.local: type PTR, class IN, Chromecast-38107d9ff4b3d0e65c7cffda********._googlecast._tcp.local
    38107d9f-f4b3-d0e6-5c7c-ffda********.local: type A, class IN, cache flush, addr 10.0.3.***
    Chromecast-38107d9ff4b3d0e65c7cffda********._googlecast._tcp.local: type SRV, class IN, cache flush, priority 0, weight 0, port 8009, target 38107d9f-f4b3-d0e6-5c7c-ffda********.local
    Chromecast-38107d9ff4b3d0e65c7cffda********._googlecast._tcp.local: type TXT, class IN, cache flush
```

We asked on the `HOME` network whether someone provided the `_googlecast._tcp.local` service, and where that provider may be contacted. Someone answered our question in the `IOT` network and the Avahi daemon running on pfSense reflected it into the `HOME` network. The answer contains the information that there is some device that provides the `_googlecast._tcp.local` service. The device is a Chromecast device with the identifier `Chromecast-38107d9ff4b3d0e65c7cffda********` which has the IP address of `10.0.3.***`. Cool, we have a working discovery in place across the `HOME` and `IOT` VLANs.


## Firewall rules

There is a lot of unofficial data on how casting works on the network. To be honest, I still do not get it 100%, but here’s what I observed and what works for me. We should set up reserved DHCP addresses for the Chromecast devices and create a pfSense alias based on their addresses.

Now let’s create a list of our Chromecast devices so we can refer to them later as one entity on pfSense:

`Firewall` > `Aliases` > `IP`

```yaml
Name: Chromecasts
Type: Host(s)
IP or FQDN:
  - <IP address or FQDN>
  - <IP address or FQDN>
  - ...
```

Let’s create the firewall rules between the networks:

```yaml
Action: PASS
Interface: HOME
Address Family: IPv4+IPv6
Protocol: TCP
Source: `HOME` subnets
Destination: Address or Alias  Chromecasts
Destination Port Range:
  From: 8008
  To: 8009
Description: Chromecast universal control
```

```yaml
Action: PASS
Interface: HOME
Address Family: IPv4+IPv6
Protocol: UDP
Source: `HOME` subnets
Destination: Address or Alias  Chromecasts
Destination Port Range:
  From: 32768
  To: 61000
Description: Chromecast universal streaming
```

```yaml
Action: PASS
Interface: HOME
Address Family: IPv4+IPv6
Protocol: TCP
Source: `HOME` subnets
Destination: Address or Alias  Chromecasts
Destination Port Range:
  From: 8443
  To: 8443
Description: Chromecast screen mirroring
```

And here’s the tricky part. Chromecast will receive the command over `8009/tcp` that it must play something from your Plex server. Chromecast will open a connection from the `IOT` network to our Plex server’s `32400/tcp` socket. Since the Plex server is located on the `HOME` network, we need a firewall rule to allow the connection.

If we have IPv6 in our network, we cannot filter for source address unfortunately, since Chromecast seems to use temporary IPv6 addresses. Be aware that I have already set up a PlexServer host alias due to my complex IPv4 and IPv6 dual stack addressing. It’s not necessary, you can do it differently if you want to, it’s up to you.

```yaml
Action: PASS
Interface: IOT
Address Family: IPv4+IPv6
Protocol: TCP
Source: IOT subnets
Destination: Address or Alias  PlexServer
Destination Port Range:
  From: 32400
  To: 32400
Description: Chromecast screen mirroring
```


## DNS

When you open a casting session, your browser tells Chromecast where your Plex server is located. Since there is proper forward and reverse DNS addressing configured on the network, the browser sends the following information to Chromecast: `tcp://plex.home.internal.example.com:32400`

Chromecast will try to resolve this internal address, and will fail to do so. This is because Chromecast does not use the network settings configured by DHCP or IPv6 Router Advertisement messages. It uses hard-coded Google DNS servers `8.8.8.8` and `8.8.4.4`. This is the kind of system design BS I get frustrated about. It would put an abrupt end to this journey if Chromecast enforced DNS over TLS or DNSsec for its resolvers. Luckily, it does not, so let's continue fixing Google's system design failure.

We need to spoof Google’s DNS servers by pfSense on the `IOT` network in order to let Chromecast resolve internal addresses. A kind of hacky, but working solution is applying an sNAT rule for the outgoing traffic. Whenever there is a packet addressed to `tcp://8.8.8.8:53` or `udp://8.8.8.8:53` or `tcp://8.8.4.4:53` or `udp://8.8.4.4:53` sockets, we will change the destination address to `10.0.3.***`, for our internal caching recursive DNS resolver that's able to resolve `plex.home.internal.example.com`.

First, let’s create an Alias for Google’s DNS servers.

`Firewall` > `Aliases` > `IP` > `Add`

```yaml
Name: GoogleDNS
Type: Host(s)
IP or FQDN:
  - 8.8.8.8
  - 8.8.4.4
```

Add a new NAT rule to the position of your preferences.

`Firewall` > `NAT` > `Port Forward` > `Add`

```yaml
Interface: IOT
Address Family: IPv4
Protocol: TCP/UDP
Destination: Address or Alias  GoogleDNS
Destination port range:
  From port: DNS
  To port: DNS
Redirect target IP:
  Type: IOT address
Redirect target port:
  Port: DNS
Description: Redirect Chromecast DNS traffic
```

Whenever a Chromecast – or any other device on the `IOT` network – tries to resolve a name via Google's DNS servers, those names are actually resolved by pfSense's `unbound`.


## Sorry! Something went wrong

My Plex setup is using an X.509 certificate signed by my own CA. Also, I wanted to help the Plex clients find the server whenever they share a common network with the server.

`Settings` > `Network` > `Custom server access URLs`

This configuration entry publishes URLs to `plex.tv`. These URLs intend to help clients find the server if server discovery fails, or it is not possible due to some kind of custom network setup. Since Plex has a TLS certificate configured, my URL was `https://plex.home.internal.example.com:32400`.

I had to find out the hard way that Chromecast receives this exact URL, which means Chromecast will initiate a TLS handshake with the Plex server whenever we want to stream something. The only problem is that Plex’s certificate is signed internally, which certificate chain is not trusted by Chromecast. pfSense packet capture shows that TLS handshakes are interrupted by a TLS Alert emitted by Chromecast due to the certificate chain trust issue.

Remove the Custom server access URLs, or provide a non-HTTPS URL, and the issue disappears.


## Plex webapp vs Plex webapp

Casting is meant to work only when using https://app.plex.tv. Do not use your internal URL (eg. `https://plex.home.internal.example.com:32400`) when you intend to cast media.


## Conclusion

Google’s Chromecast user documentation has much room to improve. The inter VLAN Chromecast setup process required a lot of experimenting so far. I hope my reverse engineering experience helps a lot of you to avoid pitfalls.
