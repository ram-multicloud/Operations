# Linux Shell Script

## Loops

### 1. until loop

> Opposite of `while` — runs as long as the condition is **FALSE**. Stops when the condition becomes true.

```bash
#!/bin/bash

count=1

until [ $count -gt 5 ]; do
  echo "Count: $count"
  count=$(( count + 1 ))
done

# Output:
# Count: 1
# Count: 2
# Count: 3
# Count: 4
# Count: 5
```

**example — wait for network:**

```bash
#!/bin/bash

echo "Waiting for network..."

until ping -c1 google.com &>/dev/null; do
  echo "Still waiting..."
  sleep 2
done

echo "Network is up!"
```

**example — wait for a file to appear:**

```bash
#!/bin/bash

until [ -f /tmp/ready.flag ]; do
  echo "Waiting for ready.flag..."
  sleep 1
done

echo "File appeared! Proceeding..."
```

---

### 2. select loop

> Automatically displays a numbered menu and reads the user's choice. Great for interactive scripts.

```bash
#!/bin/bash

echo "Choose a fruit:"
select fruit in Apple Banana Orange Quit; do
  case $fruit in
    "Apple")   echo "You chose Apple."  ;;
    "Banana")  echo "You chose Banana." ;;
    "Orange")  echo "You chose Orange." ;;
    "Quit")    echo "Goodbye!"; break   ;;
    *)         echo "Invalid choice. Try again." ;;
  esac
done

# Output (menu shown automatically):
# 1) Apple
# 2) Banana
# 3) Orange
# 4) Quit
# #? 1
# You chose Apple.
```

**Custom prompt with PS3:**

```bash
#!/bin/bash

PS3="Enter your choice (1-4): "

select action in Start Stop Restart Quit; do
  case $action in
    "Start")   echo "Starting service..."   ;;
    "Stop")    echo "Stopping service..."   ;;
    "Restart") echo "Restarting service..." ;;
    "Quit")    break ;;
    *)         echo "Invalid option: $REPLY" ;;
  esac
done
```

---

### 3. break & continue

> `break` exits the loop early. `continue` skips the current iteration and moves to the next.

```bash
#!/bin/bash

# break example
echo "--- break ---"
for i in 1 2 3 4 5 6; do
  if [ $i -eq 4 ]; then
    echo "Breaking at $i"
    break
  fi
  echo "i = $i"
done

# Output:
# i = 1
# i = 2
# i = 3
# Breaking at 4
```

```bash
#!/bin/bash

# continue example
echo "--- continue ---"
for i in 1 2 3 4 5; do
  if [ $i -eq 3 ]; then
    echo "Skipping $i"
    continue
  fi
  echo "i = $i"
done

# Output:
# i = 1
# i = 2
# Skipping 3
# i = 4
# i = 5
```

**Real-world example — skip hidden files:**

```bash
#!/bin/bash

for file in *; do
  # Skip hidden files (starting with .)
  [[ $file == .* ]] && continue

  echo "Processing: $file"
done
```

---

## Functions

### 4. Functions — Advanced Usage

> Functions can accept arguments (`$1`, `$2`...), use local variables, and return exit codes (0–255). To return a string, `echo` it and capture with `$()`.

```bash
#!/bin/bash

# Function with arguments and local variable
add_numbers() {
  local result=$(( $1 + $2 ))
  echo $result        # "return" a string/value via echo
}

sum=$(add_numbers 10 20)
echo "Sum: $sum"      # Sum: 30


# Function returning exit status (0=success, non-zero=failure)
is_even() {
  if [ $(( $1 % 2 )) -eq 0 ]; then
    return 0   # success (true)
  else
    return 1   # failure (false)
  fi
}

is_even 4 && echo "4 is even" || echo "4 is odd"
is_even 7 && echo "7 is even" || echo "7 is odd"

# Output:
# 4 is even
# 7 is odd
```

**Real-world example — reusable logger function:**

```bash
#!/bin/bash

log() {
  local level=$1
  local msg=$2
  local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
  echo "[$timestamp] [$level] $msg"
}

log "INFO"  "Script started"
log "WARN"  "Low disk space"
log "ERROR" "File not found"

# Output:
# [2025-05-18 10:30:00] [INFO] Script started
# [2025-05-18 10:30:00] [WARN] Low disk space
# [2025-05-18 10:30:00] [ERROR] File not found
```

**Real-world example — validate input:**

```bash
#!/bin/bash

is_number() {
  [[ $1 =~ ^[0-9]+$ ]]
  return $?
}

read -p "Enter a number: " input

if is_number "$input"; then
  echo "$input is a valid number"
else
  echo "Invalid: '$input' is not a number"
fi
```

---

### 5. Recursive Functions

> A function that calls itself. Used for tree traversal, factorial, directory listing, etc.

```bash
#!/bin/bash

# Factorial using recursion
factorial() {
  local n=$1
  if [ $n -le 1 ]; then
    echo 1
  else
    local prev=$(factorial $(( n - 1 )))
    echo $(( n * prev ))
  fi
}

echo "5! = $(factorial 5)"    # 5! = 120
echo "6! = $(factorial 6)"    # 6! = 720
echo "10! = $(factorial 10)"  # 10! = 3628800
```

**Real-world example — recursive directory listing:**

```bash
#!/bin/bash

list_dir() {
  local dir=$1
  local indent=$2

  for item in "$dir"/*; do
    echo "${indent}$(basename "$item")"
    if [ -d "$item" ]; then
      list_dir "$item" "  $indent"   # recurse into subdirectory
    fi
  done
}

list_dir "." ""
```

---

## I/O & Redirection

### 6. Input/Output Redirection

> Shell has 3 streams: `stdin` (0), `stdout` (1), `stderr` (2). You can redirect any of them to/from files.

```bash
#!/bin/bash

# Redirect stdout to file (overwrite)
echo "Hello World" > output.txt

# Redirect stdout to file (append)
echo "Second line" >> output.txt

# Redirect stdin from file
wc -l < output.txt

# Redirect stderr to file
ls /nonexistent 2> error.log

# Redirect both stdout and stderr to same file
ls /nonexistent > all.log 2>&1

# Bash shorthand for both stdout + stderr
ls /nonexistent &> all.log

# Discard output completely
some_command > /dev/null 2>&1
```

**Real-world example — log script output:**

```bash
#!/bin/bash

LOGFILE="/var/log/myscript.log"

echo "Script started at $(date)" >> "$LOGFILE"

# Redirect all output (stdout + stderr) to log
exec >> "$LOGFILE" 2>&1

echo "Running backup..."
tar -czf backup.tar.gz /home/user/ 
echo "Backup complete"
```

---

### 7. Pipes

> Pipe `|` connects stdout of one command to stdin of the next. Chain commands to process data.

```bash
#!/bin/bash

# Count lines in a file
cat /etc/passwd | wc -l

# Find top 5 largest files in current directory
du -sh * | sort -rh | head -5

# Find all running Java processes
ps aux | grep java | grep -v grep

# Count unique users in /etc/passwd
cut -d: -f1 /etc/passwd | sort | uniq | wc -l

# Show last 10 errors from log
cat app.log | grep "ERROR" | tail -10

# tee — write to file AND pass to next command
ls -la | tee listing.txt | wc -l
```

---

### 8. Here Document (heredoc)

> Write multi-line input directly in the script without creating a separate file.

```bash
#!/bin/bash

# Pass multi-line text to cat
cat << EOF
Hello World
This is a heredoc.
Current user : $(whoami)
Today's date : $(date)
EOF
```

```bash
#!/bin/bash

# Write to a file using heredoc
cat > /tmp/config.txt << EOF
host=localhost
port=5432
database=mydb
user=admin
EOF

echo "Config file created:"
cat /tmp/config.txt
```

```bash
#!/bin/bash

# Quoted EOF — disables variable/command expansion
cat << 'EOF'
This $variable won't be expanded.
Neither will $(this command).
Use this when writing scripts or templates.
EOF
```

**Real-world example — run multiple commands over SSH:**

```bash
#!/bin/bash

ssh user@remote-server << EOF
  echo "Connected to remote"
  ls -la /var/log/
  df -h
  uptime
EOF
```

---

### 9. Here String

> A simpler form of heredoc — pass a single string as stdin to a command using `<<<`.

```bash
#!/bin/bash

# Word count on a string
wc -w <<< "hello world bash scripting"
# Output: 4

# Check if a pattern exists in a string
if grep -q "root" <<< "$(cat /etc/passwd)"; then
  echo "root user exists"
fi

# Read fields from a string
read -r first last <<< "John Doe"
echo "First: $first, Last: $last"
# Output: First: John, Last: Doe

# Base64 decode a string
echo $(base64 --decode <<< "SGVsbG8gV29ybGQ=")
# Output: Hello World
```

---

## Strings & Arrays

### 10. String Manipulation — Advanced

> Bash has powerful built-in **parameter expansion** for strings — no need for `cut`, `awk`, or `python`.

```bash
#!/bin/bash

str="Hello, World!"

# Length of string
echo ${#str}              # 13

# Substring extraction: ${var:start:length}
echo ${str:0:5}           # Hello
echo ${str:7:5}           # World
echo ${str: -6}           # orld!  (from end)

# Uppercase / Lowercase (bash 4+)
echo ${str^^}             # HELLO, WORLD!
echo ${str,,}             # hello, world!
echo ${str^}              # Hello, World!  (first char only)

# Replace first occurrence
echo ${str/World/Shell}   # Hello, Shell!

# Replace all occurrences
echo ${str//l/L}          # HeLLo, WorLd!

# Strip prefix (from start)
path="/home/user/documents"
echo ${path#/}            # home/user/documents   (strip shortest match)
echo ${path##*/}          # documents             (strip longest match)

# Strip suffix (from end)
file="report.tar.gz"
echo ${file%.gz}          # report.tar            (strip shortest match)
echo ${file%%.*}          # report                (strip longest match)
echo ${file##*.}          # gz                    (get extension)

# Default value if variable is empty or unset
name=""
echo ${name:-"Guest"}     # Guest  (use default if empty)
echo ${name:="Admin"}     # Admin  (assign default if empty)

name="Alice"
echo ${name:-"Guest"}     # Alice  (already set, use it)
```

---

### 11. Arrays — Advanced Operations

> Beyond basics: slicing, appending, deleting, and iterating with index.

```bash
#!/bin/bash

fruits=("apple" "banana" "cherry" "date" "elderberry")

# Number of elements
echo "Count: ${#fruits[@]}"          # Count: 5

# Access all elements
echo "All: ${fruits[@]}"

# Access specific index
echo "Index 2: ${fruits[2]}"         # cherry

# Slice: from index 1, take 3 elements
echo "Slice: ${fruits[@]:1:3}"       # banana cherry date

# Append single element
fruits+=("fig")

# Append multiple elements
fruits+=("grape" "honeydew")

# Update element
fruits[0]="APPLE"

# Delete element (leaves gap at that index)
unset fruits[2]

# All indices
echo "Indices: ${!fruits[@]}"

# Loop with index
for i in "${!fruits[@]}"; do
  echo "[$i] = ${fruits[$i]}"
done

# Convert array to string
joined=$(IFS=", "; echo "${fruits[*]}")
echo "Joined: $joined"
```

**Real-world example — filter array elements:**

```bash
#!/bin/bash

servers=("web01" "web02" "db01" "db02" "cache01")
web_servers=()

for s in "${servers[@]}"; do
  [[ $s == web* ]] && web_servers+=("$s")
done

echo "Web servers: ${web_servers[@]}"
# Output: Web servers: web01 web02
```

---

### 12. Associative Arrays (Dictionaries)

> Bash 4+ supports key-value pairs — like a dictionary or hashmap. Must declare with `declare -A`.

```bash
#!/bin/bash

# Declare associative array
declare -A person

person["name"]="Alice"
person["age"]=30
person["city"]="Hyderabad"
person["role"]="DevOps Engineer"

# Access by key
echo "Name: ${person["name"]}"      # Alice
echo "City: ${person["city"]}"      # Hyderabad

# All keys
echo "Keys:   ${!person[@]}"

# All values
echo "Values: ${person[@]}"

# Number of elements
echo "Count: ${#person[@]}"

# Check if key exists
if [[ -v person["email"] ]]; then
  echo "Email exists"
else
  echo "Email not set"
fi

# Loop over key-value pairs
for key in "${!person[@]}"; do
  printf "%-10s => %s\n" "$key" "${person[$key]}"
done

# Delete a key
unset person["role"]
```

**Real-world example — port lookup table:**

```bash
#!/bin/bash

declare -A ports
ports["ssh"]=22
ports["http"]=80
ports["https"]=443
ports["mysql"]=3306
ports["postgres"]=5432

read -p "Enter service name: " svc

if [[ -v ports[$svc] ]]; then
  echo "$svc runs on port ${ports[$svc]}"
else
  echo "Unknown service: $svc"
fi
```

---

## File Tests

### 13. File Test Operators

> Test properties of files and directories inside `if` conditions.

```bash
#!/bin/bash

file="/etc/passwd"
dir="/tmp"

# -e : exists (file or directory)
[ -e "$file" ] && echo "$file exists"

# -f : regular file (not a directory)
[ -f "$file" ] && echo "$file is a regular file"

# -d : is a directory
[ -d "$dir" ]  && echo "$dir is a directory"

# -r : readable
[ -r "$file" ] && echo "$file is readable"

# -w : writable
[ -w "$file" ] && echo "$file is writable"

# -x : executable
[ -x "/bin/bash" ] && echo "/bin/bash is executable"

# -s : size > 0 (file is not empty)
[ -s "$file" ] && echo "$file is not empty"

# -L : symbolic link
[ -L "/bin/sh" ] && echo "/bin/sh is a symlink"

# -z : string is empty
name=""
[ -z "$name" ] && echo "name is empty"

# -n : string is NOT empty
name="Alice"
[ -n "$name" ] && echo "name is not empty"

# Compare two files
[ file1.txt -nt file2.txt ] && echo "file1 is newer than file2"
[ file1.txt -ot file2.txt ] && echo "file1 is older than file2"
```

**Real-world example — pre-flight check before backup:**

```bash
#!/bin/bash

SOURCE="/home/user/data"
DEST="/backup"

if [ ! -d "$SOURCE" ]; then
  echo "ERROR: Source directory does not exist: $SOURCE"
  exit 1
fi

if [ ! -d "$DEST" ]; then
  echo "Creating backup directory..."
  mkdir -p "$DEST"
fi

if [ ! -w "$DEST" ]; then
  echo "ERROR: No write permission on $DEST"
  exit 1
fi

echo "Pre-flight check passed. Starting backup..."
cp -r "$SOURCE" "$DEST"
```

---

### 14. Read File Line by Line

> Process each line of a file using a `while` loop — the most reliable method.

```bash
#!/bin/bash

# Basic line-by-line reading
while IFS= read -r line; do
  echo "Line: $line"
done < /etc/passwd
```

```bash
#!/bin/bash

# Count non-empty, non-comment lines
count=0
while IFS= read -r line; do
  # Skip empty lines and comments
  [[ -z "$line" || "$line" == \#* ]] && continue
  (( count++ ))
done < config.txt

echo "Active config lines: $count"
```

```bash
#!/bin/bash

# Parse CSV file: name,age,city
while IFS=, read -r name age city; do
  echo "Name: $name | Age: $age | City: $city"
done < employees.csv
```

**Real-world example — process a list of servers:**

```bash
#!/bin/bash

# servers.txt contains one hostname per line
while IFS= read -r server; do
  [[ -z "$server" || "$server" == \#* ]] && continue
  echo "Pinging $server..."
  ping -c1 -W1 "$server" > /dev/null 2>&1 \
    && echo "  $server is UP" \
    || echo "  $server is DOWN"
done < servers.txt
```

---

## Text Processing Tools

### 15. grep — Search with Patterns

> Searches for patterns (regular expressions) in text. Essential for log analysis and filtering.

```bash
# Basic search
grep "error" app.log

# Case-insensitive search
grep -i "error" app.log

# Show line numbers
grep -n "error" app.log

# Invert match (lines NOT containing pattern)
grep -v "debug" app.log

# Count matching lines
grep -c "error" app.log

# Show only the matching part
grep -o "ERROR:[^:]*" app.log

# Extended regex (supports |, +, ?, {})
grep -E "error|warning|fatal" app.log

# Recursive search in all files under a directory
grep -r "TODO" ./src/

# Match whole word only
grep -w "fail" app.log

# Show N lines before/after match (context)
grep -A 3 "FATAL" app.log    # 3 lines After
grep -B 2 "FATAL" app.log    # 2 lines Before
grep -C 2 "FATAL" app.log    # 2 lines both sides

# List only filenames that match
grep -l "password" /etc/*.conf

# Suppress binary file warnings
grep -a "pattern" binaryfile
```

**Real-world example — find failed SSH logins:**

```bash
#!/bin/bash

LOG="/var/log/auth.log"

echo "=== Failed SSH Login Attempts ==="
grep "Failed password" "$LOG" | \
  grep -oP 'from \K[\d.]+' | \
  sort | uniq -c | sort -rn | head -10
```

---

### 16. sed — Stream Editor

> `sed` edits text streams — substitute, delete, or insert lines without opening an editor.

```bash
# Replace first occurrence per line
sed 's/old/new/' file.txt

# Replace ALL occurrences (global flag g)
sed 's/old/new/g' file.txt

# Case-insensitive replace
sed 's/error/ERROR/Ig' file.txt

# Edit file in-place (overwrites the file)
sed -i 's/old/new/g' file.txt

# Edit in-place + keep a backup
sed -i.bak 's/old/new/g' file.txt

# Delete lines matching a pattern
sed '/^#/d' file.txt        # delete comment lines
sed '/^$/d' file.txt        # delete blank lines
sed '/error/d' file.txt     # delete lines with "error"

# Print only specific line numbers
sed -n '5p'      file.txt   # only line 5
sed -n '5,10p'   file.txt   # lines 5 to 10
sed -n '$p'      file.txt   # only last line

# Insert a line before/after a match
sed '/pattern/i\New line before' file.txt
sed '/pattern/a\New line after'  file.txt

# Replace only on specific line number
sed '3s/old/new/' file.txt

# Multiple expressions with -e
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file.txt
```

**Real-world example — update config value in place:**

```bash
#!/bin/bash

CONFIG="/etc/myapp/config.conf"

# Change port from any value to 9090
sed -i 's/^port=.*/port=9090/' "$CONFIG"

# Comment out a specific line
sed -i '/^debug=true/s/^/#/' "$CONFIG"

echo "Config updated:"
cat "$CONFIG"
```

---

### 17. awk — Text Processing

> `awk` is a full programming language for structured text. It auto-splits each line into fields `$1`, `$2`... by whitespace (or a custom delimiter).

```bash
# Print second column
awk '{print $2}' file.txt

# Custom field separator (CSV)
awk -F',' '{print $1, $3}' data.csv

# Print lines where column 3 > 100
awk '$3 > 100' data.txt

# Print line number and content
awk '{print NR": "$0}' file.txt

# Sum a numeric column
awk '{sum += $2} END {print "Total:", sum}' sales.txt

# Average of a column
awk '{sum += $3; count++} END {print "Avg:", sum/count}' data.txt

# Print lines with more than 3 fields
awk 'NF > 3' file.txt

# Print lines matching a pattern
awk '/ERROR/' app.log

# Print lines between two patterns
awk '/START/,/END/' file.txt

# BEGIN and END blocks
awk 'BEGIN {print "=== Report ==="} {print $0} END {print "=== Done ==="}' file.txt

# Print unique values from column 1
awk '!seen[$1]++' file.txt

# Reformat output
awk -F':' '{printf "%-15s %s\n", $1, $7}' /etc/passwd
```

**Real-world example — disk usage report:**

```bash
#!/bin/bash

echo "Filesystem Usage Report"
echo "-----------------------"
df -h | awk 'NR > 1 && $5+0 > 70 {
  printf "%-20s %s used (threshold: 70%%)\n", $6, $5
}'
```

**Real-world example — sum column from CSV:**

```bash
#!/bin/bash

# sales.csv: date,product,amount
awk -F',' '
  NR > 1 {
    total += $3
    count++
  }
  END {
    printf "Total Sales : $%.2f\n", total
    printf "Transactions: %d\n", count
    printf "Average     : $%.2f\n", total/count
  }
' sales.csv
```

---

## Advanced Concepts

### 18. Command Substitution

> Capture the output of a command and use it as a value. Prefer `$()` over backticks — it's more readable and supports nesting.

```bash
#!/bin/bash

# Store command output in variable
today=$(date +%Y-%m-%d)
echo "Today is: $today"

# Use directly in a string
echo "Logged in as: $(whoami) on $(hostname)"

# Nested substitution
newest_file=$(ls -t | head -1)
line_count=$(wc -l < "$newest_file")
echo "Newest file '$newest_file' has $line_count lines"

# In arithmetic
disk_used=$(df / | awk 'NR==2 {print $3}')
disk_total=$(df / | awk 'NR==2 {print $2}')
pct=$(( disk_used * 100 / disk_total ))
echo "Disk usage: $pct%"

# Process substitution (advanced)
diff <(sort file1.txt) <(sort file2.txt)
```

---

### 19. Logical Operators && and ||

> `&&` runs the next command only if the previous succeeded (exit 0).  
> `||` runs the next command only if the previous failed (non-zero exit).

```bash
#!/bin/bash

# && : AND — run second only if first succeeds
mkdir /tmp/mydir && echo "Directory created"
cd /tmp/mydir && ls

# || : OR — run second only if first fails
cd /nonexistent || echo "Directory not found!"

# Combine: check network
ping -c1 google.com > /dev/null 2>&1 \
  && echo "Internet: ONLINE" \
  || echo "Internet: OFFLINE"

# Set default if command fails
username=$(getent passwd 9999 | cut -d: -f1) || username="unknown"
echo "User: $username"

# Chain multiple commands
apt-get update && apt-get upgrade -y && apt-get autoremove -y
```

**Real-world example — safe script execution:**

```bash
#!/bin/bash

check_deps() {
  command -v curl  > /dev/null 2>&1 || { echo "curl not found";  exit 1; }
  command -v jq    > /dev/null 2>&1 || { echo "jq not found";    exit 1; }
  command -v unzip > /dev/null 2>&1 || { echo "unzip not found"; exit 1; }
  echo "All dependencies satisfied."
}

check_deps && echo "Starting deployment..." || exit 1
```

---

### 20. Double Brackets [[ ]]

> `[[ ]]` is an enhanced `[ ]` — supports regex, logical AND/OR inside, and is safer with variables.

```bash
#!/bin/bash

name="Alice"
age=25
file="report.txt"

# No need to quote variables inside [[ ]]
if [[ $name == "Alice" ]]; then
  echo "Hello, Alice!"
fi

# Pattern matching with ==
if [[ $file == *.txt ]]; then
  echo "It's a text file"
fi

# Regex matching with =~
email="user@example.com"
if [[ $email =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
  echo "Valid email: $email"
else
  echo "Invalid email"
fi

# Logical AND inside [[ ]]
if [[ $age -ge 18 && $age -le 60 ]]; then
  echo "Working age group"
fi

# Logical OR inside [[ ]]
if [[ $name == "Alice" || $name == "Bob" ]]; then
  echo "Recognized user"
fi

# String comparison
if [[ "apple" < "banana" ]]; then
  echo "apple comes before banana alphabetically"
fi
```

---

### 21. getopts — Parse Arguments

> `getopts` parses command-line flags (like `-n name -v`) in a standard, robust way.

```bash
#!/bin/bash

usage() {
  echo "Usage: $0 -n <name> -a <age> [-v] [-h]"
  echo "  -n  Name (required)"
  echo "  -a  Age  (required)"
  echo "  -v  Verbose mode"
  echo "  -h  Show this help"
  exit 1
}

name=""
age=""
verbose=false

while getopts "n:a:vh" opt; do
  case $opt in
    n) name="$OPTARG"  ;;    # -n requires a value (colon after n)
    a) age="$OPTARG"   ;;    # -a requires a value
    v) verbose=true    ;;    # -v is a flag (no value)
    h) usage           ;;    # -h prints help
    ?) usage           ;;    # unknown option
  esac
done

# Check required args
[[ -z "$name" || -z "$age" ]] && usage

echo "Name   : $name"
echo "Age    : $age"
$verbose && echo "[VERBOSE] Script ran at $(date)"

# Run as:
# ./script.sh -n Alice -a 30 -v
```

---

### 22. trap — Signal Handling

> `trap` catches OS signals and runs cleanup code before the script exits.

```bash
#!/bin/bash

# Cleanup function
cleanup() {
  echo ""
  echo "Caught signal — cleaning up..."
  rm -f /tmp/myapp_*
  echo "Cleanup complete. Goodbye!"
  exit 0
}

# Trap SIGINT (Ctrl+C), SIGTERM (kill), and EXIT (script ends)
trap cleanup INT TERM EXIT

# Create a temp file
tmpfile=$(mktemp /tmp/myapp_XXXX)
echo "Working with temp file: $tmpfile"

echo "Running... Press Ctrl+C to stop"
for i in $(seq 1 10); do
  echo "Step $i / 10"
  sleep 1
done
```

**Real-world example — lock file to prevent duplicate runs:**

```bash
#!/bin/bash

LOCKFILE="/tmp/myscript.lock"

cleanup() {
  rm -f "$LOCKFILE"
  echo "Lock released."
}

trap cleanup EXIT INT TERM

if [ -f "$LOCKFILE" ]; then
  echo "Script already running (lock: $LOCKFILE). Exiting."
  exit 1
fi

touch "$LOCKFILE"
echo "Lock acquired. Running..."

# Simulate long work
sleep 30

echo "Done."
```

---

### 23. Debugging — set flags

> Bash has built-in flags to catch errors early and trace execution.

```bash
#!/bin/bash

# -----------------------------------------------
# Best practice: always add these at the top
# -----------------------------------------------
set -e          # Exit immediately on any error
set -u          # Treat unset variables as errors
set -o pipefail # Pipeline fails if ANY command fails
set -x          # Print each command before executing (trace)

# Or combine all four:
set -euxo pipefail
```

```bash
#!/bin/bash
set -euo pipefail

# -e demo: this will cause the script to exit
# because /nonexistent doesn't exist
ls /nonexistent
echo "This line won't run"
```

```bash
#!/bin/bash
set -u

# -u demo: referencing unset variable causes error
echo "Hello, $undefined_var"   # Error: unbound variable
```

```bash
#!/bin/bash
set -o pipefail

# Without pipefail, this would succeed (grep exit code 1 ignored):
cat /nonexistent | grep "something"   # Fails correctly with pipefail
```

```bash
#!/bin/bash

# Enable trace for just a specific section
echo "Before debug"
set -x
some_complex_command
another_command
set +x    # turn off trace
echo "After debug"
```

```bash
# Run an entire script with trace from outside:
bash -x myscript.sh

# Run with all safety flags from outside:
bash -euxo pipefail myscript.sh
```

---

### 24. Background Processes

> Run commands in the background with `&` and wait for completion with `wait`.

```bash
#!/bin/bash

# Run a command in the background
sleep 5 &
echo "Background PID: $!"    # $! = PID of last background job

# Wait for a specific PID
wait $!
echo "sleep finished"

# List current background jobs
jobs
```

```bash
#!/bin/bash

# Run multiple tasks in parallel
process_file() {
  local file=$1
  echo "  Processing $file..."
  sleep 2
  echo "  Done with $file"
}

echo "Starting parallel processing..."

for file in file1.txt file2.txt file3.txt; do
  process_file "$file" &    # Run each in background
done

wait    # Wait for ALL background jobs to finish
echo "All files processed!"
```

**Real-world example — parallel health checks:**

```bash
#!/bin/bash

check_server() {
  local host=$1
  if ping -c1 -W1 "$host" > /dev/null 2>&1; then
    echo "  [UP]   $host"
  else
    echo "  [DOWN] $host"
  fi
}

servers=("web01" "web02" "db01" "cache01")

echo "Checking all servers in parallel..."
for server in "${servers[@]}"; do
  check_server "$server" &
done

wait
echo "Health check complete."
```

---

### 25. printf — Formatted Output

> `printf` gives precise control over output format — behaves consistently across all systems unlike `echo`.

```bash
#!/bin/bash

# Basic formatting
printf "Name: %s, Age: %d\n" "Alice" 30

# Left-aligned (- flag) in a field of width 15
printf "%-15s %-10s %s\n" "Name" "Age" "City"
printf "%-15s %-10d %s\n" "Alice"   30 "Hyderabad"
printf "%-15s %-10d %s\n" "Bob"     25 "Mumbai"
printf "%-15s %-10d %s\n" "Charlie" 35 "Bengaluru"

# Output:
# Name            Age        City
# Alice           30         Hyderabad
# Bob             25         Mumbai
# Charlie         35         Bengaluru
```

```bash
#!/bin/bash

# Float formatting
printf "Pi = %.4f\n"   3.14159265    # Pi = 3.1416
printf "%.2f%%\n"      98.6789       # 98.68%

# Zero-padded integers
printf "%05d\n"  42                  # 00042
printf "%08d\n"  1234                # 00001234

# Hex and Octal
printf "%x\n"  255                   # ff
printf "%o\n"  8                     # 10

# No newline (unlike echo)
printf "Enter your name: "
read name
printf "Hello, %s!\n" "$name"
```

**Real-world example — formatted report:**

```bash
#!/bin/bash

# Print a formatted table
separator="+-----------------+-------+------------+"

printf "%s\n" "$separator"
printf "| %-15s | %-5s | %-10s |\n" "Server" "CPU%" "Status"
printf "%s\n" "$separator"

declare -A servers
servers["web01"]="23"
servers["web02"]="78"
servers["db01"]="45"

for server in "${!servers[@]}"; do
  cpu="${servers[$server]}"
  if [ "$cpu" -gt 70 ]; then
    status="HIGH LOAD"
  else
    status="OK"
  fi
  printf "| %-15s | %-5s | %-10s |\n" "$server" "$cpu%" "$status"
done

printf "%s\n" "$separator"
```

---

## Quick Reference Card

| Concept | Syntax | Purpose |
|---|---|---|
| `until` loop | `until [ condition ]; do ... done` | Loop while condition is false |
| `select` loop | `select var in list; do ... done` | Interactive menu |
| `break` | `break` | Exit loop early |
| `continue` | `continue` | Skip current iteration |
| Function return value | `echo $val` + `result=$(func)` | Return a string from function |
| Redirect stdout | `cmd > file` | Overwrite file with output |
| Redirect append | `cmd >> file` | Append output to file |
| Redirect stderr | `cmd 2> file` | Send errors to file |
| Both streams | `cmd &> file` | Send all output to file |
| Pipe | `cmd1 \| cmd2` | Pass output as input |
| Heredoc | `cmd << EOF ... EOF` | Multi-line input inline |
| Here string | `cmd <<< "string"` | Single string as stdin |
| String length | `${#var}` | Number of characters |
| Substring | `${var:start:len}` | Extract substring |
| Replace | `${var/old/new}` | Replace in string |
| Strip suffix | `${var%.ext}` | Remove from end |
| Strip prefix | `${var#prefix}` | Remove from start |
| Default value | `${var:-default}` | Use default if empty |
| Assoc array | `declare -A map` | Key-value dictionary |
| File is regular | `[ -f file ]` | True if regular file |
| File is directory | `[ -d dir ]` | True if directory |
| File is readable | `[ -r file ]` | True if readable |
| File not empty | `[ -s file ]` | True if size > 0 |
| `grep` regex | `grep -E "a\|b"` | Extended regex search |
| `sed` replace | `sed 's/old/new/g'` | Replace in stream |
| `awk` field | `awk '{print $2}'` | Print 2nd column |
| `&&` | `cmd1 && cmd2` | Run cmd2 if cmd1 succeeds |
| `\|\|` | `cmd1 \|\| cmd2` | Run cmd2 if cmd1 fails |
| `[[ ]]` regex | `[[ $var =~ regex ]]` | Pattern matching |
| `getopts` | `getopts "n:v" opt` | Parse CLI flags |
| `trap` | `trap cleanup EXIT` | Run on signal/exit |
| Debug all | `set -euxo pipefail` | Safe script mode |
| Background | `cmd &` | Run in background |
| Wait all | `wait` | Wait for all bg jobs |
| `printf` | `printf "%-10s %d\n"` | Formatted output |
| Command sub | `var=$(cmd)` | Capture command output |

---

*Generated for: [ram-repo/shell_Script](https://github.com/ram-repo/shell_Script)*
