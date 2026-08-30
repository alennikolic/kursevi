# Modul 9: Održavanje, nadogradnja, migracija

*MySQL za Linux administratore*

---

## Uvod: promene na sistemu koji ne sme da stane

Do sada smo server postavljali, obezbeđivali i pratili. Ovaj modul je o tome kako ga **menjati** dok radi.

Četiri operacije, po rastućoj učestalosti:

- **Nadogradnja verzije** — jednom u godinu ili dve, ali sa najvećim ulogom.
- **Migracija na novi server** — nekoliko puta u životu sistema.
- **Izmena šeme** — možda svake nedelje, i najčešći uzrok ispada iz ove grupe.
- **Rutinsko održavanje** — stalno, i uglavnom se radi pogrešno ili nepotrebno.

Zajednička nit sve četiri: **postoji način da se to uradi sa zastojem od nekoliko minuta, i postoji način sa zastojem od nekoliko sati.** Razlika je u pripremi, ne u alatima.

I jedna stvar koja se vraća kroz ceo modul: **replika iz Modula 8 je vaš glavni alat.** Umesto da menjate produkcijski server, napravite novi pored njega, prebacite posao, i imate rollback.

---

## 43. Nadogradnja MySQL-a

### Pravilo koje određuje sve ostalo

> **Nadogradnja je jednosmerna. Vraćanje unazad nije podržano.**

Kada MySQL 8.0 podigne format podataka na 8.4, ne postoji način da se to poništi. Binarni fajl verzije 8.0 odbiće da otvori taj `datadir`.

Jedini put nazad je **restore iz backupa** — što znači gubitak svega upisanog od trenutka backupa.

Zato svaka nadogradnja počinje istim korakom: potpun, proveren backup, i to onaj iz koga ste **stvarno vratili podatke bar jednom** (Modul 5).

### Putanje nadogradnje

Ne možete preskakati serije.

```
5.7  →  8.0  →  8.4  →  ...
```

Nije moguće ići direktno sa 5.7 na 8.4. Nije moguće ni sa proizvoljne 8.0 verzije direktno na 8.4 — prvo se ide na **poslednju** 8.0 verziju, pa onda na 8.4.

Unutar iste serije (8.0.30 → 8.0.36) nadogradnja je rutinska i uglavnom bezbolna.

Aktuelne putanje i datume podrške proverite na `endoflife.date/mysql` pre planiranja, jer se menjaju.

### `mysql_upgrade` više ne postoji

Ako pratite starije uputstvo, naići ćete na komandu `mysql_upgrade`. **Ona je uklonjena u MySQL 8.0.16.**

Od te verzije server sam obavlja nadogradnju rečnika podataka pri prvom startu nove verzije. Ponašanje kontroliše:

```sql
SELECT @@upgrade;
```

| Vrednost | Ponašanje |
|---|---|
| `AUTO` (podrazumevano) | nadogradi ako je potrebno |
| `NONE` | ne nadograđuj; server neće startovati ako je nadogradnja potrebna |
| `MINIMAL` | nadogradi samo rečnik, preskoči sistemske tabele i `sys` šemu |
| `FORCE` | nadogradi sve, i kada nije potrebno |

Za standardnu nadogradnju ostavite `AUTO`.

### Provera pre nadogradnje

MySQL Shell ima ugrađen alat koji vam štedi najviše vremena:

```bash
sudo apt install mysql-shell

mysqlsh -- util check-for-server-upgrade root@localhost \
  --target-version=8.4.0 \
  --output-format=TEXT
```

Izveštaj prijavljuje:

- zastarele i uklonjene sistemske promenljive u vašoj konfiguraciji,
- naloge sa autentikacionim pluginima kojih više nema,
- imena tabela i kolona koja postaju rezervisane reči,
- zastarele tipove podataka i skupove znakova,
- probleme sa kolacijama,
- tabele bez primarnog ključa.

**Pokrenite ovo pre svake nadogradnje između serija.** Rešite sve što prijavi kao `Error`, i pogledajte sve što prijavi kao `Warning`.

### Šta najčešće pukne

**`mysql_native_password`.** Zastareo u 8.0.34, uklonjen kasnije. Nalozi koji ga koriste prestaju da rade.

```sql
SELECT user, host FROM mysql.user WHERE plugin = 'mysql_native_password';
```

Prebacite ih pre nadogradnje (Modul 3), uz proveru da aplikacioni drajveri podržavaju `caching_sha2_password`.

**Uklonjene sistemske promenljive.** MySQL ne startuje ako u konfiguraciji nađe nepoznatu opciju. Nekoliko primera koje ćete sresti pri prelasku na 8.4:

- `expire_logs_days` → `binlog_expire_logs_seconds`
- `default_authentication_plugin` → `authentication_policy`
- `master_info_repository`, `relay_log_info_repository` → uklonjene
- sve promenljive sa `slave_` prefiksom → `replica_`

**Ovo je najčešći uzrok neuspelog starta nakon nadogradnje.** Alat za proveru ih hvata.

**Podrazumevana kolacija.** U MySQL 8 podrazumevana kolacija za `utf8mb4` je `utf8mb4_0900_ai_ci`, dok je ranije bila `utf8mb4_general_ci`. Ako neke tabele nose staru, a nove se prave sa novom, `JOIN` između njih prijavljuje grešku o mešanju kolacija.

```sql
SELECT table_schema, table_name, table_collation
FROM information_schema.tables
WHERE table_collation NOT LIKE 'utf8mb4_0900%'
  AND table_schema NOT IN ('mysql','information_schema','performance_schema','sys');
```

Odlučite unapred: ili sve prevodite na novu kolaciju, ili eksplicitno postavljate staru kao podrazumevanu.

### Metoda 1: nadogradnja na licu mesta

Najjednostavnija, uz planiran zastoj.

```bash
# 1. Backup i provera
/usr/local/bin/mysql-backup.sh
zcat /backup/mysql/full-*.sql.gz | tail -1 | grep "Dump completed"
pt-show-grants > /backup/grants-pre-upgrade.sql
sudo tar czf /backup/etc-mysql-pre-upgrade.tar.gz /etc/mysql/

# 2. Provera kompatibilnosti
mysqlsh -- util check-for-server-upgrade root@localhost --target-version=8.4.0

# 3. ČISTO gašenje — obavezno
mysql -e "SET GLOBAL innodb_fast_shutdown = 0;"
sudo systemctl stop mysql

# 4. Potvrdite da je gašenje bilo uredno
sudo tail -5 /var/log/mysql/error.log | grep "Shutdown complete"

# 5. Nadogradnja paketa
sudo apt update
sudo apt install --only-upgrade mysql-server

# 6. Start i praćenje
sudo systemctl start mysql
sudo tail -f /var/log/mysql/error.log
```

**Korak 3 nije opcion.** `innodb_fast_shutdown = 0` tera InnoDB da isprazni sve i završi sve pomoćne operacije (Modul 2). Nadogradnja preko nečisto ugašene instance je najbrži put do oštećenja.

U logu tražite linije o nadogradnji:

```
[System] [MY-013381] [Server] Server upgrade from '80036' to '80400' started.
[System] [MY-013381] [Server] Server upgrade from '80036' to '80400' completed.
[System] [MY-010931] [Server] /usr/sbin/mysqld: ready for connections.
```

Provera nakon starta:

```sql
SELECT VERSION();
SHOW DATABASES;
SELECT COUNT(*) FROM information_schema.tables;
```

```bash
mysqlcheck --all-databases --check
```

Ako Oracle repozitorijum treba prebaciti na drugu seriju:

```bash
sudo dpkg-reconfigure mysql-apt-config
sudo apt update
sudo apt install mysql-server
```

### Metoda 2: nadogradnja preko replike (preporučeno)

Ovim se zastoj svodi na minute i dobija se rollback.

Ključno pravilo: **replika se nadograđuje prva.** Novija replika ume da čita binlog starijeg primarnog servera; obrnuto ne važi.

```
Korak 1.  Postavite repliku na staroj verziji (Modul 8).
Korak 2.  Nadogradite REPLIKU na novu verziju.
Korak 3.  Pustite je da radi nekoliko dana i pratite.
          Ovo je pravo testiranje — na produkcijskom saobraćaju.
Korak 4.  Preuzimanje posla: promovišite repliku u primarni.
          Zastoj = trajanje preusmeravanja aplikacije, dakle minuti.
Korak 5.  Stari primarni prepravite u repliku novog.
Korak 6.  Kada ste sigurni, nadogradite i njega.
```

Prednost koja se ne vidi odmah: **korak 3 je test koji ne možete napraviti na staging serveru.** Replika obrađuje pravi produkcijski tok izmena, sa pravim podacima i pravim obimom.

Ako u tom periodu nešto pukne — jednostavno ne radite preuzimanje. Produkcija nastavlja na staroj verziji, a vi rešavate problem bez pritiska.

Za nadogradnju između serija (npr. 5.7 → 8.0) ovo je jedini razuman način.

### Plan povratka

Zapišite pre nego što počnete:

```
NADOGRADNJA db01: 8.0.36 → 8.4.x
Datum: ...
Backup: /backup/mysql/full-2026-08-30_0200.sql.gz (proveren restore 2026-08-29)
Grants: /backup/grants-pre-upgrade.sql
Konfiguracija: /backup/etc-mysql-pre-upgrade.tar.gz

Povratak (ako start ne uspe ili provere padnu):
  1. systemctl stop mysql
  2. apt install mysql-server=8.0.36-...   (downgrade paketa)
  3. mv /var/lib/mysql /var/lib/mysql.failed
  4. Restore iz backupa
  5. mysql < grants-pre-upgrade.sql
  6. Procenjeno trajanje povratka: 4h 30min

Tačka bez povratka: prvi uspešan start nove verzije.
Do tada se vraćamo bez gubitka. Posle toga gubimo sve od 02:00.
```

Poslednja dva reda su najvažnija u dokumentu. Neka svi koji odlučuju znaju gde je ta tačka.

---

## 44. Migracija na novi server

### Tri metode

| Metoda | Zastoj | Kada |
|---|---|---|
| Dump i restore | sati | male baze, promena verzije, promena arhitekture |
| Fizička kopija | desetine minuta | ista verzija, velike baze |
| Preko replikacije | **minuti** | gotovo uvek najbolji izbor |

### Metoda 1: dump i restore

```bash
# na starom
mysqldump --single-transaction --source-data=2 \
  --routines --events --no-tablespaces \
  --all-databases | gzip > /backup/migracija.sql.gz

pt-show-grants > /backup/grants.sql

# prenos
rsync -avP /backup/migracija.sql.gz /backup/grants.sql novi-server:/tmp/

# na novom
zcat /tmp/migracija.sql.gz | mysql
mysql < /tmp/grants.sql
```

Jednostavno, ali zastoj traje koliko i restore — a to znate, jer ste ga izmerili u Modulu 5.

Prednost: radi između različitih verzija i različitih platformi.

### Metoda 2: fizička kopija

XtraBackup ili CLONE po proceduri iz Modula 5. Brže, ali zahteva istu verziju.

### Metoda 3: preko replikacije (preporučeno)

Ovo je metoda koju treba da koristite kad god možete.

```
1. Podignite novi server i podesite ga (Moduli 1–4).
2. Napunite ga podacima: CLONE, XtraBackup ili dump (Modul 8).
3. Postavite replikaciju stari → novi.
4. Pustite je da radi dan-dva. Pratite zaostajanje, proveravajte
   konzistentnost sa pt-table-checksum.
5. U dogovorenom terminu:
   a) zaustavite upise na starom (read_only = ON ili gašenje aplikacije),
   b) sačekajte da novi primeni sve (Executed = Retrieved GTID),
   c) promovišite novi (Modul 8, poglavlje 42),
   d) preusmerite aplikaciju.
6. Postavite obrnutu replikaciju novi → stari, kao rezervu za povratak.
7. Nakon nedelju dana bez problema, ugasite stari.
```

Zastoj je koraci 5a do 5d — obično dva do pet minuta.

Korak 6 je ono što ovu metodu čini bezbednom: **imate radni put nazad.** Ako se posle dva sata pokaže da nešto ne valja, vraćate se na stari server koji je u međuvremenu bio ažuran.

Ovo radi i **između verzija** — MySQL 8.0 replika može čitati sa 5.7 primarnog. Time se migracija i nadogradnja obavljaju u jednom potezu.

### Kontrolna lista migracije

Ovo je deo koji se najčešće zaboravi. Podaci su lakši deo posla.

```
PODACI
  [ ] Sve baze prenete
  [ ] Broj tabela se poklapa
  [ ] Procedure, funkcije, trigeri, eventi
  [ ] Broj redova u ključnim tabelama

NALOZI I PRISTUP
  [ ] Svi MySQL nalozi (pt-show-grants)
  [ ] Host polja odgovaraju NOVIM IP adresama
  [ ] Uloge i podrazumevane uloge
  [ ] Test prijave iz aplikacije

KONFIGURACIJA
  [ ] /etc/mysql prenet i prilagođen novom hardveru
  [ ] innodb_buffer_pool_size prema NOVOM RAM-u (Modul 7)
  [ ] server_id različit
  [ ] bind_address, firewall (Modul 4)
  [ ] LimitNOFILE, kernel parametri (Modul 7)

BEZBEDNOST
  [ ] TLS sertifikati sa ISPRAVNIM imenom novog hosta u SAN-u
  [ ] AppArmor pravila za nestandardne putanje
  [ ] UFW pravila

OPERATIVNO
  [ ] Backup skript i timer
  [ ] Streaming binlogova
  [ ] Monitoring i alarmi
  [ ] logrotate
  [ ] Cron poslovi koji dodiruju bazu

APLIKACIJA
  [ ] Konfiguracija veze (DNS, IP, port)
  [ ] Kolacije se poklapaju
  [ ] Test upisa i čitanja
  [ ] Test najsloženije funkcionalnosti

POVRATAK
  [ ] Obrnuta replikacija radi
  [ ] Procedura povratka zapisana
  [ ] Stari server se NE gasi bar nedelju dana
```

Stavka o TLS sertifikatima je klasičan previd: sertifikat glasi na `db01.interno.local`, a novi server je `db02`, pa `VERIFY_IDENTITY` puca (Modul 4). Izdajte nov sertifikat pre migracije, ne posle.

---

## 45. `ALTER TABLE` na velikoj tabeli

### Zašto je ovo najopasnija svakodnevna operacija

Izmena šeme deluje bezazleno — jedan red SQL-a. Na tabeli od pedeset miliona redova to je operacija koja može trajati sat i za to vreme zaustaviti celu aplikaciju.

Podsetnik iz Modula 6: uzrok nije samo trajanje, nego **metadata lock**. Dok `ALTER` traje, svaki naredni upit nad tom tabelom čeka. Za minut imate stotine konekcija u čekanju, udarate u `max_connections`, i aplikacija stoji.

### Tri algoritma

```sql
SELECT @@default_storage_engine;
```

MySQL bira jedan od tri načina izvršavanja:

| Algoritam | Šta radi | Trajanje |
|---|---|---|
| **`INSTANT`** | menja samo metapodatke | trenutno |
| **`INPLACE`** | menja tabelu na mestu, uglavnom bez blokiranja | srazmerno veličini |
| **`COPY`** | pravi novu kopiju tabele | **dugo, blokira upise** |

**Šta može `INSTANT`** (8.0.12+, prošireno u 8.0.29+):

- dodavanje kolone,
- brisanje kolone (8.0.29+),
- preimenovanje kolone,
- postavljanje i uklanjanje podrazumevane vrednosti,
- proširivanje `ENUM` liste na kraju.

**Šta radi `INPLACE`:**

- dodavanje i brisanje indeksa,
- preimenovanje tabele,
- dodavanje stranog ključa.

**Šta pada na `COPY`:**

- promena tipa podataka kolone,
- promena skupa znakova ili kolacije,
- dodavanje primarnog ključa,
- promena redosleda kolona.

### Navodite algoritam eksplicitno — uvek

Ovo je najkorisnija navika iz celog poglavlja.

```sql
ALTER TABLE narudzbine
  ADD COLUMN napomena VARCHAR(255),
  ALGORITHM=INSTANT;
```

Ako operacija **ne može** biti `INSTANT`, MySQL vraća grešku umesto da tiho pređe na `COPY`:

```
ERROR 1845 (0A000): ALGORITHM=INSTANT is not supported for this operation.
Try ALGORITHM=COPY.
```

Ta greška je dobra vest. Dobili ste je za milisekundu, umesto da za sat vremena otkrijete da vam je aplikacija stala.

Isto važi i za zaključavanje:

```sql
ALTER TABLE narudzbine ADD INDEX idx_status (status),
  ALGORITHM=INPLACE, LOCK=NONE;
```

`LOCK=NONE` znači "ako ovo zahteva zaključavanje, odustani".

**Pravilo: nikada ne izvršavajte goli `ALTER TABLE` na produkciji.** Uvek navedite algoritam i nivo zaključavanja, pa neka vam server kaže šta zaista sledi.

### Metadata lock — trik koji sprečava lančani zastoj

Čak i `INSTANT` operacija traži kratak metadata lock. Ako u tom trenutku neka duga transakcija drži tabelu, `ALTER` čeka — i **iza njega se reda sve ostalo**.

Rešenje je da `ALTER` odustane brzo umesto da čeka:

```sql
SET SESSION lock_wait_timeout = 5;
ALTER TABLE narudzbine ADD COLUMN napomena VARCHAR(255), ALGORITHM=INSTANT;
```

Ako u pet sekundi ne dobije lock, naredba pada sa greškom, a aplikacija nastavlja normalno. Vi pronađete dugu transakciju (Modul 6), rešite je i pokušate ponovo.

Bez ovoga, `ALTER` može čekati satima i za to vreme držati celu aplikaciju u zastoju — a najgore je što u `processlist`-u to izgleda kao da `ALTER` "ne radi ništa".

Provera pre pokretanja:

```sql
SELECT trx_id, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS traje_sek,
       trx_mysql_thread_id, LEFT(trx_query, 60) AS upit
FROM information_schema.innodb_trx
ORDER BY trx_started;
```

Ako postoji transakcija starija od nekoliko minuta — rešite je pre nego što krenete.

### Uticaj na replikaciju

Ovo se često previdi: `ALTER` koji na primarnom traje sat vremena, na replici se takođe izvršava sat vremena — **i za to vreme blokira applier**.

Rezultat: replika zaostaje sat vremena. Ako je ona vaš izvor za backup ili server za izveštaje, i to stoji.

Zato se velike izmene šeme ne rade običnim `ALTER`-om.

### `pt-online-schema-change`

Alat iz Percona Toolkit-a koji menja tabelu bez dugog zaključavanja.

Kako radi: napravi praznu kopiju tabele sa novom strukturom, postavi trigere koji prosleđuju sve izmene u kopiju, kopira postojeće redove u malim delovima, pa na kraju atomično zameni imena tabela.

```bash
pt-online-schema-change \
  --alter "ADD COLUMN status VARCHAR(20) NOT NULL DEFAULT 'novo'" \
  D=shop,t=narudzbine \
  --chunk-time=0.5 \
  --max-load="Threads_running=25" \
  --critical-load="Threads_running=50" \
  --max-lag=30 \
  --check-interval=5 \
  --alter-foreign-keys-method=auto \
  --dry-run
```

**Uvek prvo `--dry-run`.** Kada ste zadovoljni ispisom, zamenite sa `--execute`.

Ključne opcije:

- **`--chunk-time=0.5`** — alat sam podešava veličinu porcije tako da svaka traje oko pola sekunde. Time se opterećenje drži ujednačenim.
- **`--max-load`** — pauzira kada `Threads_running` pređe prag.
- **`--critical-load`** — potpuno odustaje ako opterećenje postane opasno.
- **`--max-lag`** — pauzira ako replike zaostanu više od 30 sekundi. **Ovo je razlog zbog kojeg alat postoji.**

Ograničenja koja morate znati unapred:

- tabela mora imati primarni ključ ili jedinstveni indeks,
- treba vam **slobodnog prostora koliko i cela tabela**,
- trigeri unose dodatni trošak na svaki upis dok operacija traje,
- strani ključevi su komplikovani; pročitajte dokumentaciju za `--alter-foreign-keys-method`,
- ako tabela već ima trigere, proverite kompatibilnost sa verzijom alata.

### `gh-ost` kao alternativa

Alat koji je razvio GitHub, sa drugačijim pristupom: umesto trigera, čita **binlog** da bi pratio izmene.

Prednosti nad `pt-online-schema-change`:

- nema trigera, dakle nema dodatnog troška na upise,
- može da se pauzira i nastavi,
- može se pokrenuti sa **treće mašine**, bez opterećenja baze,
- podešavanje brzine u toku rada, bez prekidanja.

```bash
gh-ost \
  --host=10.0.1.10 --user=gh-ost --password='...' \
  --database=shop --table=narudzbine \
  --alter="ADD COLUMN status VARCHAR(20) NOT NULL DEFAULT 'novo'" \
  --allow-on-master \
  --max-load='Threads_running=25' \
  --critical-load='Threads_running=50' \
  --max-lag-millis=1500 \
  --execute
```

Zahteva `binlog_format = ROW`, što ionako imate (Modul 8).

Za velike tabele, `gh-ost` je danas obično bolji izbor.

### Kako birati

```
Tabela manja od ~1 GB
  → običan ALTER sa eksplicitnim ALGORITHM i lock_wait_timeout

Operacija može biti INSTANT
  → običan ALTER sa ALGORITHM=INSTANT, bez obzira na veličinu

Velika tabela, operacija zahteva COPY
  → gh-ost (prvi izbor) ili pt-online-schema-change

Postoji planiran termin održavanja i tabela nije ogromna
  → običan ALTER, ali izmerite trajanje na kopiji pre toga
```

I uvek: **izmerite na kopiji produkcijskih podataka pre nego što dirate produkciju.** Vreme izmereno na test bazi od 100 MB ne govori ništa o tabeli od 100 GB.

---

## 46. Rutinsko održavanje

### Fragmentacija

Kada brišete redove, InnoDB ne vraća prostor operativnom sistemu. Fajl ostaje iste veličine, a unutar njega ostaju praznine.

```sql
SELECT table_schema, table_name,
       ROUND((data_length + index_length) / 1024 / 1024) AS mb,
       ROUND(data_free / 1024 / 1024) AS slobodno_mb,
       ROUND(data_free * 100.0 / NULLIF(data_length + index_length, 0), 1) AS pct
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql','information_schema','performance_schema','sys')
  AND data_free > 100 * 1024 * 1024
ORDER BY data_free DESC
LIMIT 20;
```

**Napomena o pouzdanosti:** `data_free` je procena i ume da bude netačna. Koristite je kao signal, ne kao merilo. Ako pokazuje da je 60% tabele prazno, vredi pogledati; ako pokazuje 8%, ignorišite.

### `OPTIMIZE TABLE` — retko, ne rutinski

```sql
OPTIMIZE TABLE shop.narudzbine;
```

Za InnoDB ovo znači: potpuna rekonstrukcija tabele (`ALTER TABLE ... FORCE`) plus osvežavanje statistike.

Šta to podrazumeva:

- treba **slobodnog prostora koliko i cela tabela**,
- traje srazmerno veličini,
- u MySQL 8 je uglavnom "online" za InnoDB, ali i dalje prepisuje sve podatke i troši ozbiljan I/O.

**Kada ima smisla:** nakon što ste obrisali veliki procenat redova iz tabele i želite prostor nazad.

**Kada nema smisla:** kao mesečni cron posao na svim tabelama. To je uzalud potrošen I/O na tabelama koje se normalno pune i prazne — fragmentiran prostor će se ionako ponovo iskoristiti.

Vidim ovo često kao nasleđen cron zadatak. Ako ga zateknete, prvo pitajte zašto postoji.

### `ANALYZE TABLE` — jeftino i korisno

```sql
ANALYZE TABLE shop.narudzbine;
```

Osvežava statistiku indeksa koju optimizator koristi za planiranje upita. Traje sekundama.

```sql
SELECT @@innodb_stats_auto_recalc, @@innodb_stats_persistent,
       @@innodb_stats_persistent_sample_pages;
```

Automatsko osvežavanje je uključeno i pokreće se kada se promeni oko 10% redova, pa ručno pokretanje uglavnom nije potrebno.

Kada jeste korisno:

- odmah nakon masovnog učitavanja podataka,
- nakon restore-a,
- kada optimizator odjednom bira loš plan za upit koji je ranije radio dobro.

Za velike tabele sa neravnomernom raspodelom podataka vredi podići uzorak:

```ini
[mysqld]
innodb_stats_persistent_sample_pages = 64
```

Mala napomena: `ANALYZE TABLE` uzima kratak metadata lock i poništava keširane planove za tu tabelu. Na veoma opterećenoj tabeli to ume da izazove kratak zastoj. Nije razlog da ga izbegavate, ali jeste razlog da ga ne pokrećete nad svim tabelama u špicu.

### `CHECK TABLE` — provera integriteta

```sql
CHECK TABLE shop.narudzbine;
```

Ili sve odjednom:

```bash
mysqlcheck --all-databases --check
```

Traje dugo i uzima read lock. **Pokrećite na replici**, ne na primarnom.

Kada: nakon pada servera, nakon sumnje na problem sa diskom, nakon restore-a, i povremeno kao provera.

### Neiskorišćeni i duplikati indeksa

Indeksi ubrzavaju čitanje, a usporavaju upis i troše prostor. Neiskorišćen indeks je čist gubitak.

```sql
SELECT * FROM sys.schema_unused_indexes;
```

**Budite oprezni sa ovim spiskom.** Statistika se resetuje pri restartu servera, a neki indeks se možda koristi samo za mesečni izveštaj. Pravilo: pustite server da radi bar mesec dana bez restarta pre nego što se pouzdate u ovaj upit, i konsultujte developera pre brisanja.

Duplikati:

```bash
pt-duplicate-key-checker --databases=shop
```

Ovaj alat nalazi indekse koji su podskup drugih — npr. indeks na `(a)` kada već postoji indeks na `(a, b)`. Ti su bezbedniji za brisanje.

### Rast tabela i particije

Tabele sa logovima i istorijom rastu neograničeno. Klasičan pristup:

```sql
DELETE FROM access_log WHERE kreirano < NOW() - INTERVAL 90 DAY;
```

Ovo je loše iz tri razloga: traje dugo, pravi ogroman binlog koji zaostaje replikama, i **ne oslobađa prostor** (vidi fragmentaciju).

Bolje rešenje je particionisanje po datumu:

```sql
ALTER TABLE access_log
PARTITION BY RANGE (TO_DAYS(kreirano)) (
  PARTITION p202606 VALUES LESS THAN (TO_DAYS('2026-07-01')),
  PARTITION p202607 VALUES LESS THAN (TO_DAYS('2026-08-01')),
  PARTITION p202608 VALUES LESS THAN (TO_DAYS('2026-09-01')),
  PARTITION pmax    VALUES LESS THAN MAXVALUE
);
```

Brisanje starih podataka tada postaje:

```sql
ALTER TABLE access_log DROP PARTITION p202606;
```

**Trenutno, bez binloga pun redova, i prostor se stvarno oslobađa.**

Dizajn particionisanja je posao developera, ali predlog obično dolazi od vas — jer vi vidite da disk raste.

Praćenje rasta:

```sql
SELECT table_name,
       ROUND((data_length + index_length) / 1024 / 1024) AS mb,
       table_rows
FROM information_schema.tables
WHERE table_schema = 'shop'
ORDER BY data_length + index_length DESC
LIMIT 15;
```

Snimajte ovo mesečno i čuvajte istoriju. Trend rasta je ono što vam govori kada ćete ostati bez diska — mesecima unapred.

### Predlog kalendara održavanja

| Učestalost | Zadatak |
|---|---|
| Dnevno | backup, provera starosti backupa, provera replikacije |
| Nedeljno | pregled slow loga, `pt-query-digest`, provera prostora |
| Mesečno | `pt-table-checksum`, snimak veličina tabela, pregled error loga |
| Kvartalno | vežba restore-a, revizija naloga, provera isteka sertifikata |
| Polugodišnje | pregled neiskorišćenih indeksa, planiranje nadogradnje |
| Po potrebi | `OPTIMIZE` nakon velikog brisanja, `ANALYZE` nakon masovnog upisa |

Ovo nije spisak koji treba slepo primeniti — to je polazna tačka koju prilagodite svom sistemu. Ali ako trenutno nemate nikakav kalendar, ovaj je bolji od toga.

---

## Kontrolna lista na kraju modula

```bash
# 1. Znate na koju verziju idete i kojim putem
mysql -e "SELECT VERSION();"
mysqlsh -- util check-for-server-upgrade root@localhost --target-version=8.4.0

# 2. Nema naloga sa uklonjenim autentikacionim pluginom
mysql -e "SELECT user, host FROM mysql.user WHERE plugin='mysql_native_password';"

# 3. Konfiguracija ne sadrži uklonjene promenljive
sudo mysqld --validate-config

# 4. Postoji proveren backup i zapisan plan povratka
ls -la /backup/mysql/ | tail -3

# 5. Nema dugih transakcija pre izmene šeme
mysql -e "SELECT COUNT(*) FROM information_schema.innodb_trx
          WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 60;"

# 6. Alati za online izmenu šeme su instalirani
which pt-online-schema-change gh-ost

# 7. Znate koje su tabele najveće i kako rastu
mysql -e "SELECT table_schema, table_name,
          ROUND((data_length+index_length)/1024/1024) AS mb
          FROM information_schema.tables
          ORDER BY data_length+index_length DESC LIMIT 10;"

# 8. Fragmentacija je pregledana (kao signal, ne kao merilo)

# 9. Neiskorišćeni indeksi su pregledani nakon meseca rada
mysql -e "SELECT * FROM sys.schema_unused_indexes LIMIT 20;"

# 10. Kalendar održavanja postoji i neko ga sprovodi
```

---

## Vežbe

**Vežba 1 — Provera pre nadogradnje**
Na test serveru napravite bazu sa namerno problematičnim elementima: nalog sa `mysql_native_password`, tabelu bez primarnog ključa, kolonu sa imenom koje je rezervisana reč u novijoj verziji. Pokrenite alat za proveru i objasnite svaki nalaz.

**Vežba 2 — Nadogradnja i tačka bez povratka**
Nadogradite test server na noviju seriju. Zatim pokušajte da ga vratite na staru verziju bez restore-a i dokumentujte tačno gde i kako to pukne. Ovo je vežba koja objašnjava zašto je backup obavezan.

**Vežba 3 — Nadogradnja preko replike**
Postavite repliku, nadogradite **samo nju**, i pustite je da replicira sa starijeg primarnog nekoliko sati. Zatim izvršite preuzimanje i izmerite zastoj. Uporedite sa vremenom nadogradnje na licu mesta iz vežbe 2.

**Vežba 4 — Tri algoritma**
Na tabeli sa milion redova izvršite tri operacije, svaku sa eksplicitnim `ALGORITHM`: dodavanje kolone (`INSTANT`), dodavanje indeksa (`INPLACE`), promenu tipa kolone (`COPY`). Izmerite trajanje svake i posmatrajte `processlist` iz druge sesije.

**Vežba 5 — Metadata lock zastoj i `lock_wait_timeout`**
U jednoj sesiji otvorite transakciju sa `SELECT` bez `COMMIT`. U drugoj pokrenite `ALTER TABLE` bez `lock_wait_timeout`. U trećoj pokrenite obične `SELECT` upite i posmatrajte kako se redaju. Zatim ponovite ceo eksperiment sa `SET SESSION lock_wait_timeout = 5` i uporedite ishod.

**Vežba 6 — `pt-online-schema-change` pod opterećenjem**
Pokrenite skript koji neprekidno upisuje u tabelu od nekoliko miliona redova. Dok radi, izvršite izmenu šeme pomoću `pt-online-schema-change`. Pratite `Threads_running`, zaostajanje replike i vreme odziva aplikacije. Zatim ponovite isto sa običnim `ALTER`-om i uporedite.

**Vežba 7 — Fragmentacija u praksi**
Napravite tabelu od 2 GB, obrišite 70% redova i uporedite `du` na `.ibd` fajlu sa vrednostima iz `information_schema`. Pokrenite `OPTIMIZE TABLE`, izmerite trajanje i ponovo uporedite. Zaključite kada se ovo isplati.

**Vežba 8 — Particije umesto `DELETE`**
Napravite tabelu sa logovima od nekoliko miliona redova. Izmerite trajanje i veličinu binloga za `DELETE` starijih od 90 dana. Zatim particionišite tabelu i izmerite isto za `DROP PARTITION`. Uporedite oba broja.

**Vežba 9 — Kompletna migracija**
Migrirajte bazu na novi server metodom replikacije, prolazeći kroz celu kontrolnu listu iz poglavlja 44. Namerno preskočite jednu stavku (npr. TLS sertifikat ili host polja u nalozima) i zabeležite kako i kada se to manifestuje.

---

## Šta sledi

U **Modulu 10** ulazimo u kvarove — modul napisan da se čita u tri ujutru.

Dijagnostika servera koji neće da startuje, u pet koraka. Šta uraditi kada javi "Too many connections" — odmah, a šta posle. Pun disk: kako MySQL reaguje i kako osloboditi prostor bez gubitka podataka. Oporavak od oštećenja pomoću `innodb_force_recovery` i šta svaki nivo zaista radi. I OOM killer koji je ubio `mysqld` — analiza i prevencija.

To je modul koji ćete najređe čitati i najviše ceniti kada zatreba.
