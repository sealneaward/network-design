# Lab Assignment: Layer 7 WAF Defense Against PHP-to-Shell Exploits

## Overview & Objectives
The objective of this lab is to design, deploy, and verify a Layer 7 Web Application Firewall (WAF) using ModSecurity, NGINX, and Containerlab. You will configure an isolated switched network topology (`sw0`) connecting an attacker node, a WAF proxy, and a backend vulnerable PHP application (`php:8.2.34-apache`). Your primary task is to write custom ModSecurity rules to protect against **php2shell** remote code execution vectors.

---

## 1. Network Topology & IP Allocation

Use the diagram below to configure your Containerlab node interfaces, bridge attachments, and subnet assignments (`10.0.1.0/24`).

```mermaid
graph TD
    subgraph Containerlab Switched Network (10.0.1.0/24)
        SW["Alpine Switch: sw0<br/>Bridge: br0"]
        
        A["Attacker Node<br/>eth1: 10.0.1.1/24"] ---|eth1 - eth1| SW
        W["ModSecurity WAF<br/>eth1: 10.0.1.3/24"] ---|eth1 - eth2| SW
        P["PHP Web Server<br/>eth1: 10.0.1.2/24"] ---|eth1 - eth3| SW
    end
```

---

## 2. Repository Directory Structure
Ensure your local repository matches the following structure before submitting:

```
modsec-php-lab/
├── lab.clab.yml
├── images/
│   ├── php/
│   │   └── Dockerfile
│   └── modsec/
│       └── Dockerfile
└── rules/
    └── custom_rules.conf
```

---

## 3. Configuration & Custom Image Builds

### A. Containerlab Topology (`lab.clab.yml`)
Configure your topology using the custom built images `custom-php:latest` and `custom-waf:latest` across the `10.0.1.0/24` subnet according to the IP address scheme shown in the diagram above.

```yaml
name: modsec-php-lab

topology:
  nodes:
    sw0:
      kind: linux
      image: alpine:latest
      # TODO: Add bridge and interface setup commands in exec

    attacker:
      kind: linux
      image: alpine:latest
      # TODO: Configure attacker IP (10.0.1.1/24) and routing

    phpserver:
      kind: linux
      image: custom-php:latest
      # TODO: Configure backend PHP container IP (10.0.1.2/24)

    waf:
      kind: linux
      image: custom-waf:latest
      # TODO: Configure WAF proxy IP (10.0.1.3/24), environment variables, and rule binds
  
  links:
    - endpoints: ["attacker:eth1", "sw0:eth1"]
    - endpoints: ["waf:eth1", "sw0:eth2"]
    - endpoints: ["phpserver:eth1", "sw0:eth3"]
```

### B. Building Custom Images
Because Containerlab nodes require network administration utilities like `iproute2` (`ip` command), you will build custom images locally using Docker.

**PHP Backend Image (`images/php/Dockerfile`):**
```dockerfile
FROM php:8.2.34-apache
RUN apt-get update && apt-get install -y iproute2 && rm -rf /var/lib/apt/lists/*
```
*Build command:* `docker build -t custom-php:latest ./images/php`

**ModSecurity WAF Image (`images/modsec/Dockerfile`):**
```dockerfile
FROM owasp/modsecurity-crs:nginx
USER root
RUN apt-get update && apt-get install -y iproute2 && rm -rf /var/lib/apt/lists/*
```
*Build command:* `docker build -t custom-waf:latest ./images/modsec`

### C. Custom ModSecurity Rules (`rules/custom_rules.conf`)
Provide your custom rule definitions targeting php2shell RCE vectors, suspicious parameters, and malicious user-agents:

```apache
# Enable ModSecurity Engine
SecRuleEngine On

# TODO: Add your custom rules below
```

---

## 4. Lab Validation & Testing Log
Execute the following pre-populated `curl` commands from your `attacker` container and record or paste the corresponding terminal output.

### Test 1: Legitimate Traffic (`200 OK`)
Verify that standard traffic reaches the backend PHP application successfully through the WAF.
```bash
docker exec -it clab-modsec-php-lab-attacker curl -i "http://10.0.1.3/index.php?page=home"
```
* **Expected Output:** `HTTP/1.1 200 OK`
* **Actual Output Log:**
```
# Paste command and output here
```

### Test 2: Malicious PHP Payload (`403 Forbidden`)
Test your custom ModSecurity rules by injecting a simulated shell execution string or parameter exploit vector.
```bash
docker exec -it clab-modsec-php-lab-attacker curl -i "http://10.0.1.3/index.php?cmd=system($_GET['id'])"
```
* **Expected Output:** `HTTP/1.1 403 Forbidden`
* **Actual Output Log:**
```
# Paste command and output here
```

### Test 3: Unauthorized User-Agent Block (`403 Forbidden`)
Verify that automated scanning headers or unauthorized client signatures are blocked by your rule set.
```bash
docker exec -it clab-modsec-php-lab-attacker curl -i -s -H "User-Agent: php-exploit-framework" "http://10.0.1.3/"
```
* **Expected Output:** `HTTP/1.1 403 Forbidden`
* **Actual Output Log:**
```
# Paste command and output here
```

---

## 5. Marking Rubric

This lab is graded out of a total of **100 points**. Work is evaluated based on repository configuration, file completeness, and rule functionality against test outcomes.

| Category | Requirement / Description | Weight | Points |
| :--- | :--- | :---: | :---: |
| **GitHub Setup & Repository Structure** | • Clean repository structure with a valid `lab.clab.yml` topology file.<br>• Properly documented Dockerfile definitions for custom PHP and WAF images.<br>• Inclusion of custom rule files under the `rules/` directory. | **25%** | **25 pts** |
| **Test Outcome 1: Legitimate Traffic** | • Normal traffic (`index.php?page=home`) successfully passes through the WAF proxy to the backend container.<br>• Returns an HTTP `200 OK` response code. | **25%** | **25 pts** |
| **Test Outcome 2: PHP Payload Detection** | • Custom ModSecurity rules successfully intercept and inspect query arguments.<br>• Malicious execution parameters (`cmd=system(...)`) trigger a block returning an HTTP `403 Forbidden` response. | **25%** | **25 pts** |
| **Test Outcome 3: Header / Client Enforcement** | • Request header inspection rules correctly identify and reject forbidden tool signatures or user-agents.<br>• Returns an HTTP `403 Forbidden` response code. | **25%** | **25 pts** |
| **Total** | | **100%** | **100 pts** |
