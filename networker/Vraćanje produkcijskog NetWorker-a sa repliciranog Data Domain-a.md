# Vraćanje produkcijskog NetWorker-a sa repliciranog Data Domain-a

**Situacija:** nova, sveže instalirana NetWorker mašina sa **istim FQDN-om** kao produkcijski server. Data Domain je **drugi** (vault DD), na njemu je replicirani MTree sa produkcije. Media baza, resource baza i indeksi su prazni.

**Cilj:** napraviti read/write kopiju replike na DD-u, podmetnuti je NetWorker-u kao uređaj, videti šta je unutra i oporaviti produkcijske baze.

Primeri u tekstu koriste:

| | |
|---|---|
| NetWorker (vault) | `nwvault.example.local` |
| Vault DD | `dd-vault.example.local` |
| Replicirani MTree | `/data/col1/nw-prod-repl` |
| Radna kopija (nova) | `/data/col1/nw-recover` |
| DD Boost korisnik | `ddboost` |

---

## ⚠ Pre nego što išta uradite

**Isti FQDN kao produkcija znači da ove dve mašine ne smeju biti na istoj mreži u isto vreme.** Ako produkcijski NetWorker još radi, vault instanca mora biti u izolovanoj mreži ili iza firewall-a bez rute do produkcijskih klijenata. U suprotnom klijenti počinju da pričaju sa pogrešnim serverom.

**Nikada ne labelirati (`nsrmm -l`) volumen sa produkcijskim podacima.** Labeliranje briše sadržaj volumena i podaci postaju neoporavljivi. U celom ovom uputstvu `nsrmm -l` se **ne pojavljuje** — to je namerno.

**Replicirani MTree je read-only** i ne može se koristiti za kreiranje uređaja. Zato pravimo kopiju.

Brza provera preduslova:

```bash
[root@nwvault ~]# hostname -f
nwvault.example.local

[root@nwvault ~]# rpm -qa | grep -i lgto
lgtoclnt-19.13.0.3-1.x86_64
lgtoxtdclnt-19.13.0.3-1.x86_64
lgtonode-19.13.0.3-1.x86_64
lgtoauthc-19.13.0.3-1.x86_64
lgtoserv-19.13.0.3-1.x86_64
lgtoman-19.13.0.3-1.x86_64

[root@nwvault ~]# df -h /nsr
Filesystem             Size  Used Avail Use% Mounted on
/dev/mapper/rhel-root   70G   23G   48G  33% /

[root@nwvault ~]# ls -ld /nsr
drwxr-xr-x. 12 root root 4096 Sep  9 22:02 /nsr
```

Verzija mora biti ista kao na produkciji (major verzija obavezno). Ako je `/nsr` na produkciji bio simbolički link, mora biti link i ovde.

---

# DEO A — Rad na Data Domain-u

## Korak 1. Pregled šta je stiglo replikacijom

```bash
[root@nwvault ~]# ssh sysadmin@dd-vault.example.local
```

```
sysadmin@dd-vault# mtree list
Name                          Pre-Comp (GiB)   Status
---------------------------   --------------   ------
/data/col1/backup                        0.0   RW
/data/col1/nw-prod-repl               4821.3   RD
---------------------------   --------------   ------
 D : Deleted
 Q : Quota Defined
RO : Read Only
RW : Read Write
RD : Replication Destination
```

`RD` = replikaciono odredište, read-only. To je naš izvor.

Provera da je replikacija u sinhronizaciji:

```
sysadmin@dd-vault# replication status
CTX  Destination                                    State      Sync as of Time
---  ---------------------------------------------  ---------  --------------------------
  1  mtree://dd-vault.example.local/data/col1/nw-prod-repl  initialized  Wed Sep  9 21:40:12 2026
```

> `Sync as of Time` je trenutak do kog su podaci stigli. Sve što je backup-ovano posle toga nije u vault-u.

## Korak 2. Napraviti read/write kopiju (fastcopy)

Fastcopy je trenutna operacija na DD-u — ne kopira blokove, samo pravi novu referencu. Ne troši prostor.

```
sysadmin@dd-vault# mtree create /data/col1/nw-recover
MTree "/data/col1/nw-recover" created successfully.

sysadmin@dd-vault# filesys fastcopy source /data/col1/nw-prod-repl destination /data/col1/nw-recover
(00:00) Waiting for fastcopy to complete...
Fastcopy status: fastcopy /data/col1/nw-prod-repl to /data/col1/nw-recover: copied 128432 files, 4821.3 GiB in 41 seconds
```

Provera:

```
sysadmin@dd-vault# mtree list
Name                          Pre-Comp (GiB)   Status
---------------------------   --------------   ------
/data/col1/backup                        0.0   RW
/data/col1/nw-prod-repl               4821.3   RD
/data/col1/nw-recover                 4821.3   RW
---------------------------   --------------   ------
```

`nw-recover` je `RW` — nad njom smemo da radimo.

> **Ako koristite Cyber Recovery**, ovaj korak radi CR: kreira sandbox nad zaključanom PIT kopijom i eksportuje ga. Tada preskačete fastcopy i koristite sandbox MTree.

## Korak 3. Kreirati storage unit i dodeliti DD Boost korisnika

```
sysadmin@dd-vault# ddboost storage-unit create nw-recover user ddboost
Created storage-unit "nw-recover" for user "ddboost".
```

> Ako je MTree već kreiran fastcopy-jem, umesto `create` koristi se `modify`:
> ```
> sysadmin@dd-vault# ddboost storage-unit modify nw-recover user ddboost
> ```

Provera:

```
sysadmin@dd-vault# ddboost storage-unit show
Name         Pre-Comp (GiB)   Status   User      Report Physical Size (MiB)
----------   --------------   ------   -------   --------------------------
nw-recover          4821.3    RW/Q     ddboost                     982341.2
----------   --------------   ------   -------   --------------------------
```

DD Boost korisnik mora imati **isti UID** kao na produkcijskom DD-u:

```
sysadmin@dd-vault# user show list
Name       Uid    Role
--------   ----   -------------
sysadmin   0      admin
ddboost    500    none
--------   ----   -------------
```

```
sysadmin@dd-vault# ddboost user show
User      Default Tenant-Unit
-------   --------------------
ddboost   -
-------   --------------------
```

Ako se UID razlikuje od produkcije, uskladiti ga sada — kasnije je bolno.

## Korak 4. Saznati imena device foldera

DD Boost uređaj u NetWorker-u pokazuje na **folder unutar storage unit-a**. Ti folderi nose imena uređaja sa produkcije i moramo ih znati.

Najbrži način je privremeni NFS export, samo da pogledamo:

```
sysadmin@dd-vault# nfs add /data/col1/nw-recover 10.10.20.15
NFS export for "/data/col1/nw-recover" added.

sysadmin@dd-vault# nfs enable
NFS server enabled.
```

Sa NetWorker mašine:

```bash
[root@nwvault ~]# mkdir -p /mnt/ddpeek
[root@nwvault ~]# mount -t nfs dd-vault.example.local:/data/col1/nw-recover /mnt/ddpeek

[root@nwvault ~]# ls -1 /mnt/ddpeek
dd-prod_dev01
dd-prod_dev02
dd-prod_dev03

[root@nwvault ~]# ls /mnt/ddpeek/dd-prod_dev01 | head
.nsr
nwprod.example.local.001
nwprod.example.local.002
nwprod.example.local.003
```

Sada znamo: tri uređaja, volumeni se zovu `nwprod.example.local.00X`.

Odmontirati i ukloniti export — dalje idemo preko DD Boost-a:

```bash
[root@nwvault ~]# umount /mnt/ddpeek
```

```
sysadmin@dd-vault# nfs del /data/col1/nw-recover 10.10.20.15
NFS export for "/data/col1/nw-recover" deleted.
```

> Alternativa bez NFS-a: u NMC-u **Devices → New Device Wizard → Data Domain**, uneti DD host i kredencijale, i čarobnjak sam izlista postojeće foldere u storage unit-u. Tada je **obavezno isključiti "Label and Mount"**.

---

# DEO B — Priprema NetWorker instance

## Korak 5. Rezolucija imena

U vault-u DNS obično ne postoji. Bez ovoga NetWorker startuje sporo ili se zaglavi čekajući rezoluciju svakog klijenta.

```bash
[root@nwvault ~]# vi /etc/nsswitch.conf
```

```
# bilo:  hosts: files dns myhostname
hosts: files
```

```bash
[root@nwvault ~]# vi /etc/hosts
```

```
127.0.0.1       localhost localhost.localdomain
10.10.20.15      nwvault.example.local   nwvault
10.10.20.30      dd-vault.example.local      dd-vault

# Klijenti sa produkcije koji ovde nisu dostupni — na loopback,
# da server ne čeka timeout pri startu
127.0.0.1       app01.example.local   app01
127.0.0.1       sql01.example.local   sql01
127.0.0.1       file01.example.local  file01
```

Provera:

```bash
[root@nwvault ~]# getent hosts nwvault.example.local
10.10.20.15      nwvault.example.local nwvault
```

## Korak 6. Postaviti server state na `disaster recovery`

**Ovo je najopasniji korak za preskočiti.** Ako server ostane u stanju `active`, media baza se **neće uvesti, a `nsrdr` neće prijaviti grešku**. Dobijete naizgled uspešan oporavak sa praznom media bazom.

```bash
[root@nwvault ~]# systemctl status networker | head -3
● networker.service - EMC NetWorker. A backup and restoration software package.
     Loaded: loaded (/usr/lib/systemd/system/networker.service; enabled)
     Active: active (running) since Wed 2026-09-09 22:02:27 CEST

[root@nwvault ~]# nsradmin -s nwvault.example.local
NetWorker administration program.
Use the "help" command for help, "visual" for full-screen mode.
nsradmin> . type: NSR
Current query set
nsradmin> show name; server state
nsradmin> print
                        name: nwvault.example.local;
                server state: active;
nsradmin> update server state: disaster recovery
                server state: disaster recovery;
Update? y
updated resource id 4.0.35.12.0.0.0.0.89.187.161.106.10.10.20.15(6)
nsradmin> quit
```

Potvrda:

```bash
[root@nwvault ~]# echo -e ". type: NSR\nshow server state\nprint" | nsradmin -i -
                server state: disaster recovery;
```

## Korak 7. Provera zaostalog DR marker fajla

```bash
[root@nwvault ~]# ls -l /nsr/debug/nsr_disaster_recovery_mode
ls: cannot access '/nsr/debug/nsr_disaster_recovery_mode': No such file or directory
```

Ako fajl postoji od ranijeg neuspelog pokušaja — obrisati ga:

```bash
[root@nwvault ~]# rm -f /nsr/debug/nsr_disaster_recovery_mode
```

---

# DEO C — Podmetanje uređaja

## Korak 8. Kreirati DD Boost uređaje

Za svaki folder iz koraka 4 pravimo po jedan uređaj. **Bez labeliranja.**

```bash
[root@nwvault ~]# cat > /tmp/dddev.txt <<'EOF'
create type: NSR device;
name: dd-vault_dev01;
media type: Data Domain;
device access information: dd-vault.example.local:/nw-recover/dd-prod_dev01;
remote user: ddboost;
password: <ddboost_lozinka>;
auto media management: No;
EOF

[root@nwvault ~]# nsradmin -i /tmp/dddev.txt
created resource id 174.0.49.150.0.0.0.0.89.187.161.106.10.10.20.15(2)
```

Ponoviti za `dev02` i `dev03`.

> **`auto media management: No` je bitno** — sprečava da NetWorker sam labelira nešto što ne prepoznaje.
>
> Obrisati fajl sa lozinkom kad završite: `shred -u /tmp/dddev.txt`

Provera:

```bash
[root@nwvault ~]# nsradmin -s nwvault.example.local
nsradmin> . type: NSR device
Current query set
nsradmin> show name; device access information; enabled
nsradmin> print
                        name: dd-vault_dev01;
   device access information: dd-vault.example.local:/nw-recover/dd-prod_dev01;
                     enabled: Yes;

                        name: dd-vault_dev02;
   device access information: dd-vault.example.local:/nw-recover/dd-prod_dev02;
                     enabled: Yes;
nsradmin> quit
```

## Korak 9. Montirati volumene

Volumeni već imaju labelu sa produkcije. Mount samo čita tu labelu — ništa ne piše.

```bash
[root@nwvault ~]# nsrmm -m -f dd-vault_dev01
Data Domain disk nwprod.example.local.001 mounted on dd-vault_dev01, write enabled
```

Ako se pojavi upit tipa *"volume is not in the media database"* — to je očekivano, media baza je prazna. Potvrditi.

```bash
[root@nwvault ~]# nsrmm -C
32916:nsrmm: Data Domain disk nwprod.example.local.001 mounted on dd-vault_dev01, write enabled
32916:nsrmm: Data Domain disk nwprod.example.local.002 mounted on dd-vault_dev02, write enabled
32916:nsrmm: Data Domain disk nwprod.example.local.003 mounted on dd-vault_dev03, write enabled
```

> **DD Boost uređaj se ne vidi u `df -h` ni u `mount`.** DD Boost je aplikativni protokol, ne fajl sistem. Provera je isključivo kroz `nsrmm -C` i `nsradmin`.

---

# DEO D — Šta ima na medijumu

## Korak 10. Pregled sadržaja volumena

```bash
[root@nwvault ~]# scanner -v dd-vault_dev01 | head -30
scanner: scanning Data Domain disk dd-vault_dev01
scanner: ssid 3841029447: scanning session at Sep  8 22:00:14 2026
nwprod.example.local: index:app01.example.local  level=full   size=  412 MB  files=1  B
nwprod.example.local: index:sql01.example.local  level=full   size=  198 MB  files=1  BE
nwprod.example.local: bootstrap                      level=full   size=  1.2 GB  files=1  BE
app01.example.local:  /var/www                       level=full   size= 24 GB   files=88214 BE
sql01.example.local:  D:\\SQLBACKUP                  level=full   size=310 GB   files=142   BE
```

Zastavice na kraju reda:

| Zastavica | Značenje |
|---|---|
| `B` | Begin — početak save set-a |
| `C` | Continue — save set je počeo na prethodnom volumenu |
| `S` | Synchronize — tačka od koje se izdvajanje može nastaviti |
| `E` | End — kraj save set-a |

`BE` u istom redu = ceo save set je na ovom volumenu.

Ovo je trenutak kada vidite **šta stvarno imate** pre nego što išta menjate.

## Korak 11. Naći bootstrap SSID

```bash
[root@nwvault ~]# scanner -B dd-vault_dev01
scanner: scanning Data Domain disk dd-vault_dev01 for bootstrap save sets

  date       time      level  ssid        file  record  volume
  09/06/26   22:00:41  full   3839112204  0     0       nwprod.example.local.001
  09/07/26   22:00:33  9      3840078119  0     0       nwprod.example.local.001
  09/08/26   22:00:52  9      3841029447  0     0       nwprod.example.local.001

scanner: 3 bootstrap save sets found
```

**Poslednji red je najnoviji bootstrap.** Zapisati:

- `ssid` = `3841029447` (četvrta kolona)
- `file` = `0`, `record` = `0` (peta i šesta — relevantno samo kod trake)
- volumen = `nwprod.example.local.001`

> Ako je bootstrap na drugom uređaju, ponoviti `scanner -B` za `dev02` i `dev03`.
>
> Ako `scanner -B` ništa ne nađe, bootstrap-i su možda u zasebnom uređaju/folderu koji nije replicran, ili je backup Server Protection politike padao na produkciji.

Provera vremena — da li je ovo bootstrap iz očekivanog perioda:

```bash
[root@nwvault ~]# date -d @$(date +%s) '+%Y-%m-%d'
2026-09-09
```

Bootstrap je od 08.09. — dan pre. Replikacija je bila u sinhronizaciji do 21:40 danas, što se poklapa.

---

# DEO E — Oporavak baza (`nsrdr`)

## Korak 12. Podešavanje broja niti (opciono)

Kod većeg broja klijenata podrazumevanih 5 niti je usko grlo:

```bash
[root@nwvault ~]# mkdir -p /nsr/debug
[root@nwvault ~]# cat > /nsr/debug/nsrdr.conf <<'EOF'
NSRDR_NUM_THREADS = 10
EOF
```

> Obavezan razmak pre i posle `=`. Ako editor doda `.txt` na ime fajla — ukloniti.

## Korak 13. Pokrenuti `nsrdr`

```bash
[root@nwvault ~]# nsrdr
```

Interaktivni tok (unos označen sa `-->`):

```
NetWorker Disaster Recovery Wizard

Enter the device name that contains the bootstrap save set:
                                                --> dd-vault_dev01

Enter the bootstrap save set id (ssid):         --> 3841029447

Enter the starting file number (default 0):     --> <Enter>
Enter the starting record number (default 0):   --> <Enter>

Please mount the volume 'nwprod.example.local.001' in device
'dd-vault_dev01'. Volume is already mounted. Continuing...

Recovering the NetWorker media database and resource files...
  /nsr/res
  /nsr/mm
  /nsr/authc

Do you want to keep the existing NetWorker resource files in /nsr/res
or replace them with the recovered files?
Replace the existing resource files? [y/n]      --> y

Stopping NetWorker services...
Renaming /nsr/res to /nsr/res.1757455812
Moving recovered resources into place...

Do you want to replace the existing NetWorker Authentication Service
database file, authcdb.h2.db, with the recovered database file? [y/n]
                                                --> y

Restarting NetWorker services...

Do you want to recover the client file indexes now? [y/n]  --> y
Recover indexes for all clients? [y/n]                     --> y

completed recovery of index for client 'nwprod.example.local'
completed recovery of index for client 'app01.example.local'
completed recovery of index for client 'sql01.example.local'
completed recovery of index for client 'file01.example.local'

nsrdr: Disaster recovery completed successfully.
```

**Neinteraktivna varijanta** (kada znate ssid i uređaj):

```bash
[root@nwvault ~]# nsrdr -a -B 3841029447 -d dd-vault_dev01 -I
```

> Uz `-a` **morate** navesti `-B`, inače alat izađe bez opisne greške. Opcija `-I` mora biti **poslednja**.

## Korak 14. Ponovo pokrenuti authc konfiguraciju

```bash
[root@nwvault ~]# /opt/nsr/authc-server/scripts/authc_configure.sh
```

Odgovori isti kao pri instalaciji (Java putanja, port 9090, keystore lozinka).

---

# DEO F — Provera i sređivanje

## Korak 15. Šta je stiglo

```bash
[root@nwvault ~]# mminfo | head
 volume                         client                    date       size    level  name
 nwprod.example.local.001   app01.example.local   09/08/2026  24 GB  full   /var/www
 nwprod.example.local.001   sql01.example.local   09/08/2026 310 GB  full   D:\SQLBACKUP
 nwprod.example.local.002   file01.example.local  09/08/2026 1.4 TB  full   /export/share
...

[root@nwvault ~]# mminfo -m
 state volume                         written  (%)  expires      read mounts capacity
       nwprod.example.local.001   4.8 TB   72%  09/08/2027   0 KB      1     0 KB
       nwprod.example.local.002   3.1 TB   48%  09/08/2027   0 KB      1     0 KB
```

Media baza je puna — oporavak je uspeo. **Ako je `mminfo` prazan, zaboravili ste korak 6 (server state).**

Klijenti:

```bash
[root@nwvault ~]# echo -e ". type: NSR client\nshow name\nprint" | nsradmin -i - | head -20
                        name: nwprod.example.local;
                        name: app01.example.local;
                        name: sql01.example.local;
                        name: file01.example.local;
```

## Korak 16. Ukloniti stare device resurse sa produkcijskog DD-a

Oporavljena resource baza donosi i **stare uređaje koji pokazuju na produkcijski DD** — taj DD ovde ne postoji. Ostavljeni, generisaće greške pri svakom pokušaju montiranja.

```bash
[root@nwvault ~]# echo -e ". type: NSR device\nshow name; device access information\nprint" | nsradmin -i -
                        name: dd-prod_dev01;
   device access information: dd-prod.example.local:/nw-prod/dd-prod_dev01;

                        name: dd-vault_dev01;
   device access information: dd-vault.example.local:/nw-recover/dd-prod_dev01;
```

Onemogućiti stare (ne brisati odmah — lakše je vratiti):

```bash
[root@nwvault ~]# nsradmin -s nwvault.example.local
nsradmin> . type: NSR device; name: dd-prod_dev01
nsradmin> update enabled: No
                     enabled: No;
Update? y
updated resource id 174.0.49.150.0.0.0.0.89.187.161.106.10.10.20.15(5)
nsradmin> quit
```

## Korak 17. Provera aliasa NetWorker klijentskog resursa

```bash
[root@nwvault ~]# echo -e ". type: NSR client; name: nwvault.example.local\nshow aliases\nprint" | nsradmin -i -
                     aliases: nwvault.example.local, nwvault;
```

Mora sadržati i shortname i FQDN. Ako nedostaje, dopuniti kroz `nsradmin update aliases:`.

## Korak 18. Retention na `Decade`

Podrazumevana retencija klijentskog resursa NetWorker servera je mesec dana. Bez izmene, save set-ovi stariji od toga se odbacuju pri oporavku indeksa.

```bash
[root@nwvault ~]# nsradmin -s nwvault.example.local
nsradmin> . type: NSR client; name: nwvault.example.local
nsradmin> update retention policy: Decade
             retention policy: Decade;
Update? y
nsradmin> quit
```

## Korak 19. Scan needed zastavica

`nsrdr` je postavio `scan needed` na volumene. Za DD/AFTD to znači da su *recover space* operacije suspendovane — čitanje radi normalno.

**Ako vam treba samo oporavak podataka, zastavicu možete ostaviti.** Uklanja se tek kada okruženje ide u normalan rad.

Ako sumnjate da su podaci pisani posle bootstrap-a (08.09. 22:00), skenirati:

```bash
# Preporučeni put — kada na medijumu postoje index backup-i
[root@nwvault ~]# scanner -m dd-vault_dev01
[root@nwvault ~]# nsrck -L7 -t "09/08/2026" app01.example.local

# Rezervni put — kada index backup-a nema (sporo!)
[root@nwvault ~]# scanner -i dd-vault_dev01
```

Uklanjanje zastavice kroz NMC: **Media → Disk Volumes** → desni klik na volumen → **Mark Scan Needed** → **Scan is NOT needed**.

## Korak 20. Test oporavka podataka

Sada radi ono zbog čega je sve ovo rađeno:

```bash
[root@nwvault ~]# mkdir -p /tmp/dr-test
[root@nwvault ~]# recover -s nwvault.example.local \
                              -c app01.example.local \
                              -d /tmp/dr-test
Current working directory is /
recover> cd /var/www
recover> ls
html/  cgi-bin/  logs/
recover> add html
7 files marked for recovery
recover> recover
Recovering 7 files into /tmp/dr-test
Requesting 1 file(s), this may take a while...
./var/www/html/index.php
./var/www/html/config.php
...
Received 7 file(s) from NSR server `nwvault.example.local'
Recover completion time: Wed Sep  9 23:41:07 2026
recover> quit

[root@nwvault ~]# ls -R /tmp/dr-test/var/www/html
index.php  config.php  ...
```

Ako ovo prođe — oporavak je funkcionalno potvrđen.

## Korak 21. Vratiti server state (samo kada završite oporavke)

U stanju `disaster recovery` **ne rade** backup, clone, workflow, index management i media management.

```bash
[root@nwvault ~]# nsradmin -s nwvault.example.local
nsradmin> . type: NSR
nsradmin> update server state: active
                server state: active;
Update? y
nsradmin> quit
```

> Ako vault instanca služi **samo za oporavak podataka**, ostavite je u `disaster recovery` stanju. To je zaštita — sprečava slučajno pokretanje backup-a koji bi pisao preko produkcijskih podataka.

---

# DEO G — Kada nešto pođe naopako

## Vraćanje na stanje pre `nsrdr`-a

`nsrdr` je preimenovao stare baze pre nego što ih je zamenio:

```bash
[root@nwvault ~]# ls -ld /nsr/res* /nsr/mm* /nsr/index*
drwxr-xr-x. 3 root root 4096 Sep  9 23:10 /nsr/index
drwxr-xr-x. 2 root root 4096 Sep  9 23:10 /nsr/mm
drwxr-xr-x. 5 root root 4096 Sep  9 23:10 /nsr/res
drwxr-xr-x. 5 root root 4096 Sep  9 22:02 /nsr/res.1757455812
```

```bash
[root@nwvault ~]# systemctl stop networker
[root@nwvault ~]# rm -rf /nsr/res
[root@nwvault ~]# mv /nsr/res.1757455812 /nsr/res
[root@nwvault ~]# systemctl start networker
```

Isto po potrebi za `/nsr/mm.<timestamp>` i `/nsr/index.<timestamp>`.

## Šta neće raditi — očekivano, ne greška

| Ponašanje | Zašto |
|---|---|
| `jobsdb` je prazan, svi workflow-ovi u statusu `Never Run` | Server Protection politika ne backup-uje `jobsdb` |
| NMC ne prikazuje sve resurse | NMC ima zasebnu bazu; `nsrdr` je ne dira. Rešava se komandom `recoverpsm` |
| Greške u `nsrdr.log` o CFI oporavku fizičkih hostova cluster/DAG klijenata | Lažne — NetWorker ne backup-uje fizički host u virtuelnom okruženju. Ignorisati |

## Dijagnostika

```bash
[root@nwvault ~]# cat /nsr/logs/nsrdr.log | tail -40
[root@nwvault ~]# nsr_render_log /nsr/logs/daemon.raw | tail -40
[root@nwvault ~]# nsrwatch
```

## Za Dell Support pripremiti

```bash
[root@nwvault ~]# nsr_render_log /nsr/logs/daemon.raw > /tmp/daemon.txt
[root@nwvault ~]# cp /nsr/logs/nsrdr.log /tmp/
[root@nwvault ~]# scanner -B dd-vault_dev01 > /tmp/scanner-B.txt 2>&1
[root@nwvault ~]# nsrmm -C > /tmp/nsrmm-C.txt
[root@nwvault ~]# rpm -qa | grep -i lgto > /tmp/paketi.txt
[root@nwvault ~]# ls -ld /nsr/res* /nsr/mm* /nsr/index* > /tmp/baze.txt
```

Plus: tačna komanda koja je pokrenuta, verzija DDOS-a i izlaz `ddboost storage-unit show`.

---

# Kartica komandi

## Data Domain

```
mtree list
replication status
mtree create /data/col1/nw-recover
filesys fastcopy source /data/col1/nw-prod-repl destination /data/col1/nw-recover
ddboost storage-unit create nw-recover user ddboost
ddboost storage-unit show
user show list
```

## NetWorker — priprema

```bash
hostname -f
vi /etc/nsswitch.conf              # hosts: files
vi /etc/hosts                      # nepoznati klijenti → 127.0.0.1
nsradmin -s <server>               # . type: NSR → update server state: disaster recovery
rm -f /nsr/debug/nsr_disaster_recovery_mode
```

## NetWorker — uređaj

```bash
nsradmin -i /tmp/dddev.txt         # create type: NSR device
nsrmm -m -f dd-vault_dev01         # mount (NIKAD nsrmm -l !)
nsrmm -C                           # šta je montirano
```

## NetWorker — pregled i oporavak

```bash
scanner -v dd-vault_dev01          # tabela sadržaja
scanner -B dd-vault_dev01          # bootstrap ssid (4. kolona)
nsrdr                              # interaktivno
nsrdr -a -B <ssid> -d <device> -I  # neinteraktivno
/opt/nsr/authc-server/scripts/authc_configure.sh
```

## NetWorker — posle

```bash
mminfo
mminfo -m
scanner -m <device> ; nsrck -L7 -t "<datum>" <klijent>
recover -s <server> -c <klijent> -d /tmp/dr-test
nsradmin -s <server>               # update server state: active
```

---

### Napomena o izlazima

Prikazani izlazi su reprezentativni primeri. Imena volumena, ssid-ovi, veličine i timestamp-ovi razlikuju se na svakom sistemu — čitajte strukturu izlaza, ne konkretne brojeve.
