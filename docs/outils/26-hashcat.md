# Hashcat — Cracking de Hashes

## 1. Concept

Hashcat est un outil de cracking de mots de passe par force brute, dictionnaire ou règles. Il utilise le **GPU** pour une vitesse maximale, mais fonctionne aussi en CPU.

**Cas d'usage :**

- Cracker un hash récupéré via SQLi (`$P$`, MD5, SHA1...)
- Cracker la clé secrète d'un JWT HS256
- Cracker un hash `/etc/shadow` obtenu post-RCE

---

## 2. Syntaxe de base

```bash
hashcat -a MODE -m TYPE hash.txt wordlist.txt
```

| Paramètre      | Description                      |
|----------------|----------------------------------|
| `-a MODE`      | Mode d'attaque                   |
| `-m TYPE`      | Type de hash                     |
| `hash.txt`     | Fichier contenant le(s) hash(es) |
| `wordlist.txt` | Wordlist                         |

---

## 3. Modes d'attaque (-a)

| Mode   | Nom                       | Description                      |
|--------|---------------------------|----------------------------------|
| `-a 0` | Dictionnaire              | Tester chaque mot de la wordlist |
| `-a 1` | Combinaison               | Combiner deux wordlists          |
| `-a 3` | Brute-force               | Masque de caractères             |
| `-a 6` | Hybride wordlist + masque | Mot + suffixe                    |
| `-a 7` | Hybride masque + wordlist | Préfixe + mot                    |

**Le plus utilisé en pentest :** `-a 0` (dictionnaire)

---

## 4. Types de hash (-m) — Les plus courants

| Code    | Type                      | Exemple de hash                                                    |
|---------|---------------------------|--------------------------------------------------------------------|
| `0`     | MD5                       | `5f4dcc3b5aa765d61d8327deb882cf99`                                 |
| `100`   | SHA1                      | `5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8`                         |
| `1400`  | SHA256                    | `ef92b778bafe771e89245b89ecbc08a44a4e166c06659911881f383d4473e94f` |
| `1700`  | SHA512                    | (128 chars hex)                                                    |
| `400`   | WordPress / phpBB (`$P$`) | `$P$Ghxmchgk.0YxEQutC7os3dZfBvqGIz/`                               |
| `500`   | md5crypt (`$1$`)          | `$1$abc$...`                                                       |
| `1800`  | SHA-512 Unix (`$6$`)      | `$6$salt$...`                                                      |
| `3200`  | bcrypt (`$2y$`)           | `$2y$10$...`                                                       |
| `16500` | JWT HS256                 | `eyJhbGci...` (token complet)                                      |
| `1000`  | NTLM (Windows)            | `8846f7eaee8fb117ad06bdd830b7586c`                                 |

### Identifier un hash inconnu

```bash
# hashcat intégré
hashcat --identify hash.txt

# hashid (outil Kali)
hashid '$P$Ghxmchgk.0YxEQutC7os3dZfBvqGIz/'

# hash-identifier
hash-identifier
```

---

## 5. Commandes essentielles

### Attaque dictionnaire simple

```bash
hashcat -a 0 -m 0 hash.txt /usr/share/wordlists/rockyou.txt
```

### Avec affichage en temps réel

```bash
hashcat -a 0 -m 0 hash.txt /usr/share/wordlists/rockyou.txt --status
```

### Forcer recalcul (si déjà cracké en cache)

```bash
hashcat -a 0 -m 0 hash.txt /usr/share/wordlists/rockyou.txt --force
```

### Afficher les résultats après crack

```bash
hashcat -a 0 -m 0 hash.txt /usr/share/wordlists/rockyou.txt --show
```

### Sauvegarder les résultats dans un fichier

```bash
hashcat -a 0 -m 0 hash.txt /usr/share/wordlists/rockyou.txt -o cracked.txt
```

---

## 6. Cas pratiques

### Hash WordPress `$P$` (récupéré via SQLi)

```bash
# Hash exemple : $P$Ghxmchgk.0YxEQutC7os3dZfBvqGIz/
echo '$P$Ghxmchgk.0YxEQutC7os3dZfBvqGIz/' > hash.txt

hashcat -a 0 -m 400 hash.txt /usr/share/wordlists/rockyou.txt
```

### Hash MD5 simple

```bash
echo '5f4dcc3b5aa765d61d8327deb882cf99' > hash.txt
hashcat -a 0 -m 0 hash.txt /usr/share/wordlists/rockyou.txt
```

### Hash SHA1

```bash
echo '5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8' > hash.txt
hashcat -a 0 -m 100 hash.txt /usr/share/wordlists/rockyou.txt
```

### Hash `/etc/shadow` (SHA-512 Unix `$6$`)

```bash
# Récupérer la ligne depuis /etc/shadow
echo '$6$salt$longhashvalue...' > hash.txt
hashcat -a 0 -m 1800 hash.txt /usr/share/wordlists/rockyou.txt
```

### Clé secrète JWT HS256

```bash
# Le token JWT complet (les 3 parties)
echo 'eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyIjoiZ3Vlc3QifQ.SIGNATURE' > jwt.txt
hashcat -a 0 -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt
```

### Brute-force masque (si longueur connue)

```bash
# Masque : ?l = lowercase, ?u = uppercase, ?d = digit, ?s = special, ?a = all
# Mot de passe de 6 chiffres
hashcat -a 3 -m 0 hash.txt ?d?d?d?d?d?d

# Mot de passe 8 chars alphanum
hashcat -a 3 -m 0 hash.txt ?a?a?a?a?a?a?a?a

# Minuscules + chiffres, 6 chars
hashcat -a 3 -m 0 hash.txt ?l?l?l?l?d?d
```

### Avec règles (amplifier la wordlist)

```bash
# Règles classiques : majuscules, leetspeak, suffixes numériques...
hashcat -a 0 -m 0 hash.txt /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule

# Règles populaires
ls /usr/share/hashcat/rules/
# best64.rule      → 64 transformations courantes
# rockyou-30000.rule → très complet
# d3ad0ne.rule     → nombreuses variations
```

---

## 7. Options utiles

| Option              | Description                           |
|---------------------|---------------------------------------|
| `--show`            | Afficher les hashes déjà crackés      |
| `--status`          | Affichage statut en temps réel        |
| `--force`           | Ignorer les warnings (utile en VM)    |
| `-o FILE`           | Sauvegarder résultats                 |
| `--potfile-disable` | Ne pas utiliser le cache potfile      |
| `-w 3`              | Workload profile (1=low, 4=nightmare) |
| `--username`        | Le fichier hash contient `user:hash`  |
| `--increment`       | Brute-force longueur croissante       |

---

## 8. Wordlists recommandées

```bash
# La référence absolue
/usr/share/wordlists/rockyou.txt

# Plus complètes (SecLists)
/usr/share/wordlists/seclists/Passwords/Common-Credentials/10k-most-common.txt
/usr/share/wordlists/seclists/Passwords/Common-Credentials/100k-most-used-passwords-NCSC.txt
/usr/share/wordlists/seclists/Passwords/xato-net-10-million-passwords.txt
```

---

## 9. Workflow exam

```
1. Récupérer le hash (SQLi dump, /etc/shadow, JWT, fichier config...)

2. Identifier le type
   hashcat --identify hash.txt
   ou hashid 'HASH'

3. Mettre le hash dans un fichier
   echo 'HASH' > hash.txt
   ⚠️ Pour $P$, $6$ : utiliser des quotes simples

4. Lancer hashcat avec rockyou
   hashcat -a 0 -m TYPE hash.txt /usr/share/wordlists/rockyou.txt

5. Si pas de résultat → essayer avec règles
   hashcat -a 0 -m TYPE hash.txt /usr/share/wordlists/rockyou.txt \
     -r /usr/share/hashcat/rules/best64.rule

6. Afficher le résultat
   hashcat -a 0 -m TYPE hash.txt /usr/share/wordlists/rockyou.txt --show
```

!!! tip "Hash avec `$` dans le shell"
    Les `$` dans les hashes Unix (`$P$`, `$6$`...) sont interprétés par bash.  
    Toujours utiliser des **guillemets simples** ou un fichier :  
    ```bash
    echo '$P$Ghxmchgk.0YxEQutC7os3dZfBvqGIz/' > hash.txt  # ✅
    echo "$P$Ghxmchgk..."  # ❌ bash va interpréter $P
    ```