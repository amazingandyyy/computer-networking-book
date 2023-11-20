# network namespaces

Network namespaces are crucial in setting up container communication since they abstract a logical copy of the underlying host’s network stack. They allow isolation in containers since a container running in one namespace cannot directly communicate with a container in another namespace. All items in one namespace share routing table entities and network interfaces, allowing for modular communication.&#x20;

Creating a Linux namespace is quite simple. It is achieved using a command taking the form:

```bash
$ sudo ip netns add <namespace-name>
```

For instance, to create two namespaces, Red and Blue, the following commands can be used:

```bash
$ sudo ip netns add red
```

```bash
$ sudo ip netns add blue
```

To check each namespace’s interface, the command used takes a form similar to:

```yaml
$ sudo ip netns exec red ip link
```

or

```yaml
$ sudo ip -n red link
```

These commands only function within the `red` namespace, proving that namespaces actually provide isolation. To connect two namespaces, a ‘pipe’-virtual connection is established.

First, the interfaces of the two namespaces are created and connected using a command similar to:

```bash
$ sudo ip link add veth-red type veth peer name veth-blue
```

The interfaces are then attached to their respective namespaces using the command:

```bash
$ sudo ip link set veth-red netns red
```

and

```bash
$ sudo ip link set veth-blue netns blue
```

Each namespace is then assigned an IP address:

```bash
$ sudo ip -n red addr add 192.168.15.1/24 dev veth-red
```

and

```bash
$ sudo ip -n blue addr add 192.168.15.2/24 dev veth-blue
```

The links are then brought up using the commands:

```bash
$ sudo ip -n red link set veth-red up
```

and

```bash
$ sudo ip -n blue link set veth-blue up
```

The link is up, and this can be checked using `ping` or by accessing the ARP table on each host.

```bash
$ sudo ip netns exec red ping 192.168.15.2
```

A virtual network is created within the machine to enable communication between different namespaces in a host. This is achieved by creating a Virtual Switch on the host. There are many virtual switch options available, including OpenVSwitch and Linux Bridge, among others.&#x20;

A Linux bridge can be created using the `v-net-0` interface on the host:

```bash
$ sudo ip link add v-net-0 type bridge
```

The link can then be brought up using the command:

```bash
$ sudo ip link set dev v-net-0 up
```

This network acts as a switch for the namespaces and as an interface for the host. To connect the namespaces to the virtual network, existing connections between the namespaces must first be deleted using the command:

```bash
$ sudo ip -n red link del veth-red
```

New ‘Virtual Cables’ are then created to connect namespaces to the bridge network. For instance, to connect the red namespace:

```bash
$ sudo ip link add veth-red type veth peer name veth-red-br
```

For the blue namespace:

```bash
$ sudo ip link add veth-blue type veth peer name veth-blue-br
```

These act as links between the namespaces and the bridge network. To connect the red namespace interface to the virtual cable:

```bash
$ sudo ip link set veth-red netns red
```

And the other side is connected to the bridge network:

```bash
$ sudo ip link set veth-red-br master v-net-0
```

The same is repeated for the blue namespace:

```bash
$ sudo ip link set veth-blue netns blue
```

```bash
$ sudo ip link set veth-blue-br master v-net-0
```

The IP addresses are then set for the namespaces:

```bash
$ sudo ip -n red addr add 192.168.15.1/24 dev veth-red
```

```bash
$ sudo ip -n blue addr add 192.168.15.2/24 dev veth-blue
```

The links are then set up:

```bash
$ sudo ip -n red link set veth-red up
```

and

```bash
$ sudo ip -n blue link set veth-blue up
```

The namespace links have already been set up, but there is no connectivity between the host and the namespaces. The bridge acts as a switch between the different namespaces and is also the host machine’s network interface. To establish connectivity between namespaces and the host machine, the interface is assigned an IP address:

```bash
$ sudo ip addr add 192.168.15.5/24 dev v-net-0
```

This is an isolated virtual network, and it cannot connect a host to an external network. The only passage to the world is through the `eth0` interface on the host. Assuming a machine on the blue namespace intends to reach a host 192.168.1.3 on an external LAN whose IP address is 192.168.1.0, an entry is added to the machine’s routing table, providing an external gateway.

Since the host has an external-facing `eth0` port and an interface connecting with the namespaces, it can be used as a gateway. This is enabled using the command:

```bash
$ sudo ip netns exec blue ip route add 192.168.1.0/24 via 192.168.15.5
```

While this establishes a connection between the namespace and the external LAN, a `ping` request will still return a blank response. This is because the machine attempts to forward the message across two interfaces on the host machine. To enable the forwarding of messages between a namespace and the external LAN, a Network Address Translator (NAT) is enabled on the host. With NAT, the host sends the packets from the namespaces to external networks using its own name and IP address.&#x20;

To enable NAT on a machine, IP tables are created to use the POSTROUTING chain to MASQUERADE all packets coming from the internal network as its own. This is done using the command:&#x20;

```bash
$ sudo iptables -t nat -A POSTROUTING -s 192.168.15.0/24 -j MASQUERADE
```

If the LAN is connected to the internet and the namespace is also intended to connect to the internet, an entry is added specifying the host as the default gateway:

```bash
$ sudo ip netns exec blue ip route add default via 192.168.15.5
```

For an external host to reach a machine in the namespace, there are two options:

The private network can be exposed to the external host using **iproute**_**.**_ This is not the most preferred option as it introduces some security issues.

The second option is to use a port-forwarding rule, which states that any traffic coming in through a specific port on the host is directed to a specific port on the private network. This can be achieved using the command:

```bash
$ iptables -t nat -A PREROUTING --dport 80 --to-destination 192.168.15.2:80 -j DNAT
```

Machines in the namespace are now reachable from an external network through the host’s port 80 at the internal network’s port 80.



### Resources

{% embed url="https://kodekloud.com/blog/certified-kubernetes-administrator-exam-networking/#network-namespaces" %}
