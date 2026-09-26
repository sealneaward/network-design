# Tutorial: Capturing the TCP Three-Way Handshake in Containerlab

This tutorial guides you through setting up a simple Containerlab topology, triggering a TCP connection between two Linux containers, and using Wireshark filters to capture and observe the classic **TCP Three-Way Handshake** (`SYN` -> `SYN-ACK` -> `ACK`).

---

## 1. Prerequisites

Before starting, ensure you have the following installed on your host machine:

* **Containerlab**: [Installation Guide](https://containerlab.dev/install/)
* **Docker**: Required as the engine for Containerlab nodes.
* **Wireshark** or **tshark**: For packet analysis and filtering.

---

## 2. Containerlab Topology Definition

Create a file named `tcp-lab.clab.yml`. This topology defines two Alpine Linux containers connected via a point-to-point network link.

```yaml
name: tcp-handshake-lab

topology:
  nodes:
    client:
      kind: linux
      image: alpine:latest
      exec:
        - ip addr add 10.0.0.1/24 dev eth1
        - apk add --no-cache curl iproute2

    server:
      kind: linux
      image: alpine:latest
      exec:
        - ip addr add 10.0.0.2/24 dev eth1
        - apk add --no-cache python3 iproute2
        - python3 -m http.server 80 &

  links:
    - endpoints: ["client:eth1", "server:eth1"]
```

### Explanation of the Topology
* **`client`**: Assigned IP `10.0.0.1/24` on interface `eth1`. It automatically installs `curl` to initiate the TCP request.
* **`server`**: Assigned IP `10.0.0.2/24` on interface `eth1`. It starts a lightweight Python HTTP server listening on port **80**.
* **`links`**: Connects `client:eth1` directly to `server:eth1`.

---

## 3. Deploying the Lab

Deploy the Containerlab topology using the following command:

```bash
sudo containerlab deploy -t tcp-lab.clab.yml
```

Once deployment completes, Containerlab creates virtual interface veth pairs on your Linux host.

---

## 4. Capturing Traffic with Wireshark

### Option A: Wireshark via Containerlab Capture Command
Containerlab provides a built-in feature to launch Wireshark directly attached to a specific interface:

```bash
sudo containerlab capture -t tcp-lab.clab.yml -n client -i eth1
```

### Option B: Manual Host Interface Capture
Alternatively, locate the veth interface name using `ip link` (e.g., `clab-tcp-handshake-lab-client`) and launch Wireshark manually:

```bash
sudo wireshark -k -i clab-tcp-handshake-lab-client
```

---

## 5. Triggering the TCP Handshake

Open a terminal session to execute a request from the client to the server.

1. Enter the client container:
   ```bash
   docker exec -it clab-tcp-handshake-lab-client sh
   ```
2. Initiate a TCP connection to the HTTP server:
   ```bash
   curl http://10.0.0.2:80
   ```

---

## 6. Wireshark Display Filters

To isolate and examine the TCP three-way handshake packets in Wireshark, use the following display filters in the Wireshark filter bar:

### 1. Filter by TCP Flags (The Handshake Packets)

* **Capture all TCP Handshake Control Packets (`SYN`, `SYN-ACK`, `ACK`):**
  ```wireshark
  tcp.flags.syn == 1 or (tcp.flags.syn == 1 and tcp.flags.ack == 1) or (tcp.flags.ack == 1 and tcp.len == 0)
  ```

* **Filter Specifically for `SYN` (Step 1):**
  ```wireshark
  tcp.flags.syn == 1 && tcp.flags.ack == 0
  ```

* **Filter Specifically for `SYN-ACK` (Step 2):**
  ```wireshark
  tcp.flags.syn == 1 && tcp.flags.ack == 1
  ```

* **Filter Specifically for `ACK` (Step 3):**
  ```wireshark
  tcp.flags.syn == 0 && tcp.flags.ack == 1 && tcp.len == 0
  ```

---

### 2. General Conversation Filters

* **Isolate Traffic Between Client and Server Port 80:**
  ```wireshark
  ip.addr == 10.0.0.2 && tcp.port == 80
  ```

* **Show Complete TCP Stream (Including Data Exchange):**
  ```wireshark
  tcp.stream == 0
  ```

---

## 7. Understanding the Three-Way Handshake Structure

When observing the packets in Wireshark, you should see the following sequence:

| Step | Source IP | Destination IP | TCP Flags | Description |
| :--- | :--- | :--- | :--- | :--- |
| **1** | `10.0.0.1` | `10.0.0.2` | `[SYN]` | Client sends `SYN` with an initial sequence number ($Seq = X$). |
| **2** | `10.0.0.2` | `10.0.0.1` | `[SYN, ACK]` | Server responds with `SYN-ACK` ($Seq = Y$, $Ack = X + 1$). |
| **3** | `10.0.0.1` | `10.0.0.2` | `[ACK]` | Client acknowledges with `ACK` ($Seq = X + 1$, $Ack = Y + 1$). |

---

## 8. Cleaning Up

To destroy the lab environment and clear network interfaces:

```bash
sudo containerlab destroy -t tcp-lab.clab.yml
```