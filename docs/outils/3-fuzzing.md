# Fuzzing

## 1 WFUZZ

**Fuzzing paramètre GET :**

```bash
wfuzz -c -z file,/usr/share/wordlists/wfuzz/Injections/SQL.txt \
  -d "db=mysql&name=test&sort=id&order=FUZZ" \
  -u http://sql-sandbox/discovery/api/fuzzing
```

**Options :**

- `-c` : Colored output (sortie en couleur)
- `-z file,<WORDLIST>` : Spécifie le payload (fichier wordlist)
- `-d "data"` : Data pour requête POST (body)
- `-u <URL>` : URL cible
- `FUZZ` : Keyword magique = position où injecter payloads

---

**Fuzzing paramètre dans URL :**

```bash
wfuzz -c -z file,/usr/share/wordlists/wfuzz/Injections/SQL.txt \
  -u "http://sql-sandbox/extramile/index.php?id=FUZZ"
```

---

**FILTRES (Hide) - Options CRITIQUES :**

```bash
# Cacher par code HTTP
wfuzz -c -w wordlist.txt --hc 404 -u http://target/FUZZ

# Cacher par taille (chars)
wfuzz -c -w wordlist.txt --hh 1234 -u http://target/FUZZ

# Cacher par nombre de lignes
wfuzz -c -w wordlist.txt --hl 10 -u http://target/FUZZ

# Cacher par nombre de mots
wfuzz -c -w wordlist.txt --hw 50 -u http://target/FUZZ

# Combiner filtres
wfuzz -c -w wordlist.txt --hc 404,403 --hh 0 -u http://target/FUZZ
```

**Options filtres :**

- `--hc CODE` : Hide responses with HTTP code(s)
- `--hh CHARS` : Hide responses with CHARS characters
- `--hl LINES` : Hide responses with LINES lines
- `--hw WORDS` : Hide responses with WORDS words
- `--hr REGEX` : Hide responses matching regex

---

**MATCH (Show) - Inverse des filtres :**

```bash
# Montrer SEULEMENT code 200
wfuzz -c -w wordlist.txt --sc 200 -u http://target/FUZZ

# Montrer SEULEMENT taille 1234
wfuzz -c -w wordlist.txt --sh 1234 -u http://target/FUZZ

# Montrer SEULEMENT 10 lignes
wfuzz -c -w wordlist.txt --sl 10 -u http://target/FUZZ

# Montrer SEULEMENT 50 mots
wfuzz -c -w wordlist.txt --sw 50 -u http://target/FUZZ
```

**Options match :**

- `--sc CODE` : Show responses with HTTP code(s)
- `--sh CHARS` : Show responses with CHARS characters
- `--sl LINES` : Show responses with LINES lines
- `--sw WORDS` : Show responses with WORDS words
- `--sr REGEX` : Show responses matching regex

---

**Threads (parallélisation) :**

```bash
wfuzz -c -w wordlist.txt -t 50 -u http://target/FUZZ
```

**Options :**

- `-t THREADS` : Concurrent connections (défaut: 10)

---

**Cookies :**

```bash
wfuzz -c -w wordlist.txt -b "session=abc123" -u http://target/FUZZ
```

**Options :**

- `-b COOKIE` : Cookie string

---

**Headers personnalisés :**

```bash
wfuzz -c -w wordlist.txt \
  -H "User-Agent: Mozilla" \
  -H "Authorization: Bearer token" \
  -u http://target/FUZZ
```

**Options :**

- `-H "Header: Value"` : Add custom header

---

**Suivre redirections :**

```bash
wfuzz -c -w wordlist.txt --follow -u http://target/FUZZ
```

**Options :**

- `--follow` : Follow HTTP redirections

---

**Proxy :**

```bash
wfuzz -c -w wordlist.txt -p 127.0.0.1:8080 -u http://target/FUZZ

# SOCKS proxy
wfuzz -c -w wordlist.txt -p 127.0.0.1:9050:SOCKS5 -u http://target/FUZZ
```

**Options :**

- `-p PROXY:PORT[:type]` : Use proxy (HTTP/SOCKS4/SOCKS5)

---

**Méthode HTTP :**

```bash
wfuzz -c -w wordlist.txt -X POST -u http://target/FUZZ
```

**Options :**

- `-X METHOD` : HTTP method (GET, POST, PUT, DELETE, etc.)

---

**Encoder payloads :**

```bash
wfuzz -c -w wordlist.txt -e base64 -u http://target/FUZZ
```

**Options :**

- `-e ENCODER` : Encode payload (base64, url, etc.)

---

**Plusieurs FUZZ positions :**

```bash
wfuzz -c -w users.txt -w passwords.txt \
  -d "username=FUZZ&password=FUZ2Z" \
  -u http://target/login
```

**Note :**

- `FUZZ` = première wordlist
- `FUZ2Z` = deuxième wordlist
- `FUZ3Z` = troisième wordlist, etc.

---

**Analyser les résultats :**

- Regarder : Code réponse (200, 500, etc.), Taille (Chars), Lignes (Lines), Mots (Words)
- **200 + 2 Ch** = Probablement réponse vide
- **200 + >200 Ch** = Payload intéressant
- **500 + contenu** = Erreur potentiellement exploitable

**Exemple retour :**

```
000000046:   500        0 L      60 W       458 Ch      "hi" or 1=1 --"
000000025:   200        0 L      62 W       495 Ch      "0 or 1=1"
```

---

## 2 FFUF

**Fuzzing formulaire POST (username) :**

```bash
ffuf -w users.txt \
  -u http://enum-sandbox/auth/login \
  -X POST \
  -d 'username=FUZZ&password=b' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -o results.json
```

**Options de base :**

- `-w WORDLIST` : Wordlist à utiliser
- `-u URL` : URL cible
- `-X METHOD` : Méthode HTTP (GET, POST, PUT, DELETE, etc.)
- `-d DATA` : Corps de la requête POST
- `-H "Header"` : Header HTTP personnalisé
- `FUZZ` : Position où injecter payloads

---

**FILTRES (Filter) - Options CRITIQUES :**

```bash
# Filtrer par code HTTP
ffuf -w wordlist.txt -fc 404 -u http://target/FUZZ

# Filtrer par taille (size/chars)
ffuf -w wordlist.txt -fs 1234 -u http://target/FUZZ

# Filtrer par nombre de lignes
ffuf -w wordlist.txt -fl 10 -u http://target/FUZZ

# Filtrer par nombre de mots
ffuf -w wordlist.txt -fw 50 -u http://target/FUZZ

# Filtrer par regex
ffuf -w wordlist.txt -fr "error|not found" -u http://target/FUZZ

# Combiner filtres
ffuf -w wordlist.txt -fc 404,403 -fs 0,1234 -u http://target/FUZZ
```

**Options filtres :**

- `-fc CODES` : Filter HTTP status codes (ex: 404,403)
- `-fs SIZE` : Filter response size (ex: 1234 ou 0,1234,5678)
- `-fl LINES` : Filter by amount of lines
- `-fw WORDS` : Filter by amount of words
- `-fr REGEX` : Filter by regex pattern

---

**MATCH (Match) - Inverse des filtres :**

```bash
# Montrer SEULEMENT code 200
ffuf -w wordlist.txt -mc 200 -u http://target/FUZZ

# Montrer SEULEMENT taille 1234
ffuf -w wordlist.txt -ms 1234 -u http://target/FUZZ

# Montrer SEULEMENT 10 lignes
ffuf -w wordlist.txt -ml 10 -u http://target/FUZZ

# Montrer SEULEMENT 50 mots
ffuf -w wordlist.txt -mw 50 -u http://target/FUZZ

# Match par regex
ffuf -w wordlist.txt -mr "success|admin" -u http://target/FUZZ
```

**Options match :**

- `-mc CODES` : Match HTTP status codes
- `-ms SIZE` : Match response size
- `-ml LINES` : Match amount of lines
- `-mw WORDS` : Match amount of words
- `-mr REGEX` : Match regex pattern

---

**Threads (parallélisation) :**

```bash
ffuf -w wordlist.txt -t 100 -u http://target/FUZZ
```

**Options :**

- `-t THREADS` : Number of concurrent threads (défaut: 40)

---

**Rate limiting :**

```bash
# Limiter à 10 requêtes par seconde
ffuf -w wordlist.txt -rate 10 -u http://target/FUZZ

# Pause entre chaque requête
ffuf -w wordlist.txt -p 0.5 -u http://target/FUZZ

# Delay range aléatoire
ffuf -w wordlist.txt -p 0.5-1.0 -u http://target/FUZZ
```

**Options :**

- `-rate` : Rate limit (requests per second)
- `-p SECONDS` : Delay between requests (peut être un range: 0.1-2.0)

---

**Timeout :**

```bash
ffuf -w wordlist.txt -timeout 30 -u http://target/FUZZ
```

**Options :**

- `-timeout SECONDS` : HTTP request timeout (défaut: 10)

---

**Extensions :**

```bash
# Tester avec extensions
ffuf -w wordlist.txt -e .php,.html,.txt,.bak -u http://target/FUZZ

# Sans extension originale (juste les extensions)
ffuf -w wordlist.txt -e .php,.html -u http://target/FUZZ -sf
```

**Options :**

- `-e EXTENSIONS` : Comma separated list of extensions
- `-sf` : Stop on spurious errors

---

**Recursion :**

```bash
# Fuzzing récursif
ffuf -w wordlist.txt -recursion -u http://target/FUZZ

# Avec profondeur max
ffuf -w wordlist.txt -recursion -recursion-depth 3 -u http://target/FUZZ
```

**Options :**

- `-recursion` : Scan recursively
- `-recursion-depth N` : Maximum recursion depth (défaut: 0 = illimité)

---

**Verbose / Silent :**

```bash
# Verbose (debug)
ffuf -w wordlist.txt -v -u http://target/FUZZ

# Silent (pas d'output sauf résultats)
ffuf -w wordlist.txt -s -u http://target/FUZZ
```

**Options :**

- `-v` : Verbose output
- `-s` : Silent mode

---

**Colorize :**

```bash
ffuf -w wordlist.txt -c -u http://target/FUZZ
```

**Options :**

- `-c` : Colorize output

---

**Output formats :**

```bash
# JSON
ffuf -w wordlist.txt -o results.json -u http://target/FUZZ

# JSON avec format spécifique
ffuf -w wordlist.txt -o results.json -of json -u http://target/FUZZ

# CSV
ffuf -w wordlist.txt -o results.csv -of csv -u http://target/FUZZ

# HTML
ffuf -w wordlist.txt -o results.html -of html -u http://target/FUZZ

# Markdown
ffuf -w wordlist.txt -o results.md -of md -u http://target/FUZZ

# eJSONlines
ffuf -w wordlist.txt -o results.ejson -of ejson -u http://target/FUZZ

# All formats
ffuf -w wordlist.txt -o results -of all -u http://target/FUZZ
```

**Options :**

- `-o FILENAME` : Output file
- `-of FORMAT` : Output format (json, csv, html, md, ejson, ecsv, all)

---

**Headers personnalisés :**

```bash
ffuf -w wordlist.txt \
  -H "User-Agent: Mozilla/5.0" \
  -H "Authorization: Bearer token" \
  -u http://target/FUZZ
```

**Options :**

- `-H "Header: Value"` : Add custom header (peut être répété)

---

**Cookies :**

```bash
ffuf -w wordlist.txt -b "session=abc123; token=xyz" -u http://target/FUZZ
```

**Options :**

- `-b "COOKIE"` : Cookie data

---

**Proxy :**

```bash
ffuf -w wordlist.txt -x http://127.0.0.1:8080 -u http://target/FUZZ
```

**Options :**

- `-x PROXY` : HTTP proxy URL

---

**Replay proxy (envoyer matches à proxy) :**

```bash
ffuf -w wordlist.txt -replay-proxy http://127.0.0.1:8080 -u http://target/FUZZ
```

**Options :**

- `-replay-proxy PROXY` : Replay matched requests via proxy

---

**Suivre redirections :**

```bash
ffuf -w wordlist.txt -r -u http://target/FUZZ
```

**Options :**

- `-r` : Follow redirects

---

**Ignorer auto-calibration :**

```bash
ffuf -w wordlist.txt -ac -u http://target/FUZZ
```

**Options :**

- `-ac` : Automatically calibrate filtering options

---

**Fuzzing avec plusieurs FUZZ positions :**

```bash
# Méthode 1 : Plusieurs wordlists
ffuf -w users.txt:FUZZUSER -w passwords.txt:FUZZPASS \
  -u http://target/login \
  -X POST \
  -d 'username=FUZZUSER&password=FUZZPASS'

# Méthode 2 : Keywords personnalisés
ffuf -w wordlist1.txt:W1 -w wordlist2.txt:W2 \
  -u "http://target/W1/W2"
```

**Options :**

- `-w WORDLIST:KEYWORD` : Assigner keyword personnalisé à wordlist

---

**Mode cluster bomb / pitchfork :**

```bash
# Cluster bomb (teste toutes les combinaisons)
ffuf -w users.txt:USER -w passwords.txt:PASS -mode clusterbomb \
  -u http://target/login \
  -d "user=USER&pass=PASS"

# Pitchfork (paire ligne par ligne)
ffuf -w users.txt:USER -w passwords.txt:PASS -mode pitchfork \
  -u http://target/login \
  -d "user=USER&pass=PASS"
```

**Options :**

- `-mode clusterbomb` : Clusterbomb - toutes combinaisons (défaut)
- `-mode pitchfork` : Pitchfork - associer ligne par ligne

---

**Stop conditions :**

```bash
# Arrêter après N résultats
ffuf -w wordlist.txt -maxtime 60 -u http://target/FUZZ

# Arrêter sur erreurs
ffuf -w wordlist.txt -sa -u http://target/FUZZ
```

**Options :**

- `-maxtime SECONDS` : Max running time
- `-sa` : Stop on all error cases

---

**Content-Type courants :**

- `application/x-www-form-urlencoded` : Formulaire standard
- `application/json` : Données JSON
- `multipart/form-data` : Upload fichiers

---

**Auto-calibration (ignore faux positifs) :**

```bash
ffuf -w wordlist.txt -ac -u http://target/FUZZ
```

**Options :**

- `-ac` : Automatically calibrate filtering

---
