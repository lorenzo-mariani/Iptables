# Logging network layer headers with iptables

### Logging the IP header and options

The IP header is defined by RFC 791, which describes the structure of the header used by IP. The IP header is composed of several fields:

* **Type of Service (TOS, PREC), Total Length (LEN), Identification (ID), Flags (DF, MF), Fragment Offset (FRAG), Time-to-Live (TTL), Protocol (PROTO), Source Address (SRC), Destination Address (DST)** - fields that are always logged by iptables
* **Version, IHL, Header Checksum, Padding** - fields that are not logged by iptables under any circumstances
* **Options (OPT)** - iptables only logs IP options if the *--log-ip-options* command-line argument is used when a LOG rule is added to the iptables policy

IP options have a variable length. Without IP options, an IP packet header is always exactly 20 bytes long.

The string OPT is usually followed by a long sequence of hexadecimal bytes. These bytes are the complete IP options included in the IP header, but they are not decoded by the iptables LOG target; *psad* is used to make sense of them.

### Logging ICMP

ICMP (defined by RFC 792) has a simple header that is only 32 bits wide. This header consists of 3 fields: type (8 bits), code (8 bits), and a checksum (16 bits); the remaining fields are part of the data portion of an ICMP packet.

* **Type (TYPE), Code (CODE)** - fields that are always logged by iptables
* **Checksum** - field that is not logged by iptables under any circumstances
* **Data** - there are no command-line arguments in iptables to influence how the LOG target represents fields within the data portion of ICMP packets

# Network layer attack definitions

A network layer attack is a packet or series of packets that abuses the fields of the network layer header in order to exploit a vulnerability in the network stack implementation of an end host, consume network layer resources, or conceal the delivery of exploits against higher layers. Network attacks fall into one of 3 categories:

* **Header abuses:** packets that contain maliciously constructed, broken, or falsified network layer headers.
* **Network stack exploits:** packets that contain specially constructed components designed to exploit a vulnerability in the network stack implementation of an end host.
* **Bandwidth saturation:** packets that are designed to saturate all available bandwidth on a targeted network.

# Abusing the network layer

### Nmap ICMP ping

When Nmap is used to scan systems that are not on the same subnet, host discovery is performed by sending an ICMP Echo Request and a TCP ACK to port 80 on the targeted hosts. ICMP Echo Requests generated by Nmap differ from the Echo Requests generated by the ping program in that Nmap Echo Requests do not include any data beyond the ICMP header. Therefore, if such a packet is logged by iptables, the IP length field should be 28 (20 bytes for the IP header without options, plus 8 bytes for the ICMP header, plus 0 bytes for data).

NOTE: the ping program can also generate packets without application layer data by using the *-s 0* command-line argument to set a zero size on the payload, but by default the ping program includes a few tens of bytes of payload data.

### IP spoofing

IP spoofing means to construct an IP packet with a falsified source address. You can configure routers and firewalls to not forward packets with source addresses outside of internal network ranges (so spoofed packets would never make it out).

Example of anti-spoofing rules:

	iptables -A FORWARD -i eth1 -s ! $INT_NET -j LOG --log-prefix "SPOOFED PKT"
	iptables -A FORWARD -i eth1 -s ! $INT_NET -j DROP

Note that IP spoofing is different from NAT (Network Address Translation) operations. NAT is a networking function provided by firewalls to protect internal network behind a single external address.

### IP fragmentation

The process of splitting IP packets, known as *fragmentation*, is necessary whenever an IP packet is routed to a network where the data link MTU size is too small to accommodate the packet. It is the responsibility of any router that connects two data link layers with different MTU sizes to ensure that IP packets transmitted from one data link layer to another never exceed the MTU. The IP stack of the destination host reassembles the IP fragments in order to create the original packet. IP fragmentation can be used by an attacker as an IDS evasion mechanism by constructing an attack and splitting it over multiple IP fragments. Any fully implemented IP stack can reassemble fragmented traffic, but in order to detect the attack, an IDS also has to reassemble the traffic with the same algorithm used by the targeted IP stack (this is difficult to implement).

### Low TTL values

Any IP router is supposed to decrement the TTL value in the IP header by one every time an IP packet is forwarded to another system. If packets appear within your local subnet with a TTL value of one, then someone is most likely using the traceroute program against an IP address that either exists in the local subnet or is in a subnet that is routed through the local subnet. Usually this is simply someone troubleshooting a network connectivity problem, but it can also be an instance of someone performing reconnaissance against your network in order to map out hops to a potential target.

### The Smurf attack

The Smurf attack is a technique whereby an attacker spoofs ICMP Echo Requests to a network broadcast address. The spoofed address is the intended target, and the goal is to flood the target with as many ICMP Echo Response packets as possible from systems that respond to the Echo Requests over the broadcast address. If the network is functioning without controls in place against these ICMP Echo Requests to broadcast addresses, then all hosts that receive the Echo Requests will respond to the spoofed source address. By using the broadcast address of a large network, the attacker hopes to magnify the number of packets that are generated against the target.

### DDoS attack

A DDoS attack at the network layer utilizes many systems (potentially thousands) to simultaneously flood packets at target IP addresses. The goal of such an attack is to chew up as much bandwidth on the target network as possible with garbage data in order to edge out legitimate communications. DDoS attacks are among the more difficult network layer attacks to combat because so many systems are connected via broadband to the Internet.

# Network layer responses

You can manipulate the network layer headers in 3 ways:

* A filtering operation conducted by a device such as a firewall or router to block the source IP address of an attacker
* Reconfiguration of a routing protocol to deny the ability of an attacker to route packets to an intended target by means of *route blackholing* — packets are sent into the void and are never heard from again
* Applying thresholding logic to the amount of traffic that is allowed to pass through a firewall or router based on utilized bandwidth

### Network layer filtering response

After an attack is detected from a particular IP address, you can use the following iptables rules as a network layer response that falls into the filtering category. These rules are added to the INPUT, OUTPUT, and FORWARD chains; they block all communications (regardless of protocol or ports) to or from the IP address 144.202.X.X:

	iptables -I INPUT 1 -s 144.202.X.X -j DROP
	iptables -I OUTPUT 1 -d 144.202.X.X -j DROP
	iptables -I FORWARD 1 -s 144.202.X.X -j DROP
	iptables -I FORWARD 1 -d 144.202.X.X -j DROP

### Network layer thresholding response

Applying thresholding logic to iptables targets is accomplished with the iptables *limit* extension. The following iptables rules restrict the policy to only accept 10 packets per second to or from the 144.202.X.X IP address:

	iptables -I INPUT 1 -m limit --limit 10/sec -s 144.202.X.X -j ACCEPT
	iptables -I INPUT 2 -s 144.202.X.X -j DROP
	iptables -I OUTPUT 1 -m limit --limit 10/sec -d 144.202.X.X -j ACCEPT
	iptables -I OUTPUT 2 -d 144.202.X.X -j DROP
	iptables -I FORWARD 1 -m limit --limit 10/sec -s 144.202.X.X -j ACCEPT
	iptables -I FORWARD 2 -s 144.202.X.X -j DROP
	iptables -I FORWARD 1 -m limit --limit 10/sec -d 144.202.X.X -j ACCEPT
	iptables -I FORWARD 2 -d 144.202.X.X -j DROP

For each ACCEPT rule above that uses the *limit* match, there is also a corresponding DROP rule. This accounts for packets levels that exceed the 10-per-second maximum permitted by the *limit* match; once the packet levels are higher than this threshold, they no longer match on the ACCEPT rule and are then compared against the remaining rules in the iptables policy. You can also use the limit match to place thresholds on the number of iptables log messages that are generated by default logging rules.
