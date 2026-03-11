# Linux File Management Commands (touch, file, cp, mv, rm)

---

## 📌 Overview

This lab demonstrates fundamental Linux file management commands used to create, inspect, copy, move, and delete files and directories. These commands form the foundation of daily operations for system administrators, DevOps engineers, and developers working in Linux environments.

---

## 🎯 Objective

The objective of this lab is to understand and practice essential Linux commands used for file and directory management, including:

- Creating files
- Inspecting file types
- Copying files and directories
- Moving and renaming files
- Deleting files and directories safely

---

## 🧱 Key Concepts

- **touch** → creates empty files or updates timestamps  
- **file** → identifies file type based on file content  
- **cp** → copies files or directories  
- **mv** → moves or renames files and directories  
- **rm** → removes files or directories permanently  
- **relative paths** → navigating directories relative to current location  
- **recursive operations** → handling entire directory structures  

Understanding these commands is essential for efficient Linux system management.

---

## ⚙️ Practical Demonstration

1. Create a working directory inside `/tmp`.
2. Navigate into the directory.
3. Create a new empty file using `touch`.
4. Verify the file using `ls -l`.
5. Identify the file type using the `file` command.
6. Add content to the file using `echo`.
7. Copy the file to create a duplicate.
8. Create a new directory for testing copy operations.
9. Copy files into the new directory.
10. Copy multiple files at once.
11. Use interactive mode to prevent accidental overwrites.
12. Use verbose mode to display detailed copy output.
13. Recursively copy an entire directory structure.
14. Rename a file using `mv`.
15. Move the renamed file to another directory.
16. Delete files using `rm`.
17. Remove directories recursively using `rm -r`.
18. Use `rm -f` to force deletion when needed.

---

## 🖥️ Commands Used

```bash
# create directory
mkdir /tmp/techniques

# navigate into directory
cd /tmp/techniques

# create empty file
touch first.txt

# list files with details
ls -l

# check file type
file first.txt

# write content to file
echo "Sample text content" > first.txt

# append content
echo "Additional text" >> first.txt

# copy file
cp first.txt second.txt

# create test directory
mkdir /tmp/test

# move to parent directory
cd ..

# copy file into directory
cp techniques/first.txt test/

# list directory contents
ls test

# copy multiple files
cp techniques/first.txt techniques/second.txt test/

# interactive copy
cp -i techniques/first.txt test/

# verbose copy
cp -v techniques/first.txt test/

# recursive directory copy
cp -r techniques test2

# rename file
mv techniques/second.txt techniques/third.txt

# move file into directory
mv techniques/third.txt test/

# delete file
rm test/third.txt

# remove directory recursively
rm -r test

# force deletion
rm -f test2/third.txt

```
--

# 🧠 Technical Explanation
- These commands allow direct manipulation of files and directories in Linux.

- touch creates files quickly without opening an editor
touch first.txt

- file inspects file content rather than extensions to determine type
file first.txt

- cp duplicates files while preserving the original
cp first.txt second.txt

- mv changes file location or name without creating duplicates
mv second.txt third.txt

- rm permanently removes files from the filesystem
rm third.txt

# Options
- -r for recursive operations (copy or delete directories)
cp -r source_dir destination_dir
rm -r directory_name

- -i for interactive mode to prevent accidental overwrites
cp -i first.txt test/

- -v for verbose output to show actions being performed
cp -v first.txt test/
mv -v third.txt test/

# 🌍 Real-World DevOps Use Case
- Copying application files during deployments
cp -r app/ /var/www/app

- Moving log files to backup
mv logs/app.log backup/

- Cleaning temporary build artifacts
rm -rf tmp/*

# 🛠️ Skills Learned
-  Creating and managing files
-  Inspecting file types
-  Copying files and directories
-  Moving and renaming files
-  Deleting files and directories
-  Using recursive operations
-  Navigating directories

# 📂 Example Scenario
- Prepare application files for deployment
mkdir -p /var/www/app
cp -r app/* /var/www/app
mv logs/app.log backup/
rm -rf tmp/*

# ⚡ Possible Improvements
- Use rsync for advanced file synchronization
rsync -av app/ /var/www/app

- Manage file permissions
chmod 644 /var/www/app/config.yml
chown www-data:www-data /var/www/app/config.yml

- Automate file operations using Bash scripts
- logging and safety checks
bash deploy_script.sh

# 🧩 Troubleshooting Notes
- File overwrite during copy
cp -i file1 file2

- Copying directories without -r
cp -r source_directory destination

- Deleting non-empty directories
rm -r directory_name

- Accidental forced deletion - always verify paths before running destructive commands
- rm -rf / (do not execute without confirmation)
