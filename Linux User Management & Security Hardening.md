# Linux User Management & Security Hardening

## 📖 Overview
This project covers the fundamental mechanics of user management in Linux systems. It explores the critical system files that define user identity, security policies, and environment configurations, alongside the practical commands used to manage account lifecycles in a production environment.

---

## 🎯 Learning Objectives
- Master the structure and purpose of core identity files (`/etc/passwd`, `/etc/shadow`, etc.).
- Implement security hardening through password aging and account locking.
- Automate environment consistency using skeleton templates.
- Execute safe administrative privilege delegation via the Sudoers file.

---

## 🏗️ Architecture & Configuration Files
User management in Linux is centralized within four primary system files. Understanding their structure is vital for system security.

| File Path | Description | Access Level | Key Data Stored |
| :--- | :--- | :--- | :--- |
| `/etc/passwd` | User Account Details | Readable by all | Username, UID, GID, Home Dir, Shell |
| `/etc/shadow` | Secure Password Data | Root Only | Hashed Passwords, Expiry, Lock Status |
| `/etc/skel` | Skeleton Directory | System | Template files (`.bashrc`) for new users |
| `/etc/sudoers` | Privilege Rules | Root Only | Administrative permissions and sudo rules |



### 🆔 UID Standard Ranges
- **0**: Dedicated to the `root` (Superuser) account.
- **1 - 999**: System accounts and processes.
- **1000+**: Regular unprivileged users.

---

## 💻 Implementation & Hands-on Lab

### 1. User Creation and Environment Setup
New accounts are created using the `adduser` utility, which automates the creation of home directories and copies templates from `/etc/skel`.
```bash
# Create a new developer account
sudo adduser devuser
```
### 2. Privilege Escalation (The Safe Way)
Directly editing `/etc/sudoers` is dangerous. Use visudo to ensure syntax validation before saving.

```bash
# Edit sudoers file safely
sudo visudo

# Entry added for devuser:
devuser ALL=(ALL:ALL) ALL
```

### 3. Password Policy Enforcement
Enforcing password changes is a key security compliance requirement.

```bash
# Force password change on next login
sudo passwd -e devuser

# Set password to expire every 30 days
sudo chage -M 30 devuser
```

### 4. Modifying and Moving Accounts
Managing existing users often involves changing IDs or moving home directories across mount points.

```bash
# Move home directory and update /etc/passwd
sudo usermod -d /home/devhome -m devuser

# Update UID and Default Shell
sudo usermod -u 2000 -s /bin/zsh devuser
```
## ⌨️ Command Reference
```bash
# Identity & Inspection
id                   # Display current UID, GID, and Groups
cat /etc/passwd      # List system user accounts
sudo cat /etc/shadow # View encrypted password hashes

# Management
sudo adduser <user>  # Create user (interactive/Debian)
sudo userdel -r <user> # Remove user and their home directory
sudo usermod <args>  # Modify existing user attributes

# Security & Passwords
sudo passwd -l <user> # Lock an account password
sudo passwd -u <user> # Unlock an account password
sudo chage -l <user>  # List password aging details
```

## 🛡️ Security & Best Practices (The DevOps Way)
- Principle of Least Privilege (PoLP): Never log in as `root`. Use `sudo` for specific tasks to maintain an audit trail.
- Immutable Shells: Monitor `/etc/passwd` for unauthorized changes to default shells (e.g., a hacker changing a service account's shell to `/bin/bash`).
- Standardized Skels: Use `/etc/skel` to ensure every developer has the same aliases and security warnings in their terminal upon account creation.
- SSH Key Over Passwords: In cloud environments (AWS/GCP), password-based login is usually disabled (marked as `!` or `*` in `/etc/shadow`) in favor of SSH Key Pairs.

---

## 🚀 Real-World Production Scenario
Scenario: Automated Developer Onboarding
A DevOps engineer writes a script that:

- Creates a new user account based on a ticket.
- Adds them to the developers group.
- Sets a temporary password and flags it for immediate change (`passwd -e`).
- Moves pre-configured SSH public keys into the user's `.ssh` folder (derived from `/etc/skel`).
- Grants specific sudo permissions for restarting the Nginx service without granting full root access.

---

## 🧩 Troubleshooting & Common Pitfalls
### Problem: Sudo commands fail after editing sudoers.
- Cause: Manual edit of `/etc/sudoers` with a syntax error.
- Fix: Boot into recovery mode and fix using `visudo`. Always use `visudo` to prevent this.
### Problem: User cannot log in after account creation.
- Check: Look at `/etc/shadow`. If you see `!!`, the password was never set or the account is locked. Use `sudo passwd <user>` to set it.
### Problem: "Home directory not found" after usermod.
- Fix: Ensure the `-m`flag was used during `usermod -d` to actually move the files to the new path.

---

## 🛠️ Tools & Technologies

`Linux` `Ubuntu` `Bash` `Security` `IAM` `User-Management`
