# Modul 6: Logovi i monitoring

*MySQL za Linux administratore*

---

## Uvod: da problem primetite pre nego što zazvoni telefon

U prethodnom modulu smo obezbedili da se od katastrofe možete oporaviti. Ovaj modul je o tome kako da do katastrofe uopšte ne dođe — ili bar kako da je primetite dok je još sitnica.

Postoje tri nivoa na kojima administrator može da radi:

1. **Reaktivno.** Neko javi da aplikacija ne radi, vi krenete da tražite uzrok.
2. **Detektivno.** Monitoring vas obavesti da nešto nije u redu, pre nego što korisnici primete.
3. **Preventivno.** Vidite trend i rešite problem pre nego što postane incident.

Ovaj modul vas vodi sa prvog na treći nivo. Alat je isti — logovi i metrike — razlika je u tome da li ih gledate tek kada gori.

Podsetnik na podelu odgovornosti iz Modula 1: vaš posao je da **pronađete** spor upit i **prosledite** ga onome ko ume da ga popravi. Ne morate umeti da optimizujete SQL. Morate umeti da za pet minuta kažete: "ovaj upit se izvršava 40.000 puta na sat i troši 60% ukupnog vremena na bazi, evo ga."

---

## 27. Četiri MySQL loga

MySQL vodi četiri odvojena loga, plus jedan koji se javlja samo na replikama. Svaki služi drugoj svrsi i svaki se drugačije podešava.

| Log | Čemu služi | Podrazumevano | Cena |
|---|---|---|---|
| **Error log** | greške, upozorenja, start/stop | uključen | zanemarljiva |
| **Slow query log** | spori upiti | isključen | mala |
| **General log** | svaki upit | isključen | **velika** |
| **Binary log** | replikacija, PITR | uključen (8.0) | srednja |
| *Relay log* | *samo na replikama* | *automatski* | *srednja* |

Provera trenutnog stanja svega odjednom:

```sql
SELECT @@log_error, @@log_error_verbosity, @@log_timestamps;
SELECT @@slow_query_log, @@slow_query_log_file, @@long_query_time;
SELECT @@general_log, @@general_log_file, @@log_output;
SELECT @@log_bin, @@log_bin_basename, @@binlog_expire_logs_seconds;
```

### Error log

Najvažniji log i najmanje čitan. Ovde piše zašto server nije startovao, zašto je pao, i šta ga muči dok radi.

```sql
SELECT @@log_error;
```

Ako je vrednost putanja do fajla — čitate fajl. Ako je `stderr`, poruke idu u journald:

```bash
sudo tail -f /var/log/mysql/error.log
# ili
journalctl -u mysql -f
```

**Nivo detaljnosti:**

```sql
SELECT @@log_error_verbosity;
```

- **1** — samo greške,
- **2** — greške i upozorenja (podrazumevano),
- **3** — greške, upozorenja i beleške.

Ostavite na 2. Nivo 3 je koristan dok dijagnostikujete konkretan problem, ali stalno uključen pravi šum u kome se prava greška izgubi.

### `log_timestamps` — podesite ovo odmah

```sql
SELECT @@log_timestamps;
```

Podrazumevana vrednost je **UTC**. To znači da vaš error log koristi drugu vremensku zonu od `syslog`-a, aplikacionog loga i svega ostalog na serveru.

Posledica se oseti tačno u trenutku kada vam najmanje treba: usred incidenta pokušavate da povežete događaje iz tri loga i računate razliku od dva sata u glavi.

```ini
[mysqld]
log_timestamps = SYSTEM
```

Ovo je dinamička promenljiva, pa možete i odmah:

```sql
SET GLOBAL log_timestamps = 'SYSTEM';
SET PERSIST log_timestamps = 'SYSTEM';
```

Uz podsetnik iz Modula 2 — ako koristite `SET PERSIST`, znajte da to više neće biti vidljivo u `/etc/mysql`.

### General log — moćan i opasan

Beleži **svaki** upit koji stigne do servera, uključujući neuspele prijave i sistemske upite.

```sql
SET GLOBAL general_log = ON;
SELECT @@general_log_file;
```

Ovo je izuzetno korisno kada dijagnostikujete "šta aplikacija zapravo šalje bazi". Ali:

- fajl raste **brzo** — na aktivnom serveru gigabajtima na sat,
- svaki upit se upisuje na disk, što košta performansi,
- lozinke i lični podaci završavaju u čitljivom obliku u fajlu.

**Pravilo: uključite, uhvatite šta vam treba, isključite. Minutima, ne satima.**

```sql
SET GLOBAL general_log = ON;
-- ... reprodukujete problem ...
SET GLOBAL general_log = OFF;
```

Umesto u fajl, izlaz može ići u tabelu, što je zgodno za filtriranje:

```sql
SET GLOBAL log_output = 'TABLE';
SET GLOBAL general_log = ON;

SELECT event_time, user_host, argument
FROM mysql.general_log
WHERE argument LIKE '%narudzbine%'
ORDER BY event_time DESC LIMIT 50;

SET GLOBAL general_log = OFF;
TRUNCATE TABLE mysql.general_log;
```

Ne zaboravite `TRUNCATE` — tabela ostaje na disku.

### Binary log

Detaljno obrađen u Modulima 2 i 5. Kratko podsećanje:

```sql
SHOW BINARY LOGS;
SELECT @@binlog_format, @@binlog_expire_logs_seconds, @@sync_binlog;
```

Za ovaj modul bitno je samo da je **veličina binlogova metrika koju treba pratiti** — to je najčešći uzrok naglog punjenja diska.

### Relay log

Postoji samo na replikama i sadrži događaje preuzete sa primarnog servera pre nego što budu primenjeni. Detalji u Modulu 8.

---

## 28. Čitanje error loga

### Anatomija linije

MySQL 8 format:

```
2026-08-30T14:23:11.482910+02:00  0 [Warning] [MY-010055] [Server] IP address '10.0.1.77' could not be resolved
```

Delovi:

| Deo | Značenje |
|---|---|
| `2026-08-30T14:23:11.482910+02:00` | vreme (u SYSTEM zoni, jer smo je podesili) |
| `0` | ID niti; **0 znači sistemska poruka**, ne korisnička konekcija |
| `[Warning]` | nivo: `System`, `Note`, `Warning`, `ERROR` |
| `[MY-010055]` | **kod greške** — po njemu se najbolje pretražuje |
| `[Server]` | podsistem: `Server`, `InnoDB`, `Repl`, `Xtrabackup`... |
| ostatak | poruka |

**Kod greške je najkorisniji deo.** Kada pretražujete internet, tražite `MY-010055`, ne tekst poruke — tekst se menja između verzija, kod ne.

### Normalne poruke — da ih ne biste tražili uzalud

Ove poruke znače da je sve u redu:

```
[System] [MY-010116] [Server] /usr/sbin/mysqld (mysqld 8.0.36) starting as process 1234
[System] [MY-013576] [InnoDB] InnoDB initialization has started.
[System] [MY-013577] [InnoDB] InnoDB initialization has ended.
[System] [MY-010931] [Server] /usr/sbin/mysqld: ready for connections.
```

Poslednja linija je ono što čekate kad start "visi" (Modul 2). Kada se pojavi, server prima konekcije.

Uredno gašenje:

```
[System] [MY-013105] [Server] /usr/sbin/mysqld: Normal shutdown.
[System] [MY-010910] [Server] /usr/sbin/mysqld: Shutdown complete
```

**Ako u logu nema linije o urednom gašenju, a server se ponovo startovao — server je pao ili je ubijen.** To je važan trag.

### Oporavak nakon pada

```
[Note] [MY-012560] [InnoDB] The log sequence number 12345678 in the system tablespace does not match...
[Note] [MY-013083] [InnoDB] Log background threads are being started...
[Note] [MY-012532] [InnoDB] Applying a batch of 4521 redo log records...
[Note] [MY-012533] [InnoDB] 10%  20%  30%  ...
[Note] [MY-012535] [InnoDB] Apply batch completed!
```

Ovo je crash recovery iz Modula 2. Procenti se pomeraju — server radi, sačekajte.

### Poruke koje traže vašu reakciju

**Prekinute konekcije**

```
[Note] [MY-010914] [Server] Aborted connection 4521 to db: 'shop' user: 'app'
host: '10.0.1.50' (Got timeout reading communication packets)
```

Ovo je najčešća poruka koju ćete videti i uzroci su tri:

1. **`wait_timeout`** — aplikacija drži neaktivnu konekciju duže od dozvoljenog i server je zatvara. Ako se pojavljuje redovno u pravilnim razmacima, ovo je uzrok.
2. **Aplikacija ne zatvara konekcije uredno.** Connection pool koji se gasi bez `close()`.
3. **Mrežni problem** — pravi, ako se poklapa sa drugim mrežnim simptomima.

```sql
SELECT @@wait_timeout, @@interactive_timeout, @@net_read_timeout;
SHOW GLOBAL STATUS LIKE 'Aborted_%';
```

`Aborted_connects` broji neuspele **prijave** (pogrešna lozinka, nepostojeći nalog) — to je bezbednosni signal. `Aborted_clients` broji konekcije prekinute nakon uspešne prijave — to je uglavnom aplikacioni problem.

Umerena količina ovih poruka je normalna. Nagli porast nije.

**Previše konekcija**

```
[ERROR] [MY-010913] [Server] Too many connections
```

Odmah reagujte (Modul 10), a uzrok obično nije `max_connections`, nego upiti koji se ne završavaju.

**Permission denied**

```
[ERROR] [MY-012592] [InnoDB] Operating system error number 13 in a file operation.
[ERROR] [MY-012595] [InnoDB] The error means mysqld does not have the access rights to the directory.
```

Errno 13. Ako `ls -l` pokazuje ispravna prava — to je AppArmor (Modul 2).

**Druga instanca već radi**

```
[ERROR] [MY-012574] [InnoDB] Unable to lock ./ibdata1 error: 11
```

Neko je već pokrenuo `mysqld` nad istim `datadir`-om, ili prethodni proces nije umro.

```bash
pgrep -a mysqld
```

**Neuspelo razrešavanje imena**

```
[Warning] [MY-010055] [Server] IP address '10.0.1.77' could not be resolved
```

DNS. Rešenje je `skip_name_resolve = ON` iz Modula 3, uz prethodnu proveru da nijedan nalog nema ime hosta u host polju.

**Pun disk**

```
[ERROR] [MY-011971] [InnoDB] Disk is full writing './shop/narudzbine.ibd'
[ERROR] [MY-011972] [InnoDB] Retry attempt is ended.
```

MySQL ne puca odmah — **čeka** da neko oslobodi prostor, u petlji. Aplikacija za to vreme stoji. Postupak u Modulu 10.

**Zaključavanja**

```
[Note] Deadlock found when trying to get lock; try restarting transaction
[Note] Lock wait timeout exceeded; try restarting transaction
```

Deadlock je normalna pojava u konkurentnim sistemima i aplikacija bi trebalo da ga rešava ponavljanjem transakcije. Ako ih ima mnogo, to je aplikacioni problem koji prosleđujete developeru — sa dokazom iz `SHOW ENGINE INNODB STATUS` (poglavlje 32).

### Praktičan pregled loga

Prvo što uradite na nepoznatom serveru:

```bash
# Samo greške, poslednjih 200
sudo grep -E '\[ERROR\]' /var/log/mysql/error.log | tail -200

# Šta se dešavalo oko određenog vremena
sudo grep '2026-08-30T14:' /var/log/mysql/error.log

# Da li je server ikada pao — traži start bez prethodnog urednog gašenja
sudo grep -E 'ready for connections|Shutdown complete|Normal shutdown' \
  /var/log/mysql/error.log | tail -40

# Najčešće greške, grupisano po kodu
sudo grep -oP '\[MY-\d+\]' /var/log/mysql/error.log | sort | uniq -c | sort -rn | head -20
```

Poslednja komanda je posebno korisna kod nasleđenog servera — za pet sekundi vidite šta ovaj server stvarno muči.

### Skript za brzu trijažu

```bash
#!/bin/bash
# /usr/local/bin/mysql-trijaza.sh
LOG=$(mysql -N -B -e "SELECT @@log_error;")

echo "=== Poslednjih 20 grešaka ==="
sudo grep '\[ERROR\]' "$LOG" | tail -20

echo
echo "=== Startovanja i gašenja (poslednjih 20) ==="
sudo grep -E 'ready for connections|Shutdown complete' "$LOG" | tail -20

echo
echo "=== Najčešći kodovi grešaka ==="
sudo grep -oP '\[MY-\d+\]' "$LOG" | sort | uniq -c | sort -rn | head -10

echo
echo "=== Uptime i status ==="
mysql -e "SHOW GLOBAL STATUS LIKE 'Uptime';
          SHOW GLOBAL STATUS LIKE 'Aborted_%';
          SHOW GLOBAL STATUS LIKE 'Threads_%';"
```

---

## 29. Slow query log i `pt-query-digest`

### Uključivanje

Slow query log je podrazumevano isključen, a kada se uključi, podrazumevani prag od 10 sekundi je praktično beskoristan — upit koji traje 10 sekundi je već katastrofa, ne "spor".

```ini
[mysqld]
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
log_slow_extra = ON
```

Dinamički, bez restarta:

```sql
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;
SET GLOBAL log_slow_extra = ON;
```

`log_slow_extra` (MySQL 8.0.14+) dodaje korisna polja: broj pročitanih redova, korišćeni indeksi, privremene tabele.

**Prag postavite prema svojoj aplikaciji.** Za web aplikaciju je 1 sekunda razuman početak. Ako je log prazan, spuštajte: 0.5, pa 0.2. Ako je pun, dižite.

Za kratku, ciljanu analizu možete uhvatiti **sve** upite:

```sql
SET GLOBAL long_query_time = 0;
-- ... nekoliko minuta ...
SET GLOBAL long_query_time = 1;
```

Ovo pravi ogroman fajl, ali daje potpunu sliku. Radite kratko i pratite prostor na disku.

### Opcija koju treba koristiti oprezno

```sql
SET GLOBAL log_queries_not_using_indexes = ON;
```

Zvuči korisno, ali generiše ogroman šum: mala tabela od 50 redova legitimno se čita bez indeksa i to nije problem. Ako je koristite, obavezno uz ograničenje:

```sql
SET GLOBAL log_throttle_queries_not_using_indexes = 10;
SET GLOBAL min_examined_row_limit = 1000;
```

Drugi parametar je zapravo bolje rešenje samostalno — beleži samo upite koji su pregledali bar 1000 redova, bez obzira na trajanje.

### Kako izgleda zapis

```
# Time: 2026-08-30T14:23:11.482910+02:00
# User@Host: app[app] @  [10.0.1.50]  Id: 4521
# Query_time: 3.482910  Lock_time: 0.000123  Rows_sent: 1  Rows_examined: 2451822
# Thread_id: 4521  Errno: 0  Killed: 0  Bytes_received: 142  Bytes_sent: 89
# Read_first: 1  Read_last: 0  Read_key: 0  Read_next: 2451822
SET timestamp=1788352991;
SELECT COUNT(*) FROM narudzbine WHERE status = 'na_cekanju';
```

Najvažniji odnos u celom zapisu:

```
Rows_sent: 1        Rows_examined: 2451822
```

Server je pregledao dva i po miliona redova da bi vratio jedan. To je gotovo uvek nedostajući indeks i to je nalaz koji prosleđujete developeru.

### `pt-query-digest`

Sirovi slow log od nekoliko stotina megabajta ne čita se ručno. `pt-query-digest` iz Percona Toolkit-a ga sažima.

```bash
sudo apt install percona-toolkit

pt-query-digest /var/log/mysql/slow.log > /tmp/analiza.txt
less /tmp/analiza.txt
```

### Kako se čita izveštaj

Zaglavlje:

```
# 412.4s user time, 2.1s system time, 41.23M rss
# Overall: 84.51k total, 142 unique, 23.48 QPS, 1.42x concurrency
# Time range: 2026-08-30T00:00:12 to 2026-08-30T01:00:03
```

Zatim rang-lista, koja je suština alata:

```
# Profile
# Rank Query ID           Response time   Calls  R/Call  V/M   Item
# ==== ================== =============== ====== ======= ===== ==============
#    1 0xB5A3C9E1F2D4...  1842.4821 61.2%  41982  0.0439  0.01  SELECT narudzbine
#    2 0x7F2E8A1C4B9D...   482.1029 16.0%     12 40.1752  2.31  SELECT izvestaji
#    3 0x1A9C3E7B2F8D...   201.3847  6.7%  28471  0.0071  0.00  SELECT korisnici
```

**Ključna pouka o čitanju ovog izveštaja:**

Upit broj 1 traje 0,04 sekunde po pozivu. Deluje brzo. Ali izvršava se 42.000 puta i troši **61% ukupnog vremena** servera.

Upit broj 2 traje 40 sekundi po pozivu. Deluje strašno. Ali se izvršava 12 puta i troši 16%.

**Prvo popravite broj 1.** Sortirajte po ukupnom vremenu, ne po vremenu izvršavanja. To je razlika između administratora koji gasi požare i onoga koji rešava probleme.

Kolona `V/M` je varijansa. Visoka vrednost znači da isti upit ponekad radi brzo, ponekad sporo — obično zbog zaključavanja ili keširanja, a ne zbog samog upita.

Detalji po upitu, niže u izveštaju:

```
# Query 1: 11.66 QPS, 0.51x concurrency, ID 0xB5A3C9E1F2D4 at byte 4829105
# Attribute    pct   total     min     max     avg     95%  median
# ============ === ======= ======= ======= ======= ======= =======
# Exec time     61   1842s    12ms    891ms    43ms    98ms    38ms
# Rows examine  94  102.9G   1.19k   3.42M   2.45M   2.93M   2.35M
# Rows sent      0   41.98k       1       1       1       1       1
```

Red `Rows examine` je dokaz. 102 milijarde pregledanih redova za 42 hiljade vraćenih. To je nalaz koji ne trpi raspravu.

### Korisne opcije

```bash
# samo poslednji sat
pt-query-digest --since '1h' /var/log/mysql/slow.log

# samo prvih 5 upita
pt-query-digest --limit 5 /var/log/mysql/slow.log

# samo određena baza
pt-query-digest --filter '$event->{db} eq "shop"' /var/log/mysql/slow.log

# uživo, bez slow loga — uzorkovanje process liste
pt-query-digest --processlist h=localhost --interval 0.1 --run-time 5m
```

Poslednja varijanta je korisna kada slow log nije uključen a problem se dešava sada.

### Alternativa bez slow loga: `sys` šema

MySQL 8 ima `performance_schema` uključen podrazumevano, a `sys` šema nudi gotove poglede nad njim. Za brz uvid ne treba vam nikakav log:

```sql
-- upiti po ukupnom utrošenom vremenu
SELECT query, exec_count, total_latency, avg_latency, rows_examined_avg, rows_sent_avg
FROM sys.statement_analysis
ORDER BY total_latency DESC
LIMIT 10;

-- upiti koji pregledaju mnogo, a vraćaju malo
SELECT query, exec_count, rows_examined_avg, rows_sent_avg
FROM sys.statement_analysis
WHERE rows_sent_avg > 0
ORDER BY rows_examined_avg / rows_sent_avg DESC
LIMIT 10;

-- tabele bez korišćenja indeksa
SELECT * FROM sys.schema_tables_with_full_table_scans LIMIT 10;

-- neiskorišćeni indeksi (kandidati za brisanje)
SELECT * FROM sys.schema_unused_indexes;
```

Prednost: nema fajla, nema rotacije, radi odmah. Mana: statistika se resetuje pri restartu servera i ne sadrži konkretne vrednosti parametara, nego normalizovane oblike upita.

Praktično: `sys` šema za svakodnevni uvid, slow log i `pt-query-digest` za ozbiljnu analizu.

### Šta raditi sa nalazom

Vaš izveštaj developeru treba da izgleda otprilike ovako:

```
Upit koji troši 61% vremena baze:

  SELECT COUNT(*) FROM narudzbine WHERE status = 'na_cekanju';

  Izvršava se: 42.000 puta na sat
  Prosečno: 43 ms
  Pregleda: 2,45 miliona redova po pozivu
  Vraća: 1 red

  Tabela narudzbine ima 2,45M redova i nema indeks na koloni status.

Prilog: pt-query-digest izveštaj za period 00:00–01:00.
```

To je vaš deo posla, i on je odrađen. Da li će rešenje biti indeks, keširanje ili promena logike — nije vaša odluka.

---

## 30. Rotacija logova

### Zašto nije trivijalno

MySQL drži **otvoren handle** na svoj log fajl. Ako `logrotate` preimenuje fajl, MySQL nastavi da piše u isti inode — sada pod novim imenom, ili u obrisan fajl koji zauzima prostor a ne postoji u direktorijumu.

Simptom: `du` pokazuje da je disk pun, `ls` ne pokazuje veliki fajl. Provera:

```bash
sudo lsof -p "$(pgrep -x mysqld)" | grep -E 'deleted|\.log'
```

Zato rotacija mora da **kaže MySQL-u da ponovo otvori fajlove.**

### Kako to reći MySQL-u

```sql
FLUSH ERROR LOGS;
FLUSH SLOW LOGS;
FLUSH GENERAL LOGS;
```

**Izbegavajte golo `FLUSH LOGS`.** Ono radi sve navedeno **i rotira binarni log**. Dnevna rotacija koja usput pravi nov binlog fajl svaki dan nije katastrofa, ali je nepredviđen sporedni efekat i može zbuniti kada računate PITR prozor (Modul 5).

### Ubuntu konfiguracija

Paket isporučuje `/etc/logrotate.d/mysql-server`:

```bash
cat /etc/logrotate.d/mysql-server
```

Prilagođena verzija:

```
/var/log/mysql/error.log /var/log/mysql/slow.log {
    daily
    rotate 14
    missingok
    notifempty
    create 640 mysql adm
    compress
    delaycompress
    sharedscripts
    postrotate
        if [ -x /usr/bin/mysqladmin ]; then
            /usr/bin/mysqladmin --defaults-file=/etc/mysql/debian.cnf \
                flush-error-log flush-slow-log 2>/dev/null || true
        fi
    endscript
}
```

Objašnjenje bitnih delova:

- **`create 640 mysql adm`** — nov fajl mora biti u vlasništvu korisnika `mysql`, inače MySQL ne može da piše u njega. Ovo je najčešća greška u ručno pisanim konfiguracijama.
- **`delaycompress`** — komprimuje tek u sledećem ciklusu, da ne bi komprimovao fajl u koji se možda još piše.
- **`sharedscripts`** — `postrotate` se izvršava jednom, ne po fajlu.
- **`--defaults-file=/etc/mysql/debian.cnf`** — koristi održavateljski nalog iz Modula 1, pa lozinka nije u konfiguraciji `logrotate`-a.
- **`|| true`** — ako `mysqladmin` ne uspe, `logrotate` ne prekida ostatak posla.

### Testiranje

**Uvek testirajte pre nego što se oslonite na rotaciju.**

```bash
# Suvo pokretanje — pokazuje šta bi uradio
sudo logrotate -d /etc/logrotate.d/mysql-server

# Prinudno izvršavanje
sudo logrotate -vf /etc/logrotate.d/mysql-server

# Provera rezultata
ls -la /var/log/mysql/
sudo lsof -p "$(pgrep -x mysqld)" | grep '\.log'
```

Nakon rotacije, `lsof` mora pokazivati **nove** fajlove, bez oznake `(deleted)`.

Provera da rotacija uopšte radi na duže staze:

```bash
ls -la /var/log/mysql/*.gz
```

Ako ovde nema ničega a server radi mesecima — rotacija ne radi. Najčešći uzrok je da `mysqladmin` ne može da se prijavi, pa `postrotate` tiho pada.

### Alternativa: `copytruncate`

```
    copytruncate
    nocreate
```

Kopira fajl pa skrati original na nulu, bez ponovnog otvaranja. Radi bez pristupa MySQL-u.

Mana: linije upisane između kopiranja i skraćivanja se gube. Za error log je to prihvatljivo, za bilo šta gde tačnost ima značaja nije. Koristite kao rezervnu opciju kada `mysqladmin` iz nekog razloga nije dostupan.

### Ograničenje veličine kao zaštita

`logrotate` radi jednom dnevno. Ako slow log naraste za dan više nego što disk može da podnese, to je kasno.

```
    size 500M
```

Uz `cron.hourly` pokretanje, ovim se rotacija dešava čim fajl pređe 500 MB, ne čekajući ponoć.

---

## 31. Monitoring bez DBA

### Šta zapravo pratiti

Postoji stotine MySQL metrika. Devedeset odsto vam ne treba. Evo šta treba.

**Prva kategorija — dostupnost.**

| Metrika | Značenje | Alarm |
|---|---|---|
| `mysql_up` | server odgovara | odmah, kritično |
| `Uptime` | vreme od starta | **nagli pad = server se restartovao** |

Druga stavka se često previdi. Ako `Uptime` iznenada padne sa 4.000.000 na 300, server je pao i podigao se — a možda niko nije primetio, jer je nedostupnost trajala dva minuta u tri ujutru. Alarm na nagli pad `Uptime`-a otkriva probleme koji bi inače prošli neopaženo.

**Druga kategorija — konekcije.**

```sql
SHOW GLOBAL STATUS WHERE Variable_name IN
  ('Threads_connected','Threads_running','Max_used_connections',
   'Aborted_connects','Aborted_clients','Connection_errors_max_connections');
SELECT @@max_connections;
```

| Metrika | Šta znači |
|---|---|
| `Threads_connected` | trenutno otvorenih konekcija |
| `Threads_running` | konekcija koje **stvarno rade** |
| `Max_used_connections` | najviša vrednost od starta |
| `Aborted_connects` | neuspele prijave (bezbednosni signal) |
| `Connection_errors_max_connections` | broj odbijenih zbog limita |

**`Threads_running` je važnija metrika od `Threads_connected`.** Aplikacija sa poolom može držati 200 otvorenih konekcija od kojih 198 spava — to je normalno. Ali ako `Threads_running` skoči iznad broja jezgara i tu ostane, server se guši i to korisnici osećaju.

Alarm: `Threads_connected / max_connections > 0.8`.

**Treća kategorija — InnoDB buffer pool.**

```sql
SHOW GLOBAL STATUS WHERE Variable_name IN
  ('Innodb_buffer_pool_reads','Innodb_buffer_pool_read_requests',
   'Innodb_buffer_pool_pages_dirty','Innodb_buffer_pool_pages_total');
```

Odnos `Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests` je udeo čitanja koja su morala na disk. Što manji, to bolje.

**Ali pazite: ne postavljajte alarm na apsolutnu vrednost ovog odnosa.** Zdrav server može imati različite vrednosti u zavisnosti od radnog opterećenja. Pratite **trend** — nagli pad je signal.

**Četvrta kategorija — disk.**

```bash
df -h /var/lib/mysql
```

Ovo je vaša najvažnija metrika u celom spisku. Pun disk je najčešći uzrok ispada MySQL-a i jedini koji se sa sigurnošću može predvideti.

Alarmi: upozorenje na 80%, kritično na 90%. I obavezno posebno pratite veličinu binlogova:

```sql
SELECT SUM(file_size) / 1024 / 1024 / 1024 AS binlog_gb FROM
  (SELECT @@log_bin_basename) x, information_schema.files
WHERE 1=0;  -- jednostavnije preko fajl sistema:
```

```bash
sudo du -sh /var/lib/mysql/binlog.*
```

**Peta kategorija — zaključavanja i sporost.**

```sql
SHOW GLOBAL STATUS WHERE Variable_name IN
  ('Slow_queries','Innodb_row_lock_waits','Innodb_row_lock_time_avg',
   'Table_locks_waited','Created_tmp_disk_tables');
```

`Created_tmp_disk_tables` je koristan pokazatelj: privremene tabele koje ne staju u memoriju idu na disk i to je sporo. Nagli porast znači da je neko dodao upit koji sortira mnogo podataka.

**Šesta kategorija — replikacija.** (Detaljno u Modulu 8.)

```sql
SHOW REPLICA STATUS\G
```

Pratite: `Replica_IO_Running`, `Replica_SQL_Running`, `Seconds_Behind_Source`, `Last_Error`.

**Sedma kategorija — stvari iz prethodnih modula.**

Ovo se najčešće zaboravi, a najviše boli:

- **starost poslednjeg uspešnog backupa** (Modul 5),
- **datum isteka TLS sertifikata** (Modul 4),
- **broj otvorenih fajl deskriptora prema limitu** (Modul 2).

### `mysqld_exporter` za Prometheus

```bash
# preuzmite najnoviju verziju sa GitHub stranice projekta
sudo useradd -r -s /bin/false mysqld_exporter
sudo mv mysqld_exporter /usr/local/bin/
sudo chown mysqld_exporter:mysqld_exporter /usr/local/bin/mysqld_exporter
```

Nalog iz Modula 3:

```sql
CREATE USER 'exporter'@'localhost' IDENTIFIED BY '...' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT ON *.* TO 'exporter'@'localhost';
GRANT SELECT ON performance_schema.* TO 'exporter'@'localhost';
```

```bash
sudo tee /etc/mysqld_exporter.cnf > /dev/null <<'EOF'
[client]
user = exporter
password = ...
host = localhost
EOF

sudo chown mysqld_exporter:mysqld_exporter /etc/mysqld_exporter.cnf
sudo chmod 600 /etc/mysqld_exporter.cnf
```

```ini
# /etc/systemd/system/mysqld_exporter.service
[Unit]
Description=Prometheus MySQL Exporter
After=mysql.service

[Service]
User=mysqld_exporter
ExecStart=/usr/local/bin/mysqld_exporter \
  --config.my-cnf=/etc/mysqld_exporter.cnf \
  --web.listen-address=127.0.0.1:9104 \
  --collect.info_schema.tables \
  --collect.info_schema.innodb_metrics \
  --collect.global_status \
  --collect.global_variables \
  --collect.slave_status
Restart=always

[Install]
WantedBy=multi-user.target
```

Uočite `--web.listen-address=127.0.0.1:9104`. Exporter izlaže detalje o vašoj bazi i ne treba da bude dostupan sa mreže. Prometheus mu pristupa kroz tunel ili preko interne mreže sa firewall pravilom (Modul 4).

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mysqld_exporter
curl -s http://127.0.0.1:9104/metrics | head -20
```

### Zabbix

Zabbix isporučuje gotov šablon *MySQL by Zabbix agent 2*, koji pokriva većinu navedenog bez ručnog rada. Podesite nalog za praćenje, dodelite šablon hostu i imate osnovni monitoring za dvadesetak minuta.

### Percona PMM — preporuka za sysadmina bez DBA

Ako nemate već postavljen monitoring stek i ne želite da ga gradite, **Percona Monitoring and Management** je najbolji odnos uloženog i dobijenog.

To je gotov paket (Prometheus + Grafana + agent + dashboardi + analizator upita) koji se podiže kao kontejner. Dobijate detaljne MySQL dashboarde i ugrađenu analizu upita, bez pisanja ijednog PromQL izraza.

Za administratora koji nije DBA, ovo je razuman izbor.

### Koji alarmi imaju smisla

Ovo je deo koji određuje da li će monitoring biti koristan ili ignorisan.

**Alarmi koji bude čoveka noću:**

| Uslov | Zašto |
|---|---|
| MySQL ne odgovara | očigledno |
| Disk `/var/lib/mysql` > 90% | za nekoliko sati je ispad |
| Replikacija prekinuta | gubi se rezerva i backup izvor |
| `Threads_connected` > 90% limita | pred odbijanjem konekcija |
| Backup stariji od 30h | RPO je narušen |

**Alarmi koji prave tiket za sutra:**

| Uslov | Zašto |
|---|---|
| Disk > 80% | ima vremena |
| Zaostajanje replike > 300s | treba pogledati, nije hitno |
| Skok broja sporih upita | neko je nešto deploy-ovao |
| TLS sertifikat ističe za 30 dana | ima vremena |
| Nagli pad `Uptime`-a | server se restartovao, treba utvrditi zašto |
| `Aborted_connects` naglo raste | ili napad, ili pogrešna lozinka u konfiguraciji |

**Alarmi koje ne treba praviti:**

- Skok CPU-a. Baza je tu da radi; visok CPU sam po sebi nije problem.
- Apsolutni prag na buffer pool hit ratio. Zavisi od opterećenja.
- Bilo koja metrika bez jasnog odgovora na pitanje "šta ću uraditi kada se ovo javi".

**Pravilo koje vredi zapisati:** ako se alarm javi i vi ne uradite ništa, to nije alarm — to je šum. Isključite ga ili ga pretvorite u grafikon. Monitoring u koji ljudi prestanu da veruju gori je od nikakvog.

---

## 32. Dva alata za brzu dijagnozu

Kada aplikacija stane i neko vas zove, ovo su prve dve komande.

### `SHOW PROCESSLIST`

```sql
SHOW FULL PROCESSLIST;
```

Kolone koje su bitne:

| Kolona | Značenje |
|---|---|
| `Id` | ID konekcije — koristi se za `KILL` |
| `User`, `Host` | ko i odakle |
| `db` | nad kojom bazom |
| `Command` | `Query`, `Sleep`, `Connect`... |
| **`Time`** | **sekundi u trenutnom stanju** |
| **`State`** | **šta upit trenutno radi** |
| `Info` | tekst upita |

Bolja varijanta, jer `sys.session` sortira i sažima:

```sql
SELECT * FROM sys.session WHERE command != 'Sleep' ORDER BY time DESC;
```

Još lakše čitljivo:

```sql
SELECT id, user, host, db, command, time, state, LEFT(info, 100) AS upit
FROM performance_schema.processlist
WHERE command NOT IN ('Sleep', 'Daemon')
ORDER BY time DESC
LIMIT 20;
```

**Koristite `performance_schema.processlist`, ne `SHOW PROCESSLIST`, kada je server pod pritiskom.** `SHOW PROCESSLIST` uzima interni mutex i može dodatno usporiti ionako opterećen server.

### Tumačenje `State` vrednosti

Ovo je gde je znanje:

| `State` | Šta se dešava |
|---|---|
| `Sending data` | čita i obrađuje redove — ime vara, ovo nije mrežno slanje |
| **`Waiting for table metadata lock`** | **čeka DDL — najčešći uzrok potpunog zastoja** |
| `Waiting for lock` / `updating` | čeka na zaključan red |
| `Copying to tmp table` | pravi privremenu tabelu u memoriji |
| `Copying to tmp table on disk` | **privremena tabela ne staje u RAM — sporo** |
| `Sorting result` | sortira |
| `Opening tables` | previše otvorenih tabela ili spor fajl sistem |
| `Waiting for table flush` | neko je pozvao `FLUSH TABLES` |

**`Waiting for table metadata lock` zaslužuje posebnu pažnju.** Scenario: neko je pokrenuo `ALTER TABLE` na velikoj tabeli. Dok on traje, svaki naredni upit nad tom tabelom čeka. Za minut imate stotine konekcija u čekanju, `Threads_connected` udara u limit, i cela aplikacija stoji — a uzrok je jedna naredba.

Nalaženje krivca:

```sql
SELECT * FROM sys.schema_table_lock_waits;

SELECT object_name, lock_type, lock_status, owner_thread_id
FROM performance_schema.metadata_locks
WHERE lock_status = 'GRANTED';
```

Rešenje u Modulu 9, gde obrađujemo `ALTER TABLE` na velikim tabelama i `pt-online-schema-change`.

### Prekidanje upita

```sql
-- prekini upit, ostavi konekciju
KILL QUERY 4521;

-- prekini celu konekciju
KILL 4521;
```

**Preferirajte `KILL QUERY`.** Prekidanje konekcije tera aplikaciju da je ponovo uspostavi, što u talasu problema samo pogoršava stanje.

Masovno prekidanje upita jednog naloga, kada baš morate:

```sql
SELECT CONCAT('KILL QUERY ', id, ';') AS komanda
FROM performance_schema.processlist
WHERE user = 'report' AND time > 60 AND command = 'Query';
```

Ovo generiše komande koje pregledate pa izvršite. Ne izvršavajte automatski bez pogleda.

Napomena: `KILL` na `UPDATE` ili `DELETE` koji je već obradio milione redova pokreće **rollback**, koji može trajati duže od samog upita i ne može se prekinuti. Znajte to pre nego što pritisnete enter.

### Duge transakcije

Čest, a nevidljiv uzrok problema: transakcija otvorena satima drži zaključavanja i sprečava čišćenje undo prostora (Modul 2).

```sql
SELECT trx_id, trx_state, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS traje_sek,
       trx_mysql_thread_id AS conn_id,
       LEFT(trx_query, 80) AS upit
FROM information_schema.innodb_trx
ORDER BY trx_started;
```

Transakcija u stanju `RUNNING` sa `trx_query` praznim, otvorena dva sata, znači da je aplikacija otvorila transakciju i zaboravila je. To je nalaz za developera.

### `SHOW ENGINE INNODB STATUS`

```sql
SHOW ENGINE INNODB STATUS\G
```

Izlaz je dugačak i zastrašujući. Ne treba vam ceo — treba vam četiri odeljka.

**`LATEST DETECTED DEADLOCK`**

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2026-08-30 14:23:11 0x7f2e
*** (1) TRANSACTION:
TRANSACTION 4829105, ACTIVE 3 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 4 lock struct(s), heap size 1136, 3 row lock(s)
UPDATE narudzbine SET status='obradjeno' WHERE id=1042
*** (2) TRANSACTION:
...
*** WE ROLL BACK TRANSACTION (1)
```

Ovde je zapisan **poslednji** deadlock: koje su dve transakcije, koji upiti, i koju je server poništio. Ovo kopirate i šaljete developeru — to je potpun dokaz.

Ograničenje: čuva se samo poslednji. Za praćenje svih:

```ini
[mysqld]
innodb_print_all_deadlocks = ON
```

Tada svi deadlockovi idu u error log. Uključite privremeno dok istražujete; stalno uključeno pravi šum.

**`TRANSACTIONS`**

```
---TRANSACTION 4829087, ACTIVE 7241 sec
mysql tables in use 1, locked 1
```

`ACTIVE 7241 sec` — transakcija otvorena dva sata. Ovo je problem, čak i kada trenutno ništa ne radi.

**`BUFFER POOL AND MEMORY`**

```
Buffer pool size   524288
Free buffers       1024
Database pages     498231
Modified db pages  8472
Buffer pool hit rate 998 / 1000
```

`Buffer pool hit rate` bliska 1000/1000 znači da se skoro sve čita iz memorije. Trajna vrednost ispod ~950 govori da je buffer pool premalen za radni skup podataka (Modul 7).

`Free buffers` blizu nule je normalno na serveru koji radi — to znači da se memorija koristi.

**`SEMAPHORES`**

```
--Thread 140234 has waited at btr0cur.cc line 1234 for 15.00 seconds
```

Duga čekanja ovde znače unutrašnje nadmetanje niti — obično posledica veoma visoke konkurentnosti ili sporog diska. Retko ćete ovo videti, ali kada ga vidite, to je ozbiljno.

### Prvih šezdeset sekundi

Kada vas pozovu da "baza ne radi", ovo je redosled:

```bash
# 1. Da li proces uopšte radi?
systemctl status mysql

# 2. Da li odgovara?
mysqladmin status

# 3. Ima li mesta na disku?
df -h /var/lib/mysql

# 4. Šta piše u logu?
sudo tail -50 /var/log/mysql/error.log

# 5. Šta se trenutno dešava u bazi?
mysql -e "SELECT id, user, time, state, LEFT(info,60)
          FROM performance_schema.processlist
          WHERE command NOT IN ('Sleep','Daemon')
          ORDER BY time DESC LIMIT 20;"

# 6. Koliko konekcija?
mysql -e "SHOW GLOBAL STATUS LIKE 'Threads_%'; SELECT @@max_connections;"

# 7. Ima li dugih transakcija?
mysql -e "SELECT trx_id, trx_started, trx_mysql_thread_id, LEFT(trx_query,60)
          FROM information_schema.innodb_trx ORDER BY trx_started LIMIT 10;"

# 8. Opterećenje sistema
uptime; free -h; iostat -x 1 3
```

Osam komandi, jedan minut. U devet od deset slučajeva u ovom trenutku već znate šta je problem.

Stavite ovo u skript i držite ga na svakom serveru.

---

## Kontrolna lista na kraju modula

```bash
# 1. Vremenske oznake u logu su lokalne
mysql -e "SELECT @@log_timestamps;"

# 2. Slow log je uključen sa razumnim pragom
mysql -e "SELECT @@slow_query_log, @@long_query_time, @@log_slow_extra;"

# 3. General log je ISKLJUČEN
mysql -e "SELECT @@general_log;"

# 4. Rotacija radi — postoje komprimovani stari logovi
ls -la /var/log/mysql/*.gz

# 5. Nema obrisanih fajlova koje proces još drži
sudo lsof -p "$(pgrep -x mysqld)" | grep deleted

# 6. Monitoring radi i prikuplja metrike
curl -s http://127.0.0.1:9104/metrics | grep -c mysql_

# 7. Alarmi postoje za: dostupnost, disk, konekcije, replikaciju,
#    starost backupa, istek sertifikata

# 8. Skript za trijažu postoji na serveru
ls -la /usr/local/bin/mysql-trijaza.sh

# 9. Umete da pročitate slow log kroz pt-query-digest
pt-query-digest --limit 3 /var/log/mysql/slow.log | head -40

# 10. Nema aktivnih transakcija starijih od nekoliko minuta
mysql -e "SELECT COUNT(*) FROM information_schema.innodb_trx
          WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 300;"
```

---

## Vežbe

**Vežba 1 — Vremenske zone u logovima**
Ostavite `log_timestamps` na UTC i izazovite grešku u MySQL-u i u aplikaciji istovremeno. Pokušajte da ih povežete gledajući oba loga. Zatim prebacite na `SYSTEM` i ponovite. Ovo je vežba koja se pamti tek kad je čovek sam proživi.

**Vežba 2 — Prepoznavanje pada servera**
Ubijte `mysqld` sa `kill -9`, pa ga pustite da se podigne. Pronađite u error logu dokaz da gašenje nije bilo uredno i pronađite poruke o crash recovery-ju. Zatim uporedite sa urednim `systemctl restart`.

**Vežba 3 — Prekinute konekcije**
Postavite `wait_timeout = 10`. Otvorite konekciju, sačekajte, pa pokušajte upit. Pronađite poruku u error logu i uporedite `Aborted_clients` pre i posle. Objasnite razliku u odnosu na `Aborted_connects`.

**Vežba 4 — Spor upit i njegov trag**
Napravite tabelu sa milion redova bez indeksa. Uključite slow log sa pragom 0.5s i izvršite upit sa `WHERE` po neindeksiranoj koloni. Pročitajte zapis i objasnite odnos `Rows_sent` i `Rows_examined`. Dodajte indeks i ponovite.

**Vežba 5 — `pt-query-digest` i pravi prioritet**
Napravite dva upita: jedan koji traje 10 sekundi i izvršava se jednom, i drugi koji traje 20 ms i izvršava se 10.000 puta. Pustite oba, pa analizirajte slow log. Objasnite koji je od njih pravi problem i zašto rang-lista izgleda tako.

**Vežba 6 — Rotacija koja ne radi**
Namerno pokvarite `postrotate` (npr. pogrešna putanja do `mysqladmin`). Pokrenite rotaciju i proverite sa `lsof` gde MySQL zapravo piše. Objasnite zašto `du` i `ls` pokazuju različitu sliku, pa popravite.

**Vežba 7 — Metadata lock zastoj**
U jednoj sesiji pokrenite dugu transakciju sa `SELECT` nad tabelom bez `COMMIT`. U drugoj pokrenite `ALTER TABLE` nad istom tabelom. U trećoj pokrenite običan `SELECT`. Pogledajte `processlist` i objasnite šta se desilo i kojim redom. Zatim razrešite situaciju.

**Vežba 8 — Deadlock i njegov zapis**
Izazovite deadlock sa dve sesije koje ažuriraju dva reda obrnutim redosledom. Pročitajte `LATEST DETECTED DEADLOCK` i napišite izveštaj kakav biste poslali developeru.

**Vežba 9 — Prvih šezdeset sekundi**
Napišite skript sa osam komandi iz poglavlja 32. Zatim zamolite kolegu da na test serveru izazove problem po sopstvenom izboru (pun disk, previše konekcija, dugačak `ALTER`, ubijen proces). Dijagnostikujte isključivo pomoću svog skripta i izmerite koliko vam je trebalo.

---

## Šta sledi

U **Modulu 7** prelazimo na performanse — ali iz ugla sistemca, ne DBA.

Obradićemo `innodb_buffer_pool_size` kao jedini parametar koji zaista morate podesiti, računicu koliko RAM-a MySQL stvarno troši (koja je komplikovanija nego što izgleda), sudar `max_connections`, `open_files_limit` i systemd `LimitNOFILE`, uticaj diska i `fsync` podešavanja, kernel parametre poput `vm.swappiness`, transparent huge pages i NUMA — i, možda najkorisnije, kako da **dokažete** da problem nije u bazi nego u aplikaciji.

Poslednja stavka je često pravi ishod. Ovaj modul vam je dao alate da to i pokažete brojkama.
