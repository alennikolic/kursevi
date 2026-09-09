# Instalacija Dell NetWorker 19.13 na RHEL 9.6 — sve komponente na jednom serveru

**Namena:** uputstvo za učenje i kao radna procedura za sysadmine.
**Izvor:** *NetWorker 19.13 Installation Guide* (poglavlje "CentOS, OEL, SuSE and RHEL Installation" i "Verify the Installation").

---

## 0. Šta se instalira

Sve na jednom hostu (tzv. "all-in-one" NetWorker server):

| Komponenta | RPM paket | Podrazumevana lokacija |
|---|---|---|
| NetWorker Client | `lgtoclnt` | `/usr/sbin`, `/usr/bin`, `/usr/lib`, `/opt/nsr` |
| Extended Client | `lgtoxtdclnt` | `/usr/sbin` |
| Storage Node | `lgtonode` | `/usr/sbin`, `/usr/lib` |
| NetWorker Server | `lgtoserv` | `/usr/sbin` |
| Authentication Service | `lgtoauthc` | `/opt/nsr/authc-server` |
| NMC (Management Console) server | `lgtonmc` | `/opt/lgtonmc` |
| Web UI (NWUI) na NW serveru | `lgtonwuiserv` | `/opt/nwui` |
| Man stranice (opciono) | `lgtoman` | — |
| Block Based Backup (opciono) | `lgtobbb-nw` | — |
| Message Queue Adapter (opciono, za NMM) | `lgtoadpt` | — |

Baze (client file index, media DB, resource DB) su u `/nsr`.

**Redosled instalacije je bitan:** `lgtoclnt` → `lgtoxtdclnt` → `lgtonode` → `lgtoserv` → `lgtoauthc` → `lgtonmc` → `lgtonwuiserv`.

Podrazumevani portovi:

| Servis | Port |
|---|---|
| Authentication Service (Tomcat) | 9090 |
| NMC web (https) | 9000 |
| NMC GST | 9001 |
| NMC Postgres | 5432 |
| NWUI monitoring app | 9095 |
| NWUI Postgres | 5435 |
| NetWorker servisni opseg (RPC) | 7937–9936 |

---

## 1. Preduslovi

### 1.1 Hardver / disk

- Raspakovan `nw19.13_linux_x86_64.tar.gz` paket zauzima ~1.26 GB (+ ~1.12 GB arhiva).
- `/nsr` — minimum 2 GB za start, realno planirati **desetine GB** (indeksi i media baza rastu).
- `/opt` — minimum 5 GB (NMC, NWUI, Tomcat, Postgres).

### 1.2 Hostname i DNS

Hostname mora biti razrešiv (DNS ili `/etc/hosts`) i, ako se koristi Data Domain, **mora biti mala slova**.

```bash
[root@nwsrv ~]# hostnamectl set-hostname nwsrv.firma.local
[root@nwsrv ~]# hostname -f
nwsrv.firma.local

[root@nwsrv ~]# ping -c1 nwsrv.firma.local
PING nwsrv.firma.local (10.10.20.15) 56(84) bytes of data.
64 bytes from nwsrv.firma.local (10.10.20.15): icmp_seq=1 ttl=64 time=0.031 ms
```

Ako nema DNS zapisa, dodati u `/etc/hosts`:

```bash
[root@nwsrv ~]# cat >> /etc/hosts <<'EOF'
10.10.20.15  nwsrv.firma.local  nwsrv
EOF
```

### 1.3 Backup konfiguracionih fajlova

```bash
[root@nwsrv ~]# cp /etc/rpc /etc/rpc.orig
[root@nwsrv ~]# cp /etc/ld.so.conf /etc/ld.so.conf.orig
```

### 1.4 PATH

`/usr/sbin` mora biti u PATH-u za root i korisnike:

```bash
[root@nwsrv ~]# echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
```

### 1.5 SELinux — privremeno u permissive

```bash
[root@nwsrv ~]# getenforce
Enforcing

[root@nwsrv ~]# setenforce permissive
[root@nwsrv ~]# getenforce
Permissive
```

> Ovo je **privremeno, samo za vreme instalacije RPM-ova**. Posle instalacije vraćamo na prethodnu vrednost (korak 11).

### 1.6 OS paketi

RHEL 9 zahteva `libnsl`, a NetWorker takođe traži `ksh` i 32-bitne biblioteke:

```bash
[root@nwsrv ~]# dnf install -y ksh libnsl libnsl.i686 glibc.i686 nss-softokn-freebl.i686 libxcrypt-compat
...
Installed:
  glibc-2.34-125.el9_6.i686           ksh-1.0.0~beta.1-3.el9.x86_64
  libnsl-2.34-125.el9_6.x86_64        libnsl-2.34-125.el9_6.i686
  libxcrypt-compat-4.4.18-3.el9.x86_64
  nss-softokn-freebl-3.101.0-11.el9_6.i686

Complete!
```

Provera:

```bash
[root@nwsrv ~]# rpm -q ksh libnsl glibc.i686
ksh-1.0.0~beta.1-3.el9.x86_64
libnsl-2.34-125.el9_6.x86_64
glibc-2.34-125.el9_6.i686
```

### 1.7 Cockpit — konflikt na portu 9090

Na RHEL-u cockpit koristi port 9090, isti kao NetWorker Authentication Service. **Ugasiti cockpit** (ili kasnije birati drugi port za authc).

```bash
[root@nwsrv ~]# systemctl disable --now cockpit.socket
Removed "/etc/systemd/system/sockets.target.wants/cockpit.socket".

[root@nwsrv ~]# ss -lntp | grep 9090
[root@nwsrv ~]#
```

Prazan izlaz = port je slobodan.

### 1.8 Provera init sistema

```bash
[root@nwsrv ~]# ps -p 1
    PID TTY          TIME CMD
      1 ?        00:00:04 systemd
```

`systemd` — sve komande u nastavku koriste `systemctl`.

---

## 2. Firewall

```bash
[root@nwsrv ~]# firewall-cmd --permanent --add-port=7937-9936/tcp
success
[root@nwsrv ~]# firewall-cmd --permanent --add-port=9090/tcp
success
[root@nwsrv ~]# firewall-cmd --permanent --add-port=9000/tcp
success
[root@nwsrv ~]# firewall-cmd --permanent --add-port=9001/tcp
success
[root@nwsrv ~]# firewall-cmd --permanent --add-port=5432/tcp
success
[root@nwsrv ~]# firewall-cmd --reload
success

[root@nwsrv ~]# firewall-cmd --list-ports
7937-9936/tcp 9090/tcp 9000/tcp 9001/tcp 5432/tcp
```

> Portovi 9000/9001/5432 su potrebni NMC klijentima. Portovi 9095 i 5435 (NWUI) su lokalni i obično se ne otvaraju spolja.

---

## 3. Raspakivanje instalacionog paketa

Paket se preuzima sa Dell Online Support sajta u npr. `/tmp/nw`.

```bash
[root@nwsrv ~]# mkdir -p /tmp/nw && cd /tmp/nw
[root@nwsrv nw]# tar -xzf nw19.13_linux_x86_64.tar.gz
[root@nwsrv nw]# ls -1 *.rpm
lgtoadpt-19.13-1.x86_64.rpm
lgtoauthc-19.13-1.x86_64.rpm
lgtobbb-nw-19.13-1.x86_64.rpm
lgtoclnt-19.13-1.x86_64.rpm
lgtoman-19.13-1.x86_64.rpm
lgtonmc-19.13-1.x86_64.rpm
lgtonode-19.13-1.x86_64.rpm
lgtonwuiserv-19.13-1.x86_64.rpm
lgtoserv-19.13-1.x86_64.rpm
lgtoxtdclnt-19.13-1.x86_64.rpm
```

---

## 4. Instalacija Jave (NRE 17)

NetWorker server traži 64-bit Java 17 — preporučeno NetWorker Runtime Environment (NRE), koji se instalira u `/opt/nre/java/latest`.

```bash
[root@nwsrv nw]# rpm -ivh nre-17.0.3-1.x86_64.rpm
Verifying...                          ################################# [100%]
Preparing...                          ################################# [100%]
Updating / installing...
   1:nre-17.0.3-1                     ################################# [100%]

[root@nwsrv nw]# /opt/nre/java/latest/bin/java -version
java version "17.0.12" 2024-07-16 LTS
Java(TM) SE Runtime Environment (build 17.0.12+8-LTS-286)
Java HotSpot(TM) 64-Bit Server VM (build 17.0.12+8-LTS-286, mixed mode, sharing)
```

> Ako se koristi Oracle JDK umesto NRE, podrazumevana putanja je `/usr/java/latest` i to treba uneti kada authc skripta pita za Java direktorijum.

---

## 5. Instalacija NetWorker paketa

Instaliramo klijent, extended client, storage node, server i authentication service u jednom prolazu:

```bash
[root@nwsrv nw]# dnf localinstall --nogpgcheck -y \
  lgtoclnt-19.13-1.x86_64.rpm \
  lgtoxtdclnt-19.13-1.x86_64.rpm \
  lgtonode-19.13-1.x86_64.rpm \
  lgtoauthc-19.13-1.x86_64.rpm \
  lgtoserv-19.13-1.x86_64.rpm \
  lgtoman-19.13-1.x86_64.rpm
```

Primer izlaza (skraćeno):

```
Dependencies resolved.
=========================================================================
 Package          Arch     Version      Repository            Size
=========================================================================
Installing:
 lgtoauthc        x86_64   19.13-1      @commandline          98 M
 lgtoclnt         x86_64   19.13-1      @commandline         185 M
 lgtoman          x86_64   19.13-1      @commandline         1.2 M
 lgtonode         x86_64   19.13-1      @commandline          22 M
 lgtoserv         x86_64   19.13-1      @commandline         320 M
 lgtoxtdclnt      x86_64   19.13-1      @commandline          40 M

Transaction Summary
=========================================================================
Install  6 Packages

Running transaction
  Installing : lgtoclnt-19.13-1.x86_64                          1/6
  Installing : lgtoxtdclnt-19.13-1.x86_64                       2/6
  Installing : lgtonode-19.13-1.x86_64                          3/6
  Installing : lgtoauthc-19.13-1.x86_64                         4/6
  Installing : lgtoserv-19.13-1.x86_64                          5/6
  Installing : lgtoman-19.13-1.x86_64                           6/6

NOTE: To complete configuration execute the following script as root:
      /opt/nsr/authc-server/scripts/authc_configure.sh

Complete!
```

Provera:

```bash
[root@nwsrv nw]# rpm -qa | grep lgto
lgtoclnt-19.13-1.x86_64
lgtoxtdclnt-19.13-1.x86_64
lgtonode-19.13-1.x86_64
lgtoauthc-19.13-1.x86_64
lgtoserv-19.13-1.x86_64
lgtoman-19.13-1.x86_64
```

> Ako `dnf` prijavi nedostajuće zavisnosti, instalirati ih iz RHEL repozitorijuma i ponoviti komandu. `rpm -ivh` radi isto, ali ne rešava zavisnosti automatski.

---

## 6. Konfiguracija Authentication Service

```bash
[root@nwsrv nw]# /opt/nsr/authc-server/scripts/authc_configure.sh
```

Tok skripte (unos je označen sa `-->`):

```
Specify the directory where the Java Standard Edition Runtime Environment
software is installed [/opt/nre/java/latest]:            --> <Enter>

Specify the port that Apache Tomcat should use for communication [9090]:
                                                          --> <Enter>

Specify the keystore password:                            --> ********
Confirm the password:                                     --> ********

Specify an initial password for administrator:            --> *********
Confirm the password:                                     --> *********

Creating user nsrtomcat ...
Configuring the NetWorker Authentication Service ...
Starting the NetWorker Authentication Service ...

The NetWorker Authentication Service configuration completed successfully.
```

Pravila za lozinke:

- **keystore lozinka** — min. 6 karaktera, bez rečničkih reči.
- **administrator lozinka** — min. 9 karaktera, min. 1 veliko slovo, 1 malo, 1 broj, 1 specijalni znak. Ovom lozinkom se kasnije prijavljujete na NMC.

Skripta kreira OS korisnika `nsrtomcat` (koristi ga samo Tomcat interno) i nalog `administrator` u lokalnoj bazi authentication servisa.

> Ako se pojavi `Warning: Port 9090 is already in use` — niste ugasili cockpit (korak 1.7). Ili ga ugasite pa ponovite, ili unesite drugi port (1024–49151).

---

## 7. Start NetWorker servisa

```bash
[root@nwsrv ~]# systemctl start networker
[root@nwsrv ~]# systemctl enable networker
Created symlink /etc/systemd/system/multi-user.target.wants/networker.service → /usr/lib/systemd/system/networker.service.

[root@nwsrv ~]# systemctl status networker
● networker.service - NetWorker
   Loaded: loaded (/usr/lib/systemd/system/networker.service; enabled)
   Active: active (running) since Wed 2026-09-09 10:22:41 CEST; 35s ago
 Main PID: 21877 (nsrd)
    Tasks: 96
   CGroup: /system.slice/networker.service
           ├─21850 /usr/sbin/nsrexecd
           ├─21877 /usr/sbin/nsrd
           ├─21901 /usr/sbin/nsrmmdbd
           ├─21903 /usr/sbin/nsrindexd
           ├─21905 /usr/sbin/nsrjobd
           └─21912 /usr/sbin/nsrlcpd
```

Provera procesa:

```bash
[root@nwsrv ~]# ps -ef | grep /usr/sbin/nsr | grep -v grep
root     21850     1  0 10:22 ?  00:00:00 /usr/sbin/nsrexecd
root     21877     1  0 10:22 ?  00:00:02 /usr/sbin/nsrd
root     21901 21877  0 10:22 ?  00:00:00 /usr/sbin/nsrmmdbd
root     21903 21877  0 10:22 ?  00:00:00 /usr/sbin/nsrindexd
root     21905 21877  0 10:22 ?  00:00:00 /usr/sbin/nsrjobd
```

Provera da je `/nsr` popunjen:

```bash
[root@nwsrv ~]# ls /nsr
authc  cores  db6  debug  index  logs  mm  nmc  res  tmp
```

---

## 8. Instalacija i konfiguracija NMC servera

### 8.1 Instalacija paketa

```bash
[root@nwsrv nw]# dnf localinstall --nogpgcheck -y lgtonmc-19.13-1.x86_64.rpm
...
Installing:
 lgtonmc          x86_64   19.13-1      @commandline         410 M

Running transaction
  Installing : lgtonmc-19.13-1.x86_64                           1/1

NOTE: To complete configuration execute the following script as root:
      /opt/lgtonmc/bin/nmc_config

Complete!
```

### 8.2 Konfiguracija

Pre pokretanja proveriti da NetWorker servisi rade (korak 7).

```bash
[root@nwsrv ~]# /opt/lgtonmc/bin/nmc_config
```

Tok skripte:

```
Do you want to create new(cn) certificate or use existing(ue) certificate [cn]?
                                                          --> cn <Enter>

Specify the directory to use for the NMC database [/nsr/nmc/nmcdb]:
                                                          --> <Enter>

Do you want to migrate data from a previous 8.x.x release? [n]:
                                                          --> n <Enter>

Specify the host name of the NetWorker Authentication Service host
[nwsrv.firma.local]:                                      --> <Enter>

Do you want to start the NMC server daemons now? [y]:     --> y <Enter>

Initializing the NMC database ...
Starting the NMC server daemons ...

NMC configuration completed successfully.
```

### 8.3 Provera

```bash
[root@nwsrv ~]# ps -ef | grep lgtonmc | grep -v grep
nsrnmc   23110     1  0 10:41 ?  00:00:06 /opt/lgtonmc/bin/gstd
nsrnmc   23116     1  0 10:41 ?  00:00:00 /opt/lgtonmc/apache/bin/httpd -f /opt/lgtonmc/apache/conf/httpd.conf
nsrnmc   23117 23116  0 10:41 ?  00:00:00 /opt/lgtonmc/apache/bin/httpd -f /opt/lgtonmc/apache/conf/httpd.conf
nsrnmc   23132     1  0 10:41 ?  00:00:00 /opt/lgtonmc/postgres/bin/postgres -D /nsr/nmc/nmcdb/pgdata

[root@nwsrv ~]# systemctl status gst
● gst.service - NetWorker Management Console
   Loaded: loaded (/usr/lib/systemd/system/gst.service; enabled)
   Active: active (running) since Wed 2026-09-09 10:41:03 CEST

[root@nwsrv ~]# ss -lnt | egrep '9000|9001|5432'
LISTEN 0  128  *:9000  *:*
LISTEN 0  128  *:9001  *:*
LISTEN 0  128  *:5432  *:*
```

Portove možete potvrditi i u `/opt/lgtonmc/etc/gstd.conf`:

```bash
[root@nwsrv ~]# grep port /opt/lgtonmc/etc/gstd.conf
db_svc_port=5432
http_svc_port=9000
```

---

## 9. Instalacija NetWorker Web UI (NWUI)

Zavisi od `lgtoserv` — mora biti već instaliran (jeste, korak 5).

```bash
[root@nwsrv nw]# rpm -ivh lgtonwuiserv-19.13-1.x86_64.rpm
Preparing...                          ################################# [100%]
Updating / installing...
   1:lgtonwuiserv-19.13-1             ################################# [100%]

NOTE: To complete configuration execute the following script as root:
      /opt/nwui/scripts/nwui_configure.sh
```

Konfiguracija:

```bash
[root@nwsrv nw]# /opt/nwui/scripts/nwui_configure.sh
```

```
Specify the host name of the NetWorker Authentication Service host:
                                                --> nwsrv.firma.local

Specify the host name of the NetWorker Server to be Managed by NWUI:
                                                --> nwsrv.firma.local

Specify the port for Authentication service on NetWorker Server [9090]:
                                                --> <Enter>

Deploying NetWorker Management Web UI ...
Starting the NWUI monitoring service ...

The installation completed successfully.
```

Provera:

```bash
[root@nwsrv ~]# ss -lnt | egrep '9095|5435'
LISTEN 0  128  127.0.0.1:9095  *:*
LISTEN 0  128  127.0.0.1:5435  *:*
```

Pristup: `https://nwsrv.firma.local:9090/nwui`

> Na NetWorker serveru NWUI je vezan za NetWorker servis — stop/start NWUI-ja radi se preko `systemctl stop|start networker`. Svi NWUI procesi rade kao korisnik `nsrnwui`.

---

## 10. Verifikacija instalacije

### 10.1 Osnovne provere sa CLI-ja

```bash
[root@nwsrv ~]# nsradmin -p nsrd
NetWorker administration program.
Use the "help" command for help.
nsradmin> show name; version
nsradmin> print type: NSR
                        name: nwsrv.firma.local;
                     version: 19.13.0.0.Build.123;
nsradmin> quit
```

```bash
[root@nwsrv ~]# nsrwatch
```

Prikazuje živi status servera (Server, Up since, devices, sessions, messages). Izlaz iz `nsrwatch` je `q`.

```bash
[root@nwsrv ~]# nsrports
Service ports: 7937-9936
Connection ports: 0-0
```

### 10.2 Prijava na NMC (GUI)

1. Na klijentskoj mašini instalirati **OpenWebStart** i JDK 17 / NRE 17.
2. U browseru otvoriti `https://nwsrv.firma.local:9000`.
3. Preuzeti `gconsole.jnlp` i otvoriti ga sa OpenWebStart.
4. Prijaviti se kao `administrator` sa lozinkom iz koraka 6.

### 10.3 Test backup

```bash
[root@nwsrv ~]# echo "test" > /tmp/test.txt
[root@nwsrv ~]# save -s nwsrv.firma.local /tmp/test.txt
save: /tmp/test.txt  4 B 00:00:01     1 file(s)
completed savetime=1757404912
```

```bash
[root@nwsrv ~]# mminfo -s nwsrv.firma.local
 volume        client       date      size   level  name
 Default.001   nwsrv        09/09/26   4 KB   full   /tmp/test.txt
```

---

## 11. Vraćanje SELinux-a

Posle uspešne instalacije vratiti SELinux na prethodno stanje:

```bash
[root@nwsrv ~]# setenforce enforcing
[root@nwsrv ~]# getenforce
Enforcing
```

Ako se posle toga javljaju AVC denial poruke, proveriti:

```bash
[root@nwsrv ~]# ausearch -m avc -ts recent | grep -i nsr
```

---

## 12. Licenciranje (ukratko)

NetWorker 19.x koristi Dell Licensing Solution — jedan licencni fajl sa capacity entitlement-om.

**Unserved licenca (najčešće):**

1. U NMC-u otvoriti **NetWorker Administration**, desni klik na server → **Properties**.
2. Tab **Licensing** → **CLP license** → **Browse** → izabrati licencni fajl → **Validate license**.
3. Provera sa CLI-ja:

```bash
[root@nwsrv ~]# nsrlic -C
CLP license status: Authorized
Capacity: 100 TB   Expiration: No expiration date
```

4. U NMC-u **Server → Registrations** treba da postoji stavka `CLP Capacity License` sa statusom *Authorized – No expiration date*.

**Served licenca** dodatno zahteva instalaciju Dell License Servera i kopiranje licencnog fajla u `/opt/emc/licenses` (fajl se ne preimenuje).

---

## 13. Bitne putanje i log fajlovi

| Šta | Putanja |
|---|---|
| NetWorker baze | `/nsr` (`res`, `index`, `mm`, `logs`) |
| NetWorker log | `/nsr/logs/daemon.raw` (čita se sa `nsr_render_log`) |
| Authentication service | `/opt/nsr/authc-server`, log: `/nsr/authc/tomcat/logs/catalina.out` |
| NMC binarni | `/opt/lgtonmc` |
| NMC log | `/opt/lgtonmc/logs/` (`gstd.raw`, `install.log`, `web_output`) |
| NMC baza | `/nsr/nmc/nmcdb` |
| NWUI | `/opt/nwui`, `/nsr/nwui`, log: `/nsr/authc-server/tomcat/logs/nwui.log` |

Čitanje NetWorker log fajla:

```bash
[root@nwsrv ~]# nsr_render_log /nsr/logs/daemon.raw | tail -20
09/09/26 10:22:41 nsrd NSR info Server started.
09/09/26 10:22:44 nsrd NSR info Media db is ready.
```

---

## 14. Česti problemi

| Simptom | Uzrok / rešenje |
|---|---|
| `Warning: Port 9090 is already in use` | Cockpit radi na 9090 → `systemctl disable --now cockpit.socket` |
| `dnf` prijavljuje missing dependencies | Instalirati OS pakete (`libnsl`, `ksh`, `glibc.i686`) pa ponoviti |
| NMC stranica `https://host:9000` se ne otvara | Proveriti `gstd`, `httpd`, `postgres` procese i da su portovi 9000/9001/5432 otvoreni na firewall-u |
| `gstd` ne startuje, greška `Unable to get authentication service host name` | Ponovo pokrenuti `/opt/lgtonmc/bin/nmc_config` |
| `ERROR: User nsrtomcat does not have read permission at path /nsr/authc/conf` | Dodeliti korisniku `nsrtomcat` prava čitanja na `/nsr/authc/conf` |
| `Could not authenticate this username and password` | Pogrešna lozinka za `administrator`, ili očistiti Java keš na klijentu |

---

## 15. Deinstalacija (za slučaj rollback-a)

Redosled je bitan zbog međuzavisnosti:

```bash
[root@nwsrv ~]# systemctl stop networker gst
[root@nwsrv ~]# rpm -qa | grep lgto
lgtonwuiserv-19.13-1.x86_64
lgtonmc-19.13-1.x86_64
lgtoserv-19.13-1.x86_64
...

[root@nwsrv ~]# rpm -e lgtonwuiserv lgtoserv lgtonode lgtonmc lgtoclnt lgtoxtdclnt lgtoauthc lgtoman
```

Zvanični redosled uklanjanja: `lgtolicm`, `lgtoserv`, `lgtonode`, `lgtonmc`, `lgtoclnt`, `lgtobbb`, `lgtoadpt`, `lgtoxtdclnt`, `lgtoauthc`. Man i jezički paketi nemaju zavisnosti.

Ako se ne planira reinstalacija, obrisati i `/nsr`:

```bash
[root@nwsrv ~]# rm -rf /nsr
```

---

## 16. Checklist za produkciju

- [ ] FQDN razrešiv, mala slova, u `/etc/hosts` ili DNS
- [ ] `/nsr` na zasebnom disku sa dovoljno prostora (simlink ako treba: `ln -s /disk2/nsr /nsr`)
- [ ] Cockpit ugašen ili authc na drugom portu
- [ ] OS paketi (`libnsl`, `ksh`, `glibc.i686`) instalirani
- [ ] NRE 17 / JDK 17 instaliran
- [ ] RPM-ovi instalirani ispravnim redosledom
- [ ] `authc_configure.sh` odrađen, `administrator` lozinka zabeležena u trezoru
- [ ] `systemctl enable networker` i `gst`
- [ ] NMC dostupan na 9000, NWUI na 9090/nwui
- [ ] Firewall portovi otvoreni
- [ ] SELinux vraćen u `Enforcing`
- [ ] Licenca validirana (`nsrlic -C`)
- [ ] Test backup i restore uspešni

---

### Napomena o izlazima komandi

Prikazani izlazi su reprezentativni primeri iz tipične instalacije (verzije paketa, PID-ovi, veličine i vremena će se razlikovati na vašem sistemu). Struktura izlaza i poruke skripti odgovaraju NetWorker 19.13 dokumentaciji.
