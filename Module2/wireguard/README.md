# WireGuard VPN peer-to-site (on router)


In this diagram, we are depicting a home network with some devices and a router where we can install WireGuard.

```
                       public internet              ┌─── wg0 10.10.11.1/24
10.10.11.2/24                                       │        VPN network
        home0│            xxxxxx       ppp0 ┌───────┴┐
           ┌─┴──┐         xx   xxxxx  ──────┤ router │
           │    ├─wlan0  xx       xx        └───┬────┘    home network, .home domain
           │    │       xx        x             │.1       10.10.10.0/24
           │    │        xxx    xxx             └───┬─────────┬─────────┐
           └────┘          xxxxxx                   │         │         │
Laptop in                                         ┌─┴─┐     ┌─┴─┐     ┌─┴─┐
Coffee shop                                       │   │     │   │     │   │
                                                  │pi4│     │NAS│     │...│
                                                  │   │     │   │     │   │
                                                  └───┘     └───┘     └───┘
```

Of course, this setup is only possible if you can install software on the router. Most of the time, when it's provided by your ISP, you can't. But some ISPs allow their device to be put into a bridge mode, in which case you can use your own device (a computer, a Raspberry PI, or something else) as the routing device.

Since the router is the default gateway of the network already, this means you can create a whole new network for your VPN users. You also won't have to create any (D)NAT rules since the router is directly reachable from the Internet.

Let's define some addresses, networks, and terms used in this guide:

- **laptop in coffee shop**: just your normal user at a coffee shop, using the provided Wi-Fi access to connect to their home network. This will be one of our **peers** in the VPN setup.
- **`home0`**: this will be the WireGuard interface on the laptop. It's called `home0` to convey that it is used to connect to the **home** network.
- **router**: the existing router at the home network. It has a public interface `ppp0` that has a routable but dynamic IPv4 address (not CGNAT), and an internal interface at `10.10.10.1/24` which is the default gateway for the home network.
- **home network**: the existing home network (`10.10.10.0/24` in this example), with existing devices that the user wishes to access remotely over the WireGuard VPN.
- **`10.10.11.0/24`**: the WireGuard VPN network. This is a whole new network that was created just for the VPN users.
- **`wg0`** on the **router**: this is the WireGuard interface that we will bring up on the router, at the `10.10.11.1/24` address. It is the gateway for the `10.10.11.0/24` VPN network.

With this topology, if, say, the NAS wants to send traffic to `10.10.11.2/24`, it will send it to the default gateway (since the NAS has no specific route to `10.10.11.0/24`), and the gateway will know how to send it to `10.10.11.2/24` because it has the `wg0` interface on that network.

## Configuration

First, we need to create keys for the peers of this setup. We need one pair of keys for the laptop, and another for the home router:

```bash
 umask 077
 wg genkey > laptop-private.key
 wg pubkey < laptop-private.key > laptop-public.key
 wg genkey > router-private.key
 wg pubkey < router-private.key > router-public.key
```

Let's create the router `wg0` interface configuration file. The file will be `/etc/wireguard/wg0.conf` and have these contents:

```text
[Interface]
PrivateKey = <contents-of-router-private.key>
ListenPort = 51000
Address = 10.10.11.1/24

[Peer]
PublicKey = <contents-of-laptop-public.key>
AllowedIPs = 10.10.11.2
```

There is no `Endpoint` configured for the laptop peer, because we don't know what IP address it will have beforehand, nor will that IP address be always the same. This laptop could be connecting from a coffee shop's free Wi-Fi, an airport lounge, or a friend's house.

Not having an endpoint here also means that the home network side will never be able to *initiate*  the VPN connection. It will sit and wait, and can only *respond* to VPN handshake requests, at which time it will learn the endpoint from the peer and use that until it changes (i.e. when the peer reconnects from a different site) or it times out.

> **Important**:
> This configuration file contains a secret: **PrivateKey**.
> Make sure to adjust its permissions accordingly, as follows:
> ```
> sudo chmod 0600 /etc/wireguard/wg0.conf
> sudo chown root: /etc/wireguard/wg0.conf
> ```

When activated, this will bring up a `wg0` interface with the address `10.10.11.1/24`, listening on port `51000/udp`, and add a route for the `10.10.11.0/24` network using that interface.

The `[Peer]` section is identifying a peer via its public key, and listing who can connect from that peer. This `AllowedIPs` setting has two meanings:

- When sending packets, the `AllowedIPs` list serves as a routing table, indicating that this peer's public key should be used to encrypt the traffic.
- When receiving packets, `AllowedIPs` behaves like an access control list. After decryption, the traffic is only allowed if it matches the list.

Finally, the `ListenPort` parameter specifies the **UDP** port on which WireGuard will listen for traffic. This port will have to be allowed in the firewall rules of the router. There is neither a default nor a standard port for WireGuard, so you can pick any value you prefer.

Now let's create a similar configuration on the other peer, the laptop. Here the interface is called `home0`, so the configuration file is `/etc/wireguard/home0.conf`:

```
[Interface]
PrivateKey = <contents-of-laptop-private.key>
ListenPort = 51000
Address = 10.10.11.2/24

[Peer]
PublicKey = <contents-of-router-public.key>
Endpoint = <home-ppp0-IP-or-hostname>:51000
AllowedIPs = 10.10.11.0/24,10.10.10.0/24
```

> **Important**:
> As before, this configuration file contains a secret: **PrivateKey**.
> You need to adjust its permissions accordingly, as follows:
> ```
> sudo chmod 0600 /etc/wireguard/home0.conf
> sudo chown root: /etc/wireguard/home0.conf
> ```

We have given this laptop the `10.10.11.2/24` address. It could have been any valid address in the `10.10.11.0/24` network, as long as it doesn't collide with an existing one, and is allowed in the router's peer's `AllowedIPs` list.

> **Note**:
> You may have noticed by now that address allocation is manual, and not via something like {term}`DHCP`. Keep tabs on it!

In the `[Peer]` stanza for the laptop we have:

- The usual **`PublicKey`** item, which identifies the peer. Traffic to this peer will be encrypted using this public key.
- **`Endpoint`**: this tells WireGuard where to actually send the encrypted traffic to. Since in our scenario the laptop will be initiating connections, it has to know the public IP address of the home router. If your ISP gave you a fixed IP address, great! You have nothing else to do. If, however, you have a dynamic IP address (one that changes every time you establish a new connection), then you will have to set up some sort of dynamic {term}`DNS` service. There are many such services available for free on the Internet, but setting one up is out of scope for this guide.
- In **`AllowedIPs`** we list our destinations. The VPN network `10.10.11.0/24` is listed so that we can ping `wg0` on the home router as well as other devices on the same VPN, and the actual home network, which is `10.10.10.0/24`. 

If we had used `0.0.0.0/0` alone in `AllowedIPs`, then the VPN would become our default gateway, and all traffic would be sent to this peer. See {ref}`Default Gateway <using-the-vpn-as-the-default-gateway>` for details on that type of setup.

## Testing

With these configuration files in place, it's time to bring the WireGuard interfaces up.

On the home router, run:

```bash

 sudo wg-quick up wg0

```

```
[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
[#] ip -4 address add 10.10.11.1/24 dev wg0
[#] ip link set mtu 1378 up dev wg0
```

Verify you have a `wg0` interface up with an address of `10.10.11.1/24`:

```bash
 ip a show dev wg0

```
```
9: wg0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1378 qdisc noqueue state UNKNOWN group default qlen 1000
    link/none
    inet 10.10.11.1/24 scope global wg0
       valid_lft forever preferred_lft forever
```

And a route to the `10.10.1.0/24` network via the `wg0` interface:

```bash
 ip route | grep wg0
10.10.11.0/24 dev wg0 proto kernel scope link src 10.10.11.1
```

And `wg show` should show some status information, but no connected peer yet:

```bash
 sudo wg show
```
```
interface: wg0
  public key: <router public key>
  private key: (hidden)
  listening port: 51000

peer: <laptop public key>
  allowed ips: 10.10.11.2/32
```

In particular, verify that the listed public keys match what you created (and expected!).

Before we start the interface on the other peer, it helps to leave the above `show` command running continuously, so we can see when there are changes:

```bash
 sudo watch wg show
```

Now start the interface on the laptop:

```bash
 sudo wg-quick up home0
[#] ip link add home0 type wireguard
[#] wg setconf home0 /dev/fd/63
[#] ip -4 address add 10.10.11.2/24 dev home0
[#] ip link set mtu 1420 up dev home0
[#] ip -4 route add 10.10.10.0/24 dev home0
```

Similarly, verify the interface's IP and added routes:

```bash
 ip a show dev home0

```
```
24: home0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN group default qlen 1000
    link/none
    inet 10.10.11.2/24 scope global home0
       valid_lft forever preferred_lft forever

 ip route | grep home0
10.10.10.0/24 dev home0 scope link
10.10.11.0/24 dev home0 proto kernel scope link src 10.10.11.2
```

Up to this point, the `wg show` output on the home router probably didn't change. That's because we haven't sent any traffic to the home network, which didn't trigger the VPN yet. By default, WireGuard is very "quiet" on the network.

If we trigger some traffic, however, the VPN will "wake up". Let's ping the internal address of the home router a few times:

```bash
 ping -c 3 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=603 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=300 ms
64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=304 ms
```

Note how the first ping was slower. That's because the VPN was "waking up" and being established. Afterwards, with the tunnel already established, the latency reduced.

At the same time, the `wg show` output on the home router will have changed to something like this:

```bash

 sudo wg show
```
```
interface: wg0
  public key: <router public key>
  private key: (hidden)
  listening port: 51000

peer: <laptop public key>
  endpoint: <laptop public IP>:51000
  allowed ips: 10.10.11.2/32
  latest handshake: 1 minute, 8 seconds ago
  transfer: 564 B received, 476 B sent
```











# WireGuard on an internal system (peer-to-site)

Sometimes it's not possible to install WireGuard {ref}`on the home router itself <wireguard-vpn-peer-to-site-on-router>`. Perhaps it's a closed system to which you do not have access, or there is no easy build for that architecture, or any of the other possible reasons.

However, you do have a spare system inside your network that you could use. Here we are going to show one way to make this work. There are others, but we believe this to be the least "involved" as it only requires a couple of (very common) changes in the router itself: NAT port forwarding, and {term}`DHCP` range editing.

To recap, our home network has the `10.10.10.0/24` address, and we want to connect to it from a remote location and be "inserted" into that network as if we were there:

```
                       public internet
10.10.10.3/24
        home0│            xxxxxx       ppp0 ┌────────┐
           ┌─┴──┐         xx   xxxxx  ──────┤ router │
           │    ├─ppp0  xxx       xx        └───┬────┘    home network, .home domain
           │    │       xx        x             │         10.10.10.0/24
           │    │        xxx    xxx             └───┬─────────┬─────────┐
           └────┘          xxxxxx                   │         │         │
                                                  ┌─┴─┐     ┌─┴─┐     ┌─┴─┐
                                            wg0 ──┤   │     │   │     │   │
                                  10.10.10.10/32  │pi4│     │NAS│     │...│
                                                  │   │     │   │     │   │
                                                  └───┘     └───┘     └───┘
Reserved for VPN users:
10.10.10.2-9
```

## Router changes

Since, in this scenario, we don't have a new network dedicated to our VPN users, we need to "carve out" a section of the home network and reserve it for the VPN.

The easiest way to reserve IPs for the VPN is to change the router configuration (assuming it's responsible for DHCP in this network) and tell its DHCP server to only hand out addresses from a specific range, leaving a "hole" for our VPN users.

For example, in the case of the `10.10.10.0/24` network, the DHCP server on the router might already be configured to hand out IP addresses from `10.10.10.2` to `10.10.10.254`. We can carve out a "hole" for our VPN users by reducing the DHCP range, as in this table:

|||
|---|---|
| Network | `10.10.10.0/24` |
| Usable addresses | `10.10.10.2` -- `10.10.10.254` (`.1` is the router) |
| DHCP range | `10.10.10.50` -- `10.10.10.254` |
| VPN range | `10.10.10.10` -- `10.10.10.49` |
|||

Or via any other layout that is better suited for your case. In this way, the router will never hand out a DHCP address that conflicts with one that we selected for a VPN user.

The second change we need to do in the router is to **port forward** the WireGuard traffic to the internal system that will be the endpoint. In the diagram above, we selected the `10.10.10.10` system to be the internal WireGuard endpoint, and we will run it on the `51000/udp` port. Therefore, you need to configure the router to forward all `51000/udp` traffic to `10.10.10.10` on the same `51000/udp` port.

Finally, we also need to allow hosts on the internet to send traffic to the router on the `51000/udp` port we selected for WireGuard. This is done in the firewall rules of the device. Sometimes, performing the port forwarding as described earlier also configures the firewall to allow that traffic, but it's better to check.

Now we are ready to configure the internal endpoint.

## Configure the internal WireGuard endpoint

Install the `wireguard` package:

```bash
 sudo apt install wireguard
```

Generate the keys for this host:

```bash
 umask 077
 wg genkey > internal-private.key
 wg pubkey < internal-private.key > internal-public.key
```

And create the `/etc/wireguard/wg0.conf` file with these contents:

```
[Interface]
Address = 10.10.10.10/32
ListenPort = 51000
PrivateKey = <contents of internal-private.key>

[Peer]
# laptop
PublicKey = <contents of laptop-public.key>
AllowedIPs = 10.10.10.11/32 # any available IP in the VPN range
```

> **Note**:
> Just like in the {ref}`peer-to-site <wireguard-vpn-peer-to-site-on-router>` scenario with WireGuard on the router, there is no `Endpoint` configuration here for the laptop peer, because we don't know where it will be connecting from beforehand.

The final step is to configure this internal system as a router for the VPN users. For that, we need to enable a couple of settings:

- **`ip_forward`**: to enable forwarding (aka, routing) of traffic between interfaces.
- **`proxy_arp`**: to reply to Address Resolution Protocol (ARP) requests on behalf of the VPN systems, as if they were locally present on the network segment.

To do that, and make it persist across reboots, create the file `/etc/sysctl.d/70-wireguard-routing.conf` file with this content:

```
net.ipv4.ip_forward = 1
net.ipv4.conf.all.proxy_arp = 1
```

Then run this command to apply those settings:

```bash
 sudo sysctl -p /etc/sysctl.d/70-wireguard-routing.conf -w
```

Now the WireGuard interface can be brought up:

```bash
 sudo wg-quick up wg0
```

## Configuring the peer

The peer configuration will be very similar to what was done before. What changes will be the address, since now it won't be on an exclusive network for the VPN, but will have an address carved out of the home network block.

Let's call this new configuration file `/etc/wireguard/home_internal.conf`:

```
[Interface]
ListenPort = 51000
Address = 10.10.10.11/24
PrivateKey = <contents of the private key for this system>

[Peer]
PublicKey = <contents of internal-public.key>
Endpoint = <home-ppp0-IP-or-hostname>:51000
AllowedIPs = 10.10.10.0/24
```

And bring up this WireGuard interface:

```bash
 sudo wg-quick up home_internal
```

> **Note**:
> There is no need to add an index number to the end of the interface name. That is a convention, but not strictly a requirement.

## Testing

With the WireGuard interfaces up on both peers, traffic should flow seamlessly in the `10.10.10.0/24` network between remote and local systems.

More specifically, it's best to test the non-trivial cases, that is, traffic between the remote peer and a host other than the one with the WireGuard interface on the home network.










# WireGuard VPN site-to-site

Another usual VPN configuration where one could deploy WireGuard is to connect two distinct networks over the internet. Here is a simplified diagram:

```
                      ┌─────── WireGuard tunnel ──────┐
                      │         10.10.9.0/31          │
                      │                               │
         10.10.9.0 wgA│               xx              │wgB 10.10.9.1
                    ┌─┴─┐          xxx  xxxx        ┌─┴─┐
    alpha site      │   │ext     xx        xx    ext│   │  beta site
                    │   ├───    x           x    ───┤   │
    10.10.10.0/24   │   │      xx           xx      │   │  10.10.11.0/24
                    │   │      x             x      │   │
                    └─┬─┘      x              x     └─┬─┘
            10.10.10.1│        xx             x       │10.10.11.1
    ...┌─────────┬────┘          xx   xxx    xx       └───┬─────────┐...
       │         │                  xx   xxxxx            │         │
       │         │                                        │         │
     ┌─┴─┐     ┌─┴─┐           public internet          ┌─┴─┐     ┌─┴─┐
     │   │     │   │                                    │   │     │   │
     └───┘     └───┘                                    └───┘     └───┘
```

The goal here is to seamlessly integrate network **alpha** with network **beta**, so that systems on the alpha site can transparently access systems on the beta site, and vice-versa.

Such a setup has a few particular details:

- Both peers are likely to be always up and running.
- We can't assume one side will always be the initiator, like the laptop in a coffee shop scenario.
- Because of the above, both peers should have a static endpoint, like a fixed IP address, or valid domain name.
- Since we are not assigning VPN IPs to all systems on each side, the VPN network here will be very small (a `/31`, which allows for two IPs) and only used for routing. The only systems with an IP in the VPN network are the gateways themselves.
- There will be no NAT applied to traffic going over the WireGuard network. Therefore, the networks of both sites **must** be different and not overlap.

This is what an MTR (My Traceroute) report from a system in the beta network to an alpha system will look like:

```bash
ubuntu@b1:~$ mtr -n -r 10.10.10.230
Start: 2022-09-02T18:56:51+0000
HOST: b1                Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- 10.10.11.1       0.0%    10    0.1   0.1   0.1   0.2   0.0
  2.|-- 10.10.9.0        0.0%    10  299.6 299.3 298.3 300.0   0.6
  3.|-- 10.10.10.230     0.0%    10  299.1 299.1 298.0 300.2   0.6
```

> **Note**:
> Technically, a `/31` Classless Inter-Domain Routing (CIDR) network has no usable IP addresses, since the first one is the network address, and the second (and last) one is the broadcast address. [RFC 3021](https://www.ietf.org/rfc/rfc3021.txt) allows for it, but if you encounter routing or other networking issues, switch to a `/30` CIDR and its two valid host IPs.

## Configure WireGuard

On the system that is the gateway for each site (that has internet connectivity), we start by installing WireGuard and generating the keys. For the **alpha** site:

```bash
 sudo apt install wireguard
 wg genkey | sudo tee /etc/wireguard/wgA.key
 sudo cat /etc/wireguard/wgA.key | wg pubkey | sudo tee /etc/wireguard/wgA.pub
```

And the configuration on alpha will be:

```
[Interface]
PostUp = wg set %i private-key /etc/wireguard/%i.key
Address = 10.10.9.0/31
ListenPort = 51000

[Peer]
# beta site
PublicKey = <contents of /etc/wireguard/wgB.pub>
AllowedIPs = 10.10.11.0/24,10.10.9.0/31
Endpoint = <beta-gw-ip>:51000
```

On the gateway for the **beta** site we take similar steps:

```bash
 sudo apt install wireguard
 wg genkey | sudo tee /etc/wireguard/wgB.key
 sudo cat /etc/wireguard/wgB.key | wg pubkey | sudo tee /etc/wireguard/wgB.pub
```

And create the corresponding configuration file for beta:

```
[Interface]
Address = 10.10.9.1/31
PostUp = wg set %i private-key /etc/wireguard/%i.key
ListenPort = 51000

[Peer]
# alpha site
PublicKey = <contents of /etc/wireguard/wgA.pub>
AllowedIPs = 10.10.10.0/24,10.10.9.0/31
Endpoint = <alpha-gw-ip>:51000
```

> **Important**:
> WireGuard is being set up on the gateways for these two networks. As such, there are no changes needed on individual hosts of each network, but keep in mind that the WireGuard tunneling and encryption is only happening between the *alpha* and *beta* gateways, and **NOT** between the hosts of each network.

## Bring the interfaces up

Since this VPN is permanent between static sites, it's best to use the systemd unit file for `wg-quick` to bring the interfaces up and control them in general. In particular, we want them to be brought up automatically on reboot events.

On **alpha**:

```bash
 sudo systemctl enable --now wg-quick@wgA
```

And similarly on **beta**:

```bash
 sudo systemctl enable --now wg-quick@wgB
```

This both enables the interface on reboot, and starts it right away.

## Firewall and routing

Both gateways probably already have some routing and firewall rules. These might need changes depending on how they are set up.

The individual hosts on each network won't need any changes regarding the remote alpha or beta networks, because they will just send that traffic to the default gateway (as any other non-local traffic), which knows how to route it because of the routes that `wg-quick` added.

In the configuration we did so far, there have been no restrictions in place, so traffic between both sites flows without impediments.

In general, what needs to be done or checked is:

- Make sure both gateways can contact each other on the specified endpoint addresses and UDP port. In the case of this example, that is port `51000`. For extra security, create a firewall rule that only allows each peer to contact this port, instead of the Internet at large.
- Do NOT masquerade or NAT the traffic coming from the internal network and going out via the WireGuard interface towards the other site. This is purely routed traffic.
- There shouldn't be any routing changes needed on the gateways, since `wg-quick` takes care of adding the route for the remote site, but do check the routing table to see if it makes sense (`ip route` and `ip route | grep wg` are a good start).
- You may have to create new firewall rules if you need to restrict traffic between the alpha and beta networks.

  For example, if you want to prevent SSH between the sites, you could add a firewall rule like this one to **alpha**:

  ` sudo iptables -A FORWARD -i wgA -p tcp --dport 22 -j REJECT`

  And similarly on **beta**:

  ` sudo iptables -A FORWARD -i wgB -p tcp --dport 22 -j REJECT`

  You can add these as `PostUp` actions in the WireGuard interface config. Just don't forget to remove them in the corresponding `PreDown` hook, or you will end up with multiple rules.








(using-the-vpn-as-the-default-gateway)=
# Using the VPN as the default gateway

WireGuard can be set up to route all traffic through the VPN, and not just specific remote networks. There could be many reasons to do this, but mostly they are related to privacy.

Here we will assume a scenario where the local network is considered to be "untrusted", and we want to leak as little information as possible about our behaviour on the Internet. This could apply to the case of an airport, or a coffee shop, a conference, a hotel, or any other public network.

```
                       public untrusted          ┌── wg0 10.90.90.2/24
10.90.90.1/24          network/internet          │   VPN network
        wg0│            xxxxxx            ┌──────┴─┐
         ┌─┴──┐         xx   xxxxx  ──────┤ VPN gw │
         │    ├─wlan0  xx       xx   eth0 └────────┘
         │    │       xx        x 
         │    │        xxx    xxx
         └────┘          xxxxxx
         Laptop
```

For the best results, we need a system we can reach on the internet and that we control. Most commonly this can be a simple small VM in a public cloud, but a home network also works. Here we will assume it's a brand new system that will be configured from scratch for this very specific purpose.

## Install and configure WireGuard

Let's start the configuration by installing WireGuard and generating the keys. On the client, run the following commands:

```bash
sudo apt install wireguard
umask 077
wg genkey > wg0.key
wg pubkey < wg0.key > wg0.pub
sudo mv wg0.key wg0.pub /etc/wireguard
```

And on the gateway server:

```bash
sudo apt install wireguard
umask 077
wg genkey > gateway0.key
wg pubkey < gateway0.key > gateway0.pub
sudo mv gateway0.key gateway0.pub /etc/wireguard
```

On the client, we will create `/etc/wireguard/wg0.conf`:

```
[Interface]
PostUp = wg set %i private-key /etc/wireguard/wg0.key
ListenPort = 51000
Address = 10.90.90.1/24

[Peer]
PublicKey = <contents of gateway0.pub>
Endpoint = <public IP of gateway server>
AllowedIPs = 0.0.0.0/0
```

Key points here:
- We selected the `10.90.90.1/24` IP address for the WireGuard interface. This can be any private IP address, as long as it doesn't conflict with the network you are on, so double check that. If it needs to be changed, don't forget to also change the IP for the WireGuard interface on the gateway server.
- The `AllowedIPs` value is `0.0.0.0/0`, which means "all IPv4 addresses".
- We are using `PostUp` to load the private key instead of specifying it directly in the configuration file, so we don't have to set the permissions on the config file to `0600`.

The counterpart configuration on the gateway server is `/etc/wireguard/gateway0.conf` with these contents:

```
[Interface]
PostUp = wg set %i private-key /etc/wireguard/%i.key
Address = 10.90.90.2/24
ListenPort = 51000

[Peer]
PublicKey = <contents of wg0.pub>
AllowedIPs = 10.90.90.1/32
```

Since we don't know where this remote peer will be connecting from, there is no `Endpoint` setting for it, and the expectation is that the peer will be the one initiating the VPN.

This finishes the WireGuard configuration on both ends, but there is one extra step we need to take on the gateway server.

## Routing and masquerading

The WireGuard configuration that we did so far is enough to send the traffic from the client (in the untrusted network) to the gateway server. But what about from there onward? There are two extra configuration changes we need to make on the gateway server:

- Masquerade (or apply source NAT rules) the traffic from `10.90.90.1/24`.
- Enable IPv4 forwarding so our gateway server acts as a router.

To enable routing, create `/etc/sysctl.d/70-wireguard-routing.conf` with this content:

```
net.ipv4.ip_forward = 1
```

And run:

```bash
sudo sysctl -p /etc/sysctl.d/70-wireguard-routing.conf -w
```

To masquerade the traffic from the VPN, one simple rule is needed:

```bash
sudo iptables -t nat -A POSTROUTING -s 10.90.90.0/24 -o eth0 -j MASQUERADE
```

Replace `eth0` with the name of the network interface on the gateway server, if it's different.

To have this rule persist across reboots, you can add it to `/etc/rc.local` (create the file if it doesn't exist and make it executable):

```
#!/bin/sh
iptables -t nat -A POSTROUTING -s 10.90.90.0/24 -o eth0 -j MASQUERADE
```

This completes the gateway server configuration.

## Testing

Let's bring up the WireGuard interfaces on both peers. On the gateway server:

```bash

 sudo wg-quick up gateway0
```
```

[#] ip link add gateway0 type wireguard
[#] wg setconf gateway0 /dev/fd/63
[#] ip -4 address add 10.90.90.2/24 dev gateway0
[#] ip link set mtu 1378 up dev gateway0
[#] wg set gateway0 private-key /etc/wireguard/gateway0.key
```

And on the client:

```bash

sudo wg-quick up wg0
```
```

[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
[#] ip -4 address add 10.90.90.1/24 dev wg0
[#] ip link set mtu 1420 up dev wg0
[#] wg set wg0 fwmark 51820
[#] ip -4 route add 0.0.0.0/0 dev wg0 table 51820
[#] ip -4 rule add not fwmark 51820 table 51820
[#] ip -4 rule add table main suppress_prefixlength 0
[#] sysctl -q net.ipv4.conf.all.src_valid_mark=1
[#] nft -f /dev/fd/63
[#] wg set wg0 private-key /etc/wireguard/wg0.key
```

From the client you should now be able to verify that your traffic reaching out to the internet is going through the gateway server via the WireGuard VPN. For example:

```bash
 mtr -r 1.1.1.1
```
```

Start: 2022-09-01T12:42:59+0000
HOST: laptop.lan                 Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- 10.90.90.2                 0.0%    10  184.9 185.5 184.9 186.9   0.6
  2.|-- 10.48.128.1                0.0%    10  185.6 185.8 185.2 188.3   0.9
  (...)
  7.|-- one.one.one.one            0.0%    10  186.2 186.3 185.9 186.6   0.2
```

Above, hop 1 is the `gateway0` interface on the gateway server, then `10.48.128.1` is the default gateway for that server, then come some in-between hops, and the final hit is the target.

If you only look at the output of `ip route`, however, it's not immediately obvious that the WireGuard VPN is the default gateway:

```bash
$ ip route
default via 192.168.122.1 dev enp1s0 proto dhcp src 192.168.122.160 metric 100 
10.90.90.0/24 dev wg0 proto kernel scope link src 10.90.90.1 
192.168.122.0/24 dev enp1s0 proto kernel scope link src 192.168.122.160 metric 100 
192.168.122.1 dev enp1s0 proto dhcp scope link src 192.168.122.160 metric 100 
```

That's because WireGuard is using `fwmarks` and policy routing. WireGuard cannot simply set the `wg0` interface as the default gateway: that traffic needs to reach the specified endpoint on port `51000/UDP` outside of the VPN tunnel.

If you want to dive deeper into how this works, check `ip rule list`, `ip route list table 51820`, and consult the documentation on "Linux Policy Routing".

