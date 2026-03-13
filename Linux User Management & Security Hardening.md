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

## LINUX GROUP MANAGEMENT (hızlı özet)


### --- Grup Oluşturma ve İnceleme ---
sudo addgroup devteam          # 'devteam' adında yeni grup oluştur
cat /etc/group | grep devteam  # Grubun varlığını ve üyelerini kontrol et
getent group devteam           # Grup detaylarını (GID vb.) görüntüle

### --- Kullanıcıyı Gruba Ekleme (KRİTİK) ---
sudo usermod -aG devteam ubuntu # 'ubuntu' kullanıcısını devteam grubuna GÜVENLE ekle
# NOT: -a flag'ini unutma! Unutursan kullanıcı diğer tüm gruplardan atılır.

### --- Üyelik Doğrulama ---
groups ubuntu                  # Kullanıcının dahil olduğu tüm grupları listele
id ubuntu                      # UID ve tüm GID bilgilerini detaylı gör

### --- Grup Düzenleme ve Silme ---
sudo groupmod -n developers devteam # Grubu 'developers' olarak yeniden adlandır
sudo gpasswd -d ubuntu developers   # 'ubuntu' kullanıcısını gruptan çıkar
sudo delgroup developers            # Grubu sistemden tamamen sil

### --- Değişiklikleri Aktif Etme ---
newgrp developers              # Oturumu kapatmadan yeni grup yetkilerini yükle

---

# Linux Group-Based Access Control & Shared Environments

## 📖 Overview
In a multi-user Linux environment, managing access at the individual user level is unscalable and error-prone. This project demonstrates the implementation of **Role-Based Access Control (RBAC)** using Linux groups. By leveraging group ownership and secondary memberships, we create a secure, shared directory where access is strictly governed by team affiliation rather than individual identity.

## 🎯 Learning Objectives
- Implement shared directory structures with restricted access.
- Understand the impact of `Primary` vs. `Secondary` groups on file system permissions.
- Master the `chgrp` and `chmod` utilities to enforce administrative boundaries.
- Demonstrate the "Least Privilege" principle by isolating sensitive data from "Other" users.

## 🏗️ Architecture & Configuration Files
The Linux filesystem uses a 3-tier permission model (Owner, Group, Others). Managing access effectively requires shifting the "Group" tier from a default private group to a functional team group.

| Permission Field | Component | DevOps Context |
| :--- | :--- | :--- |
| `d` | Directory Flag | Indicates if the object is a folder/container. |
| `rwx` (1st set) | Owner | Full control for the creator/root. |
| `rwx` (2nd set) | Group | **Target Tier:** Shared access for specific team GIDs. |
| `---` (3rd set) | Others | **Security Boundary:** No access for non-team members. |

---

## 💻 Implementation & Hands-on Lab

### 1. Provisioning the Organizational Unit (Group)
First, we define the "Role" by creating a group. This serves as the single source of truth for access.
```bash
sudo addgroup programmer
```

### 2. Identity Provisioning
- We create two distinct users to test access: `prod_user` (authorized) and `dev_user` (initially unauthorized).
- sudo adduser prod_user
- sudo adduser dev_user

### 3. Implementing RBAC (Role-Based Access Control)
We assign `prod_user` to the` programmer group`. We use the `-aG` flag to append the group, ensuring the user doesn't lose their primary group membership.
```bash
sudo usermod -aG programmer prod_user
```
### 4. Shared Directory Configuration & Hardening
We create a directory and reassign its group ownership. We then modify the permissions to allow the group to write (g+w) while completely stripping access from everyone else (o-rx).
```bash
sudo mkdir /home/shared_folder
sudo chgrp programmer /home/shared_folder
sudo chmod g+w,o-rx /home/shared_folder
```

## ⌨️ Command Reference
### --- Group & User Setup ---
sudo addgroup programmer          # Create the functional group
sudo usermod -aG programmer <user> # Safely add user to the group

### --- Permission Management ---
sudo chgrp programmer /path/to/dir # Change group ownership
sudo chmod g+w /path/to/dir        # Grant write access to group members
sudo chmod o-rx /path/to/dir       # Remove read/execute from all others

### --- Verification ---
ls -l /home                        # Verify ownership and permission strings
id <user>                          # Verify current user group memberships

---

## 🛡️ Security & Best Practices (The DevOps Way)
- Least Privilege Principle: By using o-rx, we ensure that any user not explicitly added to the programmer group cannot even "peek" into the directory.
- Group-Level Auditing: Instead of auditing 100 users, DevOps engineers audit the membership of a single group (/etc/group) to determine who has access to production data.
- Separation of Concerns: Using /home/shared_folder keeps project-specific data outside of individual user home directories, preventing accidental data loss if a user account is deleted.

---

## 🚀 Real-World Production Scenario
- Scenario: Centralized Log Access for Developers
In a production cluster, you may have a directory /var/log/app_logs. You don't want developers to have sudo to read logs, nor do you want everyone on the server to see them.
Solution: Create a log-readers group, use chgrp on the log directory, and add only the authorized developers to that group. This allows for safe debugging without elevating system-wide privileges.

---

### 🧩 Troubleshooting & Common Pitfalls
## Problem: "I was added to the group but still get Permission Denied."
- Cause: Linux shell sessions do not update group memberships in real-time.
- Fix: The user must restart their session (logout/login) or run newgrp programmer to refresh the current shell's token.
## Problem: Users can't enter the directory even with Read (r) access.
- Cause: Missing Execute (x) permission on the directory.
- Fix: In Linux, you need x permission to "enter" or cd into a directory. Use chmod g+x.
