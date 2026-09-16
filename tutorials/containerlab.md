# Deep-Dive Guide: Containerlab Terminology, Topology Architecture & Operations

## 1. Introduction to Containerlab
Containerlab is an open-source command-line tool designed to orchestrate containerized network topologies. It allows network engineers, DevOps practitioners, and system architects to rapidly deploy, prototype, and validate complex multi-vendor network topologies on Linux containers and virtualized network operating systems (NOS).

### Key Features
- **Declarative Topologies:** Network topologies are defined entirely in standard YAML files (`*.clab.yml`).
- **Multi-Vendor Support:** Native integration with containerized network operating systems (SR Linux, FRR, Arista cEOS, Cisco XRd, SONiC) and virtualized legacy OS images via `vrnetlab` (Cisco IOL/CSR1000v/Nexus, Juniper vMX/vQFX, Arista vEOS).
- **Automated Wiring:** Containerlab creates veth pairs and Linux bridges automatically to interconnect containers as defined in the topology.
- **Lightweight & Fast:** Deploys complex lab environments in seconds using standard Docker/Podman APIs without full hypervisor overhead for containerized targets.

---

## 2. Core Terminology & Architecture

Understanding Containerlab requires familiarity with its core architectural components and primitives.

| Term | Definition & Description |
| :--- | :--- |
| **Lab Name (`name`)** | Unique string identifier for the lab instance. Used as a namespace prefix for container resources. |
| **Node (`nodes`)** | A single virtualized device or host within the topology (e.g., router, switch, Linux server, test generator). |
| **Kind (`kind`)** | The hardware/OS abstraction type of a node (e.g., `srl`, `ceos`, `linux`, `vr-csr`). Determines how Containerlab interacts with the image. |
| **Type (`type`)** | A sub-classification of a `kind`, often corresponding to a specific hardware platform or chassis model (e.g., `ixr6`, `7050`). |
| **Image (`image`)** | The container image repository tag used to spin up the container instance (e.g., `ghcr.io/nokia/srlinux:latest`). |
| **Link (`links`)** | Point-to-point connection definitions mapping interface endpoints between nodes. |
| **Endpoint** | Interface specification formatted as `node_name:interface_name` (e.g., `router1:eth1`). |

---

## 3. Topology YAML File Anatomy (`*.clab.yml`)

Every Containerlab deployment revolves around a single YAML topology manifest file.

### Top-Level Hierarchy
```yaml
name: net-spine-leaf  # Unique lab identifier

topology:
  defaults: # Global default configurations applied to all nodes unless overridden
    kind: srl
    image: ghcr.io/nokia/srlinux:23.10.1
    
  nodes: # Node declarations
    spine1:
      # Specific properties for spine1
    leaf1:
      # Specific properties for leaf1

  links: # Edge definitions connecting endpoints
    - endpoints: ["spine1:e1-1", "leaf1:e1-1"]
```

---

## 4. Top-Level Elements Deep Dive

### `name`
Sets the unique namespace for the deployment. Containerlab prefixes all Docker containers and virtual interfaces with `clab-<name>-`.

### `topology`
The root block containing `defaults`, `nodes`, and `links`.

---

## 5. Node Configurations & Properties

Nodes inherit properties from `topology.defaults`, but individual properties can be specified per node under `topology.nodes.<node_name>`:

```yaml
topology:
  nodes:
    spine1:
      kind: srl
      image: ghcr.io/nokia/srlinux:latest
      type: ixr6
      group: spine-layer
      startup-config: configs/spine1.cfg
      binds:
        - ./scripts:/scripts:ro
      env:
        ENVIRONMENT: LAB
      cmd: --debug
      exec:
        - ip addr add 10.0.0.1/32 dev lo
      labels:
        role: backbone
      user: admin
```

### Detailed Property Breakdown
- **`kind`**: Defines node type driver (`srl`, `ceos`, `linux`, `vr-ros`, etc.).
- **`type`**: Hardware chassis/variant selector (e.g., `ixr6` for Nokia SR Linux, `c7200` for Dynamips).
- **`group`**: Visual/logical grouping identifier used in auto-generated documentation and visualizers.
- **`binds`**: Directory or file bind mounts between the host system and container (`host_path:container_path[:mode]`).
- **`env`**: Map of environment variables injected into the container environment.
- **`cmd`**: Command string override for container entrypoint.
- **`exec`**: List of commands executed inside the container right after it boots up.
- **`user`**: User identity under which the container runs.

---

## 6. Supported Vendor Kinds & Ecosystem Matrix

Containerlab natively supports a broad set of network operating systems and node kinds:

| Kind Name | Target Platform / OS | Execution Model | Typical Vendor/Project |
| :--- | :--- | :--- | :--- |
| `srl` | Nokia SR Linux | Native Container | Nokia |
| `ceos` | Arista cEOS | Native Container | Arista Networks |
| `xrd` | Cisco IOS XRd | Native Container | Cisco Systems |
| `sonic-vs` | SONiC Virtual Switch | Native Container | Open Source (Linux Foundation) |
| `frr` | Free Range Routing | Native Container | FRRouting |
| `linux` | Generic Linux (Ubuntu, Alpine, Debian) | Native Container | Generic |
| `vr-csr` | Cisco CSR1000v | VM in Container (vrnetlab) | Cisco Systems |
| `vr-vmx` | Juniper vMX | VM in Container (vrnetlab) | Juniper Networks |
| `vr-veos` | Arista vEOS | VM in Container (vrnetlab) | Arista Networks |
| `vr-sros` | Nokia SR OS | VM in Container (vrnetlab) | Nokia |
| `vr-pan` | Palo Alto PAN-OS | VM in Container (vrnetlab) | Palo Alto Networks |
| `ovs-bridge` | Open vSwitch Bridge | Host Integration | OVS |

---

## 7. Inter-Node Connections & Link Syntax

The `links` block connects interface endpoints directly using veth pairs or Linux bridges.

### Syntax Format
```yaml
links:
  - endpoints: ["<nodeA>:<interfaceA>", "<nodeB>:<interfaceB>"]
```

### Endpoint Naming Conventions
- **Nokia SR Linux (`srl`):** `e1-1`, `e1-2` (mapped to `ethernet-1/1`, `ethernet-1/2` internally).
- **Arista cEOS (`ceos`):** `eth1`, `eth2` (mapped to `Ethernet1`, `Ethernet2`).
- **Generic Linux (`linux`):** `eth1`, `eth2`, `veth1`.
- **Cisco XRd (`xrd`):** `Gi0-0-0-0`, `Gi0-0-0-1`.

### Multi-Point & Host Bridge Connectivity
To connect multiple nodes to an external host bridge or Open vSwitch:
```yaml
topology:
  nodes:
    br-wan:
      kind: ovs-bridge
    r1:
      kind: srl
    r2:
      kind: srl

  links:
    - endpoints: ["r1:e1-1", "br-wan:r1-port"]
    - endpoints: ["r2:e1-1", "br-wan:r2-port"]
```

---

## 8. External Configuration & Resource Files

Containerlab interacts extensively with local host files to automate initial provisioning and licensing.

### 1. `startup-config`
Points to a local configuration file pushed to the device upon booting.
```yaml
nodes:
  spine1:
    kind: srl
    startup-config: configs/spine1.txt
```

### 2. License Files (`license`)
Used for vendors requiring explicit software licenses (e.g., Nokia SR OS, Arista vEOS, Cisco CSR1000v).
```yaml
nodes:
  sros1:
    kind: vr-sros
    license: /opt/licenses/sros.lic
```

### 3. Bind Mounts (`binds`)
Mounts custom scripts, certificates, or telemetry configuration files directly into containers.
```yaml
nodes:
  telemetry-agent:
    kind: linux
    image: prom/prometheus:latest
    binds:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
```

### 4. Configuration Artifact Directory Structure
When a lab is deployed, Containerlab creates a local directory named `clab-<lab-name>/` containing generated runtime files:
```text
clab-net-spine-leaf/
├── topology-data.json      # Complete topology metadata
├── spine1/
│   ├── config/             # Runtime device configurations
│   └── topology-raw.json
└── leaf1/
```

---

## 9. Inspecting Topology & Node Status

Once a lab is deployed, Containerlab provides commands to inspect running status and topology details.

### Inspect Operational Summary
```bash
containerlab inspect --name net-spine-leaf
```
*Output Summary Table:*
- Container names & IDs
- Node kinds & OS images
- State (running/stopped)
- Host bind directory paths

### Detailed Topology Inspection
To output operational metadata in JSON format for automated scripts:
```bash
containerlab inspect --name net-spine-leaf --format json
```

### Graph & Interface Visualization
Generate interactive web topology views:
```bash
containerlab graph --topo topology.clab.yml
```

---

## 10. Executing Commands inside Containers (`exec`)

Containerlab allows executing commands across single or multiple lab containers directly from the host shell.

### Single Node Command Execution
```bash
containerlab exec --name net-spine-leaf --label clab-node-name=spine1 --cmd "sr_cli 'show interface brief'"
```

### Bulk Execution across All Nodes
Run diagnostic commands simultaneously across all nodes using label selection:
```bash
# Display IP routes across all lab nodes
containerlab exec --name net-spine-leaf --cmd "ip route show"
```

### Filtering Targets by Label or Format
```bash
# Target specific group of nodes
containerlab exec --name net-spine-leaf --label role=backbone --cmd "ping -c 3 10.0.0.1"

# Output results in JSON for parsing
containerlab exec --name net-spine-leaf --cmd "hostname" --format json
```

---

## 11. Interactive Access Methods (Shell Access)

Primary methods to access the CLI of running nodes in Containerlab:

### 1. Docker/Podman Exec (Direct Container Access)
Access the container interactive shell directly:
```bash
docker exec -it clab-net-spine-leaf-spine1 sr_cli
```

### 2. Containerlab Shortcut Access
Containerlab allows shell attachment without typing long container names:
```bash
containerlab enter --name net-spine-leaf --node spine1
```

---

## 12. Packet Captures & Traffic Inspection

Inspect live data plane traffic directly on inter-node veth interfaces from the host:

```bash
# Capture traffic on veth interface between spine1 and leaf1
ip netns exec clab-net-spine-leaf-spine1 tcpdump -i e1-1 -w spine1_e1-1.pcap
```

---

## 13. Full End-to-End Workflow & Command Summary

| Phase | Command | Description |
| :--- | :--- | :--- |
| **Deploy** | `containerlab deploy -t topology.clab.yml` | Parses YAML, boots containers, wires interfaces. |
| **Inspect** | `containerlab inspect -t topology.clab.yml` | Displays tabular status of nodes and container states. |
| **Graph** | `containerlab graph -t topology.clab.yml` | Launches web UI interactive topology map. |
| **Execute** | `containerlab exec -t topology.clab.yml --cmd "..."` | Runs command across topology nodes in parallel. |
| **Save** | `containerlab save -t topology.clab.yml` | Triggers configuration save across running NOS nodes. |
| **Destroy** | `containerlab destroy -t topology.clab.yml` | Removes container instances, deletes veth pairs, cleans up lab environment. |

---

## 14. Real-World Example: Multi-Vendor Spine-Leaf Topology

Below is a production-ready multi-vendor topology YAML connecting a Nokia SR Linux spine router with an Arista cEOS leaf router and a generic Linux host:

```yaml
name: lab-spine-leaf-demo

topology:
  defaults:
    kind: srl
    
  nodes:
    spine1:
      kind: srl
      image: ghcr.io/nokia/srlinux:23.10.1
      startup-config: configs/spine1.cfg
      group: fabric-spine

    leaf1:
      kind: ceos
      image: ceos:4.30.1F
      startup-config: configs/leaf1.cfg
      group: fabric-leaf

    client1:
      kind: linux
      image: alpine:latest
      exec:
        - ip addr add 192.168.10.10/24 dev eth1
        - ip route add default via 192.168.10.1
      group: endpoints

  links:
    - endpoints: ["spine1:e1-1", "leaf1:eth1"]
    - endpoints: ["leaf1:eth2", "client1:eth1"]
```