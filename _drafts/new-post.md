---
title: Accessing local resources remotely
# date:
category: 
tags: []
---
In [my last post]({% post_url 2023-09-20-network-topology %}) I briefly discussed the network topology of my homelab and a bit about its creation. Today I'd like to expand on that and explore some of the steps I took to access my local network resources over the internet. This was one of my first big self-taught hurdles, and required quite a bit of research and tinkering to figure out. As with the last post, this isn't so much a guide as post-mortem documentation---please keep that in mind.

![Diagram showing Network Topology](/assets/img/network-diagram.jpg)
*Network diagram from [my previous post]({% post_url 2023-09-20-network-topology %}).*

## Recap and Next Steps

Some details about my network and objectives:

- I have a pretty standard home network (diagram above): ISP modem, a couple of consumer-grade routers, a small switch, a wireless access point, and some other network gadgets. Additionally: a handful of IoT devices such as HomeKit sensors, cameras, lights, their associated hubs, etc. I haven't implemented any VLANs or complicated routing rules---my network probably looks like most other simple home networks.
- I wanted to access these devices, their admin dashboards and any self-hosted services through a public domain name that I owned.
- I wanted to serve some static content (this website!) from a VPS using the same domain name.
- I wanted the ability to access everything remotely and securely.

Here are the steps I planned to take (more info on this in a bit):

- Configure the **domain name** to point to the VPS via its **static IP address**
- Set up Cloudflare and Wireguard **tunnels** between the VPS and Home
- Configure a **reverse proxy** to direct requests to the appropriate server or service
- Set up SSH access on each device

## Challenges and Purpose

**Why a domain name and static IP? Why tunnels? Why a reverse proxy?** Together, these components allowed me to bypass various ISP restrictions and limitations. My ISP uses [CGNAT](https://en.wikipedia.org/wiki/Carrier-grade_NAT) to distribute shared IP addresses, which made routing internet traffic to my local network [nearly impossible](https://www.purevpn.com/blog/cgnat-port-forwarding). They also aren't super fond of hosting and block most ports within their network, so self-hosting services or even a website was a challenge---I could configure network settings locally but couldn't open ports on the ISP side to complete the route. With these considerations in mind, let's go over the individual components listed above.

Since I planned to access many different services remotely, a **domain name** was essential; it would translate an IP address to something that was easier to remember and public-facing (blog.example.com). Pointing a domain name to a **static IP** and opening the right network ports is often enough to serve content from a local network to the internet. In my case, opening ports wasn't an option and the ISP assigned a shared IP address rather than a static IP. Using a VPS with its own static IP would allow me to bypass this limitation and would also allow us to open the correct ports on the server---this would be the site of our first endpoint. The domain name and VPS enabled a way around the ISP, but I still had two problems to solve: how to **securely route traffic** from this endpoint to another on my local network, and how to direct traffic to **multiple** services on that network.

While researching secure connection protocols, one solution I came across involved using a VPN tunnel to encrypt traffic between endpoints. There are many third-party VPN Tunnel solutions such as Tailscale, ZeroTier, Cloudflare Tunnels and others, as well as self-hosted solutions---I opted for a combination of a [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) and a self-hosted [Wireguard](https://www.wireguard.com) tunnel. My initial plan was to use only Cloudflare Tunnels, but their TOS prohibits serving non-HTML traffic and I wanted to serve video content (such as from IP cameras or a media server); making the tunnel myself with Wireguard meant no TOS prohibitions on traffic. I also wanted to SSH into my servers and Cloudflare has a basic-but-decent access control feature that can be applied to their Tunnels. Since I was already using Cloudflare to manage my domain name, it made sense to use these features together to add a form of 2FA/MFA to my SSH connections. In theory, setting up these two tunnels (one for secured SSH connections, one to serve content) would create the endpoints needed to route requests to my local network. With that figured out, just one problem remained: routing to **multiple** services hosted on my local network.

This last problem was pretty simple, or at least seemed to be. I'd come across the concept of **reverse proxies** while working on a different project and knew I'd need one here, but I had never used one before. The goal was to have the VPS serve a webpage (this blog) via the domain name and forward any subdomains to their related services via reverse proxy. I'll talk more about the reverse proxy configuration later, but for now let's look at related problem I encountered while initially setting up our tunnel endpoints.

![Diagram of WG with endpoints detailed?]()

When I was first experimenting with Cloudflare Tunnels, I used a Raspberry Pi running a containerized instance of *cloudflared* on my network to serve as the local tunnel endpoint (via its Docker network). I tried to set up the Wireguard endpoint the same way---by running Wireguard in a Docker container, exposing the Docker network, then pointing the VPS/Reverse Proxy to this endpoint. This allowed everything on that Docker network to resolve through the Domain Name-->Reverse Proxy (VPS)-->Raspberry Pi route, but none of the other services on my network would resolve. Between me being *very novice* at networking and *pretty novice* at Linux, I couldn't wrap my head around troubleshooting and fixing this easily using routing tables or more advanced network rules. To work around my network configuration skill gap, I opted instead to move the Wireguard endpoint from the Raspberry Pi to the Edgerouter X being used as my primary router. With the endpoint on the router, I wouldn't need to manually set any routes from within the Docker Network, which greatly simplified the rest of the configuration. **However**, this was only possible via a [Wireguard installation package targeting Ubiquiti devices](https://github.com/WireGuard/wireguard-vyatta-ubnt), including the Edgerouter X---your specific router/device may or may not be supported and some addition *creative problem-solving* may be required to find a suitable endpoint location.

With a new endpoint in place for our Wireguard tunnel, I decided to set up a second reverse proxy within my local network to complete the route and direct traffic to my local servers and devices. This may not be the 'best' or 'right' solution, but it was very easy to set up and would be simple to update when I needed to add more services in the future. The first reverse proxy (VPS) would generate valid SSL certificates and send the encrypted traffic from a specific subdomain through the Wireguard tunnel interface, directly to my local router (with a port number attached); the router would forward traffic to the second reverse proxy (Raspberry Pi), which would then forward traffic to the local IP and matching port (10.0.0.40:8080) for that service. Success!

With all of these problems brainstormed and proper solutions planned out, it was time to configure everything. In my case, a lot of brainstorming and configuration happened incrementally and out of order on a problem-by-problem basis. In this next section I'll review some of the key setup steps and some example config files for the domain name, VPS, reverse proxies, Wireguard and Cloudflare tunnels, and other components. Once again, this is more of a post-mortem than a tutorial, but maybe it will help someone who may be puzzling through similar problems. I'll provide links to documentation and other resources, for anyone working on their own implementations or just curious to learn more.

## Execute the Plan

Some assumptions before we continue: devices are installed and configured, the local network is configured and functional, services are installed and configured--everything is 'working' locally. This took a couple weeks of tinkering in my case, as I had new and unfamiliar network equipment, makeshift servers, and even a Raspberry Pi to learn how to use and configure. Everyone has different network conditions so the intial setup will vary, but the goal here is for our home network to route traffic across our local devices/services without much trouble before moving on.

### Configure Domain Name

Our first task is to set up a domain name. Assuming you already have one, you'll need to update the **nameservers** to make Cloudflare our primary DNS provider, which will allow us to use Cloudflare Tunnels later. If you're not using Cloudflare Tunnels for SSH (I am for the extra layer of authentication) you can skip the nameservers and follow the next steps for A/CNAME records using your own provider's website---if you're not sure how, refer to their documentation. [This article](https://developers.cloudflare.com/dns/zone-setups/full-setup/setup/) details the steps we need to take to make Clouldfare our DNS provider, including registrar-specific instructions to toggle DDNSEC and update nameservers. Just carefully follow the steps provided here and your domain will be ready for the next task. Keep in mind that it may take some time for these changes to take effect, up to 24 hours in some cases---Cloudflare will send an email when the domain is active with the new nameservers.

![The DNS section in the Cloudflare UI](/assets/)

Once these changes become active on your domain name, its DNS settings should become editable within the Cloudflare UI. From here, we'll be adding A/CNAME records which will control where the domain and subdomains point. Our first will be the A record, which we'll be pointing to the static IP address of our VPS (or cloud server). For the CNAME records, you can choose any names you'd like; I'll keep it simple and name them after the services they'll be pointing to. Services might include things like: blog, plex, dashboard, logs, email, calibre, portainer, etc., and they should point to the same IP as our A record (an @ symbol can be used as an alias for your A record). We'll be relying on a reverse proxy to sort out this traffic for us later---for now, all we need to do is set up the subdomains we want and make sure they're being sent to the VPS. Here's an example:

![image of CF UI with DNS records]()

With these records added, our domain is now pointing to where it needs to and we're ready to create our tunnels. We'll start by setting up the Cloudflare Tunnel in the next section, including SSH via Cloudflare Access. Once that is working, we'll move on to setting up the Wireguard tunnel and reverse proxy.

<!-- can revise later to remove VPS section -- IPV4 forwarding can be added as a preroute in WG conf; iptables can be added 

### Configure VPS

To start, we need to enable IP forwarding. This allows incoming traffic from one network interface to be passed to a different network interface and forwarded along that route. This is exactly what we're trying to accomplish with incoming traffic on our server---routing from the default interface to our planned Wireguard interface. Follow the [instructions here](https://openvpn.net/faq/what-is-and-how-do-i-enable-ip-forwarding-on-linux/); we need to edit the sysctl.conf file and uncomment (or add) the line that references ipv4 IP forwarding. Once these changes are saved, we need to configure our tunnels and then set up our firewall rules. (*Note: you could substitute [proper networking via iptables/nftables](https://www.procustodibus.com/blog/2021/04/wireguard-access-control-with-iptables/) here; I'm not great at networking yet and my network is relatively simple, so [we'll use Ubuntu's UFW](https://www.procustodibus.com/blog/2021/05/wireguard-ufw/) when we come to this section later.*)
-->

### Configure Cloudflare Tunnel

To begin, we'll create the local endpoint for our tunnel by using a docker container. We'll follow the instructions provided by Cloudflare, with a few changes I'll note as we come to them.

- Set up Cloudflared docker container -- expose  docker network by:
- Create static route on router for (both) tunnel interfaces
- Set up Cloudflare ZeroTrust Access SSH
- Set up WG on router
- Set up WG on VPS  -- add links back to WG docs for gen keys, etc.
- (Static Route 2)

### Set up SSH

### Configure Wireguard (Router)

### Configure Wireguard Tunnel

### Configure Reverse Proxy

Configure Caddy

<!--

CF tunnel setup 
WG install on ER-X > yvatta-wg
SSH
Caddy setup

The final step will be configuring SSH for any servers or devices. For me, this included an Edgerouter X, Raspberry Pi, Asustor NAS running TrueNAS Scale, remote VPS from a hosting provider, and a spare laptop running Ubuntu Desktop.

## Task Outline

-. Sorting out Local network / DNS
-. Cloudflare account setup
-. Cloudflare DNS
- VPS security and forwarding
- Cloudflared docker container
- Cloudflare Zero Trust / Access
  - ssh access
- Reverse-proxy services with Caddy
- Self-host static content on VPS with CI/CD github action (probably a separate article)

## LINKS

CGNAT
https://en.wikipedia.org/wiki/Carrier-grade_NAT
https://www.purevpn.com/blog/cgnat-port-forwarding
https://www.purevpn.com/blog/what-is-cgnat/
https://www.purevpn.com/blog/how-to-check-whether-or-not-your-isp-performs-cgnat/

WIREGUARD
https://www.wireguard.com
https://www.procustodibus.com/blog/2020/11/wireguard-point-to-site-config/
https://www.procustodibus.com/blog/2021/01/wireguard-endpoints-and-ip-addresses/
https://github.com/WireGuard/wireguard-vyatta-ubnt/wiki/EdgeOS-and-Unifi-Gateway 

CLOUDFLARE
https://developers.cloudflare.com/dns/zone-setups/full-setup/setup/

GOOGLE DOMAINS
https://support.google.com/domains/answer/3290309

IPV4 Forward
https://openvpn.net/faq/what-is-and-how-do-i-enable-ip-forwarding-on-linux/

FIREWALL
https://www.procustodibus.com/blog/2021/04/wireguard-access-control-with-iptables/
https://www.procustodibus.com/blog/2021/05/wireguard-ufw/
-->
