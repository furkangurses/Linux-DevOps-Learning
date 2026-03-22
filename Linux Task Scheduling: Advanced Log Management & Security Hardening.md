# 📂 Linux Task Scheduling: Advanced Log Management & Security Hardening

## 🎯 Problem Statement & Business Value
- **Business Impact:** In high-traffic e-commerce or MSP environments, applications and system services generate massive amounts of telemetry and log data. Failure to implement proactive cleanup leads to **Disk Exhaustion**, which triggers database corruption, application outages (Downtime), and direct revenue loss.
- **Engineering Goal:** To ensure **High Availability (HA)** and resource stability by implementing Zero-Touch automated cleanup of logs and temporary files, maintaining system hygiene without manual intervention.

## 🏗️ Architectural Overview & Deep-Dive
Linux task scheduling is primarily managed by two daemons: `crond` (recurring tasks) and `atd` (one-time tasks). These services rely on the kernel timer to trigger execution in a non-interactive subshell.

| Component / Concept | Path / Service | Architecture (Under the Hood) | Enterprise Use Case |
| :--- | :--- | :--- | :--- |
| **Cron (Recurring)** | `/var/spool/cron/` or `/etc/cron.d/` | Managed by `crond`. Based on periodic cycles (min, hour, day). | Daily DB backups, log rotation, metric collection. |
| **At (One-time)** | `/var/spool/at/` | Managed by `atd`. Executes once at a specific future timestamp. | Service restarts after a maintenance window. |
| **Find Filter** | `find /path -mtime +7` | Reads filesystem inode metadata to calculate the last modified time. | Identifying and purging logs older than 7 days. |

---

## 🛠️ Implementation Workflow (The DevOps Way)
1. **Script Isolation & Standardization:** Cleanup scripts (e.g., `cleanup_script.sh`) are never placed in arbitrary home directories. They are standardized in `/usr/local/bin/` or `/opt/scripts/` to ensure consistency across Dev, Staging, and Production environments.
2. **Logging & Error Handling (Cause and Effect):** We redirect both standard output (`stdout`) and standard error (`stderr`) to a dedicated log file (`>> /var/log/cleanup.log 2>&1`). This is critical because a cron job failing silently creates a "Silent Failure" syndrome, only discovered when the disk is already 100% full.
3. **Selective File Purging:** The `find` command is scoped with `-type f` and `-name` filters to ensure only specific log patterns are removed, preventing accidental deletion of directories or critical system config files.
4. **Scheduler Enrollment:** The task is registered using `crontab -e` or by dropping a configuration file into `/etc/cron.d/` for Infrastructure as Code (IaC) compliance.

---

## 🤖 Automation Perspective
In a true Enterprise/MSP environment, manually SSHing into a server to edit a crontab is a **security and compliance violation**. Configuration management tools are used to maintain idempotency.

**Ansible Example:**
Using the `cron` module allows an engineer to deploy this policy across 10,000+ servers simultaneously.

```yaml
- name: Deploy Application Log Cleanup Cron Job
  ansible.builtin.cron:
    name: "Enterprise App Log Cleanup"
    minute: "0"
    hour: "0" # Runs at midnight
    job: "find /var/log/app -type f -name '*.log' -mtime +7 -delete >> /var/log/cleanup_cron.log 2>&1"
    user: "appuser" # Security: Avoid running as root if possible
```
---

## 🛡️ Security Hardening & Compliance
- Least Privilege: While the demonstration uses `root` for simplicity, a Senior Engineer ensures scripts run under a service account (e.g., `appuser`) with restricted write access to specific log directories only.
- Access Control: Implement `/etc/cron.allow` to restrict cron access to authorized users only, aligning with CIS Benchmarks.
- Immutable Infrastructure: In modern cloud-native architectures, logs are streamed instantly to central platforms (ELK, Splunk, Datadog) via agents like Fluentd or Promtail. Storing logs on-disk is considered a legacy anti-pattern.

---

## ⌨️ Commands
```bash
# Verify and enable the scheduling services
sudo systemctl status atd
sudo systemctl enable --now atd

# Create a robust, production-ready cleanup script
cat << 'EOF' > /usr/local/bin/log_cleanup.sh
#!/usr/bin/env bash
set -euo pipefail # Exit on error, undefined vars, or pipe failure

LOG_DIR="/var/log/app"
CLEANUP_LOG="/var/log/cron_cleanup.log"

echo "[$(date)] Cleanup started..." >> "$CLEANUP_LOG"

# Optimized: Using -delete is faster than -exec rm {} \;
find "$LOG_DIR" -type f -name "think*.log" -mtime +7 -delete >> "$CLEANUP_LOG" 2>&1

echo "[$(date)] Cleanup finished." >> "$CLEANUP_LOG"
EOF

# Secure the script permissions
chmod 700 /usr/local/bin/log_cleanup.sh

# Schedule a one-time task using At
echo "/usr/local/bin/log_cleanup.sh" | at now + 5 minutes

# Inspect active schedules
crontab -l
atq
```
---

## 🧩 Edge Cases & Troubleshooting
### Edge Case 1: The "Vanishing Path" (Environment Variable Mismatch)
- Problem: Cron runs with a very minimal `PATH` (usually just `/usr/bin:/bin`). If your script calls custom binaries (e.g., `aws`, `docker`), it will fail with "command not found" even if it works manually in your terminal.
- Senior Fix: Always define the full `PATH` at the top of the script or use absolute paths for every command. Add `set -x` for debugging to capture exactly what the shell sees.

### Edge Case 2: Inode Exhaustion & "Argument list too long"
- Problem: If millions of small logs accumulate, `find ... -exec rm {} \;` forks a new process for every single file, potentially crashing the server via Process Fork Exhaustion.
- Senior Fix: Use the `-delete` flag within `find`. It performs the deletion internally within the find process, avoiding the overhead of forking `rm` thousands of times.

---

## 📊 Metrics & Monitoring
- Observability: Integrate with Prometheus Node Exporter using the `textfile_collector`. The script should output its success status to a `.prom` file.
- Alerting Logic: Set up a Grafana alert: `time() - log_cleanup_last_run_seconds > 86400`. If the cleanup hasn't run successfully in over 24 hours, trigger a PagerDuty alert for the DevOps team.

---

# 📂 Linux Observability: Advanced Logging, Auditing & Log Rotation

## 🎯 Problem Statement & Business Value
- **Business Impact:** In high-traffic e-commerce and MSP environments, logging is not just a "text file" task. Improperly configured logging mechanisms lead to two major risks: **Disk Exhaustion** (uncontrolled log growth filling the disk to 100%, crashing databases and applications) and **Security Breaches** (failure to monitor authentication logs, leaving cyber-attacks unnoticed).
- **Engineering Goal:** To establish a centralized, standardized, and automated **Log Lifecycle Management** for system logs, security metrics, and application outputs, ensuring **High Availability (HA)** and **Continuous Compliance**.

## 🏗️ Architectural Overview & Deep-Dive
Ubuntu and modern Linux distributions utilize a dual-layer logging architecture. Data from the kernel and services are processed via the modern `systemd-journald` (binary-based) and the traditional `rsyslog` (text-based) daemons.

Below is an enterprise-grade comparison of these architectural components:

| Component / Concept | Path / Service | Architecture (Under the Hood) | Enterprise Use Case |
| :--- | :--- | :--- | :--- |
| **rsyslog** | `/var/log/syslog`, `/etc/rsyslog.d/` | Traditional text-based log daemon. Routes data based on Facility (source) and Priority (severity). | Forwarding logs over UDP/TCP to a central Log Aggregator (SIEM). |
| **journald** | `systemd-journald` service | Stores metadata-rich logs in binary format. Fast indexing and retrieval. | Real-time debugging of microservices or crashed units via `journalctl`. |
| **Logrotate** | `/etc/logrotate.conf`, `/etc/logrotate.d/` | A cron-based tool for archiving, compressing, and purging old logs. | Managing retention policies (e.g., compress logs to `.gz` and delete after 30 days). |
| **Auth Logs** | `/var/log/auth.log`, `wtmp` | Records all authentication events (SSH, sudo) via PAM (Pluggable Authentication Modules). | Monitoring for Brute-Force attacks and fulfilling SOC2 audit requirements. |

## 🛠️ Implementation Workflow (The DevOps Way)
1. **Defining Log Routing Rules (Cause):** To prevent application logs from getting lost in the generic `syslog`, we write custom rules under `/etc/rsyslog.d/` (e.g., `local0.conf`). *(Effect)*: This isolation standardizes service logs and decouples them from system-level noise.
2. **Daemon Reloading (Cause):** After modifying configuration files, we run `systemctl restart rsyslog`. *(Effect)*: This flushes the daemon's memory and forces it to start writing to the new I/O targets (the new log files).
3. **Logrotate Configuration (Cause):** To prevent these isolated log files from consuming all disk space, we add rotation, compression (`compress`), and delay policies (`delaycompress`) to `/etc/logrotate.d/`. *(Effect)*: This transforms the system into a "Zero-Touch" self-cleaning infrastructure.
4. **Synthetic Log Injection & Testing (Cause):** We use `logger -p local0.info -t DEPLOY_TEST "Deployment started"`. *(Effect)*: This verifies that the routing and rotation mechanisms are working correctly before deploying to production.

## 🤖 Automation Perspective
In an enterprise environment, manually SSHing into servers to configure `rsyslog` or `logrotate` is unacceptable as it creates **Configuration Drift**. This process is automated using **Ansible** or **Terraform**.

* **The Ansible Approach:** The `template` module is used to push organization-specific logrotate rules from Jinja2 templates to target nodes.
* **Automation Logic:** The configuration is copied to `/etc/logrotate.d/`, and then the `ansible.builtin.systemd` module reloads the `rsyslog` service. This ensures thousands of nodes maintain the same logging policy in seconds.

## 🛡️ Security Hardening & Compliance
- **SOC2 & CIS Benchmarks:** Corporate security policies require real-time monitoring of `/var/log/auth.log` for successful/failed logins (`failed password`, `session opened for user root`). File permissions must be hardened (only `root` and `adm` groups should have read access).
- **Immutable Log Forwarding:** In modern security architectures, logs are not kept solely on the local disk. To prevent an attacker from "covering their tracks" by deleting logs, `rsyslog` is configured to forward logs instantly to a remote SIEM (Splunk, ELK) using TLS encryption (`*.* @@log-server.enterprise.com:514`).

## ⌨️ Commands
```bash
# Check service health (Pre-flight check)
sudo systemctl status rsyslog

# Inject a manual log with tags and priority for script debugging
logger -p user.warning -t "DEPLOY_PIPELINE" "CPU threshold warning"

# Real-time monitoring of a specific service's logs for the last hour
journalctl -u ssh --since "1 hour ago" -f

# List system errors with detailed explanations (-x) jumping to the end (-e)
journalctl -xe

# Filter PAM (Authentication) logs for failed SSH attempts
cat /var/log/auth.log | grep "Failed password"

# Test a logrotate configuration in debug mode (-d) without making changes
sudo logrotate -d /etc/logrotate.d/local0
```

## 🧩 Edge Cases & Troubleshooting

1. **Edge Case 1: Application Stops Logging After Rotation (File Descriptor Hang)**
   - *Problem:* When Logrotate renames a file (`app.log` to `app.log.1`) and creates a new one, a running application (e.g., Nginx) keeps the old inode open. Logs are not written to the new file, and disk space isn't reclaimed.
   - *Senior Fix:* Add a `postrotate` block to the Logrotate config to send a **SIGHUP** signal or reload the application (e.g., `postrotate /bin/systemctl reload nginx endscript`).

2. **Edge Case 2: `/var/log/journal` Consuming Entire Disk Volume**
   - *Problem:* `systemd-journald` can grow exponentially if left unconfigured. Juniors often run `rm -rf /var/log/journal/*`, which can corrupt the journal service.
   - *Senior Fix:* Never delete binary logs with `rm`. Use `journalctl --vacuum-time=7d` or `journalctl --vacuum-size=1G`. For a permanent fix, set `SystemMaxUse=1G` in `/etc/systemd/journald.conf` and restart the service.

## 📊 Metrics & Monitoring
- **Prometheus Node Exporter:** Monitor `/var/log` partition space using `node_filesystem_free_bytes`. Trigger alerts when disk usage exceeds 80%.
- **Log Scraping:** Use `Promtail` or `Filebeat` to ingest `/var/log/auth.log` into a Grafana Loki or ELK stack.
- **Alerting Logic:** "If more than 10 'Failed password' attempts originate from the same IP within 5 minutes, send a P1 Alert to the #secops Slack channel."
