# Lab Assignment: Building Your First Network-as-Code Topology on Windows

## Objective
In this lab, you will transition from traditional, manual network configuration to **Network-as-Code (NaC)**. You will use Windows Subsystem for Linux (WSL2), and Containerlab to provision a virtual routed network automatically using declarative YAML configuration files.

### Final Deliverable
A fully running, containerized network topology verified and mapped via an interactive browser-based visualization tool.

---

## Part 1: Host Environment Installation

Windows requires a Linux compatibility layer to run network operating system containers. Follow these installation steps precisely.

### Step 1: Ensure WSL 2 is Installed & Set as Default (optional if using Linux OS)
If WSL is not installed or configured properly on your Windows machine (common on standard Windows 10 installations):

1. Right-click your Windows Start menu and select **Terminal (Admin)** or **PowerShell (Admin)**.
2. Check if WSL is installed by running:
   ```powershell
   wsl --list --verbose
   ```
3. If WSL is not installed or missing, run:
   ```powershell
   wsl --install
   ```
4. Explicitly enable WSL 2 as your default version:
   ```powershell
   wsl --set-default-version 2
   ```
5. Install the Containerlab WSL file by following these instructions on [WSL-Containerlab](https://containerlab.dev/windows/#wsl-containerlab). You should see a Containerlab app available in the start menu.

[start-menu](../images/containerlab.PNG)


## Part 2: Infrastructure as Code Configuration

Containerlab uses declarative YAML definitions natively to specify topology nodes and their links.

### Step 1: Create Your Workspace Directory
Inside your contianerlab terminal, run the following commands to create a dedicated lab folder and navigate into it:
```bash
mkdir -p ~/labs/beginner-nac
cd ~/labs/beginner-nac
```

### Step 2: Build the `lab.clab.yml` File
Create a new file named `lab.clab.yml` using a command-line editor like `nano`:
```bash
nano lab.clab.yml
```
Copy and paste (paste in-line in WSL is done via right-click) the exact block of YAML code below into the file.
```yaml
name: beginner-network-lab

topology:
  nodes:
    switch-1:
      kind: nokia_srlinux
      image: ghcr.io/nokia/srlinux:latest
    pc-1:
      kind: linux
      image: alpine:latest
  links:
    - endpoints: ["switch-1:e1-1", "pc-1:eth1"]
```

You will need to save the file once pasted.
In-line file editing in Linux requires saving files through `Ctrl+X` to save, and `Y`, then `Enter` to confirm.

---

## Part 3: Deploying the Network

Run these deployment steps inside your `~/labs/beginner-nac` workspace:

1. **Deploy the Topology:** Deploy your lab topology using Containerlab natively:
   ```bash
   sudo containerlab deploy --topo lab.clab.yml
   ```
   *Note: On your first execution, Containerlab must download the Docker images. This process can take 2-4 minutes depending on your internet connection speed.*

2. **Verify Container Health & View Management IPs:** Inspect the state of active nodes and management addresses:
   ```bash
   sudo containerlab inspect --topo lab.clab.yml
   ```

---

## Part 4: The Final Deliverable (Topology Visualization)

Your final requirement to earn credit for this lab is to generate and access the dynamic web-based topology map.

1. Launch Containerlab's web graphing engine directly using your topology file:
   ```bash
   sudo containerlab graph --topo lab.clab.yml
   ```
2. Open a standard web browser on your Windows host machine (Chrome, Edge, or Firefox).
3. Navigate to the address displayed in your terminal (typically **`http://localhost:50080`**).

### Assignment Completion Sign-off
Take a screenshot of your browser window showing the interactive, graphical mapping layout showing **router-alpha** and **router-beta** bound together by a virtual connection line. Submit this screenshot to fulfill your assignment requirements.

### Lab Clean up
When you are done experimenting, shut down and destroy the lab environment cleanly to release system resources back to Windows:
```bash
sudo containerlab destroy --topo lab.clab.yml
```

