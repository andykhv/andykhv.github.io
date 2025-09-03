---
title: "Learning LAN Peer Discovery with Go (while also learning Go)"
date: 2025-09-01T12:00:00-07:00
draft: true
---

# Introduction
A few months ago, I started my journey of self-hosting and building a homelab. Currently it's just a Raspberry Pi that serves photos via [JellyFin]().
During the holidays, there is usually somebody in the family wanting to view old family photos. These photos are unfortunately not in any _cloud_, but stored in one of my portable harddrives. So, instead of paying more for cloud storage, I used that money torwards a Raspberry Pi. In hindsight, there's better alternative machines for self-hosting (especially for the price). So, how does my home lab relate to the headline of this post? Well, in the world homelabs, there is the common problem of: "how do I securely connect to my apps outside of my personal LAN?" I found [Tailscale]() as a rather "out-of-the-box" solution for creating secure tunnels to my devices at home. And it comes with SSH! As some of y'all may know, Tailscale utilizes [Wireguard]() as the underlying communication protocol for connecting devices together.

But what was happening under the hood of Tailscale/Wireguard? I was quite interested about the implementations of Tailscale and wanted to understand it a bit more. So, just like other curious individuals out there, I queried ChatGPT. How are nodes discovered in a Tailnet? LAN Peer Discovery was a topic that was mentioned by ChatGPT and peaked my interest. In the vain of "learn by doing", I implemented a LAN Peer Discovery system in Go. Well, I also wanted to learn Go as well :).

# Background

# Designing the System

# Implementation Highlights

# Running it Locally

# Lessons Learned

# Next Steps

# Conclusion