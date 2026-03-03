# Checklist Exam

## Avant de commencer

- [ ] VPN connecté (`sudo openvpn universal.ovpn`)
- [ ] `/etc/hosts` mis à jour
- [ ] Listener prêt (`nc -nlvp 9090`)
- [ ] Serveur HTTP Python (`python3 -m http.server 80`)
- [ ] Burp Suite ouvert + scope configuré

## Recon initial (5 min)

- [ ] `nmap -sV -Pn TARGET`
- [ ] `curl -I http://TARGET`
- [ ] `gobuster dir -u http://TARGET -w raft-medium-directories.txt`
- [ ] `echo "http://TARGET" | hakrawler -u`

## Identification vulnérabilités

- [ ] Payload universel sur tous les inputs : `'"><img src=x>{{7*7}}${7*7};sleep 5`
- [ ] SQLi sur tous les params GET/POST
- [ ] XSS sur tous les inputs texte
- [ ] Observer les erreurs (stack traces)

## Flags

- [ ] `local.txt` (auth bypass)
- [ ] `proof.txt` (RCE → shell)

## Post-exploit shell

```bash
whoami && id && hostname
find / -name proof.txt 2>/dev/null
find / -name local.txt 2>/dev/null
python3 -c 'import pty;pty.spawn("/bin/bash")'
```
