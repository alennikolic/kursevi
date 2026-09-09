# Vraćanje produkcijskog NetWorker-a sa repliciranog Data Domain-a

**Situacija:** nova, sveže instalirana NetWorker mašina sa **istim FQDN-om** kao produkcijski server. Data Domain je **drugi** (vault DD), na njemu je replicirani MTree sa produkcije. Media baza, resource baza i indeksi su prazni.

**Cilj:** napraviti read/write kopiju replike na DD-u, podmetnuti je NetWorker-u kao uređaj, videti šta je unutra i oporaviti produkcijske baze.

Primeri u tekstu koriste:

| | |
|---|---|
| NetWorker (vault) | `nwvault.example.local` — 10.10.20.15 |
| Vault DD | `dd-vault.example.local` — 10.10.20.30 |
| Produkcijski NetWorker | `nwprod.example.local` |
| Replicirani MTree | `/data/col1/nw-prod-repl` |
| Radna kopija (nova) | `/data/col1/nw-recover` |
| DD Boost korisnik | `dduser` |
| Produkcijski uređaj u bootstrap-u | `ddprod.example.local_Backup` |

---

## ⚠ Pročitati pre početka

**Isti FQDN kao produkcija znači da ove dve mašine ne smeju biti na istoj mreži u isto vreme.** Ako produkcijski NetWorker još radi, vault instanca mora biti izolovana. U suprotnom klijenti počinju da pričaju sa pogrešnim serverom.

**Nikada ne labelirati (`nsrmm -l`) volumen sa produkcijskim podacima.** Labeliranje briše sadržaj volumena i podaci postaju neoporavljivi. U celom ovom uputstvu `nsrmm -l` se **ne pojavljuje** — to je namerno.

**Replicirani MTree je read-only** i ne može se koristiti za kreiranje uređaja. Zato pravimo kopiju.

**NMC i NWUI konfigurisati tek POSLE `nsrdr`-a.** `nsrdr` briše `/nsr/nmc` i `/nsr/nwui` jer produkcijski bootstrap ne sadrži te baze. Ako ih podesite pre oporavka, radićete isti posao dvaput.

---

## Redosled celog posla

```
A. Utvrditi layout /nsr na produkciji  ← najčešći uzrok pada nsrdr-a
B. Data Domain: fastcopy → storage unit
C. Priprema NetWorker instance
D. Uređaj i pregled sadržaja
E. nsrdr
F. Popravke posle nsrdr-a (authc, NMC, NWUI)
G. Indeksi i oporavak podataka
```

---

# DEO A — Layout `/nsr` na produkciji

## Zašto je ovo prvi korak

Bootstrap sadrži **apsolutne putanje** sa produkcijskog servera. Ako je tamo `/nsr` bio simbolički link na drugi filesystem (vrlo često — npr. `/data01/nsr`), a na vaultu je običan direktorijum, `nsrdr` pukne ovako:

```
123491:nsrdr: Unable to rename '/nsr/res.R': No such file or directory
123479:nsrdr: Unable to recover the NetWorker bootstrap. Exiting NetWorker server disaster recovery.
```

Poruka je neinformativna. Pravi razlog se vidi tek u logu:

```bash
[root@nwvault ~]# tail -40 /nsr/logs/nsrdr.log
/data01/nsr/res/nsrdb/09/...
/data01/nsr/res/
/data01/nsr/mm/mmvolrel/
/data01/nsr/lockbox/
```

Putanje počinju sa `/data01/nsr` — dakle na produkciji je postojao link.

## Kako saznati unapred

**Najbolje:** pitati nekoga ko poznaje produkcijski server, ili pogledati dokumentaciju/inventar.

**Ako nema koga pitati:** pustite `nsrdr` jednom, pustite ga da padne, pročitajte log. Media baza se u tom prolazu često već oporavi, pa nije izgubljen trud.

## Kako napraviti istu strukturu

```bash
[root@nwvault ~]# systemctl stop networker

[root@nwvault ~]# mkdir -p /data01
[root@nwvault ~]# mv /nsr /data01/nsr
[root@nwvault ~]# ln -s /data01/nsr /nsr

[root@nwvault ~]# ls -ld /nsr /data01/nsr
lrwxrwxrwx.  1 root root   11 Sep  9 23:58 /nsr -> /data01/nsr
drwx------.  9 root root   88 Sep  9 23:58 /data01/nsr
```

## ⚠ Odmah popraviti dozvole

`mv` zadržava restriktivne dozvole, a novi roditeljski direktorijum ih nema uopšte. Servisni nalozi `nsrtomcat` i `nsrnmc` tada ne mogu da prođu kroz putanju:

```
ERROR: User nsrtomcat does not have read permission at path /nsr/authc/conf
```

```bash
[root@nwvault ~]# chmod 755 /data01
[root@nwvault ~]# chmod 755 /data01/nsr

[root@nwvault ~]# ls -ld /data01 /data01/nsr
drwxr-xr-x.  3 root root   17 Sep  9 23:45 /data01
drwxr-xr-x.  9 root root   88 Sep  9 23:58 /data01/nsr
```

Test da servisni nalog stvarno prolazi:

```bash
[root@nwvault ~]# su -s /bin/bash nsrtomcat -c 'ls /nsr/authc/conf' && echo OK
authc.keystore  authc.properties
OK
```

> Ako `/data01` treba da bude zaseban disk (na produkciji verovatno jeste), montirajte ga **pre** `mv`. Indeksi produkcije mogu biti veliki.

---

# DEO B — Rad na Data Domain-u

## B.1 Pregled šta je stiglo replikacijom

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
 RO : Read Only    RW : Read Write    RD : Replication Destination
```

`RD` = replikaciono odredište, read-only. To je izvor.

```
sysadmin@dd-vault# replication status
CTX  Destination                                State        Sync as of Time
---  -----------------------------------------  -----------  --------------------------
  1  mtree://dd-vault.example.local/data/col1/nw-prod-repl  initialized  Wed Sep  9 21:40:12 2026
```

`Sync as of Time` je trenutak do kog su podaci stigli.

## B.2 Fastcopy — read/write kopija

Fastcopy je trenutan i ne troši dodatni prostor.

```
sysadmin@dd-vault# mtree create /data/col1/nw-recover
MTree "/data/col1/nw-recover" created successfully.

sysadmin@dd-vault# filesys fastcopy source /data/col1/nw-prod-repl destination /data/col1/nw-recover
(00:00) Waiting for fastcopy to complete...
Fastcopy status: copied 128432 files, 4821.3 GiB in 41 seconds
```

```
sysadmin@dd-vault# mtree list
/data/col1/nw-recover                 4821.3   RW
```

> **Sa Cyber Recovery:** CR sam kreira sandbox nad zaključanom PIT kopijom. Preskačete fastcopy i koristite sandbox MTree.

## B.3 Registrovati MTree kao storage unit

MTree **nije** automatski i storage unit. Ovo je čest zastoj:

```
sysadmin@dd-vault# ddboost storage-unit show
Storage-units not found
```

`create` neće raditi jer MTree već postoji:

```
sysadmin@dd-vault# ddboost storage-unit create nw-recover user dduser
**** MTree "/data/col1/nw-recover" already exists.
```

**Ispravna komanda je `modify`:**

```
sysadmin@dd-vault# ddboost storage-unit modify nw-recover user dduser

The 'ddboost modify' command should not be used to modify the 'user' of a BOOSTFS storage-unit.
Consider using the 'skip-chown' option when files and directories in the
storage-unit are owned by multiple users.
        Are you sure? (yes|no) [no]: yes

Storage-unit "nw-recover" modified.
```

> Upozorenje se odnosi na BOOSTFS i ovde nije relevantno — odgovorite `yes`.
>
> Ako su fajlovi u vlasništvu UID-a koji se razlikuje od `dduser`, dodajte `skip-chown` da se ne dira vlasništvo produkcijskih podataka:
> ```
> ddboost storage-unit modify nw-recover user dduser skip-chown
> ```

Provera:

```
sysadmin@dd-vault# ddboost storage-unit show
Name         Pre-Comp (GiB)   Status   User     Report Physical Size (MiB)
----------   --------------   ------   ------   --------------------------
nw-recover        1796750.6   RW       dduser                            -
----------   --------------   ------   ------   --------------------------
```

## B.4 DD Boost mora biti uključen

```
sysadmin@dd-vault# ddboost status
DD Boost status: enabled

sysadmin@dd-vault# ddboost user show
DD Boost user   Using Token Access
-------------   ------------------
dduser          -
-------------   ------------------

sysadmin@dd-vault# user show list
Name       Uid    Role
--------   ----   -------------
dduser     500    none
--------   ----   -------------
```

DD Boost korisnik treba da ima **isti UID** kao na produkcijskom DD-u.

## B.5 Saznati imena device foldera

DD Boost uređaj pokazuje na **folder unutar storage unit-a**. Ta imena moramo znati.

Najbrži način je privremeni NFS export. Prvo NFS klijent na RHEL-u:

```bash
[root@nwvault ~]# dnf install -y nfs-utils
```

```
sysadmin@dd-vault# nfs add /data/col1/nw-recover 10.10.20.15
NFS export for "/data/col1/nw-recover" added.
sysadmin@dd-vault# nfs enable
NFS server enabled.
```

```bash
[root@nwvault ~]# mkdir -p /mnt/ddpeek
[root@nwvault ~]# mount -t nfs 10.10.20.30:/data/col1/nw-recover /mnt/ddpeek

[root@nwvault ~]# ls -1 /mnt/ddpeek
Backup

[root@nwvault ~]# ls -la /mnt/ddpeek/Backup | head
total 179
drwxr-xr-x. 103  500 ftp  5424 Sep  1 18:06 .
drwx--x---.  28  500 ftp  1427 Sep  8 14:00 00
drwx--x---.  28  500 ftp  1427 Sep  7 19:00 01
...
drwx--x---.   2  500 ftp   101 Sep  9 10:00 active
-rw-r-----.   1  500 ftp    52 Oct 11  2066 .nsr
-rw-r-----.   1  500 ftp   360 Oct 11  2066 .nsr_serial
-rw-r-----.   1  500 ftp   332 Oct 11  2066 volhdr
```

Ovde je **jedan** device folder: `Backup`. Numerisani podfolderi (`00`–`99`) i `active` su unutrašnja struktura — ne prave se zasebni uređaji za njih.

Ukloniti export:

```bash
[root@nwvault ~]# umount /mnt/ddpeek
```

```
sysadmin@dd-vault# nfs del /data/col1/nw-recover 10.10.20.15
```

---

# DEO C — Priprema NetWorker instance

## C.1 Provera osnovnog

```bash
[root@nwvault ~]# hostname -f
nwvault.example.local

[root@nwvault ~]# rpm -qa | grep -i lgto
lgtoclnt-19.13.0.3-1.x86_64
lgtoxtdclnt-19.13.0.3-1.x86_64
lgtonode-19.13.0.3-1.x86_64
lgtoauthc-19.13.0.3-1.x86_64
lgtoserv-19.13.0.3-1.x86_64

[root@nwvault ~]# df -h /nsr
```

Major verzija mora biti ista kao na produkciji.

## C.2 Rezolucija imena

U vault-u DNS obično ne postoji. Bez ovoga NetWorker startuje sporo ili se zaglavi.

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
10.10.20.15     nwvault.example.local     nwvault
10.10.20.30     dd-vault.example.local    dd-vault

# Klijenti sa produkcije koji ovde nisu dostupni — na loopback
127.0.0.1       app01.example.local   app01
127.0.0.1       sql01.example.local   sql01
```

## C.3 Provera DD Boost konekcije PRE kreiranja uređaja

NFS koji radi ne dokazuje ništa — DD Boost je drugi protokol i drugi port.

```bash
[root@nwvault ~]# nc -zv 10.10.20.30 2051
Ncat: Connected to 10.10.20.30:2051.

[root@nwvault ~]# nc -zv 10.10.20.30 111
Ncat: Connected to 10.10.20.30:111.
```

Ako 2051 ne prolazi, uređaj se neće kreirati — vratite se na B.4.

## C.4 Server state → `disaster recovery`

**Najopasniji korak za preskočiti.** U stanju `active` media baza se **neće uvesti, a `nsrdr` neće prijaviti grešku**.

```bash
[root@nwvault ~]# nsradmin -s $(hostname -f)
NetWorker administration program.
nsradmin> . type: NSR
Current query set
nsradmin> update server state: disaster recovery
                server state: disaster recovery;
Update? y
updated resource id 3.0.255.6.0.0.0.0...
nsradmin> quit
```

> Unutar `nsradmin` prompta kucajte **samo komande** — ne lepite ceo red sa `nsradmin>` ni shell komande poput `printf`.

Provera **iz shell-a**, ne iz prompta:

```bash
[root@nwvault ~]# printf '. type: NSR\nshow server state\nprint\n' | nsradmin -i -
Current query set
                server state: disaster recovery;
```

## C.5 Zaostali DR marker

```bash
[root@nwvault ~]# rm -f /nsr/debug/nsr_disaster_recovery_mode
```

## C.6 Broj niti (opciono, kod mnogo klijenata)

```bash
[root@nwvault ~]# mkdir -p /nsr/debug
[root@nwvault ~]# cat > /nsr/debug/nsrdr.conf <<'EOF'
NSRDR_NUM_THREADS = 10
EOF
```

> Obavezan razmak pre i posle `=`.

---

# DEO D — Uređaj i pregled sadržaja

## D.1 Sintaksa `nsradmin` ulaznog fajla

Tri pravila koja se lako previde:

1. **Vrednosti sa dvotačkom moraju biti pod navodnicima** — inače `nsradmin` javi `Resource parse error: Unterminated resource value list`.
2. **Prazan red pre `EOF`** — njime se resurs zaključuje. Bez njega komanda visi.
3. **`name` je proizvoljno ime uređaja**, ne FQDN servera. Ako tu upišete ime NetWorker-a, `device access information` će pokazivati na pogrešan host i komanda će čekati timeout.

## D.2 Kreiranje uređaja

```bash
[root@nwvault ~]# cat > /tmp/dddev.txt <<'EOF'
create type: NSR device;
name: ddvault_backup;
media type: "Data Domain";
device access information: "10.10.20.30:/nw-recover/Backup";
remote user: "dduser";
password: "lozinka";
auto media management: No;

EOF

[root@nwvault ~]# nsradmin -i /tmp/dddev.txt
created resource id 174.0.49.150.0.0.0.0...

[root@nwvault ~]# shred -u /tmp/dddev.txt
```

> `auto media management: No` sprečava da NetWorker sam labelira nešto što ne prepoznaje.

Provera:

```bash
[root@nwvault ~]# printf '. type: NSR device\nshow name; device access information\nprint\n' | nsradmin -i -
                        name: ddvault_backup;
   device access information: "10.10.20.30:/nw-recover/Backup";
```

## D.3 Mount nije obavezan za `scanner`

Ako pokušate mount, dobićete:

```bash
[root@nwvault ~]# nsrmm -m -f ddvault_backup
6211:nsrmm: Data Domain disk Backup.001 not in media index
```

**To nije greška** — media baza je prazna, pa NetWorker ne prepoznaje volumen koji fizički postoji. Upravo to popravljamo. `scanner` radi i bez mount-a.

U logu se može pojaviti i:

```
Device 'ddvault_backup' datazone id changed from '...' to '...'
```

Takođe normalno: volumen je pisan u produkcijskoj datazoni, a čitate ga iz nove. **Ne pokušavati to "popraviti" labeliranjem.**

> **DD Boost uređaj se ne vidi u `df -h` ni u `mount`.** DD Boost je aplikativni protokol, ne fajl sistem. Provera je isključivo kroz `nsrmm -C` i `nsradmin`.

## D.4 Naći bootstrap SSID

```bash
[root@nwvault ~]# scanner -B ddvault_backup
8909:scanner: using 'ddvault_backup' as the device name
8936:scanner: scanning Data Domain disk Backup.001 on ddvault_backup
8761:scanner: done with Data Domain disk Backup.001

8919:scanner: Bootstrap 228659729 of  9/09/26 10:00:17 located on volume Backup.001, file 0.
```

**Zapisati ssid: `228659729`.**

Popuniti media bazu podacima sa medijuma (čita, ne piše):

```bash
[root@nwvault ~]# scanner -m ddvault_backup

[root@nwvault ~]# mminfo -m
 state volume       written  (%)  expires     read mounts capacity
       Backup.001   1885 TB 100% 09/10/2031   0 KB      0      0 KB
```

Pregled sadržaja pre bilo kakve izmene:

```bash
[root@nwvault ~]# scanner -v ddvault_backup 2>&1 | tee /tmp/scan.txt | head -40
```

> Kod velikih MTree-ova pustite u `screen` ili `tmux`.

---

# DEO E — `nsrdr`

## E.1 Ključna dinamika koju treba razumeti unapred

`nsrdr` u jednom prolazu **zamenjuje resource bazu produkcijskom**. Posledice su trenutne:

| Šta se dešava | Posledica |
|---|---|
| `server state` se vraća na `active` | Mora se ponovo postaviti pre svakog narednog prolaza |
| **Vaš vault uređaj nestaje** | U bazi ostaju samo produkcijski uređaji |
| Pojavljuju se produkcijski pool-ovi | Svaki pool ima listu dozvoljenih uređaja |
| `/nsr/nmc` i `/nsr/nwui` nestaju | NMC i NWUI treba rekonfigurisati |
| `JAVA_HOME` u authc konfiguraciji se gubi | Servis neće startovati dok se ne popravi |

Zato je **realističan tok dvoprolazni**:

```
1. prolaz: kreiraš svoj uređaj → nsrdr → media + resource baza stižu,
           ali uređaj i pool ne odgovaraju jedno drugom
2. popravke: preusmeriš PRODUKCIJSKI uređaj na vault DD
3. prolaz: nsrdr prolazi do kraja
```

## E.2 Prvi prolaz

```bash
[root@nwvault ~]# nsrdr -d ddvault_backup -B 228659729
```

Odgovori:

| Upit | Unos |
|---|---|
| Replace existing resource configuration database folder, res? | `Y` |
| Replace Authentication Service database file? | `Y` |
| Recover client file indexes now? | `Y` (default je `N`!) |
| Recover for all clients — continue? | `Y` |

> Upit o CFI-jevima ima **default `N`**. Ako samo pritisnete Enter, indeksi se preskaču. To nije fatalno — vraćaju se naknadno (Deo G).

Neinteraktivno:

```bash
[root@nwvault ~]# nsrdr -a -B 228659729 -d ddvault_backup -I
```

> Uz `-a` **morate** navesti `-B`, inače alat izađe bez opisne greške. `-I` mora biti **poslednja** opcija.

## E.3 Greška: `pool 'X' cannot be mounted on this device`

Posle prvog prolaza resource baza je produkcijska. Volumen pripada pool-u koji ima listu dozvoljenih uređaja, a u njoj je samo produkcijski uređaj:

```bash
[root@nwvault ~]# printf '. type: NSR pool; name: Backup\nshow name; devices\nprint\n' | nsradmin -i -
                        name: Backup;
                     devices: ddprod.example.local_Backup;
```

**Dodavanje vašeg uređaja u pool NE radi** — uređaj više ne postoji u bazi:

```
update failed: 'ddvault_backup' invalid choice for 'devices'
```

### Rešenje: preusmeriti produkcijski uređaj na vault DD

Time pool ostaje netaknut, jer je taj uređaj već u njegovoj listi.

```bash
[root@nwvault ~]# nsradmin -s $(hostname -f)
nsradmin> . type: NSR device; name: ddprod.example.local_Backup
nsradmin> update device access information: "10.10.20.30:/nw-recover/Backup"
Update? y
nsradmin> update remote user: "dduser"
Update? y
nsradmin> update password: "lozinka"
Update? y
nsradmin> quit
```

Provera:

```bash
[root@nwvault ~]# printf '. type: NSR device; name: ddprod.example.local_Backup\nshow name; device access information; remote user\nprint\n' | nsradmin -i -
                        name: ddprod.example.local_Backup;
   device access information: "10.10.20.30:/nw-recover/Backup";
                 remote user: dduser;
```

## E.4 Onemogućiti uređaje koji pokazuju na nepostojeće DD-ove

Oporavljena baza donosi sve produkcijske uređaje. Oni koji pokazuju na DD-ove kojih u vault-u nema blokiraju `nsrdr`:

```
61506:nsrd: Check operation in progress on ddprod2.example.local_CloneDailyDR
36566:nsrdr: Cannot Unmount volume on ddprod2.example.local_CloneDailyDR
```

Onemogućiti sve osim onog koji ste preusmerili:

```bash
[root@nwvault ~]# nsradmin -s $(hostname -f)
nsradmin> . type: NSR device; name: ddprod2.example.local_CloneDailyDR
nsradmin> update enabled: No
Update? y
nsradmin> . type: NSR device; name: ddprod2.example.local_CloneMonthlyDR
nsradmin> update enabled: No
Update? y
nsradmin> quit
```

Ponoviti za svaki nepotreban uređaj.

## E.5 Greška: `device is marked as suspect`

```
155485:nsrd: device 'ddprod.example.local_Backup' is marked as suspect
36566:nsrdr: Cannot Mount volume on ddprod.example.local_Backup
```

Zaostalo stanje od ranijeg neuspelog pokušaja:

```bash
[root@nwvault ~]# nsrmm -o notsuspect -f ddprod.example.local_Backup
```

## E.6 Drugi prolaz

Server state se vratio na `active` — postaviti ponovo:

```bash
[root@nwvault ~]# nsradmin -s $(hostname -f)
nsradmin> . type: NSR
nsradmin> update server state: disaster recovery
Update? y
nsradmin> quit

[root@nwvault ~]# printf '. type: NSR\nshow server state\nprint\n' | nsradmin -i -
                server state: disaster recovery;
```

```bash
[root@nwvault ~]# nsrdr -d ddprod.example.local_Backup -B 228659729
```

`nsrdr` sada nudi listu uređaja — izaberite preusmereni:

```
Configured devices in the server
1)ddprod.example.local_Backup
2)ddprod.example.local_Clone
...
What is the name of the device you plan to use [ddprod.example.local_Backup][1]? 1
Enter the latest bootstrap save set id [0]: 228659729
```

Na kraju:

```
Starting NetWorker services...
Waiting for NetWorker services to come up. It may take a while.
Waiting for Storage Node to come up.
173680:nsrdr: RPC client handle: Connection refused.
172089:nsrdr: Unable to create the connection with '' to host 'localhost6' with address '::1' at port 7940.
.........
```

> Poruke o `::1` i tačkice su **normalne** — `nsrdr` pokušava IPv6 pre IPv4 dok čeka da se servisi podignu. U drugom terminalu proverite `systemctl status networker`.

---

# DEO F — Popravke posle `nsrdr`-a

## F.1 NetWorker ne startuje: `Unable to read consolidated JAVA_HOME`

```
networker.sh: Error:
    Unable to read consolidated JAVA_HOME from lgtoauthc.  Execute the
    following script as root to configure lgtoauthc:
        /opt/nsr/authc-server/scripts/authc_configure.sh
```

```bash
[root@nwvault ~]# /opt/nsr/authc-server/scripts/authc_configure.sh
```

Java direktorijum je onaj sa vaše mašine — nađite ga sa:

```bash
[root@nwvault ~]# dirname $(dirname $(readlink -f $(which java)))
/usr/lib/jvm/java-17-openjdk-17.0.14.0.7-2.el9.x86_64
```

Na `Do you want to use the existing keystore /nsr/authc/conf/authc.keystore [y]?` odgovorite `y` i unesite postojeću lozinku. Ako je nemate, skripta će dozvoliti novu.

> Ako skripta padne sa `User nsrtomcat does not have read permission at path /nsr/authc/conf` — vratite se na Deo A i popravite dozvole roditeljskih direktorijuma.

Zatim:

```bash
[root@nwvault ~]# systemctl start networker
[root@nwvault ~]# systemctl status networker | head -5
```

## F.2 Provera da je oporavak uspeo

```bash
[root@nwvault ~]# mminfo -m
 state volume              written  (%)  expires     read mounts capacity
       Backup.001          1884 TB 100% 12/10/2027   0 KB     18      0 KB
       Clone.001            515 TB 100% 12/31/2027   0 KB     17      0 KB
       CloneDailyDR.001    2061 TB 100% 01/01/2031   0 KB     17      0 KB
```

```bash
[root@nwvault ~]# mminfo -avot | wc -l
8315

[root@nwvault ~]# mminfo -avot | tail -5
Backup.001  Data Domain  app01.example.local  09/09/2026 03:00:10 AM  2796 GB ...
Backup.001  Data Domain  nwvault.example.local  09/09/2026 10:00:05 AM  4255 KB  index:app01.example.local
```

Ako `mminfo` pokaže produkcijske save set-ove — **oporavak je uspeo**. Ako je prazan, server state je bio `active`.

> Prisustvo `index:` save set-ova je važno: znači da indekse možete vratiti brzim putem (`nsrck -L7`), bez sporog `scanner -i`.

## F.3 NMC ne radi: `could not create database process`

`nsrdr` je obrisao `/nsr/nmc`:

```bash
[root@nwvault ~]# ls -ld /nsr/nmc
ls: cannot access '/nsr/nmc': No such file or directory

[root@nwvault ~]# grep current_db_dir /opt/lgtonmc/etc/gstd.conf
    string current_db_dir = "/nsr/nmc/nmcdb";
```

U `systemctl status gst` nedostaje postgres proces, a log kaže:

```bash
[root@nwvault ~]# tail -30 /opt/lgtonmc/logs/gstd.raw
... Internal error: could not create database process.
... FATAL ERROR: could not stop database process
```

Rešenje — rekonfigurisati NMC:

```bash
[root@nwvault ~]# systemctl stop gst
[root@nwvault ~]# /opt/lgtonmc/bin/nmc_config
```

| Upit | Unos |
|---|---|
| create new(cn) or existing(ue) certificate | `cn` |
| directory for NMC database | Enter (`/nsr/nmc/nmcdb`) |
| migrate data from previous 8.x.x | `n` |
| host name of Authentication Service | Enter |
| start NMC daemons now | `y` |

> Dobijate **praznu** NMC bazu. Prikazivaće sve resurse iz oporavljene NetWorker baze (klijente, uređaje, pool-ove), ali ne i produkcijsku NMC istoriju. Prava NMC baza vraća se komandom `recoverpsm` — zaseban korak.

## F.4 NWUI ne radi

```bash
[root@nwvault ~]# ss -lnt | egrep '9090|9095|5435'
LISTEN 0  200  127.0.0.1:5435  0.0.0.0:*
LISTEN 0  100          *:9095        *:*
LISTEN 0  100          *:9090        *:*

[root@nwvault ~]# ls -ld /nsr/nwui
ls: cannot access '/nsr/nwui': No such file or directory
```

Portovi slušaju, ali konfiguracija je obrisana.

**Prvo osloboditi portove** — NWUI procesi ne gase se uvek sa NetWorker servisom:

```bash
[root@nwvault ~]# systemctl stop networker

[root@nwvault ~]# ss -lntp | egrep '9095|5435'
LISTEN 0  200  127.0.0.1:5435  0.0.0.0:*  users:(("postgres",pid=40060,fd=7))
LISTEN 0  100          *:9095        *:*  users:(("java",pid=40069,fd=24))

[root@nwvault ~]# kill 40069 40060

[root@nwvault ~]# ss -lnt | egrep '9090|9095|5435'
```

Prazan izlaz = spremno.

**Zatim podići NetWorker** (skripta traži živ authc na 9090):

```bash
[root@nwvault ~]# systemctl start networker

[root@nwvault ~]# ss -lnt | grep 9090
LISTEN 4  100  *:9090  *:*
```

**Tek onda konfiguracija:**

```bash
[root@nwvault ~]# /opt/nwui/scripts/nwui_configure.sh
```

| Upit | Unos |
|---|---|
| Java direktorijum | putanja iz F.1 |
| Authentication Service host | Enter |
| NetWorker Server managed by NWUI | Enter |
| AUTHC port | Enter |
| Keystore password | min. 9 karaktera, veliko+malo, broj, specijalni znak |

> Ako se pojavi `WARNING: Unable to contact authentication server ... port 9090` iako port sluša, authc servis ne odgovara ispravno. Prekinuti sa `Ctrl+C` i pogledati:
> ```bash
> tail -30 /nsr/authc/tomcat/logs/catalina.out
> ```
> Ne ponavljati skriptu dok se to ne razjasni — svaki neuspeli prolaz ostavlja zaostale procese na 9095/5435.

Posle uspešne konfiguracije:

```bash
[root@nwvault ~]# systemctl restart networker
```

Pristup: `https://nwvault.example.local:9090/nwui`

---

# DEO G — Indeksi i oporavak podataka

## G.1 Vraćanje client file indeksa

Ako ste preskočili CFI u `nsrdr`-u:

```bash
[root@nwvault ~]# nsrdr -c -I app01.example.local sql01.example.local
```

Ili po klijentu, kada na medijumu postoje `index:` save set-ovi:

```bash
[root@nwvault ~]# nsrck -L7 app01.example.local
completed recovery of index for client 'app01.example.local'
```

Na određeni trenutak u prošlosti:

```bash
[root@nwvault ~]# nsrck -L7 -t "09/08/2026" app01.example.local
```

> `scanner -i` je rezervni put, **samo ako index backup-a nema**. Znatno je sporiji.

## G.2 Retention na `Decade`

Podrazumevana retencija klijentskog resursa NetWorker servera je mesec dana — stariji save set-ovi se odbacuju pri oporavku indeksa.

```bash
[root@nwvault ~]# nsradmin -s $(hostname -f)
nsradmin> . type: NSR client; name: nwvault.example.local
nsradmin> update retention policy: Decade
Update? y
nsradmin> quit
```

## G.3 Test oporavka

```bash
[root@nwvault ~]# mkdir -p /tmp/dr-test
[root@nwvault ~]# recover -s nwvault.example.local -c app01.example.local -d /tmp/dr-test
recover> cd /var/www
recover> add html
7 files marked for recovery
recover> recover
Recovering 7 files into /tmp/dr-test
Received 7 file(s) from NSR server `nwvault.example.local'
recover> quit
```

## G.4 Scan needed zastavica

`nsrdr` je postavio `scan needed` na volumene. Za DD/AFTD to znači da su *recover space* operacije suspendovane — **čitanje radi normalno**.

Ako vam treba samo oporavak podataka, zastavicu možete ostaviti. Uklanja se kroz NMC: **Media → Disk Volumes** → desni klik → **Mark Scan Needed** → **Scan is NOT needed**.

## G.5 Server state

U `disaster recovery` stanju ne rade backup, clone, workflow, index management i media management.

```bash
[root@nwvault ~]# nsradmin -s $(hostname -f)
nsradmin> . type: NSR
nsradmin> update server state: active
Update? y
```

> Ako vault instanca služi **samo za oporavak podataka**, ostavite je u `disaster recovery`. To je zaštita — sprečava slučajno pokretanje backup-a koji bi pisao preko produkcijskih podataka.

## G.6 Restart posle svega

`nsrdr` je isključio popunjavanje DNS keša:

```
202669:nsrdr: NetWorker is running in 'Disaster recovery mode'. Therefore, RAP
consistency check for NetWorker clients and population of DNS cache is disabled.
Restart the NetWorker server.
```

```bash
[root@nwvault ~]# systemctl restart networker
```

---

# Tabela grešaka

| Poruka | Uzrok | Rešenje |
|---|---|---|
| `Unable to rename '/nsr/res.R': No such file or directory` | `/nsr` layout se razlikuje od produkcijskog | Deo A — napraviti isti symlink |
| `User nsrtomcat does not have read permission at path /nsr/authc/conf` | Roditeljski direktorijum je `drwx------` | `chmod 755` na `/data01` i `/data01/nsr` |
| `Unable to read consolidated JAVA_HOME from lgtoauthc` | `nsrdr` je zamenio authc konfiguraciju | Ponovo `authc_configure.sh` |
| `Resource parse error: Unterminated resource value list` | Vrednost sa dvotačkom bez navodnika | Staviti pod `" "` |
| `nsradmin` visi bez izlaza | Nema praznog reda pre `EOF`, ili pogrešan host u `device access information` | Ispraviti fajl |
| `5028-Client creation failed. Cannot connect to address` | DD Boost isključen, port 2051 blokiran, ili storage unit nije registrovan | Deo B.3/B.4, `nc -zv <dd> 2051` |
| `Storage-units not found` iako MTree postoji | MTree nije registrovan kao storage unit | `ddboost storage-unit modify <ime> user <user>` → `yes` |
| `**** MTree "..." already exists` | Pokušaj `create` nad postojećim MTree-om | Koristiti `modify`, ne `create` |
| `not in media index` pri mount-u | Media baza je prazna | Očekivano — `scanner` radi i bez mount-a |
| `pool 'X' cannot be mounted on this device` | Uređaj nije u listi pool-a | Preusmeriti produkcijski uređaj (E.3) |
| `invalid choice for 'devices'` | Uređaj ne postoji u bazi | Isto — E.3 |
| `device is marked as suspect` | Zaostalo od ranijeg pokušaja | `nsrmm -o notsuspect -f <uređaj>` |
| `Check operation in progress on <uređaj>` | Uređaj pokazuje na nepostojeći DD | `update enabled: No` (E.4) |
| `could not create database process` (gstd) | `/nsr/nmc` obrisan | `nmc_config` (F.3) |
| `mount: bad option; ... need a /sbin/mount.<type> helper` | Nema NFS klijenta | `dnf install -y nfs-utils` |
| `unknown command: printf` | Shell komanda kucana unutar `nsradmin` prompta | Prvo `quit` |
| `mminfo: no matches found for the query` posle nsrdr-a | Server state je bio `active` | Postaviti `disaster recovery` i ponoviti |

---

# Vraćanje unazad

`nsrdr` preimenuje stare baze pre zamene:

```bash
[root@nwvault ~]# ls -ld /nsr/res* /nsr/mm* /nsr/index* /nsr/lockbox*
drwxr-xr-x. /nsr/res
drwxr-xr-x. /nsr/res.234620_09092026
drwx------. /nsr/lockbox
drwx------. /nsr/lockbox.234516_09092026
```

```bash
[root@nwvault ~]# systemctl stop networker
[root@nwvault ~]# rm -rf /nsr/res
[root@nwvault ~]# mv /nsr/res.234620_09092026 /nsr/res
[root@nwvault ~]# systemctl start networker
```

Isto po potrebi za `mm`, `index`, `lockbox`.

---

# Šta neće raditi — očekivano, ne greška

| Ponašanje | Zašto |
|---|---|
| `jobsdb` prazan, workflow-ovi u statusu `Never Run` | Server Protection politika ne backup-uje `jobsdb` |
| NMC ne prikazuje produkcijsku istoriju | NMC ima zasebnu bazu; vraća se sa `recoverpsm` |
| Greške u `nsrdr.log` o CFI fizičkih hostova cluster/DAG klijenata | Lažne — NetWorker ne backup-uje fizički host u virtuelnom okruženju |
| `Unable to execute NSR notification using '/bin/mail'` | Nema mail klijenta na vault mašini; bezopasno |

---

# Kartica komandi

## Data Domain

```
mtree list
replication status
mtree create /data/col1/nw-recover
filesys fastcopy source /data/col1/nw-prod-repl destination /data/col1/nw-recover
ddboost storage-unit modify nw-recover user dduser      ← ne "create" ako MTree postoji
ddboost storage-unit show
ddboost status
user show list
nfs add /data/col1/nw-recover <ip>  /  nfs enable  /  nfs del ...
```

## Priprema

```bash
hostname -f
ls -ld /nsr                                  # da li treba symlink
mv /nsr /data01/nsr && ln -s /data01/nsr /nsr
chmod 755 /data01 /data01/nsr
su -s /bin/bash nsrtomcat -c 'ls /nsr/authc/conf'
dnf install -y nfs-utils
nc -zv <dd_ip> 2051
```

## Server state

```bash
nsradmin -s $(hostname -f)
  . type: NSR
  update server state: disaster recovery
printf '. type: NSR\nshow server state\nprint\n' | nsradmin -i -
```

## Uređaj i pregled

```bash
nsradmin -i /tmp/dddev.txt          # navodnici + prazan red pre EOF
scanner -B <device>                 # bootstrap ssid
scanner -m <device>                 # popuni media bazu
scanner -v <device> | tee /tmp/scan.txt
nsrmm -C
nsrmm -o notsuspect -f <device>
```

## Oporavak

```bash
nsrdr -d <device> -B <ssid>
nsrdr -a -B <ssid> -d <device> -I
nsrdr -c -I <klijent1> <klijent2>
/opt/nsr/authc-server/scripts/authc_configure.sh
/opt/lgtonmc/bin/nmc_config
/opt/nwui/scripts/nwui_configure.sh
```

## Provera

```bash
mminfo -m
mminfo -avot | wc -l
mminfo -avot | tail -5
nsr_render_log /nsr/logs/daemon.raw | tail -40
tail -40 /nsr/logs/nsrdr.log
tail -30 /opt/lgtonmc/logs/gstd.raw
tail -30 /nsr/authc/tomcat/logs/catalina.out
```

---

### Napomena o izlazima

Prikazani izlazi su reprezentativni primeri. Imena volumena, ssid-ovi, veličine i timestamp-ovi razlikuju se na svakom sistemu — čitajte strukturu izlaza, ne konkretne brojeve.
