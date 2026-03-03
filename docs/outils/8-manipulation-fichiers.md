## <span style="color:orange;">8. Manipulation de Fichiers</span>

### 8.1 Redirection & Pipes

**Créer fichier :**

```bash
echo "alert(1)" > xss.js
```

**Explication :**

- `echo "texte"` : Affiche texte
- `>` : Redirige sortie vers fichier (écrase si existe)

---

**Ajouter à fichier existant :**

```bash
echo "console.log('test')" >> xss.js
```

**Explication :**

- `>>` : Append (ajoute sans écraser)

---

**Pipe (envoyer output à autre commande) :**

```bash
ls /usr/bin | grep python
cat file.txt | grep "flag"
```

**Explication :**

- `|` : Prend sortie de commande gauche, envoie à commande droite

---

**Rediriger stderr :**

```bash
# Ignorer erreurs
command 2>/dev/null

# Rediriger stderr vers stdout
command 2>&1

# Rediriger stdout ET stderr
command &> output.txt
command > output.txt 2>&1
```

---

### 8.2 Grep

**Chercher dans fichier :**

```bash
grep "flag" results.txt
```

---

**Exclure motif :**

```bash
grep -v "/" output.txt
```

**Options :**

- `-v` : Inverse match (affiche lignes qui NE contiennent PAS le motif)

---

**Case insensitive :**

```bash
grep -i "admin" file.txt
```

**Options :**

- `-i` : Ignore case

---

**Récursif (dans dossiers) :**

```bash
grep -r "password" /var/www/
```

**Options :**

- `-r` : Recursive

---

**Afficher numéros de ligne :**

```bash
grep -n "flag" file.txt
```

**Options :**

- `-n` : Show line numbers

---

**Compter occurrences :**

```bash
grep -c "error" log.txt
```

**Options :**

- `-c` : Count matching lines

---

**Afficher contexte :**

```bash
# 3 lignes avant et après
grep -C 3 "flag" file.txt

# 5 lignes avant
grep -B 5 "flag" file.txt

# 2 lignes après
grep -A 2 "flag" file.txt
```

**Options :**

- `-C N` : Show N lines of context
- `-B N` : Show N lines before
- `-A N` : Show N lines after

---

**Regex :**

```bash
grep -E "admin|root" file.txt
```

**Options :**

- `-E` : Extended regex

---

**Chercher dans logs :**

```bash
grep "GET /k?key=" server.log | sed 's/.*key=\(.\).*/\1/' | tr -d '\n' && echo
```

**Explication :**

1. `grep "GET /k?key="` : Filtre lignes contenant ce motif
2. `sed 's/.*key=\(.\).*/\1/'` : Extrait caractère après `key=`
3. `tr -d '\n'` : Supprime newlines (tout sur une ligne)
4. `&& echo` : Ajoute newline à la fin

---

### 8.3 Édition Fichiers

**Nano (éditeur simple) :**

```bash
nano xss.js
```

**Commandes Nano :**

- `Ctrl+O` : Sauvegarder
- `Ctrl+X` : Quitter
- `Ctrl+K` : Couper ligne
- `Ctrl+U` : Coller
- `Ctrl+W` : Chercher
- `Ctrl+\` : Remplacer

---

**Vi/Vim :**

```bash
vi file.txt
vim file.txt
```

**Commandes Vi :**

- `i` : Insert mode
- `ESC` : Command mode
- `:w` : Save
- `:q` : Quit
- `:wq` : Save and quit
- `:q!` : Quit without saving
- `dd` : Delete line
- `yy` : Copy line
- `p` : Paste

---

**Cat (afficher contenu) :**

```bash
cat /etc/passwd
cat flag.txt
```

---

**Head / Tail :**

```bash
# 10 premières lignes
head file.txt
head -n 20 file.txt

# 10 dernières lignes
tail file.txt
tail -n 50 file.txt

# Suivre fichier en temps réel
tail -f /var/log/apache2/access.log
```

---

**Less / More (paginer) :**

```bash
less file.txt  # Navigation avec flèches, q pour quitter
more file.txt
```

---

**Wc (word count) :**

```bash
wc file.txt        # Lignes, mots, chars
wc -l file.txt     # Nombre de lignes
wc -w file.txt     # Nombre de mots
wc -c file.txt     # Nombre de chars
```

---

**Sort / Uniq :**

```bash
# Trier
sort file.txt

# Trier et supprimer doublons
sort file.txt | uniq

# Compter occurrences
sort file.txt | uniq -c

# Trier par fréquence
sort file.txt | uniq -c | sort -rn
```

---

**Cut (extraire colonnes) :**

```bash
# Extraire champ 1 (délimiteur :)
cut -d':' -f1 /etc/passwd

# Extraire colonnes 1-3
cut -d':' -f1-3 /etc/passwd
```

---

**Awk (manipulation texte avancée) :**

```bash
# Afficher colonne 1
awk '{print $1}' file.txt

# Délimiteur personnalisé
awk -F':' '{print $1}' /etc/passwd

# Condition
awk '$3 > 100 {print $1}' file.txt
```

---

**Sed (stream editor) :**

```bash
# Remplacer
sed 's/old/new/g' file.txt

# Supprimer lignes contenant mot
sed '/password/d' file.txt

# Supprimer lignes vides
sed '/^$/d' file.txt
```

---

### 8.4 Transfert de Fichiers

**Wget :**

```bash
wget http://192.168.49.56/file.txt
wget -O custom_name.txt http://192.168.49.56/file.txt
```

---

**Curl :**

```bash
curl http://192.168.49.56/file.txt -o file.txt
curl -O http://192.168.49.56/file.txt
```

---

**Base64 (si pas de wget/curl) :**

```bash
# Sur Kali
cat file.txt | base64 -w 0

# Sur cible
echo "base64_string" | base64 -d > file.txt
```

---

**SCP (si SSH access) :**

```bash
# Upload vers cible
scp file.txt user@target:/tmp/

# Download depuis cible
scp user@target:/tmp/file.txt .
```

---