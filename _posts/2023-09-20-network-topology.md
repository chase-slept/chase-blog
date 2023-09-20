---
title: Network Topology
date: 2023-09-20 11:55 -0500
category: homelab
tags: [networking, homelab]
---
Hello again! Today I would like to outline my network topology. This should help give a bit more context to some upcoming posts. While I don't have a particularly complicated network, a few of my projects have required a bit more network complexity than a typical home network. This post won't be a tutorial, just a bit of background info to lead into some other topics.

## Network Diagram

To start with, here's a diagram of my network (its current state) that shows the device layout and how everything is connected:
![Diagram of Network](/assets/img/network-diagram.jpg)

## The Beginning

Some months ago, I happened upon a Ubiquiti Edgerouter X and access point. They're older devices now, but I wanted to play around with them and try out the Unifi ecosystem. What a good opportunity to expand my smart home and IoT network, and maybe learn some networking skills along the way!

![image of Edgerouter X and Unifi AP](/assets/img/erx-ap.jpg)

My router at the time was a Netgear Nighthawk and my home network was basically a WiFi signal and long ethernet cable to my desktop computer in the basement. Only a handful of other ethernet devices lived nearby---a Philips Hue hub and an Ikea Tradfri hub---and shared a bookshelf in the mainfloor living room. Not ideal, and the Netgear router was bit of an eyesore, even up on a high shelf.

![image of livingroom router on shelf](/assets/img/shelf-router.jpg)
*At some point I mounted the equipment to the wall. Not sure if it helped though ...*

With the Edgerouter X in-hand, a lot of ideas came to mind, but the first step was to move everything out of the living room. Things were going to get messy and ugly, and I had a newly renovated office with an empty closet---perfect for a makeshift network area. I started by finding a spot in the basement near the closet that was convenient, drilled a hole though to the outside of the house, then ran one end of the new coax from the closet and out through the newly drilled hole. Once outside, I routed the new cable to the telecom box where the ISP line comes into the house, about 50ft in my case.

![image of telecom box](/assets/img/telecom-box.jpg)
*This is how everything looked in the telecom box before I swapped the old and new cable.*

Running a new line into the basement allowed me to move all of the network stuff into the office, out of sight but also easily accessible to add, remove, and configure as need be. This also added a lot more value to the office, which I had been using as a WFH office. Now it was basically a home-lab---or would be, once I had a more useful network and some more equipment! **More on that in a bit.**

## Messy Closet

With a suitable space prepared, I got to work on creating a mountable surface for the network equipment I already owned. A piece of scrap OSB---cut to size and screwed into studs---worked well and made for a cheap and easy place to wall-mount devices securely.

![image of OSB](/assets/img/osb-panel.jpg)
*Perfection.*

At this point I had started to experiment with the Edgerouter in earnest and ordered more network equipment as part of that learning journey. A 4-port switch here. A Raspberry Pi there. A NAS to store things. A spare laptop as a server. A friend 3D printed some brackets for particularly clunky devices and those that lacked their own mounting points, and another device was fastened with velcro. Eventually, everything was (poorly) mounted to the wall. During this time I also ran new ethernet lines to the main floor, the office near my workstation, and a reserve into a hallway for later expansion. Everything was messy, as expected!

![image of semi-final closet](/assets/img/network-closet.jpg)
*Also off-camera, on the wire rack: Asustor NAS & Ubuntu Laptop*

Those were some of the highlights during the initial setup of this homelab. Over the course of a few months I had learned quite a lot and was ready to learn more. In the network diagram posted above you can see the various devices I mentioned above and how they all connect together. You might even be able to make sense out of the tangle of wires and blinking lights in my network closet. But there's a crucial bit I haven't talked about which the diagram doesn't explain---the VPS and the Wireguard Tunnel that connects it to my local network.

## Going Remote

All of the self-hosted services so far had served just my local network: Jellyfin to serve video to TVs and phones in my home; PiHole, Home Assistant, Portainer and other services to intereact with my local devices and infrastructure; a handful of servers that I could access via SSH from my office or from a computer on my network. This was awesome, but I wanted to be able to access all of this from basically anywhere. Ideas (and the motivation to act on them) can happen anywhere, anytime---I wanted to be able to use my office resources anywhere, anytime, too!

In the next post, I'll talk more about how I tackled the 'remote access' problem and a few other challenges encountered along the way.
