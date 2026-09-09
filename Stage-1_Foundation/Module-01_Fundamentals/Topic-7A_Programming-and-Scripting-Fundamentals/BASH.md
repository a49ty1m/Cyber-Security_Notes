# Bash — Linux Automation & Operations Practice Questions

Use this as a learning checklist. Solve each question yourself, test edge cases, and keep solutions in clearly named `.sh` files. For security and reconnaissance exercises, use only systems, networks, and accounts you own or are explicitly authorized to test.

---

## 1. Shell Fundamentals

### 1.1 Variables and Data Types

- [ ] Declare a variable `name="smilo"` and print it; explain the difference between `name="smilo"` (assignment) and `name = "smilo"` (syntax error).
- [ ] Demonstrate the three quoting modes: no quotes, single quotes `'...'`, and double quotes `"..."`; show where variable expansion is suppressed and where it works.
- [ ] Use `readonly VAR=value` to declare a constant; attempt to modify it and explain the resulting error.
- [ ] Print the length of a string variable using `${#VAR}` and extract substrings with `${VAR:offset:length}`.
- [ ] Use string substitution operators: `${VAR:-default}`, `${VAR:=default}`, `${VAR:+alternate}`, and `${VAR:?error_message}`; demonstrate each with an unset variable.
- [ ] Use pattern substitution `${VAR/pattern/replacement}` and `${VAR//pattern/replacement}` to replace the first and all occurrences of a pattern.
- [ ] Use `${VAR#prefix}`, `${VAR##prefix}`, `${VAR%suffix}`, and `${VAR%%suffix}` to strip prefixes and suffixes; demonstrate with a file path.
- [ ] Use `declare -i` to create an integer variable; attempt to assign a string to it and observe the result.
- [ ] Use `declare -l` and `declare -u` to force a variable to lowercase or uppercase on assignment.
- [ ] Explain the difference between local and global variables in a shell script; demonstrate a variable that leaks out of a subshell unintentionally.

### 1.2 Command Substitution and Arithmetic

- [ ] Use command substitution `$(command)` to capture output; assign the hostname to a variable with `HOST=$(hostname)`.
- [ ] Explain the difference between `$(command)` and the legacy backtick syntax `` `command` ``; state why `$()` is preferred for nesting.
- [ ] Use `$(( ))` for integer arithmetic: add, subtract, multiply, divide (integer), and compute a modulus.
- [ ] Use `$(( ))` to check if a number is even or odd; print an appropriate message.
- [ ] Use `let` and `expr` for arithmetic; compare them with `$(( ))` and explain when `expr` is still relevant.
- [ ] Compute the number of seconds in a week using `$(( ))` arithmetic and print the result.
- [ ] Use `bc` via command substitution for floating-point arithmetic (e.g., `echo "scale=4; 22/7" | bc`).
- [ ] Use `printf "%d\n" "0xFF"` to convert hexadecimal to decimal and `printf "%x\n" 255` to convert decimal to hex.

### 1.3 Conditionals

- [ ] Write an `if / elif / else` statement that checks whether a number is positive, negative, or zero.
- [ ] Use `[[ ]]` (extended test) instead of `[ ]`; explain why `[[ ]]` is safer for string comparisons and supports regex matching with `=~`.
- [ ] Use integer comparison operators (`-eq`, `-ne`, `-lt`, `-le`, `-gt`, `-ge`) inside `[[ ]]` to compare two variables.
- [ ] Use string comparison operators (`==`, `!=`, `<`, `>`, `-z`, `-n`) inside `[[ ]]` to test strings.
- [ ] Use file test operators: `-e` (exists), `-f` (regular file), `-d` (directory), `-r` (readable), `-w` (writable), `-x` (executable), `-s` (non-empty) inside `[[ ]]`.
- [ ] Use `&&` and `||` to chain commands conditionally: `command1 && command2` (run command2 only if command1 succeeds) and `command1 || command2` (run command2 only if command1 fails).
- [ ] Write a `case` statement that matches file extensions (`.sh`, `.py`, `.c`, `.md`) and prints the file type for each.
- [ ] Use `case` with `;;`, `;&` (fall-through), and `;;&` (continue matching) and explain the behavior of each terminator.
- [ ] Write an `if` statement that tests the exit code of the previous command using `$?`; explain why checking `$?` immediately is critical.

### 1.4 Loops

- [ ] Write a `for` loop using a brace expansion range `{1..10}` to print numbers 1 through 10.
- [ ] Write a `for` loop using `for var in item1 item2 item3` to iterate over a list of strings.
- [ ] Write a C-style `for` loop `for (( i=0; i<10; i++ ))` and explain when it is cleaner than a range loop.
- [ ] Write a `while` loop that reads lines from a file using `while IFS= read -r line; do ... done < file.txt`.
- [ ] Write a `until` loop that polls for a file to appear every second and breaks when it exists.
- [ ] Use `break` to exit a loop and `continue` to skip an iteration; demonstrate both in a `for` loop over a list.
- [ ] Write a loop over the output of a command: `for f in $(ls *.sh)`; then rewrite it safely using `while IFS= read -r f; do ... done < <(find . -name "*.sh")` and explain the pitfall of the first form.
- [ ] Use `seq` to generate a sequence of numbers with a step value: `seq 0 5 100`; use it in a `for` loop.
- [ ] Write a loop that processes all files in a directory using glob expansion: `for f in /path/to/dir/*.log; do ... done`.

### 1.5 Functions

- [ ] Define a function using both `function_name() { }` and `function function_name { }` syntaxes; explain the difference.
- [ ] Pass arguments to a function and access them with `$1`, `$2`, `$@`, `$*`, and `$#`; explain the difference between `$@` and `$*` when quoted.
- [ ] Write a function that returns a value using `echo` and captures it with command substitution; explain why `return` only returns an exit code (0–255).
- [ ] Use `local` to declare a variable inside a function and demonstrate that it does not leak into the caller's scope.
- [ ] Write a function that validates its arguments and calls `return 1` with an error message to `stderr` if the input is invalid.
- [ ] Write a recursive function that computes the factorial of a number; explain the risk of infinite recursion.
- [ ] Write a library file (`lib.sh`) with utility functions; source it in another script with `. lib.sh` or `source lib.sh`; explain the difference from executing the file.
- [ ] Use a function to wrap a command with logging: log the start time, the command, and the exit code to a log file.

### 1.6 Arrays and Associative Arrays

- [ ] Declare an indexed array `arr=("a" "b" "c")`; access elements with `${arr[0]}`; print all elements with `${arr[@]}`; print the count with `${#arr[@]}`.
- [ ] Append to an array with `arr+=("d")`; delete an element with `unset arr[1]`; iterate over all elements with a `for` loop.
- [ ] Use array slicing `${arr[@]:start:length}` to extract a sub-array.
- [ ] Declare an associative array with `declare -A map`; set key-value pairs; access a value by key; iterate over all keys with `${!map[@]}`.
- [ ] Use an associative array to count word frequencies: read words from a file and increment `freq[$word]` for each.
- [ ] Pass an array to a function by using a nameref (`declare -n`) or by serializing with `"${arr[@]}"` and reconstruct with `("$@")` inside the function.
- [ ] Use `mapfile` (or `readarray`) to read all lines of a file into an array: `mapfile -t lines < file.txt`.

### 1.7 Pipes, Redirection, and File Descriptors

- [ ] Redirect stdout to a file with `>`, append with `>>`, and redirect stderr with `2>` and `2>>`; combine both with `&>` and `2>&1`.
- [ ] Use `/dev/null` to discard output; explain when `command > /dev/null 2>&1` is appropriate.
- [ ] Use a here-document (`<<EOF ... EOF`) to pass multi-line input to a command; use a here-string (`<<<`) to pass a single string.
- [ ] Pipe the output of one command into another using `|`; build a pipeline of three or more commands.
- [ ] Use `tee` to write pipeline output to a file and to stdout simultaneously.
- [ ] Use process substitution `<(command)` to use the output of a command as a file argument: `diff <(sort file1) <(sort file2)`.
- [ ] Open a file descriptor with `exec 3> output.txt`; write to it with `echo "line" >&3`; close it with `exec 3>&-`.
- [ ] Use `exec` to redirect all stdout of a script to a log file: `exec > logfile.txt 2>&1` at the top of the script.
- [ ] Explain the difference between `|` (pipe) and `<(...)` (process substitution); state when each is required.

### 1.8 Shell Expansion

- [ ] Demonstrate brace expansion: `{a,b,c}`, `{1..5}`, and `{01..10}` (zero-padded); use it to create ten numbered files in one command.
- [ ] Demonstrate tilde expansion: `~`, `~/dir`, `~username`; explain that `~` expands to `$HOME`.
- [ ] Demonstrate glob patterns: `*` (any sequence), `?` (single character), `[abc]` (character class), `[!abc]` (negation).
- [ ] Enable `globstar` with `shopt -s globstar` and use `**` to match files recursively.
- [ ] Use `shopt -s nullglob` to prevent a glob from expanding to itself when no files match; explain the bug it prevents.
- [ ] Demonstrate word splitting: explain why `for f in $(ls)` is dangerous and why `"$var"` quoting is always required.
- [ ] Use `printf '%q ' "${arr[@]}"` to print shell-quoted versions of array elements; explain its use in constructing safe command strings.

### 1.9 Environment Variables and Shell Startup

- [ ] Print all environment variables with `env` and `printenv`; access a specific one with `printenv HOME`.
- [ ] Set a variable for a single command without exporting it permanently: `VAR=value command`.
- [ ] Export a variable to child processes with `export VAR=value`; show that an unexported variable is invisible to a subshell.
- [ ] Read the value of an environment variable with a default: `${VAR:-/default/path}`.
- [ ] Explain the load order of Bash startup files: `/etc/profile` → `~/.bash_profile` → `~/.bashrc` → `~/.bash_logout`; state which are for login shells vs interactive non-login shells.
- [ ] Add a directory to `$PATH` in `~/.bashrc`; explain why duplicates accumulate and how to avoid them with a guard: `[[ ":$PATH:" != *":$dir:"* ]] && PATH="$PATH:$dir"`.
- [ ] Use `source ~/.bashrc` to reload startup files without opening a new shell; explain why `exec bash` is sometimes needed instead.
- [ ] Explain `$PS1`, `$PS2`, `$IFS`, `$OLDPWD`, `$PPID`, `$BASHPID`, `$BASH_SUBSHELL`, and `$SHLVL`; print their values and explain each.

### 1.10 Process Management and Exit Codes

- [ ] Run a command in the background with `&`; get its PID with `$!`; wait for it with `wait $!`.
- [ ] Use `jobs`, `fg`, `bg`, `disown`, and `nohup` to manage background jobs; explain the difference between `disown` and `nohup`.
- [ ] Use `kill -SIGTERM PID` and `kill -SIGKILL PID`; explain why SIGKILL cannot be caught or ignored.
- [ ] Send SIGUSR1 to a running script using `kill -USR1 PID`; trap it in the script with `trap 'handler' USR1`.
- [ ] Explain exit codes: `0` (success), `1` (general error), `2` (misuse), `126` (not executable), `127` (not found), `128+N` (killed by signal N).
- [ ] Use `$?` immediately after a command to capture its exit code; demonstrate that running any other command resets `$?`.
- [ ] Use `set -e` to make the script exit on first error; use `command || true` to allow a specific command to fail without exiting.
- [ ] Use `trap 'cleanup' EXIT` to register a cleanup function that runs regardless of how the script exits (normal, error, or signal).
- [ ] Use `trap 'echo "interrupted"; exit 130' INT TERM` to handle Ctrl+C and termination signals gracefully.

### 1.11 Permissions, Ownership, and Script Best Practices

- [ ] Make a script executable with `chmod +x script.sh`; explain the three permission classes (owner, group, other) and the three bits (read, write, execute).
- [ ] Explain the shebang line: `#!/usr/bin/env bash` vs `#!/bin/bash`; state why `env` is more portable.
- [ ] Set restrictive permissions on a script that handles secrets: `chmod 700 script.sh`; explain the risk of world-readable scripts with embedded credentials.
- [ ] Use `umask` to control default file creation permissions; demonstrate how `umask 022` vs `umask 077` affects new files.
- [ ] Explain setuid, setgid, and sticky bits; show how to set the sticky bit on a shared directory with `chmod +t`.
- [ ] Use `shellcheck script.sh` to lint a script; fix every warning it produces and explain the reason behind each one.
- [ ] Write the standard header block for every script: shebang, description, usage, author, and version comment block.
- [ ] Use `set -euo pipefail` at the top of every production script; explain what each option does and why `pipefail` catches pipeline failures that `-e` misses.

### 1.12 Aliases

- [ ] Define an alias with `alias ll='ls -lah --color=auto'`; verify it works interactively; explain why aliases defined in a script are local to that script's shell.
- [ ] List all currently defined aliases with `alias` (no arguments); remove a specific alias with `unalias ll`; remove all aliases with `unalias -a`.
- [ ] Add persistent aliases to `~/.bashrc`; reload it with `source ~/.bashrc`; explain the difference between putting aliases in `~/.bashrc` vs `~/.bash_aliases`.
- [ ] Explain why aliases do **not** expand inside shell scripts by default; demonstrate that running a script with `bash script.sh` does not see aliases, while sourcing it with `. script.sh` does.
- [ ] Enable alias expansion inside a script with `shopt -s expand_aliases`; explain when this is useful and when it is a design smell.
- [ ] Create a safety alias `alias rm='rm -i'` that always prompts before deletion; explain the risk of scripts relying on interactive aliases.
- [ ] Write a function instead of an alias when you need arguments (aliases cannot use `$1`); demonstrate the difference between `alias greet='echo Hello'` and `greet() { echo "Hello $1"; }`.
- [ ] Use `type alias_name` to check whether a name is an alias, a function, a builtin, or an external command; use `which` and explain why it cannot see aliases.

---

## 2. Linux Command-Line Tools Deep Dive

### 2.1 File & Text Processing

- [ ] Use `grep "pattern" file` to search for a fixed string; use `grep -E` (or `egrep`) for extended regex; use `grep -F` for literal fixed strings.
- [ ] Use `grep -r "pattern" dir/` to search recursively; use `-l` to list only filenames; use `-n` to show line numbers; use `-c` to count matches.
- [ ] Use `grep -v "pattern"` to invert the match (exclude lines); use `-i` for case-insensitive search; use `-w` to match whole words only.
- [ ] Use `grep -A 3 -B 2 "pattern"` to print 3 lines after and 2 lines before each match; explain the use in log analysis.
- [ ] Use `grep -oP "(?<=key=)\w+"` with Perl-compatible regex to extract only the matched group from each line.
- [ ] Use `awk '{print $1, $3}'` to print columns 1 and 3 of space-delimited input; change the delimiter with `-F ":"`.
- [ ] Use `awk '/pattern/ {action}'` to filter and process lines matching a regex; print the sum of a numeric column with `awk '{sum+=$2} END {print sum}'`.
- [ ] Use `awk 'NR==5'` to print the fifth line; `awk 'NF>3'` to print lines with more than 3 fields; `awk 'END {print NR}'` to count lines.
- [ ] Use `awk` to extract IP addresses from an access log: `awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10`.
- [ ] Use `sed 's/old/new/g' file` to replace all occurrences in a file; use `-i` to edit the file in-place (add `-i.bak` for a backup).
- [ ] Use `sed '/pattern/d'` to delete matching lines; `sed -n '/pattern/p'` to print only matching lines.
- [ ] Use `sed 'Ns/old/new/'` to substitute only on line N; `sed '2,5s/old/new/'` to substitute in a range.
- [ ] Use `cut -d: -f1,3 /etc/passwd` to extract fields 1 and 3 using `:` as delimiter.
- [ ] Use `cut -c1-10` to extract characters 1 through 10 from each line.
- [ ] Use `tr 'a-z' 'A-Z'` to convert lowercase to uppercase; `tr -d '\r'` to remove carriage returns from a Windows file; `tr -s ' '` to squeeze repeated spaces.
- [ ] Use `sort file` to sort alphabetically; `sort -n` for numeric; `sort -rn` for reverse numeric; `sort -k2,2n` to sort by the second field numerically.
- [ ] Use `sort -u` to sort and remove duplicates simultaneously; compare with `sort | uniq` and explain the difference.
- [ ] Use `uniq -c` to prefix each unique line with its count; `uniq -d` to print only duplicate lines; `uniq -u` to print only lines that appear exactly once.
- [ ] Use `xargs` to pass lines from stdin as arguments: `cat urls.txt | xargs curl -O`; use `-n 1` to process one argument per command invocation.
- [ ] Use `xargs -P 4` to run 4 parallel processes simultaneously; demonstrate with a batch download.
- [ ] Use `xargs -I {}` to place the argument at a specific position: `cat hosts.txt | xargs -I {} ping -c 1 {}`.
- [ ] Use `tee` to log all output of a pipeline while still displaying it: `command | tee output.log | grep ERROR`.
- [ ] Use `split -l 1000 bigfile.txt chunk_` to split a large file into 1000-line chunks; use `split -b 50M` to split by size.
- [ ] Use `paste file1 file2` to merge two files column by column; use `-d","` to use a comma as the delimiter.
- [ ] Use `wc -l`, `wc -w`, and `wc -c` to count lines, words, and bytes in a file.
- [ ] Use `strings -n 8 binary_file` to extract printable strings of length ≥ 8 from a binary; explain its use in forensics and malware analysis.
- [ ] Use `file suspicious_file` to detect the actual type of a file regardless of its extension; explain magic bytes.

### 2.2 Filesystem Operations

- [ ] Use `find /path -name "*.log"` to find files by name; use `-iname` for case-insensitive matching.
- [ ] Use `find /path -type f -mtime -7` to find files modified in the last 7 days; use `-mmin -60` for the last 60 minutes.
- [ ] Use `find /path -size +100M` to find files larger than 100 MB; combine with `-exec rm {} \;` to delete them (use `-ok` for interactive confirmation).
- [ ] Use `find /path -perm /4000` to find setuid files; explain why setuid files are a security concern.
- [ ] Use `find /path -user root -perm /002` to find world-writable files owned by root.
- [ ] Use `find /path -exec grep -l "password" {} \;` to search file contents using `find`+`grep`.
- [ ] Use `locate filename` to search the database for a file; run `updatedb` to refresh the database; explain why `find` is authoritative but `locate` is faster.
- [ ] Use `ls -la` to list all files with permissions, owner, size, and modification time; use `-lh` for human-readable sizes; use `--sort=time` to sort by modification time.
- [ ] Use `stat file` to display detailed metadata: inode number, permissions in octal, size, and all three timestamps (access, modify, change).
- [ ] Use `chmod 755 file`, `chmod u+x file`, `chmod o-w file`, and `chmod a=r file`; explain symbolic vs octal notation.
- [ ] Use `chown user:group file` and `chown -R user:group dir/` to change ownership recursively.
- [ ] Use `ln -s target link_name` to create a symbolic link; use `ln target link_name` (no `-s`) for a hard link; explain the difference and when each is appropriate.
- [ ] Use `tar -czvf archive.tar.gz dir/` to create a compressed archive; `tar -tzvf archive.tar.gz` to list contents without extracting; `tar -xzvf archive.tar.gz -C /dest/` to extract.
- [ ] Use `gzip file` to compress and `gzip -d file.gz` to decompress; compare with `bzip2` and `xz` for compression ratio vs speed.
- [ ] Use `zip -r archive.zip dir/` to create a zip and `unzip archive.zip` to extract; use `unzip -l` to list contents.
- [ ] Use `rsync -avz source/ dest/` to synchronize directories; use `--dry-run` to preview changes; use `--delete` to remove files in dest not in source; use `-e ssh` to sync over SSH.

### 2.3 Networking

- [ ] Use `curl -s -o /dev/null -w "%{http_code}" URL` to check HTTP response code silently.
- [ ] Use `curl -H "Authorization: Bearer TOKEN" URL` to send a Bearer token; use `-H "Content-Type: application/json"` with `-d '{"key":"val"}'` for a JSON POST.
- [ ] Use `curl -L` to follow redirects; `-k` to skip TLS verification (only in controlled lab environments); `-I` to fetch only headers; `-v` to show full request/response debug.
- [ ] Use `curl -x http://proxy:8080 URL` to route a request through a proxy; explain the security implication of using `--insecure` in production.
- [ ] Use `curl -o file.tar.gz URL` to download a file; use `--limit-rate 500k` to throttle bandwidth.
- [ ] Use `wget -q -O- URL` to download and print to stdout; use `--spider` to check if a URL is reachable without downloading; use `-r --no-parent -l 2` for recursive shallow download.
- [ ] Use `ssh user@host "command"` to run a command on a remote host; use `-i keyfile` for key-based auth; use `-p port` for a non-standard port.
- [ ] Use `ssh -N -L 8080:target:80 jumphost` to set up a local port forward (tunnel); explain the use case in pentesting.
- [ ] Use `scp -r dir/ user@host:/remote/path/` to copy a directory to a remote host; use `-P port` for non-standard ports.
- [ ] Use `nc -lvnp 4444` to listen on a port; `nc host 4444` to connect; use `nc` to transfer a file: `nc -lvnp 1234 > out.txt` on receiver, `nc host 1234 < in.txt` on sender.
- [ ] Use `nc -z host 80` to test if a port is open (port scan a single port); loop over a range to scan multiple ports.
- [ ] Use `socat TCP-LISTEN:4444,fork EXEC:/bin/bash` to bind a shell to a port in a lab; explain why this is only safe in isolated lab environments.
- [ ] Use `dig domain A` to query A records; `dig domain MX` for mail records; `dig domain NS` for nameservers; `dig domain TXT` for text records; `dig @8.8.8.8 domain` to query a specific resolver.
- [ ] Use `dig +short domain A` for compact output; `dig -x IP` for reverse DNS lookup.
- [ ] Use `host domain` and `nslookup domain` as quick DNS lookup alternatives to `dig`; explain when `dig` is preferable.
- [ ] Use `ping -c 4 host` to test reachability; interpret TTL values to infer the operating system.
- [ ] Use `traceroute host` (or `tracepath`) to trace the network path; explain what `* * *` means in the output.
- [ ] Use `ss -tulnp` to list all listening TCP and UDP ports with their process names; compare with the legacy `netstat -tulnp`.
- [ ] Use `tcpdump -i eth0 -n -w capture.pcap` to capture packets on an interface to a file; use `tcpdump -r capture.pcap` to read it back; filter with `tcpdump -i eth0 port 80`.
- [ ] Use `tcpdump -i eth0 host IP and port 443 -w tls.pcap` to capture traffic to/from a specific host and port; explain responsible capture practices (only on networks you own or are authorized to monitor).

### 2.4 System Administration

- [ ] Use `ps aux` to list all running processes; use `ps -ef` for a full-format listing; explain the meaning of each column.
- [ ] Use `ps aux | grep "process_name"` to find a process; use `pgrep process_name` as a cleaner alternative; use `pgrep -u root` to list root's processes.
- [ ] Use `top` interactively: sort by CPU (`P`), memory (`M`), and PID (`N`); press `k` to kill a process; explain load average (1, 5, 15-minute).
- [ ] Use `htop` for an interactive process viewer; explain the difference from `top` and when `htop` is not available.
- [ ] Use `kill PID` (sends SIGTERM), `kill -9 PID` (SIGKILL), and `kill -l` to list all signals; use `killall process_name` and `pkill -f pattern`.
- [ ] Use `systemctl status service_name` to check a service; `systemctl start`, `stop`, `restart`, `enable`, `disable`; explain the difference between enable/disable and start/stop.
- [ ] Use `systemctl list-units --type=service --state=running` to list all running services.
- [ ] Use `journalctl -u service_name` to view logs for a service; use `-f` to follow live; use `--since "1 hour ago"` to filter by time; use `-p err` to show only errors.
- [ ] Use `journalctl -b` to view logs since last boot; use `journalctl --disk-usage` to check log disk usage.
- [ ] Use `crontab -e` to edit cron jobs; write a cron expression for: every minute, every day at midnight, every Monday at 8 AM, and the 1st of every month.
- [ ] Explain the crontab field order: `minute hour day-of-month month day-of-week command`; use `crontab -l` to list current jobs and `crontab -r` to remove all.
- [ ] Use `env` to print all environment variables; use `env -i command` to run a command with a clean empty environment.
- [ ] Use `export VAR=value` to set an environment variable for child processes; use `unset VAR` to remove it.
- [ ] Use `history` to view command history; use `!N` to re-run command N; use `!!` for the last command; use `Ctrl+R` for reverse search.
- [ ] Use `sudo command` to run as root; use `sudo -u user command` to run as a specific user; use `sudo -l` to list allowed commands; explain why `sudo su -` is different from `sudo -i`.
- [ ] Use `service service_name status` for SysV-style service management; explain when `service` and `systemctl` coexist on the same system.

### 2.5 Data Processing

- [ ] Install `jq` and use `jq '.'` to pretty-print a JSON file; use `jq '.key'` to extract a field; use `jq '.[] | .name'` to iterate an array.
- [ ] Use `jq 'select(.status == "active")'` to filter JSON objects; use `jq '.[] | {name, age}'` to reshape each object.
- [ ] Use `jq -r '.token'` to output raw strings without JSON quotes; use `jq -c '.'` for compact single-line output.
- [ ] Use `jq 'keys'` to list all keys in a JSON object; use `jq 'length'` to count array elements.
- [ ] Use `curl -s URL | jq '.results[].url'` to extract URLs from an API response in one pipeline.
- [ ] Install `yq` and use `yq '.key'` to extract a value from a YAML file; use `yq -i '.key = "value"'` to edit a YAML file in-place.
- [ ] Use `base64` to encode a string: `echo -n "secret" | base64`; decode with `echo "c2VjcmV0" | base64 -d`; explain the `-n` flag's importance.
- [ ] Use `base64 -w 0` to produce output with no line wrapping (required for some APIs); decode a base64-encoded file using `base64 -d file.b64 > file`.
- [ ] Use `md5sum file` to compute an MD5 hash; use `sha256sum file` for SHA-256; verify a downloaded file against a published checksum.
- [ ] Use `sha256sum -c checksums.txt` to batch-verify multiple files against a checksum file; explain why MD5 is insufficient for integrity verification of untrusted files.

---

## 3. Mini Projects (Fundamentals)

- [ ] Build a menu-driven script that presents numbered options (Add, Delete, List, Quit) using a `while` loop and a `case` statement; validate all user input.
- [ ] Build a backup script: accept a source directory and destination path as arguments; create a timestamped `.tar.gz` archive; log the result with the archive size.
- [ ] Build a log rotation script: find log files older than 7 days in a directory; compress them with `gzip`; delete any compressed logs older than 30 days.
- [ ] Build a bulk file renamer: accept a directory, a pattern, and a replacement string; rename all matching files using parameter substitution; add a `--dry-run` mode.
- [ ] Build a system info reporter: collect hostname, OS version, uptime, CPU count, total/free RAM, and disk usage; format the output as a readable report and save to a file.
- [ ] Build a simple config file parser: read a `KEY=VALUE` file, skip comment lines starting with `#`, and export each variable; validate that required keys are present.
- [ ] Build a script that monitors a directory for new files using an infinite loop with `sleep 5`; when a new `.csv` file appears, count its lines and append a summary to a report file.
- [ ] Build a batch URL checker: read URLs from a file one per line; use `curl -s -o /dev/null -w "%{http_code}"` on each; output a CSV of URL and status code.

---

## 4. Linux System Programming Basics

### 4.1 Processes and Job Control

- [ ] Explain `fork`: a child process is a copy of the parent; show that `$$` (current PID) differs in a subshell created with `( )`.
- [ ] Use `( command )` to run a command in a subshell; explain that variable changes inside do not affect the parent.
- [ ] Demonstrate the difference between `command &` (background), `command` (foreground), and `( command ) &` (subshell in background).
- [ ] Use `wait` to wait for all background jobs; use `wait $PID` to wait for a specific job; capture its exit code with `$?`.
- [ ] Use `jobs -l` to list active jobs with their PIDs; use `fg %1` to bring job 1 to foreground; use `bg %2` to resume job 2 in the background.
- [ ] Use `disown %1` to detach a job from the shell so it continues after logout; compare with `nohup command &`.
- [ ] Read `/proc/self/status` from a script to retrieve its own PID, parent PID, and memory usage.
- [ ] Read `/proc/uptime` to get system uptime in seconds; format it into days, hours, minutes.

### 4.2 Signals

- [ ] List the most important Linux signals: `SIGHUP` (1), `SIGINT` (2), `SIGQUIT` (3), `SIGKILL` (9), `SIGTERM` (15), `SIGUSR1` (10), `SIGUSR2` (12), `SIGCHLD` (17), `SIGPIPE` (13).
- [ ] Use `trap 'echo "SIGINT received"; exit 130' INT` to handle Ctrl+C cleanly in a script.
- [ ] Use `trap 'cleanup_function' EXIT` to ensure temporary files are removed even if the script is interrupted.
- [ ] Use `trap '' SIGPIPE` to suppress broken pipe errors when writing to a closed pipe (e.g., piping to `head`).
- [ ] Send a signal to a background job from within the same script: capture `$!`, then `kill -USR1 $!` after a delay.
- [ ] Explain why `SIGKILL` cannot be trapped or ignored; describe scenarios where a process must be force-killed with `-9`.

### 4.3 File Descriptors and Named Pipes

- [ ] List the three standard file descriptors: `0` (stdin), `1` (stdout), `2` (stderr); demonstrate redirecting each independently.
- [ ] Create a named pipe (FIFO) with `mkfifo mypipe`; write to it in one terminal and read from it in another; clean up with `rm mypipe`.
- [ ] Use a FIFO to connect two scripts without a temporary file: `script1.sh > mypipe &` and `script2.sh < mypipe`.
- [ ] Open a custom file descriptor for writing: `exec 5> output.txt`; write to it with `echo "data" >&5`; close it with `exec 5>&-`.
- [ ] Redirect stderr of an individual command to a file while keeping stdout on the terminal: `command 2> errors.log`.
- [ ] Capture both stdout and stderr into separate variables using process substitution: `{ stdout=$(command 2>/tmp/err); stderr=$(cat /tmp/err); }`.

### 4.4 Cron Jobs and Scheduling

- [ ] Write a cron job that runs a backup script every day at 2 AM: `0 2 * * * /home/user/backup.sh >> /var/log/backup.log 2>&1`.
- [ ] Write a cron job that runs every 15 minutes: `*/15 * * * * /path/to/script.sh`.
- [ ] Write a cron job that runs only on weekdays at 8 AM: `0 8 * * 1-5 /path/to/script.sh`.
- [ ] Use `crontab -l > my_crontab.txt` to back up cron jobs; use `crontab my_crontab.txt` to restore them.
- [ ] Place a script in `/etc/cron.daily/`, `/etc/cron.weekly/`, or `/etc/cron.hourly/` and explain how `run-parts` executes them.
- [ ] Use `at now + 5 minutes` to schedule a one-time job; use `atq` to list pending jobs; use `atrm` to remove one.

### 4.5 System Logging

- [ ] View live system logs with `journalctl -f`; filter by service with `-u nginx`; filter by priority with `-p warning`.
- [ ] Use `logger "My script started"` to write a message to the system log from a script; read it back with `journalctl -t my_tag`.
- [ ] Add structured logging to a script: prefix every log line with a timestamp and severity using a `log()` function that writes to both a file and stderr.
- [ ] Rotate a custom log file manually using `logrotate`; write a `/etc/logrotate.d/myscript` configuration with `daily`, `rotate 7`, `compress`, and `missingok`.
- [ ] Read `/var/log/auth.log` (or `journalctl -u ssh`) to find failed SSH login attempts; extract unique source IPs with `grep`, `awk`, and `sort | uniq -c | sort -rn`.

### 4.6 Package Management

- [ ] Use `apt update && apt upgrade -y` (Debian/Ubuntu) to update all packages; explain the difference between `update` (refresh index) and `upgrade` (install updates).
- [ ] Use `apt install package_name`, `apt remove`, and `apt purge` (also removes config files); use `apt search keyword`.
- [ ] Use `dpkg -l | grep package` to check if a package is installed; use `dpkg -L package` to list its installed files.
- [ ] Use `which command` and `type command` to find where a binary lives; use `command -v cmd` in scripts (more portable than `which`).
- [ ] Write a script that checks if a list of required tools (`curl`, `jq`, `nmap`, `nc`) are installed and exits with an error listing the missing ones.

---

## 5. Automation Projects

- [ ] Build a **host enumeration script**: accept a CIDR range or list of IPs; `ping -c 1` each host; report live and dead hosts to separate files; run checks in parallel with `&` and `wait`.
- [ ] Build a **network reconnaissance script**: accept a target IP; use `nc -z` to scan a configurable port range; log open ports with timestamps; limit to authorized targets only.
- [ ] Build a **subdomain enumeration script**: accept a domain and a wordlist file; use `dig +short ${sub}.${domain}` for each word; record subdomains that resolve to an IP; skip CNAMEs pointing off-domain.
- [ ] Build a **log analysis script**: accept a log file path and a date filter; extract lines matching the date; count occurrences by HTTP status code; report the top 10 most requested URLs using `awk` and `sort`.
- [ ] Build a **file processing pipeline**: watch a drop directory for new `.csv` files; for each file, validate column count, deduplicate rows with `sort -u`, count records, and move processed files to an archive folder.
- [ ] Build a **backup automation script**: back up specified directories to a remote host using `rsync` over SSH; keep a rolling 7-day retention by deleting archives older than 7 days; email a summary using `mail` or `sendmail`.
- [ ] Build a **system monitoring script**: check CPU load (from `/proc/loadavg`), free memory (from `free -m`), disk usage (from `df -h`), and top 5 CPU-consuming processes (from `ps aux`); alert via `logger` when thresholds are exceeded.
- [ ] Build a **cron-driven report generator**: run daily at 6 AM; collect yesterday's log stats (error count, warning count, unique IPs); write an HTML report to `/var/www/html/report.html`.
- [ ] Build an **SSH automation script**: read a list of hosts from a file; run a command on each via `ssh -o BatchMode=yes -o ConnectTimeout=5`; collect stdout and exit codes; skip unreachable hosts gracefully.
- [ ] Build a **service monitor**: loop every 60 seconds; check that a list of services (`nginx`, `postgresql`, `redis`) are active using `systemctl is-active`; restart any failed service and log the event.
- [ ] Build a **package update automation script**: run `apt update`; collect the list of upgradable packages; log them; apply upgrades only if fewer than 20 packages are pending (to avoid large unreviewed updates); send results to a log file.
- [ ] Build an **IOC extractor**: read a directory of text files; extract and deduplicate IPv4 addresses, domain names, MD5/SHA256 hashes, and URLs using `grep -oE` with appropriate patterns; output each category to a separate file.
- [ ] Build a **firewall management script**: wrap `ufw` (or `iptables`) to accept arguments (`--allow port`, `--deny port`, `--status`, `--reset`); validate port numbers; log every rule change with a timestamp; require `--confirm` for destructive operations; run only on authorized systems you control.
- [ ] Build a **batch vulnerability scanner**: accept a list of `host:port` pairs and a list of `nmap` scripts to run; run each scan sequentially or in parallel with a configurable concurrency limit; parse output to extract CVE identifiers; produce a deduplicated CSV report of `host,port,cve`.

---

## 6. Security & Pentesting Automation

> ⚠️ All exercises in this section must be performed **only** against systems, networks, and applications you own or have explicit written authorization to test. Never run these scripts against production systems or third-party infrastructure without permission.

### 6.1 Network Scanning Automation

- [ ] Write a Bash wrapper around `nmap` that accepts a target, output directory, and scan profile (`quick`, `full`, `stealth`); save results in both XML (`-oX`) and grepable (`-oG`) formats.
- [ ] Parse `nmap -oG` output with `grep` and `awk` to extract open ports and service names; output a clean `host:port:service` CSV.
- [ ] Write a script that runs `nmap --script vuln` against a target and extracts CVE identifiers from the output using `grep -oP 'CVE-[0-9\-]+'`.
- [ ] Write a port scanner using only `bash` and `nc -z`: loop over an IP list and port list; log `OPEN` results with timestamps; run checks in parallel.

### 6.2 HTTP Requests and Web Enumeration

- [ ] Write a `curl`-based HTTP header inspector: fetch response headers with `curl -sI URL`; extract and display `Server`, `X-Powered-By`, `Content-Security-Policy`, `Strict-Transport-Security`, and `X-Frame-Options`.
- [ ] Write a directory brute-forcing script: read a wordlist; `curl -s -o /dev/null -w "%{http_code}" base_url/$word` for each word; record all non-404 responses with their status codes and content lengths.
- [ ] Write a script that tests for open redirects: append `?redirect=https://evil.com` to a list of URLs; use `curl -sI -L` to follow redirects; flag any response that ends at a different domain.
- [ ] Write a virtual host brute-forcer: send `curl -sI -H "Host: $vhost" target_ip` for each entry in a hostname wordlist; record responses with non-standard status codes or different `Content-Length` values.

### 6.3 DNS Enumeration

- [ ] Write a DNS zone-transfer attempt script: use `dig axfr @nameserver domain`; check the exit code; if successful, parse and save all records.
- [ ] Write a DNS brute-force script using a wordlist: query `${word}.${domain}` using `dig +short` for each word; record resolved entries; skip non-resolving entries silently.
- [ ] Write a reverse DNS sweep script: accept a CIDR block; iterate over all IPs; run `dig -x IP +short` on each; log hostnames that resolve.
- [ ] Write a script that collects all DNS record types for a domain (`A`, `AAAA`, `MX`, `NS`, `TXT`, `SOA`, `CNAME`) using `dig` in a loop; output a structured report.

### 6.4 SSL/TLS Certificate Inspection

- [ ] Use `echo | openssl s_client -connect host:443 2>/dev/null | openssl x509 -noout -subject -issuer -dates` to extract certificate details.
- [ ] Write a script that checks whether a domain's TLS certificate expires within 30 days: parse `notAfter` from `openssl x509`; compare with `date +%s`; alert if expiry is imminent.
- [ ] Write a batch certificate checker: read a list of `host:port` pairs from a file; check the certificate expiry for each; output a CSV of domain, expiry date, and days remaining.
- [ ] Use `openssl s_client -connect host:443 < /dev/null 2>&1 | grep -E "Protocol|Cipher"` to check the TLS protocol version and cipher suite.

### 6.5 Log Parsing and Privilege Enumeration

- [ ] Write a script that parses `/var/log/auth.log` to extract: failed login attempts, successful logins, source IPs, and usernames; output summary counts per IP.
- [ ] Write a Linux privilege enumeration script (for authorized systems): collect SUID/SGID files (`find / -perm /4000 -o -perm /2000`), world-writable directories, cron jobs (`crontab -l`, `/etc/cron*`), and sudo rules (`sudo -l`); output a formatted report.
- [ ] Write a script that identifies users with empty passwords: `awk -F: '$2 == "" {print $1}' /etc/shadow` (run as root); log findings and warn.
- [ ] Write a Bash-based Whois lookup script: read a list of IPs or domains; run `whois` on each; extract and log `OrgName`, `Country`, and `NetRange` fields.

---

## 7. Debugging & Best Practices

### 7.1 ShellCheck and Static Analysis

- [ ] Install `shellcheck` and run it on a script you wrote; fix every warning; explain the reason behind each fix.
- [ ] Enable `shellcheck` in your editor (VSCode extension or vim ALE) so you get real-time feedback while writing.
- [ ] Run `shellcheck -e SC2086` to suppress a specific warning and understand when suppression is justified vs when you should fix the code.
- [ ] Write a script intentionally containing five common ShellCheck-detected bugs (unquoted variable, `[ ]` instead of `[[ ]]`, `cd` without error check, `for f in $(ls)`, missing shebang); fix each one after `shellcheck` reports it.

### 7.2 Bash Debugging Techniques

- [ ] Add `set -x` at the top of a script to trace every command before execution; remove it by adding `set +x` before a sensitive section.
- [ ] Add `set -e` to exit immediately on error; demonstrate that a failed command in a pipeline is NOT caught without `set -o pipefail`; add `set -o pipefail` and show the difference.
- [ ] Add `set -u` to treat unset variables as errors; demonstrate an `unbound variable` error and fix it with a default value.
- [ ] Use the standard trio `set -euo pipefail` at the top of every production script; explain what each flag does independently.
- [ ] Use `bash -n script.sh` to syntax-check a script without executing it; add this to a pre-commit hook.
- [ ] Use `bash -x script.sh 2> debug.log` to write the trace to a file without cluttering the terminal.
- [ ] Add `PS4='+(${BASH_SOURCE}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'` before `set -x` to include file name, line number, and function name in trace output.
- [ ] Use `trap 'echo "Error at line $LINENO, exit $?"' ERR` to print the line number whenever the script hits an error.

### 7.3 Error Handling and Logging

- [ ] Write a `die()` function that prints a message to `stderr` and exits with a given code: `die() { echo "[ERROR] $*" >&2; exit 1; }`.
- [ ] Write a `log()` function with severity levels (INFO, WARN, ERROR) that prefixes each line with a timestamp and the level name.
- [ ] Add `|| die "message"` after every critical command (file open, network call, `cd`) so failures are caught immediately.
- [ ] Validate all script arguments at startup: check count with `$#`, check file existence with `[[ -f ]]`, and check numeric values with `[[ $var =~ ^[0-9]+$ ]]`.
- [ ] Write a script that creates a temporary directory with `TMPDIR=$(mktemp -d)` and registers `rm -rf "$TMPDIR"` with `trap ... EXIT` to ensure cleanup.
- [ ] Use `flock` to ensure only one instance of a script runs at a time: `exec 9>/var/lock/myscript.lock; flock -n 9 || die "Already running"`.

### 7.4 Modular Script Design

- [ ] Split a large script into `main.sh`, `lib/utils.sh`, `lib/network.sh`, and `lib/logging.sh`; source the libraries at the top of `main.sh`.
- [ ] Write a `usage()` function that prints a help message and is called when `--help` is passed or arguments are invalid.
- [ ] Implement argument parsing with `getopts` for short options (`-h`, `-v`, `-o file`) and explain its limitations vs a full parser.
- [ ] Implement long option parsing manually with a `while` loop and a `case` statement (`--output`, `--verbose`, `--dry-run`).
- [ ] Add a `--dry-run` mode to every script that performs destructive or network actions; in dry-run mode, print what the script would do without doing it.
- [ ] Use `readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"` to find the script's own directory portably.

### 7.5 Git for Script Version Control

- [ ] Initialize a Git repository for your scripts; write a `.gitignore` that excludes `.env`, `*.log`, `*.key`, `*.pem`, and `secrets/`.
- [ ] Use `git-secrets` or `trufflehog` to scan a repository for committed credentials; remediate any findings with `git filter-repo`.
- [ ] Write a `pre-commit` hook that runs `shellcheck` on all staged `.sh` files and blocks the commit if any errors are found.
- [ ] Use `git log --oneline --all` to review history; use `git diff HEAD~1` to see what changed in the last commit.
- [ ] Write a commit message convention for scripts: `fix:`, `feat:`, `chore:`, `sec:` — explain why descriptive messages matter for script maintenance.

### 7.6 Portable Shell Scripting

- [ ] Change the shebang to `#!/bin/sh` and run the script with `dash script.sh`; fix every bashism that `dash` rejects: `[[`, `((...))`, `local`, `declare`, arrays, `&>`, `source` (use `.` instead).
- [ ] Use `shellcheck --shell=sh script.sh` to lint for POSIX compliance; understand why SC2039 (bash-specific feature used in sh script) is raised.
- [ ] Rewrite a `[[ ]]` test using POSIX `[ ]`: replace `[[ -z $VAR ]]` with `[ -z "$VAR" ]`; replace `[[ str =~ regex ]]` with `expr` or `grep`.
- [ ] Rewrite a `for (( i=0; i<10; i++ ))` C-style loop using a POSIX `while` loop with `i=$((i+1))`.
- [ ] Avoid `local` in sh by using subshells to scope variables: replace a function using `local var` with one that runs its body in `( )` instead.
- [ ] Use `printf` instead of `echo` for portable output: explain why `echo -e`, `echo -n`, and `echo "\t"` behave differently across shells and why `printf` is consistent.
- [ ] Explain when portability matters (scripts that must run on Alpine Linux with `busybox sh`, macOS `/bin/sh`, or minimal container images) and when it is fine to require Bash explicitly.
- [ ] Test a script against multiple interpreters: `bash`, `dash`, and `busybox sh`; document which features you depend on and why.

### 7.7 Testing with BATS

- [ ] Install `bats-core` (`git clone https://github.com/bats-core/bats-core.git` or via package manager); run `bats --version` to confirm it is installed.
- [ ] Write a `test_utils.bats` file with three `@test` blocks: one that checks a function returns `0` on valid input, one that checks it returns `1` on invalid input, and one that checks its stdout output.
- [ ] Use `run command` inside a BATS test to capture exit code (`$status`) and output (`$output`) separately; use `[ "$status" -eq 0 ]` and `[[ "$output" == *expected* ]]` assertions.
- [ ] Use `setup()` and `teardown()` functions to create and remove temporary files before and after each test.
- [ ] Use `setup_file()` and `teardown_file()` to run setup once for the entire test file rather than per test.
- [ ] Write a test that mocks an external command: create a fake `curl` function that returns a controlled exit code and output; unset it in `teardown`.
- [ ] Run a test suite with `bats tests/`; interpret the TAP output (ok / not ok lines); use `--tap` and `--junit` flags to produce machine-readable output for CI.
- [ ] Add a BATS test run to the Git `pre-push` hook: `bats tests/ || exit 1`; explain why running tests before push catches regressions early.

---

## 8. Secure Scripting Practices

- [ ] Never store passwords, API keys, or tokens as plaintext variables in a script; read them from environment variables or a `~/.config/tool/credentials` file with permissions `600`.
- [ ] Validate all external input (user arguments, file contents, API responses) before using it in a command; reject input that does not match an expected pattern.
- [ ] Quote every variable expansion: use `"$VAR"` not `$VAR`; use `"${arr[@]}"` not `${arr[@]}`; explain how unquoted variables cause word-splitting and globbing bugs.
- [ ] Avoid constructing commands by string concatenation; use arrays for commands: `cmd=("curl" "-s" "$url")` and run with `"${cmd[@]}"`.
- [ ] Sanitize filenames before using them in commands: use `--` to end option parsing (`rm -- "$file"`) and validate that filenames do not start with `-`.
- [ ] Use `mktemp` or `mktemp -d` for temporary files; never use predictable names like `/tmp/script.tmp` which are vulnerable to symlink attacks.
- [ ] Run scripts with the minimum required privileges; avoid running entire scripts as root; use `sudo command` only for the specific command that needs it.
- [ ] Add a scope comment to every security-related script: state the authorized target, the purpose, and who authorized the test.
- [ ] Use `set -euo pipefail` and `trap 'cleanup' EXIT` in every production script without exception.
- [ ] Review every script against the OWASP Shell Injection guidance before deployment; never pass unsanitized user input to `eval`, backticks, or `$(...)`.

---

## 9. Learning Outcome

- [ ] Write a complete, professional Bash script from scratch — shebang, `set -euo pipefail`, argument parsing, logging, error handling, cleanup trap — without referencing templates.
- [ ] Use `grep`, `awk`, `sed`, `cut`, `sort`, `uniq`, `xargs`, and `jq` fluently in pipelines to process real log files, CSV data, and JSON API responses.
- [ ] Build a modular automation suite with a shared library, consistent logging, a `--dry-run` mode, and a `--help` flag.
- [ ] Diagnose a broken script using `set -x`, `shellcheck`, and `bash -n`; identify the root cause and fix it without guessing.
- [ ] Automate a complete operational workflow (backup, monitoring, or log analysis) that runs reliably as an unattended cron job.
- [ ] Write a security automation script (port scan parser, certificate checker, or log anomaly detector) that handles errors gracefully and produces a structured report.
- [ ] Explain the output of `ss -tulnp`, `ps aux`, `journalctl`, and `/proc/self/status` without looking them up.
- [ ] Obtain explicit written authorization before running any enumeration, scanning, or reconnaissance script against systems you do not personally own.
