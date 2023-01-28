# 4. iptables

`iptables` is a key component to Linux networking and network security. While known primarily as the basis for Linux-based firewalls (`ufw`, `firewalld` and such usually use `iptables` under the hood), `iptables` is also the primary method packet modification and NAT. While `iptables`'s syntax can be imposing, learning it is important to fully utilize the networking functionality in Linux.

## Tables and Chains

Sources:

- [Linux Firewall Tutorial: IPTables Tables, Chains, Rules Fundamentals](https://www.thegeekstuff.com/2011/01/iptables-fundamentals/)

`iptables` operates 3 default "tables", each with a different set of "chains." Chains contain rules that perform actions on the network traffic going in, out, and internally on a system. These tables and chains set the order and type of operations in `iptables`. The two main tables you will work with are `filter` and `nat` (the third, `mangle` is mostly used to more low-level packet modifications). The `filter` table is the default table, and contains the chains `INPUT`, `OUTPUT`, and `FORWARD`. The role of the chains are as follows:

- `INPUT`: Rules in this chain are applied to packets coming into an interface (even internal ones!)
- `OUTPUT`: Rules in this chain are applied to packets coming going out an interface (even internal ones!)
- `FORWARD` Rules in this chain are applied to packets going between internal interfaces

For the `nat` (short for "Network Address Translation") table, the chains are `PREROUTING`, `POSTROUTING`, `INPUT`, and `OUTPUT`. The role of the chains are as follows:

- `PREROUTING`: NAT rules performed before packets are routed by the system
- `INPUT`: Ignore this
- `OUTPUT`: NAT rules for packets created by the local system
- `POSTROUTING` NAT rules performed after packets are routed by the system

## `iptables` Basics

To view whats currently in a table's chains, use the `-vL` command for the most information. To select which tables to view, use the option `-t <TABLE>`. For the default `filter` table you don't need to set this option.

To view the default table's chains and rules:

```shell
$ sudo iptables -vL
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
```

To view the `nat` table's chains and rules:

```shell
$ sudo iptables -t nat -vL
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination  
```

Note that by default, unless you're running `firewalld` or `ufw`, there will be no rules in the chains. If there were rules, they'd be processed top to bottom. The first rule that matches is what happens to the packets.

## Table Default Action

you can set the default action of a chain with `iptables -P <CHAIN> <ACTION>`. By default the action is `ACCEPT`, which accepts all traffic. You might want to set the default action to drop inbound traffic, so you can set this with:

```shell
iptables -P INPUT DROP
```

> Be careful when running default deny! If you're SSHed to the system and have no other rules, your connection will be dropped and you will lose access!

## Blocking and Allowing Traffic

To block traffic, you'll use the `filter` chain. While you can add rules to any of the chains, the one you'll use the most is `INPUT`. There are few options that will be commonly used, which will be listed here. (for more details, you can usually use `man iptables` to bring the manual for the command)

- `-I <CHAIN> <LOC>`: Insert the rule into the chain `<CHAIN>` at location `<LOC>`. Rules in a chain are numbered starting with 1 increasing as you go down the list. Many times `<LOC>` will be 1 to insert the rule at the beginning, since rules are processed top to bottom. Cannot be used when `-A` or `-D` is used.
- `-A <CHAIN>`: Append the rule the chain `<CHAIN>` at the end. Cannot be used when `-I` or `-D` is used.
- `-D <CHAIN>`: Remove the rule matching the other options give. Cannot be used when `-I` or `-A` is used.
- `-p`: Matches the protocol, usually `tcp`, `udp`, or `icmp`.
- `--dport`: Matches the destination port (the port the packet is going to), used for `-p tcp` or `-p udp`.
- `-s`: Matches the source IP address
- `-d`: Matches the destination IP address
- `-j`: What to do with the packet that matches the rule. Some of option values and effects are as follows:
  - `ACCEPT`: Accept the packet and keep it going
  - `DROP`: Drop the packet, remove it, and don't do anything else.

For example, to allow traffic to TCP port 80 on the INPUT chain (so for inbound network traffic):

```shell
iptables -I INPUT -p tcp --dport 80 -j ACCEPT
```

## Stateful Connections

`iptables` supports stateful processing. This means that `iptables` tracks outbound connections and can use that information when it received data. Usually, this is used to allow traffic returning from an outbound connection. To do this, run the following commmand:

```shell
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT 
```

## Experimenting with Blocking and Accepting

First, on `right`, set the default to block:

```shell
(right)$ iptables -P INPUT DROP
```

Then allow TCP port 80 (this port is normally used for web servers using HTTP):

```shell
(right)$ iptables -A INPUT -p tcp --dport 80 -j ACCEPT
```

Then, use netcat to create a listener on port 80. Leave this running:

```shell
(right)$ nc -lvp 80
nc: getnameinfo: Temporary failure in name resolution
...
```

Ignore the "Temporary failure in name resolution" error.

On `left`, note that we cannot ping `right` anymore, since we are default denying all inbound packets to `right`:

```shell
(left)$ ping 172.17.40.2
PING 172.17.40.2 (172.17.40.2) 56(84) bytes of data.
^C
--- 172.17.40.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1030ms
```

However, if we use netcat, we can connect to port 80 on `right` since we opened that port up:

```shell
(left)$ nc -v 172.17.40.2 80
Connection to 172.17.40.2 80 port [tcp/http] succeeded!
```

Anything you type in the prompt now will be sent to `right`, which will print it out to the terminal.

Let's say that we want to block `left` from access port 80 now (they were send mean messages perhaps...). We want to insert a rule in the beginning so its processed first. First stop the netcat listener with ctrl-c, then run:

```shell
(right)$ iptables -I INPUT 1 -s 192.168.70.2 -p tcp --dport 80 -j DROP
```

Restart the netcat listener, then try to connect from `left` again:

```shell
(left)$ nc -v 172.17.40.2 80
...
```

Note it hangs, and have to use ctrl-c to stop it. However, if we try from `center`, it works:

```shell
(center)$ nc -v 172.17.40.2 80
Connection to 172.17.40.2 80 port [tcp/http] succeeded!
^C
```

Start a listener on `center` and try to connect from `right`. Note we can't connect. Since we blocked all inbound traffic, and only allow if the destination port is 80, traffic returning to `right` is blocked. We need to use `iptables` stateful functionality to fix this:

```shell
(right)$ iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT 
(right)$ nc -v 172.17.40.1 80
Connection to 172.17.40.1 80 port [tcp/http] succeeded!
```

## Masquerade

One more important functionality I'll do over is masquerade, also known as source NAT. Using masquerade, the system can make traffic passing through it to look like that system started the connection. This is the most common form of NAT, converting un-routable private addresses into public IPs on home and corporate routers. This is also handy from a routing perspective. Since connections going out of a masquerading system look like they came from that system, packets coming back that can route to the masquerading system can respond correctly (they know the way to at least the masquerading system). The masquerading system will transparently process the returning packets so they are routed correctly to the original sender.

With iptables, you enable masquerade on the `nat` table with the following command:

```shell
iptables -t nat -A POSTROUTING -o <OUTBOUND_IFACE> -j MASQUERADE
```

## Masquerade in Action

Let's try a third way of routing from `left` to `right`.

Remove the static route we made earlier:

```shell
(right)$ ip route del 192.168.70.0/24 via 172.17.40.1
(right)$ 
```

Run the listener again on `right`:

```shell
(right)$ nc -lvp 80
nc: getnameinfo: Temporary failure in name resolution
...
```

On `left`, not we cannot connect to `right`'s port 80 anymore:

```shell
(left)$ nc -v 172.17.40.2 80
...
```

On `center` enable maquerading going out the interface to `right`:

```shell
(center)$ iptables -t nat -A POSTROUTING -o c0 -j MASQUERADE
(center)$ 
```

Note we can connect again:

```shell
(left)$ nc -v 172.17.40.2 80
Connection to 172.17.40.2 80 port [tcp/http] succeeded!
```

If we use a tool like `tcpdump`, we can see the connect appears to come from `center` (`172.17.40.1`):

```shell
(right)$ tcpdump -n -i r0 tcp port 80
...
14:03:38.891705 IP 172.17.40.1.44176 > 172.17.40.2.80: Flags [F.], seq 3556319863, ack 653292295, win 502, options [nop,nop,TS val 461699208 ecr 525622577], length 0
14:03:38.893562 IP 172.17.40.2.80 > 172.17.40.1.44176: Flags [.], ack 1, win 510, options [nop,nop,TS val 525690667 ecr 461699208], length 0
```

Since `right` knows how to get to `center`, it routes it back to `center` who then passes it back to `left` transparently.

## Advanced `iptables`

`iptables` can do a lot more that I won't get into here right now:

- Port forwarding: Take an inbound connection and pass it to another system or port transparently
- Destination NAT: Take an connection to an IP and transparently forward it to another system
- Drop or accept ICMP packets based on ICMP type
