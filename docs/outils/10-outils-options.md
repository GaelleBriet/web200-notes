# 🎯<span style="color:orange;"> Options les Plus Importantes à Retenir</span>

## NMAP

- `-sV` : Version services
- `-p` : Port spécifique
- `-p-` : Tous les ports
- `--script` : Scripts NSE
- `-Pn` : Skip host discovery
- `-sC` : Scripts par défaut
- `-A` : Aggressive (OS + version + scripts)
- `-T4` : Fast timing
- `-oN` / `-oX` / `-oG` / `-oA` : Output formats

## CURL

- `-i` : Headers + body
- `-I` : Headers only
- `-L` : Follow redirects
- `-k` : Ignore SSL errors
- `-X METHOD` : HTTP method
- `-d DATA` : POST data
- `-H "Header"` : Custom header
- `-b COOKIE` : Send cookie
- `-c FILE` : Save cookies
- `-A UA` : User-Agent
- `--proxy` : Use proxy
- `-o FILE` : Output to file
- `-s` : Silent
- `-v` : Verbose

## NETCAT

- `-v` : Verbose
- `-l` : Listen mode
- `-p` : Port
- `-e` : Execute program
- `-n` : No DNS
- `-z` : Port scan
- `-u` : UDP mode

## HAKRAWLER

- `-u` : Unique URLs
- `-proxy` : Route via proxy
- `-d` : Depth
- `-s` : Scope regex
- `-insecure` : Ignore SSL
- `-t` : Threads
- `-dr` : Disable robots.txt
- `-h` : Custom headers
- `-cookie` : Cookie

## DIRB

- `-X .ext` : Extensions
- `-o FILE` : Output file
- `-S` : Silent mode
- `-r` : Don't recurse
- `-R` : Interactive recursion
- `-N CODE` : Ignore code
- `-a UA` : User Agent
- `-c COOKIE` : Cookie
- `-H HEADER` : Add header
- `-p PROXY` : Proxy
- `-v` : Verbose

## WFUZZ

- `-c` : Colored output
- `-w` : Wordlist
- `-d` : POST data
- `-u` : URL
- `-X` : HTTP method
- `FUZZ` : Injection point
- `--hc CODE` : Hide code
- `--hh CHARS` : Hide chars
- `--hl LINES` : Hide lines
- `--hw WORDS` : Hide words
- `--sc CODE` : Show code
- `-t THREADS` : Threads
- `-b COOKIE` : Cookie
- `-H "Header"` : Header
- `--follow` : Follow redirects
- `-p PROXY` : Proxy

## FFUF

- `-w` : Wordlist
- `-u` : URL
- `-X` : HTTP method
- `-d` : POST data
- `-H "Header"` : Header
- `FUZZ` : Injection point
- `-fc CODE` : Filter code
- `-fs SIZE` : Filter size **[CRITIQUE]**
- `-fl LINES` : Filter lines
- `-fw WORDS` : Filter words
- `-fr REGEX` : Filter regex
- `-mc CODE` : Match code
- `-ms SIZE` : Match size
- `-t THREADS` : Threads
- `-rate N` : Rate limit
- `-p DELAY` : Pause between requests
- `-e .ext` : Extensions
- `-recursion` : Recursive
- `-recursion-depth N` : Depth
- `-o FILE` : Output file **[CRITIQUE]**
- `-of FORMAT` : Output format (json, csv, html, md)
- `-v` : Verbose
- `-s` : Silent
- `-c` : Colorize
- `-b COOKIE` : Cookie
- `-x PROXY` : Proxy
- `-r` : Follow redirects
- `-ac` : Auto-calibrate

## SQLMAP

- `-u` : URL
- `--data` : POST data
- `-p` : Params to test
- `--method` : HTTP method
- `--dbms` : Force DB type
- `--flush-session` : Ignore cache **[CRITIQUE]**
- `--dbs` : List databases
- `-D` : Database
- `-T` : Table
- `-C` : Columns
- `--tables` : List tables
- `--columns` : List columns
- `--dump` : Extract data
- `--current-db` : Current database
- `--current-user` : Current user
- `--is-dba` : Check DBA
- `--batch` : No interaction **[IMPORTANT]**
- `--random-agent` : Random UA
- `--threads` : Threads
- `--level` : Test level (1-5)
- `--risk` : Risk level (1-3)
- `--technique` : SQL injection technique
- `--tamper` : Tamper script
- `--proxy` : Proxy
- `--cookie` : Cookie
- `--os-shell` : OS shell
- `--sql-shell` : SQL shell
- `--file-read` : Read file
- `--file-write / --file-dest` : Write file
- `-r FILE` : Load request from file

## CEWL

- `--write` / `-w` : Output file
- `--lowercase` : Lowercase
- `-m` : Min word length
- `-d` : Depth
- `-a` : Include meta
- `--with-numbers` : Include numbers
- `--email` / `-e` : Extract emails
- `-c` : Count words
- `--meta_file` : Meta output
- `-v` : Verbose
- `--ua` : User agent

---
