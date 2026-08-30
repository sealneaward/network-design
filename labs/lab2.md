# Hands-On Network Lab: DNS Resolution & HTTP Traffic Analysis with Containerlab & Wireshark in WSL

## 1. Overview & Objectives

In this lab, you will build a local multi-node network topology using **Containerlab** inside **Windows Subsystem for Linux (WSL)**. You will configure a custom DNS server (`bind9`), set up a simple web server running `nginx`, host a custom domain, and analyze the resulting DNS resolution and HTTP web traffic using **Wireshark** running inside WSL connected to **VS Code**.

### Learning Objectives
- Design and deploy a Containerlab topology with a bridge switch, a DNS server, a Web server, and a client workstation.
- Configure DNS records (`A` record) using `bind9` to resolve `www.example.local` to a local server IP.
- Host an HTTP web application using Nginx.
- Launch Wireshark in WSL to capture live packet traffic on simulated network interfaces.
- Filter and inspect DNS queries/responses and HTTP GET requests using display filters in Wireshark and VS Code.

---

## 2. Network Topology & Architecture

The lab consists of three nodes connected through a single Linux bridge (`br-lan`):

1. **dns-server** (`10.10.10.10/24`) — BIND9 DNS Server hosting the zone for `example.local`.
2. **web-server** (`10.10.10.20/24`) — Nginx Web Server serving a custom web page.
3. **client** (`10.10.10.30/24`) — Alpine Linux client issuing DNS requests (`dig`/`nslookup`) and HTTP requests (`curl`).

```
                 +--------------------------------+
                 |    Switch / Bridge (br-lan)    |
                 +---------------+----------------+
                                 |
         +-----------------------+-----------------------+
         |                       |                       |
  +------+-------+        +------+-------+        +------+-------+
  |  dns-server  |        |  web-server  |        |    client    |
  | 10.10.10.10  |        | 10.10.10.20  |        | 10.10.10.30  |
  +--------------+        +--------------+        +--------------+
```

---

## 3. Containerlab Topology Configuration

Create a file named `setup.sh` in your WSL lab directory:
```bash
# Create the bridge
sudo ip link add br-lan type bridge

# Bring the interface UP
sudo ip link set br-lan up
```

Make it executable and run it:
```bash
chmod +x setup.sh
./setup.sh
```

Create a file named `dns.clab.yml` in your WSL lab directory:

```yaml
name: dns

topology:
  nodes:
    br-lan:
      kind: bridge

    client:
      kind: linux
      image: alpine:latest
      exec:
        - apk add curl
        - ip addr add 10.10.10.30/24 dev eth1
        - ip link set eth1 up
        # Set 10.10.10.10 as the primary nameserver and search local domain
        - sh -c 'echo "nameserver 10.10.10.10" > /etc/resolv.conf'
        - sh -c 'echo "search example.local" >> /etc/resolv.conf'

    dns-server:
      kind: linux
      image: ubuntu:22.04
      env:
        DEBIAN_FRONTEND: noninteractive
      exec:
        - apt-get update
        - apt-get install -y -q iproute2 bind9 bind9utils bind9-doc
        - ip addr add 10.10.10.10/24 dev eth1
        - ip link set eth1 up
        - sh -c 'echo "options { directory \"/var/cache/bind\"; allow-query { any; }; listen-on { any; }; };" > /etc/bind/named.conf.options'
        - sh -c 'echo "zone \"example.local\" { type master; file \"/etc/bind/db.example.local\"; };" > /etc/bind/named.conf.local'
        - sh -c 'echo "\$TTL 86400" > /etc/bind/db.example.local'
        - sh -c 'echo "@ IN SOA dns-server.example.local. admin.example.local. (2026083001 3600 1800 604800 86400)" >> /etc/bind/db.example.local'
        - sh -c 'echo "@ IN NS dns-server.example.local." >> /etc/bind/db.example.local'
        - sh -c 'echo "@ IN A 10.10.10.10" >> /etc/bind/db.example.local'
        - sh -c 'echo "dns-server IN A 10.10.10.10" >> /etc/bind/db.example.local'
        - sh -c 'echo "web-server IN A 10.10.10.20" >> /etc/bind/db.example.local'
        # Added CNAME record for www alias
        - sh -c 'echo "www IN CNAME web-server.example.local." >> /etc/bind/db.example.local'
        - chown -R bind:bind /etc/bind
        - service named restart

    web-server:
      kind: linux
      image: nginx:alpine
      exec:
        - ip addr add 10.10.10.20/24 dev eth1
        - ip link set eth1 up
        # Setup web server response
        - printf '<!DOCTYPE html>\n<html>\n<head><title>Welcome to Example Local!</title></head>\n<body>\n<h1>Success! Hosted on web-server (10.10.10.20)</h1>\n<p>DNS Resolution provided by dns-server (10.10.10.10).</p>\n</body>\n</html>\n' > /usr/share/nginx/html/index.html

  links:
    - endpoints: ["dns-server:eth1", "br-lan:eth1"]
    - endpoints: ["web-server:eth1", "br-lan:eth2"]
    - endpoints: ["client:eth1", "br-lan:eth3"]
```

---

## 4. Lab Step-by-Step Execution Guide

### Step 1: Deploy the Containerlab Topology
In your WSL terminal (e.g., Ubuntu on WSL2), run:

```bash
sudo containerlab deploy -t dns.clab.yml
```

Verify that all containers are running and connected:

```bash
sudo containerlab inspect -t dns.clab.yml
```

---

### Step 2: Testing DNS Resolution & Web Reachability

1. Open a terminal into the `client` node:
   ```bash
   docker exec -it clab-cdns-client sh
   ```

2. Perform a DNS query using `nslookup` or `dig` targeted at `www.example.local`:
   ```bash
   nslookup www.example.local
   ```
   *Expected Output:*
   ```text
   Server:		10.10.10.10
   Address:	10.10.10.10#53

   Name:	www.example.local
   Address: 10.10.10.20
   ```

3. Fetch the webpage from the web server using domain name:
   ```bash
   curl http://www.example.local
   ```
   *Expected Output:* HTML content from the `web-server`.

---

### Step 3: Setting Up Wireshark Capture in WSL & VS Code

To inspect live traffic visually in Wireshark from WSL:

#### Method A: Direct GUI Wireshark via WSLg (Windows 11)
1. Ensure Wireshark is installed inside your WSL instance:
   ```bash
   sudo apt update && sudo apt install -y wireshark tshark
   ```
2. Launch Wireshark directly from your WSL prompt:
   ```bash
   sudo wireshark &
   ```
3. Select interface `veth-client` or the bridge interface `br-lan` to start capturing packets.

#### Method B: Remote Capture with VS Code & Remote-SSH / WSL Extension
1. Open VS Code and attach to your WSL environment (`Ctrl+Shift+P` -> `WSL: Connect to WSL`).
2. Open the terminal inside VS Code (`Ctrl + ~`).
3. Run `tshark` on the client or bridge interface and output to a PCAP file:
   ```bash
   sudo tshark -i br-lan -w /tmp/dns_http_capture.pcap
   ```
4. Perform web requests on the client container in a second terminal tab:
   ```bash
   docker exec cdns-client curl http://www.example.local
   ```
5. Stop `tshark` (`Ctrl + C`) and inspect the generated PCAP file directly inside VS Code using the **vscode-packet-pane** extension or open it with desktop Wireshark.

---

### Step 4: Applying Wireshark Display Filters

Once the capture is active or saved in Wireshark, apply the following display filters to analyze the protocol exchange:

#### Filter 1: DNS Traffic Only
```text
dns
```
- **Analysis:** Observe the `Standard query 0x... A www.example.local` sent from `10.10.10.30` to `10.10.10.10`, followed by the `Standard query response 0x... A 10.10.10.20`.

#### Filter 2: HTTP Traffic Only
```text
http
```
- **Analysis:** Look for the `GET / HTTP/1.1` request sent from `10.10.10.30` to `10.10.10.20` and the corresponding `HTTP/1.1 200 OK` response.

#### Filter 3: Combined DNS & HTTP Filter
```text
dns || http
```
- **Analysis:** Observe the full sequence:
  1. DNS Request (`10.10.10.30` -> `10.10.10.10`)
  2. DNS Response (`10.10.10.10` -> `10.10.10.30`)
  3. TCP 3-Way Handshake (`10.10.10.30` <-> `10.10.10.20`)
  4. HTTP GET Request & 200 OK Response

---

## 5. Verification Checklist & Challenge Questions

1. **DNS Verification:** What port does the DNS query operate over, and is it TCP or UDP?
2. **A-Record Verification:** What IP address is returned for `www.example.local`?
3. **HTTP Packet Inspection:** Inspect the HTTP GET request header in Wireshark. What is the value of the `Host:` header field?
4. **Cleanup:** Destroy the Containerlab environment after completing the lab:
   ```bash
   sudo containerlab destroy -t dns.clab.yml
   ```
