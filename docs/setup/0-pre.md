
```bash
# Lancer le vpn
cd Downloads
sudo openvpn universal.ovpn
 
# Editer hosts
sudo nano /etc/hosts

# apache
sudo systemctl restart apache2
sudo tail -f /var/log/apache2/access.log
sudo truncate -s 0 /var/log/apache2/access.log

# vpn check
cd Downloads
sudo ./troubleshooting.sh

#google DNS
sudo nano /etc/resolv.conf  
nameserver 8.8.8.8
nameserver 8.8.4.4
sudo systemctl daemon-reload
```

---

**VM kali**
```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl start ssh
sudo systemctl enable ssh
```

vérifier que ssh ecoute le port 22
```bash
sudo ss -tlnp | grep :22
```

récupérer l'ip de la vm
```bash
 ip a | grep "inet " | grep -v 127
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute eth0
    inet 192.168.45.208/24 brd 192.168.45.255 scope global tun0
    inet 192.168.45.208/24 brd 192.168.45.255 scope global tun1

```

**hôte**
```bash
ssh -p 2222 kali@127.0.0.1
```

transférer un dossier vm -> hôte
```bash
scp -r -P 2222 kali@127.0.0.1:/home/kali/Images /media/yuki/DisqueSecondaire/OSWA/
```

transférer une image
```bash
scp -P 2222 kali@127.0.0.1:/home/kali/Images/Screenshot_2026-01-27_13_32_48.png /media/yuki/DisqueSecondaire/OSWA/
```

Serveur python
```bash
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        print(self.headers)
        self.send_response(200)
        self.end_headers()

    def do_POST(self):
        print(self.headers)
        self.send_response(200)
        self.end_headers()

HTTPServer(('', 8000), Handler).serve_forever()
```