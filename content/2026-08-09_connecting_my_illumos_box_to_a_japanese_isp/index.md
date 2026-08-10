+++
title = "Connecting my illumos box to a Japanese ISP"
date = 2026-08-09
+++

I moved an Ethernet cable from my router to an OmniOS box that I call `fridge`. The other end was connected to the modem provided for my Docomo Hikari line. This was meant to be the last test of a Rust networking project I had worked on during weekends since the end of March. I had written a daemon that could manage a native DS-Lite tunnel on Linux and illumos. If it went well, the box would connect directly to the ISP and provide IPv4 connectivity through that tunnel.

The modem lights looked normal. OmniOS received router advertisements and installed a default IPv6 route on `ixgbe1`. It did not, however, receive an address that could be used to reach the Internet.

```
$ ipadm show-addr
ADDROBJ           TYPE     STATE        ADDR
lo0/v4            static   ok           127.0.0.1/8
ixgbe1/v6         addrconf ok           fe80::ae1f:6bff:fe6c:5ca7%ixgbe1/10

$ netstat -nr -f inet6
Destination/Mask            Gateway                   Flags  If
default                     fe80::be4a:56ff:fe1c:3c10 UG     ixgbe1
```

There was a route, but only a link local address to use as its source. Restarting the modem and recreating the IPv6 interface changed nothing. Four months earlier I would probably have assumed that the tunnel was the missing part. By this point I had also implemented the provisioning protocol used by my ISP, an SMF service, and an OmniOS package. The cable test exposed a more basic gap.

I started the project because I wanted to learn more about the illumos network stack. I had also been looking for a Rust project that would let me work directly with operating system interfaces. An article by Ryan Goodfellow about [the life of an IPv6 address on illumos](https://ry.goodwu.net/tinkering/a-day-in-the-life-of-an-ipv6-address-on-illumos/) had stayed in my mind. It follows one address through `ipadm`, libraries, doors, STREAMS, and the kernel. I wanted a problem that would make me visit some of the same layers.

There was also a practical goal. `fridge` already runs services on my home network. If it could terminate the ISP connection as well, I could use it as the entry point to that network.

Calling it my ISP connection hides some Japanese telecom layering. The physical access runs over NTT's FLET'S network. My retail contract for that line is Docomo Hikari, while ASAHI Net provides Internet connectivity on top of it using [native IPv6 over IPoE and IPv4 over DS-Lite](https://asahi-net.jp/en/service/option/ipv6/). All three names therefore describe parts of the same connection, but they do not play the same role.

DS-Lite exists because an ISP can provide native IPv6 without assigning a public IPv4 address to every customer. The customer side component is called a B4. It takes an IPv4 packet and places it inside an IPv6 packet addressed to an ISP endpoint called the AFTR. The AFTR removes the outer packet and performs IPv4 network address translation before sending the traffic to the Internet. [RFC 6333](https://www.rfc-editor.org/rfc/rfc6333) describes the architecture.

Applications on the customer network still see ordinary IPv4. Their packets reach the B4, which uses the reserved `192.0.0.0/29` range for the point to point side of the tunnel. The outer IPv6 packet travels over the access network to the AFTR. Replies follow the same path in reverse. Once the tunnel exists, the daemon is no longer involved in moving each packet. Encapsulation and forwarding remain inside the operating system network stack.

![The DS-Lite packet path](dslite-path.svg)

The B4 does not need to implement packet forwarding itself. Both Linux and illumos already know how to create an IPv4-in-IPv6 tunnel. The first useful experiment was therefore small. I created an `iptun` link on OmniOS, configured a matching Linux host on the same network as a temporary AFTR, assigned the reserved DS-Lite IPv4 endpoints, and installed a route. IPv4 pings crossed the tunnel in both directions.

That result seemed to remove the largest risk because the kernel data path worked. The remaining task looked like turning a few administrative commands into a daemon that could discover the two IPv6 endpoints and keep the tunnel in the desired state. I treated the tunnel as the uncertain part and ISP provisioning as information that some existing service would eventually hand to it. Building the tunnel daemon followed that plan, while finding a reliable source for the ISP information became the rest of the project.

I did not want the program to run `dladm`, `ipadm`, and `route` as subprocesses. Calling the underlying interfaces would teach me more, avoid parsing command output, and make it possible to observe and reconcile state through the same APIs. The Linux backend used netlink. The illumos backend used `libdladm` to create the tunnel link, `libipadm` to create its IP interface, socket ioctls to configure its addresses, and a `PF_ROUTE` socket for the IPv4 default route.

This choice made the Rust side more interesting than a sequence of command invocations. Some illumos interfaces are regular C library calls, while others exchange structures whose layout comes directly from system headers. Each foreign call required deciding which values could be represented safely in Rust, how long pointers remained valid, and which resources needed cleanup after a partial failure. The daemon runs as root, so an incorrect structure was not just an inconvenient type error. It could ask the kernel to change the wrong network state.

The division between `libipadm` and the direct ioctls was not part of the original design. It came from the debugging detour in the project.

## Four bytes between two processes

The 64 bit Rust daemon could create an `iptun` link and plumb its IP interface, but assigning the point to point IPv4 address failed with `IPADM_NOTSUP`. The equivalent `ipadm create-addr` command worked. After the failed call, the interface remained present without an address because `libipadm` had rolled back the kernel change.

I started by tracing system and library calls with [`truss`](https://man.omnios.org/man1/truss), followed the error through the illumos source, and then used [DTrace](https://man.omnios.org/man8/dtrace) to narrow it to an `nvlist_unpack` call inside `ipmgmtd`. The packed nvlist looked valid in the client, although the service read it four bytes later than I expected.

At first I suspected that I had translated one of the C structures incorrectly. That would have been a familiar FFI mistake, but comparing the buffers did not support it. The client passed an nvlist beginning with a valid encoding header, while the service saw four zero bytes before that header. The data itself had survived the door call, but the two processes disagreed about where it began.

The clue that changed the investigation was easy to overlook. After looking at the service with [`mdb`](https://man.omnios.org/man1/mdb), searching the Internet, and asking an LLM for help, I finally noticed 32 bit register names such as `%ebp` rather than `%rbp`. The Rust program was a 64 bit process using `/lib/amd64/libipadm.so.1`, while `/lib/inet/ipmgmtd` was a 32 bit process. They communicate through an [illumos door](https://man.omnios.org/man3c/door_call), a local procedure call mechanism.

One of the [private door request structures](https://github.com/illumos/illumos-gate/blob/0764e87f4a667f36d63262fcdd690064929acc48/usr/src/lib/libipadm/common/ipadm_ipmgmt.h#L235-L241) contains a `size_t` field followed by the nvlist. Its layout depends on the architecture.

```
32 bit ipmgmtd             64 bit libipadm caller

0   command                0   command
4   flags                  4   flags
8   nvlist size, 4 bytes   8   nvlist size, 8 bytes
12  nvlist                 16  nvlist
```

The service expected the nvlist at offset 12, while the client placed it at offset 16. As a result, the first four bytes passed to the unpacker came from the upper half of the size field. The unpacker rejected them, `ipmgmtd` returned an error, and `libipadm` translated it to status 36.

The failure only appeared in my process because the shipped `ipadm` command is also 32 bit. Its structure matches the 32 bit service, so the same private protocol works when used by the system tool. A 64 bit C program would have encountered the same problem. With that knowledge, I found [illumos issue 17851](https://www.illumos.org/issues/17851), which already described the same `ipadm_create_addr()` failure and proposed replacing `size_t` with a fixed width `uint32_t`. The source even contains [a comment saying that door structures must have the same size on amd64 and i386](https://github.com/illumos/illumos-gate/blob/0764e87f4a667f36d63262fcdd690064929acc48/usr/src/lib/libipadm/common/ipadm_ipmgmt.h#L168-L171). This particular structure does not follow that rule.

I kept the working parts of `libipadm`, but bypassed address creation with the same `SIOCSLIFADDR` family of ioctls used below the library. The tunnel became functional, although `ipadm show-addr` displays the address as `dslite0/?` because it was not registered with `ipmgmtd`. Later I found that Oxide's [netadm-sys](https://github.com/oxidecomputer/netadm-sys) had dealt with the same boundary by defining the door field with a fixed width. I may eventually follow the same path to restore the missing `ipadm` object registration.

Another part of the daemon needed the local IPv6 address that the kernel would use to reach the AFTR. On Linux, `ip route get` returns a preferred source address, and my first implementation read the corresponding netlink attribute. illumos also has a route lookup through `PF_ROUTE`, so I expected to write a second version of the same idea. For the address field, however, the kernel [selects the first logical IP interface on the link](https://github.com/illumos/illumos-gate/blob/0764e87f4a667f36d63262fcdd690064929acc48/usr/src/uts/common/inet/ip/ip_rts.c#L1282-L1291). On a normal IPv6 interface that is the link local address, not necessarily the source chosen for a remote destination. `route -nv get` confirmed the source code on the running system.

The replacement was a connected UDP socket. Binding it to the unspecified IPv6 address and connecting it to the AFTR makes the kernel perform its usual route and source selection. UDP `connect()` does not need to send a packet. `getsockname()` then returns the selected local address. The same three standard socket calls worked on both systems, so I deleted the tested Linux netlink implementation.

By July the daemon could create, inspect, update, and remove the tunnel. Creating it once was different from running it as a service. The local IPv6 endpoint could disappear or change when the upstream address changed. The remote endpoint came from DNS or provisioning and had its own refresh lifecycle. A crash could leave behind some combination of the tunnel link, IP interface, address, and route. Running a fixed sequence of setup commands at startup would either destroy working state or fail when the system no longer matched the expected order.

I built the daemon around a reconcile loop instead. On every wake it recalculated the desired endpoints, inspected the actual kernel state, and selected the smallest action needed to make them agree. It could keep a correct tunnel, bring up one that was down, update properties that could change in place, or rebuild it when the endpoints changed. Linux netlink and illumos `PF_ROUTE` events only woke the loop. They were hints that something might have changed, while the kernel remained the source of truth. This also allowed a restarted daemon to adopt an existing tunnel without interrupting traffic.

## Where the AFTR came from

None of that answered one remaining question. How would it learn the address of the real AFTR?

[RFC 6334](https://www.rfc-editor.org/rfc/rfc6334) defines DHCPv6 option 64 for this purpose. The option contains an AFTR name that the B4 resolves through DNS. I originally considered adding a small DHCPv6 client to the daemon, but a host already has a client bound to UDP port 546. Two clients cannot safely divide ownership of leases and replies on that port. It made more sense to let the system client keep that responsibility and pass the result to `dslite-b4` through its `set-aftr` command.

Well, that simple boundary became less portable each time I tested it. ISC `dhclient` can request option 64 and run an exit hook with the result. `systemd-networkd` can request the option but does not make the returned AFTR name available to another program. NetworkManager can expose received options and notify a dispatcher, but its configuration does not add option 64 to the request list. On illumos I could define option 64 in [`inittab6`](https://man.omnios.org/man5/dhcp_inittab.5), the system's DHCP option table, so that `dhcpagent` could request and decode it. It did not, however, provide a dependable notification when the value changed and could restore a stale value after a later reply omitted it. These experiments did not prove that RFC 6334 was unusable everywhere. They showed that decoding a DHCP option is only one part of an integration. The client also has to request it, expose it, and report its removal.

For better or worse, my access line did not return option 64 anyway. ASAHI Net's DS-Lite service, called v6connect, uses a Japanese provisioning protocol called [HB46PP](https://github.com/v6pc/v6mig-prov/blob/9020a1bd5f2f8f83712b1180db70af7dc0dad638/spec.md). A client first asks its DNS resolver for a TXT record at `4over6.info`. On the provider network, that name returns the URL and trust mode of a provisioning service. An HTTPS request then returns the available IPv4 over IPv6 mechanism and its endpoint.

The DNS part is intentional. The same name can return different provisioning information depending on the access network from which it is queried. That lets a common client discover the service selected for its line without shipping a list of Japanese providers. It also means that native IPv6 and the correct resolver must already work before HB46PP can discover anything.

I implemented HB46PP as a separate [Rust crate](https://crates.io/crates/hb46pp) because the protocol can provision [mechanisms](https://docs.rs/hb46pp/latest/hb46pp/enum.Capability.html) other than DS-Lite. Implementing it involved more than parsing DNS and JSON. The client had to apply the trust mode from the DNS bootstrap, handle redirects without leaking provisioning tokens to another origin, and follow refresh deadlines and retry instructions. From behind my existing router it discovered `dslite.v6connect.net`, and the daemon created a tunnel to the resolved AFTR. The daemon status and a kernel route lookup then showed the expected path:

```
$ dslite-b4 status
Desired: resolved
AFTR: dslite.v6connect.net (hb46pp)
Local IPv6: [redacted]
Remote IPv6: 2001:c28:1:301::11
Last action: create

$ ipadm show-addr | grep dslite0
dslite0/?         static   ok           192.0.0.2->192.0.0.1

$ route -n get 1.1.1.1
   route to: 1.1.1.1
destination: default
       mask: default
    gateway: 192.0.0.1
  interface: dslite0
      flags: <UP,GATEWAY,DONE,STATIC>
```

At that stage my own router was still between `fridge` and the bridged ISP modem, providing IPv6 and DNS. That was enough to test the tunnel, but not the whole path from the modem. The final direct connection test would show whether stock illumos could obtain IPv6 and DNS on this access line.

## What the ISP delegated

This brings the story back to `ixgbe1` and its link local address. A packet capture showed router advertisements with the Managed and Other Configuration flags, but no global prefix that the host could use for address autoconfiguration. `dhcpagent` responded by sending a DHCPv6 Solicit containing IA_NA, which requests one non temporary address. No Advertise or Reply arrived.

I first wondered whether the ISP had remembered the router's MAC address, or whether I had left part of the illumos interface configuration behind. The packet capture kept showing the same unanswered request after the modem restart and interface rebuild, ruling out the simpler versions of those explanations.

The ISP had previously delegated a `/56` to my router. That suggested a different DHCPv6 request. illumos `dhcpagent` does not implement prefix delegation, so I built ISC DHCP 4.4.3-P1 in a temporary directory on OmniOS. ISC DHCP is no longer maintained, which made it suitable for an experiment but not a permanent answer:

```
$ dhclient -6 -P --prefix-len-hint 56 ixgbe1

dhclient  -> Solicit   IA_PD /56
ISP       <- Advertise IA_PD /56
dhclient  -> Request   IA_PD /56
ISP       <- Reply     IA_PD /56
```

The server answered immediately with a delegated `/56` and normal lease lifetimes. DS-Lite itself does not require prefix delegation, but this access service expects the connected router to request one. I appreciate that model because it lets me divide the allocation among my own networks. `IA_PD`, however, delegates a routed prefix rather than configuring an address on `ixgbe1`. The client must decide how to use the prefix, advertise its subnets, and maintain the lease. illumos `dhcpagent` could request a host address through `IA_NA`, but it was not prepared to take on that router role.

For the experiment I selected one `/128` from the delegated prefix and added it temporarily to `ixgbe1`. IPv6 pings and HTTPS requests then worked, which confirmed that the prefix was routed to the host.

The DHCP Reply also included the addresses of two DNS resolvers for the access network. Once I configured one address from the delegated prefix and those resolvers, HB46PP found the AFTR and the daemon brought up `dslite0`. IPv4 ICMP and HTTPS worked through the native illumos tunnel with my router no longer in the path. This confirmed that both HB46PP and the tunnel implementation worked against the real service.

What I had configured by hand was now the missing part. Making it permanent would require a DHCPv6 client that acquired, renewed, and rebound the delegation, divided the prefix among local networks, configured addresses and DNS, and withdrew that configuration when the lease expired. Because it would have to coexist with or replace `dhcpagent`, this was another system network service rather than a small adapter around `dslite-b4`.

Earlier in the project I had decided that the daemon should not become a second DHCP client. That decision still seems right. Owning the B4 tunnel is a narrow responsibility. Competing with the operating system client on port 546, while each process believes it controls addresses and DNS, would leave neither process with a reliable view of the lease. The alternative would be a new component that replaces the relevant part of `dhcpagent`, which is another project rather than the final milestone of this one.

The [repository](https://github.com/michalskalski/ipv4-over-ipv6) now contains the result of that work, including Linux and illumos backends, a reconcile loop, and service definitions. I also published two Rust crates, [dslite-b4](https://crates.io/crates/dslite-b4) and [hb46pp](https://crates.io/crates/hb46pp). The daemon can manage DS-Lite when the host already has working IPv6 provisioning. It cannot turn stock illumos into a direct router for this Asahi Net line.

I had wanted the project to end with `fridge` at the edge of my home network. Instead, it ended at a boundary in the network stack that I do not yet want the daemon to cross. For now that feels like a reasonable place to stop, write down what I learned, and think about what a future project would need to own.
