Week 01: Linux filesystem, find, pipes and logs

Dates: Sep 28 – Oct 4, 2026 Environment: Ubuntu Server 24.04 LTS (no GUI) in VMware Workstation, accessed via SSH from Windows 11

1. Lab setup

Installed Ubuntu Server in VMware (2 vCPU, 4 GB RAM, 25 GB thin-provisioned disk) with OpenSSH enabled during installation. The VM uses a NAT network adapter, and I connected to it from Windows 11 PowerShell with ssh user@<vm-ip>.

Note: The login message showed Last login ... from 192.168.25.1. This is my Windows host as seen from VMware's virtual NAT network, not my home network. Windows and the VM share this private virtual network.

2. Creating directories and files in one command
bash
mkdir -p ~/lab1/projects/{web,db,logs}
touch ~/lab1/projects/logs/app{1..5}.log
mkdir -p creates the directory and any missing parent directories (lab1, projects). Without -p, it fails because the parents do not exist.
{web,db,logs} and {1..5} are brace expansion. The shell (bash) expands them before the command runs. The command never sees the braces, only the final list of names.
touch creates an empty file if it does not exist.

Tip: Run echo first to preview what the shell will expand. This is especially important before destructive commands like rm.

Mistake: echo app{app1.log,app2.log,app3.log,app4.log,app5.log}.log

What happened: I got appapp1.log.log instead of app1.log.
Why: Text outside the braces is added to every item inside them.
Fix: Put only the part that changes inside the braces: app{1..5}.log. Thanks to echo, I caught this before creating any files.
3. Counting .conf files in /etc
bash
find /etc -name '*.conf' | wc -l
find /etc searches /etc recursively, including all subdirectories.
-name '*.conf' matches file names ending in .conf. The quotes stop the shell from expanding *, so find receives the pattern itself.
| wc -l passes the list to wc, which counts the lines (one line = one file).

Result: 157 files visible to my user (a few directories returned Permission denied, so the real total is higher and would need sudo).

Mistake 1: find *.conf (run from inside /etc)

What happened: I got only about 28 files instead of 157.
Why: The shell sees * and expands it before find runs, replacing it with matching .conf files from the current directory. So find never receives the pattern *.conf. It only gets a list of file names and does not search recursively through /etc subdirectories.
Fix: Quote the pattern and tell find where to search: find /etc -name '*.conf'.

Mistake 2: find /etc -name '*.conf' && ls /etc | wc -l

What happened: It counted every item in /etc, not just the .conf files. The find results were only printed to the screen and were never counted.
Why: && runs commands independently (the second one only if the first succeeds), while | passes the output of one command as input to the next. Example: "Give me all bikes && give me red bikes" returns 2 separate lists, but "Give me all bikes | give me red bikes" returns 1 list with only red bikes.
Fix: find /etc -name '*.conf' | wc -l. I replaced && with | and removed ls /etc, because ls ignores input from a pipe and just lists its own directory, so the find results would be lost.
4. Finding the 3 largest files in /var/log
bash
sudo find /var/log -type f | sudo xargs du -h | sort -hr | head -n3
sudo runs the command with root privileges (some logs are readable only by root).
find /var/log -type f finds regular files only (no directories) in /var/log and its subdirectories.
| passes the output of one command as input to the next.
xargs converts the incoming list into arguments for the next command. This is needed because du does not read from a pipe; it only works on file names given as arguments.
du -h shows the size of each file in human-readable units (K, M, G).
sort -hr sorts by human-readable size (-h) in reverse order (-r), largest first.
head -n3 keeps only the first 3 lines.
Why sudo twice: sudo applies only to the command right after it, not to the whole pipeline. Both find and du need root access to read every file.

Result: the largest files were systemd journal files in /var/log/journal/ (8.0M, 8.0M, 5.1M). These are binary logs, read with journalctl (week 4).

Mistakes along the way:

find /var/log du | sort | head
What happened: find: 'du': No such file or directory, and the output was sorted alphabetically.
Why: find treated du as a second place to search. du is a separate command, not a find option. With no sizes in the output, sort could only sort by name.
Fix: Start the pipeline with the command that produces the data (du), and give it the path as an argument.
sudo du -a /var/log | sort -r | head -n3
What happened: Three 8 KB files appeared as the "largest", which made no sense.
Why: Without -n or -h, sort compares numbers as text, character by character, so "8" comes before "4096". I replaced -h with -r instead of combining them.
Fix: Combine flags: sort -nr (plain numbers) or sort -hr (K/M/G).
sudo du -ah /var/log | sort -nr | head -n3
What happened: The biggest result was 696K, but /var/log itself was about 33 MB.
Why: -n reads 33M as 33 and 696K as 696, ignoring the units. du and sort must use the same format: du -a with sort -n, or du -ah with sort -h.
Fix: sort -hr.
sudo du -a /var/log | sort -nr | head -n3
What happened: The top 3 were /var/log, /var/log/journal and a journal subdirectory.
Why: du reports directories too, and a directory's size is the total of everything inside it, so directories always win.
Fix: Use find -type f to pass only regular files to du through xargs.
5. Reading and monitoring logs
bash
tail -n20 /var/log/syslog      # last 20 lines
tail -fn20 /var/log/syslog     # last 20 lines, then keep following new lines
logger "Test message"          # write a test entry to syslog (from a second SSH session)
tail shows the end of a file. New log entries are added at the bottom, so this shows the most recent events.
-f (--follow) keeps the file open and prints new lines as they are written. Ctrl+C stops it.
Combined flags: -fn20 works because -n (the flag that takes a value) is last. -nf20 would fail.
My test entry appeared instantly in the first window: ... mac0179 matu: Test message.

Anatomy of a log line:

2026-10-01T12:26:00.507650+00:00  mac0179  systemd-timesyncd[655]:  Contacted time server ...
        when (UTC)                 host     process [PID]           message

Entries written by logger show the user name (matu) instead of a process name with a PID.

Observation: syslog contained many multipathd ... failed messages. These are harmless on a VMware virtual disk (it has no unique WWID). Not every "error" in a log is a real problem. First ask: what is actually not working?

Key lessons
The shell expands {} and * before the command runs. Use echo to preview, and quotes to stop expansion.
&& vs |: && runs commands one after another; | sends one command's output into the next. Commands that ignore input (ls, du, find) belong at the start of a pipeline.
Look up flags instead of guessing: command --help | grep -i keyword, or man command and search with /.
Sanity-check results. A command that runs without errors can still give wrong output. Compare the result with what you already know.
Debug long commands by running each part separately.
