# 2. The Very Basics

The next step is get the basic connectivity, so at least the new namespaces can connect to the ones next to it.

## Sidenote: Keeping Track of Namespaces

To differentiate between shells in different namespaces, you can change the prompt when you open a new shell with (replace `center` with the name of the namespace):
```
export PS1="(center)$PS1"
```

It's also recommended using multiple `tmux` windows or terminal tabs to make things easy to switch between.

## Bringing an Interface Up

Firstly, we need to bring the interface up. By default interfaces are added in the DOWN state, meaning they are off. You can see this by running `ip addr` an the interface will say `state DOWN`. (Note: ignore the value after the `@`, the inteface name if everything before the `@`)

```shell
root@test:/home/jacob# ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
5: c0@if6: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 16:8e:9d:07:22:d5 brd ff:ff:ff:ff:ff:ff link-netns right
8: c1@if7: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 3a:06:cb:1f:17:27 brd ff:ff:ff:ff:ff:ff link-netns left
```

To bring an interface up, use the following command (since the shells for the namespaces are root, I'm not including `sudo`):

```shell
ip link set <INTERFACE> up
```

If you check with `ip addr`, you'll see the output has changed:

```shell
7: l0@if8: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN group default qlen 1000
    link/ether 9e:97:5e:57:83:54 brd ff:ff:ff:ff:ff:ff link-netns center
```

The `state` will be `LOWERLAYERDOWN` until the other end of the connection is brought up, then it will be `state UP`:

```shell
7: l0@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 9e:97:5e:57:83:54 brd ff:ff:ff:ff:ff:ff link-netns center
```

Once all interfaces (`l0`, `c0`, `c1`, `r0`) are in the `UP` state, move onto the next section.

> Note: Most Linux distributions will bring up their interfaces by default. However, if new interfaces are added (new NIC, for example), they may not.

## Setting IP Addresses

We want to set up the network with these ranges:

```shell
    192.168.70.0/24        172.17.40.0/24
left <------------> center <------------> right
   .2            .1       .1            .2
```

To set an IP address on an interface (since the shells for the namespaces are as "root", I'm not including `sudo`):

```shell
ip addr add <IP>/<PREFIX> dev <INTERFACE> 
```

For `left`, the interface is `l0` (verify with `ip addr` in the `left` namespace), and the IP and prefix we want is `192.168.70.2` and `24`. To set the IP then is:

```shell
ip addr add 192.168.70.2/24 dev l0
```

Before you move on, make sure the following IPs are configured on the interfaces in the namespace:

| Namespace | Interface   | Address           | Prefix  |
| --------- | ----------- | ----------------- | ------- |
| left      | `l0`        | `192.168.70.2`    | 24      |
| center    | `c1`        | `192.168.70.1`    | 24      |
| center    | `c0`        | `172.17.40.1`     | 24      |
| right     | `r0`        | `172.17.40.2`     | 24      |

## Verifying Connectivity

We can verify we can connect to our neighbor namespaces by pinging them with the `ping` command. So on `left` we can verify we can connect to `center` by pinging the IP on the interface connected to `left` (use ctrl-c to stop `ping`, otherwise it will go forever):

```shell
(left)$ ping 192.168.70.1
PING 192.168.70.1 (192.168.70.1) 56(84) bytes of data.
64 bytes from 192.168.70.1: icmp_seq=1 ttl=64 time=0.080 ms
64 bytes from 192.168.70.1: icmp_seq=2 ttl=64 time=0.081 ms
64 bytes from 192.168.70.1: icmp_seq=3 ttl=64 time=0.085 ms
```

On `center` we can ping both the addresses on `left` and `right`:

```shell
(center)$ ping 192.168.70.2
PING 192.168.70.2 (192.168.70.2) 56(84) bytes of data.
64 bytes from 192.168.70.2: icmp_seq=1 ttl=64 time=0.071 ms
64 bytes from 192.168.70.2: icmp_seq=2 ttl=64 time=0.057 ms
^C
--- 192.168.70.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1003ms
rtt min/avg/max/mdev = 0.057/0.064/0.071/0.007 ms

(center)$ ping 172.17.40.2
PING 172.17.40.2 (172.17.40.2) 56(84) bytes of data.
64 bytes from 172.17.40.2: icmp_seq=1 ttl=64 time=0.060 ms
64 bytes from 172.17.40.2: icmp_seq=2 ttl=64 time=0.080 ms
64 bytes from 172.17.40.2: icmp_seq=3 ttl=64 time=0.036 ms
^C
--- 172.17.40.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2044ms
rtt min/avg/max/mdev = 0.036/0.058/0.080/0.018 ms
```

And `right` can ping `center`:

```shell
(right)$ ping 172.17.40.1
PING 172.17.40.1 (172.17.40.1) 56(84) bytes of data.
64 bytes from 172.17.40.1: icmp_seq=1 ttl=64 time=0.072 ms
64 bytes from 172.17.40.1: icmp_seq=2 ttl=64 time=0.082 ms
```

However, `left` cannot ping `right`:

```shell
(left)$ ping 172.17.40.2
ping: connect: Network is unreachable
```

Remember our question: "how do I get there?" `left` can ping `center` because they are directly connected. `left` knows that when we ping `192.168.70.1`, its connected to that network and so it can get there ("route" it there). A computer knows how to route to networks directly connected to it. Past those direct connections though, it doesn't know how to get there. Thus, when we try to ping `right`, it gives up since it doesn't know what to do with it. We need to tell it what and where to route packets outside its directly connected network. That's the goal of [part 3](3-Simple-Routing)!
