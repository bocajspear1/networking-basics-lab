# Getting Set Up

All you need for this networking lab is a Linux box with any fairly recent distribution. Since we'll be running as root, you might want to create a VM to run this on. If you follow the instructions, you should have no issues, but you might as well be safe!

## Assumptions

- You are familiar with basic networking concepts, like IP addresses, network masks, and simple subnetting.
- You have some familiarity with Linux and the Linux command line.

## Networking Mindset

One of the biggest problems to solve in networking is the question "how do I get there?" Sometimes questions "how fast do I get there?" or "how do I get there securely?" are asked, but the essential issue is getting their in the first place. If you can't get there at all, no other questions matter.

When setting up and configuring networks, always keep that in mind. By default, computers don't know how to get anywhere, and you need to tell them how to get from point A to point B, or tell them somebody else that knows how to get from point A to point B. When troubleshooting a networking issues, first ask if the computer you are on knows how to get to that destination or not. This mindset will be helpful in lab ahead and your day-to-day networking problems.

## Some Configuration

We need to allow Linux to route traffic. This is done with:

```shell
$ sudo bash -c 'echo 1 > /proc/sys/net/ipv4/ip_forward'
$
```

> Note: This only hold until the system reboots. You'll need to run this every time the system reboots.

To verify Linux routing is enabled, run the following:

```shell
$ cat /proc/sys/net/ipv4/ip_forward
1
```

## Network Namespaces

The basis of this lab will using Linux network namespaces. Network namespaces are separate network stacks, essentially different sets of routing and address information. Linux usually uses these for things like containers, which allows them to network similar to a physical system. Here we use them to go over basics in networking and network troubleshooting, beyond just setting IP addresses.

## Our Topology

Using network namespaces, we'll create a simple network like this:

```shell
left <---> center <---> right
```

## Creating Our Namespaces

We will need 3 namespaces, which we'll call `left`, `center` and `right`. You will need root access to create namespaces, note the use of `sudo`. Use these commands:

```shell
sudo ip netns add left
sudo ip netns add center
sudo ip netns add right
```

You can verify their creation with the `ip netns list` command:

```shell
$ sudo ip netns list
right
center
left
```

Now we need to enter the namespaces. What I recommend is opening separate tabs in your terminal emulator (or separate tmux windows) for each namespace. You could Use the following command to run `bash` in a namespace:

```shell
sudo ip netns exec <NAMESPACE_NAME> bash
```

You should be given a root shell, but notice that if you run `ip addr` you have no interfaces:

```shell
# ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

Welcome to the namespace! We will need to add virtual interfaces to connect them together. These interfaces will effectively work like real network adapters.

To do this, enter the following commands **not in a namespace!**:

```shell
sudo ip link add r0 type veth peer name c0
sudo ip link add c1 type veth peer name l0
sudo ip link set l0 netns left
sudo ip link set r0 netns right
sudo ip link set c0 netns center
sudo ip link set c1 netns center
sudo ip netns exec center bash -c 'echo 1 > /proc/sys/net/ipv4/ip_forward'
```

If you run `ip addr` in your namespaces, you should now see more than one interface. Now you can move on to [part 2](2-The-Very-Basics)!
