# Advanced Linux Text Processing: Mastering Sed & Awk

---

## 📌 Overview
This repository focuses on two of the most powerful text-processing utilities in the Linux ecosystem: **Sed** (Stream Editor) and **Awk**. These tools enable non-interactive text manipulation, data extraction, and automated reporting, making them indispensable for DevOps engineers managing large-scale infrastructure.

---

## 🎯 Objective
The goal of this lab is to demonstrate how to perform complex text transformations and data analysis directly from the terminal or within scripts, bypassing the need for manual text editors.

---

## 🧱 Key Concepts
* **Sed (Stream Editor):** Used for filtering and transforming text. It excels at substitution, deletion, and line-based operations on data streams.
* **Awk:** A complete pattern-scanning and processing language. It treats files as a series of records (lines) and fields (columns), allowing for sophisticated data extraction and logic.
* **Global Substitution (`g`):** Ensures a pattern is replaced every time it appears in a line, rather than just the first occurrence.
* **In-place Editing (`-i`):** Instructs `sed` to save changes directly to the source file.
* **Field Separators (`-F`):** Defines the character (e.g., `:`, `,`, `|`) used to split lines into columns in `awk`.

---

## ⚙️ Practical Demonstration
1.  **Basic Substitution:** Using `sed` to replace strings in a data stream.
2.  **Global File Updates:** Applying `sed -i` to update employee records across a file.
3.  **Advanced Filtering:** Using line ranges and the `-n` flag to print specific results.
4.  **Field Extraction:** Utilizing `awk` to parse system logs and `/etc/passwd`.
5.  **Conditional Reporting:** Writing `awk` scripts with `BEGIN`, `END`, and `if` statements to generate formatted reports.

---

## 🖥️ Commands Used

```bash
# Sed: Substitute 'world' with 'Linux'
echo "hello world" | sed 's/world/Linux/'

# Sed: Global replacement in a file
sed 's/Charlie/David/g' employees.txt

# Sed: In-place edit (saves to file)
sed -i 's/Charlie/David/g' employees.txt

# Sed: Delete lines containing 'Eve' or the last line
sed '/Eve/d' employees.txt
sed '$d' employees.txt

# Awk: Print specific columns (Date, Time, Level)
awk '{print $1, $2, $3}' sample_log.txt

# Awk: Search for 'error' and print timestamp
awk '/error/ {print $1, $2}' sample_log.txt

# Awk: Use custom separator and conditional logic
awk -F ":" '$3 == "error" {print $0}' sample_log.txt

# Awk: Formatted output for users
awk -F ":" 'BEGIN {print "User List"} {printf "%-20s %-10s\n", $1, $3} END {print "End of List"}' /etc/passwd

```

## 🧠 Technical Explanation
- --- SED LOGIC ---
- s = substitute command
- /.../ = delimiters separating pattern and replacement
- g = global flag (without this, sed only hits the first match per line)
- -i = edit in-place (uses a temporary file under the hood to overwrite the original)
- -e = allows chaining multiple expressions in one command

---

# --- AWK LOGIC ---
- $0 = Represents the entire current line
- $1, $2, $n = Represents the nth field (column) in the record
- NF = Number of Fields (useful for accessing the last column via $NF)
- NR = Number of Records (the current line number)
- -F = Field Separator (default is whitespace; change to ":" for system files)
- BEGIN/END = Blocks executed before/after processing the file
- printf = Formatted printing (similar to C), allowing for alignment and padding

---

## 🌍 Real-World DevOps Use Case 
- Configuration Management: Using sed in a CI/CD pipeline to inject environment variables or update image tags in a deployment.yaml file.

- Log Parsing: Using awk to extract IP addresses from Nginx access logs to identify potential DDoS attacks.

- User Auditing: Parsing /etc/passwd or /etc/group to generate security reports on user IDs and shell access.

---

## 🛠️ Skills Learned

- Automated text replacement and file cleanup.
- Advanced log file analysis and field-level data extraction.
- Writing one-liner scripts for system reporting.
- Understanding the difference between terminal output and file modification.

---

## 📂 Example Scenario: Production Log Filtering
Problem: A production server is throwing intermittent errors. You have a 5GB log file and need to find the timestamps of every "Connection Timeout" error to correlate with network spikes.

Solution:
```bash
awk '/Connection Timeout/ {print $1, $2}' production_log.txt
```
This scans the file without opening it in a memory-heavy editor, extracting only the relevant time data.

---

## ⚡ Possible Improvements
- Regex Mastery: Learning Extended Regular Expressions (ERE) to use with sed -E.
- Integration: Piping awk output into sort and uniq -c to count occurrences of specific errors.
- Alternative Tools: Exploring jq for JSON processing and yq for YAML, which are more common in modern cloud-native workflows.

---

## 🧩 Troubleshooting Notes
```bash
- Sed not saving? Ensure you are using the -i flag. On macOS, sed -i '' is required.

- Awk outputting one big block? Your field separator (-F) might be wrong. Check if your file uses tabs, commas, or colons.

- Pattern not matching? Remember that sed and awk are case-sensitive by default. Use I flag in sed or tolower() in awk for case-insensitive searches.
```
