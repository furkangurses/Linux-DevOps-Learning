# 📂 Enterprise Linux Security: SSH/SCP Key-Based Authentication & Infrastructure Hardening

## 🎯 Problem Statement & Business Value
In high-traffic e-commerce platforms and Managed Service Provider (MSP) environments, relying on password-based authentication is a critical vulnerability. It exposes the infrastructure to brute-force attacks, credential stuffing, and internal lateral movement. If a single edge server is compromised due to a weak password, the enterprise risks catastrophic data breaches (violating PCI-DSS/GDPR), severe downtime, and loss of customer trust.

**The Engineering Goal:** Establish a highly secure, automated, and passwordless authentication mechanism using Asymmetric Cryptography (SSH Key Pairs). This enforces the principle of Least Privilege, secures machine-to-machine (M2M) communication (e.g., CI/CD pipelines, Jump Servers/Bastions), and encrypts data in transit via SCP.

## 🏗️ Architectural Overview & Deep-Dive
Under the hood, SSH (Secure Shell) and SCP (Secure Copy Protocol) operate over TCP port 22, establishing an encrypted tunnel. Key-based authentication uses a mathematical key pair. The **Private Key** never leaves the client and signs a cryptographic challenge. The **Public Key** resides on the remote server and verifies the signature. If they match, access is granted without transmitting any secrets over the network.

| Component / File Path | Location | Core Function & Architectural Role |
| :--- | :--- | :--- |
| `/etc/ssh/sshd_config` | Target Server | The main SSH daemon configuration file. Controls authentication methods, allowed users, and port bindings. |
| `~/.ssh/id_ed25519` | Client/Origin | The **Private Key**. Must be heavily restricted. Used to cryptographically prove identity. |
| `~/.ssh/id_ed25519.pub` | Client/Origin | The **Public Key**. Safe to share. Acts as the "lock" that only your private key can open. |
| `~/.ssh/authorized_keys` | Target Server | A list of approved Public Keys. If a client's key is here, they are granted access. |
| `~/.ssh/known_hosts` | Client/Origin | Stores the cryptographic fingerprints of remote servers to prevent Man-in-the-Middle (MITM) attacks. |

## 🛠️ Implementation Workflow (The DevOps Way)
1. **Daemon Hardening (Pre-requisite):** Modify `/etc/ssh/sshd_config` on the target servers to explicitly disable `PasswordAuthentication` and enable `PubkeyAuthentication`. This forces the system to reject any password attempts.
2. **Cryptographic Generation:** Generate a modern, secure key pair (ED25519 is preferred over RSA for performance and security) on the local machine or bastion host.
3. **Secure Payload Delivery:** Use `scp` to push the generated Public Key to the target server's temporary directory over an encrypted channel.
4. **Permission Sandboxing:** Ensure strict POSIX file permissions. The SSH daemon is designed to aggressively reject private keys that are readable by other users to prevent credential theft.
5. **Trust Establishment:** Append the Public Key to the target's `authorized_keys` file. Upon first connection, validate and accept the target server's fingerprint into `known_hosts` to establish a baseline of trust.

## 🤖 Automation Perspective
In an enterprise environment, manually copying keys using `scp` or `cat` is an anti-pattern (not scalable and prone to human error). 
* **Ansible:** We use the `ansible.builtin.authorized_key` module to idempotently distribute public keys to thousands of servers simultaneously. Ansible handles the SSH daemon reloads and file permissions automatically.
* **Terraform / Cloud-Init:** For ephemeral cloud infrastructure (AWS EC2), we define an `aws_key_pair` resource and inject the public key during the instance bootstrap phase using user-data. Instances boot up already secure and accessible only via CI/CD runners or Bastion hosts.

## 🛡️ Security Hardening & Compliance
This workflow directly aligns with **CIS (Center for Internet Security) Benchmarks** and **SOC2** compliance requirements:
* **Immutable Identity:** By disabling password authentication, we eliminate the risk of password sharing and weak credentials.
* **MITM Protection:** Strict `known_hosts` checking ensures we are connecting to the legitimate server, not an intercepted endpoint.
* **Auditing:** Every key-based login is logged in `/var/log/auth.log` (Ubuntu/Debian) or `/var/log/secure` (RHEL/CentOS) with the specific key fingerprint, ensuring non-repudiation and clear audit trails for security teams.

## ⌨️ Commands
```bash
# 1. Generate a modern SSH key pair (No passphrase for automated CI/CD use cases)
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""

# 2. Securely copy files/keys to a remote server using SCP
scp -i ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub ubuntu@<TARGET_IP>:~/

# 3. Restrict permissions on the Private Key (CRITICAL for SSH to accept it)
chmod 600 ~/.ssh/id_ed25519

# 4. Append Public Key to the authorized_keys file on the remote server
ssh ubuntu@<TARGET_IP> "cat ~/id_ed25519.pub >> ~/.ssh/authorized_keys"

# 5. Hardening SSH Daemon (Run on Target Server)
sudo sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/^#*PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
```
# 6. Apply daemon configuration changes
sudo systemctl restart ssh

---

## 🧩 Edge Cases & Troubleshooting
### Edge Case 1: `WARNING: UNPROTECTED PRIVATE KEY FILE!` (Permissions 0644 are too open)
- The Cause: The SSH daemon implements strict sanity checks. If your private key (`id_ed25519`) can be read by other users on the system, SSH will refuse to use it.
- The Senior Fix: Instantly apply the principle of least privilege. Run `chmod 600 ~/.ssh/id_ed25519`. Never bypass this warning.

### Edge Case 2: Host key verification failed. (StrictHostKeyChecking)
- The Cause: The remote server's cryptographic fingerprint does not match the one stored in your local `~/.ssh/known_hosts`. This happens if the server was rebuilt with the same IP address, or (worst case) a Man-in-the-Middle attack is occurring.
- The Senior Fix: If you deployed a fresh server on an old IP, purge the old signature using `ssh-keygen -R <TARGET_IP>`. Then reconnect to cache the new, valid fingerprint.

---

## 📊 Metrics & Monitoring
### To ensure the health and security of the SSH infrastructure:
- Log Aggregation: Stream `/var/log/auth.log` to a centralized SIEM (like ELK Stack or Datadog).
- Alerting Logic: Create PromQL queries or Grafana alerts to trigger on `rate(sshd_failed_logins_total[5m]) > 10`. A sudden spike in failed SSH logins indicates a potential brute-force attempt.
- Service Uptime: Monitor the `sshd` systemd process via Node Exporter to ensure the service hasn't crashed, which would lock out administrative access.

---

