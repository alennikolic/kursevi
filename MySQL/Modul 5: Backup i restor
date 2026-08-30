# Modul 5: Backup i restore

*MySQL za Linux administratore*

---

## Uvod: jedini modul koji stvarno mora da vam uđe u prste

Sve što smo do sada radili može da se ponovi. Konfiguracija se prepiše, nalozi se ponovo naprave, firewall se podesi za deset minuta, sertifikat se izda ponovo.

Podaci ne mogu.

Zato je ovo najveći modul u kursu i zato ga treba čitati sa alatom u ruci, a ne kao referencu. Postoji jedno merilo koje odvaja administratora koji zna svoj posao od onoga koji ne zna, i ono glasi:

> **Da li ste ikada, na ovom konkretnom serveru, izvršili potpuni restore i izmerili koliko traje?**

Ako je odgovor ne, nemate backup. Imate fajlove za koje se nadate da su backup.

### Dva pojma koja moraju biti definisana

Pre bilo kakve tehnike, potrebna su dva broja. Bez njih ne možete doneti nijednu smislenu odluku o backupu.

**RPO — Recovery Point Objective.** Koliko podataka smete da izgubite. Ako radite backup jednom dnevno u 2h, a server padne u 20h, izgubili ste 18 sati rada. Vaš RPO je 24 sata.

**RTO — Recovery Time Objective.** Koliko dugo smete da budete nedostupni. Ako restore od 500 GB traje šest sati, vaš RTO je šest sati — bez obzira šta piše u ugovoru.

Ova dva broja određuju sve ostalo:

- RPO od 24 sata → dovoljan je noćni `mysqldump`.
- RPO od 5 minuta → potrebni su binlogovi i point-in-time recovery.
- RPO blizu nule → potrebna je replika, ne backup (Modul 8).
- RTO od 30 minuta na velikoj bazi → logički dump ne dolazi u obzir, treba fizički backup.

**Vaš posao je da ova dva broja izmerite i saopštite, a ne da ih pretpostavite.** Vrlo često će neko reći "ne smemo da izgubimo ništa i moramo biti gore odmah", a kada mu pokažete cenu takvog rešenja, zahtev se odjednom prilagodi realnosti. Taj razgovor je deo posla.

### Pravilo 3-2-1

Klasično, jednostavno, i dalje tačno:

- **3** kopije podataka (original + dva backupa),
- na **2** različita medija/sistema,
- od kojih je **1** na drugoj fizičkoj lokaciji.

Backup na istom disku nije backup. Backup na istom serveru nije backup. Backup u istom rack-u je bolje nego ništa, ali nije rezervna lokacija.

Dodatak koji je danas neophodan: **bar jedna kopija mora biti izvan domašaja kompromitovanog servera.** Ransomware koji uđe na server prvo traži i briše backupe. Ako backup server ima SSH ključ ka bazi, veza treba da ide u suprotnom smeru — backup server *povlači* podatke, baza ne *gura*. Ili koristite skladište sa write-once politikom.

---

## 19. Tri tipa backupa

### Logički backup

Baza se izvozi kao **tekst** — `CREATE TABLE` naredbe i `INSERT` redovi. Alati: `mysqldump`, `mydumper`, `mysqlpump`.

Kako radi: klijent se poveže na server, pročita podatke kroz obične SQL upite i zapiše ih u fajl.

**Prednosti:**
- Potpuno prenosivo: između verzija, između MySQL-a i MariaDB (uz opreznost), između arhitektura.
- Selektivno: možete vratiti jednu tabelu, jednu bazu, ili čak jedan red.
- Čitljivo: možete otvoriti fajl i videti šta je unutra, popraviti nešto, izbaciti nešto.
- Ne zahteva pristup fajl sistemu servera — radi i sa udaljene mašine i sa managed baze.

**Mane:**
- **Restore je spor.** Ovo je odlučujuća mana. Vraćanje znači ponovno izvršavanje svih `INSERT`-a i ponovnu izgradnju svih indeksa. Za bazu od 100 GB to su često sati.
- Backup je takođe spor na velikim bazama.
- Opterećuje server tokom izrade.

**Kada:** baze do nekoliko desetina gigabajta, migracije između verzija, selektivan restore, arhiviranje.

### Fizički backup

Kopiraju se **fajlovi** iz `datadir`-a, u binarnom obliku. Alati: Percona XtraBackup, MySQL Enterprise Backup.

Kako radi: alat kopira `.ibd` fajlove dok server radi, istovremeno prateći redo log da bi uhvatio izmene koje su se desile tokom kopiranja.

**Prednosti:**
- **Restore je brz** — kopiranje fajlova nazad, bez ponovne izgradnje ičega.
- Backup manje opterećuje server nego logički.
- Podržava inkrementalne backupe.

**Mane:**
- Vezano za verziju i platformu. Backup sa MySQL 8.0 ne vraća se na 8.4 bez nadogradnje.
- Sve ili ništa — teško je izvući jednu tabelu.
- Zahteva pristup fajl sistemu servera.
- Alat mora odgovarati verziji servera.

**Kada:** baze preko ~50 GB, kratak RTO, česti backupi.

### Snapshot

Snima se stanje **fajl sistema ili volumena** u datom trenutku. Alati: LVM, ZFS, cloud volume snapshot.

**Prednosti:**
- Izrada je gotovo trenutna, nezavisno od veličine.
- Zanemarljiv uticaj na server u trenutku snimanja.

**Mane:**
- **Snapshot sam po sebi nije backup.** LVM snapshot živi na istom disku, u istoj grupi volumena, na istom serveru. Ako disk otkaže, nestaju i original i snapshot.
- Zahteva pažljivo rukovanje konzistentnošću.
- Nema granularnosti na nivou tabele.
- LVM snapshot koji se prepuni postaje neupotrebljiv, tiho.

**Kada:** kao dopuna, ne kao jedini mehanizam. ZFS je ovde znatno bolji od LVM-a.

### Poređenje

| | Logički | Fizički | Snapshot |
|---|---|---|---|
| Brzina izrade | spor | srednji | trenutan |
| **Brzina restore-a** | **spor** | **brz** | **brz** |
| Veličina | mala (kompresija) | ≈ datadir | mala pa raste |
| Prenosivost verzija | ✅ | ❌ | ❌ |
| Vraćanje jedne tabele | ✅ | teško | teško |
| Inkrementalno | ❌ | ✅ | ✅ |
| Radi sa managed bazom | ✅ | ❌ | zavisi |
| Podržava PITR | ✅ uz binlog | ✅ uz binlog | ✅ uz binlog |

### Realna preporuka

Za većinu servera koje ćete održavati, ovo je razumna kombinacija:

- **Dnevni `mysqldump`**, komprimovan, sa zapisanom pozicijom binloga. Pokriva selektivan restore i migracije.
- **Kontinuirano čuvanje binlogova** izvan servera. Pokriva RPO.
- **XtraBackup** ako baza pređe pedesetak gigabajta ili ako RTO postane kratak.
- **Snapshot** kao dopuna, ako imate ZFS ili cloud volumene.
- **`pt-show-grants` uz svaki backup.** Nalozi nisu u dumpu podataka.

---

## 20. `mysqldump` — opcije koje su obavezne

`mysqldump` je alat koji svi koriste i skoro niko ne koristi ispravno. Podrazumevana podešavanja su nasleđena iz vremena kada je MyISAM bio glavni engine i **nisu bezbedna za InnoDB**.

### Zašto goli `mysqldump` nije dobar

```bash
mysqldump -u root -p baza > backup.sql
```

Ova komanda pravi backup koji **nije konzistentan**. Tabele se čitaju jedna za drugom; ako se podaci menjaju tokom dumpa, dobijate stanje u kojem tabela `narudzbine` odgovara vremenu 02:00, a `stavke` vremenu 02:07. Referencijalni integritet je narušen i to nećete primetiti dok ne bude kasno.

Uz to, nema zapisane pozicije binloga, pa point-in-time recovery nije moguć.

### Kanonska komanda

```bash
mysqldump \
  --single-transaction \
  --source-data=2 \
  --routines \
  --events \
  --no-tablespaces \
  --hex-blob \
  --default-character-set=utf8mb4 \
  --all-databases \
  | gzip > /backup/full-$(date +%F_%H%M).sql.gz
```

Idemo redom kroz svaku opciju, jer ćete ih prilagođavati.

**`--single-transaction`** — otvara transakciju sa `REPEATABLE READ` izolacijom i čita ceo dump iz jednog konzistentnog snimka. Bez zaključavanja tabela, dakle bez blokiranja aplikacije.

Dva ograničenja: radi samo za transakcione engine-e (InnoDB), i DDL naredbe tokom dumpa mogu ga narušiti. MySQL 8 uzima `LOCK INSTANCE FOR BACKUP`, što DDL sprečava, pa je ovaj drugi problem uglavnom rešen.

**Ako imate MyISAM tabele, ova opcija ih ne pokriva.** Provera:

```sql
SELECT table_schema, table_name, engine
FROM information_schema.tables
WHERE engine NOT IN ('InnoDB') AND table_schema NOT IN
      ('mysql','information_schema','performance_schema','sys');
```

Ako ovaj upit vrati redove, imate posla: ili konvertujte tabele u InnoDB, ili prihvatite `--lock-all-tables` (koje blokira ceo server za vreme dumpa).

**`--source-data=2`** — ovo je opcija koja pretvara dump iz "kopije podataka" u "osnovu za point-in-time recovery".

Zapisuje poziciju binloga u trenutku snimka, kao komentar na početku fajla:

```sql
-- CHANGE REPLICATION SOURCE TO SOURCE_LOG_FILE='binlog.000042', SOURCE_LOG_POS=1234567;
```

Vrednost `2` znači "zapiši kao komentar". Vrednost `1` bi zapisala kao izvršnu naredbu, što ne želite kod običnog restore-a.

Na starijim verzijama opcija se zove `--master-data=2`. Ako komanda javi da opcija ne postoji, probajte drugo ime.

Bez ove opcije **ne znate od koje tačke da primenite binlogove** i PITR postaje pogađanje.

**`--routines`** — uključuje stored procedure i funkcije. **Podrazumevano je isključeno.** Ovo je klasičan način da izgubite deo baze a da ne primetite: restore prođe uredno, tabele su tu, a aplikacija puca jer nedostaje procedura.

**`--events`** — uključuje zakazane događaje. Takođe podrazumevano isključeno.

**`--triggers`** — trigeri. Ovo *jeste* podrazumevano uključeno, ali navedite ga eksplicitno ako želite da komanda bude samodokumentujuća.

**`--no-tablespaces`** — izostavlja `CREATE TABLESPACE` naredbe. Praktična korist: bez ove opcije `mysqldump` u MySQL 8 zahteva `PROCESS` privilegiju. Ako je koristite, backup nalog može biti skromniji (Modul 3).

**`--hex-blob`** — binarne kolone zapisuje heksadecimalno. Sprečava oštećenje binarnih podataka pri prolasku kroz alate koji nisu binarno bezbedni.

**`--default-character-set=utf8mb4`** — eksplicitno postavljanje kodiranja. Bez ovoga se povremeno javljaju problemi sa emoji znakovima i ćirilicom, u zavisnosti od podešavanja klijenta.

**`--all-databases`** — sve baze uključujući sistemske. Alternative:

```bash
# jedna baza, SA CREATE DATABASE naredbom
mysqldump --databases shop > shop.sql

# jedna baza, BEZ CREATE DATABASE
mysqldump shop > shop.sql
```

Razlika je važna. Prvi oblik možete vratiti bilo gde i baza će biti kreirana. Drugi zahteva da baza već postoji i da ste izabrali ciljnu bazu — što je korisno kad vraćate pod drugim imenom.

### Opcije koje se dodaju po potrebi

```bash
# GTID — pri restoreu na server koji nije replika
--set-gtid-purged=OFF

# veliki redovi (BLOB kolone)
--max-allowed-packet=512M

# kompresija na mreži pri dumpu sa udaljenog servera
--compress

# preskakanje velikih tabela sa logovima
--ignore-table=shop.access_log --ignore-table=shop.sesije

# samo struktura, bez podataka
--no-data

# samo podaci, bez strukture
--no-create-info
```

Kombinacija poslednja dva je korisna: struktura u jednom fajlu (mala, ide u Git, lako se pregleda), podaci u drugom.

### Nalozi se dumpuju odvojeno

Podsetnik iz Modula 3, jer je ovo najčešća rupa u backup proceduri:

```bash
pt-show-grants > /backup/grants-$(date +%F).sql
# ili, MySQL 8.0.20+
mysqldump --system=users > /backup/korisnici-$(date +%F).sql
```

Scenario koji se ponavlja: server je pao, restore je prošao savršeno, podaci su svi tu — a aplikacija se ne može prijaviti jer nalozi ne postoje. Sat vremena panike zbog jedne komande koja fali u skriptu.

### Zamka sa `pipe`-om koja tiho kvari backup

Ovo je detalj koji zaslužuje pažnju svakog sysadmina.

```bash
mysqldump ... | gzip > backup.sql.gz
echo $?   # 0
```

**Izlazni status pipeline-a je status POSLEDNJE komande.** Ako `mysqldump` padne na pola posla, `gzip` će uredno komprimovati ono što je dobio i vratiti 0. Vaš skript zaključuje da je sve u redu, alarm se ne pali, a vi imate polovičan backup koji izgleda potpuno normalno.

Rešenje, u svakom backup skriptu:

```bash
set -o pipefail
mysqldump ... | gzip > backup.sql.gz
```

Ili eksplicitna provera:

```bash
mysqldump ... | gzip > backup.sql.gz
if [ "${PIPESTATUS[0]}" -ne 0 ]; then
    echo "mysqldump nije uspeo" >&2
    exit 1
fi
```

Dodatna provera koja hvata skraćene fajlove: ispravan `mysqldump` izlaz završava se linijom sa vremenom završetka.

```bash
zcat backup.sql.gz | tail -1
# -- Dump completed on 2026-08-30 02:14:33
```

Ako te linije nema, dump je prekinut. Ovo je jednostavna, pouzdana provera integriteta i treba da bude deo skripta.

### Restore

```bash
zcat /backup/full-2026-08-30_0200.sql.gz | mysql
```

Sa prikazom napretka:

```bash
sudo apt install pv
pv /backup/full.sql.gz | zcat | mysql
```

Za jednu bazu iz punog dumpa:

```bash
zcat full.sql.gz | mysql shop
```

### Ubrzavanje restore-a

Restore velikog dumpa može trajati satima. Sledeća podešavanja ga značajno skraćuju.

Privremeno, na nivou servera:

```sql
SET GLOBAL innodb_flush_log_at_trx_commit = 2;
SET GLOBAL sync_binlog = 0;
```

Ovo smanjuje broj `fsync` poziva. Cena je da bi iznenadni pad servera *tokom restore-a* značio ponavljanje restore-a — što je prihvatljivo, jer podatke ionako imate u dumpu. **Obavezno vratite na prethodne vrednosti nakon završetka.**

U samom restore toku:

```bash
{
  echo "SET SESSION foreign_key_checks = 0;"
  echo "SET SESSION unique_checks = 0;"
  echo "SET SESSION autocommit = 0;"
  zcat backup.sql.gz
  echo "COMMIT;"
} | mysql
```

Ako server **nema replike**, može i:

```sql
SET SESSION sql_log_bin = 0;
```

Ovo sprečava da restore napuni binlogove svojom kopijom svih podataka. **Ne radite ovo ako imate replike** — one bi ostale bez vraćenih podataka i tiho se razišle sa primarnim serverom.

Zapamtite i ovo: restore je pravi trenutak da izmerite RTO. Zabeležite koliko je trajao.

---

## 21. `mydumper` i `myloader` — kada `mysqldump` postane presporo

`mysqldump` je jednonitni. Jedan proces, jedna konekcija, jedan tok. Na serveru sa 16 jezgara koristi jedno.

`mydumper` radi isti posao paralelno.

### Instalacija

```bash
sudo apt install mydumper
```

Ako verzija u repozitorijumu zaostaje, preuzmite `.deb` sa GitHub stranice projekta.

### Backup

```bash
mydumper \
  --host=127.0.0.1 \
  --user=backup \
  --ask-password \
  --outputdir=/backup/mydumper-$(date +%F) \
  --threads=4 \
  --compress \
  --rows=500000 \
  --build-empty-files \
  --triggers --events --routines \
  --logfile=/var/log/mydumper.log
```

Ključne opcije:

- **`--threads=4`** — broj paralelnih niti. Razumno polazište je broj jezgara podeljen sa dva. Više nije uvek brže; u nekom trenutku disk postaje usko grlo.
- **`--rows=500000`** — deli velike tabele na delove. Ovo je bitno za paralelizam pri **restore-u**, jer se delovi jedne tabele mogu učitavati istovremeno.
- **`--compress`** — kompresija u hodu.
- **`--build-empty-files`** — kreira fajl i za prazne tabele, da restore ne bi propustio njihovu strukturu.

### Struktura izlaza

```bash
ls /backup/mydumper-2026-08-30/
```

```
metadata
shop-schema-create.sql.gz
shop.korisnici-schema.sql.gz
shop.korisnici.00000.sql.gz
shop.korisnici.00001.sql.gz
shop.narudzbine-schema.sql.gz
...
```

Fajl po tabeli, pa i po delu tabele. To znači da možete izvući **jednu tabelu** bez raspakivanja celog backupa — što je kod `mysqldump`-a mučan posao.

Fajl `metadata` sadrži poziciju binloga:

```
Started dump at: 2026-08-30 02:00:11
SHOW MASTER STATUS:
        Log: binlog.000042
        Pos: 1234567
        GTID: ...
Finished dump at: 2026-08-30 02:08:44
```

Ovo je ekvivalent `--source-data=2` i osnova za PITR.

### Restore

```bash
myloader \
  --host=127.0.0.1 \
  --user=root \
  --ask-password \
  --directory=/backup/mydumper-2026-08-30 \
  --threads=4 \
  --overwrite-tables \
  --verbose=3
```

Samo jedna baza:

```bash
myloader --directory=/backup/... --source-db=shop --database=shop_test --threads=4
```

### Kada preći na `mydumper`

Nema oštre granice, ali orijentiri:

- Backup traje duže od raspoloživog prozora.
- Restore traje duže od RTO-a.
- Baza je preko 50–100 GB.
- Treba vam mogućnost da izvučete jednu tabelu iz backupa bez muke.

Realan dobitak u brzini je često tri do pet puta, i pri backupu i pri restore-u.

---

## 22. Percona XtraBackup

Kada logički dump više ne stiže, ovo je alat.

XtraBackup kopira `.ibd` fajlove dok server radi, istovremeno prateći redo log. Rezultat je fizička kopija cele instance koja se vraća kopiranjem fajlova nazad — bez ponovne izgradnje indeksa, bez izvršavanja `INSERT`-a.

### Verzija mora da odgovara

Ovo je prva stvar koja se pogreši:

| Server | Alat |
|---|---|
| MySQL 8.0 | `percona-xtrabackup-80` |
| MySQL 8.4 | `percona-xtrabackup-84` |
| MariaDB | **`mariabackup`**, ne XtraBackup |

Za MariaDB koristite `mariabackup` iz MariaDB paketa. XtraBackup na MariaDB neće raditi ispravno — podsetnik na razlike iz Modula 1.

### Instalacija

```bash
wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb
sudo dpkg -i percona-release_latest.generic_all.deb
sudo apt update

sudo percona-release enable-only tools release
sudo apt install percona-xtrabackup-80 qpress
```

### Nalog

Iz Modula 3:

```sql
CREATE USER 'xtrabackup'@'localhost' IDENTIFIED BY '...';
GRANT BACKUP_ADMIN, PROCESS, RELOAD, LOCK TABLES, REPLICATION CLIENT
  ON *.* TO 'xtrabackup'@'localhost';
GRANT SELECT ON performance_schema.log_status TO 'xtrabackup'@'localhost';
```

### Puni backup

```bash
sudo xtrabackup --backup \
  --user=xtrabackup --password='...' \
  --target-dir=/backup/base-$(date +%F)
```

Uspešan završetak prepoznajete po poslednjoj liniji:

```
completed OK!
```

**Proveravajte ovu liniju u skriptu.** XtraBackup može izaći sa statusom 0 a da posao nije potpun.

Šta je nastalo:

```bash
ls /backup/base-2026-08-30/
```

```
ibdata1  mysql/  shop/  undo_001  undo_002
xtrabackup_binlog_info
xtrabackup_checkpoints
xtrabackup_info
backup-my.cnf
```

`xtrabackup_binlog_info` sadrži poziciju binloga — osnova za PITR:

```
binlog.000042    1234567    3a7f...:1-98234
```

### Priprema — korak koji se ne preskače

Sirov backup **nije upotrebljiv.** Podaci u njemu su nekonzistentni, jer su fajlovi kopirani dok se menjaju. Redo log koji je uz njih sadrži izmene koje treba primeniti.

```bash
sudo xtrabackup --prepare --target-dir=/backup/base-2026-08-30
```

Ponovo tražite `completed OK!`.

**Pripremu možete uraditi odmah nakon backupa, na backup serveru, ili tek pri restore-u.** Preporuka je odmah — tako u trenutku hitnog restore-a imate spreman backup, a i otkrićete odmah ako nešto nije u redu.

### Restore

```bash
# 1. Zaustavite server
sudo systemctl stop mysql

# 2. Sklonite postojeći datadir — ne brišite ga odmah
sudo mv /var/lib/mysql /var/lib/mysql.old

# 3. Vratite podatke
sudo xtrabackup --copy-back --target-dir=/backup/base-2026-08-30

# 4. Prava — bez ovoga server neće startovati
sudo chown -R mysql:mysql /var/lib/mysql

# 5. Start
sudo systemctl start mysql
```

Korak 4 je najčešće mesto na kome ljudi zapnu. `xtrabackup` radi kao root i vraćeni fajlovi su u vlasništvu root-a. Server, koji radi kao `mysql`, ne može da ih otvori.

Alternativa `--copy-back`-u kada nemate dovoljno prostora za dve kopije:

```bash
sudo xtrabackup --move-back --target-dir=/backup/base-2026-08-30
```

Ovo premešta umesto da kopira — brže je i ne troši duplo prostora, ali uništava backup u procesu. Koristite samo kada imate još jednu kopiju.

### Inkrementalni backup

```bash
# nedeljni puni
sudo xtrabackup --backup --target-dir=/backup/base

# dnevni inkrementalni
sudo xtrabackup --backup \
  --target-dir=/backup/inc1 \
  --incremental-basedir=/backup/base

sudo xtrabackup --backup \
  --target-dir=/backup/inc2 \
  --incremental-basedir=/backup/inc1
```

Priprema inkrementalnog lanca ima **strog redosled** i opciju `--apply-log-only` na svim koracima osim poslednjeg:

```bash
sudo xtrabackup --prepare --apply-log-only --target-dir=/backup/base

sudo xtrabackup --prepare --apply-log-only \
  --target-dir=/backup/base --incremental-dir=/backup/inc1

# POSLEDNJI korak — BEZ --apply-log-only
sudo xtrabackup --prepare \
  --target-dir=/backup/base --incremental-dir=/backup/inc2
```

Nakon ovoga, `/backup/base` sadrži spojeno stanje do `inc2`.

Budite realni oko inkrementalnih backupa: **oni komplikuju restore i uvode nove načine da se pogreši.** Ako vam prostor i vreme dozvoljavaju dnevni puni backup, uzmite dnevni puni backup. Inkrementalne uvodite tek kada zaista morate.

### Streaming direktno na drugi server

```bash
sudo xtrabackup --backup --stream=xbstream --target-dir=./ \
  | ssh backup@backup-server "xbstream -x -C /backup/db01-$(date +%F)"
```

Ovim backup nikada ne dodiruje lokalni disk baze. Korisno kada na bazi nema slobodnog prostora.

Sa kompresijom:

```bash
sudo xtrabackup --backup --compress --compress-threads=4 \
  --stream=xbstream --target-dir=./ \
  | ssh backup@backup-server "xbstream -x -C /backup/db01"
```

Dekompresija pre pripreme:

```bash
xtrabackup --decompress --remove-original --target-dir=/backup/db01
```

---

## 23. Snapshot backup

### Zašto snapshot sam nije backup

Rečeno je u poglavlju 19, ali ponavljam jer je ovo najčešći nesporazum:

LVM snapshot je na **istoj grupi volumena**, dakle na istom fizičkom skladištu, na istom serveru. Otkaz diska, otkaz kontrolera, brisanje VM-a, ransomware — sve to odnosi i original i snapshot.

**Snapshot je mehanizam koji vam omogućava da napravite konzistentnu kopiju, a ne kopija sama po sebi.** Kopiju morate odneti negde drugde.

### Konzistentnost

InnoDB je otporan na iznenadan prekid — na tome počiva redo log. Snapshot uzet bez ikakve pripreme daje "crash-consistent" stanje, koje će se pri startu oporaviti kao nakon nestanka struje.

To je upotrebljivo, ali ima dva nedostatka: oporavak traje, i **ne znate tačnu poziciju binloga**, pa PITR postaje neprecizan.

Ispravan postupak:

```sql
LOCK INSTANCE FOR BACKUP;
FLUSH TABLES WITH READ LOCK;
SHOW BINARY LOG STATUS;   -- MySQL 8.4+; na 8.0: SHOW MASTER STATUS
```

— sada, u drugoj sesiji, napravite snapshot —

```sql
UNLOCK TABLES;
UNLOCK INSTANCE;
```

Ključno je da **sesija koja drži zaključavanje ostane otvorena** dok snapshot ne bude gotov. Ako se ta sesija zatvori, zaključavanje se otpušta.

Zato se ovo u praksi radi skriptom, ne ručno.

### LVM u praksi

```bash
# Ima li mesta za snapshot u grupi volumena?
sudo vgs
sudo lvs
```

```bash
#!/bin/bash
set -euo pipefail

SNAP_SIZE=20G
VG=vg_data
LV=lv_mysql
MNT=/mnt/mysql-snap

# Zaključavanje u pozadinskoj sesiji koja ostaje otvorena
mysql -e "LOCK INSTANCE FOR BACKUP;
          FLUSH TABLES WITH READ LOCK;
          SELECT SLEEP(60);" &
LOCK_PID=$!
sleep 3

mysql -e "SHOW MASTER STATUS\G" > /backup/binlog-pozicija.txt

sudo lvcreate -L "$SNAP_SIZE" -s -n mysql-snap "/dev/$VG/$LV"

kill "$LOCK_PID" 2>/dev/null || true

sudo mkdir -p "$MNT"
sudo mount -o ro,nouuid "/dev/$VG/mysql-snap" "$MNT"

# Kopiranje NEGDE DRUGDE — ovo je pravi backup
sudo rsync -a --info=progress2 "$MNT/" backup@backup-server:/backup/db01/

sudo umount "$MNT"
sudo lvremove -f "/dev/$VG/mysql-snap"
```

Dve stvari koje treba znati o LVM snapshotima:

**Performanse padaju dok snapshot postoji.** LVM koristi copy-on-write: svaki upis u originalni volumen zahteva i upis u snapshot prostor. Zato snapshot držite što kraće.

**Snapshot koji se prepuni postaje neupotrebljiv, i to tiho.** Ako ste dodelili 20 GB a tokom trajanja se promeni više, snapshot se invalidira. Praćenje:

```bash
sudo lvs -o +snap_percent
```

### ZFS — znatno bolje rešenje

Ako imate izbor, ZFS je za ovu namenu bolji od LVM-a po svim parametrima.

```bash
# snapshot je trenutan i praktično besplatan
sudo zfs snapshot tank/mysql@$(date +%F-%H%M)

# prenos na drugi server
sudo zfs send tank/mysql@2026-08-30-0200 \
  | ssh backup@backup-server "zfs receive tank/backup/db01@2026-08-30-0200"

# inkrementalni prenos — samo razlika
sudo zfs send -i tank/mysql@2026-08-29-0200 tank/mysql@2026-08-30-0200 \
  | ssh backup@backup-server "zfs receive tank/backup/db01"
```

Prednosti u odnosu na LVM: nema degradacije performansi dok snapshot postoji, nema opasnosti od prepunjavanja, inkrementalni prenos je ugrađen, a `zfs send` daje pravu kopiju na drugom sistemu.

Podešavanja koja se preporučuju za MySQL na ZFS-u:

```bash
sudo zfs set recordsize=16k tank/mysql      # veličina InnoDB stranice
sudo zfs set atime=off tank/mysql
sudo zfs set logbias=throughput tank/mysql
sudo zfs set primarycache=metadata tank/mysql   # ako je buffer pool velik
```

Poslednja opcija sprečava dvostruko keširanje istih podataka — jednom u InnoDB buffer pool-u, drugi put u ZFS ARC-u.

### Cloud snapshot volumena

EBS snapshot, DigitalOcean volume snapshot i slično. Uzimaju crash-consistent stanje.

Za bolju konzistentnost, kombinujte sa zamrzavanjem fajl sistema:

```bash
mysql -e "LOCK INSTANCE FOR BACKUP; FLUSH TABLES WITH READ LOCK; SELECT SLEEP(30);" &
sleep 2
sudo fsfreeze -f /var/lib/mysql
# ... pozovite API za snapshot ...
sudo fsfreeze -u /var/lib/mysql
```

`fsfreeze` blokira sve upise u fajl sistem. Držite ga zamrznutim što kraće — aplikacija za to vreme stoji.

---

## 24. Restore koji zaista radi

Ovo je poglavlje zbog kog ceo modul postoji.

### Testiranje nije opciono

Ponavljam tvrdnju sa početka: **netestiran backup nije backup.**

Stvari koje se otkriju tek pri prvom pravom restore-u, sve viđene u praksi:

- Dump nema `--routines`, pa aplikacija puca na nedostajućoj proceduri.
- Nalozi nisu backupovani, pa se aplikacija ne može prijaviti.
- Restore traje devet sati, a RTO je bio dva.
- Backup fajl je poslednjih pet meseci prazan, jer je nalog promenio lozinku.
- Nema dovoljno prostora na disku za restore.
- Backup je sa MySQL 8.0, a novi server je 8.4 i ne prihvata ga.
- `max_allowed_packet` je premalen za red sa velikim BLOB-om i restore pada na pola.
- Kodiranje se ne poklapa, pa su ćirilični tekstovi neupotrebljivi.

Svaka od ovih stavki je banalna kad se otkrije u testu i katastrofalna kad se otkrije u incidentu.

### Procedura vežbe restore-a

Radite je bar kvartalno, a idealno automatizovanu noću.

**1. Izolovan server.** Nikada na produkciji. Virtuelna mašina ili kontejner, sa istom verzijom MySQL-a.

**2. Merite vreme.**

```bash
START=$(date +%s)
zcat /backup/full-2026-08-30.sql.gz | mysql
END=$(date +%s)
echo "Restore trajao: $(( (END - START) / 60 )) minuta"
```

Ovaj broj je vaš stvarni RTO. Zapišite ga i saopštite ga.

**3. Vratite naloge.**

```bash
mysql < /backup/grants-2026-08-30.sql
```

**4. Provera strukture.**

```sql
SELECT table_schema, COUNT(*) AS tabela
FROM information_schema.tables
WHERE table_type = 'BASE TABLE'
GROUP BY table_schema;

SELECT routine_schema, COUNT(*) FROM information_schema.routines
GROUP BY routine_schema;

SELECT trigger_schema, COUNT(*) FROM information_schema.triggers
GROUP BY trigger_schema;
```

Uporedite sa produkcijom. Razlike u broju tabela, procedura ili trigera znače da je backup nepotpun.

**5. Provera podataka.**

```sql
SELECT COUNT(*) FROM shop.korisnici;
SELECT MAX(id), MAX(kreirano) FROM shop.narudzbine;
CHECKSUM TABLE shop.korisnici;
```

Za ozbiljno poređenje sa produkcijom:

```bash
pt-table-checksum --databases=shop
```

**6. Provera konzistentnosti tabela.**

```bash
mysqlcheck --all-databases --check
```

**7. Provera iz ugla aplikacije.** Ovo je korak koji se najčešće preskoči, a najviše vredi. Podignite aplikaciju protiv vraćene baze i probajte prijavu, jedan upit i jedan upis. Baza koja "izgleda dobro" a aplikacija ne radi nije vraćena.

**8. Zapišite nalaz.**

```
Datum vežbe:        2026-08-30
Izvor backupa:      full-2026-08-29_0200.sql.gz (48 GB komprimovano)
Veličina baze:      312 GB
Trajanje restore-a: 4h 12min
Vraćeni nalozi:     da (grants-2026-08-29.sql)
Provera podataka:   OK, brojevi redova se poklapaju
Test aplikacije:    OK
Problemi:           max_allowed_packet morao na 512M
Izmereni RTO:       4h 30min uključujući podešavanja
```

Ovaj dokument je ono što pokazujete kada vas neko pita da li imate backup.

### Automatizovana vežba restore-a

Najbolja praksa koju možete uvesti: **noćni automatizovan restore poslednjeg backupa na test instancu**, sa proverom i alarmom ako nešto nije u redu.

Time se problem sa backupom otkriva narednog jutra, a ne u toku incidenta. Skicu skripta imate u poglavlju 26.

---

## 25. Point-in-time recovery

Scenario: u 14:35 neko je izvršio `DELETE FROM narudzbine` bez `WHERE`. Poslednji backup je od 02:00. Treba vratiti stanje na 14:34.

### Šta je potrebno

1. Puni backup sa **zapisanom pozicijom binloga** (`--source-data=2` ili `xtrabackup_binlog_info`).
2. **Sve binlogove** od te pozicije do trenutka nesreće.
3. `binlog_format = ROW` — podrazumevano u MySQL 8 i pouzdanije od `STATEMENT`.

Ako nemate poziciju binloga, PITR je pogađanje. Ako nemate binlogove, PITR ne postoji.

### Prvo: zaustavite dalju štetu

```sql
SET GLOBAL super_read_only = ON;
```

Ovo sprečava dalje izmene dok razmišljate. Prvi refleks u ovakvoj situaciji treba da bude zaustavljanje upisa, ne trčanje ka backupu.

### Pronalaženje tačne pozicije

Prvo pogledajte koji binlogovi postoje:

```sql
SHOW BINARY LOGS;
```

Zatim čitajte binlog u čitljivom obliku:

```bash
sudo mysqlbinlog --base64-output=DECODE-ROWS --verbose \
  /var/lib/mysql/binlog.000042 | less
```

Traženje problematične naredbe:

```bash
sudo mysqlbinlog --base64-output=DECODE-ROWS --verbose \
  --start-datetime="2026-08-30 14:30:00" \
  --stop-datetime="2026-08-30 14:40:00" \
  /var/lib/mysql/binlog.000042 | grep -n -B 20 "DELETE FROM"
```

U izlazu tražite blok koji izgleda ovako:

```
# at 89234567
#260830 14:35:02 server id 1  end_log_pos 89234789  Query   thread_id=4521
SET TIMESTAMP=1788352502;
DELETE FROM narudzbine
```

Broj **89234567** je pozicija pre koje treba stati.

**Koristite poziciju, ne vreme.** `--stop-datetime` ima preciznost od jedne sekunde, a u jednoj sekundi može biti stotine događaja. Pozicija je tačna.

### Postupak oporavka

**Radite na zasebnoj instanci, ne na produkciji.** To je pravilo bez izuzetka. Vratite podatke sa strane, proverite ih, pa tek onda prebacite na produkciju.

**Korak 1 — vratite puni backup na test instancu**

```bash
zcat /backup/full-2026-08-30_0200.sql.gz | mysql -h 127.0.0.1 -P 3307
```

**Korak 2 — pročitajte polaznu poziciju iz dumpa**

```bash
zcat /backup/full-2026-08-30_0200.sql.gz | head -30 | grep "SOURCE_LOG"
```

```
-- CHANGE REPLICATION SOURCE TO SOURCE_LOG_FILE='binlog.000042', SOURCE_LOG_POS=1234567;
```

**Korak 3 — primenite binlogove do tačke pre nesreće**

```bash
sudo mysqlbinlog \
  --start-position=1234567 \
  --stop-position=89234567 \
  /var/lib/mysql/binlog.000042 \
  | mysql -h 127.0.0.1 -P 3307
```

Ako se raspon prostire preko više binlog fajlova, navedite ih sve u jednoj komandi — **nikad odvojenim pozivima**:

```bash
sudo mysqlbinlog \
  --start-position=1234567 \
  --stop-position=89234567 \
  /var/lib/mysql/binlog.000042 \
  /var/lib/mysql/binlog.000043 \
  /var/lib/mysql/binlog.000044 \
  | mysql -h 127.0.0.1 -P 3307
```

Razlog: odvojeni pozivi prekidaju transakcije koje se prostiru preko granice fajlova i mogu ostaviti privremene tabele u nedefinisanom stanju.

**Korak 4 — proverite**

```sql
SELECT COUNT(*) FROM shop.narudzbine;
SELECT MAX(kreirano) FROM shop.narudzbine;
```

**Korak 5 — prebacite na produkciju**

Ako je pogođena samo jedna tabela, ne vraćajte celu bazu:

```bash
mysqldump -h 127.0.0.1 -P 3307 --single-transaction shop narudzbine \
  > /tmp/narudzbine-vracene.sql

mysql shop < /tmp/narudzbine-vracene.sql
```

Zatim pustite upise nazad:

```sql
SET GLOBAL super_read_only = OFF;
```

### GTID varijanta

Ako koristite GTID, umesto pozicija radite sa identifikatorima transakcija:

```bash
sudo mysqlbinlog --exclude-gtids='3a7f8c2e-...:98234' \
  /var/lib/mysql/binlog.0000{42,43,44} | mysql
```

Ovo je preciznije — isključujete tačno jednu transakciju, umesto da presecate tok po poziciji.

### Binlogovi moraju biti negde drugde

Najvažnija operativna posledica ovog poglavlja:

**Ako binlogovi žive samo na serveru baze, onda u slučaju otkaza tog servera nemate PITR.** Imate samo noćni backup, dakle RPO od 24 sata — bez obzira što ste mislili da imate bolje.

Rešenje je kontinuirano prenošenje binlogova na drugu mašinu:

```bash
mysqlbinlog \
  --read-from-remote-server \
  --host=db01.interno.local \
  --user=repl --password='...' \
  --raw \
  --stop-never \
  --result-file=/backup/binlogs/ \
  binlog.000042
```

Ova komanda se ponaša kao replika: povezuje se na server i **kontinuirano** preuzima binlogove kako nastaju. Pokrenite je kao systemd servis na backup mašini i vaš RPO pada sa 24 sata na nekoliko sekundi.

```ini
[Unit]
Description=Preuzimanje MySQL binlogova sa db01
After=network-online.target

[Service]
User=backup
ExecStart=/usr/bin/mysqlbinlog --read-from-remote-server \
  --host=db01.interno.local --user=binlog_repl \
  --raw --stop-never --result-file=/backup/binlogs/ binlog.000001
Restart=always
RestartSec=30

[Install]
WantedBy=multi-user.target
```

Ovo je jedan od najboljih odnosa uloženog i dobijenog u celom kursu.

### Prozor za oporavak

Podsetnik iz Modula 2:

```sql
SELECT @@binlog_expire_logs_seconds / 86400 AS dana;
```

Ova brojka mora biti **veća od razmaka između punih backupa**, sa rezervom. Ako radite nedeljni puni backup a binlogove čuvate tri dana, imate četiri dana u kojima PITR nije moguć — a verovatno mislite da jeste.

---

## 26. Automatizacija backupa

### Zašto systemd timer umesto cron-a

Za novo pisanje preporučujem systemd timer:

- izlaz ide u journal, sa oznakom servisa,
- `systemctl status` odmah pokazuje da li je poslednje izvršavanje uspelo,
- `OnFailure=` omogućava automatsko slanje obaveštenja,
- `Persistent=true` nadoknađuje propušteno izvršavanje ako je server bio ugašen,
- `RandomizedDelaySec` razbija sinhrono pokretanje na više servera.

Ako već imate cron i radi — nema hitne potrebe za migracijom. Ali novi skriptovi neka idu u timer.

### Backup skript

```bash
#!/bin/bash
#
# /usr/local/bin/mysql-backup.sh
# Dnevni logički backup MySQL-a sa rotacijom i prenosom na udaljenu lokaciju.

set -euo pipefail

BACKUP_DIR="/backup/mysql"
LOG_TAG="mysql-backup"
DATUM=$(date +%F_%H%M)
RETENCIJA_DANA=14
UDALJENO="backup@backup-server:/backup/db01/"
LOCK="/var/lock/mysql-backup.lock"

log() { logger -t "$LOG_TAG" "$*"; echo "$(date '+%F %T') $*"; }

greska() {
    log "GREŠKA: $*"
    exit 1
}

# Sprečava paralelno pokretanje ako prethodni backup još traje
exec 9>"$LOCK"
flock -n 9 || greska "Backup već radi, prekidam."

log "Početak backupa"

mkdir -p "$BACKUP_DIR"

# --- Provera slobodnog prostora -------------------------------------------
VELICINA_KB=$(du -sk /var/lib/mysql | cut -f1)
SLOBODNO_KB=$(df -k --output=avail "$BACKUP_DIR" | tail -1)
if [ "$SLOBODNO_KB" -lt "$VELICINA_KB" ]; then
    greska "Nedovoljno prostora: treba ~${VELICINA_KB}KB, slobodno ${SLOBODNO_KB}KB"
fi

# --- Dump podataka ---------------------------------------------------------
DUMP="$BACKUP_DIR/full-$DATUM.sql.gz"

set -o pipefail
mysqldump \
    --defaults-file=/etc/mysql/backup.cnf \
    --single-transaction \
    --source-data=2 \
    --routines --events \
    --no-tablespaces \
    --hex-blob \
    --default-character-set=utf8mb4 \
    --all-databases \
    | gzip -6 > "$DUMP" \
    || greska "mysqldump nije uspeo"

# --- Nalozi ----------------------------------------------------------------
GRANTS="$BACKUP_DIR/grants-$DATUM.sql"
pt-show-grants --defaults-file=/etc/mysql/backup.cnf > "$GRANTS" \
    || greska "pt-show-grants nije uspeo"

# --- Kopija konfiguracije --------------------------------------------------
tar czf "$BACKUP_DIR/config-$DATUM.tar.gz" /etc/mysql/ 2>/dev/null || true

# --- Provera integriteta ---------------------------------------------------
gzip -t "$DUMP" || greska "Komprimovani fajl je oštećen"

if ! zcat "$DUMP" | tail -1 | grep -q "Dump completed"; then
    greska "Dump je nepotpun — nedostaje završna linija"
fi

VELICINA=$(du -h "$DUMP" | cut -f1)
log "Dump uspešan: $DUMP ($VELICINA)"

# --- Prenos na udaljenu lokaciju -------------------------------------------
rsync -a --partial "$DUMP" "$GRANTS" "$UDALJENO" \
    || greska "Prenos na udaljenu lokaciju nije uspeo"

log "Prenos završen"

# --- Rotacija --------------------------------------------------------------
find "$BACKUP_DIR" -name "full-*.sql.gz"     -mtime +$RETENCIJA_DANA -delete
find "$BACKUP_DIR" -name "grants-*.sql"      -mtime +$RETENCIJA_DANA -delete
find "$BACKUP_DIR" -name "config-*.tar.gz"   -mtime +$RETENCIJA_DANA -delete

# --- Marker za monitoring --------------------------------------------------
date +%s > "$BACKUP_DIR/.poslednji-uspeh"

log "Backup završen uspešno"
```

Nekoliko stvari u ovom skriptu zaslužuje pažnju:

**`set -euo pipefail`** — prekid pri prvoj grešci, greška na nedefinisanoj promenljivoj, i propagacija greške kroz pipeline. Bez poslednjeg dela `mysqldump | gzip` tiho vraća uspeh, kao što smo videli u poglavlju 20.

**`flock`** — sprečava da se dva backupa preklope. Ako je baza narasla i dump sada traje 26 sati, bez ovoga biste imali dva paralelna dumpa koja se međusobno usporavaju.

**Provera prostora pre početka** — bolje odustati unapred nego napuniti disk na pola posla i time oboriti i bazu.

**Dvostruka provera integriteta** — i `gzip -t` i završna linija dumpa.

**Marker `.poslednji-uspeh`** — objašnjen niže, u odeljku o monitoringu.

**`/etc/mysql/backup.cnf`** — lozinka ne stoji u skriptu:

```ini
[client]
user = backup
password = ...
```

```bash
sudo chmod 600 /etc/mysql/backup.cnf
sudo chown root:root /etc/mysql/backup.cnf
```

Ili, još bolje, koristite `auth_socket` nalog iz Modula 3 i nemate lozinku uopšte.

### Systemd servis i timer

```bash
sudo nano /etc/systemd/system/mysql-backup.service
```

```ini
[Unit]
Description=Dnevni backup MySQL baze
After=mysql.service
Requires=mysql.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/mysql-backup.sh
User=root
Nice=10
IOSchedulingClass=idle
OnFailure=mysql-backup-alarm.service
```

`Nice=10` i `IOSchedulingClass=idle` daju backupu nizak prioritet, da ne bi ugušio produkcijski saobraćaj. Backup koji obori aplikaciju je gori od backupa koji traje sat duže.

```bash
sudo nano /etc/systemd/system/mysql-backup.timer
```

```ini
[Unit]
Description=Pokretanje MySQL backupa svakodnevno

[Timer]
OnCalendar=*-*-* 02:00:00
RandomizedDelaySec=900
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mysql-backup.timer

systemctl list-timers mysql-backup.timer
sudo systemctl start mysql-backup.service   # test odmah
journalctl -u mysql-backup.service -n 50
```

### Obaveštenje o neuspehu

```bash
sudo nano /etc/systemd/system/mysql-backup-alarm.service
```

```ini
[Unit]
Description=Obaveštenje o neuspelom backupu

[Service]
Type=oneshot
ExecStart=/usr/local/bin/posalji-alarm.sh "MySQL backup NIJE uspeo na %H"
```

Skript može slati mail, poruku na Slack, ili gurati metriku u monitoring sistem — nebitno je koji kanal, bitno je da postoji.

### Monitoring backupa — najvažniji deo

Ovo je greška koju pravi većina: **prati se samo neuspeh, a ne i izostanak.**

Ako skript uopšte ne bude pokrenut — timer je isključen, server je bio ugašen, neko je preimenovao fajl — nema greške koju bi neko prijavio. Tišina se tumači kao uspeh, a znači suprotno.

Zato pratite **starost poslednjeg uspešnog backupa**:

```bash
#!/bin/bash
# /usr/local/bin/provera-backupa.sh
# Vraća 0 ako je backup svež, 2 ako nije. Za Nagios/Zabbix/Prometheus.

MARKER="/backup/mysql/.poslednji-uspeh"
MAX_SATI=30

if [ ! -f "$MARKER" ]; then
    echo "CRITICAL: marker uspešnog backupa ne postoji"
    exit 2
fi

POSLEDNJI=$(cat "$MARKER")
SADA=$(date +%s)
STAROST_SATI=$(( (SADA - POSLEDNJI) / 3600 ))

if [ "$STAROST_SATI" -gt "$MAX_SATI" ]; then
    echo "CRITICAL: poslednji uspešan backup pre $STAROST_SATI sati"
    exit 2
fi

echo "OK: poslednji backup pre $STAROST_SATI sati"
exit 0
```

Pratite i **veličinu** backupa. Nagli pad veličine za 80% znači da nešto nije u redu — obrisana baza, greška u skriptu, promenjene privilegije naloga — čak i kada je izlazni status bio nula.

### Šifrovanje kopije na udaljenoj lokaciji

Ako backup napušta vašu infrastrukturu (S3, tuđi server), šifrujte ga.

Sa `age`, koji je jednostavniji od GPG-a:

```bash
sudo apt install age

# jednom: generisanje ključa
age-keygen -o /root/.backup-key.txt
# javni deo iz izlaza ide u skript

# u skriptu, umesto običnog gzip-a:
mysqldump ... | gzip | age -r age1abc... > backup-$DATUM.sql.gz.age

# dešifrovanje pri restore-u
age -d -i /root/.backup-key.txt backup.sql.gz.age | zcat | mysql
```

**Privatni ključ držite izvan servera koji backupujete.** Ako je ključ samo na serveru koji je izgoreo, šifrovani backup je nepovratan.

### Prenos u objektno skladište

```bash
sudo apt install rclone
rclone config   # interaktivno podešavanje S3/B2/GDrive

# u skriptu:
rclone copy "$DUMP" "s3:moj-backup-bucket/db01/" --transfers=4
rclone delete "s3:moj-backup-bucket/db01/" --min-age 30d
```

Za ozbiljniju upotrebu razmotrite `restic`, koji donosi deduplikaciju, šifrovanje i proveru integriteta u jednom alatu.

Šta god koristili, **uključite versioning i object lock ako provajder to nudi.** To je odbrana od scenarija u kome kompromitovan server obriše i udaljene kopije.

### Automatizovana vežba restore-a

Kruna cele automatizacije. Noću, na test mašini:

```bash
#!/bin/bash
# /usr/local/bin/test-restore.sh
set -euo pipefail

TEST_PORT=3307
POSLEDNJI=$(ls -t /backup/mysql/full-*.sql.gz | head -1)

echo "Testiram restore fajla: $POSLEDNJI"

START=$(date +%s)
zcat "$POSLEDNJI" | mysql -h 127.0.0.1 -P "$TEST_PORT"
KRAJ=$(date +%s)
TRAJANJE=$(( (KRAJ - START) / 60 ))

TABELA=$(mysql -h 127.0.0.1 -P "$TEST_PORT" -N -B -e \
  "SELECT COUNT(*) FROM information_schema.tables WHERE table_type='BASE TABLE';")

REDOVA=$(mysql -h 127.0.0.1 -P "$TEST_PORT" -N -B -e \
  "SELECT COUNT(*) FROM shop.korisnici;")

echo "Restore: ${TRAJANJE} min | tabela: ${TABELA} | korisnika: ${REDOVA}"

if [ "$TABELA" -lt 50 ] || [ "$REDOVA" -lt 1000 ]; then
    echo "CRITICAL: restore je sumnjiv — premalo podataka"
    exit 2
fi

echo "$(date +%F) restore OK, ${TRAJANJE} min" >> /var/log/test-restore.log
```

Ako ovo pokrećete svake noći, vaš RTO je **izmeren broj koji se sam ažurira**, a problem sa backupom otkrivate narednog jutra umesto tokom incidenta.

Ovo je razlika između administratora koji ima backup i onoga koji misli da ga ima.

### Dokumentacija

Poslednji korak, i jedini koji nije tehnički. Napišite jednu stranicu i držite je van servera:

```
BACKUP MYSQL — db01.interno.local

Šta se backupuje:  sve baze, nalozi, konfiguracija /etc/mysql
Kada:              svakodnevno 02:00 (systemd timer mysql-backup.timer)
Gde lokalno:       /backup/mysql, čuva se 14 dana
Gde udaljeno:      backup-server:/backup/db01, čuva se 90 dana
Šifrovanje:        age, javni ključ age1abc...
                   PRIVATNI KLJUČ: sef u kancelariji + password manager
Binlogovi:         kontinuirano na backup-server:/backup/binlogs
                   (servis mysql-binlog-stream.service)

RPO:               ~1 minut (binlogovi), 24h ako izgubimo backup server
RTO:               izmereno 4h 30min (poslednja vežba: 2026-08-30)

Ko vraća:          <ime>, zamena <ime>
Procedura restore: /docs/mysql-restore-procedura.md
Poslednja vežba:   2026-08-30, uspešna
Sledeća vežba:     2026-11-30
```

Ovo je dokument koji treba da bude dostupan i kada je server nedostupan. Odštampan, u wiki-ju koji ne stoji na istom hostu, u password manageru — bilo gde osim isključivo na mašini o kojoj govori.

---

## Kontrolna lista na kraju modula

```bash
# 1. Backup se izvršava i marker je svež
/usr/local/bin/provera-backupa.sh

# 2. Dump sadrži procedure, evente i poziciju binloga
zcat /backup/mysql/full-*.sql.gz | head -30 | grep "SOURCE_LOG"
zcat /backup/mysql/full-*.sql.gz | grep -c "CREATE.*PROCEDURE"

# 3. Dump je potpun
zcat /backup/mysql/full-*.sql.gz | tail -1 | grep "Dump completed"

# 4. Nalozi su backupovani odvojeno
ls -la /backup/mysql/grants-*.sql

# 5. Postoji kopija van servera
ssh backup@backup-server "ls -la /backup/db01/ | tail -5"

# 6. Binlogovi se prenose kontinuirano
systemctl status mysql-binlog-stream.service

# 7. Prozor za PITR je duži od razmaka između backupa
mysql -e "SELECT @@binlog_expire_logs_seconds / 86400 AS dana;"

# 8. Restore je testiran i vreme izmereno
tail -5 /var/log/test-restore.log

# 9. Monitoring prati STAROST backupa, ne samo greške

# 10. Dokumentacija postoji IZVAN ovog servera
```

---

## Vežbe

**Vežba 1 — Dokaz da goli `mysqldump` nije konzistentan**
Napravite dve povezane tabele. Pokrenite skript koji neprekidno upisuje u obe. Dok radi, uradite `mysqldump` bez `--single-transaction`, pa sa njim. Vratite oba na test instancu i uporedite referencijalni integritet. Ovo je vežba koja ubeđuje.

**Vežba 2 — Nepotpun backup**
Napravite bazu sa tabelama, procedurom, trigerom i eventom. Uradite `mysqldump` bez `--routines --events`. Vratite i utvrdite šta nedostaje. Zatim napravite naloge i utvrdite da ni oni nisu u dumpu.

**Vežba 3 — Zamka sa `pipe`-om**
Napišite `mysqldump | gzip > f.gz` sa namerno pogrešnom lozinkom. Proverite `echo $?` i veličinu fajla. Zatim dodajte `set -o pipefail` i ponovite. Objasnite zašto je ovo najopasnija tiha greška u backup skriptovima.

**Vežba 4 — Merenje RTO-a**
Napravite bazu od bar 5 GB. Izmerite trajanje `mysqldump`-a i trajanje restore-a. Zatim ponovite sa `mydumper`/`myloader` sa četiri niti. Uporedite i izračunajte na kojoj veličini baze se prelaz isplati.

**Vežba 5 — XtraBackup od nule**
Napravite puni backup, pripremite ga, obrišite `datadir` i vratite. Namerno preskočite `chown -R mysql:mysql` i pročitajte grešku u error logu. Zatim popravite.

**Vežba 6 — Point-in-time recovery**
Napravite backup, pa unesite podatke tokom sledećih pola sata. Zatim izvršite `DELETE` bez `WHERE`. Pronađite tačnu poziciju u binlogu i vratite stanje na trenutak pre brisanja, na zasebnoj instanci. Izmerite koliko je ceo postupak trajao.

**Vežba 7 — Streaming binlogova**
Podignite `mysqlbinlog --read-from-remote-server --stop-never` kao systemd servis na drugoj mašini. Uverite se da fajlovi pristižu. Zatim ugasite bazu naglo i proverite da li ste na drugoj mašini dobili sve do poslednje transakcije.

**Vežba 8 — Kompletna automatizacija**
Implementirajte skript, timer, obaveštenje o neuspehu i proveru starosti. Zatim namerno slomite backup (promenite lozinku naloga) i proverite koliko sati prođe pre nego što vas monitoring obavesti. Podesite pragove na osnovu rezultata.

**Vežba 9 — Restore pod pritiskom**
Zamolite kolegu da bez najave obriše bazu na vašem test serveru u nekom trenutku tokom dana. Vratite je koristeći isključivo svoju dokumentaciju, bez guglanja. Zabeležite šta je nedostajalo u dokumentaciji i dopunite je.

---

## Šta sledi

U **Modulu 6** prelazimo na logove i monitoring — kako da problem primetite pre nego što vas neko pozove telefonom.

Obradićemo četiri MySQL loga i čemu služi svaki, čitanje error loga i tumačenje tipičnih poruka, slow query log i `pt-query-digest` kao način da pronađete upit koji obara server, rotaciju logova bez gubitka handle-a, šta konkretno pratiti u Zabbixu ili Prometheusu i koji alarmi imaju smisla, i dva alata za brzu dijagnozu: `SHOW PROCESSLIST` i `SHOW ENGINE INNODB STATUS`.

I jedna stvar iz ovog modula ide direktno u naredni: **starost poslednjeg uspešnog backupa mora biti metrika u vašem monitoringu.** Ako ništa drugo iz ovog kursa ne implementirate, implementirajte to.
