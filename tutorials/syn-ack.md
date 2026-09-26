# Tutorial: Capturing the TCP Three-Way Handshake in Containerlab

This tutorial guides you through setting up a simple Containerlab network topology connected via a Linux bridge interface (`br-lan`), triggering a TCP connection between two containers, and using Wireshark filters to observe the classic **TCP Three-Way Handshake** (`SYN` $\rightarrow$ `SYN-ACK` $\rightarrow$ `ACK`).

---

## 1. Prerequisites

Before starting, ensure you have installed:
* **Containerlab**: [Installation Guide](https://containerlab.dev/install/?utm_source=gemini)
* **Docker**: Required as the container engine for Containerlab nodes.
* **Wireshark & tshark**: Installed on your Linux host/WSL instance.

---

## 2. Setting Up the Bridge Switch

Using a Linux bridge switch makes capturing packets in Wireshark straightforward because all traffic across the local network segment passes through a single host interface (`br-lan`).

Run the following commands on your host machine to create and activate the bridge interface:

```bash
# Create the bridge interface
sudo ip link add br-lan type bridge

# Bring the interface UP
sudo ip link set br-lan up
```

---

## 3. Containerlab Topology Definition

Create a file named `syn-ack.clab.yml` with the following configuration. 

> **Important Note on Bridges:** Notice that `br-lan` is **not** listed under `nodes:`. Containerlab directly connects interfaces to pre-existing host bridges defined in the `links:` section to prevent host configuration drift errors.

```yaml
name: syn-ack

topology:
  nodes:
    client:
      kind: linux
      image: alpine:latest
      exec:
        - apk add curl
        - ip addr add 10.10.10.10/24 dev eth1
        - ip link set eth1 up

    web-server:
      kind: linux
      image: nginx:alpine
      exec:
        - ip addr add 10.10.10.20/24 dev eth1
        - ip link set eth1 up

  links:
    - endpoints: ["web-server:eth1", "br-lan:eth1"]
    - endpoints: ["client:eth1", "br-lan:eth2"]
```

### Network Topology Diagram

```
                 +--------------------------------+
                 |    Switch / Bridge (br-lan)    |
                 +---------------+----------------+
                                 |
         +-----------------------+-----------------------+
         |                                               |
  +------+-------+                               +-------+------+
  |  web-server  |                               |    client    |
  | 10.10.10.20  |                               | 10.10.10.10  |
  +--------------+                               +--------------+
```

---

## 4. Deploying the Lab

Deploy the Containerlab topology using the following command:

```bash
sudo containerlab deploy -t syn-ack.clab.yml
```

Verify that all containers are running and connected properly:

```bash
sudo containerlab inspect -t syn-ack.clab.yml
```

---

## 5. Packet Capture Setup (Recommended Method)

### **Recommended: Method A — Direct GUI Wireshark via WSLg / Native Linux**

Capturing directly on the `br-lan` bridge interface allows you to view all frames moving through the network switch in real time without entering container namespaces.

1. Launch Wireshark with root privileges from your host terminal:
   ```bash
   sudo wireshark &
   ```
2. In the Wireshark interface selection window, double-click **`br-lan`** as the capture interface.
3. Click the green fin icon to start live packet capture.

---

### Alternative: Method B — Terminal Capture with `tshark`

If you are running in a headless CLI environment without GUI access, run `tshark` directly on the bridge interface and write to a `.pcap` file:

```bash
sudo tshark -i br-lan -w /tmp/tcp_handshake.pcap
```

---

## 6. Triggering the TCP Handshake

With Wireshark actively monitoring `br-lan`, open a terminal window to trigger an HTTP request from the `client` (`10.10.10.10`) to the `web-server` (`10.10.10.20`):

1. Open an interactive shell inside the `client` container:
   ```bash
   docker exec -it clab-syn-ack-client sh
   ```

2. Initiate a TCP HTTP connection to the web server:
   ```bash
   curl http://10.10.10.20:80
   ```

---

## 7. Wireshark Display Filters

To isolate and analyze the three-way handshake in Wireshark, enter the following display filters into the top filter bar:

### 1. Complete TCP Conversation Filter
To view the full handshake alongside the HTTP exchange between client and web server:
```text
ip.addr == 10.10.10.20 && tcp.port == 80
```

### 2. Isolate Handshake Flags (`SYN` $\rightarrow$ `SYN-ACK` $\rightarrow$ `ACK`)
```text
tcp.flags.syn == 1 or (tcp.flags.syn == 1 and tcp.flags.ack == 1) or (tcp.flags.ack == 1 and tcp.len == 0)
```

### 3. Step-by-Step Individual Flag Filters
* **Step 1: `SYN` Packet (Client to Server)**
  ```text
  tcp.flags.syn == 1 && tcp.flags.ack == 0
  ```
* **Step 2: `SYN-ACK` Packet (Server to Client)**
  ```text
  tcp.flags.syn == 1 && tcp.flags.ack == 1
  ```
* **Step 3: `ACK` Packet (Client to Server)**
  ```text
  tcp.flags.syn == 0 && tcp.flags.ack == 1 && tcp.len == 0
  ```

---

## 8. Analyzing Sequence & Acknowledgment Numbers

When evaluating the packets in Wireshark, confirm the sequence ($Seq$) and acknowledgment ($Ack$) values:

| Step | Packet Type | Source IP | Destination IP | Sequence / Acknowledgment Numbers |
| :--- | :--- | :--- | :--- | :--- |
| **1** | **`[SYN]`** | `10.10.10.10` | `10.10.10.20` | $Seq = 0$ (Relative ISN) |
| **2** | **`[SYN, ACK]`** | `10.10.10.20` | `10.10.10.10` | $Seq = 0$, $Ack = 1$ |
| **3** | **`[ACK]`** | `10.10.10.10` | `10.10.10.20` | $Seq = 1$, $Ack = 1$ |

---

## 9. Cleanup

When finished with the lab, destroy the topology and remove the bridge interface:

```bash
# Destroy Containerlab nodes
sudo containerlab destroy -t syn-ack.clab.yml

# Remove the Linux bridge interface from host
sudo ip link delete br-lan
```
