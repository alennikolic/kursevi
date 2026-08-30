# Modul 2: Servis, konfiguracija, fajlovi

*MySQL za Linux administratore*

---

## Uvod: modul u kome MySQL prestaje da bude crna kutija

U prvom modulu smo instalirali server i povezali se na njega. Sada ga rastavljamo.

Ovaj modul je najviše "sysadmin" od svih u kursu, jer se bavi isključivo stvarima koje vidite sa strane operativnog sistema: procesom, unit fajlom, konfiguracionim fajlovima, fajlovima na disku i mehanizmom koji kontroliše čemu proces sme da pristupi.

Cilj je konkretan. Nakon ovog modula treba da umete da odgovorite na sledeća pitanja bez guglanja:

- Zašto `systemctl restart mysql` visi već sedam minuta i da li da ga prekinem?
- Odakle je došla vrednost `max_connections = 500` kad je nema ni u jednom fajlu u `/etc/mysql`?
- Koji od ovih fajlova u `/var/lib/mysql` sme da se obriše kad je disk pun?
- Zašto MySQL kaže "Permission denied" na direktorijum koji je u vlasništvu korisnika `mysql` i ima prava 750?

Sva četiri pitanja su realna i sva četiri se javljaju u prvoj godini rada sa MySQL-om.

---

## 6. MySQL kao systemd servis

### Unit fajl

Prvo pravilo: **ne čitajte unit fajl iz `/lib/systemd/system/`, čitajte ga kroz systemd.**

```bash
systemctl cat mysql
```

Ova komanda prikazuje osnovni unit fajl **i sve override fajlove** koji na njega utiču, redom kojim se primenjuju. Ako neko pre vas menja konfiguraciju servisa, videćete to ovde, a ne u originalnom fajlu.

Na Ubuntuu sa paketom `mysql-server-8.0` dobijate nešto ovakvo:

```ini
[Unit]
Description=MySQL Community Server
After=network.target

[Install]
WantedBy=multi-user.target

[Service]
Type=notify
User=mysql
Group=mysql
TimeoutStartSec=infinity
ExecStartPre=/usr/share/mysql/mysql-systemd-start pre
ExecStart=/usr/sbin/mysqld
ExecStartPost=/usr/share/mysql/mysql-systemd-start post
TimeoutSec=infinity
Restart=on-failure
RestartPreventExitStatus=1
RuntimeDirectory=mysqld
RuntimeDirectoryMode=755
LimitNOFILE=10000
```

Detalji se razlikuju između verzija i između Ubuntu i Oracle paketa, pa uvek proverite na svom sistemu. Ali nekoliko linija zaslužuje objašnjenje, jer direktno objašnjavaju ponašanje koje ćete viđati.

**`Type=notify`**

Ovo je najvažnija linija u fajlu. Znači da `mysqld` sam javlja systemd-u kada je spreman — putem `sd_notify` mehanizma. Sve dok server ne kaže "spreman sam", systemd smatra da se servis još uvek startuje.

Posledica: **`systemctl start mysql` ne vraća prompt dok server zaista ne počne da prihvata konekcije.** To je dobro (nema lažno pozitivnog "startovano"), ali znači da će komanda naizgled "visiti" kad god start traje dugo.

**`TimeoutStartSec=infinity` i `TimeoutSec=infinity`**

Nema tajmauta. Nikakvog. Ovo je namerna odluka Ubuntu paketa i objašnjava zašto restart ume da stoji unedogled umesto da padne sa greškom.

Razlog je opravdan: ako se MySQL oporavlja nakon pada, taj proces može trajati minutima ili desetinama minuta na velikoj bazi. Da postoji tajmaut, systemd bi ubio proces usred oporavka — što bi napravilo veću štetu.

Praktična posledica za vas: **kada restart visi, to nije bug, to je posao koji traje.** Nikada ne prekidajte oporavak.

**`Restart=on-failure` i `RestartPreventExitStatus=1`**

Ako `mysqld` padne, systemd ga podiže. Ali ako izađe sa statusom 1 — što je tipično za grešku u konfiguraciji — systemd **neće** pokušavati ponovo. To sprečava beskonačnu petlju restartovanja zbog zareza koji fali u `my.cnf`.

Ako vidite da servis pada i ne pokušava restart, prva sumnja je konfiguracija.

**`LimitNOFILE=10000`**

Ograničenje broja otvorenih fajl deskriptora. Ovo je čest izvor problema na serverima sa mnogo tabela ili mnogo konekcija, jer ograničava koliko fajlova `mysqld` uopšte sme da otvori — bez obzira na to šta piše u `/etc/security/limits.conf`. Za systemd servise `limits.conf` **ne važi**; važi `LimitNOFILE` iz unit fajla. Detaljno u Modulu 7.

**`RuntimeDirectory=mysqld`**

Systemd sam kreira `/run/mysqld` pri svakom startu i briše ga pri gašenju. Zato ne pokušavajte da ručno "popravljate" prava na tom direktorijumu — nestaju pri sledećem restartu.

### Kako se pravilno menja unit fajl

Nikad ne editujte `/lib/systemd/system/mysql.service`. Nadogradnja paketa će ga prepisati.

```bash
sudo systemctl edit mysql
```

Otvara prazan fajl u `/etc/systemd/system/mysql.service.d/override.conf`. Upišete samo ono što menjate:

```ini
[Service]
LimitNOFILE=65535
```

Zatim:

```bash
sudo systemctl daemon-reload
sudo systemctl restart mysql
systemctl show mysql -p LimitNOFILE
```

Poslednja komanda potvrđuje da je vrednost stvarno primenjena. Uvek proverite — pisanje override fajla i njegovo dejstvo nisu ista stvar.

### Šta se dešava tokom starta

Redosled, korak po korak:

1. **`ExecStartPre` skripta** proverava osnovne preduslove: postoji li `datadir`, ima li prava, postoji li konfiguracija.
2. **`mysqld` startuje kao root**, otvara fajlove, veže se na port ako treba, pa odbacuje privilegije i nastavlja kao korisnik `mysql`.
3. **Učitava konfiguraciju** — o tome u sledećem poglavlju.
4. **Inicijalizuje InnoDB**: alocira buffer pool, otvara tablespace fajlove.
5. **Ako je prethodno gašenje bilo nečisto — pokreće crash recovery.** Ovo je faza koja traje.
6. **Učitava data dictionary**, otvara sistemske tabele, učitava privilegije.
7. **Opciono učitava buffer pool iz `ib_buffer_pool`** ako je uključen `innodb_buffer_pool_load_at_startup`.
8. **Počinje da sluša** na socket-u i portu.
9. **Javlja systemd-u da je spreman.** Tek sada `systemctl start` vraća prompt.

Ovaj redosled nije akademski. Kad start visi, treba da znate **u kojoj fazi** visi — a to piše u error logu.

### Šta raditi kad start visi

Ne prekidajte. Umesto toga, u drugom terminalu:

```bash
sudo tail -f /var/log/mysql/error.log
```

Tražite linije o InnoDB oporavku. Videćete napredak u procentima ili brojeve LSN-a. Ako se brojevi menjaju — server radi, sačekajte.

Paralelno možete pratiti da li proces troši CPU i I/O:

```bash
top -p "$(pgrep -x mysqld)"
sudo iotop -o
```

Ako proces troši I/O i log napreduje — sve je u redu, samo traje.

Ako u logu **nema pomaka nekoliko minuta** i proces ne troši ništa, tada tražite dalje: greška u pravima, zauzet port, pun disk. Postupak dijagnostike u pet koraka radimo u Modulu 10.

Kad servis već radi, korisno je i:

```bash
journalctl -u mysql -f
journalctl -u mysql --since "10 min ago"
```

Napomena: journal sadrži poruke systemd-a i ono što `mysqld` pošalje na stderr. Ako je `log_error` usmeren na fajl, glavnina poruka **neće** biti u journalu. Zato uvek imate dva izvora, ne jedan.

### Gašenje i zašto ume da traje

`systemctl stop mysql` šalje SIGTERM. `mysqld` tada:

1. prestaje da prima nove konekcije,
2. čeka da se aktivne transakcije završe ili ih poništava,
3. **prazni prljave stranice iz buffer pool-a na disk**,
4. opciono snima sadržaj buffer pool-a u `ib_buffer_pool`,
5. uredno zatvara tablespace fajlove i izlazi.

Korak 3 je razlog trajanja. Ako imate buffer pool od 64 GB pun izmenjenih podataka, to je 64 GB koje treba upisati na disk pre nego što proces sme da izađe.

Ponašanje kontroliše:

```sql
SELECT @@innodb_fast_shutdown;
```

- **0** — potpuno "čisto" gašenje: prazni sve i završava merge operacije. Najsporije. Obavezno pre nadogradnje na novu major verziju (Modul 9).
- **1** — podrazumevano. Prazni prljave stranice, ali ne radi sve pomoćne operacije.
- **2** — najbrže, ali je efekat isti kao pad servera: sledeći start radi crash recovery. Koristi se u hitnim situacijama.

**Nikad `kill -9` na `mysqld`.** To je isto što i iznenadni nestanak struje: podaci ostaju konzistentni zahvaljujući redo logu, ali plaćate dugim oporavkom pri sledećem startu, a u retkim slučajevima i oštećenjem.

Ako `stop` traje predugo a morate da ubrzate, ispravan način je:

```sql
SET GLOBAL innodb_fast_shutdown = 2;
```

pa tek onda `systemctl stop mysql`. Svesno birate brz izlaz i dug start, umesto da ubijate proces.

### Imenovanje servisa po distribucijama

Sitnica koja gubi vreme kad prelazite između sistema:

| Sistem | Naziv servisa |
|---|---|
| Ubuntu / Debian, MySQL | `mysql` |
| RHEL / Rocky / Alma, MySQL | `mysqld` |
| MariaDB (svuda) | `mariadb`, sa `mysql`/`mysqld` kao aliasom |
| Percona Server na Ubuntu | `mysql` |

Provera na nepoznatom sistemu:

```bash
systemctl list-units --type=service | grep -Ei 'mysql|maria'
```

---

## 7. Anatomija konfiguracije

### Redosled učitavanja

MySQL ne čita jedan konfiguracioni fajl. Čita niz njih, redom, i **kasnije vrednosti pregaze ranije**.

Nemojte pamtiti listu — pitajte binarni fajl:

```bash
mysqld --verbose --help | grep -A 1 "Default options"
```

```
Default options are read from the following files in the given order:
/etc/my.cnf /etc/mysql/my.cnf ~/.my.cnf
```

Ista provera za klijent:

```bash
mysql --help | grep -A 1 "Default options"
```

Uočite da klijent i server čitaju **različite skupove fajlova**. To objašnjava zašto podešavanje koje ste stavili "u my.cnf" ponekad utiče samo na klijent.

### Ubuntu struktura

```bash
ls -la /etc/mysql/
```

```
my.cnf -> /etc/alternatives/my.cnf
mysql.cnf
conf.d/
mysql.conf.d/
debian.cnf
debian-start
```

Lanac je: `/etc/mysql/my.cnf` → `/etc/alternatives/my.cnf` → `/etc/mysql/mysql.cnf`.

Zašto `alternatives`? Zato što isti mehanizam koriste i MySQL i MariaDB paketi, pa se ovako izbegava sukob oko istog putanjskog imena.

Sadržaj `mysql.cnf` je gotovo prazan:

```ini
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/
```

Dva pravila o `!includedir`:

1. Čitaju se **samo fajlovi sa ekstenzijom `.cnf`**. Fajl `mysqld.cnf.backup` biće ignorisan — što je zgodno, jer možete bezbedno ostaviti kopiju.
2. Čitaju se **abecednim redom**. Zato se konvencija numeričkog prefiksa isplati: `99-custom.cnf` se učitava posle `50-server.cnf` i pobeđuje.

Šta ide gde:

- **`conf.d/`** — po paketskoj konvenciji, podešavanja klijentskih alata (`mysql.cnf`, `mysqldump.cnf`).
- **`mysql.conf.d/`** — podešavanja servera. Ovde je `mysqld.cnf`, glavni paketski fajl.

### Zlatno pravilo izmena

**Ne menjajte `/etc/mysql/mysql.conf.d/mysqld.cnf`.**

To je fajl koji pripada paketu. Pri nadogradnji `apt` će vas pitati da li da ga prepiše, i u tom trenutku birate između gubitka svojih izmena i zaostajanja za paketskim promenama. Nijedno nije dobro.

Umesto toga:

```bash
sudo nano /etc/mysql/mysql.conf.d/99-custom.cnf
```

```ini
[mysqld]
# Postavio: <vaše ime>, <datum>
# Razlog: server ima 16 GB RAM-a, buffer pool na ~60%
innodb_buffer_pool_size = 10G

# Razlog: aplikacija otvara do 300 paralelnih konekcija
max_connections = 400
```

Komentarišite **zašto**, ne šta. `innodb_buffer_pool_size = 10G` je očigledno; činjenica da ste do te brojke došli iz 16 GB RAM-a nije, a biće presudna kad neko za godinu dana doda RAM.

### Grupe (sekcije)

Konfiguracioni fajl je podeljen u grupe u uglastim zagradama. Program čita samo grupe koje se na njega odnose.

| Grupa | Ko je čita |
|---|---|
| `[mysqld]` | server |
| `[mysqld_safe]` | wrapper skripta (nebitno pod systemd-om) |
| `[client]` | **svi** klijentski alati |
| `[mysql]` | samo `mysql` klijent |
| `[mysqldump]` | samo `mysqldump` |
| `[mysqladmin]` | samo `mysqladmin` |
| `[client-server]` | i klijenti i server |

Najčešća greška početnika: stavljanje serverskog parametra u `[client]` grupu, ili obrnuto. Server tada tiho ignoriše podešavanje, jer ne čita tu grupu — ili, u nekim verzijama, odbija da startuje zbog nepoznate opcije.

Postoje i grupe vezane za verziju, npr. `[mysqld-8.0]`, koje čita samo ta serija. Korisno kad održavate isti fajl na serverima sa različitim verzijama.

Sitnica koja zbunjuje: **crtica i donja crta su ravnopravne** u imenima opcija. `innodb_buffer_pool_size` i `innodb-buffer-pool-size` su isto. Sistemske promenljive u SQL-u, međutim, uvek koriste donju crtu.

### Provera pre restarta — obavezno

Loša konfiguracija znači da server neće startovati. Ako to otkrijete tek posle `systemctl restart` na produkciji, imate ispad umesto izmene.

```bash
sudo mysqld --validate-config
```

Ako ništa ne ispiše — konfiguracija je sintaksno ispravna. Ako nešto nije u redu, dobijate konkretnu poruku i broj linije.

Ovo **ne** proverava da li su vrednosti razumne. `innodb_buffer_pool_size = 900G` na serveru sa 8 GB RAM-a proći će validaciju i onda oboriti start. Validacija hvata sintaksu, ne zdrav razum.

Šta će server zaista pročitati:

```bash
my_print_defaults mysqld
```

Ova komanda ispisuje sve opcije iz svih fajlova, spojene i po redosledu. Ako se neka opcija pojavljuje dvaput, **poslednja pobeđuje** — a ovako to vidite na jednom mestu.

### Dinamičke i statičke promenljive

Neke promenljive se menjaju bez restarta, neke ne.

```sql
-- pregled
SHOW VARIABLES LIKE 'max_connections';
SELECT @@global.max_connections;

-- promena do sledećeg restarta
SET GLOBAL max_connections = 500;
```

Ako parametar nije dinamičan, dobijate:

```
ERROR 1238 (HY000): Variable 'innodb_buffer_pool_chunk_size' is a read only variable
```

Tada je jedini put kroz konfiguracioni fajl i restart.

### `SET PERSIST` — zamka koju morate poznavati

Od MySQL 8.0 postoji mogućnost trajne izmene iz SQL-a:

```sql
SET PERSIST max_connections = 500;
```

Ovo menja vrednost odmah **i** zapisuje je u fajl:

```bash
sudo cat /var/lib/mysql/mysqld-auto.cnf
```

```json
{ "Version": 2, "mysql_server": { "max_connections": { "Value": "500", ... } } }
```

Zapamtite ovo dobro, jer je izvor pravih misterija: **`mysqld-auto.cnf` se učitava posle svih fajlova iz `/etc/mysql` i pregazi ih.**

Scenario koji ćete sresti: postavite `max_connections = 300` u `99-custom.cnf`, restartujete, a server i dalje prijavljuje 500. Fajl je ispravan, sintaksa je ispravna, vi ste ispravni — a neko je pre šest meseci uradio `SET PERSIST` i zaboravio.

Kako da uvek znate odakle vrednost dolazi:

```sql
SELECT variable_name, variable_source, variable_path
FROM performance_schema.variables_info
WHERE variable_name IN ('max_connections', 'innodb_buffer_pool_size');
```

```
+-------------------------+-----------------+----------------------------------------------+
| variable_name           | variable_source | variable_path                                |
+-------------------------+-----------------+----------------------------------------------+
| max_connections         | PERSISTED       | /var/lib/mysql/mysqld-auto.cnf               |
| innodb_buffer_pool_size | EXPLICIT        | /etc/mysql/mysql.conf.d/99-custom.cnf        |
+-------------------------+-----------------+----------------------------------------------+
```

**Ovo je jedna od najkorisnijih komandi u celom kursu.** Kad god vas neko pita "zašto je ovaj parametar takav", ovo je odgovor u jednom potezu.

Vrednosti kolone `variable_source`: `COMPILED` (podrazumevano ugrađeno), `GLOBAL` / `SERVER` / `EXPLICIT` (iz fajla), `PERSISTED` (iz `mysqld-auto.cnf`), `COMMAND_LINE`, `DYNAMIC` (promenjeno sa `SET GLOBAL` u toku rada).

Uklanjanje persistovane vrednosti:

```sql
RESET PERSIST max_connections;
-- ili sve odjednom:
RESET PERSIST;
```

Preporuka za disciplinu u timu: **koristite `SET PERSIST` samo za privremene intervencije, a trajna konfiguracija neka bude u fajlovima** koji su pod verzionisanjem i pod Ansible kontrolom (Modul 11). Fajl vidi svako ko pogleda repozitorijum; `mysqld-auto.cnf` ne vidi niko dok ne krene da traži.

---

## 8. Gde MySQL drži podatke

```bash
mysql -e "SELECT @@datadir;"
sudo ls -la /var/lib/mysql/
```

Idemo kroz ono što ćete videti. Neću objašnjavati unutrašnju strukturu svakog fajla — objasniću **čemu služi i šta se dešava ako ga dirate**, jer je to ono što vam treba.

### Sistemski tablespace i pomoćni fajlovi

**`ibdata1`**

Sistemski tablespace. U MySQL 8 sadrži znatno manje nego ranije (change buffer, deo metapodataka), ali i dalje je obavezan.

Osobina koju treba znati: **nikad se ne smanjuje.** Ako je narastao, ostaje velik i nakon brisanja podataka. Jedini način da se smanji je potpuna rekonstrukcija instance kroz logički dump i restore.

Ne brisati. Nikad.

**`#innodb_redo/`** (MySQL 8.0.30 i noviji) ili **`ib_logfile0`, `ib_logfile1`** (starije verzije)

Redo log. Ovde se upisuje svaka izmena **pre** nego što stigne u tabele. To je mehanizam koji omogućava oporavak nakon pada — server pri startu pročita redo log i primeni sve što nije stiglo na svoje mesto.

Brisanje redo loga sa nečisto ugašenim serverom znači **gubitak podataka**. Ovo je najskuplja greška koju možete napraviti u `/var/lib/mysql`.

**`undo_001`, `undo_002`**

Undo tablespace-ovi. Čuvaju stare verzije redova, potrebne za rollback i za konzistentno čitanje u transakcijama (MVCC).

Umeju neprijatno da narastu ako neka aplikacija drži otvorenu transakciju danima — što je čest uzrok "disk se puni bez razloga". Provera:

```sql
SELECT * FROM information_schema.innodb_trx ORDER BY trx_started LIMIT 5;
```

**`#ib_16384_0.dblwr`, `#ib_16384_1.dblwr`**

Doublewrite bafer (od 8.0.20 u zasebnim fajlovima). Zaštita od delimično upisanih stranica pri padu sistema. Ne dirati.

**`ibtmp1`**

Privremeni tablespace za privremene tabele. Raste tokom rada, a **prazni se pri restartu**. Ako vam ovaj fajl pojede disk, uzrok je neki upit koji pravi ogromne privremene tabele — problem je u aplikaciji, ne u konfiguraciji.

**`mysql.ibd`**

Data dictionary — metapodaci o svim bazama, tabelama, kolonama, korisnicima. U MySQL 8 je to transakciona tabela unutar InnoDB-a, zbog čega `.frm` fajlova više nema.

**`ib_buffer_pool`**

Snimljena lista stranica koje su bile u buffer pool-u pri gašenju. Pri startu se koristi da se cache "zagreje" unapred. Mali fajl, potpuno bezbedan za brisanje — jedina posledica je sporiji rad prvih nekoliko minuta nakon starta.

### Podaci po bazama

```bash
sudo ls -la /var/lib/mysql/nekabaza/
```

```
korisnici.ibd
narudzbine.ibd
proizvodi.ibd
```

Jedan direktorijum po bazi, jedan `.ibd` fajl po tabeli. To je posledica `innodb_file_per_table = ON`, što je podrazumevano već dugo.

Zašto vam je to bitno: **prostor se oslobađa po tabeli.** Kad se obriše tabela, njen `.ibd` fajl nestaje i disk se stvarno oslobodi. Da su svi podaci u `ibdata1`, brisanje tabele ne bi oslobodilo ni bajt.

Zamka koju treba znati odmah: `.ibd` fajlovi **nisu prenosivi običnim kopiranjem.** Ne možete iskopirati `korisnici.ibd` na drugi server i očekivati da radi — postoji procedura sa `FLUSH TABLES ... FOR EXPORT` i `.cfg` fajlovima, i ona ima ograničenja. Za prenos podataka koristite backup alate iz Modula 5.

### Binarni logovi

**`binlog.000001`, `binlog.000002`, ..., `binlog.index`**

Ako je binarno logovanje uključeno (na MySQL 8 jeste podrazumevano), ovde je zapis svake izmene podataka. Služi za dve stvari: **replikaciju** (Modul 8) i **point-in-time recovery** (Modul 5).

Ovo je fajl grupa koja najčešće puni disk. Kontrola:

```sql
SELECT @@log_bin, @@binlog_expire_logs_seconds, @@max_binlog_size;
SHOW BINARY LOGS;
```

`binlog_expire_logs_seconds` podrazumevano iznosi 2592000 sekundi — trideset dana. Na serveru sa mnogo upisa, trideset dana binloga je lako stotine gigabajta.

Ručno uklanjanje starih logova, **ispravan način**:

```sql
PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 3 DAY);
```

**Nikad `rm binlog.000042`.** Server vodi evidenciju u `binlog.index` i brisanje fajla iza njegovih leđa pravi nekonzistentno stanje. Ako imate replike, brisanje binloga koji replika još nije pročitala **prekida replikaciju** i zahteva ponovno postavljanje replike od nule.

Podešavanje kraćeg roka, uz svest o posledicama po PITR:

```ini
[mysqld]
binlog_expire_logs_seconds = 604800   # 7 dana
max_binlog_size = 256M
```

Nemojte ovo skraćivati bez razmišljanja. Rok čuvanja binloga određuje **koliko unazad možete da radite point-in-time recovery.** Odnos backupa i binloga razrađujemo u Modulu 5.

### `auto.cnf` — mali fajl, veliki problem

```bash
sudo cat /var/lib/mysql/auto.cnf
```

```ini
[auto]
server-uuid=8f3a1b2c-...
```

Jedinstveni identifikator instance servera, generisan pri prvom startu.

Zašto je ovo važno: **ako klonirate virtuelnu mašinu ili iskopirate `datadir` na drugi server, dobijate dva servera sa istim UUID-om.** Replikacija tada odbija da radi, uz poruku koja na prvi pogled nema veze sa uzrokom.

Postupak kod kloniranja: zaustavite MySQL na klonu, obrišite `auto.cnf`, startujte. Server će generisati nov UUID.

Ovo je jedan od retkih fajlova u `datadir`-u koji smete namerno da obrišete — i to isključivo dok je server ugašen.

### Ostali fajlovi

**`*.pem`** — `ca.pem`, `server-cert.pem`, `server-key.pem` i ostali. MySQL 8 pri prvoj inicijalizaciji sam generiše self-signed sertifikate za TLS. Rade, ali ih klijent ne može verifikovati. Prava TLS konfiguracija je u Modulu 4.

**`mysqld-auto.cnf`** — persistovane promenljive, objašnjeno gore.

**`performance_schema`, `sys`** — vidljive kao baze u `SHOW DATABASES`, ali `performance_schema` živi u memoriji, a `sys` je skup pogleda. Nemaju značajan otisak na disku.

### Merenje veličine

Šta pokazuje operativni sistem:

```bash
sudo du -sh /var/lib/mysql
sudo du -sh /var/lib/mysql/* | sort -rh | head -20
```

Šta pokazuje MySQL:

```sql
SELECT table_schema AS baza,
       ROUND(SUM(data_length + index_length) / 1024 / 1024, 1) AS mb
FROM information_schema.tables
GROUP BY table_schema
ORDER BY mb DESC;
```

Najveće tabele:

```sql
SELECT table_schema, table_name,
       ROUND((data_length + index_length) / 1024 / 1024, 1) AS mb,
       ROUND(data_free / 1024 / 1024, 1) AS slobodno_mb
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql','information_schema','performance_schema','sys')
ORDER BY (data_length + index_length) DESC
LIMIT 15;
```

Ova dva broja **neće** biti isti i to je normalno. `du` uključuje binlogove, redo log, undo i fragmentaciju; `information_schema` daje procenu logičke veličine podataka i indeksa. Kolona `data_free` pokazuje fragmentaciju — prostor koji je zauzet na disku, a nije u upotrebi. O sređivanju fragmentacije u Modulu 9.

### Šta smete da obrišete kad je disk pun

Referenca za tri ujutru. Pun spisak postupaka je u Modulu 10, ali evo kratke verzije:

| Fajl | Smete? | Kako |
|---|---|---|
| `binlog.NNNNNN` | ✅ | `PURGE BINARY LOGS`, nikad `rm` |
| `ib_buffer_pool` | ✅ | `rm` je bezbedan |
| stari `slow.log`, `general.log` | ✅ | rotacija ili `FLUSH LOGS` |
| `ibtmp1` | ⚠️ | ne `rm`; nestaje pri restartu |
| `.ibd` fajlovi | ❌ | samo kroz `DROP TABLE` |
| `ibdata1` | ❌ | nikad |
| `#innodb_redo/`, `ib_logfile*` | ❌ | nikad |
| `undo_*` | ❌ | nikad |
| `mysql.ibd` | ❌ | nikad |
| `auto.cnf` | ⚠️ | samo pri kloniranju, sa ugašenim serverom |

---

## 9. Premeštanje `datadir` na drugi disk

Klasičan zadatak: baza je narasla, `/var` je na sistemskom disku, dodali ste novi disk ili LVM volume.

Ovo je operacija koja izgleda jednostavno i onda ne proradi zbog AppArmor-a. Zato je radimo pažljivo i celu.

### Priprema

Pretpostavka: novi uređaj je formatiran i montiran na `/data`.

```bash
lsblk
df -h /data
```

**Prva odluka koja se često promaši:** `datadir` **ne sme** biti sam koren montiranja.

Loše:
```
datadir = /data
```

Dobro:
```
datadir = /data/mysql
```

Razlog: ext4 na korenu fajl sistema kreira `lost+found` direktorijum. MySQL svaki poddirektorijum u `datadir`-u tumači kao bazu podataka, pa ćete dobiti greške ili upozorenja o bazi `lost+found`. Poddirektorijum rešava problem u startu.

Opcije montiranja u `/etc/fstab` koje vrede razmatranja:

```
UUID=... /data ext4 defaults,noatime 0 2
```

`noatime` uklanja nepotrebne upise metapodataka pri svakom čitanju. Za bazu je to čist dobitak.

### Postupak

**Korak 1 — zabeležite polazno stanje**

```bash
mysql -e "SELECT @@datadir;"
sudo du -sh /var/lib/mysql
df -h /data
```

Proverite da na odredištu ima **bar 20% više prostora** nego što je trenutna veličina.

**Korak 2 — zaustavite server**

```bash
sudo systemctl stop mysql
systemctl is-active mysql
```

Sačekajte da druga komanda ispiše `inactive`. Kopiranje živog `datadir`-a daje neupotrebljivu kopiju.

**Korak 3 — kopirajte podatke**

```bash
sudo rsync -avh --progress /var/lib/mysql /data/
```

Obratite pažnju na kose crte. Bez završne kose crte na izvoru, `rsync` kreira `/data/mysql`. To je ono što želimo.

Opcija `-a` čuva vlasništvo, prava i vremenske oznake — to je ovde obavezno.

Za velike baze pokrenite u `screen` ili `tmux` sesiji.

**Korak 4 — proverite vlasništvo**

```bash
sudo ls -la /data/mysql | head
sudo chown -R mysql:mysql /data/mysql
sudo chmod 750 /data/mysql
```

**Korak 5 — sačuvajte original, ne brišite ga**

```bash
sudo mv /var/lib/mysql /var/lib/mysql.old
```

Ovo je vaš put nazad ako nešto pođe naopako. Obrisaćete ga kad sve proradi i kad prođe nekoliko dana.

**Korak 6 — izmenite konfiguraciju**

```bash
sudo nano /etc/mysql/mysql.conf.d/99-custom.cnf
```

```ini
[mysqld]
datadir = /data/mysql
```

**Korak 7 — podesite AppArmor**

Ovaj korak preskaču svi tutorijali koji ne rade na Ubuntuu. Bez njega server neće startovati.

Najjednostavniji način je alias:

```bash
sudo nano /etc/apparmor.d/tunables/alias
```

Dodajte na kraj:

```
alias /var/lib/mysql/ -> /data/mysql/,
```

Obratite pažnju na završne kose crte i zarez na kraju — sintaksa je stroga.

```bash
sudo systemctl restart apparmor
```

Alternativa, ako želite eksplicitna pravila umesto aliasa:

```bash
sudo nano /etc/apparmor.d/local/usr.sbin.mysqld
```

```
/data/mysql/ r,
/data/mysql/** rwk,
```

```bash
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.mysqld
```

Fajl `local/usr.sbin.mysqld` postoji upravo zbog ovoga — glavni profil ga uključuje, a nadogradnja paketa ga ne prepisuje.

**Korak 8 — proverite konfiguraciju i startujte**

```bash
sudo mysqld --validate-config
sudo systemctl start mysql
systemctl status mysql
```

**Korak 9 — potvrdite**

```bash
mysql -e "SELECT @@datadir;"
mysql -e "SHOW DATABASES;"
sudo lsof -p "$(pgrep -x mysqld)" | grep /data/mysql | head
df -h /data /var
```

Naredbom `lsof` potvrđujete da proces stvarno drži otvorene fajlove na novoj lokaciji — jača potvrda od same promenljive.

**Korak 10 — čišćenje, kasnije**

Kad ste sigurni da sve radi i kad je backup sa nove lokacije uspešno testiran:

```bash
sudo rm -rf /var/lib/mysql.old
```

Ne žurite sa ovim korakom. Nekoliko dana zauzetog prostora je jeftinije od jedne loše noći.

### Ako nešto pođe naopako

Povratak je zbog koraka 5 trivijalan:

```bash
sudo systemctl stop mysql
sudo mv /var/lib/mysql.old /var/lib/mysql
# uklonite datadir liniju iz 99-custom.cnf
sudo systemctl start mysql
```

---

## 10. AppArmor i MySQL

### Šta AppArmor radi

AppArmor je MAC sistem (Mandatory Access Control) koji je na Ubuntuu podrazumevano aktivan. Za svaki zaštićen program definiše **spisak fajlova i operacija koje su dozvoljene**. Sve što nije eksplicitno dozvoljeno — zabranjeno je.

Ključna posledica: AppArmor radi **iznad** klasičnih Unix prava. Fajl može biti u vlasništvu korisnika `mysql`, sa pravima 777, a `mysqld` i dalje ne može da mu pristupi ako profil to ne dozvoljava.

### Prepoznavanje AppArmor problema

Simptom je karakterističan i, kad ga jednom prepoznate, štedi vam sate:

> MySQL prijavljuje `Permission denied` (errno 13) na fajl ili direktorijum koji, prema `ls -l`, ima potpuno ispravno vlasništvo i prava.

Ako ste proverili prava tri puta i sve je u redu — pogledajte AppArmor.

### Dijagnostika

```bash
sudo aa-status
sudo aa-status | grep mysql
```

Traženje odbijanja:

```bash
sudo dmesg | grep -i apparmor | tail -20
sudo journalctl -k | grep DENIED | tail -20
sudo grep DENIED /var/log/syslog | tail -20
```

Tipičan zapis:

```
apparmor="DENIED" operation="open" profile="/usr/sbin/mysqld"
name="/data/mysql/ibdata1" pid=1234 comm="mysqld"
requested_mask="rw" denied_mask="rw" fsuid=114 ouid=114
```

Ovo je čitljivije nego što izgleda:

- `profile=` — koji profil je odbio,
- `name=` — koji fajl je tražen,
- `operation=` i `requested_mask=` — šta se tražilo (`r` čitanje, `w` pisanje, `k` zaključavanje),
- `ouid=114` — vlasnik fajla je uid 114, dakle `mysql`. Vlasništvo je ispravno, a pristup je ipak odbijen — to je potvrda da je AppArmor krivac, a ne prava na fajlu.

### Fajlovi profila

| Putanja | Namena |
|---|---|
| `/etc/apparmor.d/usr.sbin.mysqld` | glavni profil, vlasništvo paketa — **ne menjati** |
| `/etc/apparmor.d/local/usr.sbin.mysqld` | vaša pravila — **ovde pišete** |
| `/etc/apparmor.d/tunables/alias` | preslikavanje putanja |

### Postupak rešavanja

**Metoda 1 — alias, kada ste samo premestili putanju**

Opisano u prethodnom poglavlju. Najbrža i najčistija opcija za premeštanje `datadir`-a.

**Metoda 2 — eksplicitna pravila**

```bash
sudo nano /etc/apparmor.d/local/usr.sbin.mysqld
```

```
# Backup direktorijum za XtraBackup
/backup/mysql/ r,
/backup/mysql/** rw,

# Zaseban direktorijum za logove
/var/log/mysql-custom/ r,
/var/log/mysql-custom/** rw,
```

```bash
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.mysqld
sudo systemctl restart mysql
```

Sintaksa ukratko: `**` znači rekurzivno kroz poddirektorijume, `*` samo jedan nivo, a modifikatori su `r` (read), `w` (write), `k` (lock), `m` (mmap), `x` (execute).

**Metoda 3 — complain režim, kada ne znate šta tačno fali**

Ovo je pravi alat kad rešavate nepoznat problem. U complain režimu AppArmor ne blokira ništa, ali **zapisuje sve što bi blokirao.**

```bash
sudo apt install apparmor-utils

sudo aa-complain /usr/sbin/mysqld
sudo systemctl restart mysql

# izvršite operaciju koja je pucala — start, backup, INTO OUTFILE...

sudo grep 'apparmor="ALLOWED"' /var/log/syslog | tail -40
```

Iz tih zapisa vidite tačno koje putanje treba dodati. Napišete pravila, pa nazad u enforce:

```bash
sudo aa-enforce /usr/sbin/mysqld
sudo systemctl restart mysql
```

**Complain režim je alat za dijagnostiku, ne trajno rešenje.** Ako ga ostavite uključenim, isključili ste zaštitu. To se sme na razvojnom serveru; na produkciji je to propust koji će vam neko naći u reviziji.

### Šta AppArmor nije kriv za

Dve stvari koje se stalno mešaju sa AppArmor-om:

**`secure_file_priv`** — ovo je MySQL-ovo sopstveno ograničenje, potpuno nezavisno od AppArmor-a. Određuje iz kog direktorijuma smeju `LOAD DATA INFILE` i `SELECT ... INTO OUTFILE`.

```sql
SELECT @@secure_file_priv;
```

Ako je vrednost `/var/lib/mysql-files/`, izvoz u `/tmp` neće raditi ma šta AppArmor dozvoljavao. Poruka o grešci je drugačija (`--secure-file-priv option`), pa se razlikuje na prvi pogled.

**systemd sandboxing** — direktive poput `ProtectSystem`, `ProtectHome`, `ReadWritePaths` u unit fajlu takođe mogu blokirati pristup. Ubuntu paket ih ne koristi agresivno, ali Oracle paket i ručno pisani unit fajlovi mogu. Provera:

```bash
systemctl show mysql | grep -E 'Protect|ReadWrite|ReadOnly|Private'
```

Tri nezavisna sloja, dakle: Unix prava, AppArmor, systemd sandbox. Kad dijagnostikujete "Permission denied", proverite sva tri.

### SELinux, za svaki slučaj

Ako ikada budete radili na RHEL/Rocky/Alma sistemu, ista uloga pripada SELinux-u, sa drugim alatima: `getenforce`, `ausearch -m avc`, `semanage fcontext`, `restorecon`. Koncept je isti — dodatni sloj iznad Unix prava — samo je implementacija drugačija.

---

## Kontrolna lista na kraju modula

```bash
# 1. Umete da pročitate unit fajl sa svim override-ima
systemctl cat mysql

# 2. Znate da li servis ima tajmaut i koliko fajl deskriptora sme
systemctl show mysql -p TimeoutStartUSec -p LimitNOFILE -p Restart

# 3. Znate koje fajlove server čita i kojim redom
mysqld --verbose --help | grep -A 1 "Default options"
my_print_defaults mysqld

# 4. Imate sopstveni konfiguracioni fajl, odvojen od paketskog
ls -la /etc/mysql/mysql.conf.d/

# 5. Umete da proverite konfiguraciju pre restarta
sudo mysqld --validate-config

# 6. Znate da proverite ima li persistovanih vrednosti
sudo cat /var/lib/mysql/mysqld-auto.cnf 2>/dev/null || echo "nema persistovanih"

# 7. Umete da utvrdite poreklo bilo koje vrednosti
mysql -e "SELECT variable_name, variable_source, variable_path
          FROM performance_schema.variables_info
          WHERE variable_source NOT IN ('COMPILED');"

# 8. Znate koliko zauzimaju podaci, a koliko binlogovi
sudo du -sh /var/lib/mysql
mysql -e "SHOW BINARY LOGS;" | tail -5

# 9. Znate da li je AppArmor aktivan za mysqld
sudo aa-status | grep mysqld

# 10. Znate koliko unazad možete raditi PITR
mysql -e "SELECT @@binlog_expire_logs_seconds / 86400 AS dana;"
```

---

## Vežbe

**Vežba 1 — Override unit fajla**
Podignite `LimitNOFILE` na 65535 preko `systemctl edit`. Potvrdite tri puta: kroz `systemctl show`, kroz `cat /proc/$(pgrep -x mysqld)/limits` i kroz `SELECT @@open_files_limit;`. Objasnite zašto se treća vrednost može razlikovati od prve dve.

**Vežba 2 — Redosled konfiguracije**
Postavite `max_connections = 200` u `50-server.cnf`, `250` u `60-test.cnf` i `300` u `99-custom.cnf`. Restartujte i utvrdite koja vrednost važi. Zatim izvršite `SET PERSIST max_connections = 400`, restartujte i utvrdite ponovo. Objasnite oba rezultata i očistite za sobom sa `RESET PERSIST`.

**Vežba 3 — Namerno slomljena konfiguracija**
Ubacite grešku (npr. `innodb_buffer_pool_size = 900G`) i pokušajte restart. Pronađite tačnu poruku u error logu, popravite i uporedite: šta bi `mysqld --validate-config` uhvatio, a šta ne.

**Vežba 4 — Analiza `datadir`-a**
Napravite tabelu od nekoliko stotina megabajta. Pronađite njen `.ibd` fajl. Uporedite `du` vrednost sa onim što prijavljuje `information_schema.tables`. Zatim obrišite polovinu redova i uporedite ponovo — objasnite šta se desilo sa veličinom fajla i sa kolonom `data_free`.

**Vežba 5 — Premeštanje `datadir`-a**
Dodajte drugi disk na virtuelnoj mašini, montirajte ga i prebacite `datadir` po proceduri iz poglavlja 9. Namerno **preskočite AppArmor korak**, pokušajte start i pronađite odbijanje u logu. Zatim to popravite. Ova vežba je najvrednija u modulu jer ćete zapamtiti simptom.

**Vežba 6 — Complain režim**
Postavite `mysqld` u complain režim. Pokušajte `SELECT ... INTO OUTFILE '/tmp/test.csv'`. Objasnite šta se desilo i zašto AppArmor **nije** bio prepreka. Pronađite pravu prepreku.

**Vežba 7 — Binlogovi i disk**
Utvrdite koliko prostora zauzimaju binlogovi. Izračunajte prosečan dnevni prirast. Odredite koliko dana čuvanja možete priuštiti na trenutnom disku i obrazložite kako ta brojka utiče na vašu mogućnost oporavka.

---

## Šta sledi

U **Modulu 3** prelazimo na korisnike i privilegije — temu u kojoj se prave najveće bezbednosne greške, upravo zato što izgleda jednostavno. Objasnićemo zašto `user@host` model nije isto što i Linux nalozi, koji minimalni set privilegija treba aplikaciji a koji backup nalogu, kako se resetuje zaboravljena root lozinka na dva načina, i zašto `GRANT ALL PRIVILEGES ON *.*` gotovo nikad nije tačan odgovor.
