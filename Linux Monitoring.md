# 📂 System & Network Observability: Real-Time Performance Diagnostics

## 🎯 Problem Statement & Business Value
- **Business Impact:** In high-traffic e-commerce ecosystems, infrastructure blind spots directly translate to extended Mean Time to Recovery (MTTR) during outages. Inability to instantly diagnose CPU, Memory, Disk I/O, or Network bottlenecks leads to degraded user experiences, cart abandonment, and severe SLA breaches.
- **Engineering Goal:** Establish a deterministic, real-time diagnostic framework across cloud fleets to ensure **High-Availability** and rapid root-cause analysis (RCA) without relying purely on delayed aggregated metrics.

## 🏗️ Architectural Overview & Deep-Dive
Under the hood, Linux performance tools act as user-space parsers for the Kernel's pseudo-filesystems (primarily `/proc` and `/sys`). Tools like `free` and `vmstat` do not physically scan RAM; they parse in-memory kernel structures like `/proc/meminfo` and `/proc/vmstat` to deliver low-latency telemetry. 

| Diagnostic Tool | Core Kernel Source | Primary Metric Focus | Enterprise Use Case |
| :--- | :--- | :--- | :--- |
| **`ntopng`** | Libpcap / Netlink | L7 Traffic Flow & Bandwidth | Identifying malicious inbound IPs or L7 DDoS patterns. |
| **`vmstat`** | `/proc/vmstat` | Virtual Memory & CPU Wait | Correlating memory paging with CPU wait times. |
| **`iostat`** | `/proc/diskstats` | Block Device I/O & IOPS | Diagnosing AWS EBS volume bottlenecks (e.g., `xvda`). |
| **`iftop`** | Raw Network Sockets | Interface Throughput | Real-time tracking of outbound/inbound TX/RX spikes. |

## 🛠️ Implementation Workflow (The DevOps Way)
1. **Dependency Resolution (Cause):** We install `software-properties-common` and `wget`. *(Effect):* Standardizes the repository management layer across Debian-based fleets, ensuring consistent package fetching.
2. **Repository Injection (Cause):** We add the universe repository and the official `ntop.org` apt lists. *(Effect):* Bypasses outdated default OS packages, ensuring we install the latest, patched versions of observability tools.
3. **Execution & Service Orchestration (Cause):** We install `ntopng` and verify via `systemctl`. *(Effect):* Registers the tool as a `systemd` daemon, ensuring it survives system reboots and adheres to the OS lifecycle.
4. **Cloud Network Access (Cause):** We modify the AWS Security Group to allow inbound TCP 3000. *(Effect):* Opens the necessary ingress path to the web UI, bridging the cloud network boundary with the EC2 instance.

## 🤖 Automation Perspective
In an enterprise environment, executing these commands manually is an anti-pattern. We enforce state via **Infrastructure as Code (IaC)**:
- **Terraform:** Used to define the AWS Security Group ingress rules dynamically, tying the `ntopng` port (3000) strictly to corporate VPN CIDR blocks.
- **Ansible:** A `monitoring-agent` role is deployed across the fleet. The `ansible.builtin.apt_repository` module handles the ntop repository, while `ansible.builtin.apt` installs `sysstat` (for iostat/vmstat) and `iftop`. Services are ensured to be `state: started` and `enabled: yes`.

## 🛡️ Security Hardening & Compliance
- **Least Privilege Access (Network):** By default, `ntopng` listens on `0.0.0.0:3000`. Exposing this to the public internet (`0.0.0.0/0`) violates SOC2 and CIS benchmarks. The AWS Security Group MUST restrict inbound traffic exclusively to the Bastion host or internal VPC subnets.
- **Credential Rotation:** The default `admin:admin` credentials in `ntopng` pose an immediate critical security risk. Enterprise setups must automate the initial password change via API or configuration injection during bootstrapping.
- **Kernel Capability Restrictions:** Tools like `iftop` require raw socket access. Instead of granting blanket `sudo` privileges to junior engineers, we grant specific Linux capabilities (e.g., `setcap cap_net_raw,cap_net_admin=eip /usr/sbin/iftop`) to adhere to the principle of least privilege.

## ⌨️ Commands
```bash
# ==========================================
# 1. NETWORK OBSERVABILITY (ntopng & iftop)
# ==========================================
# Install dependencies and official ntop repository
sudo apt-get install software-properties-common wget
wget -q [http://apt.ntop.org/24.04/all/apt-ntop.deb](http://apt.ntop.org/24.04/all/apt-ntop.deb)
sudo dpkg -i apt-ntop.deb
sudo apt-get clean all && sudo apt-get update

# Install ntopng and iftop
sudo apt-get install pfring-dkms ntopng n2disk cento iftop -y

# Verify service state
sudo systemctl status ntopng

# Monitor live interface traffic (disable DNS resolution for speed)
sudo iftop -n -N -i enX0

# ==========================================
# 2. SYSTEM OBSERVABILITY (Memory, CPU, I/O)
# ==========================================
# Install sysstat (contains iostat, vmstat)
sudo apt-get install sysstat -y

# View Active vs Inactive memory
vmstat -a

# View comprehensive disk I/O stats (updates every 5 seconds, 2 times)
iostat -x -d 5 2

# Watch real-time human-readable memory usage (updates every 2 seconds)
watch -n 2 free -h
```

## 🧩 Edge Cases & Troubleshooting

1. **Edge Case 1: `iftop` throws "Operation not permitted"**
   - *Symptom:* Executing `iftop` as a standard user fails instantly.
   - *Senior Fix:* `iftop` needs to sniff packets at the data link layer. You must either execute it via `sudo` or, for secure automated environments, assign the correct binary capabilities (`sudo setcap cap_net_raw,cap_net_admin=eip $(which iftop)`).

2. **Edge Case 2: High System Load but CPU is mostly `idle`**
   - *Symptom:* Users report slow APIs, `uptime` shows load > 10, but `top` shows 80% CPU idle.
   - *Senior Fix:* Run `iostat -x 2`. Check the `%iowait` column and `await` times. The CPUs are idling because they are blocked waiting for the storage subsystem (EBS volume). The architectural fix is not to add more CPU, but to upgrade the EC2 instance to an EBS-optimized type or switch the volume from `gp2` to `gp3` with higher provisioned IOPS.

## 📊 Metrics & Monitoring
While command-line tools are vital for active incident response, long-term health is tracked via centralized telemetry:
- **Node Exporter:** Deployed via DaemonSet/Ansible to expose `/proc/vmstat` and `/proc/diskstats` as Prometheus metrics.
- **PromQL Alerting:** - Trigger P1 alerts if `rate(node_disk_io_time_seconds_total[5m]) > 0.8` (indicating a disk is saturated 80% of the time).
  - Trigger P2 alerts if `node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.1` (indicating less than 10% memory available for application scaling).
 
---

