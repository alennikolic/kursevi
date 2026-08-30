# Modul 8: Replikacija i visoka dostupnost

*MySQL za Linux administratore*

---

## Uvod: replika nije backup, a HA nije replika

Tri pojma se stalno mešaju, pa da ih odmah razdvojimo.

**Backup** je kopija podataka u prošlosti. Štiti od greške, brisanja, oštećenja i ransomware-a. Sporo se vraća.

**Replika** je kopija podataka u sadašnjosti. Štiti od otkaza hardvera. **Ne štiti od greške** — `DROP TABLE` na primarnom serveru stiže na repliku za milisekundu, uredno i pouzdano.

**Visoka dostupnost (HA)** je automatika koja prebacuje saobraćaj kada primarni server otkaže. To je zaseban sloj iznad replikacije i ne dolazi sam od sebe.

Najčešća zabluda u praksi: "imamo repliku, znači imamo backup". Nemate. Imate dva servera sa istovremeno obrisanom tabelom.

Ono što replika **jeste** dobra za:

- rezervu koja preuzima posao kada primarni otkaže,
- izvor za backup, bez opterećenja primarnog servera,
- read-only server za izveštaje i analitiku,
- **zaštitu od ljudske greške — ako je odložena** (o tome u poglavlju 40, i to je jedan od najkorisnijih trikova u kursu).

### Napomena o terminologiji

Od MySQL 8.0.22 stara terminologija (`master`/`slave`) zamenjena je novom (`source`/`replica`). Stare komande i dalje rade u seriji 8.0 kao zastareli sinonimi, ali su **uklonjene u 8.4**.

| Staro | Novo |
|---|---|
| `CHANGE MASTER TO` | `CHANGE REPLICATION SOURCE TO` |
| `START SLAVE` / `STOP SLAVE` | `START REPLICA` / `STOP REPLICA` |
| `SHOW SLAVE STATUS` | `SHOW REPLICA STATUS` |
| `SHOW MASTER STATUS` | `SHOW BINARY LOG STATUS` (8.4+) |
| `Slave_IO_Running` | `Replica_IO_Running` |
| `Seconds_Behind_Master` | `Seconds_Behind_Source` |

Praktična posledica: **proverite svoje monitoring skriptove pre nadogradnje na 8.4.** Skript koji parsira `SHOW SLAVE STATUS` tiho prestaje da radi, a vi ostajete bez praćenja replikacije baš u trenutku kada je najosetljivija.

U tekstu koristim novu terminologiju.

---

## 39. Replikacija od nule

### Kako radi

Mehanizam je jednostavniji nego što deluje:

1. **Primarni server** upisuje svaku izmenu u binarni log (Moduli 2 i 5).
2. **IO nit** na replici povezuje se na primarni, preuzima događaje iz binloga i upisuje ih u svoj **relay log**.
3. **SQL nit** (applier) na replici čita relay log i primenjuje izmene.

Dve niti znače dve tačke otkaza — i to je prva stvar koju gledate kada nešto ne radi.

Replikacija je podrazumevano **asinhrona**: primarni server ne čeka repliku. Commit se potvrđuje aplikaciji čim je zapisan lokalno. Posledica: ako primarni padne, poslednje transakcije možda nikad nisu stigle do replike.

### Konfiguracija primarnog servera

```ini
# /etc/mysql/mysql.conf.d/99-custom.cnf na PRIMARNOM
[mysqld]
server_id = 1

log_bin = /var/lib/mysql/binlog
binlog_format = ROW
binlog_expire_logs_seconds = 604800

gtid_mode = ON
enforce_gtid_consistency = ON

sync_binlog = 1
innodb_flush_log_at_trx_commit = 1

binlog_transaction_dependency_tracking = WRITESET
```

Objašnjenje bitnih stavki:

**`server_id`** mora biti jedinstven u celoj topologiji. Ovo je obavezno.

**`binlog_format = ROW`** je podrazumevano u MySQL 8 i treba tako da ostane. `STATEMENT` format ponekad daje različite rezultate na primarnom i na replici (npr. kod `NOW()` ili `UUID()`), što tiho razilazi podatke.

**`gtid_mode = ON`** — GTID (Global Transaction Identifier) daje svakoj transakciji jedinstven identifikator. Bez GTID-a replikaciju podešavate ručnim navođenjem fajla i pozicije binloga; sa GTID-om replika sama zna dokle je stigla. **Koristite GTID.** Failover, ponovno povezivanje i dijagnostika su neuporedivo jednostavniji.

**`sync_binlog = 1` i `innodb_flush_log_at_trx_commit = 1`** — obećanje iz Modula 7 se ovde naplaćuje. Sa slabijim vrednostima primarni server nakon pada može "izgubiti" transakcije koje je replika već primila, pa replika ode dalje od primarnog. To razbija replikaciju na način koji se teško popravlja.

**`binlog_transaction_dependency_tracking = WRITESET`** — govori primarnom serveru da u binlog upiše informacije o tome koje transakcije su međusobno nezavisne. To omogućava replici da ih primenjuje paralelno. Ovo je jedno od najkorisnijih podešavanja za smanjenje zaostajanja i često se previdi.

### Konfiguracija replike

```ini
# /etc/mysql/mysql.conf.d/99-custom.cnf na REPLICI
[mysqld]
server_id = 2

log_bin = /var/lib/mysql/binlog
log_replica_updates = ON
relay_log = /var/lib/mysql/relay-bin

gtid_mode = ON
enforce_gtid_consistency = ON

read_only = ON
super_read_only = ON

skip_replica_start = ON

replica_parallel_workers = 4
replica_parallel_type = LOGICAL_CLOCK
replica_preserve_commit_order = ON
```

**`read_only` i `super_read_only`** — replika ne sme da prima upise. `read_only` sprečava obične korisnike; `super_read_only` sprečava i one sa administratorskim privilegijama. Uključite oba.

Zašto je ovo važno: jedan `INSERT` izvršen direktno na replici može izazvati grešku duplikata pri narednoj replikaciji i zaustaviti je. To je najčešći uzrok "replikacija je stala" u praksi.

**`skip_replica_start = ON`** — nakon restarta replika neće sama krenuti da replicira. Deluje kontraintuitivno, ali je dobra praksa: ako je replika restartovana zbog problema, želite priliku da pogledate stanje pre nego što nastavi. Startujete je ručno sa `START REPLICA`.

**`log_replica_updates = ON`** — replika upisuje primenjene izmene i u svoj binlog. Potrebno je ako repliku koristite kao izvor za backup sa PITR-om, ili ako će jednog dana postati primarni server. Uključite.

**`replica_parallel_workers = 4`** — više niti koje primenjuju izmene. U kombinaciji sa `WRITESET` na primarnom, ovo drastično smanjuje zaostajanje. `replica_preserve_commit_order = ON` osigurava da se transakcije potvrđuju istim redosledom kao na primarnom.

### Zamka sa `server_uuid`

Ako ste repliku napravili kloniranjem virtuelne mašine, podsetnik iz Modula 2:

```bash
sudo cat /var/lib/mysql/auto.cnf
```

Dva servera sa istim `server-uuid` **ne mogu replicirati**. Poruka o grešci ne ukazuje jasno na uzrok.

```bash
sudo systemctl stop mysql
sudo rm /var/lib/mysql/auto.cnf
sudo systemctl start mysql
mysql -e "SELECT @@server_uuid;"
```

### Nalog za replikaciju

Na primarnom (Modul 3):

```sql
CREATE USER 'repl'@'10.0.1.%' IDENTIFIED BY 'duga-nasumicna-lozinka' REQUIRE SSL;
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'10.0.1.%';
```

`REQUIRE SSL` nije formalnost — replikacioni tok sadrži **sve** vaše podatke i putuje mrežom (Modul 4).

### Prenos početnih podataka — tri načina

Replika mora krenuti od nekog konzistentnog stanja.

#### Način 1: CLONE plugin (najlakši, MySQL 8.0.17+)

Ovo je danas najjednostavniji put i vredi ga koristiti kad god možete.

Na oba servera:

```sql
INSTALL PLUGIN clone SONAME 'mysql_clone.so';
```

Na primarnom, nalogu za repliku dodajte:

```sql
GRANT BACKUP_ADMIN ON *.* TO 'repl'@'10.0.1.%';
```

Na replici:

```sql
SET GLOBAL clone_valid_donor_list = '10.0.1.10:3306';

CLONE INSTANCE FROM 'repl'@'10.0.1.10':3306
  IDENTIFIED BY 'duga-nasumicna-lozinka';
```

Replika **briše svoj `datadir`**, kopira ceo primarni i sama se restartuje. Napredak:

```sql
SELECT stage, state, end_time FROM performance_schema.clone_progress;
```

Ovo je fizičko kopiranje, dakle brzo, i automatski prenosi GTID stanje.

#### Način 2: `mysqldump`

Za manje baze:

```bash
mysqldump -h 10.0.1.10 -u backup -p \
  --all-databases \
  --single-transaction \
  --source-data=2 \
  --routines --events \
  --set-gtid-purged=ON \
  | gzip > /tmp/seed.sql.gz

zcat /tmp/seed.sql.gz | mysql
```

`--set-gtid-purged=ON` upisuje u dump informaciju o tome koje su GTID transakcije već sadržane, pa replika zna odakle da nastavi.

#### Način 3: XtraBackup

Za velike baze, po proceduri iz Modula 5. Nakon `--prepare` i `--copy-back`, pozicija se nalazi u:

```bash
cat /var/lib/mysql/xtrabackup_binlog_info
```

```
binlog.000042    1234567    3a7f8c2e-...:1-98234
```

Treći deo je GTID skup koji treba postaviti pre pokretanja replikacije:

```sql
RESET MASTER;   -- na MySQL 8.4: RESET BINARY LOGS AND GTIDS
SET GLOBAL gtid_purged = '3a7f8c2e-...:1-98234';
```

### Pokretanje replikacije

Sa GTID-om:

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = '10.0.1.10',
  SOURCE_PORT = 3306,
  SOURCE_USER = 'repl',
  SOURCE_PASSWORD = 'duga-nasumicna-lozinka',
  SOURCE_AUTO_POSITION = 1,
  SOURCE_SSL = 1,
  SOURCE_CONNECT_RETRY = 10,
  SOURCE_RETRY_COUNT = 86400;

START REPLICA;
```

`SOURCE_AUTO_POSITION = 1` znači "sam pronađi gde si stao, na osnovu GTID-a". Zbog toga GTID i koristimo.

Bez GTID-a, sa ručnom pozicijom:

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = '10.0.1.10',
  SOURCE_USER = 'repl',
  SOURCE_PASSWORD = '...',
  SOURCE_LOG_FILE = 'binlog.000042',
  SOURCE_LOG_POS = 1234567;
```

### Provera

```sql
SHOW REPLICA STATUS\G
```

Tražite tri stvari:

```
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
Seconds_Behind_Source: 0
```

Ako su prve dve `Yes`, replikacija radi. Brzi test:

```sql
-- na primarnom
CREATE DATABASE test_replikacije;
CREATE TABLE test_replikacije.t (id INT PRIMARY KEY, v VARCHAR(50));
INSERT INTO test_replikacije.t VALUES (1, 'proba');
```

```sql
-- na replici, nekoliko sekundi kasnije
SELECT * FROM test_replikacije.t;
```

Zatim počistite za sobom.

---

## 40. Održavanje replikacije

### Čitanje `SHOW REPLICA STATUS`

Izlaz ima preko pedeset polja. Bitno je desetak.

| Polje | Šta znači |
|---|---|
| `Replica_IO_Running` | preuzimanje sa primarnog: `Yes` / `No` / `Connecting` |
| `Replica_SQL_Running` | primena izmena: `Yes` / `No` |
| `Seconds_Behind_Source` | zaostajanje; **`NULL` znači prekid** |
| `Last_IO_Error` | zašto je preuzimanje stalo |
| `Last_SQL_Error` | zašto je primena stala |
| `Last_SQL_Errno` | broj greške — po njemu pretražujete |
| `Retrieved_Gtid_Set` | šta je preuzeto |
| `Executed_Gtid_Set` | šta je primenjeno |
| `Auto_Position` | 1 ako se koristi GTID |
| `Replica_SQL_Running_State` | šta applier trenutno radi |

Kratka provera za skript:

```bash
mysql -e "SHOW REPLICA STATUS\G" | grep -E \
  'Replica_IO_Running|Replica_SQL_Running|Seconds_Behind_Source|Last_.*Error'
```

### `Seconds_Behind_Source` vara — koristite heartbeat

Ovo polje meri razliku između vremenske oznake događaja koji se trenutno primenjuje i trenutnog vremena na replici. Zvuči razumno, ali:

- **Ako IO nit stane, a SQL nit primeni sve što je imala, vrednost je 0** — iako replika zaostaje satima. Ovo je najopasniji slučaj: metrika kaže da je sve u redu, a nije.
- U lančanoj replikaciji vrednost je nepouzdana.
- Osetljiva je na razliku u satu između servera.

Pouzdano rešenje je `pt-heartbeat` iz Percona Toolkit-a. Primarni server upisuje trenutno vreme u malu tabelu svake sekunde; replika čita tu tabelu i računa razliku. To meri **stvarno** kašnjenje kroz ceo lanac.

Na primarnom:

```bash
pt-heartbeat --database=percona --create-table --update --daemonize \
  --defaults-file=/etc/mysql/heartbeat.cnf
```

Na replici:

```bash
pt-heartbeat --database=percona --monitor --defaults-file=/etc/mysql/heartbeat.cnf
```

```
0.00s [  0.00s,  0.00s,  0.00s ]
```

Postavite ovo kao systemd servise i vežite metriku za monitoring iz Modula 6.

Dodatna provera, direktno iz baze:

```sql
SELECT * FROM performance_schema.replication_applier_status_by_worker\G
SELECT * FROM performance_schema.replication_connection_status\G
```

### Uzroci zaostajanja

**Jednonitni applier.** Ako je `replica_parallel_workers = 0`, replika primenjuje izmene jednom niti, dok je primarni upisivao sa stotinu. Rešenje je paralelna replikacija iz prethodnog poglavlja, uz `WRITESET` na primarnom.

**Spor disk na replici.** Česta ušteda koja se skupo plaća: replika na slabijem hardveru. Proverite sa `iostat` (Modul 7).

**Velika transakcija.** Jedan `DELETE` koji briše pet miliona redova na primarnom se izvršava paralelno; na replici se primenjuje kao jedan blok. Replika zaostane za minute. Rešenje je aplikaciono — brisanje u serijama.

**`ALTER TABLE`.** Izmena velike tabele blokira applier dok traje. Rešenje u Modulu 9.

**Nedostajući indeks na replici.** Sa `ROW` formatom, replika za svaki izmenjen red mora da ga pronađe. Bez primarnog ključa to znači pun pregled tabele **po redu**. Tabela bez primarnog ključa je najbrži način da uništite replikaciju.

```sql
SELECT t.table_schema, t.table_name
FROM information_schema.tables t
LEFT JOIN information_schema.table_constraints c
  ON t.table_schema = c.table_schema
 AND t.table_name = c.table_name
 AND c.constraint_type = 'PRIMARY KEY'
WHERE c.constraint_name IS NULL
  AND t.table_type = 'BASE TABLE'
  AND t.table_schema NOT IN ('mysql','information_schema','performance_schema','sys');
```

Ako ovaj upit vrati redove — to je nalaz za developera, sa obrazloženjem.

### Kada replikacija stane

**Prvo pravilo: ne pokušavajte da je "preskočite" pre nego što shvatite zašto je stala.**

```sql
SHOW REPLICA STATUS\G
```

```
Replica_SQL_Running: No
Last_SQL_Errno: 1062
Last_SQL_Error: Could not execute Write_rows event on table shop.korisnici;
Duplicate entry '4521' for key 'PRIMARY'
```

#### Greška 1062 — duplikat ključa

Replika već ima red koji primarni pokušava da unese. Uzrok je gotovo uvek: **neko je pisao direktno u repliku.**

Zato `super_read_only = ON`. Ako se ovo dešava, prvo proverite da li je uključeno.

#### Greška 1032 — red nije pronađen

Replika nema red koji primarni pokušava da izmeni. Isti uzrok, obrnut smer — podaci su se razišli.

#### Greška 1236 — binlog je obrisan

```
Last_IO_Error: Got fatal error 1236 from source when reading data from binary log:
'Cannot replicate because the source purged required binary logs.'
```

Replika je bila zaustavljena duže nego što primarni čuva binlogove. Ne postoji način da se ovo popravi delimično — **repliku morate ponovo napraviti od nule** (CLONE ili XtraBackup).

Prevencija: `binlog_expire_logs_seconds` dovoljno velik, i alarm na prekinutu replikaciju koji stiže odmah, a ne posle tri dana (Modul 6).

#### Preskakanje greške — kada baš morate

Sa GTID-om, preskače se tako što se prazna transakcija sa tim identifikatorom "potroši":

```sql
STOP REPLICA;

-- iz Last_SQL_Error ili iz Retrieved_Gtid_Set nađite konkretan GTID
SET GTID_NEXT = '3a7f8c2e-1234-5678-9abc-def012345678:98235';
BEGIN; COMMIT;
SET GTID_NEXT = 'AUTOMATIC';

START REPLICA;
```

Bez GTID-a:

```sql
STOP REPLICA;
SET GLOBAL sql_slave_skip_counter = 1;
START REPLICA;
```

**Obavezno upozorenje:** preskakanje greške ne rešava problem — ono ga sakriva. Podaci na replici sada se **razlikuju** od primarnog. Ako to radite, obavezno zatim proverite konzistentnost.

Nikada ne koristite `replica_skip_errors` u konfiguraciji. To je automatsko preskakanje svih grešaka i garantovan način da tiho dobijete dve različite baze.

### Provera konzistentnosti

```bash
pt-table-checksum \
  --defaults-file=/etc/mysql/checksum.cnf \
  --databases=shop \
  --no-check-binlog-format
```

Alat se pokreće **na primarnom**. Računa kontrolne sume po delovima tabela, upisuje ih u svoju tabelu, a te izmene se repliciraju — pa svaka replika izračuna svoju sumu nad svojim podacima. Razlike se prijavljuju.

```
            TS ERRORS  DIFFS     ROWS  DIFF_ROWS  CHUNKS SKIPPED    TIME TABLE
08-30T14:23      0      0   482910          0      52       0   38.291 shop.korisnici
08-30T14:24      0      2  1204882         14     124       0   91.482 shop.narudzbine
```

Kolona `DIFFS` različita od nule znači da su se podaci razišli.

Popravka:

```bash
pt-table-sync --dry-run --replicate=percona.checksums h=10.0.1.10
# pregledajte šta bi uradio, pa tek onda:
pt-table-sync --execute --replicate=percona.checksums h=10.0.1.10
```

**Uvek prvo `--dry-run`.** Ovaj alat menja podatke.

### Odložena replika — jeftina zaštita od ljudske greške

Ovo je jedan od najkorisnijih trikova u celom kursu i košta jedan dodatni server.

```sql
STOP REPLICA;
CHANGE REPLICATION SOURCE TO SOURCE_DELAY = 3600;
START REPLICA;
```

Replika sada namerno kasni **sat vremena**.

Zašto je to dobro: kada neko u 14:35 izvrši `DELETE FROM narudzbine` bez `WHERE`, obična replika izvrši istu naredbu u 14:35:00.2. Odložena replika će je izvršiti tek u 15:35 — a vi imate ceo sat da to primetite i zaustavite je:

```sql
STOP REPLICA SQL_THREAD;
```

Zatim, na miru, primenite događaje **do trenutka pre greške**:

```sql
START REPLICA SQL_THREAD UNTIL SQL_BEFORE_GTIDS = '3a7f8c2e-...:98235';
```

Podatke izvučete i vratite na primarni.

Poređenja radi: point-in-time recovery iz Modula 5 za istu situaciju traži restore punog backupa i primenu binlogova — sati posla. Sa odloženom replikom to je nekoliko minuta.

Preporuka: ako imate dve replike, neka jedna bude bez odlaganja (za failover i izveštaje), a druga sa odlaganjem od sat vremena (kao zaštita od greške).

### Monitoring replikacije

Vratite se na Modul 6 i dodajte:

| Metrika | Alarm |
|---|---|
| `Replica_IO_Running != Yes` | kritično, odmah |
| `Replica_SQL_Running != Yes` | kritično, odmah |
| `Seconds_Behind_Source` je `NULL` | kritično |
| `pt-heartbeat` kašnjenje > 300 s | upozorenje |
| `pt-heartbeat` kašnjenje > 1800 s | kritično |
| `pt-table-checksum` prijavljuje razlike | tiket |

**Prekinuta replikacija mora da bude alarm koji stiže odmah**, ne izveštaj koji neko pogleda za tri dana. Do tada su binlogovi obrisani i repliku pravite iz početka.

---

## 41. Replika kao izvor za backup i kao read-only server

### Backup sa replike

Ovo je najpraktičnija svakodnevna korist od replike.

Backup opterećuje server — čita sve podatke, troši I/O, a kod nekih metoda i zaključava. Ako se izvršava na replici, primarni server to uopšte ne oseti.

Uz to možete priuštiti nešto što na primarnom ne biste:

```sql
STOP REPLICA;
-- backup nad potpuno mirnim serverom
START REPLICA;
```

Replika će nakon nastavka brzo nadoknaditi zaostatak.

**Ključni detalj:** kada radite `mysqldump` na replici, treba vam pozicija **primarnog** servera, ne replike:

```bash
mysqldump --all-databases \
  --single-transaction \
  --dump-replica=2 \
  --routines --events \
  --no-tablespaces \
  | gzip > /backup/full-$(date +%F).sql.gz
```

`--dump-replica=2` zapisuje koordinate primarnog servera kao komentar. Bez toga imate backup iz koga ne možete raditi PITR, jer ne znate na koju tačku primarnog se odnosi.

GTID pozicija:

```sql
SELECT @@gtid_executed;
```

### Read-only server za izveštaje

Analitički upiti su tipičan uzrok obaranja produkcijske baze: jedan `GROUP BY` nad deset miliona redova zauzme I/O i sve ostalo stane.

Rešenje je da takvi upiti idu na repliku.

Nalog sa ograničenjima (Modul 3):

```sql
CREATE USER 'report'@'10.0.2.%'
  IDENTIFIED BY '...'
  WITH MAX_USER_CONNECTIONS 5 MAX_QUERIES_PER_HOUR 20000;

GRANT SELECT ON shop.* TO 'report'@'10.0.2.%';
```

Na replici možete i dodati indekse kojih na primarnom nema, ako služe samo izveštajima. To radi jer se DDL replicira sa primarnog, ali dodatni indeks na replici ne smeta — samo pazite da ne dođe u sukob sa budućim izmenama šeme.

**Ograničenje koje morate saopštiti korisnicima:** replika zaostaje. Izveštaj napravljen u 14:00 može odražavati stanje od 13:58. Za većinu izveštavanja to je nebitno, ali za nešto poput provere stanja naloga jeste. Recite to unapred, umesto da objašnjavate kasnije.

### Razdvajanje čitanja i pisanja

Aplikacija može slati `SELECT` upite na replike, a upise na primarni. To se radi na dva načina:

**U aplikaciji** — framework podržava dve konekcije. Najjednostavnije, ali zahteva izmenu koda.

**Kroz proxy** — ProxySQL ili MySQL Router stoje između aplikacije i baze i sami usmeravaju saobraćaj. O tome u sledećem poglavlju.

Zamka koju treba znati unapred: **"pročitaj odmah nakon upisa" ne radi.** Aplikacija upiše red na primarni, odmah ga pročita sa replike, i ne nađe ga — jer replikacija još nije stigla. Ovo je aplikaciona odluka (koje čitanje sme sa replike), ne infrastrukturna.

---

## 42. Visoka dostupnost — i kada vam ne treba

### Iskren uvod

Replikacija nije automatsko preuzimanje posla. Ako primarni server otkaže, replika i dalje stoji kao `read_only` i čeka da neko nešto uradi.

To "nešto" je ili čovek sa dokumentovanom procedurom, ili automatika. Automatika je složenija nego što izgleda i uvodi sopstvene načine otkaza.

### Merdevine dostupnosti

Popnite se samo onoliko koliko vam stvarno treba:

| Nivo | Šta imate | Vreme oporavka | Složenost |
|---|---|---|---|
| 1 | Backup i testiran restore | sati | mala |
| 2 | + replika | 10–30 min (ručno) | mala |
| 3 | + dokumentovana procedura preuzimanja | 5–15 min | mala |
| 4 | + proxy sloj (ProxySQL) | 1–5 min | srednja |
| 5 | + automatsko preuzimanje | sekunde | **velika** |
| 6 | Sinhroni klaster | sekunde | **velika** |

**Većini sistema koje ćete održavati dovoljan je nivo 3.**

To nije lenjost, to je inženjerska procena. Loše podešen klaster otkazuje **češće** od jednog dobro održavanog servera, i to na načine koje je teže dijagnostikovati. Split-brain u tri ujutru je gori problem od servera koji je pao i koji vraćate po proceduri.

Pre nego što krenete u HA, odgovorite na tri pitanja:

1. **Koliko stvarno košta sat nedostupnosti?** Često manje nego što se tvrdi.
2. **Ko će održavati klaster kada vas nema?** Ako je odgovor "niko", ne pravite ga.
3. **Da li je restore testiran?** Ako nije, HA je pogrešan prioritet — vratite se na Modul 5.

### Ručno preuzimanje — procedura koju treba imati

Ovo je nivo 3 i treba da bude zapisano pre nego što zatreba.

**Korak 1 — potvrdite da je primarni stvarno mrtav.**

Najveća opasnost pri preuzimanju je **split-brain**: stari primarni oživi i nastavi da prima upise, dok novi takođe prima upise. Podaci se razilaze nepovratno.

```bash
ping 10.0.1.10
ssh 10.0.1.10 "systemctl status mysql"
```

Ako sumnjate — **ugasite stari server**. Isključite ga iz mreže, zaustavite instancu kod provajdera, isključite port na svič-u. Bolje mrtav server nego dva živa primarna.

**Korak 2 — proverite da je replika primenila sve što ima.**

```sql
STOP REPLICA IO_THREAD;
SHOW REPLICA STATUS\G
```

Čekajte da `Retrieved_Gtid_Set` i `Executed_Gtid_Set` budu jednaki.

**Korak 3 — promovišite repliku.**

```sql
STOP REPLICA;
RESET REPLICA ALL;
SET GLOBAL super_read_only = OFF;
SET GLOBAL read_only = OFF;
```

**Korak 4 — preusmerite aplikaciju.**

Zavisi od arhitekture: promena DNS zapisa, prebacivanje virtuelne IP adrese, izmena konfiguracije aplikacije, ili promena u ProxySQL-u.

**Unapred odlučite kako ovo radite** i držite komandu u dokumentaciji. Ako u trenutku incidenta razmišljate o TTL-u DNS zapisa, izgubili ste dvadeset minuta.

**Korak 5 — napravite novu repliku.**

Stari primarni, kada se oporavi, **ne sme sam da se vrati u pogon**. Prepravite ga u repliku novog primarnog (CLONE je najlakši put) ili ga ponovo izgradite.

Cela procedura, uvežbana, traje deset do petnaest minuta.

### Poluautomatski: semi-sync replikacija

Ako je glavna briga gubitak podataka pri otkazu, a ne brzina preuzimanja:

```sql
INSTALL PLUGIN rpl_semi_sync_source SONAME 'semisync_source.so';
SET GLOBAL rpl_semi_sync_source_enabled = 1;
SET GLOBAL rpl_semi_sync_source_timeout = 1000;
```

Primarni čeka potvrdu bar jedne replike da je primila transakciju pre nego što potvrdi commit aplikaciji.

Cena: dodatna latencija na svakom upisu, jednaka vremenu odlaska i povratka do replike. Na istoj mreži to je milisekunda ili dve; preko WAN-a je neprihvatljivo.

Napomena: semi-sync garantuje da je događaj **primljen**, ne da je **primenjen**. To je i dalje veliko poboljšanje u odnosu na potpuno asinhronu replikaciju.

### ProxySQL

Nije HA rešenje samo po sebi, ali je najkorisniji dodatni sloj i vredi ga razmotriti pre svega ostalog.

Šta radi:

- **Razdvaja čitanja i pisanja** bez izmene aplikacije — `SELECT` na replike, ostalo na primarni.
- **Objedinjuje konekcije** — hiljadu aplikacionih konekcija ka ProxySQL-u postaje pedeset ka bazi. Ovo direktno rešava problem iz Modula 7.
- **Prati stanje servera** i sam izbacuje repliku koja je pala ili previše zaostala.
- **Omogućava preusmeravanje bez restarta aplikacije** — pri preuzimanju menjate konfiguraciju ProxySQL-a, aplikacija ne primećuje.

Poslednja stavka sama po sebi opravdava uvođenje, jer pretvara korak 4 iz procedure preuzimanja u jednu komandu.

ProxySQL je i sam tačka otkaza, pa se u ozbiljnim postavkama pokreće u paru sa Keepalived-om, ili se instalira lokalno na svakom aplikacionom serveru — što je često jednostavnije rešenje.

### MySQL InnoDB Cluster

Oracleovo zvanično rešenje: **Group Replication** (konsenzus među čvorovima) + **MySQL Router** (usmeravanje) + **MySQL Shell** (upravljanje).

```bash
mysqlsh
```

```javascript
dba.createCluster('mojKlaster')
cluster.addInstance('user@10.0.1.11:3306')
cluster.status()
```

Prednosti: zvanično podržano, automatsko preuzimanje, ugrađena zaštita od split-brain-a.

Zahtevi i ograničenja:
- **najmanje tri čvora** (za kvorum),
- stabilna mreža male latencije,
- sve tabele moraju imati primarni ključ,
- nema `SERIALIZABLE` izolacije,
- ograničenja kod stranih ključeva sa kaskadama.

### Galera / Percona XtraDB Cluster

Sinhroni multi-master klaster. Upis se potvrđuje tek kada ga svi čvorovi prihvate.

Prednosti: svaki čvor prima upise, nema zaostajanja, automatsko preuzimanje.

Ograničenja koja aplikacija mora poštovati:
- samo InnoDB,
- sve tabele moraju imati primarni ključ,
- velike transakcije su problematične,
- **upisi u iste redove sa različitih čvorova izazivaju konflikte** koji se prijavljuju aplikaciji kao greške pri commit-u,
- upis je onoliko brz koliko je brz najsporiji čvor.

Zbog poslednje dve stavke, Galera se u praksi često koristi tako da **svi upisi idu na jedan čvor**, a ostali su rezerva. Time se dobija automatsko preuzimanje, a izbegavaju konflikti.

### Kako birati

| Situacija | Rešenje |
|---|---|
| Mala firma, prihvatljiv zastoj od nekoliko sati | backup + testiran restore |
| Prihvatljiv zastoj od 15 minuta | replika + zapisana procedura |
| Mnogo čitanja, malo upisa | replike + ProxySQL |
| Zastoj mora biti minut, ima ko da održava | InnoDB Cluster |
| Više lokacija, upisi svuda | Galera, uz prilagođenu aplikaciju |
| Nemate vremena za održavanje | managed baza (Modul 11) |

Poslednji red nije šala. Ako ste jedini sysadmin i imate još petnaest servera, managed baza kod provajdera je često racionalnija odluka od klastera koji ćete održavati u slobodno vreme.

---

## Kontrolna lista na kraju modula

```bash
# 1. server_id i server_uuid su različiti
mysql -e "SELECT @@server_id, @@server_uuid;"

# 2. Replika je read-only na oba nivoa
mysql -e "SELECT @@read_only, @@super_read_only;"

# 3. GTID je uključen na oba servera
mysql -e "SELECT @@gtid_mode, @@enforce_gtid_consistency;"

# 4. Replikacija radi
mysql -e "SHOW REPLICA STATUS\G" | grep -E 'Running|Behind|Error'

# 5. Paralelna replikacija je uključena
mysql -e "SELECT @@replica_parallel_workers, @@replica_parallel_type;"

# 6. Nema tabela bez primarnog ključa
# (upit iz poglavlja 40)

# 7. Stvarno kašnjenje se meri heartbeat-om, ne samo Seconds_Behind_Source
systemctl status pt-heartbeat-monitor

# 8. Konzistentnost je proverena u poslednjih mesec dana
tail -5 /var/log/pt-table-checksum.log

# 9. Postoji odložena replika kao zaštita od ljudske greške
mysql -e "SHOW REPLICA STATUS\G" | grep SQL_Delay

# 10. Procedura preuzimanja je zapisana IZVAN ovih servera
```

---

## Vežbe

**Vežba 1 — Replikacija od nule kroz CLONE**
Postavite dva servera i podignite replikaciju koristeći CLONE plugin. Izmerite koliko traje ceo postupak. Zatim ponovite sa `mysqldump` metodom i uporedite.

**Vežba 2 — Zamka sa `server_uuid`**
Klonirajte virtuelnu mašinu sa MySQL-om i pokušajte da postavite replikaciju bez brisanja `auto.cnf`. Pročitajte poruku o grešci i objasnite zašto ne ukazuje jasno na uzrok. Zatim popravite.

**Vežba 3 — Namerno razbijanje replikacije**
Isključite `super_read_only` na replici i unesite red direktno. Zatim unesite isti red na primarnom. Pročitajte grešku 1062, pa razrešite situaciju na oba načina: preskakanjem GTID-a i ponovnom izgradnjom. Nakon preskakanja pokrenite `pt-table-checksum` i objasnite rezultat.

**Vežba 4 — `Seconds_Behind_Source` vara**
Zaustavite IO nit, a ostavite SQL nit da radi. Pratite `Seconds_Behind_Source` i objasnite zašto pokazuje 0 iako replika zaostaje. Zatim postavite `pt-heartbeat` i ponovite eksperiment.

**Vežba 5 — Paralelna replikacija**
Izmerite zaostajanje pri intenzivnom upisu sa `replica_parallel_workers = 0`. Zatim uključite četiri niti i `WRITESET` na primarnom, pa ponovite isto opterećenje. Uporedite brojke.

**Vežba 6 — Odložena replika u akciji**
Postavite `SOURCE_DELAY = 600`. Zatim na primarnom izvršite `DROP TABLE`. Zaustavite SQL nit na replici pre nego što stigne do te naredbe, izvucite tabelu i vratite je na primarni. Izmerite koliko je ceo postupak trajao i uporedite sa PITR procedurom iz Modula 5.

**Vežba 7 — Backup sa replike**
Napravite backup na replici sa `--dump-replica=2`. Pronađite koordinate primarnog u dumpu. Zatim uradite backup **bez** te opcije i objasnite šta ste izgubili.

**Vežba 8 — Ručno preuzimanje**
Ugasite primarni server naglo. Promovišite repliku po proceduri iz poglavlja 42 i izmerite ukupno vreme. Zatim podignite stari primarni i prepravite ga u repliku novog. Zapišite proceduru sa stvarnim komandama za svoju arhitekturu.

**Vežba 9 — Split-brain**
Na test okruženju namerno napravite situaciju u kojoj oba servera prihvataju upise. Unesite različite podatke na oba, pa pokušajte da ih ponovo povežete. Objasnite zašto ovo nema čisto rešenje — i zašto je gašenje starog primarnog obavezan korak.

---

## Šta sledi

U **Modulu 9** bavimo se održavanjem i promenama: nadogradnjom MySQL-a bez gubitka podataka, migracijom baze na novi server sa minimalnim zastojem, izmenama velikih tabela koje ne obaraju aplikaciju, i rutinskim održavanjem — fragmentacijom, `OPTIMIZE`, `ANALYZE`.

Jedna stvar iz ovog modula ide direktno tamo: **replika je najbolji alat za migraciju i za nadogradnju.** Umesto da menjate produkcijski server, napravite novi, replicirate na njega, i preuzmete posao po proceduri koju ste upravo naučili. Zastoj se meri minutima umesto satima.
