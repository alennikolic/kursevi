# Napredne Linux komande — `lsof` (LiSt Open Files)

## 1. Uvod

U Linuxu je **sve fajl**: obični fajlovi, direktorijumi, mrežni soketi, pipe-ovi, uređaji,
deljene biblioteke, `eventfd`/`epoll` objekti. `lsof` prikazuje **sve otvorene fajlove svih procesa**
i, obrnuto, sve procese koji drže određeni fajl.

Zbog toga je `lsof` univerzalni dijagnostički alat koji odgovara na pitanja koja drugi alati ne mogu:

- Zašto `umount` kaže „device is busy“?
- `df` kaže da je disk pun, a `du` ne nalazi ništa — gde je prostor?
- Koji proces drži port 8080?
- Koji proces piše u ovaj log fajl?
- Koje sve biblioteke koristi ovaj proces i da li su neke obrisane posle nadogradnje?

Instalacija: `apt install lsof` (Debian/Ubuntu), `dnf install lsof` (RHEL/Rocky/Fedora).
Na minimalnim instalacijama i u kontejnerima često **nije** prisutan.

```bash
lsof -v
```

```
lsof version information:
    revision: 4.95.0
    latest revision: https://github.com/lsof-org/lsof
    ...
```

> `lsof` bez `sudo` prikazuje samo fajlove procesa koje pokreće vaš korisnik. Za sistemsku
> dijagnostiku gotovo uvek je potreban root.

---

## 2. Sintaksa i najvažnije pravilo

```
lsof [OPCIJE] [IMENA_FAJLOVA...]
```

**Kritično pravilo koje najviše zbunjuje:** kada navedete više kriterijuma različitog tipa,
`lsof` ih podrazumevano kombinuje kao **ILI (OR)**, ne kao I (AND).

```bash
lsof -u marko -i :443
```

Ovo znači: „fajlovi korisnika marko **ILI** bilo koji soket na portu 443“ — dakle prikazaće
i tuđe sokete na 443 i sve fajlove korisnika marko.

Za logičko **I (AND)** dodajte `-a`:

```bash
lsof -a -u marko -i :443
```

Sada: „soketi na portu 443 **koji pripadaju** korisniku marko“.

Kriterijumi istog tipa (npr. dva `-u`) su uvek OR međusobno; `-a` deluje između različitih tipova.

Negacija se piše sa `^`:

```bash
lsof -u ^root        # svi osim root-a
lsof -p ^1,^2        # svi procesi osim PID 1 i 2
```

---

## 3. Kolone izlaza

```bash
sudo lsof -p 1330
```

```
COMMAND  PID     USER   FD      TYPE DEVICE SIZE/OFF   NODE NAME
nginx   1330 www-data  cwd       DIR  253,0     4096      2 /
nginx   1330 www-data  rtd       DIR  253,0     4096      2 /
nginx   1330 www-data  txt       REG  253,0  1310224 262541 /usr/sbin/nginx
nginx   1330 www-data  mem       REG  253,0  2029592 131204 /usr/lib/x86_64-linux-gnu/libc.so.6
nginx   1330 www-data  DEL       REG  253,0           131199 /usr/lib/x86_64-linux-gnu/libssl.so.3
nginx   1330 www-data    0r      CHR    1,3      0t0      6 /dev/null
nginx   1330 www-data    1w      CHR    1,3      0t0      6 /dev/null
nginx   1330 www-data    2w      REG  253,0    18432 393230 /var/log/nginx/error.log
nginx   1330 www-data    6u     IPv4  28901      0t0    TCP *:80 (LISTEN)
nginx   1330 www-data    8u     unix 0x0000...      0t0  31002 /run/nginx.sock type=STREAM
nginx   1330 www-data    9u  a_inode   0,14        0   1035 [eventpoll]
nginx   1330 www-data   11r     FIFO   0,13      0t0  31904 pipe
```

| Kolona | Značenje |
|---|---|
| `COMMAND` | ime procesa, podrazumevano skraćeno na 9 znakova (`+c 0` za puno ime) |
| `PID` | ID procesa |
| `USER` | vlasnik procesa (login ime; `-l` prikazuje numerički UID) |
| `FD` | file descriptor ili tip specijalne reference — **najinformativnija kolona**, vidi 3.1 |
| `TYPE` | tip fajla, vidi 3.2 |
| `DEVICE` | broj uređaja `major,minor` (za obične fajlove) ili adresa kernel strukture (za sokete) |
| `SIZE/OFF` | veličina fajla u bajtovima **ili** trenutni offset u formatu `0tBROJ` |
| `NODE` | inode broj fajla; kod soketa je protokol (`TCP`, `UDP`) |
| `NAME` | putanja, mrežna adresa ili opis objekta |

### 3.1 Vrednosti kolone `FD`

Kolona `FD` je ili **broj deskriptora + režim pristupa**, ili **simbolička oznaka**.

Simboličke oznake:

| Oznaka | Značenje |
|---|---|
| `cwd` | trenutni radni direktorijum procesa |
| `rtd` | root direktorijum (`/`, ili chroot putanja) |
| `txt` | izvršni kod programa (program text) |
| `mem` | memorijski mapiran fajl (obično deljena biblioteka) |
| `mmap` | memorijski mapiran uređaj |
| `DEL` | **memorijski mapiran fajl koji je obrisan sa diska** |
| `ltx` | tekst deljene biblioteke |
| `pd` | roditeljski direktorijum |
| `err` | greška pri čitanju informacija o deskriptoru |
| `NOFD` | `/proc/PID/fd` nije mogao da se otvori (proces je nestao ili nemate prava) |
| `Lnn` | biblioteka sa rednim brojem nn |
| `tr` | kernel trace fajl |

Numeričke oznake su broj deskriptora sa sufiksom režima:

| Sufiks | Režim |
|---|---|
| `r` | otvoren samo za čitanje |
| `w` | otvoren samo za pisanje |
| `u` | otvoren za čitanje i pisanje |
| ` ` (razmak) | režim nepoznat, nema zaključavanja |
| `-` | režim nepoznat, ima zaključavanje |

Iza režima može stajati i **znak zaključavanja (lock)**:

| Znak | Značenje |
|---|---|
| `N` | Solaris NFS lock nepoznatog tipa |
| `r` / `R` | read lock na delu fajla / na celom fajlu |
| `w` / `W` | write lock na delu fajla / na celom fajlu |
| `u` | read i write lock bilo koje dužine |
| `x` / `X` | SCO OpenServer Xenix lock |

Standardni deskriptori: `0` = stdin, `1` = stdout, `2` = stderr.
U gornjem primeru `2w REG ... /var/log/nginx/error.log` znači da nginx piše standardnu grešku
direktno u log fajl.

### 3.2 Vrednosti kolone `TYPE`

| Vrednost | Značenje |
|---|---|
| `REG` | običan fajl |
| `DIR` | direktorijum |
| `CHR` | karakter uređaj (`/dev/null`, `/dev/pts/0`) |
| `BLK` | blok uređaj (`/dev/sda`) |
| `FIFO` | imenovani ili anonimni pipe |
| `PIPE` | pipe (na nekim platformama umesto FIFO) |
| `unix` | UNIX domain soket |
| `IPv4` / `IPv6` | mrežni soket |
| `sock` | soket čiji tip nije mogao da se odredi |
| `netlink` | netlink soket (komunikacija sa kernelom) |
| `a_inode` | anonimni inode: `epoll`, `eventfd`, `inotify`, `signalfd`, `timerfd` |
| `LINK` | simbolički link |
| `PSXSEM` / `PSXSHM` | POSIX semafor / deljena memorija |
| `unknown` | tip nije prepoznat |

---

## 4. Opcije — kompletan pregled

### 4.1 Izbor procesa

| Opcija | Značenje |
|---|---|
| `-p PID` | procesi po PID-u; lista razdvojena zarezom, `^` za isključivanje |
| `-c IME` | procesi čije ime **počinje** navedenim nizom |
| `-c /regex/i` | procesi čije ime odgovara regularnom izrazu (`i` = ignoriši veličinu slova) |
| `+c W` | širina kolone COMMAND; `+c 0` znači neograničeno |
| `-u KORISNIK` | procesi korisnika (ime ili UID); `-u ^root` isključuje |
| `-g PGID` | procesi po ID-u grupe procesa; ujedno prikazuje kolonu PGID |
| `-R` | dodaj kolonu `PPID` (roditeljski proces) |
| `-K` | prikaži i pojedinačne niti (task) procesa, dodaje kolonu `TID` |

### 4.2 Izbor fajlova i direktorijuma

| Opcija | Značenje |
|---|---|
| `IME_FAJLA` | fajl naveden kao argument (bez opcije) |
| `+d DIR` | fajlovi otvoreni u direktorijumu, **jedan nivo dubine** |
| `+D DIR` | fajlovi otvoreni u direktorijumu, **rekurzivno** (sporo na velikim stablima) |
| `-d LISTA` | izbor po deskriptoru: `-d 1,2`, `-d 0-3`, `-d cwd`, `-d ^txt` |
| `+L` | prikaži kolonu `NLINK` (broj hard linkova) |
| `+L1` | **prikaži samo fajlove sa manje od 1 linka** — tj. obrisane a još otvorene |
| `-L` | ne prikazuj broj linkova (podrazumevano) |
| `-N` | samo NFS fajlovi |
| `-U` | samo UNIX domain soketi |

### 4.3 Mrežni soketi

| Opcija | Značenje |
|---|---|
| `-i` | svi mrežni soketi |
| `-i 4` / `-i 6` | samo IPv4 / samo IPv6 |
| `-i TCP` / `-i UDP` | po protokolu |
| `-i :80` | po portu (bilo koji protokol) |
| `-i TCP:80` | protokol + port |
| `-i TCP:1-1024` | opseg portova |
| `-i @10.0.0.5` | po adresi (lokalnoj ili udaljenoj) |
| `-i @host:port` | adresa i port zajedno |
| `-i TCP@10.0.0.5:22` | puna specifikacija |
| `-s TCP:LISTEN` | filtriranje po **stanju** protokola |
| `-s TCP:^ESTABLISHED` | negacija stanja |
| `-T q` | prikaži veličine redova (queue) soketa |
| `-T s` | prikaži stanje veze |
| `-T f` | prikaži zastavice soketa |
| `-M` | prikaži portmapper registracije |

### 4.4 Formatiranje izlaza

| Opcija | Značenje |
|---|---|
| `-n` | ne razrešavaj IP adrese u host imena (**bitno za brzinu**) |
| `-P` | ne razrešavaj brojeve portova u imena servisa |
| `-l` | prikaži numerički UID umesto login imena |
| `-t` | **terse**: ispiši samo PID-ove, bez zaglavlja — za skripte |
| `-F LISTA` | mašinski čitljiv izlaz po poljima; `-F0` koristi NUL separator |
| `-o` | prikaži offset umesto veličine |
| `-o o` | broj cifara offseta |
| `-h` | pomoć |
| `-v` | verzija i parametri kompajliranja |
| `-Z` | prikaži SELinux kontekst |
| `+f g` / `+f G` | prikaži zastavice fajla (imenima / heksadecimalno) |

### 4.5 Ponašanje i performanse

| Opcija | Značenje |
|---|---|
| `-b` | izbegavaj kernel pozive koji mogu da blokiraju (`stat`, `lstat`, `readlink`) |
| `-e PUTANJA` | izuzmi putanju od blokirajućih poziva (npr. montirani NFS) |
| `-S VREME` | timeout u sekundama za kernel pozive (podrazumevano 15) |
| `-w` | ne prikazuj upozorenja |
| `+w` | prikazuj upozorenja |
| `-r [SEK]` | **repeat mode** — ponavljaj svakih SEK sekundi (podrazumevano 15) dok se ne prekine |
| `+r [SEK]` | ponavljaj dok izlaz ne postane prazan, pa izađi |
| `-a` | kombinuj sve izbore logičkim I |

---

## 5. Primeri sa objašnjenjem

### 5.1 Ko koristi port

```bash
sudo lsof -i :22 -n -P
```

```
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
sshd    1044 root    3u  IPv4  21053      0t0  TCP *:22 (LISTEN)
sshd    1044 root    4u  IPv6  21055      0t0  TCP *:22 (LISTEN)
sshd    2210 root    4u  IPv4  38104      0t0  TCP 10.0.10.15:22->10.0.10.4:51422 (ESTABLISHED)
sshd    2214 marko   4u  IPv4  38104      0t0  TCP 10.0.10.15:22->10.0.10.4:51422 (ESTABLISHED)
```

**Objašnjenje:**

- `-n -P` sprečava DNS i `/etc/services` pretragu — bez njih komanda ume da traje sekundama.
- Prva dva reda su isti servis na IPv4 i IPv6 stogu, dva odvojena deskriptora (`3u` i `4u`).
- Poslednja dva reda pokazuju **istu vezu iz dva procesa**: `sshd` posle prijave forkuje
  privilegovani i neprivilegovani proces koji dele deskriptor. Isti `NODE` (38104) to potvrđuje.
- Format `lokalna->udaljena` u koloni NAME i stanje veze u zagradi.

### 5.2 Samo servisi koji slušaju

```bash
sudo lsof -nP -iTCP -sTCP:LISTEN
```

```
COMMAND  PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
systemd-  712 systemd-resolve 13u IPv4 20114 0t0  TCP 127.0.0.53:53 (LISTEN)
sshd     1044     root    3u  IPv4  21053      0t0  TCP *:22 (LISTEN)
nginx    1329     root    6u  IPv4  28901      0t0  TCP *:80 (LISTEN)
nginx    1330 www-data    6u  IPv4  28901      0t0  TCP *:80 (LISTEN)
mysqld   1502    mysql   21u  IPv4  30188      0t0  TCP 127.0.0.1:3306 (LISTEN)
```

`-sTCP:LISTEN` je ključ — bez njega dobijate i sve uspostavljene veze.
`nginx` se pojavljuje dvaput jer master (root) i worker (www-data) dele isti listening soket.

### 5.3 Sve što drži jedan proces

```bash
sudo lsof -p 1502 +c 0
```

`+c 0` uklanja skraćivanje imena na 9 znakova, što je važno kod dugih imena
(`systemd-resolve`, `NetworkManager`, `containerd-shim`).

Za brojanje otvorenih deskriptora procesa:

```bash
sudo lsof -p 1502 | wc -l
```

Uporedite sa limitom:

```bash
cat /proc/1502/limits | grep "open files"
```

```
Max open files            1024                 4096                 files
```

Ako je broj otvorenih deskriptora blizu soft limita, aplikacija će uskoro dobiti
`EMFILE: Too many open files`.

### 5.4 Ko drži fajl ili direktorijum

```bash
sudo lsof /var/log/syslog
```

```
COMMAND    PID    USER   FD   TYPE DEVICE SIZE/OFF   NODE NAME
rsyslogd   793 syslog    7w   REG  253,0   482103 393221 /var/log/syslog
```

Za direktorijum i sve ispod njega:

```bash
sudo lsof +D /var/lib/mysql
```

> `+D` prolazi kroz celo stablo i može biti veoma spor na direktorijumima sa milionima fajlova.
> Ako vam treba samo brz odgovor „ko drži mountpoint“, koristite `lsof /mnt/data` ili `+d`.

### 5.5 Fajlovi korisnika

```bash
sudo lsof -u marko
sudo lsof -u marko -a -i          # samo mrežni soketi korisnika marko
sudo lsof -u ^root -a -i -nP      # svi mrežni soketi osim root-ovih
```

### 5.6 Po imenu procesa

```bash
sudo lsof -c nginx
sudo lsof -c /^postgres/i         # regularni izraz
sudo lsof -c ssh -c nginx         # OR: i ssh* i nginx*
```

`-c nginx` hvata **prefiks**, pa će uhvatiti i `nginx-debug`.

---

## 6. Praktični scenariji za administratore

### 6.1 `df` pokazuje pun disk, `du` ne nalazi ništa

Najčešći produkcijski problem: neko je obrisao veliki log fajl dok ga je proces još držao otvorenim.
Direktorijumski unos je nestao, ali kernel neće osloboditi blokove dok je deskriptor otvoren.

```bash
sudo lsof +L1
```

```
COMMAND    PID   USER   FD   TYPE DEVICE   SIZE/OFF NLINK   NODE NAME
mysqld    1502  mysql   12u   REG  253,0 8589934592     0 393244 /var/lib/mysql/slow.log (deleted)
java      3391 tomcat   87w   REG  253,0 4294967296     0 393301 /opt/app/logs/app.log (deleted)
```

**Objašnjenje:**

- `+L1` znači „prikaži fajlove čiji je broj linkova manji od 1“ — dakle obrisane a otvorene.
- Kolona `NLINK` = 0 potvrđuje da fajl više nema ime u fajl sistemu.
- `(deleted)` u koloni NAME je isti podatak drugim rečima.
- `SIZE/OFF` pokazuje koliko prostora se i dalje troši — ovde 8 GB + 4 GB.

**Rešenja, od najmanje do najviše nasilnog:**

```bash
# 1. Najčistije: reci aplikaciji da ponovo otvori logove
sudo systemctl reload mysql
sudo kill -HUP 793                 # rsyslog i slični

# 2. Ako aplikacija ne podržava reload — isprazni fajl kroz živi deskriptor
sudo truncate -s 0 /proc/1502/fd/12

# 3. Poslednja opcija: restart procesa
sudo systemctl restart mysql
```

Varijanta 2 je trik koji vredi znati: `/proc/PID/fd/N` je i dalje validna putanja do fajla
koji više nema ime, pa se prostor oslobađa bez prekida servisa.

Ograničavanje pretrage na jedan fajl sistem:

```bash
sudo lsof -a +L1 /var
```

### 6.2 `umount: target is busy`

```bash
sudo umount /mnt/backup
```

```
umount: /mnt/backup: target is busy.
```

```bash
sudo lsof /mnt/backup
```

```
COMMAND   PID   USER   FD   TYPE DEVICE SIZE/OFF   NODE NAME
bash     4102  marko  cwd    DIR   8,17     4096      2 /mnt/backup
rsync    4488   root    3r    REG   8,17 10485760  14022 /mnt/backup/db.tar.gz
```

**Objašnjenje:** `bash` proces uopšte nema otvoren fajl — samo mu je **radni direktorijum** tamo
(`FD` = `cwd`). To je dovoljno da mount bude zauzet. Rešenje je da korisnik izađe iz direktorijuma
(`cd /`), a `rsync` da završi ili da se prekine.

Alternativa koja radi bez `lsof`:

```bash
sudo fuser -vm /mnt/backup
```

Prisilno odmontiranje ako ništa drugo ne pomaže (rizik od gubitka podataka):

```bash
sudo umount -l /mnt/backup      # lazy: odvezuje odmah, čisti kad se oslobodi
```

### 6.3 Zastarele biblioteke posle nadogradnje

Posle `apt upgrade` procesi i dalje koriste stare, obrisane verzije biblioteka u memoriji.
Dok se ne restartuju, bezbednosna zakrpa **nije primenjena**.

```bash
sudo lsof -n | grep -E 'DEL.*lib' | awk '{print $1, $2}' | sort -u
```

```
mysqld 1502
nginx 1329
nginx 1330
sshd 1044
```

Kolona `FD` = `DEL` znači memorijski mapiran fajl koji više ne postoji na disku.
Ovi procesi zahtevaju restart.

Na Debianu/Ubuntu isti posao radi i `needrestart`, na RHEL-u `needs-restarting -r`.

### 6.4 Proces koji „curi“ deskriptore

```bash
watch -n5 "sudo lsof -p 3391 | wc -l"
```

Ako broj stalno raste i nikad ne pada, aplikacija ne zatvara deskriptore.
Da vidite **šta** curi, grupišite po tipu:

```bash
sudo lsof -p 3391 | awk '{print $5}' | sort | uniq -c | sort -rn
```

```
   8412 IPv4
    203 REG
     44 a_inode
     12 DIR
```

8412 otvorenih IPv4 soketa uz mali broj aktivnih veza znači da aplikacija ne zove `close()`
posle završetka veze — isti simptom koji se u `ss` vidi kao gomila `CLOSE-WAIT` soketa.

### 6.5 Prekid procesa koji drži resurs

`-t` daje samo PID-ove, pogodne za prosleđivanje:

```bash
sudo lsof -t /mnt/backup
```

```
4102
4488
```

```bash
sudo kill $(sudo lsof -t /mnt/backup)
```

> **Oprez:** uvek prvo pokrenite komandu **bez** `kill` i pogledajte šta bi bilo pogođeno.
> `lsof -t /` bi vratio praktično sve procese na sistemu.

### 6.6 Praćenje u realnom vremenu

```bash
sudo lsof -r2 -nP -i :443
```

`-r2` ponavlja pretragu svake 2 sekunde i razdvaja iteracije linijom `=======`.
Korisno kad pratite da li se veze uspostavljaju i zatvaraju kako treba.

```bash
sudo lsof +r5 -t /mnt/backup && echo "Mount je slobodan" && sudo umount /mnt/backup
```

`+r` ponavlja **dok izlaz ne postane prazan**, pa automatski izlazi — idealno za čekanje
da se resurs oslobodi u skripti.

### 6.7 Veličine redova i stanja soketa

```bash
sudo lsof -nP -i TCP -a -p 1330 -Tqs
```

```
COMMAND  PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nginx   1330 www-data    6u  IPv4  28901      0t0  TCP *:80 (LISTEN)
nginx   1330 www-data   14u  IPv4  40122      0t0  TCP 10.0.10.15:80->10.0.10.7:60112 (ESTABLISHED)
```

Uz `-Tqs` uz svaki soket ide i `QR=` (receive queue), `QS=` (send queue) i `ST=` (stanje).
Ovo je `lsof` ekvivalent onoga što `ss` prikazuje kolonama `Recv-Q`/`Send-Q`.

### 6.8 UNIX domain soketi

```bash
sudo lsof -U | grep docker
```

```
dockerd 1102 root 9u unix 0x00000000abc 0t0 28714 /run/docker.sock type=STREAM
docker  5501 marko 3u unix 0x00000000def 0t0 41002 type=STREAM
```

Prvi red je server koji sluša na soketu; drugi je klijent koji je povezan (bez imena putanje,
jer klijentska strana nije vezana za ime u fajl sistemu).

### 6.9 Skriptovanje sa `-F`

```bash
sudo lsof -nP -i :80 -F pcn
```

```
p1329
cnginx
f6
n*:80
p1330
cnginx
f6
n*:80
```

Svaka linija počinje slovom polja: `p` = PID, `c` = command, `f` = FD, `n` = name,
`u` = UID, `L` = login, `t` = type, `s` = size, `a` = access mode.
Ovaj format je stabilan između verzija i pravi izbor za parsiranje, za razliku od
kolonskog izlaza čija se poravnanja menjaju.

Za NUL-separator (bezbedno kod imena sa razmacima):

```bash
sudo lsof -nP -i :80 -F0pcn
```

---

## 7. Performanse i zaglavljivanje

`lsof` prolazi kroz `/proc` za svaki proces i radi `stat()` nad svakim deskriptorom.
Na sistemu sa montiranim NFS-om koji ne odgovara, ta `stat()` može da blokira i `lsof` „visi“.

Rešenja:

```bash
lsof -b -w                     # izbegavaj blokirajuće pozive, sakrij upozorenja
lsof -e /mnt/nfs               # izuzmi konkretnu putanju
lsof -S 5                      # skrati timeout na 5 sekundi
lsof -n -P                     # ukloni DNS i /etc/services kašnjenje
```

Kombinacija za „bezbedan“ `lsof` na problematičnom sistemu:

```bash
sudo lsof -bwnP -S 5
```

Uz `-b` neki podaci nedostaju (npr. tačne putanje), ali komanda se sigurno završava.

---

## 8. Česte greške

1. **Zaboravljen `-a`** — `lsof -u marko -i :443` daje uniju umesto preseka.
2. **Bez `-n -P`** — komanda traje sekundama ili minutima zbog reverse-DNS upita.
3. **Bez `sudo`** — vidite samo svoje procese i pogrešno zaključujete da port nije zauzet.
4. **`+D` na velikom stablu** — komanda traje predugo; koristite `+d` ili ime mountpointa.
5. **Skraćeno ime u `COMMAND`** — `systemd-` može biti bilo šta; koristite `+c 0`.
6. **Parsiranje kolonskog izlaza** — kolona `NAME` sadrži razmake i strelice; koristite `-F`.
7. **`kill $(lsof -t ...)` bez provere** — može oboriti pola sistema.
8. **`lsof` u kontejneru** vidi samo procese svog PID namespace-a; sa hosta koristite `nsenter`.
9. **Očekivanje da `lsof` prikaže obrisane fajlove bez `+L1`** — DEL/deleted redovi postoje,
   ali `+L1` je pouzdan način da ih izolujete.

---

## 9. Podsetnik (cheat sheet)

```bash
lsof -nP -i :8080                       # ko drži port
lsof -nP -iTCP -sTCP:LISTEN             # svi servisi koji slušaju
lsof -p 1234                            # sve što drži proces
lsof -p 1234 | wc -l                    # broj otvorenih deskriptora
lsof /var/log/syslog                    # ko drži fajl
lsof +D /mnt/data                       # ko drži bilo šta u direktorijumu (rekurzivno)
lsof /mnt/data                          # ko drži mountpoint (brzo)
lsof +L1                                # obrisani a otvoreni fajlovi — "nestali" prostor
lsof -u marko -a -i                     # mrežni soketi jednog korisnika
lsof -c nginx                           # sve što drži nginx
lsof -t /mnt/data                        # samo PID-ovi
lsof -i @10.0.0.5                       # sve veze ka/od te adrese
lsof -U                                 # UNIX domain soketi
lsof -r2 -nP -i :443                    # ponavljaj svake 2 sekunde
lsof -n | grep -E 'DEL.*lib'            # procesi sa zastarelim bibliotekama
lsof -nP -i :80 -F pcn                  # mašinski čitljiv izlaz
lsof -bwnP -S 5                         # bezbedan režim na sistemu sa NFS-om
```

---

## 10. Povezani alati

| Alat | Kada ga koristiti umesto `lsof` |
|---|---|
| `fuser -vm /mnt` | brža i jednostavnija provera „ko drži mount“; ima `-k` za ubijanje |
| `ss -tulpn` | znatno brži za čisto mrežnu dijagnostiku |
| `ls -l /proc/PID/fd/` | isto što i `lsof -p PID`, bez instalacije dodatnog paketa |
| `find /proc/*/fd -lname '/putanja*'` | pronalaženje držaoca fajla bez `lsof` |
| `pmap -x PID` | detaljan pregled memorijskih mapiranja jednog procesa |
| `inotifywait` | praćenje događaja nad fajlom, a ne trenutnog stanja |
| `nsenter -t PID -m -n lsof` | pokretanje `lsof` unutar namespace-a kontejnera |
