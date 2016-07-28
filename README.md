namespaced-openvpn
==================

`namespaced-openvpn` is a wrapper script for OpenVPN on Linux that uses network namespaces to solve a variety of deanonymization, information disclosure, and usability issues.

    sudo /path/to/namespaced-openvpn --config ./my_openvpn_config_file
    sudo ip netns exec protected sudo -u myusername -i

## Summary

OpenVPN is widely used as a privacy technology: both for concealing the user's IP address from remote services, and for protecting otherwise unencrypted IP traffic from interception or modification by a malicious local Internet provider. However, default configurations of OpenVPN are susceptible to various "leaks" or "holes" that can violate these guarantees. Notable examples include route injection attacks and the so-called "Port Fail" vulnerability.

In my view, these issues have two root causes. One is that VPNs are not inherently a privacy technology --- their core function is to bridge two trusted networks using an untrusted network (generally the public Internet) as a link. On this reading, since the fundamental role of VPNs is to *connect* networks, rather than to *isolate* them (as required in the privacy use case), it is unsurprising that in the absence of additional precautions, they can allow information to escape. The second cause is that the `openvpn` process itself, the system-level network configuration software (e.g., `NetworkManager`), and the user's applications are all sharing a single routing table --- and when their use cases for this shared resource come into conflict, it can lead to expectations being violated.

This repository contains two approaches to these problems:

* `namespaced-openvpn`, a wrapper script which systematically solves all of the enumerated security issues by moving the tunnel interface into an isolated network namespace
* `seal-unseal-gateway`, a helper script which solves "Port Fail" only, but which otherwise preserves the default OpenVPN routing behavior

This code is released under the MIT (Expat) license.

## Security and usability problems with OpenVPN's default configuration

OpenVPN's `--redirect-gateway` option (described in detail in its [manual page](https://community.openvpn.net/openvpn/wiki/Openvpn23ManPage)) is the basic mechanism by which it attempts to provide network-layer privacy. In the following examples, a physical interface `eth0` is connected to a LAN on `192.168.1.0/24`, with `192.168.1.1` as its default gateway; `1.2.3.4` is the publicly routable address of the remote VPN server, and `10.10.10.10` is the default gateway of the tunnel interface `tun0`. If the `--redirect-gateway` option is set by the client or pushed from the server side, after establishing the tunnel, the `openvpn` process performs these three steps:

1. First, create a static route for `1.2.3.4` going over `eth0` via `192.168.1.1`
2. Delete the default route over `eth0` via `192.168.1.1`
3. Add a new default route over `tun0` via `10.10.10.10`

(Under a near variant of this option, `--redirect-gateway def1`, steps 2 and 3 are combined by adding routes to `0.0.0.0/1` and `128.0.0.0/1` via `tun0`. These routes override the default route, while being in turn overridden by the static route defined in step 1, without requiring the deletion of the original default route. The distinction between these two configurations will not be important.)

The result is a routing table that routes all public IPs over `tun0`, with the exception of `1.2.3.4`, which is still routed over `eth0`: without this exception, the `openvpn` process itself would be stuck in a routing loop, unable to send its encrypted output packets to the Internet. The LAN subnet `192.168.1.0/24` will still be routed over `eth0` as well: otherwise the system would lose access to LAN services, such as printers. We can now describe the aforementioned problems in detail:

### Route injection attacks

Route injection attacks are described by [Perta et al., 2015](https://www.eecs.qmul.ac.uk/~hamed/papers/PETS2015VPN.pdf). A brief example: suppose that you're connected to a VPN, but your physical interface is connected to a malicious local gateway (e.g., a rogue wireless access point). Your DNS server is set to `8.8.8.8`, which is correctly being routed over the VPN. If the local gateway guesses the address of your DNS server, it can force a DHCP renew on your physical interface and then claim that the gateway's IP is `8.8.8.8`. Your DHCP client will then add a route for 8.8.8.8 over the physical interface, allowing interception and modification of your DNS requests.

### "Port Fail"

["Port Fail"](https://www.perfect-privacy.com/blog/2015/11/26/ip-leak-vulnerability-affecting-vpn-providers-with-port-forwarding/) is an issue first documented by the commercial VPN provider Perfect Privacy. Some VPN providers offer to forward a port from the publicly routable VPN gateway back to the client, over the tunnel interface. Suppose a malicious adversary is a client of the same gateway `1.2.3.4` as you, and that `1.2.3.4` is also the egress for the VPN traffic. If the adversary gets a forward of port 56789 on the gateway, then tricks you into accessing a network service (e.g., by including a tracking pixel in a webpage they control) hosted at `1.2.3.4:56789`, your request will be routed over `eth0` instead of `tun0` and the adversary will see your real IP address.

A more salient concern than such a deanonymization attack may be the possibility of "Port Fail" occurring by *accident*. For example, a BitTorrent client running over the VPN with port 56789 forwarded will see trackers reporting `1.2.3.4:56789` as a leecher in its swarms --- it will then attempt to request chunks from *itself* over `eth0`. Depending on the client's support for BitTorrent protocol encryption, this traffic may be identifiable to ISP deep packet inspection as BitTorrent traffic, or result in the disclosure of BitTorrent infohashes.

### Asymmetric routing
In a [Medium article](https://medium.com/@ValdikSS/another-critical-vpn-vulnerability-and-why-port-fail-is-bullshit-352b2ebd22e2), ValdikSS describes a different deanonymization vulnerability related to port forwarding. Suppose that you first enable your VPN, which forwards port 56789 to you from the egress `1.2.3.4`, and then also forward port 56789 from the egress IP of your physical interface `eth0` (call it `2.4.6.8`) via a NAT traversal mechanism like [UPnP](https://en.wikipedia.org/wiki/Universal_Plug_and_Play#NAT_traversal). An adversary who knows that you have a network service accessible at `1.2.3.4:56789`, and who can guess a set of candidate IPs including `2.4.6.8`, can attempt to access port 56789 on each of the candidate IPs. When the adversary sends a packet to `2.4.6.8` and gets a reply from `1.2.3.4`, your real IP will be revealed. (In particular, for UDP services, the IPv4 space is small enough that it's feasible to send a UDP packet to *every* publicly routable address.)

On Linux, this attack already has an effective mitigation: the sysctl options `net.ipv4.conf.*.rp_filter`, when set to `1` (as is the default in many distributions), will drop incoming packets with "asymmetric routes" (such as the attacker's probes, which come in on `eth0` but whose replies would go out on `tun0`).

### IPv6 leaks

"IPv6 leaks" are a fairly trivial problem: on a dual-stack system, changing the default route for IPv4 has no effect on the IPv6 stack, so applications will continue to route their IPv6 traffic over the physical interface. Despite the straightforward nature of the issue, its incidence in the wild is apparently high: [Perta et al., 2015](https://www.eecs.qmul.ac.uk/~hamed/papers/PETS2015VPN.pdf) discuss the scope of the problem.

### DNS leaks

Many residential gateways include a caching DNS resolver (such as `dnsmasq`) and then push their own IP to their DHCP clients as the nameserver. Since by default, OpenVPN does not remove LAN routes and does not modify `/etc/resolv.conf`, it will not prevent clients in this situation from having their DNS requests routed over the physical interface to the gateway, which will then forward them in cleartext over the public Internet.

### ping-restart

This is a usability issue, rather than a privacy issue, but it stems from the same root cause (the shared routing table) as many of the privacy issues. Commercial VPN providers commonly balance traffic to their endpoints via round-robin DNS: the remote endpoint (e.g., `vpn.example.com`) will have A records for multiple IPs. Periodically, a server may be shut down and replaced with another. By default, OpenVPN's `--ping-restart` option will respond to this by gracefully restarting the session after 120 seconds of inactivity: without bringing down the routes, the client will attempt to re-resolve the server's address, then reconnect to the remote. But if name resolution returns a new IP (say `1.2.3.5`) for the remote, the client will be unable to connect to it, because only `1.2.3.4` was whitelisted to use `eth0` --- `1.2.3.5` is still being routed over the defunct `tun0` interface. (In fact, if DNS is being routed over `tun0`, we won't even get this far, because we'll be unable to resolve the name `vpn.example.com` a second time.)

## namespaced-openvpn

The [network namespace](https://lwn.net/Articles/580893/) functionality of Linux provides, in effect, additional isolated copies of the entire kernel networking stack. The idea behind `namespaced-openvpn` is this: the `openvpn` process itself can run in the root namespace, but its tunnel interface `tun0` can be transferred into a new, protected network namespace. This new namespace will have the loopback adapter `lo` and `tun0` as its only interfaces, and all non-localhost traffic will be routed over `tun0`. The `openvpn` process is not disrupted by this because it communicates with `tun0` via the file descriptor it opened to `/dev/net/tun`, which is unaffected by the change of namespace.

As long as sensitive applications are correctly launched within the new, isolated namespace, all of the enumerated issues are systematically resolved:

1. Route injection is impossible because `NetworkManager`, `dhclient`, etc. are running in the root namespace, so they're free to respond to any changes in the external network environment with no danger of affecting the default route in the protected namespace.
2. "Port Fail" is blocked because the protected namespace has no routing exception for the remote gateway: every packet must go to `tun0`.
3. Asymmetric routing attacks, IPv6 leaks, and DNS leaks are all blocked because the protected namespace has no access to any physical interface.
4. The `openvpn` process can freely restart because it runs in the root namespace, which has unmodified routes --- so its DNS request for the remote, and then its handshake with the resulting remote IP, all use `eth0`.

This approach has some further strengths:

1. It does not require any configuration changes to the root namespace, e.g., recreating `eth0` as a virtual bridge.
2. Non-sensitive applications are free to use the physical interfaces, which may have better bandwidth or latency characteristics.
3. A `namespaced-openvpn` instance can peacefully coexist with another OpenVPN connection in the root namespace, without any concerns about conflicting private IPv4 addresses and routes.
4. `openvpn` can be stopped and started without exposing processes in the protected namespace. If `tun0` goes away, those processes don't revert to using a physical interface; instead, they have no connectivity at all.

Use it like this:

    sudo /path/to/namespaced-openvpn --config ./my_openvpn_config_file

The new, isolated namespace will be named `protected` by default. Start an unprivileged shell in it like this:

    sudo ip netns exec protected sudo -u myusername -i

Any applications started from this shell will inherit the namespace.

## seal-unseal-gateway
Unfortunately, `namespaced-openvpn` sacrifices one of the traditional strengths of VPNs as privacy tools: it is relatively prone to user error, because the user must be careful to start any sensitive applications in the protected namespace. Processes running in the root namespace receive no protection. Therefore, it's worth presenting an alternative approach, one applicable to a traditional configuration that alters routes in the root namespace.

`seal-unseal-gateway` is a helper script that addresses the "Port Fail" vulnerability. It attempts to stop processes other than `openvpn` itself from using the whitelisted route. Specifically, it uses the owner matching functionality of `iptables` to drop outgoing packets to the remote gateway, unless they originate from a process with the same EUID as `openvpn`. Since `openvpn` will typically run as root or as a dedicated user, ordinary applications will be unable to use the route. (Some sources on the Internet claim that `iptables` can do owner matching by PID, which would be more precise. However, this functionality was removed from the kernel in [2005](https://github.com/torvalds/linux/commit/34b4a4a624bafe089107966a6c56d2a1aca026d4).)

Use it by adding these lines to your OpenVPN config file (or adding the equivalent command-line options):

    script-security 2
    up /path/to/seal_unseal_gateway
    down /path/to/seal_unseal_gateway

The other privacy issues have relatively standard mitigations. To wit, route injection can be mitigated by using only trusted DHCP servers (e.g., trusted residential gateways), IPv6 leaks can be mitigated by disabling IPv6 (`sysctl -w net.ipv6.conf.all.disable_ipv6=1`), DNS leaks can be mitigated by ensuring that no LAN nameservers appear in `/etc/resolv.conf`, and asymmetric routing attacks can be mitigated by using only trusted DHCP servers.

## Caveats

This is alpha software. It has only been tested with a few VPN configurations, and with very new versions of OpenVPN (2.3.11) and the Linux kernel (4.6). If privacy is critical for your use case and you're not comfortable with monitoring that `namespaced-openvpn` is working as expected, I can't recommend it yet.

To borrow a phrase from Stroustrup, `namespaced-openvpn` "protects against accident, not against fraud." It should be impossible for any normal application to have its traffic escape from the protected namespace back to the physical interface. However, without additional hardening, there is no guarantee that a malicious application can't force such an escape --- therefore, `namespaced-openvpn` should not be used by itself to "jail" an untrusted application.

## Wishlist

* Support for tunnelling IPv6 when the VPN provider supports it. Right now, `namespaced-openvpn` disables all IPv6 in the new namespace.
* Ideally, there would be an option to offer both limited protection to the root namespace (e.g., without protecting against route injection) and full protection to an isolated namespace. This seems difficult to achieve in a nondisruptive way.