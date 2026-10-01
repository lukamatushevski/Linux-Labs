# Linux-Labs
Hands-on Linux labs I complete while preparing for an IT Support / SysAdmin role.
Each week I document the commands I used, the mistakes I made, and what I learned.

##Lab Environment
- Host: Windows 11, Vmware Workstation
- Server: Ubuntu Server 24.04 LTS (no GUI), accessed via SSH

##Progress
Week | Topic | Notes
01   | Filesystem navigation | find, pipes, logs

##Skills practiced so far
- Set up an Ubuntu Server VM in VMware and connected to it over SSH from Windows (NAT network)
- Navigated the filesystem and created directory structures and files in one command (`mkdir -p`, `touch`, bash brace expansion `{a,b}` / `{1..5}`)
- Searched for files with `find` (`-name`, `-type f`) and counted results with `wc -l`
- Built multi-step pipelines to find the largest log files (`find | xargs du | sort -hr | head`)
- Read and monitored system logs in real time (`tail -n`, `tail -f`) and wrote test entries with `logger`
- Looked up command options with `--help`, `man` and `grep` instead of guessing
- Learned to sanity-check output: a command that runs without errors can still give wrong results (e.g. `sort -n` vs `sort -h`)
