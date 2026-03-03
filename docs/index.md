# 🎯 WEB200 Notes

Cheatsheets de préparation à la certification **OSWA (WEB-200)**.  
Toutes les commandes, payloads et méthodologies pour l'exam.

---

## Navigation rapide

<div class="grid cards" markdown>

- :material-magnify: **Outils**  
  Nmap, curl, gobuster, wfuzz, ffuf, sqlmap...  
  [:octicons-arrow-right-24: Voir les outils](outils/index.md)

- :material-code-braces: **Vulnérabilités**  
  SQLi, XSS, SSTI, XXE, SSRF, LFI, File Upload...  
  [:octicons-arrow-right-24: Voir les vulns](vulns/index.md)

- :material-console: **Post-Exploitation**  
  Reverse shells, SUID, capabilities, enum Linux...  
  [:octicons-arrow-right-24: Post-exploit](post-exploit/index.md)

- :material-map: **Méthodologie**  
  Workflows type, checklist exam, payloads universels...  
  [:octicons-arrow-right-24: Méthodologie](methodology/index.md)

</div>

---

## Payload universel de détection

```bash
# Teste SQLi, XSS, SSTI, EL injection, Command injection en une requête
'"><img src=x>{{7*7}}${7*7};sleep 5
```

## Checklist démarrage exam

```bash
# 1. VPN actif
sudo openvpn universal.ovpn

# 2. Hosts
sudo nano /etc/hosts   # ajouter IP cible

# 3. Scan initial
nmap -sV -O -Pn TARGET
nmap -A -sC -T4 TARGET

# 4. Enum web
gobuster dir -u http://TARGET -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt
echo "http://TARGET" | hakrawler -u

# 5. Listener prêt
nc -nlvp 9090
```

---

!!! tip "Rappel important"
    **ENUMERATION** = "Je **découvre** ce qui existe"  
    **EXPLOITATION** = "J'**abuse** de ce que j'ai trouvé"
