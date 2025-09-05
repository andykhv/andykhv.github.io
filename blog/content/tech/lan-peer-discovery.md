---
title: "Learning LAN Peer Discovery with Go"
date: 2025-09-01T12:00:00-07:00
draft: true
---

# Introduction
A few months ago, I started my journey of self-hosting and building a homelab.
Currently it's just a Raspberry Pi that serves photos via [JellyFin]().
During the holidays, there is usually somebody in the family wanting to view old family photos.
These photos are unfortunately not in any _cloud_, but stored in one of my portable harddrives.
So, instead of paying more for cloud storage, I used that money torwards a Raspberry Pi.
In hindsight, there's better alternative machines for self-hosting (especially for the price).
So, how does my home lab relate to the headline of this post?
Well, in the world homelabs, there is the common problem of: "how do I securely connect to my apps outside of my personal LAN?" I found [Tailscale]() as a rather "out-of-the-box" solution for creating secure tunnels to my devices at home.
And it comes with SSH! 

But what was happening under the hood of Tailscale?
I was quite interested about the implementations of Tailscale and wanted to understand it a bit more.
So, like many other curious individuals out there, I queried ChatGPT hehe. How are nodes discovered in a Tailnet?
LAN Peer Discovery was a topic that was mentioned by ChatGPT and peaked my interest.
In the vain of "learn by doing", I implemented a LAN Peer Discovery system in Go.

# Background
LAN Peer Discovery solves the problem of "Who else is on my LAN?".
In my case, who else is on my home network?
How does my personal computer know there are other devices on the network?
Devices can broadcast their subnet IPs and setup direct connections with each other.
These connections exist within the LAN, which eliminates the need for any connections out to an external server.
In the context of Tailscale: are there devices on my tailnet that exist on the same LAN?
If so, these devices can connect directly without having to route through the internet!

# Architecture
Before delving deeper into network topics and the implementation with Go, it's important to understand what I actually built!

# Broadcasting
So how do devices discover other devices that are on the same LAN? Similarly in life, devices need to *speak up* to make themselves known to others!
Technically, this occurs by announcing their subnet IP on a dedicated broadcast address of the chosen network interface.
On MacOS, the wifi runs on interface `en0`.
There are other interfaces that exist on MacOS on such as `lo0`, which is the loopback interface used for local traffic to `127.0.0.1` or `::1`.
*(side note: to list all interfaces on your macbook, run `ifconfig` in your terminal)*
By looking through the `en0` interface via `ifconfig`, the IPv4 address can be found for my device, as well as the broadcast address.
The IPv4 broadcast address is always the highest value of the subnet address space.

Programmatically, the IPv4 broadcast address can always be found by flipping all subnet host bits to 1:

```go
//GOAL: Find broadcast address with ipnet = 192.168.4.99/22
//1. get the prefix length, ones convey the number of leading 1 bits in the subnet mask (i.e. the network portion)
ones, _ := ipnet.Mask.Size()
prefix := netip.PrefixFrom(netIpAddr, ones)
ones = prefix.Bits()

//2. create the subnet mask (leading 1s and remaining 0s, 0s convey the host address portion)
mask := uint32(0xffffffff << (32 - ones))

//3. get IPv4 address and convert to a big-endian uint32
ip4 := prefix.Addr().As4()
ipInt := binary.BigEndian.Uint32(ip4[:])

//4. turn on all host bits
broadcastInt := ipInt | ^mask
```

# Announcing
After finding the broadcast address for `en0`, each node will execute a goroutine that contains an announce loop to periodically send announcements to the broadcast address.
The purpose of an announcement is to say: *"Hello! I exist, I live at this IP address!*. Remember, this helps with discovering other devices on the same LAN. I'm pretty sure products like Chromecast utilize LAN Peer Discovery to be able to find other eligible Chromecast devices on the network (don't quote me on this!).

Here in the code, I took advantage of **channels**, **tickers**, and the parent **Context** to create the announce loop. Very refreshing implementation compared to what I do for work (C#, OOP).
```go
//some code was retracted for brevity
func announceLoop(ctx context.Context, interfaces []netx.InterfaceInfo, privateKey ed25519.PrivateKey) {
	t := time.NewTicker(AnnounceInterval)
	defer t.Stop()
	announce := wire.Announce{retracted...}

	for {
		select {
		case <-ctx.Done():
			return
		case <-t.C:
			//1. for each eligible interface:
			for _, iface := range interfaces {
				//2. setup the announce body
				a := announce
				a.Addr = iface.IP
				a.EpochMS = time.Now().UnixMilli()

                ...

				//3. Sign the announcement and store signature in body (ed25519)
				a.Sign(privateKey)
				packet, _ := wire.Encode(a)

				//4. Bind UDP socket listen at device's IP, remote address is broadcast address
				remoteAddress := &net.UDPAddr{IP: iface.Broadcast.AsSlice(), Port: AnnouncePort}
				listenAddress := &net.UDPAddr{IP: iface.IP.AsSlice(), Port: 0}
				conn, err := net.ListenUDP("udp4", listenAddress)

                ...

				//5. Send announcement packet to broadcast IP
				_, _ = conn.WriteToUDP(packet, remoteAddress)
				conn.Close()
			}
		}
	}
}
```


# Go Highlights

# Demo

# Networking is fun!

# Next Steps

# Conclusion