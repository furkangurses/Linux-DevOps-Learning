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

# 📂 Enterprise Linux Observability: Advanced Resource Profiling & Systemd Log Analysis

## 🎯 Problem Statement & Business Value
In high-traffic e-commerce and enterprise MSP environments, system degradation and silent failures are inevitable. Without granular resource profiling and centralized, queryable logs, identifying the root cause of an outage—such as a memory leak, a noisy neighbor VM, or a failing kernel module—becomes guesswork. This lack of visibility directly translates to prolonged Mean Time To Recovery (MTTR), missed Service Level Agreements (SLAs), and severe revenue loss during critical events like Black Friday. 

The ultimate engineering goal is **High-Availability and Rapid Incident Response**. By mastering `sar` (System Activity Reporter) and `journalctl`, engineers shift from reactive firefighting to proactive observability, ensuring infrastructure bottlenecks are identified before they impact the end-user.

## 🏗️ Architectural Overview & Deep-Dive
Under the hood, these tools operate differently but complement each other to form a complete picture of system health. `sar` relies on the `sysstat` daemon to collect counters from the virtual filesystem (`/proc`) and saves them in binary format (usually in `/var/log/sysstat/` or `/var/log/sa/`). `journalctl` interfaces with the `systemd-journald` daemon, which captures syslog messages, kernel logs, and systemd service logs, storing them in a secure, indexed binary format in `/var/log/journal/`.

| Feature / Component | `sar` (sysstat) | `journalctl` (systemd-journald) | Which to use when? |
| :--- | :--- | :--- | :--- |
| **Data Source** | Kernel counters (`/proc/stat`, `/proc/meminfo`) | Service outputs, kernel ring buffer, syslog | Use `sar` for performance metrics; `journalctl` for application/system events. |
| **Data Format** | Binary data (`/var/log/sa/saDD`) | Indexed binary data (`/var/log/journal/`) | Both prevent easy tampering compared to plain text logs. |
| **Primary Domain** | CPU, Memory, IO, Network Utilization | Service crashes, application logs, boot errors | `sar` answers "Is the server overloaded?", `journalctl` answers "Why did the service crash?". |
| **Historical Reach** | Dependent on cron (usually 28 days default) | Dependent on `journald.conf` limits | Use `sar` for capacity planning; `journalctl` for forensic post-mortems. |

## 🛠️ Implementation Workflow (The DevOps Way)
1.  **Baseline Establishment (Pre-check):** Before an incident occurs, we use `sar` to profile normal operation states. Knowing that idle CPU `%iowait` is usually 1% makes a spike to 25% meaningful.
2.  **Real-Time Triage (Execution):** When an alert triggers (e.g., High CPU), we run real-time `sar -u 1 5` to confirm the load, checking specific vectors like `%steal` (hypervisor resource throttling) or `%system` (kernel overload). 
3.  **Log Correlation (Verification):** Once the resource bottleneck is identified, we use `journalctl -f` and filter by priority (`journalctl -p 3 -b`) to pinpoint the exact failing service, permission denial, or hardware fault causing the resource spike.

## 🤖 Automation Perspective
In a modern enterprise, manually SSH-ing into nodes to run `sar` or `journalctl` is an anti-pattern used only as a last resort. 
* **Infrastructure as Code (IaC):** We use **Ansible** to ensure the `sysstat` package is installed and its data collection interval (`/etc/cron.d/sysstat`) is aggressively tuned (e.g., 1-minute intervals instead of 10) across fleets.
* **Centralized Logging:** We automate the shipping of `systemd-journald` logs using a daemonset like **Fluent Bit** or **Promtail**, forwarding the structured logs to an Elasticsearch or Loki cluster. This allows querying logs from 1,000 servers in a single dashboard.

## 🛡️ Security Hardening & Compliance
* **Immutable Infrastructure & Auditing:** The binary nature of both `sysstat` and `journald` makes them resistant to casual tampering by attackers. Tracking pseudo-access, failed SSH logins, and privilege escalations via `journalctl` is critical for SOC2 and PCI-DSS compliance.
* **Log Retention Policies:** To prevent denial-of-service (DoS) via disk exhaustion, we strictly enforce retention limits in `/etc/systemd/journald.conf` by setting parameters like `SystemMaxUse=1G` and `MaxRetentionSec=30day`, aligning with CIS Benchmarks.

## ⌨️ Commands

```bash
# ==========================================
# SYSTEM ACTIVITY REPORTER (SAR) COMMANDS
# ==========================================

# Display CPU usage in real-time (1-second interval, 5 iterations)
# Key metrics: %user, %system, %iowait (disk/network delay), %steal (VM hypervisor limit)
sar -u 1 5

# Display Memory usage statistics (3-second interval, 4 iterations)
# Key metrics: kbmemfree, kbmemused, %memused, kbbuffers, kbcached
sar -r 3 4

# ==========================================
# SYSTEMD JOURNAL (JOURNALCTL) COMMANDS
# ==========================================

# Tail system logs in real-time (Follow mode)
# Ideal for monitoring live service deployments and restarts
journalctl -f

# Display only the last 2 lines of the journal
journalctl -n 2

# Filter logs to show ONLY Errors, Critical, Alert, and Emergency (Priority 3 and above)
# Restricted to the current boot session (-b) to cut through historical noise
journalctl -p err -b
# OR use the numerical equivalent:
journalctl -p 3 -b
```

## 🧩 Edge Cases & Troubleshooting
- Edge Case 1: Running `sar` returns `Cannot open /var/log/sysstat/saXX: No such file or directory`.
- Senior Fix: The `sysstat` data collector is not running. The fix is not just creating a file. You must start and enable the service: `sudo systemctl enable --now sysstat`. Then, ensure the cron job in `/etc/cron.d/sysstat` is un-commented to generate historical data.

- Edge Case 2: `journalctl` commands are extremely slow, and the `/var/log/journal` directory is consuming hundreds of Gigabytes of disk space, triggering I/O alerts.
- Senior Fix: The journal has grown unchecked or corrupted. First, rotate and clear old logs immediately using `journalctl --vacuum-time=7d` or `journalctl --vacuum-size=1G`. Then, permanently fix the architecture by enforcing `SystemMaxUse` in `/etc/systemd/journald.conf` and restarting the daemon via `systemctl restart systemd-journald`.

---

## 📊 Metrics & Monitoring
To elevate this beyond manual command-line checks:
* **Prometheus Node Exporter:** We expose the underlying `/proc` metrics (that `sar` reads) to Prometheus.
* **Grafana Alerts:** We configure PromQL alerts to trigger Slack/PagerDuty incidents if CPU `%iowait` > 20% (indicating disk latency) or if CPU `%steal` > 10% (indicating our Cloud Provider's hypervisor is starving our VM of resources).
* **Log Alarms:** We configure our centralized log aggregator to alert us immediately if `journald` registers any `Priority 0` (Emergency) or `Priority 1` (Alert) messages matching critical systemd services (e.g., `kubelet.service`).

---

# 🖥️ Linux Performance Monitoring & System Observability

## 🎯 Problem Statement & Business Value
Monitoring system performance is more than just asking "What is the CPU usage?". In high-traffic e-commerce platforms or MSP (Managed Service Provider) environments, failing to identify where a bottleneck originates (Network, Disk, or RAM) within seconds leads to total system collapse and significant financial loss.

The goal of this project is to proactively monitor system health, diagnose anomalies, and optimize resource usage to ensure an uninterrupted user experience.

## 🏗️ Architectural Overview & Deep-Dive
Linux monitoring tools operate by reading metrics from the `/proc` and `/sys` virtual filesystems provided by the kernel. These tools are categorized into two types based on how they present data: **Point-in-time Reporting** and **Real-time Interactive Monitoring**.

| Component | Tool | Engineering Scope |
| :--- | :--- | :--- |
| **Network (Next-Gen)** | `ntopng` | Advanced web interface for deep traffic analysis and bandwidth monitoring. |
| **Memory (RAM)** | `free` | Quick summary of physical RAM and Swap usage. |
| **System Stats** | `vmstat` | Consolidated report of processes, memory, paging, block IO, and CPU activity. |
| **Disk & CPU IO** | `iostat` | Measures storage device read/write performance and CPU wait times. |
| **Process Loop** | `watch` | Makes static commands (e.g., `ls`, `df`) dynamic by running them periodically. |
| **Network Flow** | `iftop` | Displays active network connections and real-time bandwidth consumption per host. |

## 🛠️ Implementation Workflow (The DevOps Way)
1.  **Memory Health Check:** Check `cached` and `available` values via `free -m` to detect potential memory leaks.
2.  **Bottleneck Diagnosis:** If the system is sluggish, analyze whether CPU is consumed by system or user processes using `vmstat 1 5`.
3.  **IO Performance Audit:** Examine `iostat` reports to identify disk bottlenecks; high `iowait` values indicate a slow storage unit.
4.  **Real-time Interaction:** Integrate the `watch` command to monitor static reports live (e.g., `watch -n 1 iostat`).
5.  **Traffic Analysis:** Use the `iftop` interface to inspect instantaneous connections and catch unusual data flows.

## 🤖 Automation Perspective
In enterprise environments, data from these tools is automated:
* **Bash Scripting:** Scripts are written to send alerts when specific thresholds (e.g., 90% RAM usage) are exceeded in `free` or `iostat` outputs.
* **Metric Collection:** Data from these tools is gathered via agents like **Prometheus Node Exporter** and visualized on **Grafana** dashboards.

## 🛡️ Security Hardening & Compliance
* **Anomaly Detection:** Using `iftop` and `ntopng` helps detect unauthorized data exfiltration and suspicious external connections.
* **Audit Logging:** Regularly logging critical system performance data is essential for post-mortem analysis and compliance audits (e.g., SOC2, PCI-DSS).

## ⌨️ Commands
```bash
# Display memory usage in MB
free -m

# Report virtual memory statistics every 2 seconds
vmstat 2

# Report CPU and Disk I/O statistics
iostat

# Monitor disk usage changes every 1 second
watch -n 1 df -h

# Monitor real-time network bandwidth (Requires sudo)
sudo iftop

# Real-time network traffic analysis via ntopng
sudo systemctl start ntopng
```
## 🧩 Edge Cases & Troubleshooting
* **High CPU usage but low Process load:** If `top` shows no heavy processes but CPU is high, check `iowait` with `iostat`. The CPU might be waiting for the disk to respond.
* **Command not found:** `ntopng` or `iftop` might not be installed. **Senior Fix:** Ensure necessary packages (especially `sysstat`) are installed via `sudo apt update && sudo apt install ntopng iftop sysstat -y`.

## 📊 Metrics & Monitoring
Critical metrics to watch for a healthy system:
* **Memory Availability:** Available RAM should not drop below 10%.
* **CPU IOWait:** Consistently being above 20% indicates a disk performance issue.
* **Network Throughput:** Unexpected external IP traffic on `iftop` is a high-priority security alert.

