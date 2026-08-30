# Dodatak: Minimum SQL-a za sysadmina

*MySQL za Linux administratore*

---

## Uvod

Kroz ceo kurs stoji ista podela: SQL nije vaš posao. Optimizacija upita, dizajn šeme i indeksi pripadaju developeru ili DBA.

Ali postoji minimum bez koga ne možete raditi svoj deo. Treba da umete da:

- proverite da li je restore stvarno prošao,
- pročitate bazu koju niste vi pravili,
- napravite ručnu izmenu bez straha da ćete obrisati pola tabele,
- pogledate plan izvršavanja dovoljno da kažete "ovaj upit skenira dva miliona redova",
- razgovarate sa developerom tako da vas obojica razumete iz prve.

To je ono što je u ovom dodatku. Ništa više.

---

## 56. Deset SQL naredbi koje ćete stvarno koristiti

### 1. `SELECT` — čitanje, uvek sa `LIMIT`

```sql
SELECT * FROM shop.korisnici LIMIT 10;
```

**Navika koja se isplati: uvek stavite `LIMIT`, čak i kad mislite da tabela ima malo redova.** `SELECT * FROM logovi;` nad tabelom od pedeset miliona redova je siguran način da opteretite server i zatrpate sopstveni terminal.

Za tabele sa mnogo kolona, vertikalni ispis je čitljiviji:

```sql
SELECT * FROM shop.korisnici LIMIT 3\G
```

```
*************************** 1. row ***************************
        id: 1
       ime: Marko
     email: marko@primer.rs
  kreirano: 2024-03-15 10:22:41
```

Ovo je jedna od najkorisnijih sitnica u `mysql` klijentu.

Filtriranje i sortiranje:

```sql
SELECT id, ime, kreirano
FROM shop.korisnici
WHERE kreirano > '2026-08-01'
ORDER BY kreirano DESC
LIMIT 20;
```

Operatori koji vam trebaju: `=`, `<>`, `>`, `<`, `LIKE 'nesto%'`, `IN (1,2,3)`, `BETWEEN`, `IS NULL`, `IS NOT NULL`, i spajanje sa `AND` / `OR`.

### 2. `COUNT` i osnovne agregacije

Ovo je vaš alat za proveru nakon restore-a (Modul 5):

```sql
SELECT COUNT(*) FROM shop.narudzbine;

SELECT MAX(id), MIN(kreirano), MAX(kreirano) FROM shop.narudzbine;

SELECT status, COUNT(*) AS broj
FROM shop.narudzbine
GROUP BY status
ORDER BY broj DESC;
```

Poslednji oblik — `GROUP BY` sa brojanjem — pokriva devedeset odsto onoga što ćete ikada trebati.

Napomena o performansama: `COUNT(*)` nad velikom InnoDB tabelom **nije trenutan** — server stvarno prebroji redove. Za grubu procenu bez opterećenja:

```sql
SELECT table_rows FROM information_schema.tables
WHERE table_schema = 'shop' AND table_name = 'narudzbine';
```

Ta vrednost je procena i može odstupati i po 20%, ali je besplatna.

### 3. `SHOW` i `DESCRIBE` — orijentacija

```sql
SHOW DATABASES;
USE shop;
SHOW TABLES;
DESCRIBE korisnici;
SHOW CREATE TABLE korisnici\G
SHOW INDEX FROM korisnici;
```

`SHOW CREATE TABLE` je najkorisnija od njih — daje kompletnu definiciju: kolone, tipove, indekse, strane ključeve, engine, kodiranje.

### 4. Transakcija — najvažnija stvar u ovom dodatku

Ovo je jedina tehnika koja vas štiti od ručne greške na produkciji. Naučite je napamet.

```sql
START TRANSACTION;

-- 1. Prvo POGLEDAJTE šta ćete dirati
SELECT id, status FROM narudzbine WHERE status = 'na_cekanju' AND kreirano < '2026-01-01';

-- 2. Izvršite izmenu
UPDATE narudzbine SET status = 'otkazano'
WHERE status = 'na_cekanju' AND kreirano < '2026-01-01';
-- Query OK, 142 rows affected

-- 3. Proverite da li je broj redova onaj koji ste očekivali
--    i da li rezultat izgleda ispravno
SELECT status, COUNT(*) FROM narudzbine GROUP BY status;

-- 4. Tek onda potvrdite
COMMIT;
```

Ako u bilo kom trenutku nešto ne izgleda dobro:

```sql
ROLLBACK;
```

Sve se poništava kao da se nije desilo.

**Ključni detalj: broj redova koji `UPDATE` prijavi mora da se poklapa sa brojem koji je `SELECT` vratio u koraku 1.** Ako `SELECT` kaže 142, a `UPDATE` kaže 89.412 — imate grešku u `WHERE` klauzuli i upravo ste je uhvatili pre nego što je postala incident.

Dodatna mera opreza za nepovratne operacije — kopija tabele pre izmene:

```sql
CREATE TABLE narudzbine_backup_20260830 AS SELECT * FROM narudzbine;
```

Ovo ne kopira indekse, ali kopira podatke. Za tabelu do nekoliko gigabajta je najbrža polisa osiguranja koja postoji. Obrišite je za nedelju dana.

### 5. `--safe-updates` — mreža ispod žice

Podsetnik iz Modula 1:

```ini
# ~/.my.cnf
[mysql]
safe-updates
```

Sa ovim uključenim, `UPDATE` ili `DELETE` bez `WHERE` klauzule (ili bez `LIMIT`) biva odbijen:

```
ERROR 1175 (HY000): You are using safe update mode and you tried to update
a table without a WHERE that uses a KEY column.
```

Jednom kada vas spasi, isplatilo se.

### 6. `INSERT`, `UPDATE`, `DELETE`

```sql
INSERT INTO korisnici (ime, email) VALUES ('Ana', 'ana@primer.rs');

UPDATE korisnici SET email = 'novi@primer.rs' WHERE id = 42;

DELETE FROM sesije WHERE istice < NOW();
```

Pravila koja važe bez izuzetka:

- **uvek `WHERE`**, osim ako baš ne želite sve redove,
- **uvek u transakciji** kada radite ručno na produkciji,
- **`DELETE` nad milionima redova radite u serijama**, jer jedan veliki `DELETE` pravi ogroman binlog i zaostajanje replika (Modul 8):

```sql
-- ponavljajte dok ne vrati 0 redova
DELETE FROM access_log WHERE kreirano < '2026-01-01' LIMIT 10000;
```

A još bolje — particije i `DROP PARTITION` (Modul 9).

### 7. `JOIN` — koliko vam treba

Nećete pisati složene upite, ali ćete ih čitati.

```sql
SELECT n.id, n.ukupno, k.ime, k.email
FROM narudzbine n
JOIN korisnici k ON n.korisnik_id = k.id
WHERE n.status = 'na_cekanju'
LIMIT 20;
```

Čitanje: uzmi redove iz `narudzbine`, i za svaki nađi odgovarajući red u `korisnici` gde se `korisnik_id` poklapa sa `id`.

`JOIN` (odnosno `INNER JOIN`) vraća samo redove koji imaju par sa obe strane. `LEFT JOIN` vraća sve iz leve tabele, sa `NULL` vrednostima gde para nema — što je korisno za pronalaženje "siročića":

```sql
-- narudžbine čiji korisnik ne postoji
SELECT n.id, n.korisnik_id
FROM narudzbine n
LEFT JOIN korisnici k ON n.korisnik_id = k.id
WHERE k.id IS NULL
LIMIT 20;
```

Ovaj upit je koristan nakon restore-a ili nakon sumnje na razilaženje podataka.

### 8. `EXPLAIN` — taman toliko da prepoznate problem

Nećete optimizovati upite. Ali kada nađete spor upit (Modul 6), `EXPLAIN` vam daje dokaz koji šaljete dalje.

```sql
EXPLAIN SELECT * FROM narudzbine WHERE status = 'na_cekanju'\G
```

```
           id: 1
  select_type: SIMPLE
        table: narudzbine
         type: ALL
possible_keys: NULL
          key: NULL
         rows: 2451822
        Extra: Using where
```

Četiri polja koja treba da razumete:

**`type`** — kako se pristupa tabeli, od najgoreg ka najboljem:

| Vrednost | Značenje |
|---|---|
| `ALL` | **pun pregled tabele** — čita svaki red |
| `index` | pun pregled indeksa — bolje, ali i dalje sve |
| `range` | opseg po indeksu — u redu |
| `ref` | pretraga po indeksu — dobro |
| `eq_ref`, `const` | jedan red — odlično |

`ALL` na velikoj tabeli je nalaz.

**`key`** — koji indeks je iskorišćen. `NULL` znači da nijedan.

**`rows`** — procena koliko redova server očekuje da pregleda.

**`Extra`** — dodatne informacije:

| Vrednost | Značenje |
|---|---|
| `Using index` | dobro — sve je pročitano iz indeksa |
| `Using where` | normalno |
| `Using filesort` | sortiranje bez indeksa; može biti sporo |
| `Using temporary` | pravi privremenu tabelu; često sporo |

Za stvarne brojke umesto procene:

```sql
EXPLAIN ANALYZE SELECT * FROM narudzbine WHERE status = 'na_cekanju'\G
```

Ovo **stvarno izvršava** upit i prikazuje izmereno vreme po koracima. Ne pokrećite ga nad upitom koji traje deset minuta.

Vaš nalaz developeru glasi otprilike: *"Ovaj upit ima `type: ALL`, `key: NULL` i `rows: 2451822`. Nema indeksa na koloni `status`."* To je sve što treba da kažete.

### 9. `information_schema` — metapodaci

Ovo je vaša najkorišćenija "tabela" i praktično svaki administratorski upit u kursu dolazi odavde.

```sql
-- veličine baza
SELECT table_schema,
       ROUND(SUM(data_length + index_length) / 1024 / 1024, 1) AS mb
FROM information_schema.tables
GROUP BY table_schema ORDER BY mb DESC;

-- najveće tabele
SELECT table_schema, table_name,
       ROUND((data_length + index_length) / 1024 / 1024, 1) AS mb,
       table_rows
FROM information_schema.tables
WHERE table_type = 'BASE TABLE'
ORDER BY data_length + index_length DESC LIMIT 15;

-- tabele koje nisu InnoDB
SELECT table_schema, table_name, engine
FROM information_schema.tables WHERE engine <> 'InnoDB';

-- aktivne transakcije
SELECT trx_id, trx_started, trx_mysql_thread_id, LEFT(trx_query, 60)
FROM information_schema.innodb_trx ORDER BY trx_started;
```

### 10. `sys` šema — gotovi odgovori

`sys` je skup pogleda nad `performance_schema`, napravljenih da budu čitljivi.

```sql
-- ko je trenutno aktivan
SELECT * FROM sys.session WHERE command <> 'Sleep' ORDER BY time DESC;

-- najskuplji upiti
SELECT query, exec_count, total_latency, rows_examined_avg
FROM sys.statement_analysis ORDER BY total_latency DESC LIMIT 10;

-- neiskorišćeni indeksi
SELECT * FROM sys.schema_unused_indexes;

-- tabele koje se pregledaju u celini
SELECT * FROM sys.schema_tables_with_full_table_scans LIMIT 10;

-- gde odlazi memorija
SELECT event_name, current_alloc FROM sys.memory_global_by_current_bytes LIMIT 10;
```

Ako zapamtite samo dva izvora — neka to budu `information_schema.tables` i `sys.statement_analysis`.

---

## 57. Kako pročitati šemu koju niste vi pravili

Situacija: preuzeli ste server, u bazi je dvesta tabela, dokumentacije nema, a osoba koja ju je pravila više ne radi tu.

Evo redosleda koji daje sliku za pola sata.

### Korak 1: opseg

```sql
SHOW DATABASES;
```

```sql
SELECT table_schema,
       COUNT(*) AS tabela,
       ROUND(SUM(data_length + index_length) / 1024 / 1024, 1) AS mb
FROM information_schema.tables
WHERE table_type = 'BASE TABLE'
  AND table_schema NOT IN ('mysql','information_schema','performance_schema','sys')
GROUP BY table_schema
ORDER BY mb DESC;
```

Sada znate koliko baza, koliko tabela i koja je velika.

### Korak 2: koje tabele su bitne

Veličina nije isto što i važnost, ali je dobar početak:

```sql
SELECT table_name,
       ROUND((data_length + index_length) / 1024 / 1024, 1) AS mb,
       table_rows
FROM information_schema.tables
WHERE table_schema = 'shop' AND table_type = 'BASE TABLE'
ORDER BY data_length + index_length DESC
LIMIT 20;
```

Bolji pokazatelj je **koje se tabele stvarno koriste**:

```sql
SELECT object_schema, object_name,
       count_read, count_write, count_fetch
FROM performance_schema.table_io_waits_summary_by_table
WHERE object_schema = 'shop'
ORDER BY count_star DESC
LIMIT 20;
```

Ovo pokazuje stvarni saobraćaj po tabeli, od poslednjeg restarta servera.

Obrnuto je takođe korisno — tabele koje niko ne dodiruje:

```sql
SELECT object_schema, object_name
FROM performance_schema.table_io_waits_summary_by_table
WHERE object_schema = 'shop' AND count_star = 0
ORDER BY object_name;
```

To su kandidati za mrtve tabele. **Ne brišite ih na osnovu ovog upita** — statistika se resetuje pri restartu, a neka tabela se možda koristi jednom mesečno. Ali jeste spisak za razgovor sa developerom.

### Korak 3: struktura pojedinačne tabele

```sql
SHOW CREATE TABLE shop.narudzbine\G
```

```sql
CREATE TABLE `narudzbine` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `korisnik_id` bigint NOT NULL,
  `status` varchar(20) NOT NULL DEFAULT 'novo',
  `ukupno` decimal(10,2) NOT NULL,
  `kreirano` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `idx_korisnik` (`korisnik_id`),
  CONSTRAINT `fk_narudzbine_korisnik` FOREIGN KEY (`korisnik_id`)
    REFERENCES `korisnici` (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

Ovde vidite sve: kolone, tipove, indekse, veze, engine, kodiranje.

### Korak 4: veze između tabela

Ako postoje strani ključevi, veze su eksplicitne:

```sql
SELECT
  kcu.table_name        AS tabela,
  kcu.column_name       AS kolona,
  kcu.referenced_table_name  AS ka_tabeli,
  kcu.referenced_column_name AS ka_koloni
FROM information_schema.key_column_usage kcu
WHERE kcu.table_schema = 'shop'
  AND kcu.referenced_table_name IS NOT NULL
ORDER BY kcu.table_name;
```

**Vrlo često strani ključevi ne postoje.** Mnogi ORM-ovi i mnogi timovi ih namerno izbegavaju iz razloga performansi ili fleksibilnosti.

Tada veze zaključujete iz imenovanja:

```sql
SELECT table_name, column_name, data_type
FROM information_schema.columns
WHERE table_schema = 'shop'
  AND (column_name LIKE '%\_id' OR column_name = 'id')
ORDER BY table_name, ordinal_position;
```

Kolona `korisnik_id` u tabeli `narudzbine` gotovo sigurno pokazuje na `korisnici.id`. Konvencija se razlikuje po framework-u (`user_id`, `userId`, `fk_user`), ali obrazac je uvek prepoznatljiv.

Provera pretpostavke:

```sql
SELECT COUNT(*) FROM narudzbine n
LEFT JOIN korisnici k ON n.korisnik_id = k.id
WHERE k.id IS NULL;
```

Ako je rezultat 0, pretpostavka je verovatno tačna.

### Korak 5: pogledi, procedure, trigeri, eventi

```sql
SELECT table_name FROM information_schema.views WHERE table_schema = 'shop';

SELECT routine_name, routine_type
FROM information_schema.routines WHERE routine_schema = 'shop';

SELECT trigger_name, event_object_table, action_timing, event_manipulation
FROM information_schema.triggers WHERE trigger_schema = 'shop';

SELECT event_name, status, interval_value, interval_field
FROM information_schema.events WHERE event_schema = 'shop';
```

**Trigeri su najvažniji od ovih.** Oni tiho menjaju podatke pri svakom `INSERT`-u ili `UPDATE`-u. Ako ne znate da postoje, ne razumete šta se dešava sa podacima — a to postaje bitno pri migraciji i pri `pt-online-schema-change` (Modul 9).

Eventi su drugi po važnosti — to su zakazani poslovi unutar baze, koje ćete inače tražiti u `crontab`-u i nećete ih naći.

### Korak 6: kodiranja i kolacije

```sql
SELECT table_name, table_collation
FROM information_schema.tables
WHERE table_schema = 'shop' AND table_type = 'BASE TABLE'
GROUP BY table_name, table_collation;

SELECT DISTINCT character_set_name, collation_name
FROM information_schema.columns
WHERE table_schema = 'shop' AND character_set_name IS NOT NULL;
```

Mešavina kodiranja ili kolacija je izvor problema pri `JOIN`-u i pri migraciji (Modul 9). Vredi znati unapred.

### Korak 7: uzorak podataka

```sql
SELECT * FROM shop.narudzbine ORDER BY id DESC LIMIT 3\G
```

Pet redova iz svake ključne tabele vam kažu više o nameni nego imena kolona.

**Oprez:** ako baza sadrži lične podatke, budite svesni šta gledate i gde to završava. Ne kopirajte uzorke u tikete i chat.

### Skript za brzu orijentaciju

```bash
#!/bin/bash
# /usr/local/bin/mysql-orijentacija.sh
DB="${1:?Upotreba: $0 <baza>}"

echo "=== VELIČINA I BROJ TABELA ==="
mysql -t -e "SELECT COUNT(*) AS tabela,
  ROUND(SUM(data_length+index_length)/1024/1024,1) AS mb
  FROM information_schema.tables
  WHERE table_schema='$DB' AND table_type='BASE TABLE';"

echo "=== NAJVEĆE TABELE ==="
mysql -t -e "SELECT table_name,
  ROUND((data_length+index_length)/1024/1024,1) AS mb, table_rows
  FROM information_schema.tables
  WHERE table_schema='$DB' AND table_type='BASE TABLE'
  ORDER BY data_length+index_length DESC LIMIT 15;"

echo "=== NAJKORIŠĆENIJE TABELE ==="
mysql -t -e "SELECT object_name, count_read, count_write
  FROM performance_schema.table_io_waits_summary_by_table
  WHERE object_schema='$DB' ORDER BY count_star DESC LIMIT 15;"

echo "=== STRANI KLJUČEVI ==="
mysql -t -e "SELECT table_name, column_name,
  referenced_table_name, referenced_column_name
  FROM information_schema.key_column_usage
  WHERE table_schema='$DB' AND referenced_table_name IS NOT NULL;"

echo "=== TABELE BEZ PRIMARNOG KLJUČA ==="
mysql -t -e "SELECT t.table_name FROM information_schema.tables t
  LEFT JOIN information_schema.table_constraints c
    ON t.table_schema=c.table_schema AND t.table_name=c.table_name
   AND c.constraint_type='PRIMARY KEY'
  WHERE c.constraint_name IS NULL AND t.table_type='BASE TABLE'
    AND t.table_schema='$DB';"

echo "=== TRIGERI, PROCEDURE, EVENTI ==="
mysql -t -e "SELECT trigger_name, event_object_table FROM information_schema.triggers
  WHERE trigger_schema='$DB';"
mysql -t -e "SELECT routine_name, routine_type FROM information_schema.routines
  WHERE routine_schema='$DB';"
mysql -t -e "SELECT event_name, status FROM information_schema.events
  WHERE event_schema='$DB';"

echo "=== NEJEDINSTVENE KOLACIJE ==="
mysql -t -e "SELECT table_collation, COUNT(*) FROM information_schema.tables
  WHERE table_schema='$DB' AND table_type='BASE TABLE'
  GROUP BY table_collation;"
```

Sačuvajte izlaz kao polaznu dokumentaciju za taj sistem.

---

## 58. Kako razgovarati sa developerima o bazi

### Zašto ovo poglavlje postoji

Najveći deo problema sa bazom u praksi nije tehnički, nego komunikacijski. Vi vidite simptome na serveru, developer vidi ponašanje aplikacije, i dok se ne složite oko rečnika, gubite sate.

Ovaj deo ima dva cilja: da razumete šta vam govore, i da vas oni razumeju kada im nešto prosledite.

### Rečnik — pojmovi koje treba razumeti

**Indeks.** Struktura koja omogućava brzo pronalaženje redova, kao indeks u knjizi. Bez indeksa server čita celu tabelu.
*Za vas:* nedostajući indeks je najčešći uzrok sporog upita i vidite ga kao `type: ALL` u `EXPLAIN`-u i kao veliki `Rows_examined` u slow logu.

**Primarni ključ.** Jedinstveni identifikator reda. U InnoDB-u određuje i fizički raspored podataka.
*Za vas:* tabela bez primarnog ključa ubija replikaciju (Modul 8) i pravi probleme sa `pt-online-schema-change` (Modul 9).

**Pun pregled tabele (full table scan).** Server čita svaki red da bi našao ono što traži.
*Za vas:* nije uvek loše — nad tabelom od 200 redova je normalno. Nad tabelom od dva miliona jeste.

**Kardinalnost.** Koliko različitih vrednosti kolona ima. Indeks na koloni sa dve moguće vrednosti (`da`/`ne`) je slabo koristan; indeks na koloni sa milion različitih vrednosti je vrlo koristan.

**Pokrivajući indeks (covering index).** Indeks koji sadrži sve kolone potrebne upitu, pa server ne mora ni da dodiruje samu tabelu. U `EXPLAIN`-u se vidi kao `Using index`.

**Transakcija.** Grupa naredbi koje se izvršavaju sve ili nijedna.
*Za vas:* duga otvorena transakcija drži zaključavanja i sprečava čišćenje undo prostora (Moduli 2, 6, 10).

**Deadlock.** Dve transakcije čekaju jedna drugu. MySQL detektuje i poništava jednu.
*Za vas:* povremeni deadlock je normalan i aplikacija treba da ga rešava ponavljanjem. Mnogo njih je aplikacioni problem — dokaz je u `SHOW ENGINE INNODB STATUS`.

**Zaključavanje reda, tabele i metapodataka.** Tri različite stvari.
*Za vas:* metadata lock je onaj koji zaustavlja celu aplikaciju kada neko pokrene `ALTER TABLE` (Moduli 6, 9).

**N+1.** Aplikacija umesto jednog upita sa `JOIN`-om izvršava jedan upit za listu i još po jedan za svaki rezultat.
*Za vas:* mnogo brzih upita, visok `Questions`, niska `Threads_running`, spora aplikacija. Klasičan nalaz iz Modula 7.

**Connection pool.** Skup unapred otvorenih konekcija koje aplikacija ponovo koristi.
*Za vas:* nepostojanje poola vidite kroz `Threads_created` blizu `Connections` (Modul 7).

**ORM.** Sloj koji preslikava objekte u tabele (Doctrine, Eloquent, Hibernate, ActiveRecord, SQLAlchemy).
*Za vas:* ORM piše upite umesto developera, pa ponekad ni developer ne zna tačno šta se šalje bazi. Kada ga pitate "koji upit ovo pravi", sasvim je legitimno da mu pokažete slow log.

**Migracija (šeme).** Skript koji menja strukturu baze pri deploy-u.
*Za vas:* razlog zbog kog postoji zaseban nalog sa `ALTER` i `DROP` privilegijama (Modul 3), i razlog zbog kog aplikacija ponekad stane pri deploy-u (Modul 9).

**DDL i DML.** DDL menja strukturu (`CREATE`, `ALTER`, `DROP`), DML menja podatke (`INSERT`, `UPDATE`, `DELETE`).
*Za vas:* DDL je opasan po dostupnost, DML po sadržaj.

**Particionisanje, replikacija, šardovanje.** Tri različite stvari koje se stalno mešaju:
- *particionisanje* — jedna tabela podeljena u delove **na istom serveru**,
- *replikacija* — kopija cele baze **na drugom serveru**,
- *šardovanje* — podaci podeljeni **između više servera** po nekom ključu.

**OLTP i OLAP.** Prvo su kratke transakcije aplikacije, drugo su analitički upiti nad mnogo podataka.
*Za vas:* razlog zbog kog izveštaji idu na repliku (Modul 8).

**Zaostajanje replike i "pročitaj nakon upisa".** Replika kasni; aplikacija koja upiše red na primarni pa ga odmah pročita sa replike neće ga naći.
*Za vas:* ovo je aplikaciona odluka, ne vaš propust — objasnite je unapred (Modul 8).

**RPO i RTO.** Koliko podataka smete izgubiti i koliko dugo smete biti nedostupni.
*Za vas:* dva broja koja tražite od vlasnika sistema, a merite sami (Modul 5).

**Kodiranje i kolacija.** Kodiranje određuje koji znakovi mogu da se čuvaju (`utf8mb4`), kolacija određuje kako se porede i sortiraju (`utf8mb4_0900_ai_ci`).
*Za vas:* mešanje kolacija pravi greške pri `JOIN`-u i probleme pri nadogradnji (Modul 9).

### Prevod: šta kažu i šta to znači za vas

| Kažu | Znači za vas | Šta proverite |
|---|---|---|
| "Baza je spora" | možda jeste, možda nije | `Threads_running`, `sys.statement_analysis` (Modul 7) |
| "Treba nam više konekcija" | verovatno curenje ili pool fali | `Threads_running` vs `Threads_connected` (Modul 10) |
| "Treba nam pun pristup bazi" | ne treba | pet pitanja iz Modula 3 |
| "Deploy je oborio sajt" | migracija i metadata lock | `processlist`, `ALTER` u toku (Modul 9) |
| "Podaci nedostaju" | replika, keš ili stvarno brisanje | odakle čitaju, kada je nestalo (Moduli 5, 8) |
| "Timeout na bazi" | spor upit ili čekanje na lock | slow log, `innodb_trx` (Modul 6) |
| "Radilo je juče" | nešto se promenilo | deploy, novi upit, rast tabele |

### Kako formulisati nalaz

Loše:

> "Vaš upit je loš i obara mi server."

Ovo je stav. Ne može se proveriti, a osoba sa druge strane odmah prelazi u odbranu.

Dobro:

> Upit
> ```sql
> SELECT COUNT(*) FROM narudzbine WHERE status = 'na_cekanju';
> ```
> izvršava se 42.000 puta na sat i troši 61% ukupnog vremena baze.
>
> Po pozivu pregleda 2,45 miliona redova, a vraća 1.
> `EXPLAIN` pokazuje `type: ALL`, `key: NULL`.
> Tabela ima 2,45M redova i nema indeks na koloni `status`.
>
> Prilog: `pt-query-digest` izveštaj 00:00–01:00.

Ovo su merenja. Nema o čemu da se raspravlja, i rešenje je očigledno obema stranama.

### Šta tražiti kada dođe zahtev

Kada developer traži nalog za bazu (Modul 3):

1. Koje baze aplikacija dodiruje?
2. Menja li strukturu u toku rada ili samo pri deploy-u?
3. Sa kojih mašina se povezuje?
4. Koristi li stored procedure?
5. Koristi li `LOAD DATA INFILE`?

Kada developer traži izmenu šeme (Modul 9):

1. Koja tabela i koliko ima redova?
2. Koja operacija tačno — koji `ALTER`?
3. Sme li tabela da bude nedostupna, i koliko?
4. Ima li rollback ako se pokaže da nešto ne valja?

Kada developer prijavljuje sporost (Modul 7):

1. Od kada?
2. Šta je poslednje deploy-ovano i kada?
3. Koliko upita aplikacija šalje po zahtevu?
4. Šta aplikacija meri kao "vreme baze"?

Ova pitanja nisu prepreka — ona su najbrži put do tačnog odgovora, i obično traju pet minuta.

### Šta vi treba da saopštite bez pitanja

Postoje stvari koje developer ne može znati, a treba da zna:

- **Replika zaostaje** i koliko, ako čitaju sa nje.
- **Backup se pravi u 2h** i tada je server sporiji.
- **Prozor za point-in-time recovery** je toliko i toliko dana — određuje šta je moguće vratiti.
- **Izmereni RTO** — koliko traje restore. Ovo naročito, jer ljudi obično pretpostavljaju da je brže.
- **Tabele bez primarnog ključa** i zašto su problem.
- **Rast tabela** — kada će disk postati problem, mesecima unapred.

Redovan kratak izveštaj o ovome sprečava veći deo hitnih situacija.

---

## Kraj

Time je kurs završen.

Vratite se na tabelu podele odgovornosti iz Modula 1 kad god se zapitate da li nešto treba da naučite. Ona i dalje važi: vaš posao je server, podaci i njihova sigurnost. SQL je alat koji koristite, a ne oblast koju morate osvojiti.

I jedna stvar koja se ponavljala kroz sve module, pa neka bude i poslednja:

> **Netestiran backup nije backup.**

Ako od celog kursa zapamtite samo tu rečenicu i postupite po njoj, bio je vredan vremena.
