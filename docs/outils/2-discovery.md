# Discovery & Content Discovery

## 1 Gobuster

```bash
# Directory bruteforce
gobuster dir -u http://TARGET -w /usr/share/wordlists/dirb/common.txt

# Avec extensions
gobuster dir -u http://TARGET -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-files-lowercase.txt -x php,html,txt

# Vhost enumeration
gobuster vhost -u http://TARGET -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain

# Options utiles
gobuster dir -u http://TARGET -w WORDLIST -t 50 -k -r
# -t : threads  -k : ignore SSL  -r : follow redirects
```

**Wordlists recommandées :**
```bash
/usr/share/wordlists/dirb/common.txt                          # rapide
/usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt  # moyen
/usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-files-lowercase.txt
/usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

---

## 2 Hakrawler

```bash
# Crawl basique
echo "http://TARGET" | hakrawler -u

# Via Burp proxy
echo "http://TARGET" | hakrawler -u -proxy http://127.0.0.1:8080

# Options
# -u : unique URLs  -d : profondeur  -s : scope regex
# -insecure : ignore SSL  -t : threads  -dr : disable robots.txt
```

---

## 3 Dirb

```bash
# Scan basique
dirb http://TARGET

# Avec extensions
dirb http://TARGET/automated/ -X .php

# Ignorer code
dirb http://TARGET -N 404,403
```

---

## 4 Curl - Inspection rapide

```bash
# Headers seulement
curl -I http://TARGET

# Headers + body + redirects + ignore SSL
curl -i -L -k https://TARGET

# Avec cookie
curl -b "session=abc123" http://TARGET

# Via Burp proxy
curl --proxy http://127.0.0.1:8080 http://TARGET
```
