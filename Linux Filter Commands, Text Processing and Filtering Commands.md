# Linux Filter Commands and Text Processing (grep, egrep, wc, pipe, comm, find)

---

## 📌 Overview

This lab explores essential Linux filtering and search commands used to analyze files, extract information, and automate text processing. These commands allow engineers to search patterns, count results, compare files, and locate files efficiently within Linux systems.

---

## 🎯 Objective

- Learn how to search text patterns using `grep`
- Understand how to filter command output using pipes
- Count lines, words, and characters with `wc`
- Compare files using `comm`
- Search files and directories using `find`

---

## 🧱 Key Concepts

- **grep** – searches for patterns inside files  
- **egrep** – supports extended regular expressions  
- **wc** – counts lines, words, and characters  
- **pipe ( | )** – passes output of one command to another  
- **comm** – compares two sorted files  
- **find** – searches files and directories based on attributes  

These tools are fundamental for Linux system administration and DevOps automation tasks.

---

## ⚙️ Practical Demonstration

1. Search for specific text patterns in files using `grep`.
2. Perform case-insensitive searches with `grep -i`.
3. Exclude lines containing a pattern using `grep -v`.
4. Match whole words using `grep -w`.
5. Display line numbers where matches occur using `grep -n`.
6. Show lines before or after a match using context options.
7. Search across multiple files and directories.
8. Count matching lines using `grep -c`.
9. Count file statistics using `wc`.
10. Chain commands using pipes (`|`).
11. Use `egrep` for extended regular expressions.
12. Compare files using `comm`.
13. Locate files and directories using `find`.

---

## 🖥️ Commands Used

```bash
grep "Alice" details.txt
grep -i "alice" details.txt
grep -v "Alice" details.txt
grep -iv "Alice" details.txt
grep -iw "IT" details.txt
grep -n "Alice" details.txt

grep -A 2 "Alice" details.txt
grep -B 2 "Alice" details.txt
grep -C 2 "Alice" details.txt

grep "Alice" ./*
grep "Alice" *.txt

grep -r "Alice" .

grep -c "Developer" details.txt

wc details.txt
wc -l details.txt

grep -i "developer" details.txt | wc -l

ls /usr/bin | grep apt
ls /usr/bin | grep apt | wc -l

grep -r "error" /var/log --include="*.log"
grep -r "error" /var/log --exclude="*.back"

egrep '\+1-[0-9]{3}-[0-9]{3}-[0-9]{4}' contacts/contacts.txt

comm file1.txt file2.txt
comm -12 file1.txt file2.txt

find /etc -name nsswitch.conf
find /etc -name "*.conf"
find /etc -type d
find /etc -type f

find /etc -type d | head

```

#🧠 Technical Explanation
grep
- searches text inside files using patterns
- prints lines that match the specified pattern
- supports options for case-insensitive search, line numbers, and context display

grep -i
- performs case insensitive pattern matching

grep -v
- excludes lines that contain the specified pattern

grep -w
- matches only whole words instead of partial matches

grep -n
- prints line numbers for matching results

grep -A / -B / -C
- displays lines after, before, or around a matching pattern

pipe |
- sends output from one command to another command as input
- enables command chaining for efficient processing

wc
- counts file statistics including lines, words, and bytes
- often used to measure file size or count filtered results

egrep
- uses extended regular expressions
- supports advanced pattern syntax like repetition and alternation

comm
- compares two sorted files line by line
- output is divided into three columns
- column 1 unique to file1
- column 2 unique to file2
- column 3 common lines

find
- searches files and directories based on attributes
- can filter results by name, type, size, permissions, and timestamps
- commonly used in system administration for locating files

--

#🌍 Real-World DevOps Use Case
- grep -r "error" /var/log --include="*.log"

- tail -f /var/log/syslog | grep error

- ls /usr/bin | grep apt

- grep -i "failed" /var/log/auth.log | wc -l

-DevOps engineers frequently analyze system and application logs to detect failures, monitor deployments, and investigate incidents. These commands allow fast filtering of critical information from large log files.

--

#🛠️ Skills Learned

- Searching text patterns in files
- Filtering command output using pipes
- Counting lines and matches
- Comparing file contents
- Locating files within Linux directories
- Using regular expressions for pattern matching

--

#📂 Example Scenario

- A DevOps engineer needs to investigate errors occurring in a production environment.

- Steps:

- grep -r "error" /var/log --include="*.log"

- grep -i "timeout" /var/log/app.log

- grep -i "failed" /var/log/auth.log | wc -l

--

#⚡ Possible Improvements

- Combine grep with awk for advanced filtering
- Use sed for text transformation
- Automate log analysis using shell scripts
- Integrate log monitoring with observability tools
- Use centralized logging platforms such as ELK or Loki

--

#🧩 Troubleshooting Notes

- problem:
grep returns unexpected matches

- solution:
grep -w "pattern" file.txt

--

- problem:
grep throws directory errors when searching multiple files

- solution:
grep "pattern" *.txt

--

- problem:
permission denied when using find

- solution:
sudo find /etc -name filename

--

- problem:
too much output from find command

- solution:
find /etc -type f | head


---


# Linux Text Processing and Filtering Commands (sort, uniq, nl, cut, tr, column, pr, tee)

---

## 📌 Overview

This lab explores several Linux text processing utilities used to organize, filter, and format textual data. These commands are commonly used in shell pipelines to manipulate structured text such as logs, CSV files, and command outputs.

---

## 🎯 Objective

- Learn how to organize and sort text data using `sort`
- Remove duplicate lines using `uniq`
- Add line numbers using `nl`
- Extract specific fields using `cut`
- Replace characters using `tr`
- Format output into readable tables using `column`
- Prepare text for printing using `pr`
- Capture command output using `tee`

---

## 🧱 Key Concepts

- **sort** – organizes lines alphabetically, numerically, or by specific fields  
- **uniq** – filters duplicate lines from sorted input  
- **nl** – adds line numbers to text output  
- **cut** – extracts specific columns from structured data  
- **tr** – translates or replaces characters  
- **column** – formats output into aligned columns  
- **pr** – prepares files for printing with formatted layout  
- **tee** – writes command output to both terminal and file

---

## ⚙️ Practical Demonstration

1. Sort employee names alphabetically using `sort`.
2. Handle case-sensitive sorting and ignore case differences.
3. Sort months using the correct chronological order.
4. Sort numeric values correctly using numeric sorting.
5. Sort CSV data by specific fields using delimiters.
6. Remove duplicate records using `uniq`.
7. Display the number of duplicate occurrences.
8. Add line numbers to records using `nl`.
9. Extract specific columns from CSV files using `cut`.
10. Replace characters using `tr`.
11. Format structured output using `column`.
12. Prepare text output for printing using `pr`.
13. Save command output to both terminal and file using `tee`.

---

## 🖥️ Commands Used

```bash
sort employees.txt
sort -f employees.txt

sort -M months.txt
sort -Mr months.txt

sort numbers.txt
sort -n numbers.txt

sort -t ',' -k2 employees.csv
sort -t ',' -k3n employees.csv

uniq employees.txt
uniq -c employees.txt
uniq -D employees.txt
uniq -d employees.txt

nl employees.txt

uniq employees.txt | nl

cut -d ',' -f1 employees.csv

cat employees.csv | tr ',' '\t'

cat employees.csv | tr ',' '\t' | column -t

pr -t -2 employees.txt

cat employees.txt | tee output.txt

```

---

#🧠 Technical Explanation

- SORT reads input lines and arranges them in a specific order. By default it performs alphabetical sorting based on ASCII values. Options allow case-insensitive sorting, numeric sorting, month-based sorting, and sorting based on specific fields.

- UNIQ filters repeated lines from sorted input. It can display unique lines, show counts of repeated entries, or display only duplicates.

- NL adds line numbers to each line of text, making it easier to reference specific entries.

- CUT extracts selected portions of each line. It is commonly used with structured files such as CSV where fields are separated by delimiters.

- TR translates or replaces characters in input streams. It is useful for transforming delimiters or performing simple text substitutions.

- COLUMN formats input text into aligned columns, making structured data easier to read.

- PR prepares files for printing by organizing text into pages or columns.

- TEE reads from standard input and writes the output both to the terminal and to a file, allowing real-time logging while saving results.

---

#🌍 Real-World DevOps Use Case

DevOps engineers frequently manipulate structured text such as logs, metrics, and CSV exports.

Example usage:
- cat access.log | cut -d ' ' -f1 | sort | uniq -c | sort -n

This pipeline extracts IP addresses from logs, counts occurrences, and sorts them to identify the most active clients.

Another example for formatting deployment data:
- cat deployments.csv | tr ',' '\t' | column -t

This improves readability when reviewing structured deployment data in the terminal.

---

#🛠️ Skills Learned
- Sorting text and numerical data
- Removing duplicate records
- Extracting structured fields from files
- Transforming text streams
- Formatting command output for readability
- Building efficient shell pipelines
- Capturing command output for logging

---

#📂 Example Scenario

A DevOps engineer analyzes a CSV file containing deployment information.

Steps: 
- cut -d ',' -f2 deployments.csv
- sort -t ',' -k3 deployments.csv
- uniq deployments.csv

These commands help extract specific fields, sort deployments by attributes, and identify duplicate entries.

---

#⚡ Possible Improvements

- Combine these commands with grep for advanced filtering
- Use awk for more complex text processing
- Automate analysis with shell scripts
- Integrate text pipelines into CI/CD workflows
- Process large log datasets efficiently

---

#🧩 Troubleshooting Notes

- sort numbers incorrectly
- sort -n numbers.txt

---

- uniq not removing duplicates
- sort file.txt | uniq

---

- columns misaligned in CSV output
- cat file.csv | tr ',' '\t' | column -t

---

- output needs to be saved and displayed
- command | tee output.txt
