# <span style="color:orange;">1. Reconnaissance & Énumération</span>

### 1.1 NMAP

```bash
# Scan basique (1000 ports les plus communs)
nmap -Pn enum-sandbox
# -Pn : Skip host discovery (considère l'hôte comme actif)
#  Par défaut : scan les 1000 ports TCP les plus communs

-------------------------------------
# Scan TOUS les ports (65535) :
nmap -p- enum-sandbox
nmap -p- -T4 enum-sandbox  # Plus rapide
# -p- : Scan tous les ports de 1 à 65535
# -T4 : Timing template (0=paranoid → 5=insane), T4 = rapide

-------------------------------------
# Détection de version des services
nmap -sV enum-sandbox
# `-sV` : Service Version detection (détecte versions logicielles)

-------------------------------------
# Scan avec scripts par défaut
nmap -sC enum-sandbox
# `-sC` : Equivalent à `--script=default`


-------------------------------------
# Scan agressif (OS + version + scripts + traceroute)
nmap -A enum-sandbox


-------------------------------------
# Scan d'un port spécifique avec script
nmap -p 80 --script http-methods enum-sandbox
nmap -p 8080 --script http-methods enum-sandbox


-------------------------------------
# Scan des N ports les plus communs
nmap --top-ports 100 enum-sandbox
nmap --top-ports 1000 enum-sandbox

-------------------------------------
# Scanner des ports spécifiques
nmap -p 22,80,443,3389,8080 enum-sandbox
nmap -p 1-1000 enum-sandbox  # Range


-------------------------------------
# Scan UDP
nmap -sU -p 53,161,162 enum-sandbox
```


**Output formats :**

```bash
nmap -sV enum-sandbox -oN scan.txt       # Normal output
nmap -sV enum-sandbox -oX scan.xml       # XML output
nmap -sV enum-sandbox -oG scan.grep      # Grepable output
nmap -sV enum-sandbox -oA scan           # All formats (scan.nmap, scan.xml, scan.gnmap)
```

**Options utiles supplémentaires :**

```bash
# Désactiver résolution DNS (plus rapide)
nmap -n enum-sandbox

# Scan avec résolution DNS
nmap -R enum-sandbox

# Reason (pourquoi port open/closed)
nmap --reason enum-sandbox

# Verbose (voir progression)
nmap -v enum-sandbox
nmap -vv enum-sandbox  # Plus verbeux

# Résumé seulement
nmap --open enum-sandbox  # Montre que les ports ouverts
```

---


### 1.2 CURL

**Afficher headers + contenu :**

```bash
curl -i http://enum-sandbox
```


**Afficher SEULEMENT les headers (HEAD request) :**

```bash
curl -I http://enum-sandbox
```


**Suivre les redirections :**

```bash
curl -L http://enum-sandbox
```

**Ignorer erreurs SSL (certificats non valides) :**

```bash
curl -k https://cors-sandbox
curl -i -L -k https://cors-sandbox
```


**Spécifier méthode HTTP :**

```bash
curl -X GET http://enum-sandbox
curl -X POST http://enum-sandbox
curl -X PUT http://enum-sandbox
curl -X DELETE http://enum-sandbox
curl -X OPTIONS http://enum-sandbox
```


**Envoyer données POST :**

```bash
# Form data
curl -X POST -d "username=admin&password=test" http://enum-sandbox/login

# JSON data
curl -X POST -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"test"}' \
  http://enum-sandbox/api/login
```


**Headers personnalisés :**

```bash
curl -H "User-Agent: Mozilla/5.0" http://enum-sandbox
curl -H "Authorization: Bearer token123" http://enum-sandbox
curl -H "X-Custom-Header: value" http://enum-sandbox

# Plusieurs headers
curl -H "Header1: value1" -H "Header2: value2" http://enum-sandbox
```


**Cookies :**

```bash
# Envoyer cookie
curl -b "session=abc123" http://enum-sandbox

# Sauvegarder cookies dans fichier
curl -c cookies.txt http://enum-sandbox

# Utiliser cookies depuis fichier
curl -b cookies.txt http://enum-sandbox

# Combiné (load + save)
curl -b cookies.txt -c cookies.txt http://enum-sandbox
```


**User-Agent personnalisé :**

```bash
curl -A "Mozilla/5.0" http://enum-sandbox
curl --user-agent "MyBot/1.0" http://enum-sandbox
```

**Proxy :**

```bash
# HTTP proxy
curl --proxy http://127.0.0.1:8080 http://enum-sandbox

# SOCKS5 proxy
curl --socks5 127.0.0.1:9050 http://enum-sandbox

# Avec authentification
curl --proxy http://user:pass@proxy.com:8080 http://enum-sandbox
```

**Timeout :**

```bash
curl --connect-timeout 10 http://enum-sandbox
curl --max-time 30 http://enum-sandbox
```

**Output vers fichier :**

```bash
curl http://enum-sandbox -o output.html
curl http://enum-sandbox --output output.html

# Utiliser nom de fichier distant
curl -O http://enum-sandbox/file.pdf
```


**Silent / Verbose :**

```bash
# Silent (pas de progress bar)
curl -s http://enum-sandbox

# Verbose (debug complet)
curl -v http://enum-sandbox

# Très verbose
curl -vv http://enum-sandbox
```


**Authentification :**

```bash
# Basic auth
curl -u username:password http://enum-sandbox

# Bearer token
curl -H "Authorization: Bearer TOKEN" http://enum-sandbox
```

**Upload fichier :**

```bash
curl -F "file=@/path/to/file.txt" http://enum-sandbox/upload
curl -F "file=@file.txt;filename=custom.txt" http://enum-sandbox/upload
```

**Combinaison complète (headers + redirections + SSL) :**

```bash
curl -i -L -k https://cors-sandbox
```

---

### 1.3 NETCAT (nc)

**Banner grabbing SSH :**

```bash
netcat -v enum-sandbox 22
nc -v enum-sandbox 22
```

**Options :**

- `-v` : Mode verbose (affiche infos détaillées)
- Utile pour : identifier version SSH, OS

**Exemple retour :**

```
OpenSSH 8.9p1 Ubuntu 3ubuntu0.7 (Ubuntu Linux; protocol 2.0)
```

---

**Bind Shell - Se connecter :**

```bash
netcat enum-sandbox 9999
nc enum-sandbox 9999
```

**Contexte :**

- La cible écoute sur le port 9999
- Tu te connectes à elle pour obtenir un shell

---

**Reverse Shell - Écouter (attendre connexion) :**

```bash
netcat -vlp 9090
nc -vlp 9090
```

**Options :**

- `-v` : Mode verbose
- `-l` : Mode listen (écoute/serveur)
- `-p 9090` : Port sur lequel écouter

**Contexte :**

- Tu écoutes sur ton port 9090
- La cible se connecte à toi pour te donner un shell

**Workflow :**

1. Lance `nc -vlp 9090` sur Kali
2. Exécute reverse shell sur cible avec ton IP + port 9090
3. Reçois shell dans terminal Kali

---

**Exécuter commande sur connexion :**

```bash
# Windows
nc -vlp 9999 -e cmd.exe

# Linux
nc -vlp 9999 -e /bin/bash
```

**Options :**

- `-e PROGRAM` : Execute program on connect
- **Note** : Option souvent désactivée pour sécurité (utiliser ncat ou autres alternatives)

---

**Scan de ports (alternative nmap) :**

```bash
nc -zv enum-sandbox 1-1000
nc -zv -w 1 enum-sandbox 20-25
```

**Options :**

- `-z` : Zero I/O mode (scan ports sans envoyer data)
- `-w SECONDS` : Timeout (secondes)
- `-v` : Verbose

---

**UDP mode :**

```bash
nc -u enum-sandbox 53
nc -ulp 161  # Listen UDP
```

**Options :**

- `-u` : Use UDP instead of TCP

---

**Désactiver résolution DNS (plus rapide) :**

```bash
nc -n -v enum-sandbox 80
```

**Options :**

- `-n` : No DNS resolution

---

**Transfert de fichiers :**

```bash
# Récepteur (attente du fichier)
nc -vlp 9999 > received_file.txt

# Envoyeur
nc target 9999 < file_to_send.txt
```

---

**Chat simple entre deux machines :**

```bash
# Machine 1
nc -vlp 9999

# Machine 2
nc machine1_ip 9999
```

---

