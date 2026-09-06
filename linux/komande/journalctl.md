# Napredne Linux komande — `journalctl` (systemd journal)

## 1. Uvod

`journalctl` je alat za pretragu i prikaz sistemskog dnevnika koji vodi `systemd-journald`.
Za razliku od klasičnih tekstualnih logova u `/var/log/`, journal je **binarni, indeksirani
i strukturirani** format: svaki zapis nosi desetine polja (jedinica, PID, UID, komandna linija,
prioritet, boot ID, cgroup) i po svakom od njih se može filtrirati bez `grep`-a.

Šta journal skuplja:

- standardni izlaz i grešku svih systemd jedinica (`stdout`, `stderr`),
- poruke poslate preko `syslog(3)` i `sd_journal_print()`,
- kernel poruke (ono što se ranije čitalo sa `dmesg`),
- audit poruke,
- rane poruke iz `initrd` faze pokretanja.

Ključna prednost nad `grep`-om po `/var/log`: **strukturirano filtriranje**. Umesto da tražite
tekst i nadate se da nećete uhvatiti pogrešne redove, tražite tačno poruke jedne jedinice,
jednog PID-a i jednog prioriteta, u tačno određenom vremenskom intervalu.

```bash
journalctl --version
```

```
systemd 252 (252.22-1~deb12u1)
+PAM +AUDIT +SELINUX ... +ZSTD +LZ4 ...
```

> Neke opcije (`-S`/`-U` kao skraćenice, `--list-boots` u tabelarnom formatu, `--facility`)
> zavise od verzije systemd-a. Ako opcija ne postoji, proverite verziju.

---

## 2. Dozvole: ko šta sme da vidi

| Situacija | Šta se vidi |
|---|---|
| root ili `sudo` | ceo sistemski journal |
| član grupe `systemd-journal` | ceo sistemski journal |
| član grupe `adm` ili `wheel` | ceo sistemski journal (na većini distribucija) |
| običan korisnik | samo sopstvene poruke (`journalctl --user`) |

Dodavanje korisnika u grupu:

```bash
sudo usermod -aG systemd-journal marko
```

Promena važi tek posle nove prijave.

---

## 3. Trajnost logova — prvo proverite ovo

Podrazumevano na nekim distribucijama journal je **samo u memoriji** (`/run/log/journal`),
što znači da se **briše pri svakom restartu**. Ako `journalctl -b -1` javlja da nema podataka
za prethodni boot, uzrok je ovo.

Provera:

```bash
ls -d /var/log/journal 2>/dev/null && echo "TRAJNO" || echo "SAMO U MEMORIJI"
```

Uključivanje trajnog čuvanja:

```bash
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
```

Ili eksplicitno u `/etc/systemd/journald.conf`:

```ini
[Journal]
Storage=persistent
```

Vrednosti `Storage`: `volatile` (samo RAM), `persistent` (disk, pravi direktorijum sam),
`auto` (disk ako `/var/log/journal` postoji — podrazumevano), `none` (ništa se ne čuva).

---

## 4. Osnovna sintaksa

```
journalctl [OPCIJE] [KRITERIJUMI...]
```

Bez argumenata prikazuje **ceo journal od najstarijeg zapisa**, u pageru:

```bash
journalctl
```

```
-- Journal begins at Mon 2026-08-24 09:12:01 CEST, ends at Sun 2026-09-06 14:31:02 CEST. --
Aug 24 09:12:01 web01 kernel: Linux version 6.1.0-18-amd64 ...
Aug 24 09:12:01 web01 kernel: Command line: BOOT_IMAGE=/vmlinuz-6.1.0-18-amd64 root=UUID=...
...
```

**Format standardnog reda (`-o short`):**

```
Sep 06 10:12:03 web01 nginx[1330]: 10.0.10.7 - - [06/Sep/2026:10:12:03] "GET / HTTP/1.1" 200
└─ vreme        └─ host └─ identifikator[PID]: └─ poruka
```

Vreme je u **lokalnoj zoni sistema**. Za UTC dodajte `--utc`.

---

## 5. Opcije — kompletan pregled

### 5.1 Izbor izvora poruka

| Opcija | Značenje |
|---|---|
| `-u JEDINICA` | poruke jedne systemd jedinice; može se ponoviti (`-u nginx -u php-fpm`) |
| `--user-unit=JEDINICA` | korisnička jedinica |
| `--system` | samo sistemski journal |
| `--user` | samo journal trenutnog korisnika |
| `-k`, `--dmesg` | samo kernel poruke |
| `-t IDENT`, `--identifier=IDENT` | po syslog identifikatoru (`-t sudo`, `-t cron`) |
| `/putanja/do/izvrsne` | poruke procesa pokrenutog sa te putanje (`journalctl /usr/sbin/sshd`) |
| `--facility=LISTA` | po syslog facility (`auth`, `daemon`, `cron`, `mail`, `kern`) |
| `-M MASINA` | journal systemd-nspawn kontejnera |
| `-D DIR`, `--directory=DIR` | čitaj journal iz navedenog direktorijuma |
| `--file=GLOB` | čitaj konkretne journal fajlove |
| `--root=DIR` | čitaj journal iz alternativnog korena (montirani disk) |
| `-m`, `--merge` | spoji sve dostupne journale, uključujući udaljene |

### 5.2 Vremensko ograničenje

| Opcija | Značenje |
|---|---|
| `--since IZRAZ` (`-S`) | od navedenog trenutka |
| `--until IZRAZ` (`-U`) | do navedenog trenutka |
| `-b`, `-b 0` | trenutni boot |
| `-b -1`, `-b -2` | prethodni, pretprethodni boot |
| `-b BOOT_ID` | konkretan boot po ID-u |
| `--list-boots` | spisak svih sačuvanih boot-ova |

Prihvaćeni oblici vremena:

```bash
journalctl --since "2026-09-06 08:00:00"
journalctl --since "2026-09-06"                    # od ponoći tog dana
journalctl --since today
journalctl --since yesterday --until today
journalctl --since "1 hour ago"
journalctl --since "-30min"
journalctl --since "2 days ago" --until "1 day ago"
journalctl --since "09:00" --until "09:15"         # danas
journalctl --since "@1725609600"                   # UNIX timestamp
```

### 5.3 Ograničenje količine i redosled

| Opcija | Značenje |
|---|---|
| `-n N`, `--lines=N` | poslednjih N redova (podrazumevano 10 kada se navede `-n` bez broja) |
| `-e`, `--pager-end` | otvori pager odmah na kraju (najnovije poruke) |
| `-r`, `--reverse` | najnovije poruke prvo |
| `-f`, `--follow` | prati u realnom vremenu (kao `tail -f`) |
| `--no-pager` | ne koristi pager — obavezno za skripte i pipe |
| `-q`, `--quiet` | sakrij informativne poruke i upozorenja |

`-f` podrazumevano prikazuje poslednjih 10 redova pa nastavlja praćenje.
Kombinacija `-n 100 -f` daje širi kontekst pre početka praćenja.

### 5.4 Filtriranje po prioritetu

| Opcija | Značenje |
|---|---|
| `-p NIVO` | poruke navedenog **i višeg** prioriteta |
| `-p OD..DO` | opseg prioriteta |

Nivoi (syslog standard):

| Broj | Ime | Značenje |
|---|---|---|
| 0 | `emerg` | sistem je neupotrebljiv |
| 1 | `alert` | potrebna trenutna intervencija |
| 2 | `crit` | kritični uslov |
| 3 | `err` | greška |
| 4 | `warning` | upozorenje |
| 5 | `notice` | normalan ali značajan događaj |
| 6 | `info` | informativna poruka |
| 7 | `debug` | poruka za otklanjanje grešaka |

`-p err` znači „`err` i sve **ozbiljnije**“ — dakle nivoi 0, 1, 2 i 3.
Za tačno jedan nivo koristite opseg: `-p 3..3`.

### 5.5 Pretraga teksta

| Opcija | Značenje |
|---|---|
| `-g ŠABLON`, `--grep=ŠABLON` | filtriraj poruke po regularnom izrazu (PCRE) |
| `--case-sensitive=yes\|no` | osetljivost na veličinu slova (podrazumevano: pametno) |

`--grep` je **znatno brži** od `\| grep` jer se filtriranje dešava pre formatiranja izlaza,
i radi ispravno sa `-f` (praćenje uživo ne pati od baferisanja koje muči `grep`).

### 5.6 Format izlaza

| Vrednost `-o` | Opis |
|---|---|
| `short` | podrazumevani, syslog-sličan format |
| `short-full` | puni datum sa danom u nedelji i zonom |
| `short-iso` | ISO 8601 vremenska oznaka |
| `short-iso-precise` | ISO 8601 sa mikrosekundama |
| `short-precise` | podrazumevani format sa mikrosekundama |
| `short-monotonic` | sekunde od pokretanja kernela |
| `short-unix` | UNIX timestamp |
| `verbose` | **sva polja** zapisa, čitljivo |
| `export` | binarni serijalizovani format za prenos |
| `json` | jedan JSON objekat po redu |
| `json-pretty` | formatiran JSON |
| `json-seq` | JSON sekvenca (RFC 7464) |
| `cat` | **samo tekst poruke**, bez vremena, hosta i identifikatora |
| `with-unit` | kao `short-full`, ali sa imenom jedinice umesto identifikatora |

Prateće opcije:

| Opcija | Značenje |
|---|---|
| `--output-fields=LISTA` | ograniči polja u `verbose`/`json` izlazu |
| `-a`, `--all` | ne skraćuj duga polja i prikaži nečitljive znakove |
| `-x`, `--catalog` | dodaj objašnjenje poznatih poruka iz kataloga |
| `--no-hostname` | izostavi ime hosta iz izlaza |
| `--utc` | prikaži vremena u UTC |
| `--truncate-newlines` | prikaži samo prvi red višerednih poruka |

### 5.7 Kursori (za skripte i inkrementalno čitanje)

| Opcija | Značenje |
|---|---|
| `--show-cursor` | ispiši kursor poslednjeg prikazanog zapisa |
| `-c KURSOR`, `--cursor=KURSOR` | počni od navedenog kursora |
| `--after-cursor=KURSOR` | počni **posle** navedenog kursora |
| `--cursor-file=FAJL` | automatski čitaj i upisuj kursor u fajl |

### 5.8 Ispitivanje polja

| Opcija | Značenje |
|---|---|
| `-N`, `--fields` | spisak svih imena polja prisutnih u journalu |
| `-F POLJE`, `--field=POLJE` | spisak svih **vrednosti** navedenog polja |
| `--list-catalog` | spisak poruka u katalogu |

### 5.9 Održavanje

| Opcija | Značenje |
|---|---|
| `--disk-usage` | koliko prostora zauzimaju journal fajlovi |
| `--vacuum-size=VELIČINA` | obriši najstarije dok se ne siđe ispod veličine (`1G`, `500M`) |
| `--vacuum-time=VREME` | obriši zapise starije od (`2weeks`, `30d`, `6months`) |
| `--vacuum-files=N` | zadrži najviše N arhivskih fajlova |
| `--rotate` | odmah rotiraj aktivne journal fajlove |
| `--flush` | prebaci poruke iz `/run` u `/var/log/journal` |
| `--sync` | upiši sve baferisane poruke na disk i sačekaj |
| `--verify` | proveri integritet journal fajlova |
| `--relinquish-var` | prestani da pišeš u `/var`, vrati se na `/run` |

---

## 6. Filtriranje po poljima — najmoćnija mogućnost

Svaki zapis u journalu je skup `POLJE=vrednost` parova. Filtriranje se piše direktno:

```bash
journalctl _PID=1330
journalctl _UID=33
journalctl _COMM=sshd
journalctl _SYSTEMD_UNIT=nginx.service
```

### 6.1 Pravila kombinovanja

Ovo je pravilo koje se najčešće pogrešno razume:

| Kombinacija | Logika |
|---|---|
| ista polja, više vrednosti | **ILI** |
| različita polja | **I** |
| grupe razdvojene sa `+` | **ILI** između grupa |

```bash
# ILI: poruke iz nginx-a ili iz php-fpm-a
journalctl _SYSTEMD_UNIT=nginx.service _SYSTEMD_UNIT=php-fpm.service

# I: poruke iz nginx-a KOJE dolaze od PID-a 1330
journalctl _SYSTEMD_UNIT=nginx.service _PID=1330

# (nginx I PID 1330) ILI (sshd bilo koji PID)
journalctl _SYSTEMD_UNIT=nginx.service _PID=1330 + _COMM=sshd
```

### 6.2 Najkorisnija polja

| Polje | Značenje |
|---|---|
| `MESSAGE` | tekst poruke |
| `PRIORITY` | prioritet 0-7 |
| `_PID` | PID procesa koji je poslao poruku |
| `_UID` / `_GID` | vlasnik procesa |
| `_COMM` | ime izvršne datoteke (skraćeno na 15 znakova) |
| `_EXE` | puna putanja izvršne datoteke |
| `_CMDLINE` | cela komandna linija procesa |
| `_SYSTEMD_UNIT` | systemd jedinica |
| `_SYSTEMD_CGROUP` | cgroup putanja — korisno za kontejnere |
| `_SYSTEMD_SLICE` | systemd slice |
| `_HOSTNAME` | ime hosta |
| `_BOOT_ID` | identifikator boot-a |
| `_MACHINE_ID` | trajni identifikator mašine |
| `_TRANSPORT` | odakle je poruka stigla: `kernel`, `syslog`, `journal`, `stdout`, `audit`, `driver` |
| `SYSLOG_IDENTIFIER` | identifikator koji je program sam prijavio |
| `SYSLOG_FACILITY` | syslog facility broj |
| `MESSAGE_ID` | UUID poznatih systemd poruka (za katalog) |
| `CODE_FILE`, `CODE_LINE`, `CODE_FUNC` | mesto u izvornom kodu (za systemd komponente) |
| `_AUDIT_SESSION` | audit sesija |

**Kako otkriti koja polja postoje i koje vrednosti imaju:**

```bash
journalctl -N | head -20
```

```
MESSAGE
PRIORITY
_UID
_GID
_COMM
_EXE
_CMDLINE
_SYSTEMD_UNIT
...
```

```bash
journalctl -F _SYSTEMD_UNIT
```

```
systemd-journald.service
ssh.service
nginx.service
cron.service
mysql.service
...
```

`-F` je odličan način da se vidi **koje jedinice uopšte pišu u journal** pre nego što počnete
da filtrirate.

---

## 7. Praktični scenariji

### 7.1 Servis se ne pokreće

```bash
sudo journalctl -u nginx.service -n 50 --no-pager
```

```
Sep 06 10:12:03 web01 systemd[1]: Starting nginx - high performance web server...
Sep 06 10:12:03 web01 nginx[4501]: nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
Sep 06 10:12:03 web01 nginx[4501]: nginx: configuration file /etc/nginx/nginx.conf test failed
Sep 06 10:12:03 web01 systemd[1]: nginx.service: Control process exited, code=exited, status=1/FAILURE
Sep 06 10:12:03 web01 systemd[1]: nginx.service: Failed with result 'exit-code'.
Sep 06 10:12:03 web01 systemd[1]: Failed to start nginx - high performance web server.
```

**Objašnjenje:** redovi iz `systemd[1]` su okvir (pokušaj pokretanja, rezultat), a redovi iz
`nginx[4501]` su stvarni uzrok — port 80 je zauzet. Dalje: `sudo ss -tulpn '( sport = :80 )'`.

`status=1/FAILURE` je izlazni kod procesa. Tabela najčešćih:

| Kod | Značenje |
|---|---|
| `0/SUCCESS` | uspeh |
| `1/FAILURE` | opšta greška aplikacije |
| `2/INVALIDARGUMENT` | pogrešan argument |
| `200/CHDIR` | ne može da uđe u `WorkingDirectory` |
| `203/EXEC` | **ne može da izvrši `ExecStart`** — pogrešna putanja ili nema `+x` |
| `217/USER` | korisnik iz `User=` ne postoji |
| `226/NAMESPACE` | greška u namespace izolaciji (`ProtectSystem`, `PrivateTmp`) |
| `signal=KILL` | ubijen, najčešće OOM killer ili istekao `TimeoutStopSec` |

### 7.2 Praćenje servisa uživo

```bash
sudo journalctl -u mysql.service -f -n 100
```

Praćenje više jedinica odjednom:

```bash
sudo journalctl -u nginx -u php8.2-fpm -u mysql -f
```

Praćenje sa filtrom teksta:

```bash
sudo journalctl -u myapp -f -g "ERROR|FATAL"
```

`--grep` radi ispravno u `-f` režimu, za razliku od `| grep` koji zbog baferisanja
ume da kasni ili da ništa ne prikaže.

### 7.3 Šta se dešavalo u vreme incidenta

```bash
sudo journalctl --since "2026-09-06 02:55" --until "2026-09-06 03:10" -p warning
```

Ovo je najkorisniji obrazac za analizu incidenta: uzak vremenski prozor, svi servisi,
samo `warning` i ozbiljnije. Bez `-u` filtera se vidi **korelacija** između servisa —
često je pravi uzrok u sasvim drugoj jedinici.

Sa objašnjenjima poznatih poruka:

```bash
sudo journalctl --since "02:55" --until "03:10" -p err -x
```

`-x` dodaje pasus objašnjenja ispod poruka koje imaju `MESSAGE_ID` u katalogu:

```
Sep 06 03:02:41 web01 systemd[1]: mysql.service: Main process exited, code=killed, status=9/KILL
-- Subject: Unit process exited
-- Defined-By: systemd
--
-- An ExecStart= process belonging to unit mysql.service has exited.
--
-- The process' exit code is 'killed' and its exit status is 9.
```

### 7.4 Analiza pada sistema

```bash
journalctl --list-boots
```

```
IDX BOOT ID                          FIRST ENTRY                   LAST ENTRY
 -2 3a1b4c5d6e7f8a9b0c1d2e3f4a5b6c7d Mon 2026-08-24 09:12:01 CEST  Wed 2026-08-26 18:44:11 CEST
 -1 8f7e6d5c4b3a2918273645ab cdef0123 Wed 2026-08-26 18:45:02 CEST  Sat 2026-09-05 22:13:55 CEST
  0 1122334455667788990011223344556 6 Sat 2026-09-05 22:14:31 CEST  Sun 2026-09-06 14:31:02 CEST
```

- `IDX 0` je trenutni boot, `-1` prethodni, i tako unazad.
- Ako se `LAST ENTRY` prethodnog boot-a ne poklapa sa urednim gašenjem, sistem je pao ili je bio prisilno resetovan.

Poslednje poruke pre pada:

```bash
sudo journalctl -b -1 -n 100 --no-pager
sudo journalctl -b -1 -p err
sudo journalctl -b -1 -k                 # kernel poruke prethodnog boot-a
```

Traženje OOM killera:

```bash
sudo journalctl -b -1 -k -g "Out of memory|oom_reaper|Killed process"
```

```
Sep 05 22:11:47 web01 kernel: Out of memory: Killed process 1502 (mysqld) total-vm:8412344kB,
                              anon-rss:6104220kB, file-rss:0kB, shmem-rss:0kB, UID:110 pgtables:12980kB
```

Traženje hardverskih grešaka:

```bash
sudo journalctl -b -1 -k -g "I/O error|Hardware Error|MCE|EDAC|ata[0-9]"
```

### 7.5 Bezbednosna analiza

Neuspešne prijave:

```bash
sudo journalctl -u ssh.service --since "24 hours ago" -g "Failed password"
```

Brojanje po izvornoj IP adresi:

```bash
sudo journalctl -u ssh.service --since "24 hours ago" --no-pager -o cat \
  | grep "Failed password" \
  | grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' \
  | sort | uniq -c | sort -rn | head
```

```
    842 203.0.113.44
    117 198.51.100.9
      3 10.0.10.4
```

`-o cat` daje samo tekst poruke, bez vremena i hosta, što pojednostavljuje dalju obradu.

Upotreba `sudo`:

```bash
sudo journalctl -t sudo --since today
```

```
Sep 06 09:41:12 web01 sudo[4102]: marko : TTY=pts/0 ; PWD=/home/marko ; USER=root ; COMMAND=/usr/bin/systemctl restart nginx
```

Uspešne prijave:

```bash
sudo journalctl -u ssh.service -g "Accepted (password|publickey)" --since "7 days ago"
```

### 7.6 Poruke jednog procesa ili korisnika

```bash
sudo journalctl _PID=1330
sudo journalctl _UID=33 --since today            # www-data
sudo journalctl _COMM=cron --since "1 hour ago"
sudo journalctl /usr/sbin/sshd                    # po putanji izvršne datoteke
sudo journalctl _SYSTEMD_CGROUP=/system.slice/docker.service
```

Napomena: `_COMM` je ograničen na 15 znakova (kernel ograničenje), pa je za duga imena
pouzdaniji `_EXE`.

### 7.7 Kernel poruke

```bash
sudo journalctl -k                          # trenutni boot
sudo journalctl -k -b -1                    # prethodni boot
sudo journalctl -k -p err                   # samo greške
sudo journalctl -k -f                       # praćenje uživo
sudo journalctl -k --since "10 min ago"
```

Prednost nad `dmesg`: `journalctl -k` čuva poruke i posle restarta (ako je storage trajan),
ima prava vremena umesto sekundi od boot-a, i može se filtrirati po prioritetu.

### 7.8 Strukturirani izlaz za obradu

```bash
sudo journalctl -u nginx -n 1 -o verbose
```

```
Sun 2026-09-06 10:12:03.481239 CEST [s=a1b2...;i=4f2a;b=1122...;m=8d2f1a;t=63a1;x=9f2e]
    _BOOT_ID=112233445566778899001122334455 66
    _MACHINE_ID=9f8e7d6c5b4a39281706
    _HOSTNAME=web01
    PRIORITY=6
    SYSLOG_FACILITY=3
    SYSLOG_IDENTIFIER=nginx
    _UID=0
    _GID=0
    _COMM=nginx
    _EXE=/usr/sbin/nginx
    _CMDLINE=nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
    _SYSTEMD_CGROUP=/system.slice/nginx.service
    _SYSTEMD_UNIT=nginx.service
    _SYSTEMD_SLICE=system.slice
    _TRANSPORT=stdout
    _PID=1329
    MESSAGE=Starting nginx - high performance web server...
```

Ovaj prikaz otkriva **sva polja po kojima možete filtrirati**. Kad ne znate kako da suzite
pretragu, pogledajte jedan zapis u `verbose` režimu i uzmite polje odatle.

JSON za mašinsku obradu:

```bash
sudo journalctl -u nginx --since "1 hour ago" -o json --no-pager \
  | jq -r 'select(.PRIORITY|tonumber <= 3) | "\(.__REALTIME_TIMESTAMP) \(.MESSAGE)"'
```

Ograničavanje polja radi brzine:

```bash
sudo journalctl -u nginx -o json --output-fields=MESSAGE,PRIORITY,_PID --no-pager
```

### 7.9 Inkrementalno čitanje u skriptama

Klasičan problem: skripta za nadzor koja svakih 5 minuta gleda nove greške, ali ne sme
da prijavljuje iste dvaput. Rešenje je kursor:

```bash
#!/bin/bash
CURSOR=/var/lib/monitoring/journal.cursor

journalctl --cursor-file="$CURSOR" -p err --no-pager -o cat \
  | while read -r line; do
      echo "NOVA GREŠKA: $line"
    done
```

`--cursor-file` pri prvom pokretanju čita sve, a zatim upisuje poziciju i pri svakom
sledećem pokretanju nastavlja tačno odatle. Nema duplikata ni preskočenih zapisa.

### 7.10 Forenzika sa nepokrenutog sistema

Kad sistem ne diže, montirajte njegov disk sa live medija i čitajte journal direktno:

```bash
sudo mount /dev/sda2 /mnt
sudo journalctl -D /mnt/var/log/journal -b -1 -p err
```

ili:

```bash
sudo journalctl --root=/mnt --list-boots
```

Ovo radi jer je journal samostalan binarni format koji ne zavisi od pokrenutog systemd-a.

### 7.11 Journal kontejnera

```bash
sudo journalctl -M ime-kontejnera            # systemd-nspawn / machinectl
sudo journalctl _SYSTEMD_UNIT=docker.service # sam Docker daemon
sudo journalctl CONTAINER_NAME=web           # ako se koristi journald log driver
```

Za Docker sa `--log-driver=journald` poruke kontejnera dobijaju polja
`CONTAINER_ID`, `CONTAINER_NAME`, `CONTAINER_TAG`.

---

## 8. Održavanje i veličina journala

### 8.1 Provera zauzeća

```bash
journalctl --disk-usage
```

```
Archived and active journals take up 3.8G in the file system.
```

### 8.2 Ručno čišćenje

```bash
sudo journalctl --vacuum-size=1G          # smanji na 1 GB
sudo journalctl --vacuum-time=30d         # obriši starije od 30 dana
sudo journalctl --vacuum-files=10         # zadrži najviše 10 arhivskih fajlova
```

```
Deleted archived journal /var/log/journal/9f8e.../system@a1b2.journal (128.0M).
Deleted archived journal /var/log/journal/9f8e.../system@c3d4.journal (128.0M).
Vacuuming done, freed 256.0M of archive files.
```

> `--vacuum-*` briše **samo arhivske** fajlove, nikada aktivni. Ako aktivni fajl sam zauzima
> previše, prvo pokrenite `journalctl --rotate`, pa onda vacuum.

### 8.3 Trajna konfiguracija

`/etc/systemd/journald.conf`:

```ini
[Journal]
Storage=persistent
Compress=yes
SystemMaxUse=2G                 # gornja granica ukupne veličine
SystemKeepFree=1G               # uvek ostavi bar toliko slobodnog prostora
SystemMaxFileSize=128M          # veličina pojedinačnog fajla
SystemMaxFiles=100              # maksimalan broj fajlova
MaxRetentionSec=1month          # maksimalna starost zapisa
MaxFileSec=1week                # rotacija po vremenu
RateLimitIntervalSec=30s        # prozor za ograničavanje brzine
RateLimitBurst=10000            # maksimalno poruka po servisu u tom prozoru
ForwardToSyslog=no              # isključi ako ne koristite rsyslog
```

Primena:

```bash
sudo systemctl restart systemd-journald
```

Bez eksplicitnih vrednosti journal koristi **10% veličine fajl sistema**, uz obavezno
ostavljenih 15% slobodnog prostora.

### 8.4 Ograničavanje brzine — tiho gubljenje poruka

Ako u logu vidite:

```
Sep 06 11:02:14 web01 systemd-journald[712]: Suppressed 14203 messages from /system.slice/myapp.service
```

servis je premašio `RateLimitBurst` i **poruke su bezpovratno izgubljene**. Rešenja:

- povećati `RateLimitBurst` globalno,
- isključiti ograničenje za jednu jedinicu: `LogRateLimitBurst=0` u njenom `[Service]` odeljku,
- smanjiti nivo logovanja same aplikacije.

Ovo je čest uzrok „nedostajućih“ redova u analizi incidenta.

### 8.5 Provera integriteta

```bash
sudo journalctl --verify
```

```
PASS: /var/log/journal/9f8e.../system.journal
FAIL: /var/log/journal/9f8e.../user-1000@a1b2.journal (Bad message)
```

Za kriptografsko pečaćenje (zaštita od naknadne izmene logova) postavite `Seal=yes`
i generišite ključ sa `journalctl --setup-keys`.

---

## 9. Česte greške

1. **Zaboravljen `--no-pager` u skripti** — komanda čeka na interakciju i skripta se zaglavi.
2. **`-p err` shvaćen kao „samo greške“** — obuhvata i `crit`, `alert`, `emerg`. Za tačan nivo: `-p 3..3`.
3. **Očekivanje logova prethodnog boot-a bez trajnog skladišta** — vidi odeljak 3.
4. **Pogrešno ime jedinice** — `-u nginx` radi jer systemd dopunjuje `.service`, ali `-u nginx.conf` ćutke ne vraća ništa. Proverite sa `journalctl -F _SYSTEMD_UNIT`.
5. **`| grep` umesto `--grep` uz `-f`** — baferisanje ume da zadrži izlaz; `--grep` je i brži.
6. **Mešanje `_COMM` i `SYSLOG_IDENTIFIER`** — nisu isto; `_COMM` je stvarno ime procesa, identifikator prijavljuje sama aplikacija.
7. **`--vacuum-size` bez prethodnog `--rotate`** — aktivni fajl se ne dira, pa oslobođeni prostor može biti manji od očekivanog.
8. **Ignorisanje poruka o `Suppressed N messages`** — deo logova nedostaje.
9. **Vremenska zona** — `--since "09:00"` koristi lokalnu zonu servera; kod korelacije više sistema koristite `--utc`.

---

## 10. Podsetnik (cheat sheet)

```bash
journalctl -u nginx -n 50 --no-pager           # poslednjih 50 redova jedinice
journalctl -u nginx -f                         # praćenje uživo
journalctl -u nginx -f -g "ERROR|FATAL"        # praćenje sa filtrom
journalctl -b -p err -x                        # greške ovog boot-a sa objašnjenjima
journalctl -b -1 -n 200                        # kraj prethodnog boot-a (analiza pada)
journalctl -k -b -1 -g "Out of memory"         # OOM killer
journalctl --list-boots                        # spisak boot-ova
journalctl --since "1 hour ago" -p warning     # poslednji sat, upozorenja i gore
journalctl --since "02:55" --until "03:10"     # uzak prozor oko incidenta
journalctl -t sudo --since today               # upotreba sudo
journalctl _PID=1330                           # jedan proces
journalctl _UID=33 --since today               # jedan korisnik
journalctl /usr/sbin/sshd                      # po putanji izvršne datoteke
journalctl -N                                  # spisak svih polja
journalctl -F _SYSTEMD_UNIT                    # koje jedinice pišu u journal
journalctl -u nginx -n 1 -o verbose            # sva polja jednog zapisa
journalctl -o json --output-fields=MESSAGE,PRIORITY --no-pager   # za jq
journalctl --cursor-file=/var/lib/mon.cur -p err --no-pager      # inkrementalno u skripti
journalctl -D /mnt/var/log/journal -b -1       # journal sa montiranog diska
journalctl --disk-usage                        # zauzeće
journalctl --rotate && journalctl --vacuum-size=1G   # čišćenje
journalctl --verify                            # provera integriteta
```

---

## 11. Povezani alati

| Alat | Kada ga koristiti uz ili umesto `journalctl` |
|---|---|
| `systemctl status JEDINICA` | brz pregled stanja + poslednjih 10 redova loga |
| `systemd-analyze blame` | trajanje pokretanja pojedinačnih jedinica |
| `systemd-analyze critical-chain` | kritični put pri pokretanju sistema |
| `systemd-cat` | slanje proizvoljne komande ili teksta u journal |
| `dmesg -T` | kernel prsten, radi i kad journald nije aktivan |
| `coredumpctl` | pregled i analiza core dump-ova (koristi isti journal) |
| `logger` | slanje poruke u syslog/journal iz skripte |
| `rsyslog` / `syslog-ng` | prosleđivanje logova na centralni server |
| `journal-upload` / `journal-remote` | prenos journala između mašina u nativnom formatu |

Slanje sopstvenih poruka u journal iz skripte:

```bash
echo "Backup završen" | systemd-cat -t backup -p info
logger -t backup -p daemon.info "Backup završen"
```

Obe komande zapisuju poruku koju kasnije nalazite sa `journalctl -t backup`.
