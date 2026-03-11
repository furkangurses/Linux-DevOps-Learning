# Linux Compression and Archiving (zip, gzip, tar)

---

## 📌 Overview

This lab demonstrates how Linux handles **file compression and archiving** using common utilities such as `zip`, `gzip`, and `tar`. These tools help reduce file size, package multiple files together, and efficiently transfer or back up data.

Compression reduces storage usage and speeds up file transfers, while archiving groups multiple files into a single container.

---

## 🎯 Objective

The objective of this lab is to understand how to:

- Compress files to reduce storage usage
- Archive multiple files into a single file
- Combine archiving and compression
- Extract and inspect compressed archives
- Apply compression techniques commonly used in Linux systems

---

## 🧱 Key Concepts

- **Compression** – Reduces file size to save disk space and improve transfer speed.
- **Archiving** – Combines multiple files into a single file without necessarily compressing them.
- **zip** – Provides both compression and archiving.
- **gzip** – Compresses individual files only.
- **tar** – Archives files and directories while preserving structure and permissions.
- **tar + gzip** – A common Linux method to archive and compress files into `.tar.gz`.

---

## ⚙️ Practical Demonstration

1. Compress multiple files into a single `.zip` archive.
2. Verify archive contents using `zipinfo`.
3. Extract files from a zip archive.
4. Compress a single file using `gzip`.
5. View compressed content without extracting.
6. Apply maximum compression using gzip options.
7. Decompress gzip files.
8. Archive multiple files using `tar`.
9. Extract files from a tar archive.
10. Combine `tar` and `gzip` to create compressed archives (`.tar.gz`).
11. List archive contents before extraction.
12. Create compressed backups of directories.

---

## 🖥️ Commands Used

```bash
# Install zip if missing
sudo apt install zip

# Compress files using zip
zip files.zip file1.txt file2.txt

# View archive information
zipinfo files.zip

# Extract zip archive
unzip files.zip

# Compress a directory recursively
zip -r var_backup.zip /var

# Compress a single file using gzip
gzip file1.txt

# Apply maximum compression
gzip -9 file1.txt

# View gzip content without extraction
zcat file1.txt.gz

# View gzip content interactively
zmore file1.txt.gz
zless file1.txt.gz

# Decompress gzip file
gunzip file1.txt.gz

# Create tar archive
tar -cvf archive.tar file1.txt file2.txt

# Extract tar archive
tar -xvf archive.tar

# Create compressed tar archive
tar -czvf archive.tar.gz file1.txt file2.txt

# List files inside tar archive
tar -tf archive.tar.gz

# Extract compressed tar archive
tar -xzvf archive.tar.gz

# Backup directory using tar and gzip
tar -czvf var_backup.tar.gz /var
```

---

## 🧠 Technical Explanation

The `zip` command compresses and archives files into a single `.zip` file. It is commonly used for cross-platform file sharing because it works well across Windows, Linux, and macOS systems.

The `gzip` utility compresses individual files by replacing them with a `.gz` version. It cannot compress directories directly. Instead, it is often combined with `tar` to compress multiple files.

The `tar` command creates archive files while preserving file permissions, directory structure, and metadata. By default, it only archives files without compressing them.

When combined with `gzip` using the `-z` option, `tar` produces compressed archives (`.tar.gz`). These files are commonly called **tarballs** and are widely used for backups, software distribution, and data transfer on Linux systems.

---

## 🌍 Real-World DevOps Use Case

DevOps engineers frequently use compression and archiving for **backups, artifact packaging, and deployment pipelines**.

Example: creating a backup of application logs.

```bash
tar -czvf logs_backup.tar.gz /var/log
```

Example: packaging a build artifact before transferring it to a deployment server.

```bash
tar -czvf app_release.tar.gz build/
```

These archives can then be transferred efficiently using tools like `scp`, `rsync`, or artifact repositories.

---

## 🛠️ Skills Learned

- Compressing files to reduce storage size
- Archiving multiple files into a single container
- Creating and extracting `.zip` archives
- Using `gzip` for efficient file compression
- Combining `tar` and `gzip` for Linux backups
- Inspecting archive contents without extracting them
- Managing compressed files in Linux environments

---

## 📂 Example Scenario

A DevOps engineer needs to back up an application directory before performing a deployment.

First, create a compressed archive:

```bash
tar -czvf app_backup.tar.gz /var/www/app
```

Transfer the archive to a backup server:

```bash
scp app_backup.tar.gz backup-server:/backups/
```

If a rollback is required, restore the files:

```bash
tar -xzvf app_backup.tar.gz
```

This process ensures safe and efficient application backups.

---

## ⚡ Possible Improvements

- Automate backups using cron jobs
- Integrate compression with CI/CD pipelines
- Use `rsync` for incremental backups
- Explore advanced compression tools like `xz` or `bzip2`

---

## 🧩 Troubleshooting Notes

```bash
zip: command not found
Install zip package:
sudo apt install zip

gzip replaces original file
Use gzip -k to keep original file

tar extraction fails
Verify archive format (.tar vs .tar.gz)

large archive extraction slow
Consider using faster compression methods
```

---
