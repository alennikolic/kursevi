# Napredne Linux komande — `rsync`

## 1. Uvod

`rsync` sinhronizuje fajlove i direktorijume — lokalno, preko SSH ili preko sopstvenog demona.
Ključna razlika u odnosu na `cp` i `scp` je u tome što `rsync` **prenosi samo razliku**:
poredi izvor i odredište, preskače fajlove koji su isti, a za izmenjene fajlove prenosi
samo blokove koji su se promenili.

Kada koristiti šta:

| Zadatak | Alat |
|---|---|
| jednokratna kopija nekoliko fajlova | `cp`, `scp` |
| pakovanje u arhivu | `tar` |
| **ponovljena sinhronizacija, bekap, migracija, ogledalo** | `rsync` |
| replikacija na nivou bloka / celog uređaja | `dd`, `LVM snapshot`, `DRBD` |
| sinhronizacija ka objektnom skladištu | `rclone`, `aws s3 sync` |

Tri osobine koje ga čine standardom za administratore:

- **prekid se nastavlja** — ponovno pokretanje ne kreće ispočetka,
- **čuva metapodatke** — dozvole, vlasništvo, vremena, ACL, xattr, hard linkove,
- **idempotentan je** — isto pokretanje dvaput daje isti rezultat, drugi put gotovo trenutno.

```bash
rsync --version | head -2
```

```
rsync  version 3.2.7  protocol version 31
Copyright (C) 1996-2022 by Andrew Tridgell, Wayne Davison, and others.
```

Verzija je bitna. `rsync` 3.2.x donosi `zstd` kompresiju, `xxhash` kontrolne sume i
`--mkpath`; sve to nedostaje na starijim sistemima. **Verzija na obe strane utiče na rezultat** —
prenos radi po najnižem zajedničkom protokolu.

---

## 2. Načini rada

```
rsync [OPCIJE] IZVOR... ODREDIŠTE
```

| Oblik | Primer | Transport |
|---|---|---|
| lokalno → lokalno | `rsync -a /src/ /dst/` | direktno, bez mreže |
| lokalno → udaljeno | `rsync -a /src/ user@host:/dst/` | SSH |
| udaljeno → lokalno | `rsync -a user@host:/src/ /dst/` | SSH |
| preko demona | `rsync -a /src/ rsync://host/modul/` | rsync protokol, port 873 |
| preko demona kroz SSH | `rsync -a /src/ host::modul/` | dvostruka dvotačka |

**Udaljeno na udaljeno nije podržano.** Za to je potrebno da jedna strana bude posrednik
(`ssh host1 "rsync ... host2:"`), pri čemu ključ mora biti dostupan na prvoj mašini.

---

## 3. Završna kosa crta — pravilo koje se najčešće promaši

Kosa crta na kraju **izvora** menja značenje komande. Na odredištu nema efekta.

```bash
rsync -a /podaci/  /backup/      # sadržaj /podaci ide u /backup
rsync -a /podaci   /backup/      # nastaje /backup/podaci
```

Mnemotehnika: `izvor/` znači „sadržaj ovog direktorijuma“, `izvor` znači „ovaj direktorijum“.

Ponovljeno pokretanje pogrešne varijante pravi `/backup/podaci/podaci/`, a u kombinaciji
sa `--delete` može obrisati sve što je ranije preneto ispravno.

**Provera pre svakog rizičnog pokretanja:**

```bash
rsync -avn --delete /podaci/ /backup/ | head -20
```

---

## 4. Kako `rsync` odlučuje šta da prenese

### 4.1 Brza provera (podrazumevano)

Fajl se **preskače** ako se poklapaju **veličina i vreme izmene (mtime)**.
Sadržaj se ne čita. Zato je ponovljeno pokretanje na nepromenjenom stablu vrlo brzo.

Posledice:

- fajl izmenjen tako da mu se veličina i mtime nisu promenili **neće** biti prenet,
- fajl kome je samo `touch` promenio mtime **biće** ponovo prenet (delta algoritmom, pa je jeftino),
- razlika u vremenskoj zoni ili preciznosti između fajl sistema ume da izazove nepotrebne prenose — vidi `--modify-window`.

### 4.2 Delta algoritam

Kada fajl postoji na obe strane i razlikuje se, `rsync` ne prenosi ceo fajl:

1. Odredište deli svoju kopiju na blokove i za svaki računa slabu i jaku kontrolnu sumu.
2. Sume šalje pošiljaocu.
3. Pošiljalac kliza prozor kroz svoju verziju i nalazi blokove koji se poklapaju.
4. Prenosi se **samo ono što se ne poklapa**, uz uputstva kako da se fajl sastavi.

Rezultat se vidi u `--stats` kao odnos `Literal data` (stvarno preneto) prema
`Matched data` (ponovo iskorišćeno sa odredišta).

> **Delta algoritam ne pomaže za nove fajlove** — oni se prenose celi.
> Takođe, kada su **oba puta lokalna**, `rsync` podrazumevano koristi `--whole-file`,
> jer je čitanje lokalnog diska brže od računanja suma.

### 4.3 Načini poređenja

| Opcija | Ponašanje |
|---|---|
| (podrazumevano) | veličina + mtime |
| `-c`, `--checksum` | računa kontrolnu sumu **celog** fajla na obe strane; sporo, ali pouzdano |
| `--size-only` | samo veličina; korisno kad su vremena nepouzdana (posle `cp` bez `-p`) |
| `--ignore-times` | prenesi sve, bez obzira na poređenje |
| `-u`, `--update` | preskoči fajlove koji su na odredištu **noviji** |
| `--ignore-existing` | prenesi samo fajlove kojih na odredištu nema |
| `--existing` | ažuriraj samo postojeće, ne kreiraj nove |
| `--modify-window=N` | tretiraj razliku u mtime do N sekundi kao jednakost (FAT/exFAT: `-—modify-window=1`) |

---

## 5. Opcije

### 5.1 Osnovne

| Opcija | Duga forma | Značenje |
|---|---|---|
| `-a` | `--archive` | zbirna opcija: `-rlptgoD` |
| `-r` | `--recursive` | rekurzivno |
| `-v` | `--verbose` | ispiši imena fajlova; `-vv`, `-vvv` za više detalja |
| `-q` | `--quiet` | prigušen izlaz (za cron) |
| `-n` | `--dry-run` | **simulacija** bez ijedne izmene |
| `-h` | `--human-readable` | čitljive veličine; `-hh` za jedinice od 1000 |
| `-P` | `--partial --progress` | prikaz napretka + čuvanje delimičnih fajlova |
| `--progress` | | napredak po fajlu |
| `--partial` | | zadrži delimično prenet fajl za nastavak |
| `--partial-dir=DIR` | | čuvaj delimične fajlove u zasebnom direktorijumu |
| `--stats` | | detaljna statistika na kraju |
| `-i` | `--itemize-changes` | **za svaki fajl ispiši šta se tačno menja** (vidi 6.3) |
| `--list-only` | | samo izlistaj izvor, ne prenosi ništa |

### 5.2 Šta se čuva

`-a` je skraćenica za `-rlptgoD` i **ne obuhvata** ACL, proširene atribute ni hard linkove.

| Opcija | Čuva |
|---|---|
| `-r` | rekurzivan obilazak |
| `-l` | simboličke linkove kao linkove |
| `-p` | dozvole |
| `-t` | vremena izmene (**bitno za brzu proveru**) |
| `-g` | grupu |
| `-o` | vlasnika (zahteva root na odredištu) |
| `-D` | uređaje i specijalne fajlove (`--devices --specials`) |
| `-A` | ACL (podrazumeva `-p`) |
| `-X` | proširene atribute (SELinux kontekst, capabilities) |
| `-H` | **hard linkove** — inače se svaki link kopira kao zaseban fajl |
| `-S` | retke (sparse) fajlove efikasno |
| `-U` | vremena pristupa (rsync 3.2+) |
| `--numeric-ids` | ne prevodi UID/GID u imena — obavezno između različitih sistema |
| `--super` | prinudi privilegovane operacije |
| `--fake-super` | čuvaj vlasništvo u xattr kad nema root prava (za bekap servere) |

Preporučeni skup za punu vernost sistema:

```bash
rsync -aHAXS --numeric-ids /src/ /dst/
```

> `-o` bez root prava tiho ne uspeva. Ako bekap server nema root, koristite `--fake-super`
> na daljinskoj strani preko `--rsync-path='rsync --fake-super'`.

### 5.3 Izbor fajlova

| Opcija | Značenje |
|---|---|
| `--exclude=ŠABLON` | isključi |
| `--include=ŠABLON` | uključi (ima smisla samo pre nekog `--exclude`) |
| `--exclude-from=FAJL` | šabloni iz fajla, jedan po redu |
| `--include-from=FAJL` | isto, za uključivanje |
| `--filter=PRAVILO` | opšti oblik pravila (vidi odeljak 7) |
| `-F` | `--filter='dir-merge /.rsync-filter'` |
| `--files-from=FAJL` | **prenesi tačno navedene putanje** iz fajla |
| `-0`, `--from0` | lista razdvojena NUL znakovima (bezbedno za imena sa razmacima) |
| `-x`, `--one-file-system` | ne prelazi granice fajl sistema |
| `--max-size=VEL` | preskoči fajlove veće od (`--max-size=100m`) |
| `--min-size=VEL` | preskoči manje od |
| `-m`, `--prune-empty-dirs` | ne kreiraj prazne direktorijume na odredištu |

`-x` je za migracije obavezan koliko i kod `find`: bez njega `rsync /` ulazi u `/proc`,
`/sys`, `/dev` i montirane mrežne diskove.

### 5.4 Brisanje na odredištu

| Opcija | Značenje |
|---|---|
| `--delete` | obriši na odredištu ono čega nema na izvoru |
| `--delete-before` | brisanje pre prenosa (potrebno kad nema mesta) |
| `--delete-during` | brisanje tokom prenosa (podrazumevano u 3.x) |
| `--delete-delay` | odluči tokom, obriši na kraju |
| `--delete-after` | brisanje posle celog prenosa |
| `--delete-excluded` | obriši i ono što je isključeno filterima |
| `--delete-missing-args` | obriši odredište za argumente kojih na izvoru nema |
| `--max-delete=N` | **prekini ako bi trebalo obrisati više od N fajlova** |
| `--force` | dozvoli brisanje nepraznog direktorijuma koji je zamenjen fajlom |
| `--ignore-errors` | briši i kad je bilo grešaka pri prenosu (opasno) |

> `--max-delete` je najvažnija zaštitna opcija u celom alatu. Ako izvorni disk nije montiran,
> `rsync` vidi prazan direktorijum i sa `--delete` briše **kompletno** odredište.
> `--max-delete=100` pretvara katastrofu u poruku o grešci i izlazni kod 25.

### 5.5 Prenos i mreža

| Opcija | Značenje |
|---|---|
| `-z`, `--compress` | kompresija u letu |
| `--compress-level=N` | nivo kompresije 0-9 |
| `--compress-choice=ALG`, `--zc` | `zstd`, `lz4`, `zlibx`, `zlib`, `none` (rsync 3.2+) |
| `--skip-compress=LISTA` | ne kompresuj navedene ekstenzije |
| `-e KOMANDA`, `--rsh=` | daljinska ljuska (`-e 'ssh -p 2222'`) |
| `--rsync-path=KOMANDA` | kako se `rsync` pokreće na daljinskoj strani |
| `--port=N` | port demona |
| `--bwlimit=BRZINA` | ograniči protok (`--bwlimit=10m`; bez sufiksa je KiB/s) |
| `--timeout=N` | prekini ako N sekundi nema saobraćaja |
| `--contimeout=N` | timeout za uspostavljanje veze ka demonu |
| `-W`, `--whole-file` | prenesi ceo fajl, bez delta algoritma |
| `--no-whole-file` | prinudno koristi delta i lokalno |
| `--inplace` | upisuj direktno u ciljni fajl, bez privremene kopije |
| `--append` | dodaj na kraj postojećeg fajla (samo za fajlove koji rastu) |
| `--append-verify` | isto, uz proveru već prenetog dela |
| `--preallocate` | rezerviši prostor unapred (smanjuje fragmentaciju) |
| `--mkpath` | kreiraj nedostajuće direktorijume odredišta (rsync 3.2.3+) |

### 5.6 Bekap i snimci

| Opcija | Značenje |
|---|---|
| `-b`, `--backup` | pre prepisivanja sačuvaj staru verziju |
| `--backup-dir=DIR` | gde se čuvaju stare verzije |
| `--suffix=SUF` | sufiks za stare verzije (podrazumevano `~`) |
| `--link-dest=DIR` | ako je fajl isti kao u DIR, napravi **hard link** umesto kopije |
| `--copy-dest=DIR` | isto, ali kopira umesto linkovanja |
| `--compare-dest=DIR` | ako je isti kao u DIR, uopšte ga ne prenosi |
| `--remove-source-files` | obriši sa izvora fajlove koji su uspešno preneti |

### 5.7 Izlaz i dijagnostika

| Opcija | Značenje |
|---|---|
| `--info=OZNAKE` | fino podešavanje izlaza: `progress2`, `stats2`, `name`, `del`, `skip`, `flist` |
| `--debug=OZNAKE` | dijagnostika interne logike |
| `--out-format=FORMAT` | prilagođen format reda za svaki fajl |
| `--log-file=FAJL` | pisanje u log |
| `--log-file-format=FORMAT` | format reda u logu |

`--info=progress2` daje **ukupan** napredak celog prenosa umesto po fajlu — daleko korisnije
kod hiljada fajlova:

```bash
rsync -a --info=progress2 /src/ /dst/
```

```
      4.21G  63%   82.14MB/s    0:00:28 (xfr#8412, to-chk=3104/14992)
```

### 5.8 Vlasništvo i dozvole na odredištu

| Opcija | Značenje |
|---|---|
| `--chown=KORISNIK:GRUPA` | postavi vlasništvo na odredištu |
| `--chmod=PRAVILA` | postavi dozvole (`--chmod=D755,F644`) |
| `--usermap=`, `--groupmap=` | preslikavanje imena/ID-eva između sistema |
| `-s`, `--secluded-args` | ne dozvoli daljinskoj ljusci da tumači argumente (starije ime: `--protect-args`) |

```bash
rsync -a --chown=www-data:www-data --chmod=D755,F644 /build/ /var/www/html/
```

---

## 6. Analiza izlaza

### 6.1 Standardni prenos

```bash
rsync -avh --delete --stats /var/www/html/ backup@10.0.10.20:/srv/backup/www/
```

```
sending incremental file list
./
index.php
app/config.php
deleting cache/old.tmp

Number of files: 1,204 (reg: 1,150, dir: 54)
Number of created files: 2 (reg: 2)
Number of deleted files: 1 (reg: 1)
Number of regular files transferred: 2
Total file size: 512.34M bytes
Total transferred file size: 14.55K bytes
Literal data: 4.10K bytes
Matched data: 10.45K bytes
File list size: 32.71K
File list generation time: 0.012 seconds
File list transfer time: 0.000 seconds
Total bytes sent: 41.20K
Total bytes received: 8.41K

sent 41.20K bytes  received 8.41K bytes  33.07K bytes/sec
total size is 512.34M  speedup is 10,327.15
```

**Objašnjenje:**

| Red | Značenje |
|---|---|
| `sending incremental file list` | rsync 3.x gradi listu **postupno** i počinje prenos pre nego što je obiđe celu — zato prenos kreće odmah |
| `./` | sam koreni direktorijum je ažuriran (obično vreme ili dozvole) |
| `deleting cache/old.tmp` | posledica `--delete` |
| `Number of files` | ukupno pregledanih stavki, ne prenetih |
| `Total file size` | veličina celog stabla |
| `Total transferred file size` | veličina fajlova koji su ušli u prenos |
| **`Literal data`** | bajtovi koji su **stvarno prešli mrežu** kao novi sadržaj |
| **`Matched data`** | bajtovi ponovo iskorišćeni sa odredišta zahvaljujući delta algoritmu |
| `File list size` | veličina same liste — na milionima fajlova ovo postaje značajno |
| `speedup` | odnos `Total file size` prema stvarno poslatim bajtovima |

Odnos `Literal` prema `Matched` je pravo merilo koristi delta algoritma.
Kada je `Matched data` blizu nule, delta ne pomaže (novi ili u potpunosti izmenjeni fajlovi)
i `-W` bi bio brži.

### 6.2 Red napretka

```
    12,450  100%   11.87MB/s    0:00:00 (xfr#1, to-chk=2/4)
```

| Deo | Značenje |
|---|---|
| `12,450` | preneto bajtova ovog fajla |
| `100%` | procenat ovog fajla |
| `11.87MB/s` | trenutna brzina |
| `0:00:00` | proteklo vreme za ovaj fajl |
| `xfr#1` | redni broj prenetog fajla |
| `to-chk=2/4` | preostalo za proveru / ukupno u trenutno poznatoj listi |

`to-chk` **raste i pada** dok rsync 3.x otkriva nove direktorijume — nije pouzdana procena
ukupnog posla dok se lista ne izgradi do kraja.

### 6.3 `--itemize-changes` — najkorisniji dijagnostički režim

```bash
rsync -ain --delete /var/www/html/ /srv/backup/www/
```

```
cd+++++++++ app/novi/
>f+++++++++ app/novi/index.php
>f.st...... index.php
.f...p..... .htaccess
.d..t...... assets/
hf          app/link.php => app/original.php
*deleting   cache/old.tmp
```

Oznaka ima **11 znakova**: `YXcstpoguax`.

**Pozicija 1 — vrsta izmene:**

| Znak | Značenje |
|---|---|
| `<` | fajl se šalje na daljinsku stranu |
| `>` | fajl se prima na lokalnu stranu |
| `c` | lokalno kreiranje (direktorijum, simbolički link, uređaj) |
| `h` | fajl je hard link ka drugom fajlu |
| `.` | fajl se ne prenosi, menjaju se samo atributi |
| `*` | sledi poruka (npr. `deleting`) |

**Pozicija 2 — tip stavke:**

| Znak | Značenje |
|---|---|
| `f` | običan fajl |
| `d` | direktorijum |
| `L` | simbolički link |
| `D` | uređaj |
| `S` | specijalni fajl (soket, FIFO) |

**Pozicije 3-11 — šta se razlikuje:**

| Poz. | Znak | Značenje |
|---|---|---|
| 3 | `c` | razlikuje se kontrolna suma sadržaja |
| 4 | `s` | razlikuje se veličina |
| 5 | `t` | razlikuje se mtime (`T` = biće postavljeno vreme prenosa) |
| 6 | `p` | razlikuju se dozvole |
| 7 | `o` | razlikuje se vlasnik |
| 8 | `g` | razlikuje se grupa |
| 9 | `u` | razlikuje se vreme pristupa (uz `-U`) |
| 10 | `a` | razlikuje se ACL |
| 11 | `x` | razlikuju se prošireni atributi |

Pored toga: `+` na svim pozicijama znači **nov fajl**, a `.` znači „bez promene“.

**Čitanje primera:**

- `cd+++++++++ app/novi/` — kreira se nov direktorijum.
- `>f+++++++++ app/novi/index.php` — potpuno nov fajl.
- `>f.st......` — postojeći fajl, razlikuju se veličina i vreme; sadržaj se prenosi.
- `.f...p.....` — **ništa se ne prenosi**, menjaju se samo dozvole.
- `.d..t......` — direktorijumu se koriguje vreme izmene.
- `*deleting` — brisanje na odredištu.

Kombinacija `-ain` (itemize + dry-run) je standardni postupak provere pre svakog rizičnog
pokretanja: vidi se tačno šta bi se prenelo, šta izmenilo i šta obrisalo, bez ijedne izmene.

### 6.4 Izlazni kodovi

Za skripte i nadzorne sisteme:

| Kod | Značenje |
|---|---|
| 0 | uspeh |
| 1 | greška u sintaksi ili upotrebi |
| 2 | nekompatibilnost protokola (različite verzije) |
| 3 | greška pri izboru ulaznih/izlaznih fajlova |
| 4 | tražena akcija nije podržana na ovoj platformi |
| 5 | greška pri pokretanju klijent-server protokola |
| 6 | demon ne može da piše u log |
| 10 | greška u soket ulazu/izlazu |
| 11 | greška u fajl ulazu/izlazu |
| 12 | greška u toku rsync protokola |
| 13 | greška u dijagnostici programa |
| 14 | greška u međuprocesnoj komunikaciji |
| 20 | primljen SIGUSR1 ili SIGINT |
| 21 | greška iz `waitpid()` |
| 22 | greška pri alokaciji memorije |
| **23** | **delimičan prenos zbog greške** (npr. nedostatak dozvola za neke fajlove) |
| **24** | **delimičan prenos jer su izvorni fajlovi nestali tokom prenosa** |
| **25** | **`--max-delete` je zaustavio brisanje** |
| 30 | timeout pri slanju/prijemu |
| 35 | timeout pri čekanju na vezu ka demonu |

Kod **24** je čest i po pravilu bezopasan — dešava se kad se privremeni fajlovi obrišu
između izgradnje liste i prenosa. Skripte ga obično tretiraju kao uspeh:

```bash
rsync -a /src/ /dst/
rc=$?
if [ $rc -ne 0 ] && [ $rc -ne 24 ]; then
    echo "rsync neuspešan, kod $rc" >&2
    exit $rc
fi
```

---

## 7. Filter pravila

### 7.1 Osnovna pravila

- Ispituje se **prvo pravilo koje se poklopi** — redosled je presudan.
- `--include` ima smisla samo ako stoji **ispred** nekog `--exclude`.
- Šablon bez `/` poredi se sa **imenom** fajla bilo gde u stablu.
- Šablon koji počinje sa `/` vezan je za **koren prenosa**, ne za koren fajl sistema.
- Šablon koji se završava sa `/` poklapa se samo sa direktorijumima.
- `*` se ne prostire preko `/`; `**` se prostire.
- `?` je jedan znak, `[a-z]` je klasa znakova.

```bash
rsync -a --exclude='*.tmp' --exclude='cache/' /app/ /backup/app/
rsync -a --exclude='/logs/' /app/ /backup/app/       # samo /app/logs, ne i /app/x/logs
rsync -a --exclude='**/node_modules/' /src/ /dst/
```

### 7.2 Prenos samo određenog tipa fajla

Ovo je čest zahtev i **ne radi naivno**:

```bash
rsync -a --include='*.conf' --exclude='*' /etc/ /backup/etc/     # ne radi
```

Problem: `--exclude='*'` isključuje i direktorijume, pa `rsync` u njih nikada ne uđe.
Ispravno je prvo uključiti direktorijume:

```bash
rsync -a --include='*/' --include='*.conf' --exclude='*' --prune-empty-dirs /etc/ /backup/etc/
```

- `--include='*/'` propušta sve direktorijume,
- `--include='*.conf'` propušta ciljne fajlove,
- `--exclude='*'` odbacuje sve ostalo,
- `--prune-empty-dirs` uklanja direktorijume koji su ostali prazni.

### 7.3 `--filter` i pravila po direktorijumu

```bash
rsync -a --filter='- .git/' --filter='+ /vazno.log' --filter='- *.log' /src/ /dst/
```

Modifikatori pravila: `-` isključi, `+` uključi, `P` zaštiti od brisanja,
`R` ukloni zaštitu, `H` sakrij od pošiljaoca, `S` prikaži primaocu.

Zaštita fajlova na odredištu od `--delete`:

```bash
rsync -a --delete --filter='P /uploads/' /src/ /dst/
```

Sve u `/dst/uploads/` preživljava brisanje iako toga na izvoru nema — nezamenljivo kada
odredište sadrži podatke koje generiše sama aplikacija.

Pravila po direktorijumu, kao `.gitignore`:

```bash
rsync -a -F /src/ /dst/
```

`rsync` u svakom direktorijumu čita fajl `.rsync-filter` i primenjuje pravila iz njega
na to podstablo. Dvostruko `-FF` dodatno isključuje same `.rsync-filter` fajlove iz prenosa.

### 7.4 Eksplicitna lista fajlova

```bash
find /var/log -name '*.log' -mtime -1 -print0 > /tmp/lista
rsync -a --files-from=/tmp/lista --from0 / backup@host:/srv/logovi/
```

Kod `--files-from` putanje u listi su **relativne u odnosu na izvor**, ovde `/`.
Ovaj oblik implicira `--relative`, pa se struktura direktorijuma čuva na odredištu.

---

## 8. Praktični scenariji

### 8.1 Migracija servera

```bash
rsync -aHAXS --numeric-ids -x --info=progress2 \
      --exclude='/proc/*' --exclude='/sys/*' --exclude='/dev/*' \
      --exclude='/run/*' --exclude='/tmp/*' --exclude='/mnt/*' \
      --exclude='/media/*' --exclude='/lost+found' \
      -e 'ssh -T -o Compression=no -c aes128-gcm@openssh.com' \
      / root@novi-server:/
```

**Objašnjenje izbora:**

- `-aHAXS` čuva sve: hard linkove, ACL, SELinux kontekste, retke fajlove.
- `--numeric-ids` sprečava pogrešno preslikavanje korisnika kada `/etc/passwd` na dve mašine nije isti.
- `-x` ostaje na korenskom fajl sistemu; dodatne particije se prenose zasebnim pozivima.
- `-T` isključuje pseudoterminal, `Compression=no` isključuje SSH kompresiju (rsync je već radi bolje ako je uključena sa `-z`), a lakši šifarnik smanjuje opterećenje procesora na gigabitnoj mreži.

Migracija se radi u dva prolaza: prvi dok sistem radi, drugi kratko posle gašenja servisa —
drugi prolaz prenosi samo razliku i traje nekoliko minuta.

### 8.2 Inkrementalni snimci sa `--link-dest`

Najvredniji obrazac u celom alatu. Svaki snimak izgleda kao pun bekap, a nepromenjeni fajlovi
zauzimaju prostor samo jednom, jer su hard linkovi ka prethodnom snimku.

```bash
#!/bin/bash
set -u
IZVOR=/srv/podaci/
BAZA=/backup/snimci
DANAS=$(date +%Y-%m-%d_%H%M)
POSLEDNJI=$(ls -1d "$BAZA"/20* 2>/dev/null | sort | tail -1)

mkdir -p "$BAZA"

rsync -aHAX --delete --numeric-ids \
      --max-delete=5000 \
      ${POSLEDNJI:+--link-dest="$POSLEDNJI"} \
      "$IZVOR" "$BAZA/$DANAS.nepotpun/"

rc=$?
if [ $rc -eq 0 ] || [ $rc -eq 24 ]; then
    mv "$BAZA/$DANAS.nepotpun" "$BAZA/$DANAS"
    ln -sfn "$BAZA/$DANAS" "$BAZA/latest"
else
    echo "Bekap neuspešan, kod $rc" >&2
    exit $rc
fi

# Zadrži poslednjih 30 snimaka
ls -1d "$BAZA"/20* | sort | head -n -30 | xargs -r rm -rf
```

Provera efikasnosti:

```bash
du -sh /backup/snimci/*        # svaki snimak izgleda kao pun
du -sh /backup/snimci          # ukupno je bitno manje od zbira
```

```
41G	/backup/snimci/2026-09-04_0300
41G	/backup/snimci/2026-09-05_0300
41G	/backup/snimci/2026-09-06_0300
43G	/backup/snimci
```

Tri snimka od po 41 GB zauzimaju ukupno 43 GB. Vraćanje podataka je obično `cp` ili `rsync`
iz odgovarajućeg direktorijuma — bez alata za dekompresiju i bez lanca inkremenata.

Ograničenja: `--link-dest` traži da odredište bude na **istom fajl sistemu** kao referentni
direktorijum, i ne funkcioniše na fajl sistemima bez hard linkova (FAT, mnoge mrežne deonice).
Za ZFS ili Btrfs jednostavniji izbor su nativni snapshot-ovi.

Obrazac `.nepotpun` pa `mv` je bitan: prekinut bekap se nikada ne prikazuje kao valjan snimak.

### 8.3 Ogledalo (mirror) sa zaštitom

```bash
rsync -a --delete --delete-excluded --max-delete=1000 \
      --exclude='.snapshot/' \
      --filter='P /uploads/' \
      --log-file=/var/log/rsync-mirror.log \
      /srv/sajt/ backup@10.0.10.20:/srv/ogledalo/sajt/
```

Obavezan postupak pre prvog pokretanja:

```bash
rsync -ain --delete /srv/sajt/ backup@10.0.10.20:/srv/ogledalo/sajt/ | grep '^\*deleting' | head -50
rsync -ain --delete /srv/sajt/ backup@10.0.10.20:/srv/ogledalo/sajt/ | grep -c '^\*deleting'
```

Ako broj za brisanje deluje previsoko, izvor verovatno nije montiran.

### 8.4 Nastavak prekinutog prenosa velikog fajla

```bash
rsync -avP --partial-dir=.rsync-partial --timeout=300 \
      /iso/ubuntu-24.04.iso backup@host:/srv/iso/
```

- `--partial-dir` čuva delimičan fajl u skrivenom poddirektorijumu umesto pod ciljnim imenom, pa nedovršen fajl nikada ne izgleda kao gotov.
- Ponovno pokretanje iste komande nastavlja odatle.
- `--timeout` sprečava da se proces zauvek zaglavi na mrtvoj vezi.

Za fajlove koji samo rastu (logovi, dumpovi u toku):

```bash
rsync -av --append-verify /var/log/velika.log backup@host:/srv/logovi/
```

`--append-verify` proverava da se već preneti deo poklapa pre nego što nastavi.
Obično `--append` to ne proverava i može tiho oštetiti fajl ako se početak promenio.

### 8.5 Slike virtuelnih mašina i retki fajlovi

```bash
rsync -av --sparse --inplace --no-compress \
      /var/lib/libvirt/images/vm1.qcow2 backup@host:/srv/vm/
```

- `--sparse` ne zapisuje nule, pa se rezervisani a neiskorišćeni prostor ne prenosi.
- `--inplace` upisuje direktno u ciljni fajl umesto da pravi punu privremenu kopiju — neophodno kada je fajl 200 GB, a na odredištu nema mesta za dve kopije.
- `--no-compress` jer je qcow2 već kompresovan; kompresija bi samo trošila procesor.

> `--inplace` gubi atomičnost: ako se prenos prekine, ciljni fajl je u nekonzistentnom stanju.
> Ne kombinujte ga sa `--link-dest`, jer bi izmena razbila deljene hard linkove svih snimaka.
> Virtuelnu mašinu prethodno ugasite ili napravite snapshot — kopiranje aktivnog diska
> daje neupotrebljivu sliku.

### 8.6 Ograničenje uticaja na produkciju

```bash
nice -n 19 ionice -c3 \
rsync -a --bwlimit=20m --info=progress2 /srv/podaci/ backup@host:/srv/bekap/
```

- `--bwlimit=20m` ograničava na 20 MiB/s. Bez sufiksa vrednost se tumači kao KiB/s (`--bwlimit=20000`).
- `nice -n 19` daje najniži prioritet procesoru.
- `ionice -c3` stavlja disk zahteve u „idle“ klasu — izvršavaju se samo kada disk nema drugog posla.

### 8.7 Bezbedan bekap preko SSH

Zaseban ključ bez lozinke, ograničen na jednu komandu, na bekap serveru:

```
# ~backup/.ssh/authorized_keys na bekap serveru
command="/usr/bin/rrsync -ro /srv/bekap",no-agent-forwarding,no-port-forwarding,no-pty,no-X11-forwarding,restrict ssh-ed25519 AAAA... bekap@web01
```

`rrsync` je skripta iz paketa `rsync` (obično `/usr/share/doc/rsync/scripts/rrsync`) koja
dozvoljava **samo** `rsync` operacije i to ograničene na navedeni direktorijum.
Opcija `-ro` znači „samo čitanje“; za prijem bekapa koristite `-wo` (samo pisanje),
čime kompromitovani web server ne može da pročita ni obriše ranije bekape.

Sa strane koja pokreće prenos:

```bash
rsync -a -e 'ssh -p 2222 -i /root/.ssh/backup_ed25519 -o StrictHostKeyChecking=accept-new' \
      /srv/podaci/ backup@10.0.10.20:podaci/
```

Kada je za čitanje izvora potreban root na daljinskoj strani:

```bash
rsync -a --rsync-path='sudo rsync' user@host:/etc/ /backup/etc/
```

uz odgovarajući `NOPASSWD` unos u `sudoers` samo za `rsync`.

### 8.8 `rsync` demon

Kada SSH nije poželjan (velika brzina, unutrašnja mreža, javno ogledalo),
`/etc/rsyncd.conf` na serveru:

```ini
uid = nobody
gid = nogroup
use chroot = yes
max connections = 10
pid file = /var/run/rsyncd.pid
log file = /var/log/rsyncd.log

[bekap]
    path = /srv/bekap
    comment = Prijem bekapa
    read only = no
    hosts allow = 10.0.10.0/24
    hosts deny = *
    auth users = web01
    secrets file = /etc/rsyncd.secrets
```

```bash
echo "web01:tajna-lozinka" > /etc/rsyncd.secrets
chmod 600 /etc/rsyncd.secrets
sudo systemctl enable --now rsync
```

Klijent:

```bash
echo "tajna-lozinka" > /root/.rsync-pass
chmod 600 /root/.rsync-pass
rsync -a --password-file=/root/.rsync-pass /srv/podaci/ web01@10.0.10.20::bekap/podaci/
```

Izlistavanje dostupnih modula:

```bash
rsync rsync://10.0.10.20/
```

```
bekap           Prijem bekapa
javno           Javno ogledalo
```

> Saobraćaj demona **nije šifrovan**. Koristite ga samo u poverljivoj mreži ili kroz VPN.
> `--password-file` mora imati dozvole 600, inače `rsync` odbija da ga pročita.

### 8.9 Provera bez prenosa

```bash
rsync -avnc --delete /src/ /dst/ | grep -v '/$'
```

`-c` prisiljava poređenje po kontrolnoj sumi, `-n` sprečava izmene. Ovako se otkriva
oštećenje podataka na odredištu koje brza provera po veličini i vremenu ne vidi.
Sporo je, jer se čitaju svi bajtovi na obe strane, ali je jedini pouzdan način.

Ubrzanje na novijim verzijama:

```bash
rsync -avnc --checksum-choice=xxh128 /src/ /dst/
```

`xxh128` je višestruko brži od podrazumevanog MD5, uz istu praktičnu pouzdanost za
otkrivanje oštećenja (nije kriptografski otporan, ali za ovu namenu to nije potrebno).

### 8.10 Prilagođen izlaz za logove i nadzor

```bash
rsync -a --out-format='%t %o %f %b %l' --log-file=/var/log/rsync.log /src/ /dst/
```

| Oznaka | Značenje |
|---|---|
| `%t` | vreme |
| `%o` | operacija (`send`, `recv`, `del.`) |
| `%f` | ime fajla |
| `%b` | preneto bajtova |
| `%l` | ukupna veličina fajla |
| `%i` | itemize oznaka od 11 znakova |
| `%n` | ime fajla bez putanje |
| `%M` | vreme izmene fajla |

Brojanje prenetih fajlova za nadzorni sistem:

```bash
rsync -a --out-format='%i %n' /src/ /dst/ | awk '/^[<>]/ {n++} END {print n+0}'
```

### 8.11 Automatizacija kroz systemd

`/etc/systemd/system/bekap.service`:

```ini
[Unit]
Description=Noćni bekap podataka
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
Nice=19
IOSchedulingClass=idle
ExecStart=/usr/local/sbin/bekap.sh
SuccessExitStatus=24
TimeoutStartSec=6h
```

`/etc/systemd/system/bekap.timer`:

```ini
[Unit]
Description=Pokreni bekap svakog dana u 03:00

[Timer]
OnCalendar=*-*-* 03:00:00
RandomizedDelaySec=15m
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now bekap.timer
systemctl list-timers bekap.timer
journalctl -u bekap.service -n 50
```

`SuccessExitStatus=24` sprečava lažne alarme zbog nestalih privremenih fajlova.
`Persistent=true` pokreće propušten posao ako je server bio ugašen.
Praćenje ide kroz `journalctl`, kao u prethodnom uputstvu iz serije.

---

## 9. Performanse

| Simptom | Uzrok | Rešenje |
|---|---|---|
| spor start, dugo „building file list“ | milioni fajlova | `--files-from` sa unapred pripremljenom listom; podeliti posao po poddirektorijumima |
| velika potrošnja memorije | `-H` gradi tabelu svih inode-ova | izbaciti `-H` ako hard linkovi nisu potrebni |
| spor prenos na LAN-u uz visok CPU | `-z` na već kompresovanim podacima | izbaciti `-z` ili koristiti `--zc=zstd` |
| lokalna kopija sporija od `cp` | delta algoritam nema smisla lokalno | `-W` (podrazumevano za lokalno, ali ne i kroz `--rsh`) |
| mnogo malih fajlova, niska brzina | ograničava latencija, ne propusnost | pokrenuti više paralelnih `rsync` procesa po poddirektorijumima |
| nepotreban ponovni prenos svega | fajl sistem sa grubom vremenskom rezolucijom | `--modify-window=1` ili `--size-only` |
| prekid na `-c` proveri velikog stabla | MD5 je spor | `--checksum-choice=xxh128` |

Paralelizacija po poddirektorijumima:

```bash
ls -1 /srv/podaci | xargs -P 4 -I{} \
  rsync -a "/srv/podaci/{}/" "backup@host:/srv/bekap/{}/"
```

Četiri istovremena prenosa često daju višestruko bolju propusnost od jednog, jer jedan
`rsync` proces na malim fajlovima čeka na latenciju mreže i diska umesto da zasiti vezu.

---

## 10. Česte greške

1. **Završna kosa crta** — `/src` i `/src/` nisu isto; sa `--delete` razlika je destruktivna.
2. **`--delete` bez `--dry-run` i bez `--max-delete`** — nemontiran izvor briše celo odredište.
3. **`-a` uz očekivanje da čuva ACL i xattr** — potrebni su `-A` i `-X` posebno.
4. **Bez `-H`** — hard linkovi se pretvaraju u zasebne kopije i bekap naraste višestruko.
5. **Bez `--numeric-ids` između sistema** — vlasništvo se preslikava po imenima i ispadne pogrešno.
6. **`--include='*.conf' --exclude='*'`** bez `--include='*/'` — ne ulazi u poddirektorijume.
7. **`-z` na LAN-u** — troši procesor, a ne ubrzava ništa.
8. **`--inplace` uz `--link-dest`** — razbija deljene hard linkove svih ranijih snimaka.
9. **Tretiranje koda 24 kao greške** — lažni alarmi u nadzoru.
10. **Prenos aktivne baze podataka** — dobija se nekonzistentna kopija; koristite `mysqldump`, `pg_basebackup` ili snapshot, pa tek onda `rsync`.
11. **`--password-file` sa dozvolama 644** — `rsync` odbija da ga koristi.
12. **Različite verzije `rsync`** — nove opcije tiho ne rade ili prenos puca sa kodom 2.
13. **Nedostajući direktorijum odredišta** — bez `--mkpath` prenos ne uspeva; starije verzije zahtevaju ručni `mkdir -p`.
14. **Očekivanje delta algoritma za nove fajlove** — delta radi samo kada fajl već postoji na odredištu.

---

## 11. Podsetnik (cheat sheet)

```bash
rsync -avh /src/ /dst/                                  # osnovna sinhronizacija
rsync -avhn --delete /src/ /dst/                        # OBAVEZNA proba pre brisanja
rsync -ain --delete /src/ /dst/                          # tačan spisak izmena
rsync -aHAXS --numeric-ids -x / root@novi:/              # migracija sistema
rsync -avP velika.iso host:/srv/                         # prenos sa nastavkom
rsync -a --link-dest=/backup/juce /src/ /backup/danas/   # inkrementalni snimak
rsync -a --delete --max-delete=1000 /src/ /dst/          # ogledalo sa zaštitom
rsync -a --exclude='*.tmp' --exclude='cache/' /src/ /dst/
rsync -a --include='*/' --include='*.conf' --exclude='*' --prune-empty-dirs /etc/ /bkp/
rsync -a --filter='P /uploads/' --delete /src/ /dst/     # zaštita direktorijuma od brisanja
rsync -a -e 'ssh -p 2222 -i /root/.ssh/bkp' /src/ user@host:/dst/
rsync -a --rsync-path='sudo rsync' user@host:/etc/ /bkp/ # root na daljinskoj strani
rsync -a --bwlimit=20m --info=progress2 /src/ host:/dst/ # ograničen protok, ukupan napredak
rsync -avnc /src/ /dst/                                  # provera integriteta bez prenosa
rsync -a --chown=www-data:www-data --chmod=D755,F644 /build/ /var/www/
rsync -a --files-from=/tmp/lista --from0 / host:/dst/    # tačna lista fajlova
rsync -av --append-verify rastuci.log host:/srv/logovi/  # fajl koji raste
rsync -av --sparse --inplace --no-compress vm.qcow2 host:/srv/vm/
rsync --list-only host:/srv/                             # izlistaj bez prenosa
rsync rsync://host/                                      # moduli demona
```

---

## 12. Povezani alati

| Alat | Kada ga koristiti umesto `rsync` |
|---|---|
| `scp` | jednokratno kopiranje jednog fajla; nema delta ni nastavak |
| `sftp` | interaktivan rad sa fajlovima preko SSH |
| `tar` + `ssh` | prvi puni prenos ogromnog broja malih fajlova; često brži od `rsync` |
| `rclone` | sinhronizacija ka S3, Google Drive, Backblaze i drugim objektnim skladištima |
| `restic` / `borg` | bekap sa deduplikacijom, kompresijom i šifrovanjem repozitorijuma |
| `zfs send` / `btrfs send` | replikacija snapshot-ova na nivou fajl sistema, znatno efikasnija |
| `lsyncd` | trajno praćenje izmena preko `inotify` i automatsko pokretanje `rsync`-a |
| `unison` | dvosmerna sinhronizacija; `rsync` je jednosmeran |
| `syncthing` | trajna dvosmerna sinhronizacija između više uređaja |
| `dd` / `Clonezilla` | kopija celog uređaja, uključujući boot sektor i nealocirani prostor |

Kombinacija koja se često previđa: prvi pun prenos milionima malih fajlova uraditi kroz
`tar`, a sve naredne sinhronizacije kroz `rsync`.

```bash
tar -C /srv/podaci -cf - . | ssh host 'tar -C /srv/bekap -xf -'
```

`tar` šalje neprekidan tok bez pregovaranja po fajlu, pa je na prvom prolazu često
nekoliko puta brži. Od drugog prolaza `rsync` je neuporedivo efikasniji.
