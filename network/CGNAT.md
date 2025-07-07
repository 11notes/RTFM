![banner](https://github.com/11notes/static/blob/main/img/banner/README.png?raw=true)

# I HAVE CGNAT AND CAN’T EXPOSE SERVICES TO WAN, WHAT CAN I DO?

*This question gets asked a lot and is faced with a multitude of solutions that in end all do the exact same thing. So what can you do to solve this?*

# SYNOPSIS 📖

CGANT or nat44 is a method for an ISP to not give the client a routable public IPv4 address. Why do they do this? To cut cost. 254 IPv4 addresses cost about 8k $, that’s ~31.50$ per IP address. With monthly subscription costs in the same or double region, some ISP are hesitant to give their clients an IPv4 address. Instead, they do what you already do at home: Routing with NAT. All your clients at home are in a private LAN with a private address space[^1] that is not routable on the public internet. An ISP using CGNAT is doing exactly the same, just at a larger scale. Each router they provide to their clients (you), is their client in their private network[^1]. That’s why you will see a private IP address[^1] on your WAN side, instead of a public IPv4 address. This prevents you from port forwarding services from WAN to your LAN, since you don’t even have direct WAN access, but your router sits behind the NAT of your ISP.

# HOW CAN I CIRCUMVENT THIS NAT FROM MY ISP?

There is only one solution for this problem: Access to your LAN from WAN, must be done via a third party. That’s the can of worms you have to open. You need a third party that will connect to your LAN and is exposed to WAN, since you, yourself are not directly exposed to WAN but sit behind the NAT of your ISP.

*Just use Tailscale bro, easy as that!* Well, not exactly. First off, ZTNA[^2] solutions like Tailscale, are not exposing services to WAN, they are exposing them to an overlay network that is only accessible within said network. Which means all clients and services must participate in that network to be accessible. You can’t access a service run on Tailscale from a non-Tailscale client. This is akin to a normal VPN, where you need to establish the VPN connection before you can access the services hosted behind the VPN. Using ZTNA requires a control server which needs to be exposed to WAN, just like the services you want to expose behind your CGNAT. Which invalidates the use of ZTNA, since you can’t expose your own control server to WAN. You can use a cloud based control server, like Tailscale or similar offerings, which often are offered for free[^3].

If not using ZTNA, what other options are available? Some use tunneling services offered for free[^3] from providers like Cloudflare, which often results in a MitM[^4] form of proxy connection. The cloud provider accepts the connection on your behalf, and then tunnels it with different methods to your services. This can be a proxy solution like Cloudflare does with their MitM[^4] proxy or a tunnel solution where you have a VPN protocol between you and the cloud provider. Regardless of which method is used, these offerings are often very restricted in terms of bandwidth (how many megabytes per second you get and how many TB of data you can transfer per month or even what kind of data transfer is allowed). This on itself, is another can of worms and you must decide if it’s worth it to you to open it up.

# SO WHAT CAN I DO WHEN I VALUE PRIVACY AND DON’T WANT TO DEPEND ON A FREE PRODUCT?

As initially said, you do have to depend on a third party, but you can depend on a third party that is as much under your control as possible, a VPS[^5]. With a VPS you now have a multitude of options available to you. You can use your VPS to …

- ... run your own control server for a ZTNA solution like [Netbird](https://github.com/11notes/docker-netbird) if you don’t want to expose your services directly to WAN, but via a VPN like system
- ... run a reverse proxy, like [Traefik](https://github.com/11notes/docker-traefik) with some added plugins for protection (geo block, crowdsec, 2FA, passkeys, etc)
- ... run a software router/firewall (bare Linux, opnsense,e tc), that will port forward to your LAN like your actual router would
- ... run a normal VPN (not ZTNA) like Wireguard or OpenVPN to allow accessing your services hosted at home via VPN

Whatever your use case is, it will work. There are still restrictions in terms of bandwidth and traffic, but less so than on compared free tier cloud offering.

You might now ask what a VPS you need to run any of the above? Well, that depends on what you need, but I would say you will be pretty fine with an arm64 based virtual machine, at least 2GB of RAM, 2 CPU cores and at least 2GB of storage. Most VPS have more than that and cost about 5$/month. With such a pricing, and depending on your budget, you can even build a HA[^6] solution by renting a VPS from multiple providers, just in case one VPS is not reachable due to problems with their infrastructure or physical distance, like accessing your German VPS from France, but accessing your US VPS from, well, the US!

If you are waiting for a link from me to a VPS I recommend to you with an added referral from me, sorry to disappoint you, but I don’t role that way. Any search engine and customer reviews will tell you which VPS are worth paying for.

# CONCLUSION

If you don’t care about privacy and the dependency on cloud offerings, use one of the mentioned cloud providers like Tailscale or Cloudflare, to expose your services directly to WAN or via VPN or ZTNA. If you are more concerned about your data being misused, go for a VPS, with the added benefit that you can now run other services[^7] too!

**If you want things done right, you have to do it yourself.**

[^1]: Private IPv4 address spaces (subnets) are defined as followed: [Wikipedia](https://en.wikipedia.org/wiki/Private_network)
[^2]: ZTNA is a fancy acronym for an overlay network with ACL, you can find out more how it works here: [ZTNA explained](https://www.zscaler.com/resources/security-terms-glossary/what-is-zero-trust-network-access)
[^3]: Free cloud offerings are often restricted in their features and you are at the mercy of the company who can cancel your free account at any time for any reason, consider this security implication before signing up for free services. Nothing is free, if it is, you or your data is the product and value they are getting
[^4]: Man in the middle, is a cyber security attack where a service is handling and decrypting a connection on your behalf and then sending you the data. This means that the man in the middle, sees all data in clear text. The goal of such an attack is to exfiltrate information
[^5]: Virtual private servers, are virtual machines that are rented to you on a monthly basis. They occur different costs depending on the CPU architecture, RAM and storage rented to you. They often come by default with a public IPv4 and IPv6 address
[^6]: High availability is the concept that a service stays available even if other nodes or entire data centres are unavailable
[^7]: DNS, NTP, SMTP, just to name a few