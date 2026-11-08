# Lab Assignment: Enterprise Campus Network Security & Application Defense

## Objective

In this comprehensive lab assignment, you will synthesize core enterprise network design concepts into a single campus simulation using **Containerlab**, **Alpine Linux**, **`iptables`**, and **OWASP ModSecurity WAF**.

You will design, configure, and secure a 4-subnet campus environment connecting Students, Faculty, IT Admin staff, and Centralized Services. The network requires both **Layer 3/4 Packet Filtering Firewalls** on the central router to enforce strict inter-subnet access policies and a **Layer 7 Web Application Firewall (WAF)** to defend vulnerable application services against web exploits.

## 1. Campus Network Architecture & Topology

The campus network consists of four distinct subnets connected through a central Alpine Linux Gateway / Firewall router, utilizing dedicated virtual switch bridges (`bridge` kind in Containerlab) per subnet to maintain independent gateway interfaces (`.1`):

1. **Student Subnet (`10.10.10.0/24`):** General student population client zone.
2. **Faculty Subnet (`10.10.20.0/24`):** Academic staff client zone.
3. **IT Admin Subnet (`10.10.30.0/24`):** Network and system administration client zone.
4. **Services Subnet (`10.10.40.0/24`):** Centralized campus servers, databases, and web applications.

```mermaid
flowchart TD
    subgraph STUDENT ["Student Subnet (10.10.10.0/24)"]
        student_node["Student Client<br/>10.10.10.10/24<br/>GW: 10.10.10.1"]
    end

    subgraph FACULTY ["Faculty Subnet (10.10.20.0/24)"]
        faculty_node["Faculty Client<br/>10.10.20.10/24<br/>GW: 10.10.20.1"]
    end

    subgraph ADMIN ["IT Admin Subnet (10.10.30.0/24)"]
        admin_node["IT Admin Client<br/>10.10.30.10/24<br/>GW: 10.10.30.1"]
    end

    subgraph GATEWAY ["Central Perimeter Router / Firewall"]
        router["Alpine Router (iptables)<br/><br/>eth1: 10.10.10.1/24 (Student GW)<br/>eth2: 10.10.20.1/24 (Faculty GW)<br/>eth3: 10.10.30.1/24 (Admin GW)<br/>eth4: 10.10.40.1/24 (Services GW)"]
    end

    subgraph SERVICES ["Services Subnet (10.10.40.0/24)"]
        direction TB
        waf["ModSecurity WAF Proxy<br/>10.10.40.80/24<br/>GW: 10.10.40.1"]
        juiceshop["OWASP Juice Shop<br/>(Bulletin App)<br/>10.10.40.81/24<br/>GW: 10.10.40.1"]
        postgres["PostgreSQL Database<br/>(Grading DB)<br/>10.10.40.100/24<br/>GW: 10.10.40.1"]
    end

    student_node ---|eth1 <--> sw-student| sw_student((Student Switch))
    sw_student ---|router:eth1| router
    
    faculty_node ---|eth1 <--> sw-faculty| sw_faculty((Faculty Switch))
    sw_faculty ---|router:eth2| router
    
    admin_node ---|eth1 <--> sw-admin| sw_admin((Admin Switch))
    sw_admin ---|router:eth3| router
    
    waf ---|eth1 <--> sw-services| sw_services((Services Switch))
    juiceshop ---|eth1 <--> sw-services| sw_services
    postgres ---|eth1 <--> sw-services| sw_services
    sw_services ---|router:eth4| router

    classDef admin fill:#ffe6e6,stroke:#d9534f,stroke-width:2px;
    classDef faculty fill:#fff0e6,stroke:#f0ad4e,stroke-width:1px;
    classDef student fill:#e6ffe6,stroke:#5cb85c,stroke-width:1px;
    classDef services fill:#e1f5fe,stroke:#0288d1,stroke-width:2px;
    classDef router fill:#f9f,stroke:#333,stroke-width:2px;

    class ADMIN admin;
    class FACULTY faculty;
    class STUDENT student;
    class SERVICES services;
    class GATEWAY router;
```

## 2. Lab Objectives & Security Policies

### Policy 1: Gateway Routing & Inter-Subnet Stateful Filtering

* The central router must enable kernel IP forwarding (`sysctl -w net.ipv4.ip_forward=1`) and serve as the default gateway (`.1`) across all router interfaces (`eth1` through `eth4`).
* Enforce stateful packet tracking (`ESTABLISHED,RELATED`) on `iptables` to allow return traffic automatically for established outbound sessions.
* Enforce a default `DROP` policy on the `FORWARD` filter chain.

### Policy 2: Admin SSH Privilege Restrictions

* **Only hosts residing on the IT Admin Subnet (`10.10.30.0/24`)** are permitted to initiate SSH connections (`tcp/22`) to devices/servers in any other subnet.
* All SSH traffic initiated from Student, Faculty, or Services subnets must be blocked and dropped by the central router firewall.

### Policy 3: Campus Web & Database Access Control

1. **School Bulletin App (OWASP Juice Shop):**
   * Must be accessible via HTTP (`tcp/8080`) through the **ModSecurity WAF Proxy** (`10.10.40.80`) by **Students, Faculty, and IT Admin staff**.
   * Direct user connections from client subnets to the backend container directly (`10.10.40.81`) must be prevented or isolated by firewall rules; all user HTTP traffic must transit through the WAF. The WAF routes backend traffic via its environment configuration (`BACKEND="http://10.10.40.81:3000"`).

2. **Grading Database (PostgreSQL):**
   * Accessible over PostgreSQL port (`tcp/5432`) on target IP `10.10.40.100`. Database admin username must be set to `postgres` and admin password to `postgres`.
   * **Only Faculty (`10.10.20.0/24`) and IT Admin (`10.10.30.0/24`)** subnets may initiate connections to port 5432.
   * Access attempts from the Student subnet to the grading database must be dropped by the network firewall.

### Policy 4: Layer 7 WAF Protection Against SQL Injection

* The ModSecurity reverse proxy (`10.10.40.80`) front-ends the bulletin app.
* Standard legitimate web access through the WAF must yield an HTTP `200 OK` response.
* Requests containing SQL Injection attack vectors targeting the application (e.g., `' OR '1'='1`) must be intercepted by custom ModSecurity rules and return `403 Forbidden`.

---

## 3. Directory Layout & Custom Image Building Instructions

### Directory Structure

```
.
├── README.md                   # Lab setup and execution documentation
├── lab.clab.yml                # Master Containerlab topology definition file
├── config/
│   ├── iptables.rules          # Student-configured network firewall rules
│   └── modsec-custom.conf      # Student-configured Layer 7 WAF rules
└── images/
    ├── juiceshop/
    │   └── Dockerfile          # Custom Juice Shop build file (with iproute2)
    └── modsec/
        └── Dockerfile          # Custom ModSecurity build file (with iproute2)
```

### Pre-requisite: Local Image Builds

Because lightweight application containers omit networking utilities required by Containerlab for static IP/gateway configuration, you **must build custom local container images** prior to deploying the topology.

#### 1. Build Custom Juice Shop Image (`images/juiceshop/Dockerfile`)

```dockerfile
FROM node:24-trixie

USER root

RUN apt-get update && apt-get install -y --no-install-recommends \
    bash \
    iproute2 \
    iputils-ping \
    git \
    make \
    g++ \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /juice-shop

RUN git clone --depth 1 https://github.com/juice-shop/juice-shop.git .
RUN npm install

EXPOSE 3000

CMD ["npm", "start"]
```

*Build Command:*
```bash
docker build -t custom-juiceshop:latest ./images/juiceshop
```

#### 2. Build Custom ModSecurity WAF Image (`images/modsec/Dockerfile`)

```dockerfile
FROM owasp/modsecurity-crs:nginx

USER root

RUN apt-get update \
    && apt-get install -y --no-install-recommends iproute2 iputils-ping \
    && rm -rf /var/lib/apt/lists/*
```

*Build Command:*
```bash
docker build -t custom-waf:latest ./images/modsec
```

---

## 4. Starter Templates

### Starter: `config/iptables.rules`

```text
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]

# =======================================================
# TODO: IMPLEMENT YOUR IPTABLES RULES BELOW
# =======================================================

# --- 1. ALLOW ESTABLISHED & RELATED TRAFFIC ---


# --- 2. SSH MANAGEMENT ACCESS CONTROL ---
# Allow ONLY 10.10.30.0/24 (Admin) to initiate SSH anywhere


# --- 3. BULLETIN APP ACCESS CONTROL ---
# Allow Student, Faculty, and Admin to access WAF Proxy (10.10.40.80:8080)
# Block direct client access to Juice Shop backend (10.10.40.81)


# --- 4. GRADING DATABASE ACCESS CONTROL ---
# Allow Faculty and Admin to access PostgreSQL DB (10.10.40.100:5432)


COMMIT
```

### Starter: `lab.clab.yml` (Using Switch Bridges per Subnet)

```yaml
name: campus-network-security

topology:
  nodes:
    # 1. Central Router / Firewall Node
    router:
      kind: linux
      image: alpine:latest
      exec:
        - apk add --no-cache iproute2 iptables
        - sysctl -w net.ipv4.ip_forward=1
        
        # Configure Gateway IP Addresses per Interface
        - ip addr replace 10.10.10.1/24 dev eth1
        - ip addr replace 10.10.20.1/24 dev eth2
        - ip addr replace 10.10.30.1/24 dev eth3
        - ip addr replace 10.10.40.1/24 dev eth4
        
        # Bring router interfaces up
        - ip link set dev eth1 up
        - ip link set dev eth2 up
        - ip link set dev eth3 up
        - ip link set dev eth4 up

        # TODO: Apply iptables security policies and rules

    # 2. Subnet Switches (Linux bridge kind to host multiple hosts per subnet)
    sw-student:
      kind: bridge
    sw-faculty:
      kind: bridge
    sw-admin:
      kind: bridge
    sw-services:
      kind: bridge

    # 3. Subnet Clients
    student-client:
      kind: linux
      image: alpine:latest
      exec:
        - apk add --no-cache curl netcat-openbsd iproute2
        - ip addr add 10.10.10.10/24 dev eth1
        - ip route replace default via 10.10.10.1 dev eth1
        - ip link set dev eth1 up

    faculty-client:
      kind: linux
      image: alpine:latest
      exec:
        - apk add --no-cache curl netcat-openbsd iproute2
        - ip addr add 10.10.20.10/24 dev eth1
        - ip route replace default via 10.10.20.1 dev eth1
        - ip link set dev eth1 up

    admin-client:
      kind: linux
      image: alpine:latest
      exec:
        - apk add --no-cache curl netcat-openbsd iproute2
        - ip addr add 10.10.30.10/24 dev eth1
        - ip route replace default via 10.10.30.1 dev eth1
        - ip link set dev eth1 up

    # 4. Centralized Services Subnet
    waf-proxy:
      kind: linux
      image: custom-waf:latest
      env:
        BACKEND: "http://10.10.40.81:3000"
        MODSEC_RULE_ENGINE: "On"
      binds:
        - config/modsec-custom.conf:/etc/modsecurity.d/owasp-crs/rules/RESPONSE-999-CUSTOM.conf:ro
      exec:
        - ip addr add 10.10.40.80/24 dev eth1
        - ip route replace default via 10.10.40.1 dev eth1
        - ip link set dev eth1 up

    juiceshop-app:
      kind: linux
      image: custom-juiceshop:latest
      exec:
        - ip addr add 10.10.40.81/24 dev eth1
        - ip route replace default via 10.10.40.1 dev eth1
        - ip link set dev eth1 up

    postgres-db:
      kind: linux
      image: postgres:15-alpine
      env:
        POSTGRES_USER: postgres
        POSTGRES_PASSWORD: postgres
        POSTGRES_DB: grading_db
      exec:
        - apk add --no-cache iproute2 || true
        - ip addr add 10.10.40.100/24 dev eth1 || true
        - ip route replace default via 10.10.40.1 dev eth1 || true
        - ip link set dev eth1 up || true

  links:
    # Connect Subnet 1 (Student) to Switch
    - endpoints: ["student-client:eth1", "sw-student:eth1"]
    - endpoints: ["router:eth1", "sw-student:eth2"]

    # Connect Subnet 2 (Faculty) to Switch
    - endpoints: ["faculty-client:eth1", "sw-faculty:eth1"]
    - endpoints: ["router:eth2", "sw-faculty:eth2"]

    # Connect Subnet 3 (Admin) to Switch
    - endpoints: ["admin-client:eth1", "sw-admin:eth1"]
    - endpoints: ["router:eth3", "sw-admin:eth2"]

    # Connect Subnet 4 (Services) to Switch (WAF, Juice Shop, Postgres)
    - endpoints: ["waf-proxy:eth1", "sw-services:eth1"]
    - endpoints: ["juiceshop-app:eth1", "sw-services:eth2"]
    - endpoints: ["postgres-db:eth1", "sw-services:eth3"]
    - endpoints: ["router:eth4", "sw-services:eth4"]
```

---

## 5. Verification & Automated Test Suite

Your topology must successfully run and pass all six of the following test cases executed directly via `docker exec`:

### **Test 1: Legitimate Bulletin Web Access (Student to WAF)**
* **Command:** `docker exec -it clab-campus-network-security-student-client curl -s -o /dev/null -w "%{http_code}" http://10.10.40.80:8080`
* **Expected Result:** `200`

### **Test 2: ModSecurity WAF Interception of SQL Injection**
* **Command:** `docker exec -it clab-campus-network-security-student-client curl -s -o /dev/null -w "%{http_code}" "http://10.10.40.80:8080/rest/user/login?email='OR'1'='1"`
* **Expected Result:** `403`

### **Test 3: Authorized Grading DB Access (Faculty Subnet)**
* **Command:** `docker exec -it clab-campus-network-security-faculty-client nc -zv -w 3 10.10.40.100 5432`
* **Expected Result:** `open` / connection succeeded

### **Test 4: Unauthorized Grading DB Access Block (Student Subnet)**
* **Command:** `docker exec -it clab-campus-network-security-student-client nc -zv -w 3 10.10.40.100 5432`
* **Expected Result:** Connection timeout / failure

### **Test 5: SSH Privilege Validation (IT Admin Host Allowed)**
* **Command:** `docker exec -it clab-campus-network-security-admin-client nc -zv -w 3 10.10.40.100 22`
* **Expected Result:** `open` / connection succeeded (or connection reset)

### **Test 6: SSH Privilege Restriction (Student Host Blocked)**
* **Command:** `docker exec -it clab-campus-network-security-student-client nc -zv -w 3 10.10.40.100 22`
* **Expected Result:** Connection timeout / failure

---

## 6. Submission Requirements

You must host your completed assignment in a **Public GitHub Repository** containing:

1. Valid `lab.clab.yml` topology definition file.
2. Complete Dockerfiles under `images/juiceshop/` and `images/modsec/`.
3. Complete configuration files under `config/iptables.rules` (or defined in the container `exec` in the `yml` file) and `config/modsec-custom.conf`.
4. A comprehensive `README.md` file providing:
   * Prerequisites and local Docker image build instructions.
   * Containerlab deployment (`sudo clab deploy -t lab.clab.yml`) and destroy (`sudo clab destroy -t lab.clab.yml`) commands.
   * Step-by-step verification commands proving all security test cases pass.

---

## 7. Grading Rubric

This lab assignment is graded out of **100 total points**:

| Category | Description & Criteria | Weight | Points |
| :--- | :--- | :---: | :---: |
| **Repository Structure & Documentation** | • Valid `lab.clab.yml` syntax without execution errors.<br>• Clean repository layout matching the specified directory structure.<br>• Comprehensive `README.md` containing image build steps, deployment/cleanup instructions, and clear test steps. | **25%** | **25 pts** |
| **Test 1 & 2: WAF & Web Security** | • Legitimate web traffic passes through WAF to backend with HTTP `200 OK`.<br>• SQL Injection payload blocked by custom ModSecurity rules returning HTTP `403 Forbidden`. | **25%** | **25 pts** |
| **Test 3 & 4: Database Access Control** | • Faculty and IT Admin subnets can communicate with PostgreSQL on port `5432`.<br>• Student subnet access attempts to port `5432` are dropped by `iptables`. | **25%** | **25 pts** |
| **Test 5 & 6: SSH Management Control** | • Only IT Admin subnet can reach devices over SSH port `22`.<br>• SSH attempts from all non-admin subnets are blocked by `iptables`. | **25%** | **25 pts** |
| **Total** | | **100%** | **100 pts** |
