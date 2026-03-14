# Linux Advanced Permissions & The Sticky Bit: Secure Shared Environments

## 📖 Overview
In multi-tenant or collaborative Linux environments (such as CI/CD nodes, bastion hosts, or shared developer servers), managing file access goes beyond standard read/write permissions. This documentation details the architectural approach to securing shared directories using strict User/Group ownership bindings (`chown`) and special permission flags, specifically the **Sticky Bit**. This setup ensures a collaborative workspace where team members can create and edit files, but are cryptographically prevented from modifying or deleting each other's assets.

---

## 🎯 Learning Objectives
- **Solve Production Vulnerabilities:** Prevent accidental data loss in shared directories where multiple users have write access.
- **Implement Filesystem RBAC:** Map organizational roles (Team Lead, Developer, Outsider) to native Linux `rwx` permissions.
- **Master Directory Execution Contexts:** Understand why directory permissions (`read`, `write`, `execute`) behave fundamentally differently than file permissions.
- **Deploy Advanced Security Bits:** Apply and diagnose the **Sticky Bit** (`+t` / `T`) to enforce strict deletion boundaries.

---

## 🏗️ Architecture & Configuration Files
Linux handles permissions at the kernel level. Understanding the difference between a standard directory and a hardened directory is crucial for compliance and security audits.

| Permission State | Diagnostic Output (`ls -ld`) | Explanatory Context | Architectural Impact |
| :--- | :--- | :--- | :--- |
| **Standard Shared** | `drwxrwx---` | Owner & Group have full access. Others have none. | **Vulnerable:** Any group member can delete any file within the directory. |
| **Active Sticky Bit (Passive Others)** | `drwxrwx--T` | Sticky bit enabled. Others lack `execute` (x). | **Secure & Private:** Files can only be deleted by their creators. Outsiders are blocked. |
| **Active Sticky Bit (Public Others)**| `drwxrwxrwt` | Sticky bit enabled. Others have `execute` (x). | **Secure & Public:** Standard for `/tmp`. Everyone can write, but isolation is enforced. |

---

## 💻 Implementation & Hands-on Lab

### 1. Identity & Workspace Provisioning
We start by creating a secure landing zone (`/home/shared`) and immediately decoupling it from the default `root:root` ownership.
```bash
# Escalate privilege to configure system-level directories
sudo su -

# Provision the directory and apply Role-Based Access Control (RBAC)
sudo mkdir -p /home/shared
sudo chown alex:developers /home/shared

# Grant Full Access to the Owner (7) and Group (7), strip access from Others (0)
sudo chmod 770 /home/shared
```

- Why: By mapping the directory to the `developers` group, any user added to this group dynamically inherits access, eliminating the need to manage permissions on a per-user basis.

---

### 2. File-Level Collaboration Hardening
- When `alex` creates a file, the default group ownership is usually their primary group (alex). To allow the team to collaborate, we must bind the file to the shared group.

```bash
# Change both the User and Group ownership of a specific file
sudo chown alex:developers /home/shared/thinknix

# Grant read/write to the group (6) while maintaining owner rights (6)
chmod 660 /home/shared/thinknix
```
---

### 3. Mitigating the Cross-Deletion Vulnerability
- Because the `developers` group has `write` access to `/home/shared`, a junior developer (`bob`) could accidentally run `rm *` and delete alex's critical files. Directory write permissions override file ownership. We mitigate this using the Sticky Bit.

# Apply the Sticky Bit to the directory
sudo chmod +t /home/shared

```bash
# Verification
ls -ld /home/shared
# Expected Output: drwxrwx--T (Capital T indicates Sticky Bit is ON, Others lack Execute)
```

- Why: The Sticky Bit forces the Linux kernel to perform an ownership check during the unlink (delete) system call. Only the file owner, the directory owner, or root can now delete the file.

## ⌨️ Command Reference
```bash
# --- Ownership Management ---
sudo chown user:group /path/to/target  # Reassign both User and Group
sudo chown :groupname /path/to/target  # Reassign ONLY the Group (equivalent to chgrp)

# --- Numeric Permissions (Octal) ---
sudo chmod 770 /dir                    # rwx for User/Group, --- for Others
sudo chmod 660 file.txt                # rw- for User/Group, --- for Others

# --- Security Bits ---
sudo chmod +t /dir                     # Enable Sticky Bit
sudo chmod -t /dir                     # Disable Sticky Bit

# --- Auditing ---
ls -ld /dir                            # Inspect permissions of the directory itself
```
---

## 🛡️ Security & Best Practices (The DevOps Way)
- The Principle of Least Privilege (PoLP): Never grant `777` permissions to solve an "access denied" error. If a pipeline fails to write a file, fix the group memberships or apply targeted numeric modes like `770` or `775`.
- Immutable User Contexts: Standard users cannot execute `chown` to give away files. This prevents users from bypassing storage quotas or shifting accountability for malicious scripts. `chown` is strictly a `root`/`sudo` operation.
- The `/tmp` Architecture: The Sticky Bit is the exact mechanism that prevents total system chaos in the `/tmp` directory. Always apply `+t`to any custom cache or artifact directories shared by multiple CI runners or application services.

---

## 🚀 Real-World Production Scenario

### Scenario: Secure CI/CD Artifact Storage
- In a Jenkins or GitLab CI environment, multiple pipeline runners (operating under different user accounts like `runner-01`, `runner-02`) write build artifacts (`.jar`, `.tar.gz`) to a centralized Network File System (NFS) mount at `/mnt/build_artifacts`.
- If the directory simply has `770` permissions, a poorly written cleanup script in `runner-02` could accidentally delete `runner-01's` artifacts. By configuring the directory with `chown` `root:ci-runners` and `chmod 1770` (the `1` sets the Sticky Bit), you guarantee that runners can dump their artifacts and read others, but can only execute cleanup operations on their own files.

---

## 🧩 Troubleshooting & Common Pitfalls
- Problem: The `ls -l` output shows a capital `T` instead of a lowercase `t`. Is it broken?
- Fix: No. A capital `T` indicates the Sticky Bit is active, but the Others permission block lacks the `execute` (`x`) bit. For private team directories, a capital `T` is actually the desired, more secure state. Do not arbitrarily run `chmod o+x` just to get a lowercase `t`.

- Problem: "Operation not permitted" when running `chown`.
- Cause: A regular user (even the file owner) is trying to change the file's ownership.
- Fix: Prepend the command with `sudo`. Only root can reassign ownership to prevent security circumvention.

- Problem: User has `read` access to a directory but gets "Permission Denied" when trying to `cd`into it.
- Cause: The user lacks `execute` (`x`) permissions on the directory. In Linux, `read` allows listing file names, but `execute` is required to traverse or enter the directory.
- Fix: Run `chmod g+x /directory` (if they are in the group).





---





# Linux Advanced IAM: Special Permissions, ACLs, and Umask Hardening

## 📖 Overview
Standard POSIX permissions (`rwx`) often fall short in complex, multi-tenant production environments. This project explores the implementation of advanced Identity and Access Management (IAM) on Linux. We cover **Special Permissions** (SUID, SGID, Sticky Bit) for temporary privilege elevation and group inheritance, **Access Control Lists (ACLs)** for granular per-user overrides, and **Umask** for defining system-wide security baselines for new assets.

---

## 🎯 Learning Objectives
- **Privilege Management:** Implementing SUID to allow restricted users to execute specific binary tasks with root-level authority without granting full sudo access.
- **Collaborative Integrity:** Utilizing SGID on directories to ensure automatic group-ownership inheritance, preventing "permission drift" in shared developer environments.
- **Granular Access Control:** Bypassing the "User-Group-Others" limitation using ACLs to grant specific, temporary access to individual third-party users.
- **Security Baselining:** Configuring `umask` to enforce a "Secure by Default" posture for all newly created files and directories.

---

## 🏗️ Architecture & Configuration Files
Special permissions add a fourth digit to the octal permission set, fundamentally changing how the Linux kernel handles process execution and file creation.

| Special Bit | Octal Value | Symbol | Target | Function |
| :--- | :--- | :--- | :--- | :--- |
| **SUID** (Set User ID) | 4 | `s` (Owner) | Executable File | Runs the file with the permissions of the **Owner**. |
| **SGID** (Set Group ID) | 2 | `s` (Group) | Directory/File | Inherits the **Group** of the parent directory. |
| **Sticky Bit** | 1 | `t` (Others) | Directory | Only the **File Owner** can delete/rename assets. |

### Configuration Path
- **User Environment:** `~/.bashrc` (Used for persistent `umask` settings).
- **System Binaries:** `/usr/bin/` (Location of SUID binaries like `passwd`).

---

## 💻 Implementation & Hands-on Lab

### 1. SUID: Temporary Privilege Elevation
Standard users cannot write to `/etc/shadow`. However, the `passwd` utility must write to it. We analyze how the SUID bit allows a user (`alex`) to trigger a root process.
```bash
# Verify SUID bit on the passwd binary
ls -l /usr/bin/passwd
# Output: -rwsr-xr-x  (The 's' indicates SUID is active)
```

### 2. SGID: Enforcing Group Consistency
- In shared directories, files created by different users often end up with mismatched group owners. SGID solves this by forcing inheritance.

```bash
# Apply SGID to the shared directory
sudo chmod g+s /home/shared

# Result: Any file created by Alex or Bob will automatically belong to 'developers'
```

### 3. ACLs: Granular Per-User Permissions
- When a user (`charlie`) who is not in the `developers` group needs access to a single directory, we avoid the security risk of adding him to the group or opening permissions to "Others."

### 4. Umask: Defining Default Security
- By default, Linux uses a `umask` (e.g., `0002`) to subtract permissions from a base (Files: `666`, Dirs: `777`). We harden this to ensure new files are private.

```bash
# Harden umask to '0077' (No access for Group or Others)
umask 0077

# Resulting New File: 666 - 077 = 600 (rw-------)
# Resulting New Directory: 777 - 077 = 700 (rwx------)
```
---

## ⌨️ Command Reference

```bash
# --- Special Permissions ---
chmod 4755 <file>        # Set SUID (4)
chmod 2770 <dir>         # Set SGID (2)
chmod 1777 <dir>         # Set Sticky Bit (1)

# --- Access Control Lists (ACL) ---
getfacl <path>           # View detailed permissions
setfacl -m u:user:rwx <path>  # Modify/Add ACL for a user
setfacl -x u:user <path> # Remove ACL entry for a user

# --- Umask (Defaults) ---
umask                    # Check current session mask
umask 0027               # Set session mask (Files: 640, Dirs: 750)
```

---

## 🛡️ Security & Best Practices (The DevOps Way)
- Audit SUID Binaries: Regularly run `find / -perm /4000` to identify SUID binaries. Maliciously set SUID bits are a primary vector for Privilege Escalation attacks.
- Avoid 777 Permissions: Use ACLs instead of opening directories to "Others." This maintains the Principle of Least Privilege.
- Persistent Hardening: Always define `umask` in `~/.bashrc` or `/etc/profile` to ensure that automated scripts do not create over-privileged logs or artifacts.

---

## 🚀 Real-World Production Scenario
### Automated Project Onboarding:
- An MSP manages a shared server for multiple client teams. When a new "Contractor" user joins, they need access to the `/projects/client_alpha` folder. Instead of modifying the primary group ownership (which would break the existing CI/CD automation), the DevOps engineer uses `setfacl` to grant the contractor specific access and sets a Default ACL on the directory so that any future files created by the automated build system are also readable by that contractor.

---

## 🧩 Troubleshooting & Common Pitfalls

- Persistence Issue: `umask` settings disappear after logout.
- Fix: Add `umask 0077` to the user's `~/.bashrc` and run `source ~/.bashrc`.

- The "Mask" in ACL: Sometimes `getfacl` shows a "mask" entry that restricts permissions even if the user has `rwx`.
- Fix: Ensure the ACL mask is set high enough to allow the user's permissions: `setfacl -m m:rwx <path>`.

- SUID on Scripts: SUID typically only works on compiled binaries in Linux; it is ignored on most shell scripts for security reasons.
