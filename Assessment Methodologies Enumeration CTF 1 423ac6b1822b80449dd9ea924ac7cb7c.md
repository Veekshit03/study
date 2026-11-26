# Assessment Methodologies: Enumeration CTF 1

## Lab overview

This module covers Enumeration CTF 1 in the INE Assessment Methodologies track. The exercises focus on discovering and accessing exposed SMB/Samba shares, assessing account authentication weaknesses in the authorized lab, identifying an FTP service and testing the supplied lab accounts, and interpreting an SSH login warning.

Enumeration is the process of making structured requests to identified services to learn what information and resources they expose. Unlike a basic port scan, which identifies reachable ports and probable services, enumeration examines service-specific details such as share names, account names, access permissions, directory contents, and login banners.

> **Scope and safety:** Run the commands and authentication tests in this module only against the assigned INE target or another system for which you have explicit written authorization. Password-guessing examples are restricted to the deliberately vulnerable training environment. Do not reuse these wordlists or procedures against public systems or real user accounts.
> 

## Learning objectives

- Explain SMB, Samba, FTP, SSH, shares, anonymous access, and authentication.
- Enumerate SMB shares and understand the difference between discovering a share and being authorized to access it.
- Create, inspect, execute, and troubleshoot a Bash automation script.
- Understand the purpose and limits of Metasploit's SMB login scanner and Hydra.
- Interpret service discovery results and use evidence to choose the next step.
- Compare alternative tools and manual verification methods.
- Record findings accurately and explain the defensive controls that reduce the risk.

---

# Exercise 1 — Identify an anonymously accessible Samba share

## Objective

Identify the SMB/Samba shares exposed by the target, determine whether any share permits guest or anonymous access, and retrieve the challenge file from the accessible share.

**Challenge prompt:** “There is a samba share that allows anonymous access. Wonder what’s in there!”

## Step 1 — Enumerate SMB information with enum4linux

SMB (Server Message Block) is a network protocol commonly used for file and printer sharing and other Windows-compatible network services. Samba is a software implementation that allows Unix/Linux systems to provide and use SMB services.

The `enum4linux` utility queries SMB-related information. Its broad enumeration mode can attempt to discover workgroup details, users, shares, policies, and other information, depending on what the server permits.

Run this against the assigned lab target:

```bash
enum4linux -a target.ine.local
```

**Command breakdown**

- `enum4linux`: starts the SMB enumeration utility.
- `-a`: requests a broad set of enumeration checks supported by the tool.
- `target.ine.local`: the lab hostname. The hostname must resolve to the target's lab IP address.

**What to look for**

Review the output for share names, access or permission indications, discovered usernames, domain/workgroup details, and errors. A share being listed does not prove that it can be opened. Share discovery and share authorization are separate checks.

![SMB enumeration output showing discovered share information](https://miro.medium.com/v2/resize:fit:875/1*X3VHlceygYcjuCen36Kxbg.png)

SMB enumeration output showing discovered share information

### Technical explanation — interpreting the enumeration result

The output indicates that two shares were discovered, but the checks shown did not establish anonymous access to either one. This does not mean the server has no anonymously accessible share: enumeration tools may be restricted by server configuration, protocol behavior, or the particular checks they perform. The next step is to test candidate share names with an SMB client.

Useful terms:

- **Share:** a named resource exported by an SMB server, such as a directory.
- **Guest/anonymous access:** access that does not require a normal named user account, or uses a guest identity. The exact behavior depends on server configuration.
- **Share permissions:** controls whether a user can connect to a particular share.
- **Filesystem permissions:** controls applied by the underlying operating system to files and directories. A successful share connection does not guarantee read or write access to every file.

## Step 2 — Test candidate share names with a Bash script

The supplied walkthrough uses a wordlist at `/root/Desktop/wordlists/shares.txt` and a Bash script to test each candidate share name. This is a useful automation exercise: rather than manually entering the same SMB command many times, the script reads one candidate per line and checks whether an anonymous connection can list the share.

### Prepare and inspect the wordlist

```bash
ls -l /root/Desktop/wordlists/shares.txt
head -n 10 /root/Desktop/wordlists/shares.txt
```

- `ls -l` confirms that the file exists and shows its permissions and size.
- `head -n 10` displays the first ten candidate names so you can check that the file contains one share name per line.
- If the file is missing, verify the lab image and wordlist location rather than silently substituting an unrelated list.

### Improved script: [shares.sh](http://shares.sh)

The pasted script's intent is correct, but its formatting has collapsed several Bash statements together. As written, it is likely to produce syntax errors. The version below separates the statements, quotes variables safely, handles blank lines and Windows-style carriage returns, checks prerequisites, applies a per-share timeout, and distinguishes a successful listing from a failed attempt.

Create the file:

```bash
nano shares.sh
```

Paste this script:

```bash
#!/usr/bin/env bash

set -u

TARGET="target.ine.local"
WORDLIST="/root/Desktop/wordlists/shares.txt"
TIMEOUT_SECONDS=8

# Confirm that required programs and input files are available.
for program in smbclient timeout; do
    if ! command -v "$program" >/dev/null 2>&1; then
        printf '[!] Required command not found: %s\n' "$program" >&2
        exit 1
    fi
done

if [[ ! -r "$WORDLIST" ]]; then
    printf '[!] Wordlist is missing or unreadable: %s\n' "$WORDLIST" >&2
    exit 1
fi

printf '[*] Testing anonymous SMB access on %s\n' "$TARGET"

tested=0
accessible=0

while IFS= read -r SHARE || [[ -n "$SHARE" ]]; do
    # Remove a trailing carriage return and ignore blank/comment lines.
    SHARE="${SHARE%$'\r'}"
    [[ -z "$SHARE" || "$SHARE" == \#* ]] && continue

    # Avoid passing path separators or whitespace as share names.
    if [[ "$SHARE" == */* || "$SHARE" == *[[:space:]]* ]]; then
        printf '[!] Skipping malformed candidate: %s\n' "$SHARE"
        continue
    fi

    tested=$((tested + 1))
    printf '[*] Testing share: %s\n' "$SHARE"

    if timeout "$TIMEOUT_SECONDS" smbclient "//$TARGET/$SHARE" -N -c 'ls' >/dev/null 2>&1; then
        printf '[+] Anonymous listing succeeded: %s\n' "$SHARE"
        accessible=$((accessible + 1))
    else
        printf '[-] Anonymous listing failed or timed out: %s\n' "$SHARE"
    fi
done < "$WORDLIST"

printf '\n[*] Candidates tested: %d\n' "$tested"
printf '[*] Shares with successful anonymous listing: %d\n' "$accessible"
```

Save in nano with **Ctrl+O**, press **Enter**, then exit with **Ctrl+X**.

### How the script works

- `#!/usr/bin/env bash` selects Bash using the environment's command path.
- `set -u` makes the script report use of an unset variable instead of silently treating it as an empty value.
- `TARGET` stores the lab hostname. Change it only if the assigned lab target changes.
- `WORDLIST` stores the full path to the candidate-share file.
- `command -v` checks whether the required programs are installed and available in the current `PATH`.
- `[[ ! -r "$WORDLIST" ]]` checks whether the wordlist exists and can be read.
- `while IFS= read -r SHARE` reads each line literally. This avoids trimming or interpreting backslashes in candidate names.
- `${SHARE%$'\r'}` removes a trailing carriage return that can appear when a file was created with Windows line endings.
- The blank-line and comment check skips empty lines and lines beginning with `#`.
- The validation check skips malformed candidates containing a slash or whitespace.
- `timeout 8` prevents one unresponsive candidate from holding up the entire run indefinitely.
- `smbclient "//$TARGET/$SHARE"` attempts to connect to that named share.
- `-N` tells smbclient not to prompt for a password; this tests a no-password/guest-style connection, subject to server configuration.
- `-c 'ls'` runs the SMB client's `ls` command and exits, instead of leaving an interactive prompt open.
- `>/dev/null 2>&1` suppresses normal output and error output because this script is only counting success/failure. During troubleshooting, remove this redirection so you can see the server's response.
- The final counters report how many candidates were tested and how many returned a successful directory listing.

**Important interpretation:** a successful command indicates that the server accepted the connection and allowed the requested listing. A failed command is not definitive proof that a share does not exist; it may be a wrong candidate, network issue, timeout, protocol mismatch, or permission denial. Confirm interesting results manually.

### Make the script executable and run it

```bash
chmod +x shares.sh
./shares.sh
```

- `chmod` changes file permissions.
- `+x` adds execute permission.
- `./shares.sh` runs the script from the current directory. The `./` prefix tells the shell to execute the file in this directory.

You can also run it without changing the executable bit:

```bash
bash shares.sh
```

![Bash script execution and share testing](https://miro.medium.com/v2/resize:fit:875/1*le4S9PIssuiZ_FZwTqQaTA.png)

Bash script execution and share testing

![Results identifying the share that permits anonymous access](https://miro.medium.com/v2/resize:fit:875/1*Bacb3jEWTOrlYD_JThQQMw.png)

Results identifying the share that permits anonymous access

### Technical explanation — script output and the accessible share

The result identifies `pubfiles` as a share for which anonymous directory listing succeeds. This is the lead to validate manually. Do not treat a script's positive line as the end of the investigation: reconnect using the SMB client, list the contents, and record what is actually readable.

Common script issues:

| Symptom | Likely cause | Check |
| --- | --- | --- |
| `Permission denied` when running | Execute bit is not set | Run `chmod +x shares.sh` or use `bash shares.sh` |
| `bad interpreter` | Incorrect line endings or interpreter path | Run `file shares.sh`; if needed, use `dos2unix shares.sh` |
| `command not found: smbclient` | SMB client tools are not installed or not in PATH | Run `command -v smbclient` |
| Wordlist not found | Incorrect path or lab image differs | Run `ls -l /root/Desktop/wordlists/` |
| All candidates fail | Target resolution, network, protocol, or permissions issue | Check target resolution and test one candidate manually |
| A candidate times out | Service is slow or unreachable | Retry once manually and verify the lab service state |

## Step 3 — Connect to the share and inspect its files

Use the discovered share name:

```bash
smbclient //target.ine.local/pubfiles -N
```

**Command breakdown**

- `smbclient`: command-line SMB client.
- `//target.ine.local/pubfiles`: SMB server and share path.
- `-N`: do not prompt for a password.

If the connection succeeds, the prompt changes to an SMB client prompt. Common interactive commands include:

- `ls` — list files and directories in the current remote directory.
- `cd directory_name` — change the remote directory.
- `pwd` — show the current remote directory.
- `get filename` — download one remote file to the local working directory.
- `help` — show supported SMB client commands.
- `quit` — close the SMB session.

![Interactive connection to the pubfiles SMB share](https://miro.medium.com/v2/resize:fit:875/1*8fJCidk-9WU8JEGq5At8SQ.png)

Interactive connection to the pubfiles SMB share

### Technical explanation — browsing an SMB share

The screenshot demonstrates an SMB session to `pubfiles`. Once connected, use `ls` to identify available files. Download only the challenge-relevant file required by the exercise. The `get` command transfers a file from the remote share to the local machine; it does not execute the file.

For example, inside the SMB prompt:

```
ls
get flag1.txt
quit
```

Back in the Linux shell, inspect the downloaded file:

```bash
pwd
ls -l
cat flag1.txt
```

- `pwd` identifies the local destination directory.
- `ls -l` confirms that the downloaded file exists and shows its size.
- `cat` prints the file contents to the terminal. Do not use it on unknown binary files; use `file filename` first when the file type is uncertain.

![Downloaded challenge file and Flag 1 evidence](https://miro.medium.com/v2/resize:fit:875/1*SNiAlZWXQ7kI8I9w3MDYWg.png)

Downloaded challenge file and Flag 1 evidence

**Flag 1**

```
ec16bc35f93d4e93a9b48ca747e331d4
```

### Alternative enumeration methods

Use these as alternatives or cross-checks in the same authorized lab:

**List shares with smbclient**

```bash
smbclient -L //target.ine.local -N
```

- `-L` asks the server for a list of available shares.
- `-N` avoids a password prompt.
- This is a quick check, but the server may restrict anonymous share listing even when a known share can be accessed.

**Use Nmap's SMB scripts**

```bash
nmap -p 139,445 --script smb-enum-shares target.ine.local
```

This runs the SMB share-enumeration script against the common NetBIOS/SMB ports. Nmap's script results are useful evidence, but access should still be validated with an SMB client.

**Use smbmap if installed**

```bash
smbmap -H target.ine.local
```

Smbmap summarizes accessible shares and permissions. Tool availability and behavior depend on the lab image and version.

**Professional assessment note:** corroborate important findings with more than one method when possible. Record the tool version, command, target, time, and relevant output. Avoid claiming write access unless you have explicitly tested and are authorized to test it.

### Defensive perspective

Anonymous share access can expose internal documents, backups, scripts, configuration files, or credentials. Defenders should disable guest access where it is not required, apply least-privilege share and filesystem permissions, review share access logs, monitor unusual SMB connections, and periodically audit exported shares.

---

# Exercise 2 — Identify weak credentials and access a user-specific SMB share

## Objective

Use the usernames discovered during SMB enumeration to identify the deliberately weak account in the assigned lab, then authenticate to the share named after that user and retrieve the challenge file.

**Challenge prompt:** “One of the samba users have a bad password. Their private share with the same name as their username is at risk!”

## Step 1 — Review the discovered usernames

The SMB enumeration results include the following usernames:

- `josh`
- `bob`
- `nancy`
- `alice`

![Usernames discovered during SMB enumeration](https://miro.medium.com/v2/resize:fit:875/1*Yg1vUGaEE0peHxdsM-FyRw.png)

Usernames discovered during SMB enumeration

### Technical explanation — why usernames matter

A username is an account identifier; it is not proof that the account is valid for every service or that it has access to a particular share. Enumeration can reveal candidate accounts, but authentication and authorization must be evaluated separately.

The exercise describes a weak password. In a controlled lab, a small, supplied password list can demonstrate why weak or reused passwords are risky. In a production assessment, password testing must have explicit scope, rate limits, account-lockout safeguards, and approval. Never run password guessing against real accounts without that authorization.

## Step 2 — Prepare the username list

Create a text file containing one username per line:

```bash
nano users.txt
```

Enter:

```
josh
bob
nancy
alice
```

Save the file, then verify it:

```bash
cat users.txt
wc -l users.txt
```

- `cat` displays the list.
- `wc -l` counts the lines. Here, it should show four candidate usernames.

![Username list configured for the lab authentication check](https://miro.medium.com/v2/resize:fit:875/1*ToRUjJKeb7JubaBtpjrp6w.png)

Username list configured for the lab authentication check

## Step 3 — Configure the Metasploit SMB login scanner

Metasploit is a security testing framework that provides modules for service discovery, validation, and controlled security testing. The `scanner/smb/smb_login` module checks SMB authentication using the username and password inputs configured for it.

Start Metasploit:

```bash
msfconsole
```

Select the module:

```
use auxiliary/scanner/smb/smb_login
```

Review its options before setting values:

```
show options
```

Configure the lab target and input files. The common module options are:

```
set RHOSTS target.ine.local
set USER_FILE /path/to/users.txt
set PASS_FILE /root/Desktop/wordlists/unix_passwords.txt
```

Replace `/path/to/users.txt` with the actual full path of the file you created. If you created it in your home directory, for example, use the path shown by `pwd`.

Then review the options again:

```
show options
```

Only after confirming the target and both file paths are correct, run the module:

```
run
```

![Metasploit SMB login scanner password-list configuration](https://miro.medium.com/v2/resize:fit:875/1*f43z0KlAnGG9yuunYAfWMA.png)

Metasploit SMB login scanner password-list configuration

![Metasploit SMB authentication test results](https://miro.medium.com/v2/resize:fit:875/1*dlmMxn2L0mEVqFcI6JVavQ.png)

Metasploit SMB authentication test results

### Technical explanation — Metasploit options and result interpretation

- `use` loads a module into the Metasploit console.
- `auxiliary` identifies a supporting module rather than a payload-based exploit.
- `scanner/smb/smb_login` is the SMB login-checking module.
- `RHOSTS` is the remote host or hosts in scope. For this exercise it must refer only to the assigned lab target.
- `USER_FILE` points to the file containing candidate usernames.
- `PASS_FILE` points to the password list used by the module.
- `show options` displays required and optional settings and helps catch incorrect paths or missing values.
- `run` executes the module with the configured options.

A successful authentication result means the server accepted a username/password combination for the tested SMB service. It does not automatically prove that the account can access every share. The next step is to test the share named after the identified username.

**Operational caution:** password-list checks can generate many authentication attempts. In real environments they can trigger account lockouts, alerts, or service disruption. This lab intentionally provides a training context; do not increase concurrency or expand the target scope.

## Step 4 — Authenticate to the user-named share

The exercise indicates that the private share has the same name as the account. The walkthrough identifies `josh` as the account to validate.

Use:

```bash
smbclient //target.ine.local/josh -U josh
```

- `//target.ine.local/josh` specifies the SMB server and share.
- `-U josh` tells smbclient to authenticate as user `josh`.
- The client prompts for the password. Enter the lab credential discovered in the exercise; do not put passwords directly in shell history or shared notes.

![Authenticated SMB session to the user-named share](https://miro.medium.com/v2/resize:fit:875/1*Lmr59u6LKixjhfyBDv2uxQ.png)

Authenticated SMB session to the user-named share

### Technical explanation — authentication versus authorization

Authentication answers “Who are you?” Authorization answers “What are you allowed to access?” A valid login proves the server accepted the credentials. The share connection and the ability to list or read files provide further evidence about the account's permissions.

Inside the SMB prompt, list the share and retrieve the challenge file:

```
ls
get flag2.txt
quit
```

Then inspect it locally:

```bash
ls -l flag2.txt
cat flag2.txt
```

![Downloaded challenge file and Flag 2 evidence](https://miro.medium.com/v2/resize:fit:875/1*NjFLahCJ3OH2eKaqYWJL3A.png)

Downloaded challenge file and Flag 2 evidence

**Flag 2**

```
2ea59c892e6343b78273aba7ce4140ec
```

### Alternative validation and assessment approaches

- **Manual SMB authentication:** use `smbclient` with the explicitly identified lab account to verify a single known credential.
- **Nmap SMB scripts:** use SMB-focused scripts to gather share and security-mode information; these do not replace a successful authenticated access test.
- **Smbmap:** where installed and permitted, use it to summarize share permissions for a supplied account. Avoid exposing passwords in command-line arguments.
- **Defensive password auditing:** in a real organization, prefer approved identity-provider reports, password policy reviews, breached-password screening, and controlled audit procedures over ad hoc online guessing.

### Defensive perspective

Weak passwords can allow an attacker to turn a discovered username into authenticated access. Defenses include long unique passwords, multifactor authentication where supported, account lockout or throttling designed to resist abuse without enabling denial of service, removal of unused accounts, SMB signing and encryption where appropriate, least-privilege share permissions, and monitoring for repeated failed logins followed by a success.

---

# Exercise 3 — Enumerate the FTP service referenced by the previous finding

## Objective

Use the service clue from the previous exercise to identify the FTP listener, verify the service, and validate the supplied lab accounts against the FTP service.

**Challenge prompt:** “Follow the hint given in the previous flag to uncover this one.”

## Step 1 — Identify the FTP service and port

FTP (File Transfer Protocol) is a protocol used to transfer files between a client and a server. Traditional FTP separates control and data connections and does not encrypt credentials or file contents by default. Secure alternatives include SFTP (which operates over SSH) and FTPS (FTP protected with TLS).

The previous challenge file indicates that FTP is running. Use Nmap to inspect the target:

```bash
nmap -sV -p- target.ine.local
```

- `nmap` starts the network scanner.
- `-sV` attempts to identify service names and versions on discovered open ports.
- `-p-` scans all TCP ports from 1 through 65535.
- `target.ine.local` is the assigned lab hostname.

A full TCP port scan can take longer than checking common ports. If you already have a specific port clue, a narrower scan is more efficient:

```bash
nmap -sV -p 5554 target.ine.local
```

The results in this exercise identify FTP on TCP port `5554`.

![Nmap service discovery showing the FTP listener on port 5554](https://miro.medium.com/v2/resize:fit:875/1*mc9nrmCpa3A4-kfNMwQCEA.png)

Nmap service discovery showing the FTP listener on port 5554

### Technical explanation — interpreting a non-standard port

FTP commonly uses TCP port 21 for its control connection, but services can be configured to listen on other ports. Port 5554 is therefore not automatically a different protocol. The service-detection result is more informative than assuming a protocol from the port number alone.

A port scan tells you that a service appears reachable. It does not prove that a particular account can log in or that files are readable. Verify the service with a client and record the server response.

## Step 2 — Connect to the FTP service

Use the FTP client with the identified port:

```bash
ftp target.ine.local 5554
```

The client attempts to connect to the FTP control service on port 5554. If prompted, enter a username and password only when you have a valid lab credential. The server may display a greeting or warning before authentication.

![FTP service connection and server response](https://miro.medium.com/v2/resize:fit:875/1*HZcmMwaKdvM28Ssm9XmiJw.png)

FTP service connection and server response

### Technical explanation — FTP session basics

FTP clients commonly provide these commands:

- `help` — list supported commands.
- `pwd` — show the current remote directory.
- `ls` — list remote files.
- `cd directory` — change the remote directory.
- `get filename` — download one file.
- `binary` — select binary transfer mode, useful for non-text files.
- `quit` or `bye` — close the session.

FTP is not encrypted by default. On a real network, credentials and transferred data may be observable to parties able to capture the traffic. Prefer SFTP or FTPS for secure file transfer and disable legacy FTP where it is not required.

## Step 3 — Validate the supplied lab accounts

The walkthrough identifies three candidate accounts:

- `ashley`
- `alice`
- `amanda`

Create or inspect the username file:

```bash
nano users.txt
```

Contents:

```
ashley
alice
amanda
```

Verify the file:

```bash
cat users.txt
```

![Candidate FTP usernames prepared for the lab](https://miro.medium.com/v2/resize:fit:875/1*0HIE113ehtzy1zb97U8Mog.png)

Candidate FTP usernames prepared for the lab

The lab walkthrough uses Hydra, a network login testing tool, to validate the supplied candidate accounts against the FTP service with the provided training wordlist.

For this intentionally vulnerable lab only, the command pattern is:

```bash
hydra -L users.txt -P /root/Desktop/wordlists/unix_passwords.txt -s 5554 -t 1 -f ftp://target.ine.local
```

**Option breakdown**

- `hydra`: starts the login-testing tool.
- `-L users.txt`: reads multiple candidate usernames from the specified file. Uppercase `-L` means a username list.
- `-P /root/Desktop/wordlists/unix_passwords.txt`: reads candidate passwords from the specified file. Uppercase `-P` means a password list.
- `-s 5554`: selects the FTP service's non-default TCP port.
- `-t 1`: uses one task at a time to keep the lab test conservative and easier to interpret.
- `-f`: stops after the first valid login is found.
- `ftp://target.ine.local`: selects the FTP protocol module and lab target.

The supplied walkthrough's shorter form is `hydra -L users.txt -P /root/Desktop/wordlists/unix_passwords.txt ftp://target.ine.local:5554`. The explicit `-s 5554` form makes the port setting clear; do not combine both port specifications.

![FTP authentication result and access to the service](https://miro.medium.com/v2/resize:fit:875/1*VQEYqTDvDjCP8W4GOxmyng.png)

FTP authentication result and access to the service

### Technical explanation — reading Hydra output

Hydra tests combinations from the supplied username and password lists against the specified FTP service. A reported valid pair means the service accepted that authentication attempt. It is not proof that the account has access to every remote directory or file.

If the tool reports no valid login, possible explanations include an incorrect target or port, a wrong protocol, a username/password list that does not match the lab, account throttling, or a service that is temporarily unavailable. Confirm connectivity and the exact lab parameters before drawing conclusions.

**Authorization reminder:** do not run Hydra against third-party systems or real accounts without explicit written authorization and an agreed test plan. In production, coordinate account-lockout thresholds, test windows, source IPs, rate limits, and monitoring with the system owner.

## Step 4 — Retrieve and verify the challenge file

After the lab credential is identified, connect using the FTP client:

```bash
ftp target.ine.local 5554
```

Enter the assigned lab username and its validated password at the prompts. Then inspect the remote directory:

```
pwd
ls
```

Download the relevant challenge file:

```
get flag3.txt
bye
```

Back in the local shell:

```bash
ls -l flag3.txt
file flag3.txt
cat flag3.txt
```

The local checks confirm that the file was downloaded and help identify its type before displaying it.

**Flag 3**

```
dc6ea4029bfd42548c2bf19be77d8498
```

### Alternative FTP enumeration and validation methods

**Nmap service and script checks**

```bash
nmap -sV -p 5554 target.ine.local
```

Use service detection to confirm the listener. FTP-specific Nmap scripts can provide additional service information where permitted; review each script's behavior before running it.

**Manual FTP client**

The standard `ftp` client is useful for verifying a single known lab login and manually inspecting directory permissions. It is often the clearest way to understand the protocol interaction.

**Netcat for a basic banner check**

```bash
nc -nv target.ine.local 5554
```

This can display an initial service greeting if the server sends one. It is a basic TCP connectivity check, not a full FTP client and not an authentication test.

**Defensive perspective**

Defenders should disable FTP where possible, replace it with SFTP or FTPS, enforce strong unique credentials, restrict access by network and account, monitor repeated authentication failures, and review file-transfer logs. FTP credentials should be treated as exposed if they were transmitted over an untrusted network without encryption.

---

# Exercise 4 — Interpret the SSH login warning

## Objective

Identify the SSH service and understand the purpose of a warning displayed to users before or during login.

**Challenge prompt:** “This is a warning meant to deter unauthorized users from logging in.”

## Step 1 — Identify SSH in the service scan

SSH (Secure Shell) is an encrypted protocol used for remote administration and secure communication. It commonly listens on TCP port 22, although administrators can configure another port.

The earlier service scan identifies SSH as another service on the lab target. Review the Nmap output and note the port and service identification.

![SSH service and login warning evidence](https://miro.medium.com/v2/resize:fit:875/1*x--IUAFeoVDdUMQZJZbkAQ.png)

SSH service and login warning evidence

### Technical explanation — login banners and legal notices

An SSH server can display a warning or notice to users. There are two commonly encountered mechanisms:

- **Pre-authentication banner:** text configured by the SSH server and displayed before authentication, commonly through the `Banner` setting in `sshd_config`.
- **Post-authentication message:** a message of the day (MOTD) or shell profile message displayed after successful login, depending on the server configuration.

A legal warning typically states that access is restricted to authorized users and that activity may be monitored. Its purpose is to communicate acceptable-use conditions and support organizational policy. A banner does not itself provide technical access control; authentication, authorization, logging, and network controls must enforce security.

## Step 2 — Understand the SSH connection syntax

For a standard SSH connection, the syntax is:

```bash
ssh username@target.ine.local
```

If the service is configured on a non-standard port, specify it with uppercase `-p`:

```bash
ssh -p PORT username@target.ine.local
```

- `ssh` starts the Secure Shell client.
- `username@` specifies the account to authenticate as.
- `target.ine.local` identifies the assigned lab host.
- `-p PORT` selects a non-default TCP port. Replace `PORT` with the port confirmed by the lab scan.

Do not guess credentials or attempt to bypass the warning. The purpose of this exercise is to recognize and interpret the notice. Follow the lab's supplied access instructions if authentication is required to view the intended message.

### Technical explanation — what the warning proves and what it does not

The warning is evidence that the service is configured to present an access notice. It does not prove that unauthorized access is technically prevented, that activity is actually monitored, or that the host is secure. Those claims require separate verification of authentication controls, authorization, logging, and monitoring.

For a defender, the SSH configuration should restrict permitted users and authentication methods, disable insecure options, use key-based authentication where appropriate, keep the server patched, and forward authentication logs to a monitored logging platform.

**Flag 4**

```
4a03de945ffb4bc0947c3dfe5d4e507b
```

---

# Consolidated command reference

| Tool or command | Purpose in this lab |
| --- | --- |
| `enum4linux -a HOST` | Broad SMB information enumeration |
| `smbclient -L //HOST -N` | Request a list of SMB shares without a password prompt |
| `smbclient //HOST/SHARE -N` | Attempt a no-password connection to a named share |
| `smbclient //HOST/SHARE -U USER` | Connect to a share as a named user |
| `ls`, `get`, `quit` | List and download files in an SMB client session |
| `chmod +x script.sh` | Add execute permission to a script |
| `bash script.sh` | Run a Bash script using the Bash interpreter |
| `nmap -sV -p- HOST` | Scan TCP ports and attempt service/version identification |
| `ftp HOST PORT` | Connect to an FTP service |
| `hydra -L USERS -P PASSWORDS -s PORT ftp://HOST` | Controlled credential validation in the assigned lab |
| `ssh USER@HOST` | Start an SSH connection using the default port |
| `ssh -p PORT USER@HOST` | Start an SSH connection on a specified port |
| `cat FILE`, `file FILE`, `ls -l FILE` | Inspect downloaded challenge files |

# Troubleshooting checklist

1. **Hostname does not resolve:** check the lab VPN/network connection and confirm that `target.ine.local` resolves in the lab environment.
2. **Connection refused:** verify the target is running, the port is correct, and the relevant service is listening.
3. **Connection times out:** check VPN connectivity, routing, firewall rules, and whether the lab machine is still active.
4. **SMB share is listed but cannot be opened:** listing and share access are different permissions; validate the exact share name and access method.
5. **Anonymous SMB test fails:** confirm that `-N` is being used as intended, then inspect the visible error output instead of relying only on a suppressed script result.
6. **Bash script fails:** check the shebang, line endings, file path, executable permission, and whether `smbclient` is installed.
7. **FTP login fails:** confirm the target port, protocol, username file, and lab-provided wordlist path. Avoid increasing test rates to work around a failure.
8. **SSH displays a warning:** read and record the notice. A warning is not a substitute for authentication or authorization controls.
9. **Downloaded file is missing:** check the remote working directory before `get`, then use local `pwd` and `ls -l` after the transfer.

# Professional assessment notes

For every finding, record:

- Target hostname and resolved lab IP address.
- Date/time and lab scope.
- Tool name and version, where available.
- Exact command or relevant configuration.
- The important output and screenshot evidence.
- Whether the result was observed, inferred, or independently validated.
- Security impact and recommended remediation.

Do not include reusable real-world passwords, private keys, or unrelated sensitive data in the report. Keep challenge credentials and flags within the authorized training record.

# Key concepts glossary

- **SMB:** Server Message Block, a protocol for network file and resource sharing.
- **Samba:** software that implements SMB services on Unix/Linux systems.
- **Share:** a named SMB resource made available by a server.
- **Anonymous/guest access:** access that does not require an ordinary named-user password, depending on server configuration.
- **Authentication:** verification of an account's identity.
- **Authorization:** determination of what an authenticated identity is allowed to do.
- **Wordlist:** a text file containing candidate values, such as usernames, share names, or passwords.
- **Enumeration:** structured querying of a service to discover detailed information and accessible resources.
- **FTP:** File Transfer Protocol, a legacy file-transfer protocol that is unencrypted by default.
- **SSH:** Secure Shell, an encrypted protocol for remote administration.
- **Banner:** text presented by a service, often to identify the service or communicate a legal notice.
- **Least privilege:** granting only the permissions needed to perform an approved task.
- **Evidence validation:** confirming a finding through observable output or an independent method.

# Module review and learning outcomes

By completing these exercises, you have practiced moving from service enumeration to permission validation, controlled authentication checks, file retrieval, and interpretation of service notices. The important assessment habit is to separate discovery from proof: a listed share is not necessarily accessible, a valid login does not imply unrestricted permissions, and a banner does not enforce security by itself.

For the defensive perspective, the recurring themes are reducing anonymous exposure, enforcing strong authentication, limiting account and share permissions, replacing insecure file-transfer protocols, monitoring authentication activity, and documenting findings so another analyst can reproduce and verify them.