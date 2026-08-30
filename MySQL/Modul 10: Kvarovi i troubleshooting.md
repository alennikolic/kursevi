# Modul 10: Kvarovi i troubleshooting

*MySQL za Linux administratore*

---

## Kako koristiti ovaj modul

Ovo je jedini modul u kursu koji nije napisan da se čita s kraja na kraj. Napisan je da se **pretražuje u tri ujutru**, kada ste pospani i kada vas neko zove.

Zato je struktura drugačija: prvo brzo grananje, pa detaljno po scenariju, sa tačnim komandama.

Preporuka: pročitajte ga jednom u miru, uradite vežbe, pa ga odštampajte ili držite u wiki-ju koji **ne stoji na serveru o kome govori**.

### Pravila incidenta

Pre bilo koje komande, četiri stvari:

1. **Ne žurite u prvu ideju.** Deset sekundi razmišljanja štedi sat oporavka. Najgori ishodi incidenata dolaze od ishitrenih poteza, ne od sporosti.
2. **Ne brišite ništa dok ne razumete šta je.** Naročito u `/var/lib/mysql`.
3. **Zapisujte šta radite, u realnom vremenu.** Otvorite `tail -f` u jednom prozoru i beležnicu u drugom. Za dva sata nećete pamtiti šta ste u kom redosledu izvršili — a to je tačno ono što treba za postmortem.
4. **Ako postoji rizik da pogoršate stanje, prvo napravite hladnu kopiju.** Sa ugašenim serverom:
   ```bash
   sudo systemctl stop mysql
   sudo tar czf /backup/datadir-incident-$(date +%F_%H%M).tar.gz /var/lib/mysql
   ```
   Ovo vam daje mogućnost da probate agresivnije metode bez straha.

### Brzo grananje

```
Aplikacija javlja grešku pri povezivanju
├─ "Can't connect to MySQL server"
│   └─ Proces ne radi ili ne sluša      → poglavlje 47
├─ "Too many connections"
│   └─ Limit konekcija iscrpljen        → poglavlje 48
├─ "Access denied"
│   └─ Nalog, host ili lozinka          → Modul 3
└─ "SSL connection error"
    └─ TLS / sertifikat                  → Modul 4

Aplikacija se povezuje, ali stoji ili je spora
├─ Disk je pun                           → poglavlje 49
├─ Upit blokira ostale                   → Modul 6, poglavlje 32
├─ Metadata lock                         → Modul 9, poglavlje 45
└─ Baza je mirna, aplikacija spora       → Modul 7, poglavlje 38

MySQL je nestao bez poruke o gašenju
├─ dmesg pokazuje "Killed process"       → poglavlje 51
├─ Error log pokazuje oštećenje          → poglavlje 50
└─ Ništa ne piše nigde                   → hardver; smartctl, memtest
```

---

## 47. MySQL neće da startuje

### Korak 1: Da li stvarno ne radi?

```bash
systemctl status mysql
pgrep -a mysqld
sudo mysqladmin status
```

**Prva zamka:** `systemctl start mysql` koji "visi" nije kvar. Podsetnik iz Modula 2 — unit je `Type=notify` sa `TimeoutStartSec=infinity`, pa komanda ne vraća prompt dok server ne bude spreman.

Proverite da li se nešto dešava:

```bash
sudo tail -f /var/log/mysql/error.log
top -p "$(pgrep -x mysqld)"
sudo iotop -o
```

Ako u logu vidite napredak oporavka (procente ili LSN brojeve) i proces troši I/O — **server radi, sačekajte.** Ne prekidajte.

Ako je proces mrtav, idemo dalje.

### Korak 2: Pročitajte error log

U devedeset odsto slučajeva odgovor je ovde. Problem je što ljudi ovaj korak preskoče.

```bash
sudo tail -50 /var/log/mysql/error.log
```

**Ako je log prazan ili se nije menjao**, server je pao pre nego što ga je otvorio — obično zbog greške u konfiguraciji. Tada poruka ide u journal:

```bash
journalctl -u mysql -n 50 --no-pager
journalctl -xe
```

**Najkorisniji potez kada ništa nije jasno** je da pokrenete server ručno, u prvom planu:

```bash
sudo -u mysql /usr/sbin/mysqld --user=mysql --console
```

Ovim zaobilazite systemd i sve poruke idu direktno na ekran. Nema skrivenih grešaka, nema pitanja gde je log. Prekinete sa `Ctrl+C` kada pročitate šta vam treba.

Ovo je jedna od najkorisnijih komandi u celom modulu.

### Korak 3: Proverite konfiguraciju

```bash
sudo mysqld --validate-config
```

Tipične poruke:

```
[ERROR] unknown variable 'expire_logs_days=7'
```

Uklonjena promenljiva (Modul 9). Zamenite je aktuelnom.

```
[ERROR] Found option without preceding group in config file
```

Opcija van sekcije `[mysqld]` (Modul 2).

Šta server zaista čita:

```bash
my_print_defaults mysqld
```

Ne zaboravite `mysqld-auto.cnf` (Modul 2) — vrednost može doći odande, a ne iz `/etc/mysql`:

```bash
sudo cat /var/lib/mysql/mysqld-auto.cnf
```

Ako sumnjate da je persistovana vrednost kriva, privremeno je sklonite:

```bash
sudo mv /var/lib/mysql/mysqld-auto.cnf /var/lib/mysql/mysqld-auto.cnf.bak
```

### Korak 4: Prava, vlasništvo, AppArmor

```bash
sudo ls -ld /var/lib/mysql
sudo ls -la /var/lib/mysql | head
id mysql
```

Sve u `datadir`-u mora pripadati korisniku `mysql`:

```bash
sudo chown -R mysql:mysql /var/lib/mysql
sudo chmod 750 /var/lib/mysql
```

Ovo je najčešća greška nakon restore-a iz XtraBackup-a (Modul 5).

Ako su prava ispravna a i dalje piše `Permission denied` — AppArmor (Modul 2):

```bash
sudo dmesg | grep -i apparmor | tail -20
sudo journalctl -k | grep DENIED | tail -20
```

Za brzu proveru da li je AppArmor kriv, privremeno ga prebacite u complain režim:

```bash
sudo aa-complain /usr/sbin/mysqld
sudo systemctl start mysql
```

Ako sada startuje — potvrdili ste uzrok. Napišite pravila i vratite u enforce.

### Korak 5: Resursi i sukobi

```bash
df -h /var/lib/mysql
df -i /var/lib/mysql          # inode-ovi, ne samo prostor
free -h
sudo ss -lntp | grep -E '3306|33060'
pgrep -a mysqld
```

`df -i` se retko proverava, a iscrpljeni inode-ovi daju istu grešku kao pun disk, dok `df -h` pokazuje slobodan prostor. Retko, ali kad se desi, izgubite sat.

### Tabela: simptom → uzrok → rešenje

| Poruka u logu | Uzrok | Rešenje |
|---|---|---|
| `unknown variable 'x'` | uklonjena/pogrešna opcija | ukloniti iz konfiguracije |
| `Operating system error number 13` | prava ili AppArmor | `chown`, pa AppArmor |
| `Operating system error number 24` | fajl deskriptori | `LimitNOFILE` (Modul 7) |
| `Unable to lock ./ibdata1 error: 11` | druga instanca radi | `pgrep -a mysqld`, ugasiti je |
| `Can't start server: Bind on TCP/IP port` | port zauzet | `ss -lntp`, promeniti port ili ugasiti drugi proces |
| `Can't open the mysql.plugin table` | pogrešan ili prazan `datadir` | proveriti `datadir` u konfiguraciji |
| `Table 'mysql.user' doesn't exist` | `datadir` nije inicijalizovan | proveriti putanju; ne inicijalizovati preko postojećih podataka |
| `The innodb_system data file must be writable` | vlasništvo fajlova | `chown -R mysql:mysql` |
| `Disk is full writing` | pun disk | poglavlje 49 |
| `Database page corruption` | oštećenje | poglavlje 50 |
| `Failed to initialize DD Storage Engine` | oštećenje ili neslaganje verzija | poglavlje 50, proveriti verziju paketa |

### Poseban slučaj: `datadir` je prazan ili pogrešan

```bash
mysql -e "SELECT @@datadir;" 2>/dev/null   # ako server radi
grep -r datadir /etc/mysql/
sudo ls -la /var/lib/mysql | head
```

Ako je `datadir` prazan a vaši podaci su negde drugde — **ne pokrećite inicijalizaciju.** Komanda `mysqld --initialize` nad pogrešnom putanjom može prepisati postojeće podatke. Prvo pronađite gde su:

```bash
sudo find / -name "ibdata1" -o -name "mysql.ibd" 2>/dev/null
```

---

## 48. "Too many connections"

### Kako da uđete kada niko ne može

```
ERROR 1040 (HY000): Too many connections
```

Prvi problem je što ni vi ne možete da se povežete da vidite šta se dešava.

**Ako ste uključili administrativni port** (Modul 7, poglavlje 35 — i ovo je trenutak kada se to isplati):

```bash
mysql -h 127.0.0.1 -P 33062 -u root
```

**Ako niste**, MySQL rezerviše jedan dodatni slot za naloge sa `CONNECTION_ADMIN` ili `SUPER` privilegijom. Probajte lokalno:

```bash
sudo mysql
```

Često prolazi. Ako i to padne, ostaje vam da čekate da se neka konekcija oslobodi, ili da restartujete servis — što je poslednja opcija, jer restart velike baze traje.

**Pouka: uključite `admin_port` na svakom serveru pre nego što vam zatreba.**

### Odmah: utvrdite koja je vrsta problema

```sql
SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL STATUS LIKE 'Threads_running';
SELECT @@max_connections;
```

Dva potpuno različita scenarija:

**Scenario A — `Threads_running` je visok (desetine ili više).**

Konekcije se gomilaju jer se upiti ne završavaju. Ovo je pravi problem, uzrok je u bazi ili u zaključavanju.

**Scenario B — `Threads_running` je nizak (1–5), a `Threads_connected` udara u limit.**

Konekcije su otvorene ali ne rade ništa. Ovo je curenje konekcija u aplikaciji ili nedostatak connection poola. Baza je nedužna.

Ovo grananje određuje sve što sledi.

### Scenario A: upiti blokiraju

```sql
SELECT id, user, host, db, command, time, state, LEFT(info, 80) AS upit
FROM performance_schema.processlist
WHERE command NOT IN ('Sleep', 'Daemon')
ORDER BY time DESC
LIMIT 30;
```

Tražite obrazac. Obično je jedan od tri:

**Jedan spor upit, mnogo istih upita iza njega.** Nađite najstariji i pogledajte njegov `state`.

**`Waiting for table metadata lock`** — neko je pokrenuo `ALTER TABLE` (Modul 9). Nađite ga:

```sql
SELECT * FROM sys.schema_table_lock_waits;
```

**Čekanje na zaključane redove:**

```sql
SELECT * FROM sys.innodb_lock_waits\G
```

Duge transakcije:

```sql
SELECT trx_id, trx_state, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS traje_sek,
       trx_mysql_thread_id AS conn_id,
       LEFT(trx_query, 80) AS upit
FROM information_schema.innodb_trx
ORDER BY trx_started;
```

**Prekidanje blokera:**

```sql
KILL QUERY 4521;
```

Preferirajte `KILL QUERY` nad `KILL` (Modul 6). I zapamtite: `KILL` nad `UPDATE`-om koji je već obradio milione redova pokreće rollback koji može trajati duže od samog upita i ne može se prekinuti.

Ako morate prekinuti više njih, prvo pogledajte spisak:

```sql
SELECT CONCAT('KILL QUERY ', id, ';') AS komanda
FROM performance_schema.processlist
WHERE command = 'Query' AND time > 60 AND user NOT IN ('system user');
```

### Scenario B: curenje konekcija

```sql
SELECT user, host, COUNT(*) AS broj
FROM performance_schema.processlist
GROUP BY user, host
ORDER BY broj DESC;
```

Ovo vam odmah kaže **koja aplikacija** i **sa kog servera** drži konekcije.

```sql
SELECT @@wait_timeout, @@interactive_timeout;
SHOW GLOBAL STATUS LIKE 'Threads_created';
SHOW GLOBAL STATUS LIKE 'Connections';
```

Odnos `Threads_created / Connections` blizak jedinici znači da poola nema (Modul 7).

Privremeno rešenje je spuštanje `wait_timeout`, čime server sam zatvara neaktivne konekcije brže:

```sql
SET GLOBAL wait_timeout = 60;
```

**Ovo je zavoj, ne lek.** Pravo rešenje je connection pool u aplikaciji, i to je nalaz koji predajete developeru.

### Privremeno podizanje limita

```sql
SET GLOBAL max_connections = 500;
```

Ovo je dinamičko i deluje odmah. Ali:

**Budite oprezni.** Podizanje limita ne rešava scenario A — samo dozvoljava da se nagomila više konekcija, što server dodatno guši. A svaka konekcija troši memoriju (Modul 7), pa preterivanje vodi u swap ili OOM killer.

Redosled je: prvo razumeti scenario, pa tek onda odlučiti da li limit uopšte treba dizati.

### Posle incidenta

```
1. Koji je bio scenario, A ili B?
2. Ako A — koji upit je bio bloker? (slow log, pt-query-digest)
3. Ako B — koja aplikacija je curila? Ima li pool?
4. Da li je max_connections realno postavljen? (Modul 7, računica memorije)
5. Da li je alarm na 80% konekcija postojao? (Modul 6)
6. Da li je admin_port uključen za sledeći put?
```

---

## 49. Pun disk

### Kako MySQL reaguje

Ovo je važno razumeti, jer je ponašanje neobično: **MySQL ne pada kada disk postane pun. On čeka.**

```
[ERROR] [MY-011971] [InnoDB] Disk is full writing './shop/narudzbine.ibd'
[ERROR] [MY-011972] [InnoDB] Retry attempt is ended.
```

Server u petlji pokušava upis i svakih minut zapisuje poruku. Aplikacija za to vreme stoji na upisima, a čitanja uglavnom rade.

Dobra vest: kada oslobodite prostor, MySQL nastavlja sam od sebe, bez restarta i bez gubitka podataka.

Loša vest: dok ne oslobodite, sve stoji.

### Trijaža — šta je popunilo disk

```bash
df -h /var/lib/mysql
df -i /var/lib/mysql

sudo du -sh /var/lib/mysql/* | sort -rh | head -20
sudo du -sh /var/lib/mysql/binlog.* 2>/dev/null | tail -1
sudo du -sh /var/log/mysql/
```

Ne zaboravite obrisane fajlove koje neki proces još drži otvorenim (Modul 6):

```bash
sudo lsof +L1 | head -20
sudo lsof -p "$(pgrep -x mysqld)" | grep deleted
```

Ako nešto drži obrisani fajl od 40 GB, prostor se oslobađa tek kada se taj proces restartuje ili fajl zatvori.

### Šta osloboditi, redom

**1. Binarni logovi — obično najveći dobitak**

```sql
SHOW BINARY LOGS;
```

**Prvo proverite replike.** Brisanje binloga koji replika još nije pročitala prekida replikaciju nepovratno (Modul 8):

```sql
SHOW REPLICAS;
```

Na svakoj replici:

```sql
SHOW REPLICA STATUS\G
-- pogledajte Source_Log_File
```

Zatim brišite samo ono što je sigurno pročitano:

```sql
PURGE BINARY LOGS TO 'binlog.000210';
-- ili po vremenu:
PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 2 DAY);
```

**Nikad `rm` nad binlog fajlovima.** Server vodi evidenciju u `binlog.index` i brisanje iza njegovih leđa pravi nekonzistentno stanje.

Podsetnik: skraćivanjem roka čuvanja skraćujete i prozor za point-in-time recovery (Modul 5). Ako ste binlogove kontinuirano prenosili na drugu mašinu, ovaj potez je bezbolan. Ako niste — sada znate zašto to treba.

**2. Stari backupi na istom volumenu**

```bash
ls -laSh /backup/ | head -20
```

Ako backupi stoje na istom disku kao podaci, to i nije backup (Modul 5), ali u ovom trenutku je izvor prostora.

**3. Rotirani i tekući logovi**

```bash
sudo ls -laSh /var/log/mysql/
sudo rm /var/log/mysql/*.gz          # stari rotirani — bezbedno
```

Za tekući slow log, koji ume da naraste na desetine gigabajta:

```bash
sudo truncate -s 0 /var/log/mysql/slow.log
mysql -e "FLUSH SLOW LOGS;"
```

Redosled je bitan — `truncate` pa `FLUSH`, da server ponovo otvori fajl.

**4. Sve što nije MySQL, a deli volumen**

```bash
sudo du -sh /var/* 2>/dev/null | sort -rh | head -10
sudo journalctl --vacuum-size=200M
sudo apt clean
```

**5. Hitno proširenje volumena**

Ako imate LVM ili cloud volumen, ovo je najbrže i najbezbednije:

```bash
sudo lvextend -L +50G /dev/vg_data/lv_mysql
sudo resize2fs /dev/vg_data/lv_mysql     # ext4
# ili
sudo xfs_growfs /var/lib/mysql           # XFS
```

Kod cloud provajdera: proširite volumen kroz konzolu, pa isto ove dve komande.

### Šta se ne dira

Tabela iz Modula 2, ovde skraćeno:

| Fajl | Sme? |
|---|---|
| `binlog.NNNNNN` | ✅ samo kroz `PURGE BINARY LOGS` |
| `ib_buffer_pool` | ✅ `rm` je bezbedan |
| rotirani `*.gz` logovi | ✅ |
| `ibtmp1` | ⚠️ ne `rm`; smanjuje se pri restartu |
| `.ibd` fajlovi | ❌ samo kroz `DROP TABLE` |
| `ibdata1`, `undo_*`, `#innodb_redo/` | ❌ nikad |
| `mysql.ibd` | ❌ nikad |

### Poseban slučaj: undo je narastao

Ako `du` pokazuje da `undo_001` ili `undo_002` zauzimaju desetine gigabajta, uzrok je duga otvorena transakcija koja sprečava čišćenje starih verzija redova (Modul 2).

```sql
SELECT trx_id, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS traje_sek,
       trx_mysql_thread_id
FROM information_schema.innodb_trx
ORDER BY trx_started LIMIT 5;
```

Prekinite je, pa proverite da je automatsko skraćivanje uključeno:

```sql
SELECT @@innodb_undo_log_truncate;
```

Prostor se oslobađa postepeno, nakon što transakcija nestane.

### Poseban slučaj: iscrpljeni inode-ovi

```bash
df -i /var/lib/mysql
```

Ako je iskorišćenost 100%, a `df -h` pokazuje slobodan prostor, greške izgledaju isto kao kod punog diska. Uzrok su hiljade sitnih fajlova — obično nije MySQL, nego nešto drugo na istom volumenu:

```bash
sudo find /var -xdev -type f | awk -F/ '{print $1"/"$2"/"$3}' | sort | uniq -c | sort -rn | head
```

### Nakon incidenta

```
1. Zašto monitoring nije javio na 80%? (Modul 6)
2. Koliki je binlog_expire_logs_seconds i da li je realan?
3. Da li binlogovi treba da idu na zaseban volumen?
4. Koliki je stvarni dnevni prirast? (Modul 9, praćenje rasta)
5. Da li backupi stoje na istom disku kao podaci?
```

---

## 50. Oštećenje podataka i `innodb_force_recovery`

### Prepoznavanje

```
[ERROR] [MY-012153] [InnoDB] Database page corruption on disk or a failed
file read of page [page id: space=42, page number=1289].
[ERROR] [MY-013183] [InnoDB] Assertion failure: ...
InnoDB: Failing assertion: ...
```

Tipično ponašanje: server startuje, pa padne. Restartuje se, pa opet padne. Petlja.

### Prvo: napravite hladnu kopiju

**Ovo nije opcion korak.** Sve što sledi može dodatno oštetiti podatke.

```bash
sudo systemctl stop mysql
sudo systemctl disable mysql          # da se ne podiže dok radite

sudo tar czf /backup/datadir-corrupt-$(date +%F_%H%M).tar.gz /var/lib/mysql
# ili, ako ima prostora:
sudo cp -a /var/lib/mysql /var/lib/mysql.corrupt
```

Zatim proverite šta imate od backupa:

```bash
ls -la /backup/mysql/
```

**Ako imate svež, proveren backup — restore je gotovo uvek bolji izbor od oporavka.** Oporavak oštećene instance je neizvestan i dugotrajan; restore je poznat postupak sa poznatim trajanjem (koje ste izmerili u Modulu 5).

`innodb_force_recovery` koristite kada backupa nema, kada je prestar, ili kada treba da izvučete podatke nastale nakon poslednjeg backupa.

### Nivoi

```ini
[mysqld]
innodb_force_recovery = 1
```

| Nivo | Šta radi | Rizik |
|---|---|---|
| **1** | ignoriše oštećene stranice i nastavlja | mali |
| **2** | ne pokreće pozadinske niti (purge) | mali |
| **3** | ne poništava nedovršene transakcije | srednji |
| **4** | ne spaja change buffer, ne računa statistiku | **može oštetiti** |
| **5** | tretira nedovršene transakcije kao potvrđene | **može oštetiti** |
| **6** | ne primenjuje redo log | **najveći rizik** |

**Pravilo: počnite od 1 i povećavajte samo ako server i dalje ne startuje.** Ne krećite od 6 "da bude sigurno" — to je najbrži način da nepovratno pokvarite podatke koji su se možda mogli spasiti.

Nivoi 4 i naviše mogu **trajno oštetiti** fajlove. Na tim nivoima cilj nije da server radi, nego da iz njega izvučete podatke.

Uz to: kada je `innodb_force_recovery` veći od nule, InnoDB **zabranjuje `INSERT`, `UPDATE` i `DELETE`**. Server je praktično u režimu za čitanje — što je namerno.

### Postupak

```bash
# 1. Postavite nivo 1
echo -e "[mysqld]\ninnodb_force_recovery = 1" | \
  sudo tee /etc/mysql/mysql.conf.d/99-recovery.cnf

sudo systemctl start mysql
```

Ako ne startuje, povećajte na 2, pa 3, i tako dalje. Nakon svakog pokušaja pročitajte log.

```bash
# 2. Kada startuje — ODMAH izvucite podatke
mysqldump --all-databases --routines --events --no-tablespaces \
  | gzip > /backup/spaseno-$(date +%F_%H%M).sql.gz
```

Ako `mysqldump` pada na određenoj tabeli, preskočite je i vratite se na nju kasnije:

```bash
mysqldump --all-databases --ignore-table=shop.ostecena_tabela ... > spaseno.sql
mysqldump shop ostecena_tabela > ostecena.sql   # zasebno, može pasti
```

Nalozi, jer nisu u dumpu podataka (Modul 3):

```bash
pt-show-grants > /backup/grants-spaseno.sql
```

Identifikacija oštećenih tabela:

```bash
mysqlcheck --all-databases --check
```

```sql
CHECK TABLE shop.narudzbine EXTENDED;
```

```bash
# 3. Napravite instancu iznova
sudo systemctl stop mysql
sudo rm /etc/mysql/mysql.conf.d/99-recovery.cnf
sudo mv /var/lib/mysql /var/lib/mysql.corrupt2
sudo mkdir /var/lib/mysql
sudo chown mysql:mysql /var/lib/mysql
sudo chmod 750 /var/lib/mysql

sudo mysqld --initialize --user=mysql
sudo systemctl start mysql
```

```bash
# 4. Vratite podatke
zcat /backup/spaseno-*.sql.gz | mysql
mysql < /backup/grants-spaseno.sql
```

**Nikada ne ostavljajte server da radi u produkciji sa uključenim `innodb_force_recovery`.** To nije rešenje, to je alat za izvlačenje podataka.

### Oštećenje jedne tabele

Ako je pogođena samo jedna tabela, a ostatak radi:

```sql
-- sa innodb_force_recovery = 1
```

```bash
mysqldump shop ostecena_tabela > /tmp/tabela.sql
```

```sql
-- bez force_recovery, na zdravom serveru
DROP TABLE shop.ostecena_tabela;
SOURCE /tmp/tabela.sql;
```

Ako se tabela ne može ni pročitati, vratite je iz poslednjeg backupa, pa primenite binlogove od tada za tu tabelu (Modul 5).

### Nađite pravi uzrok

Oštećenje InnoDB podataka nije normalna pojava. Ako se desilo, nešto konkretno nije u redu.

**Disk:**

```bash
sudo smartctl -a /dev/nvme0n1
sudo dmesg | grep -iE 'i/o error|ata|nvme|sd[a-z]'
```

**Memorija:**

```bash
sudo dmesg | grep -i "machine check\|mce\|edac"
# ozbiljnija provera:
sudo apt install memtester
sudo memtester 1G 3
```

Za pravu proveru je potreban `memtest86+` iz GRUB menija, sa isključenom mašinom.

**Fajl sistem:**

```bash
sudo dmesg | grep -iE 'ext4|xfs|remount-ro'
```

**Ostali uzroci:**

- `kill -9` nad `mysqld` tokom upisa (Modul 2 — zato se to ne radi),
- nestanak struje bez UPS-a, uz `innodb_flush_log_at_trx_commit != 1`,
- isključen doublewrite bafer,
- kopiranje `datadir`-a sa živog servera (Modul 5),
- mrežno skladište sa prekidima.

### Prevencija

```sql
SELECT @@innodb_doublewrite,              -- mora biti ON
       @@innodb_flush_log_at_trx_commit,  -- 1 za pun integritet
       @@innodb_checksum_algorithm;
```

Uz to: ECC memorija, UPS, praćenje SMART vrednosti, i redovan `CHECK TABLE` na replici.

---

## 51. OOM killer je ubio `mysqld`

### Prepoznavanje

Simptom: MySQL je nestao, a u error logu **nema** poruke o gašenju. Nema `Normal shutdown`, nema `Shutdown complete`. Samo prestanak, pa nov start.

```bash
sudo dmesg -T | grep -i "killed process"
sudo journalctl -k | grep -i "out of memory"
sudo grep -i "oom" /var/log/syslog | tail -20
```

Karakterističan zapis:

```
[Sun Aug 30 03:14:22 2026] Out of memory: Killed process 1234 (mysqld)
total-vm:34521844kB, anon-rss:31204812kB, file-rss:0kB, shmem-rss:0kB,
UID:114 pgtables:64512kB oom_score_adj:0
```

`anon-rss` je koliko je procesa zauzimao u trenutku ubistva. Uporedite sa ukupnim RAM-om.

Kernel obično ispisuje i tabelu svih procesa sa njihovom potrošnjom — pogledajte je, jer možda krivac uopšte nije MySQL:

```bash
sudo dmesg -T | grep -A 40 "Out of memory"
```

### Pet uzroka

**1. Buffer pool je prevelik**

Najčešći uzrok. Podsetnik iz Modula 7 — na serveru mora ostati prostora za operativni sistem, konekcije i rezervu.

```sql
SELECT @@innodb_buffer_pool_size / 1024 / 1024 / 1024 AS gb;
```

```bash
free -h
```

Ako je buffer pool preko 80% RAM-a, to je odgovor.

**2. Baferi po konekciji × broj konekcija**

```sql
SELECT @@sort_buffer_size, @@join_buffer_size, @@read_buffer_size,
       @@read_rnd_buffer_size, @@max_connections;
```

Klasična greška iz "tuning" članaka (Modul 7). Ako su ovi baferi u desetinama megabajta, a `max_connections` u stotinama — našli ste uzrok.

**3. Neko drugi na istom serveru**

```bash
ps aux --sort=-rss | head -10
```

Backup skript, aplikacija, indeksiranje, cron posao. Naročito ako se ubistvo dešava uvek u isto vreme.

```bash
sudo dmesg -T | grep "Killed process"
# uporedite vreme sa crontab i systemd timerima
systemctl list-timers
```

**4. Nema swap prostora uopšte**

```bash
free -h
swapon --show
```

Bez swap-a nema nikakvog amortizera. Kratak skok potrošnje odmah znači OOM.

Mali swap prostor (2–4 GB) sa `vm.swappiness = 1` (Modul 7) daje sistemu vremena da preživi kratkotrajne skokove. To nije poziv da se koristi swap — to je sigurnosna mreža.

**5. Cgroup ograničenje u kontejneru**

Ako MySQL radi u Dockeru ili Kubernetesu, ograničenje memorije je na nivou kontejnera, a MySQL vidi ceo RAM hosta:

```bash
cat /sys/fs/cgroup/memory.max
docker stats --no-stream
```

Buffer pool računajte prema ograničenju kontejnera, ne prema hostu.

### Šta uraditi

**Odmah:**

```bash
sudo systemctl start mysql
sudo tail -f /var/log/mysql/error.log
```

Očekujte crash recovery. Sačekajte (poglavlje 47).

**Zatim izmerite, ne pretpostavljajte:**

```bash
grep VmRSS /proc/"$(pgrep -x mysqld)"/status
```

```sql
SELECT event_name, ROUND(current_alloc / 1024 / 1024, 1) AS mb
FROM sys.memory_global_by_current_bytes
ORDER BY current_alloc DESC
LIMIT 15;
```

Ovo pokazuje **gde** memorija odlazi po podsistemima. Ako je nešto neočekivano visoko, tu je odgovor.

**Popravite dimenzionisanje** po računici iz Modula 7. Ovo je pravo rešenje.

**Ublažavanje, kao dodatak:**

```bash
sudo systemctl edit mysql
```

```ini
[Service]
OOMScoreAdjust=-600
```

Ovim smanjujete verovatnoću da kernel izabere `mysqld`. Ali budite iskreni: **ovo pomera problem, ne rešava ga.** Ako je memorija loše dimenzionisana, kernel će ubiti nešto drugo — možda aplikaciju, možda SSH.

**Provera swap-a:**

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
sysctl vm.swappiness   # treba da bude 1
```

### Postmortem

```
1. Koliko je mysqld zauzimao u trenutku ubistva? (anon-rss iz dmesg)
2. Koliko je ukupno RAM-a? Koliko je buffer pool?
3. Koliko je konekcija bilo aktivno? (monitoring, Modul 6)
4. Šta se još izvršavalo u tom trenutku? (cron, timeri, backup)
5. Da li je bilo alarma na potrošnju memorije?
6. Nova računica raspodele memorije, zapisana.
```

---

## Obrazac za postmortem

Nakon svakog ozbiljnijeg incidenta napišite ovo. Ne kao formalnost — kao način da se isti problem ne ponovi.

```
INCIDENT: <kratak naslov>
Datum:              2026-08-30
Trajanje:           03:14 – 04:02 (48 min)
Uticaj:             aplikacija nedostupna, ~1200 neuspelih zahteva
Prijavio:           monitoring / korisnik / slučajno otkriveno

ŠTA SE DESILO
  03:14  mysqld ubijen od strane OOM killera
  03:15  systemd podigao servis, započeo crash recovery
  03:41  server spreman za konekcije
  03:42  aplikacija i dalje javlja greške — pool nije obnovljen
  03:58  restart aplikacije
  04:02  potvrđen normalan rad

UZROK
  innodb_buffer_pool_size = 28G na serveru sa 32G RAM-a.
  Noćni mysqldump je alocirao dodatnu memoriju, ukupno je prešlo limit.

ŠTA JE PROŠLO DOBRO
  - systemd je automatski podigao servis
  - podaci su konzistentni, crash recovery je prošao bez greške
  - backup je bio ispravan

ŠTA NIJE PROŠLO DOBRO
  - nije postojao alarm na potrošnju memorije
  - aplikacija se nije sama oporavila nakon povratka baze
  - 26 minuta crash recovery-ja niko nije pratio — mislili smo da je zaglavljeno

MERE
  [ ] buffer pool 28G → 20G                         (odgovoran, do 2026-09-01)
  [ ] dodati 4G swap sa vm.swappiness=1             (odgovoran, do 2026-09-01)
  [ ] alarm na memoriju > 85%                       (odgovoran, do 2026-09-03)
  [ ] OOMScoreAdjust=-600                           (odgovoran, do 2026-09-01)
  [ ] backup pomeriti van špica i dati mu Nice=10   (odgovoran, do 2026-09-05)
  [ ] aplikacija: automatsko obnavljanje poola      (developerski tim)
  [ ] dopuniti runbook: kako izgleda normalan
      crash recovery i zašto se ne prekida          (odgovoran, do 2026-09-10)
```

Poslednja stavka je tipična. Polovina vremena u incidentu često ode na neizvesnost oko toga da li nešto radi ili je zaglavljeno — a to se rešava jednim pasusom u dokumentaciji.

---

## Kontrolna lista pripremljenosti

Ovo proverite **pre** nego što zatreba:

```bash
# 1. Administrativni port je uključen
mysql -e "SELECT @@admin_port;" 2>/dev/null || echo "NIJE — uključi ga"

# 2. Skript za trijažu postoji (Modul 6)
ls -la /usr/local/bin/mysql-trijaza.sh

# 3. Znate gde je error log
mysql -e "SELECT @@log_error;"

# 4. Backup je proveren restore-om i RTO je izmeren
tail -3 /var/log/test-restore.log

# 5. Monitoring pokriva: dostupnost, disk, memoriju, konekcije,
#    replikaciju, starost backupa

# 6. Postoji swap i vm.swappiness = 1
swapon --show; sysctl vm.swappiness

# 7. Znate koliko traje crash recovery na ovom serveru
#    (izmereno u vežbi, ne pretpostavljeno)

# 8. Runbook postoji IZVAN ovog servera

# 9. Znate ko je zamena kada vas nema

# 10. Kontakt developerskog tima je dostupan i van radnog vremena
```

---

## Vežbe

Sve vežbe radite na test mašini koju možete da bacite.

**Vežba 1 — Pet načina da server ne startuje**
Izazovite redom: nepoznatu promenljivu u konfiguraciji, pogrešno vlasništvo `datadir`-a, zauzet port 3306, drugu instancu nad istim `datadir`-om, i pun disk. Za svaki slučaj pronađite tačnu poruku i vreme koje vam je trebalo. Napravite sopstvenu tabelu simptoma.

**Vežba 2 — Start u prvom planu**
Pokvarite konfiguraciju tako da server padne pre nego što otvori error log. Pokušajte dijagnostiku samo preko `systemctl status`, pa preko `journalctl`, pa preko `mysqld --console`. Uporedite koliko informacija daje svaki pristup.

**Vežba 3 — Too many connections, oba scenarija**
Napravite scenario A (skript sa sporim upitima koji se gomilaju) i scenario B (skript koji otvara konekcije i ne zatvara ih). Za svaki utvrdite tip iz `Threads_running` i primenite odgovarajuću reakciju. Izmerite koliko vam je trebalo da razlikujete scenarije.

**Vežba 4 — Ulazak kada je limit iscrpljen**
Iscrpite `max_connections`. Pokušajte da se povežete običnim putem, pa preko `sudo mysql`, pa preko administrativnog porta. Zabeležite šta radi a šta ne. Zatim uključite `admin_port` i ponovite.

**Vežba 5 — Pun disk, ceo ciklus**
Napunite volumen sa `fallocate`. Posmatrajte kako se MySQL ponaša — potvrdite da čeka, a ne pada. Zatim oslobodite prostor kroz `PURGE BINARY LOGS` i potvrdite da se rad nastavlja bez restarta.

**Vežba 6 — Obrisani fajl koji drži prostor**
Napravite veliki slow log, pa ga obrišite sa `rm` dok MySQL radi. Uporedite `df -h` i `du -sh`. Pronađite fajl sa `lsof +L1` i oslobodite prostor ispravno.

**Vežba 7 — `innodb_force_recovery`**
Napravite backup, pa namerno oštetite `.ibd` fajl (npr. `dd` preko sredine fajla). Pokušajte oporavak od nivoa 1 naviše, izvucite podatke, izgradite instancu iznova i vratite ih. Izmerite ukupno trajanje i uporedite sa običnim restore-om iz backupa.

**Vežba 8 — OOM killer**
Postavite buffer pool na 90% RAM-a bez swap-a i opteretite server. Sačekajte ubistvo procesa, pa pronađite zapis u `dmesg` i pročitajte `anon-rss`. Zatim ispravite dimenzionisanje, dodajte swap i ponovite isto opterećenje.

**Vežba 9 — Vežba incidenta bez najave**
Zamolite kolegu da na test serveru izazove problem po sopstvenom izboru, bez da vam kaže koji. Dijagnostikujte i rešite koristeći isključivo svoj runbook i skript za trijažu. Zatim napišite postmortem po obrascu iz ovog modula — uključujući iskren odeljak o tome šta u dokumentaciji nije valjalo.

---

## Šta sledi

U **Modulu 11**, poslednjem, prelazimo na moderno okruženje i automatizaciju.

MySQL u Docker i Podman kontejnerima — volumeni, konfiguracija, i iskren razgovor o tome kada ovo ne raditi u produkciji. Ansible role koja radi sve što smo kroz kurs radili ručno: instalaciju, konfiguraciju, naloge, backup. Bezbedno rukovanje lozinkama u skriptama, `.my.cnf`, `mysql_config_editor` i alternative. I na kraju, managed baze kod cloud provajdera — šta prestaje da bude vaš posao, a šta, uprkos svemu, ostaje.

Nakon toga sledi dodatak sa minimumom SQL-a koji vam je zaista potreban.
