# Napredne Linux komande — `find`

## 1. Uvod

`find` rekurzivno obilazi stablo direktorijuma i za svaki fajl proverava niz uslova.
Za fajlove koji zadovolje uslove izvršava akciju — podrazumevano ispisuje putanju,
ali može i da pokrene proizvoljnu komandu, obriše fajl ili ispiše prilagođeni format.

Za administratora `find` nije alat za „pronalaženje fajla po imenu“ — za to je `locate`
znatno brži. `find` je alat za **selekciju po metapodacima**: vlasnik, dozvole, veličina,
vreme izmene, broj linkova, tip fajl sistema. Tipični zadaci:

- pronaći sve SUID izvršne datoteke na sistemu (bezbednosna revizija),
- obrisati rezervne kopije starije od 30 dana,
- naći šta je popunilo disk ili potrošilo inode-ove,
- pronaći fajlove izmenjene u prozoru incidenta,
- masovno primeniti komandu na hiljade fajlova, efikasno.

Ovo uputstvo se odnosi na **GNU find** (paket `findutils`), koji je standard na Linuxu.
BSD/macOS varijanta nema `-printf`, `-delete` se ponaša drugačije, a `-perm` sintaksa se razlikuje.

```bash
find --version | head -1
```

```
find (GNU findutils) 4.9.0
```

---

## 2. Sintaksa i redosled argumenata

```
find [OPCIJE_PUTANJE] [PUTANJE...] [IZRAZ]
```

Redosled **nije proizvoljan** i to je izvor većine grešaka:

1. **Opcije putanje** (`-P`, `-L`, `-H`, `-O`) — ispred svega.
2. **Putanje** — jedna ili više polaznih tačaka.
3. **Izraz** — testovi, akcije i operatori.

```bash
find /var/log /tmp -type f -name "*.log" -mtime -1 -print
      └─ putanje    └───────── izraz ──────────────┘
```

Bez putanje `find` koristi tekući direktorijum. Bez akcije podrazumeva `-print`.

### 2.1 Implicitni `-a` (I)

Testovi napisani jedan za drugim su spojeni logičkim **I**:

```bash
find /var -type f -name "*.log"        # tip JESTE fajl I ime odgovara
```

Ovo je isto što i `find /var -type f -a -name "*.log"`.

### 2.2 Kratko spajanje — zašto redosled testova utiče na brzinu

`find` evaluira izraz sleva nadesno i **prekida čim rezultat postane poznat**.
Zato jeftine testove treba staviti prvo:

```bash
find / -type f -name "*.conf" -size +1M        # brzo: -type odsecа direktorijume odmah
find / -size +1M -name "*.conf" -type f        # sporije: -size traži stat() za svaki unos
```

Praktično pravilo: `-name` i `-type` prvo, `-size`, `-perm` i vremenski testovi posle,
`-exec` uvek poslednji.

---

## 3. Opcije putanje (ispred putanja)

| Opcija | Značenje |
|---|---|
| `-P` | **ne prati** simboličke linkove (podrazumevano) |
| `-L` | prati simboličke linkove; testovi se odnose na cilj linka |
| `-H` | prati linkove samo za putanje navedene u komandnoj liniji |
| `-D VRSTA` | dijagnostika: `tree`, `search`, `stat`, `rates`, `opt`, `exec` |
| `-O0` do `-O3` | nivo optimizacije preuređivanja izraza (podrazumevano `-O1`) |

`-L` je koristan kad je struktura izgrađena od simboličkih linkova, ali nosi rizik od
beskonačne petlje; GNU `find` je detektuje i prijavljuje.

`-O3` daje `find`-u slobodu da sam preuredi testove po ceni; korisno na velikim stablima:

```bash
find -O3 / -type f -name "*.log" -size +100M
```

---

## 4. Globalne opcije izraza

Ove opcije važe za ceo izraz bez obzira na to gde su napisane, ali ih zbog čitljivosti
**uvek pišite odmah posle putanje**. `find` inače ispisuje upozorenje.

| Opcija | Značenje |
|---|---|
| `-maxdepth N` | ne silazi dublje od N nivoa (0 = samo polazne putanje) |
| `-mindepth N` | ignoriši nivoe plići od N |
| `-depth` | obradi sadržaj direktorijuma pre samog direktorijuma |
| `-xdev`, `-mount` | **ne prelazi granice fajl sistema** |
| `-daystart` | vremenski testovi se mere od početka dana, ne od tekućeg trenutka |
| `-regextype TIP` | dijalekat za `-regex`: `posix-basic`, `posix-extended`, `egrep`, `emacs` |
| `-ignore_readdir_race` | ne prijavljuj grešku ako fajl nestane tokom obilaska |
| `-files0-from FAJL` | uzmi polazne putanje iz fajla, razdvojene NUL znakom |
| `-nowarn` | isključi upozorenja |

`-xdev` je gotovo obavezan pri pretrazi od korena: bez njega `find /` ulazi u `/proc`,
`/sys`, `/run`, montirane NFS i mrežne diskove, USB uređaje i kontejnerske slojeve.

```bash
sudo find / -xdev -type f -size +1G          # samo root fajl sistem
```

---

## 5. Testovi

### 5.1 Ime i putanja

| Test | Značenje |
|---|---|
| `-name ŠABLON` | ime fajla odgovara shell šablonu (`*`, `?`, `[...]`) |
| `-iname ŠABLON` | isto, bez osetljivosti na veličinu slova |
| `-path ŠABLON` | **cela putanja** odgovara šablonu |
| `-ipath ŠABLON` | isto, neosetljivo na veličinu slova |
| `-regex IZRAZ` | cela putanja odgovara regularnom izrazu |
| `-iregex IZRAZ` | isto, neosetljivo |
| `-lname ŠABLON` | cilj simboličkog linka odgovara šablonu |
| `-ilname ŠABLON` | isto, neosetljivo |

> **Šablon uvek pod navodnicima.** Bez njih shell proširi `*.log` u imena fajlova iz
> tekućeg direktorijuma pre nego što `find` uopšte krene, i dobijate nasumične rezultate
> ili grešku „paths must precede expression“.

Razlika `-name` i `-path`:

```bash
find /var -name "*.log"              # bilo koji .log fajl bilo gde ispod /var
find /var -path "*/nginx/*.log"      # samo .log fajlovi u direktorijumu nginx
```

Kod `-name` `*` **ne prelazi** preko `/`; kod `-path` prelazi.

### 5.2 Tip

| Vrednost `-type` | Značenje |
|---|---|
| `f` | običan fajl |
| `d` | direktorijum |
| `l` | simbolički link |
| `b` | blok uređaj |
| `c` | karakter uređaj |
| `p` | imenovani pipe (FIFO) |
| `s` | soket |

`-xtype` radi isto, ali za simboličke linkove ispituje **cilj**:

```bash
find /usr -xtype l                   # pokvareni simbolički linkovi (cilj ne postoji)
```

### 5.3 Veličina

```
-size N[jedinica]
```

| Sufiks | Jedinica |
|---|---|
| `c` | bajtovi |
| `w` | reči od 2 bajta |
| `b` | blokovi od 512 bajtova (**podrazumevano ako se sufiks izostavi**) |
| `k` | kibibajti (1024 B) |
| `M` | mebibajti |
| `G` | gibibajti |

Prefiksi: `+N` = veće od N, `-N` = manje od N, `N` = tačno N.

> **Zamka zaokruživanja:** veličina se **zaokružuje naviše** na celu jedinicu.
> Fajl od 1 bajta ima `-size 1M`. Zato `-size -1M` ne znači „manji od 1 MB“ nego
> „zaokružena veličina manja od 1“, što pogađa **samo prazne fajlove**.
> Za tačan rezultat koristite bajtove:

```bash
find /var -type f -size -1048576c        # zaista manji od 1 MiB
find /var -type f -size +100M            # ovo radi kako se očekuje (veće od)
```

Prazni fajlovi i direktorijumi:

```bash
find /data -type f -empty
find /data -type d -empty
```

### 5.4 Dozvole

Tri različita oblika, često pobrkana:

| Oblik | Značenje |
|---|---|
| `-perm 644` | dozvole su **tačno** 644 |
| `-perm -644` | **svi** navedeni bitovi su postavljeni (mogu i drugi) |
| `-perm /644` | **bar jedan** od navedenih bitova je postavljen |

```bash
find /etc -type f -perm 600              # tačno rw-------
find / -xdev -type f -perm -4000         # SUID postavljen (bilo koje ostale dozvole)
find /var/www -type f -perm /022         # upisivo za grupu ILI za sve
find /home -type d -perm -0002           # svetski upisivi direktorijumi
```

Simbolički zapis takođe radi:

```bash
find /srv -type f -perm -u+x,g+x
```

### 5.5 Vlasništvo

| Test | Značenje |
|---|---|
| `-user IME` | vlasnik je korisnik |
| `-group IME` | grupa je navedena |
| `-uid N` / `-gid N` | numerički |
| `-nouser` | UID vlasnika ne postoji u `/etc/passwd` |
| `-nogroup` | GID ne postoji u `/etc/group` |

`-nouser` i `-nogroup` su bezbednosni testovi: takvi fajlovi ostaju posle brisanja korisnika
i mogu postati vlasništvo novog korisnika koji dobije isti UID.

### 5.6 Vreme

Tri vremenske oznake u Linuxu:

| Oznaka | Menja se kada |
|---|---|
| **mtime** | promeni se **sadržaj** fajla |
| **atime** | fajl je **pročitan** |
| **ctime** | promene se **metapodaci** (dozvole, vlasnik, ime, broj linkova) ili sadržaj |

Ne postoji vreme kreiranja koje `find` može da čita; `btime` postoji na ext4/XFS,
ali se čita samo preko `statx()` (`stat --printf='%W'`).

| Test | Jedinica |
|---|---|
| `-mtime N` / `-atime N` / `-ctime N` | dani (24 sata) |
| `-mmin N` / `-amin N` / `-cmin N` | minuti |

Prefiksi: `+N` = starije od, `-N` = novije od, `N` = tačno u N-tom intervalu.

> **Najčešća greška:** `-mtime 1` **ne znači** „juče“. Znači „između 24 i 48 sati staro“.
> `-mtime -1` znači „mlađe od 24 sata“. `-mtime +7` znači „starije od 8×24 sata“
> (jer se ostatak odbacuje pri celobrojnom deljenju).

Precizniji i čitljiviji su `-newerXY` testovi:

| Test | Značenje |
|---|---|
| `-newer FAJL` | mtime noviji od mtime navedenog fajla |
| `-newermt "IZRAZ"` | mtime noviji od navedenog **vremena** |
| `-newerct "IZRAZ"` | ctime noviji od navedenog vremena |
| `-newerat "IZRAZ"` | atime noviji od navedenog vremena |
| `-anewer FAJL` | atime noviji od mtime fajla |
| `-cnewer FAJL` | ctime noviji od mtime fajla |

Opšti oblik je `-newerXY`, gde je `X` oznaka koja se poredi za pronađeni fajl
(`a`, `c`, `m`, `B`), a `Y` referenca (`a`, `c`, `m`, `B`, `t` za literalno vreme).

```bash
find /etc -newermt "2026-09-06 02:00" ! -newermt "2026-09-06 03:00" -type f
```

Ovo je tačan prozor od jednog sata — nešto što se sa `-mtime` ne može izraziti.
Prihvataju se svi oblici koje razume `date -d`: `"yesterday"`, `"2 hours ago"`, `"-30 min"`.

> **Napomena o `atime`:** većina modernih sistema montira fajl sisteme sa opcijom `relatime`,
> koja ažurira atime samo ako je stariji od mtime ili od 24 sata. Pretraga po `-atime`
> zato **nije pouzdana** za „koji fajlovi se ne koriste“. Proverite sa `mount | grep relatime`.

### 5.7 Ostali testovi

| Test | Značenje |
|---|---|
| `-inum N` | inode broj |
| `-samefile FAJL` | isti inode kao navedeni fajl (pronalazi sve hard linkove) |
| `-links N` | broj hard linkova (`-links +1` = fajl ima više imena) |
| `-fstype TIP` | tip fajl sistema (`ext4`, `xfs`, `nfs`, `tmpfs`) |
| `-readable` / `-writable` / `-executable` | provera stvarnog pristupa za tekućeg korisnika |
| `-true` / `-false` | uvek tačno / netačno |

`-readable` uzima u obzir ACL-ove i capabilities, za razliku od `-perm` koji gleda samo bitove.

---

## 6. Operatori

| Operator | Značenje | Prioritet |
|---|---|---|
| `\( IZRAZ \)` | grupisanje | najviši |
| `! IZRAZ`, `-not IZRAZ` | negacija | |
| `IZRAZ1 IZRAZ2`, `-a`, `-and` | logičko I | |
| `-o`, `-or` | logičko ILI | |
| `IZRAZ1 , IZRAZ2` | oba se evaluiraju, vraća se rezultat drugog | najniži |

Zagrade i `!` su specijalni znaci u shell-u — moraju se zaštititi:

```bash
find /var \( -name "*.log" -o -name "*.gz" \) -mtime +30
find /home ! -user marko -type f
find /srv \( -nouser -o -nogroup \) -ls
```

> **Klasična greška:** bez zagrada, `find /var -name "*.log" -o -name "*.gz" -mtime +30`
> znači „(.log) ILI (.gz I starije od 30 dana)“ — starost se primenjuje samo na drugi uslov.
> `-a` ima viši prioritet od `-o`, kao `*` i `+` u aritmetici.

---

## 7. Akcije

| Akcija | Značenje |
|---|---|
| `-print` | ispiši putanju i novi red (podrazumevano) |
| `-print0` | ispiši putanju i NUL znak — bezbedno za imena sa razmacima |
| `-printf FORMAT` | ispiši po prilagođenom formatu |
| `-ls` | ispiši u formatu `ls -dils` |
| `-fprint FAJL` / `-fprintf` / `-fls` | isto, ali u fajl |
| `-exec KOMANDA {} \;` | pokreni komandu **jednom po fajlu** |
| `-exec KOMANDA {} +` | pokreni komandu sa **više fajlova odjednom** |
| `-execdir KOMANDA {} \;` | isto, ali iz direktorijuma u kom je fajl |
| `-ok KOMANDA {} \;` | kao `-exec`, ali traži potvrdu za svaki fajl |
| `-okdir KOMANDA {} \;` | kao `-execdir`, uz potvrdu |
| `-delete` | obriši fajl ili prazan direktorijum |
| `-prune` | **ne silazi** u ovaj direktorijum |
| `-quit` | prekini `find` posle prvog pogotka |

### 7.1 `-exec ... \;` naspram `-exec ... +`

```bash
find /var/log -name "*.log" -exec gzip {} \;       # pokreće gzip 5000 puta
find /var/log -name "*.log" -exec gzip {} +        # pokreće gzip nekoliko puta
```

Oblik sa `+` grupiše što više fajlova u jedan poziv, po uzoru na `xargs`. Na 5000 fajlova
razlika je red veličine u brzini. Koristite `\;` samo kada komanda prima **tačno jedan**
argument ili kada `{}` mora da se pojavi više puta:

```bash
find . -name "*.txt" -exec cp {} {}.bak \;         # {} dvaput — mora \;
```

`\;` mora biti zaštićen od shell-a: `\;` ili `';'`.

### 7.2 `-execdir` i bezbednost

```bash
find /tmp -name "*.sh" -execdir chmod 700 {} \;
```

`-execdir` pokreće komandu iz direktorijuma u kom se fajl nalazi i prosleđuje `./ime`
umesto pune putanje. Time se izbegava klasa napada u kojoj napadač zameni direktorijum
simboličkim linkom između trenutka kada ga `find` pronađe i trenutka kada se komanda izvrši.
Za bilo šta destruktivno u direktorijumima u koje mogu da pišu drugi korisnici,
`-execdir` je ispravan izbor.

### 7.3 `-delete`

```bash
find /var/backups -type f -name "*.tar.gz" -mtime +30 -delete
```

Napomene:

- `-delete` implicitno uključuje `-depth`, pa se sadržaj briše pre direktorijuma.
- Zbog toga se **ne kombinuje sa `-prune`** — `-prune` ne radi kada je `-depth` aktivan.
- Briše samo prazne direktorijume; za neprazne koristite `-exec rm -rf {} +`.
- `-delete` je brži i bezbedniji od `-exec rm {} \;` jer koristi `unlinkat()` relativno u odnosu na već otvoreni direktorijum.

> **Obavezan postupak:** uvek prvo pokrenite istu komandu bez akcije, sa `-print`,
> i pregledajte spisak. Tek onda zamenite `-print` sa `-delete`.
> `-delete` napisan **ispred** testova briše sve, jer je akcija istinita i kratko spajanje
> nikad ne stigne do testova.

```bash
find /var/backups -delete -mtime +30     # KATASTROFA: briše sve odmah
find /var/backups -mtime +30 -delete     # ispravno
```

### 7.4 `-prune`

`-prune` sprečava silazak u direktorijum. Standardni obrazac je uvek isti:

```bash
find PUTANJA -path 'ŠTA_PRESKOČITI' -prune -o IZRAZ -print
```

```bash
find /data -path '/data/cache' -prune -o -type f -name "*.csv" -print
```

Preskakanje više direktorijuma:

```bash
find / -xdev \( -path /proc -o -path /sys -o -path /var/lib/docker \) -prune \
       -o -type f -size +500M -print
```

Preskakanje po imenu bilo gde u stablu:

```bash
find /srv -name node_modules -prune -o -name .git -prune -o -type f -name "*.js" -print
```

**Zašto je `-o -print` obavezan:** `-prune` vraća „tačno“, pa bi bez `-o` podrazumevani
`-print` ispisao i same preskočene direktorijume. Kada eksplicitno napišete `-print`
na drugoj grani, podrazumevana akcija se ne dodaje.

### 7.5 `-printf` — formati

| Oznaka | Značenje |
|---|---|
| `%p` | puna putanja |
| `%P` | putanja bez polazne tačke |
| `%f` | samo ime fajla |
| `%h` | direktorijum u kom se fajl nalazi |
| `%s` | veličina u bajtovima |
| `%k` | veličina u blokovima od 1 KB |
| `%b` | veličina u blokovima od 512 B |
| `%m` | dozvole oktalno (`644`) |
| `%M` | dozvole simbolički (`-rw-r--r--`) |
| `%u` / `%g` | ime vlasnika / grupe |
| `%U` / `%G` | numerički UID / GID |
| `%n` | broj hard linkova |
| `%i` | inode broj |
| `%d` | dubina u stablu |
| `%y` | tip fajla (`f`, `d`, `l`, ...) |
| `%Y` | tip prateći simbolički link |
| `%l` | cilj simboličkog linka |
| `%F` | tip fajl sistema |
| `%T@` | mtime kao UNIX timestamp sa decimalama |
| `%TY-%Tm-%Td %TH:%TM` | formatiran mtime |
| `%A...` / `%C...` | isto, ali za atime / ctime |
| `\n`, `\t`, `\0` | novi red, tabulator, NUL |

```bash
find /var/log -type f -printf '%10s  %TY-%Tm-%Td %TH:%TM  %M %u:%g  %p\n' | sort -rn | head
```

```
  524288000  2026-09-06 03:12  -rw-r----- root:adm     /var/log/journal/system.journal
  104857600  2026-09-05 23:58  -rw-r--r-- www-data:adm /var/log/nginx/access.log
   52428800  2026-09-04 11:02  -rw-r----- mysql:adm    /var/log/mysql/slow.log
```

`-printf` je znatno brži i pouzdaniji od `find ... -exec stat {} \;` jer `find` već ima
sve metapodatke u memoriji.

---

## 8. Praktični scenariji

### 8.1 Šta je popunilo disk

```bash
sudo find / -xdev -type f -size +200M -printf '%s\t%p\n' 2>/dev/null | sort -rn | head -20
```

```
5368709120	/var/lib/mysql/ibdata1
2147483648	/var/log/app/debug.log
1073741824	/home/marko/dump.sql
```

- `-xdev` sprečava ulazak u `/proc`, `/sys` i montirane diskove.
- `2>/dev/null` uklanja poruke „Permission denied“ (i bez `sudo` ih ima mnogo).
- Sortiranje po veličini radi `sort -rn`, jer `find` ne sortira.

Ako ovo ne nađe krivca, prostor drže **obrisani a otvoreni fajlovi** — vidi `lsof +L1`.

### 8.2 Iscrpljeni inode-ovi

```bash
df -i
```

```
Filesystem      Inodes   IUsed   IFree IUse% Mounted on
/dev/sda2      6553600 6551203    2397  100% /
```

Pronalaženje direktorijuma sa najviše fajlova:

```bash
sudo find / -xdev -type f -printf '%h\n' 2>/dev/null | sort | uniq -c | sort -rn | head
```

```
4218903 /var/spool/postfix/maildrop
  84102 /var/lib/php/sessions
  31004 /usr/share/man/man3
```

`%h` ispisuje roditeljski direktorijum svakog fajla; grupisanje daje broj fajlova po direktorijumu.
Ovo je znatno brže od `du --inodes` i radi svuda.

### 8.3 Rotacija i čišćenje starih fajlova

```bash
# Prvo pregledati
find /var/backups -type f -name "*.tar.gz" -mtime +30 -printf '%TY-%Tm-%Td %8s %p\n' | sort

# Pa obrisati
find /var/backups -type f -name "*.tar.gz" -mtime +30 -delete

# Kompresovati starije od 7 dana, ali ne već kompresovane
find /var/log/app -type f -name "*.log" -mtime +7 ! -name "*.gz" -exec gzip {} +

# Obrisati prazne direktorijume koji ostanu
find /var/backups -type d -empty -delete
```

Poslednja komanda se oslanja na to da `-delete` implicitno radi `-depth`, pa se prazni
poddirektorijumi brišu pre roditelja i lanac se čisti u jednom prolazu.

### 8.4 Bezbednosna revizija

SUID i SGID izvršne datoteke:

```bash
sudo find / -xdev -type f \( -perm -4000 -o -perm -2000 \) -printf '%M %u %g %p\n' 2>/dev/null
```

```
-rwsr-xr-x root root /usr/bin/sudo
-rwsr-xr-x root root /usr/bin/passwd
-rwsr-sr-x root root /usr/bin/nesto-sumnjivo
```

Sačuvajte referentni spisak i redovno ga poredite:

```bash
sudo find / -xdev -type f -perm /6000 2>/dev/null | sort > /root/suid-baseline.txt
# kasnije
sudo find / -xdev -type f -perm /6000 2>/dev/null | sort | diff /root/suid-baseline.txt -
```

Svetski upisivi direktorijumi bez sticky bita:

```bash
sudo find / -xdev -type d -perm -0002 ! -perm -1000 -printf '%M %p\n' 2>/dev/null
```

Sticky bit (`t`) sprečava korisnike da brišu tuđe fajlove u zajedničkom direktorijumu.
Direktorijum kao `/tmp` bez njega je ozbiljan propust.

Fajlovi bez vlasnika:

```bash
sudo find / -xdev \( -nouser -o -nogroup \) -printf '%U:%G %p\n' 2>/dev/null
```

Svetski upisive izvršne datoteke:

```bash
sudo find /usr /bin /sbin -xdev -type f -perm -0002 -ls 2>/dev/null
```

Skriveni fajlovi na neočekivanim mestima:

```bash
sudo find /tmp /var/tmp /dev/shm -name ".*" -type f -ls 2>/dev/null
```

### 8.5 Šta je izmenjeno u prozoru incidenta

```bash
sudo find /etc /usr/local /opt -xdev \
     -newermt "2026-09-06 02:00" ! -newermt "2026-09-06 03:30" \
     -printf '%TY-%Tm-%Td %TH:%TM:%TS %p\n' 2>/dev/null | sort
```

```
2026-09-06 02:41:18 /etc/ssh/sshd_config
2026-09-06 02:41:22 /etc/cron.d/update
2026-09-06 03:02:55 /usr/local/bin/helper
```

Za forenziku je **ctime** važniji od mtime, jer napadač lako menja mtime sa `touch`,
ali ctime može promeniti samo kernel:

```bash
sudo find / -xdev -newerct "2026-09-06 02:00" -type f -printf '%CY-%Cm-%Cd %CH:%CM %p\n' 2>/dev/null | sort
```

Neslaganje između mtime i ctime je samo po sebi sumnjivo:

```bash
sudo find /usr/bin -type f -newerct "2026-01-01" ! -newermt "2026-01-01" -ls
```

Ovo pronalazi fajlove čiji su metapodaci menjani nedavno, a sadržaj tobože davno.

### 8.6 Masovna obrada sa paralelizacijom

```bash
find /data -type f -name "*.csv" -print0 | xargs -0 -P 8 -n 20 gzip
```

- `-print0` i `-0` koriste NUL kao separator — jedini bezbedan način kod imena sa razmacima, novim redovima ili navodnicima.
- `-P 8` pokreće 8 paralelnih procesa.
- `-n 20` daje po 20 fajlova svakom pozivu.

Kada je dovoljna sekvencijalna obrada, `-exec ... +` je jednostavniji i ne zahteva `xargs`.

### 8.7 Pronalaženje hard linkova i duplikata

Svi hard linkovi jednog fajla:

```bash
find / -xdev -samefile /var/lib/app/data.bin 2>/dev/null
```

Fajlovi sa više od jednog imena:

```bash
find /home -type f -links +1 -printf '%n %i %p\n' | sort -n
```

Kandidati za duplikate po veličini (brza predselekcija pre skupog heširanja):

```bash
find /data -type f -printf '%s %p\n' | sort -n | awk '{
    if ($1 == prev) { print prevline; print $0 }
    prev = $1; prevline = $0
}' | sort -u
```

### 8.8 Pokvareni simbolički linkovi

```bash
find /usr /opt -xtype l -printf '%p -> %l\n'
```

```
/usr/lib/libfoo.so -> libfoo.so.2
/opt/app/current -> /opt/app/releases/2026-08-14
```

Česta posledica nadogradnje ili obrisanog izdanja.

### 8.9 Ispravljanje dozvola

```bash
find /var/www -type d -exec chmod 755 {} +
find /var/www -type f -exec chmod 644 {} +
find /var/www -type f -name "*.sh" -exec chmod 755 {} +
```

Razdvajanje po tipu je bitno: `chmod -R 644` bi oduzeo `x` bit direktorijumima
i učinio ih neprolaznim.

Efikasnija varijanta koja preskače fajlove koji su već ispravni:

```bash
find /var/www -type f ! -perm 644 -exec chmod 644 {} +
```

### 8.10 Brz izlaz posle prvog pogotka

```bash
find / -xdev -name "config.yml" -print -quit 2>/dev/null
```

`-quit` prekida obilazak čim se prvi fajl pronađe — korisno u skriptama koje samo
proveravaju postojanje.

---

## 9. Česte greške

1. **Nezaštićen šablon** — `find . -name *.log` shell proširi pre pokretanja. Uvek pod navodnicima.
2. **Nerazumevanje prioriteta `-o`** — bez zagrada se drugi uslov veže samo za drugu granu.
3. **`-delete` ispred testova** — briše sve. Akcija mora biti poslednja.
4. **Pogrešno razumevanje `-mtime N`** — to je opseg od 24 sata, ne „pre N dana“. Koristite `-newermt` za precizne prozore.
5. **Zaokruživanje kod `-size`** — `-size -1M` pogađa samo prazne fajlove. Koristite `c` jedinicu.
6. **Bez `-xdev` pri pretrazi od `/`** — pretraga ulazi u `/proc`, `/sys` i mrežne diskove i traje beskonačno.
7. **`-exec ... \;` na hiljadama fajlova** — pokreće proces za svaki; koristite `+`.
8. **`-prune` sa `-delete`** — ne rade zajedno jer `-delete` uključuje `-depth`.
9. **Zaboravljeno `-o -print` uz `-prune`** — u izlazu se pojavljuju i preskočeni direktorijumi.
10. **Parsiranje izlaza bez `-print0`** — ime fajla sa razmakom ili novim redom razbija skriptu.
11. **Oslanjanje na `-atime`** — `relatime` čini rezultat nepouzdanim.
12. **`-perm 644` umesto `-perm -644`** — prva forma traži tačno poklapanje i propušta fajlove sa dodatnim bitovima.

---

## 10. Podsetnik (cheat sheet)

```bash
find /var -type f -name "*.log"                       # osnovno
find / -xdev -type f -size +200M -printf '%s\t%p\n' | sort -rn | head
find / -xdev -type f -perm /6000 2>/dev/null          # SUID/SGID revizija
find / -xdev -type d -perm -0002 ! -perm -1000        # svetski upisivi bez sticky bita
find / -xdev \( -nouser -o -nogroup \) -ls            # fajlovi bez vlasnika
find /var/backups -type f -mtime +30 -delete          # čišćenje (prvo bez -delete!)
find /var/log -name "*.log" -mtime +7 -exec gzip {} + # kompresija u grupama
find /data -type d -empty -delete                     # prazni direktorijumi
find /etc -newermt "02:00" ! -newermt "03:30" -type f # tačan vremenski prozor
find / -xdev -newerct "2026-09-06" -type f 2>/dev/null # promene metapodataka (forenzika)
find /srv -name node_modules -prune -o -type f -print # preskakanje direktorijuma
find /usr -xtype l -printf '%p -> %l\n'               # pokvareni linkovi
find /home -type f -links +1 -printf '%n %i %p\n'     # hard linkovi
find /var -xdev -type f -printf '%h\n' | sort | uniq -c | sort -rn | head  # inode potrošači
find /data -type f -print0 | xargs -0 -P8 -n20 gzip   # paralelna obrada
find /var/www -type d -exec chmod 755 {} +            # dozvole po tipu
find / -xdev -name "config.yml" -print -quit          # prvi pogodak pa stop
```

---

## 11. Povezani alati

| Alat | Kada ga koristiti umesto `find` |
|---|---|
| `locate` / `plocate` | trenutno pronalaženje po imenu iz indeksa; ne vidi promene od poslednjeg `updatedb` |
| `fd` | brža i ergonomskija alternativa za svakodnevnu pretragu po imenu |
| `du -h --max-depth=1` | zauzeće po direktorijumima, umesto po fajlovima |
| `ncdu` | interaktivna analiza zauzeća diska |
| `xargs` | fleksibilnije od `-exec`, sa paralelizacijom `-P` |
| `rsync` | kopiranje sa filtriranjem po obrascima, umesto `find \| cpio` |
| `tmpwatch` / `systemd-tmpfiles` | deklarativno čišćenje privremenih fajlova po pravilima |
| `auditd` | praćenje promena fajlova u realnom vremenu, umesto naknadne pretrage |
| `AIDE` / `Tripwire` | provera integriteta sistema sa referentnom bazom heševa |
