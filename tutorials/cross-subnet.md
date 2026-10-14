# Building a Multi-Subnet Containerlab: A Step-by-Step Routing & Switching Guide

This tutorial guides you through setting up a simple multi-subnet network topology using **Containerlab**, **Linux Bridges (Alpine)**, and an **FRRouting (FRR) Router**.

We will start with a baseline topology file where cross-subnet connectivity is **intentionally absent**, then incrementally break down the required configurations at the **Router**, **Switch**, and **Host Node** levels to make inter-subnet routing work properly.

## 1. Initial Topology (`topology.clab.yml`)

Save the following YAML file as `topology.clab.yml`. In this initial state:

* IP addresses are assigned to the host interfaces.

* Switches bridge local interfaces.

* **Cross-subnet connectivity is broken** because routing and default gateways are missing or incomplete.

```yaml
name: multi-subnet-lab

topology:
  nodes:
    # --- Router ---
    router:
      kind: linux
      image: frrouting/frr:latest
      labels:
        graph-icon: router
        graph-level: 1
      exec:
        - ip link set eth1 up
        - ip link set eth2 up

    # --- Switches (Alpine Containers with Linux Bridge) ---
    switch-a:
      kind: linux
      image: alpine:latest
      labels:
        graph-icon: switch
        graph-level: 2
      exec:
        - ip link set eth1 up
        - ip link set eth2 up
        - ip link add br0 type bridge
        - ip link set eth1 master br0
        - ip link set eth2 master br0
        - ip link set br0 up

    switch-b:
      kind: linux
      image: alpine:latest
      labels:
        graph-icon: switch
        graph-level: 2
      exec:
        - ip link set eth1 up
        - ip link set eth2 up
        - ip link add br0 type bridge
        - ip link set eth1 master br0
        - ip link set eth2 master br0
        - ip link set br0 up

    # --- Subnet A Host ---
    host-a:
      kind: linux
      image: alpine:latest
      labels:
        graph-icon: host
        graph-level: 3
      exec:
        - ip link set eth1 up
        - ip addr add 192.168.10.2/24 dev eth1

    # --- Subnet B Host ---
    host-b:
      kind: linux
      image: alpine:latest
      labels:
        graph-icon: host
        graph-level: 3
      exec:
        - ip link set eth1 up
        - ip addr add 192.168.20.2/24 dev eth1

  links:
    # Subnet A (192.168.10.0/24)
    - endpoints: ["host-a:eth1", "switch-a:eth1"]
    - endpoints: ["router:eth1", "switch-a:eth2"]

    # Subnet B (192.168.20.0/24)
    - endpoints: ["host-b:eth1", "switch-b:eth1"]
    - endpoints: ["router:eth2", "switch-b:eth2"]
```

To deploy this initial lab:

```bash
sudo containerlab deploy -t topology.clab.yml
```

## 1. Breakdown of Exec Commands in the Initial Topology

In the initial `topology.clab.yml` file, the `exec` blocks configure the network interfaces and Linux bridges inside containers at startup. Below is an analysis of what each command does and how it translates directly into physical networking equipment.

### A. Router Exec Commands
```yaml
exec:
  - ip link set eth1 up
  - ip link set eth2 up
```
* **What it does virtually:** The `ip link set dev up` command changes the operational state of network interface cards (`eth1` and `eth2`) from `DOWN` to `UP`. In Linux, interfaces are created in a dormant, disabled state by default. Bringing them up allows the kernel to process incoming and outgoing physical/virtual signals on those ports.
* **Physical translation:** This is equivalent to plugging a patch cable into a physical router's interface port and ensuring the administrative status of the port is set to `no shutdown` within the router's CLI configuration.

### B. Switch Exec Commands (`switch-a` & `switch-b`)
```yaml
exec:
  - ip link set eth1 up
  - ip link set eth2 up
  - ip link add br0 type bridge
  - ip link set eth1 master br0
  - ip link set eth2 master br0
  - ip link set br0 up
```
* **What it does virtually:**
  1. `ip link set eth1 up` & `eth2 up`: Enables the individual port interfaces on the Alpine container.
  2. `ip link add br0 type bridge`: Creates a virtual software-based Layer 2 Ethernet bridge device named `br0`. This bridge acts just like a hardware switch backplane.
  3. `ip link set eth1 master br0` & `ip link set eth2 master br0`: Assigns (`enslaves`) interfaces `eth1` and `eth2` to the `br0` bridge. Any Ethernet frame arriving on `eth1` can now be forwarded out of `eth2` based on MAC address learning.
  4. `ip link set br0 up`: Activates the bridge interface so it can start processing traffic.
* **Physical translation:** This mirrors unboxing an unmanaged physical Layer 2 switch (e.g., a Cisco Catalyst or Netgear switch) and plugging cables into physical ports 1 and 2. All connected ports share the same broadcast domain and forward frames across the physical ASIC switching backplane.

### C. Host Exec Commands (`host-a` & `host-b`)
```yaml
# Host A
exec:
  - ip link set eth1 up
  - ip addr add 192.168.10.2/24 dev eth1

# Host B
exec:
  - ip link set eth1 up
  - ip addr add 192.168.20.2/24 dev eth1
```
* **What it does virtually:**
  1. Activates the host's network interface (`eth1`).
  2. Binds an IPv4 address (`192.168.10.2` with a 24-bit subnet mask, `/24`) to that interface. This informs the host's networking stack which network segment it belongs to (`192.168.10.0/24`).
* **Physical translation:** This is identical to opening the network configuration menu on a physical computer (like Windows Network Connections or Ubuntu Netplan) and statically assigning an IP address and subnet mask to its physical Ethernet network interface card (NIC).

## 2. Router Configuration for Cross-Subnet Connectivity

For a router to route packets between Subnet A (`192.168.10.0/24`) and Subnet B (`192.168.20.0/24`), two essential conditions must be met: gateway IP assignment and IP forwarding enabled in the kernel.

### 2.1 What is a Default Gateway?

When a host wants to communicate with another device, it first checks if the destination IP address resides on its own local subnet. 
* **If it is local:** The host uses Address Resolution Protocol (ARP) to discover the destination device's MAC address and sends the frame directly.
* **If it is remote (out-of-subnet):** The host cannot reach it directly. It must hand off the packet to an intermediary device that *does* have access to other networks. 

### Definition
A **Default Gateway** is the designated IP address on a local network subnet assigned to a router (or layer 3 switch). It acts as the "exit door" or checkpoint for all traffic destined outside of the local network segment. When a device does not have a specific routing table entry for a destination IP, it forwards the packet to its default gateway.

### 2.2 Gateway IP Assignment: `ip addr replace` / `ip addr add`

The router must have an IP address on each attached broadcast domain to act as the Default Gateway for nodes within that subnet.

Commands added to the `router` execution list:

```bash
- ip addr replace 192.168.10.1/24 dev eth1
- ip addr replace 192.168.20.1/24 dev eth2
```

#### Why use `ip addr replace` instead of `ip addr add`?

When deploying or re-executing scripts in container environments, `ip addr add` will throw an error if the IP address is already bound to the interface (e.g., `RTNETLINK answers: File exists`).

Using `ip addr replace` ensures **idempotency**:

* If the IP address does not exist on `eth1`, it will be created.

* If it already exists, it will overwrite/refresh the assignment cleanly without breaking lab deployment scripts.

### 2.3 IPv4 Forwarding and Cross-Subnet Communication

By default, standard operating systems (like Linux kernels on hosts) are configured as **end hosts**. If a Linux kernel receives a packet destined for an IP address assigned to a *different* machine, the kernel drops the packet immediately to save resources and enhance security.

#### How IPv4 Forwarding Works
When you enable IPv4 forwarding via `sysctl -w net.ipv4.ip_forward=1` on a router:
1. **Packet Reception:** The router receives an Ethernet frame on `eth1` from `host-a` (`192.168.10.2`). It unpacks the frame and inspects the inner Layer 3 packet header, seeing the destination IP is `192.168.20.2`.
2. **Lookup:** The router checks its internal routing table to determine where `192.168.20.2` lives.
3. **Decision & Rewriting:** The routing table indicates that `192.168.20.0/24` is directly connected to `eth2`. The router decrements the packet's Time-To-Live (TTL), strips off the old Layer 2 Ethernet header (which contained `host-a` and router `eth1` MAC addresses), and creates a brand-new Layer 2 header.
4. **Transmission:** The router transmits the packet out of `eth2` toward `host-b` via `switch-b`.

#### The Synergy: Gateways + Forwarding
* **The IP Gateway** provides the destination address on the local network that hosts look to when stepping outside their subnet.
* **IPv4 Forwarding** provides the router with the internal permission and capability to accept those packets on one interface, look them up, and push them out through another interface. Without forwarding, a router acts like a dead-end wall.

### 2.4 Enabling Kernel Forwarding: `sysctl -w net.ipv4.ip_forward=1`

By default, Linux network stacks act as **end hosts**—meaning if an IP packet arrives on `eth1` addressed to `192.168.20.2`, the Linux kernel drops it because the destination IP does not belong to any local interface on the system.

Command added to the `router` execution list:

```bash
- sysctl -w net.ipv4.ip_forward=1
```

#### What happens behind the scenes:

Setting `net.ipv4.ip_forward=1` changes the kernel behavior from an end host to an **IP router**:

1. When a packet for `192.168.20.2` enters `eth1`, the kernel consults its routing table (`ip route`).

2. It identifies `192.168.20.0/24` as a directly connected network on interface `eth2`.

3. It rewrites the Layer 2 Ethernet headers (MAC source/destination) and forwards the packet out through `eth2`.

## 3. Switch Configuration: Is Action Needed?

### Step-by-Step Overview on the Switches

In this topology, **no additional routing configuration is required on `switch-a` or `switch-b`**.

#### Why?

`switch-a` and `switch-b` operate strictly at **Layer 2 (Data Link Layer)** as transparent Ethernet bridges (`br0`).

1. **MAC Learning / Switching:** When `host-a` sends a frame destined for gateway `192.168.10.1`, the frame contains `host-a`'s MAC address as the source and `router`'s `eth1` MAC address as the destination.

2. **Transparent Forwarding:** `switch-a` looks up the destination MAC address in its Forwarding Database (FDB) and switches the frame from `eth1` to `eth2`.

3. **No IP Awareness:** Switches do not inspect IP headers or care whether traffic is cross-subnet or intra-subnet. Their only job is to transport Layer 2 Ethernet frames intact between the hosts and the router interfaces.

## 4. Host Node Configuration: Default Gateway Override

### Routing Table

When a router or host evaluates traffic, it consults its kernel routing table. Below is an example of an output command (`ip route`) executed on the routing container once multi-subnet routing is active:

```text
# ip route
default via 172.20.20.1 dev eth0 
172.20.20.0/24 dev eth0 scope link src 172.20.20.2 
192.168.10.0/24 dev eth1 proto kernel scope link src 192.168.10.1 
192.168.20.0/24 dev eth2 proto kernel scope link src 192.168.20.1 
```

#### Breakdown of the Routing Table Fields:
1. `192.168.10.0/24 dev eth1 proto kernel scope link src 192.168.10.1`: Direct route for Subnet A. Any packet destined for `192.168.10.x` is sent straight out interface `eth1` using the source IP `192.168.10.1`.
2. `192.168.20.0/24 dev eth2 proto kernel scope link src 192.168.20.1`: Direct route for Subnet B. Any packet destined for `192.168.20.x` is sent straight out interface `eth2` using the source IP `192.168.20.1`.

### Step-by-Step Overview on the Host Nodes

For `host-a` to reach `host-b`, `host-a` must know where to send traffic that falls outside its local subnet (`192.168.10.0/24`).

Command added to `host-a`:

```bash
- ip route replace default via 192.168.10.1 dev eth1
```

Command added to `host-b`:

```bash
- ip route replace default via 192.168.20.1 dev eth1
```

### Why `ip route replace default via ...` is Necessary in Containerlab

When Containerlab spawns container nodes, Docker/Containerlab automatically creates a management interface (**`eth0`**) and populates a default gateway targeting the Docker host's bridge network (e.g., `172.20.20.1`):

```text
$ ip route
default via 172.20.20.1 dev eth0
172.20.20.0/24 dev eth0 scope link src 172.20.20.7
192.168.10.0/24 dev eth1 scope link src 192.168.10.2
```

#### The Problem:

When you run `ping 192.168.20.2` from `host-a`:

1. `host-a` checks its routing table.

2. `192.168.20.2` is not in `192.168.10.0/24` (eth1).

3. `host-a` falls back to the existing **default route**, attempting to forward the packet out of **`eth0`** toward Docker's internal host bridge instead of through `eth1` toward your lab router!

#### The Solution:

Executing `ip route replace default via 192.168.10.1 dev eth1` overwrites Docker's default `eth0` gateway. It instructs the node kernel: *"Forward all out-of-subnet traffic to `192.168.10.1` via our lab-facing interface `eth1`."*

## 5. Verification & Troubleshooting

Redeploy your updated topology after applying the commands learned above:

```bash
sudo containerlab deploy -t topology.clab.yml --reconfigure
```

### 1. Check `host-a` Routing Table

```bash
docker exec -it clab-multi-subnet-lab-host-a ip route
```

*Expected Output:*

```text
default via 192.168.10.1 dev eth1
172.20.20.0/24 dev eth0 scope link src 172.20.20.7
192.168.10.0/24 dev eth1 scope link src 192.168.10.2
```

### 2. Test Cross-Subnet Ping

```bash
docker exec -it clab-multi-subnet-lab-host-a ping -c 4 192.168.20.2
```

*Expected Output:*

```text
PING 192.168.20.2 (192.168.20.2): 56 data bytes
64 bytes from 192.168.20.2: seq=0 ttl=63 time=0.142 ms
64 bytes from 192.168.20.2: seq=1 ttl=63 time=0.088 ms
--- 192.168.20.2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
```
