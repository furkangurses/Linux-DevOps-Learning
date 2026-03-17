# 📂 Linux Storage Management: EBS Provisioning, Partitioning & Persistent Mounts

## 🎯 Problem Statement & Business Value
In cloud environments (specifically AWS), managing storage volumes correctly is critical for system continuity and data integrity. Keeping application data on the same volume as the Operating System (Root/Boot disk) is a high-risk practice; log overflows or unexpected data growth can lead to disk exhaustion, causing the entire system to crash (Downtime).

**Engineering Goal:** Implement the **Separation of Concerns** principle by isolating data on separate, scalable, and high-performance EBS (Elastic Block Store) volumes. Furthermore, ensure system reliability by configuring `/etc/fstab` to guarantee consistent and automatic mounting of disks across system reboots.

## 🏗️ Architectural Overview & Deep-Dive
The transition from a raw block volume to a usable directory in Linux involves three primary architectural layers:

1.  **Block Device Layer:** The Kernel recognizes the hardware (or virtual hardware) via device paths (e.g., `/dev/xvd*`).
2.  **Partition & Filesystem Layer:** Defining logical boundaries and establishing the data indexing architecture (XFS, EXT4). Linux reserves `2048 sectors` at the start of the disk for metadata (inodes, partition tables).
3.  **VFS (Virtual File System) Layer:** Mapping the created filesystem into the Linux directory hierarchy (`/`) via mounting.

| Tool / Concept | Description | When to Use? |
| :--- | :--- | :--- |
| `lsblk` | Lists block devices in a tree structure. | For quick overview, identifying new disks, and verifying mount status. |
| `fdisk` | Creates and manipulates partition tables. | To divide physical disks into logical segments (Primary/Extended). |
| `blkid` | Displays UUIDs and filesystem (FS) types. | Essential for obtaining unique IDs for `/etc/fstab` configuration. |
| `/etc/fstab` | A configuration file that automates mounting. | To ensure disks are mounted automatically after a reboot. |

## 🛠️ Implementation Workflow (The DevOps Way)
1.  **Discovery & Provisioning:** The EBS volume is created in AWS and attached to the EC2 instance. Use `lsblk` to verify the device path (e.g., `/dev/xvdf`).
2.  **Partitioning:** The disk is partitioned using `fdisk`. This allows for logical isolation of different data types (e.g., separating DB files from logs).
3.  **Formatting (Filesystem Creation):** Partitions are initialized with `mkfs.xfs`. This defines the metadata structure and assigns a unique **UUID** to the disk.
4.  **Mount Point Creation:** A target directory (Mount Point) is created in the Linux hierarchy (e.g., `mkdir /data`).
5.  **Persistence Registration:** The **UUID**, mount point, and filesystem parameters are appended to `/etc/fstab`. Run `mount -a` to validate the syntax immediately.

## 🤖 Automation Perspective
In a production Enterprise or MSP environment, these steps are never performed manually. Infrastructure changes are managed via **Infrastructure as Code (IaC)**.

**Terraform & Ansible Scenario:**
* **Terraform:** Utilizes `aws_ebs_volume` and `aws_volume_attachment` resources to provision and attach the disk to the instance.
* **Ansible:** * The `community.general.parted` module partitions the disk.
    * The `community.general.filesystem` module formats it as XFS.
    * The `ansible.posix.mount` module adds the entry to `/etc/fstab` and mounts it, ensuring the process is **idempotent** and standardized.

## 🛡️ Security Hardening & Compliance
* **Immutable Names (Use of UUID):** Never use device paths like `/dev/xvdf1` in `/etc/fstab`. In cloud environments, these paths can change after a reboot. Always use the **UUID** provided by `blkid`.
* **Decommissioning (Secure Disposal):** When a disk is no longer needed, it must be `umounted` and removed from `/etc/fstab` before being detached. For high-security environments, partitions should be wiped to ensure data at rest is not recoverable.
* **FSTAB Options:** Enhance security by adding flags like `nodev`, `nosuid`, or `noexec` to the `/etc/fstab` entry to prevent unauthorized binary execution or setuid escalations (CIS Benchmark standard).

## ⌨️ Commands

```bash
# 1. List disks hierarchically (Identify the newly added disk)
lsblk

# 2. Start the partitioning tool (Replace X with the correct letter)
fdisk /dev/xvdX
# Internal commands: 'n' (new), 'p' (primary), 'w' (write/save)

# 3. Create a filesystem (XFS) on the partition (Generates the UUID)
mkfs.xfs /dev/xvdX1

# 4. Retrieve the UUID of the disk
blkid /dev/xvdX1

# 5. Create the mount point directory
mkdir /mount_point

# 6. Append persistent entry to /etc/fstab
# Format: <UUID> <mount_point> <fs_type> <options> <dump> <pass>
echo "UUID=1234abcd-12ab-34cd-56ef-1234567890ab /mount_point xfs defaults 0 2" >> /etc/fstab

# 7. Notify systemd of fstab changes
systemctl daemon-reload

# 8. Mount all entries in fstab and check for errors
mount -a

# 9. Verify successful mount
df -h
```

## 🧩 Edge Cases & Troubleshooting

* **Edge Case 1: System Fails to Boot (Emergency Mode) Due to `/etc/fstab` Syntax Error**
    * **Scenario:** An incorrect UUID or a typo was entered into the configuration. The server hangs during the boot process because it cannot find the critical resource.
    * **Senior Fix:** Always run `mount -a` immediately after editing `fstab`. If the system is already in "Emergency Mode," log in as root, remount the root filesystem as writable using `mount -o remount,rw /`, and correct the entry. Pro-tip: Use the `nofail` option in `fstab` for non-critical data volumes to allow the system to boot even if the disk is missing.

* **Edge Case 2: Disk is Attached in Cloud Console but Invisible in OS**
    * **Scenario:** AWS console shows "Attached," but `lsblk` does not show the new device. This usually happens due to a stuck SCSI bus in older kernels or specific virtualization types.
    * **Senior Fix:** Trigger a manual scan of the SCSI host without rebooting the instance:
        ```bash
        for host in /sys/class/scsi_host/host*; do echo "- - -" > $host/scan; done
        ```
        This forces the kernel to recognize new hardware on the fly.

## 📊 Metrics & Monitoring

In a high-traffic e-commerce environment, storage health is a tier-1 metric. We monitor the following:

* **Capacity Monitoring (Prometheus/Grafana):**
    * **Metric:** `node_filesystem_avail_bytes` / `node_filesystem_size_bytes`
    * **Alert Logic:** Trigger a *Warning* at 80% usage and *Critical* (PagerDuty/Slack) at 90%.
* **I/O Performance:**
    * **Metric:** `node_disk_io_time_seconds_total`
    * **Insight:** Monitoring "Disk Wait" time to detect EBS throttling or volume bottlenecks.
* **Inodes Monitoring:**
    * **Metric:** `node_filesystem_files_free`
    * **Criticality:** In high-transaction systems creating many small files (sessions/cache), you might run out of Inodes even if disk space is available.
 
---

## ⌨️ Disk Management Command Reference

| Command | Description | Practical Use Case |
| :--- | :--- | :--- |
| `lsblk` | **List Block Devices:** Displays all available disks, partitions, and their mount points in a tree format. | Identifying newly attached EBS volumes and checking hierarchy. |
| `blkid` | **Block ID:** Displays the Universally Unique Identifier (UUID) and filesystem type of partitions. | Essential for configuring persistent mounts in `/etc/fstab`. |
| `fdisk -l` | **List Partition Tables:** Provides a detailed technical summary of all disks and their partition boundaries. | Verifying partition types (Primary/Extended) and sector alignment. |
| `df -h` | **Disk Free (Human-Readable):** Shows the disk space usage, total size, and availability of all *mounted* filesystems. | Monitoring storage capacity to prevent application crashes. |
| `fdisk /dev/sdx` | **Partitioning Utility:** Opens the interactive dialog-driven menu to create, delete, or modify partitions. | Dividing a raw EBS volume into manageable logical sections. |
| `mkfs.<type> <path>`| **Make Filesystem:** Formats a partition with a specific filesystem (e.g., `mkfs.xfs` or `mkfs.ext4`). | Initializing a partition so it can store files and directories. |
| `mount -a` | **Mount All:** Automatically mounts all filesystems defined in the `/etc/fstab` configuration file. | Testing persistence settings without needing to reboot the server. |
| `umount <path>` | **Unmount:** Detaches the filesystem from the directory tree. | Safely preparing a disk for detachment or decommissioning. |

---

## 🧠 Core Engineering Concepts

### 1. The Partitioning Logic
* **Primary Partition:** Maximum of **4** allowed per disk on MBR.
* **Extended Partition:** A "container" used to bypass the 4-partition limit. It holds **Logical Partitions**.
* **Metadata Reservation:** Linux rezerve **2048 sectors** (approx. 1MB) at the start of each partition for the **Partition Table** and **Inode Table**. 
    * *Sector:* The smallest unit of data (typically 512 bytes).



### 2. Filesystem & Access
* **Formatting:** A partition is unusable until a filesystem (XFS/ext4) is created. This process builds the structure required for data storage.
* **Mount Point:** A directory (access point) created within the Linux root (`/`) where the disk’s data is "plugged in."
* **Persistence:** Temporary mounts disappear after reboot. **Permanent mounts** must be registered in the `/etc/fstab` file.

### 3. Secure Decommissioning
Simply detaching a disk in the cloud is a security risk. To properly decommission:
1.  **Unmount** the partition.
2.  **Delete** the data and partition table.
3.  **Reformat** the disk to ensure no forensic data recovery is possible before deleting the volume in AWS.
