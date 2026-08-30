# Modul 3: Korisnici, privilegije, pristup

*MySQL za Linux administratore*

---

## Uvod: modul u kome se prave najskuplje greške

Ovo je tema koja izgleda jednostavno. Napravi korisnika, daj mu prava, gotovo. Zato je i najčešći izvor ozbiljnih bezbednosnih propusta na produkcijskim serverima.

Tri stvari koje ćete naći na skoro svakom nasleđenom serveru:

1. Aplikacija se povezuje kao `root`.
2. Postoji nalog sa `%` u host polju i lozinkom od osam znakova.
3. Niko ne zna koji nalozi postoje niti šta smeju, jer to nikada nije zapisano.

Sve tri se rešavaju istim znanjem, a ono staje u jedan modul.

Važna napomena pre početka: **ovo jeste vaš posao.** U tabeli podele odgovornosti iz Modula 1, korisnici i privilegije su na vašoj strani. Developer će vam reći *šta aplikacija treba da radi*; vi odlučujete *koje privilegije su za to minimalno potrebne*. To je razlika između sysadmina i čoveka koji izvršava `GRANT ALL` jer je developer tako tražio.

---

## 11. MySQL korisnici nisu Linux korisnici

### Dva odvojena sveta

Ovo je koncept od koga sve počinje i koji se stalno meša.

Linux ima svoje korisnike u `/etc/passwd`. MySQL ima svoje korisnike u tabeli `mysql.user`. **Između njih ne postoji nikakva veza** — osim jednog izuzetka koji ćemo pomenuti.

Konkretno:

- Linux korisnik `marko` ne može se prijaviti na MySQL osim ako u MySQL-u ne postoji nalog za njega.
- MySQL korisnik `app_shop` ne postoji na Linux nivou, nema home direktorijum, ne može da se uloguje na server, ne pojavljuje se u `id` ili `getent passwd`.
- Brisanje Linux korisnika ne briše MySQL nalog i obrnuto.
- MySQL korisnik `root` i Linux korisnik `root` dele samo ime. To je sve.

Jedini izuzetak je **`auth_socket` plugin** iz Modula 1, koji namerno pravi most: kada se povezujete preko Unix socket-a, plugin pita kernel koji je OS korisnik otvorio konekciju i traži MySQL nalog istog imena. To je jedina tačka dodira i ona postoji samo kod tog plugina.

### Nalog je par: korisnik + host

Druga ključna ideja: **MySQL nalog nije samo korisničko ime.** Nalog je kombinacija imena i hosta sa koga se povezujete.

```
'app_shop'@'localhost'
'app_shop'@'10.0.1.50'
'app_shop'@'10.0.1.%'
'app_shop'@'%'
```

Ovo su **četiri različita naloga.** Mogu imati četiri različite lozinke i četiri različita skupa privilegija. Ne postoji "korisnik app_shop" kao jedinstven entitet — postoje četiri naloga koji dele ime.

Ovo je najveća konceptualna razlika u odnosu na Linux i izvor je ogromnog broja "ali ja sam mu dao privilegije" situacija.

Pregled svih naloga na serveru:

```sql
SELECT user, host, plugin, account_locked, password_expired
FROM mysql.user
ORDER BY user, host;
```

### Šta sme da stoji u host delu

| Oblik | Značenje |
|---|---|
| `localhost` | **specijalno** — konekcija preko Unix socket-a |
| `127.0.0.1` | TCP sa lokalne mašine, IPv4 |
| `::1` | TCP sa lokalne mašine, IPv6 |
| `10.0.1.50` | tačno ta IP adresa |
| `10.0.1.%` | bilo koja adresa iz tog opsega |
| `10.0.0.0/255.255.255.0` | isto, zapisano kao mreža i maska |
| `%` | **bilo odakle** |
| `app01.interno.local` | ime hosta — zahteva DNS |

Podsetnik iz Modula 1: `localhost` nije sinonim za `127.0.0.1`. Za MySQL, `localhost` znači socket, a `127.0.0.1` znači TCP. Nalog `'app'@'localhost'` neće primiti konekciju sa `mysql -h 127.0.0.1`.

Zvezdica `%` u host polju je džoker koji pokriva bilo šta, uključujući ceo internet ako je port otvoren. Njena upotreba je opravdana u tačno jednom slučaju: kada zaista ne možete predvideti izvornu adresu (npr. kontejneri sa dinamičkim IP-jem), i tada uz obavezan firewall i TLS.

### Pravilo za produkciju

**Host polje je vaš prvi sloj zaštite i besplatno je.**

Ako aplikacija radi na `10.0.1.50`, nalog treba da bude `'app_shop'@'10.0.1.50'`. Tada ukradena lozinka ne vredi ništa sa druge mašine. To je jedan red u `CREATE USER` naredbi i ne košta ništa.

Ako imate pool aplikacionih servera u podmreži, koristite `'app_shop'@'10.0.1.%'`. I dalje mnogo bolje od `%`.

### Kako MySQL bira nalog — pravilo koje morate znati

Kada stigne konekcija, MySQL prolazi kroz naloge i traži poklapanje. Problem nastaje kada se poklapa **više njih**.

Server tada bira **najspecifičniji** zapis, po ovim pravilima:

1. Doslovno navedeni hostovi imaju prednost nad onima sa džokerom. `10.0.1.50` pobeđuje `10.0.1.%`, koji pobeđuje `%`.
2. Doslovno korisničko ime pobeđuje prazno (anonimno).
3. Kada je odabran zapis, **traženje se zaustavlja.** Nema spajanja privilegija iz više zapisa.

Ta treća stavka je uzrok klasičnog problema. Scenario:

```sql
-- neko je davno napravio ovo
CREATE USER 'app'@'%' IDENTIFIED BY 'tajna1';
GRANT ALL ON shop.* TO 'app'@'%';

-- kasnije neko dodaje ovo, misleći da "pooštrava"
CREATE USER 'app'@'localhost' IDENTIFIED BY 'tajna2';
-- i zaboravi GRANT
```

Aplikacija koja se povezuje sa `localhost` i lozinkom `tajna1` sada dobija grešku o pogrešnoj lozinci — jer se poklopio specifičniji zapis `'app'@'localhost'`. Čak i ako ispravi lozinku, nema nijednu privilegiju, jer se privilegije sa `'app'@'%'` **ne nasleđuju**.

Dijagnostika je jednostavna kada znate gde da gledate:

```sql
SELECT USER(), CURRENT_USER();
```

```
+-------------------+-----------------+
| USER()            | CURRENT_USER()  |
+-------------------+-----------------+
| app@localhost     | app@localhost   |
+-------------------+-----------------+
```

`USER()` je ono što ste tražili. `CURRENT_USER()` je nalog na koji vas je server stvarno mapirao. **Kad se ta dva razlikuju, našli ste problem.** Uz to:

```sql
SHOW GRANTS FOR CURRENT_USER();
```

Ovo je prva komanda koju izvršavate kada aplikacija prijavljuje "access denied" a vi ste sigurni da ste dali prava.

### DNS i `skip_name_resolve`

Podrazumevano, MySQL pri svakoj konekciji radi **reverzno DNS razrešavanje** izvorne IP adrese, da bi mogao da uporedi sa host poljima koja sadrže imena.

To je loše iz dva razloga. Prvo, ako je DNS spor ili nedostupan, konekcije se otvaraju sporo ili pucaju — sa porukama koje ne ukazuju na DNS. Drugo, oslanjate se na DNS kao bezbednosni mehanizam, što je slaba osnova.

```sql
SELECT @@skip_name_resolve;
```

Preporuka: **uključite ga.**

```ini
[mysqld]
skip_name_resolve = ON
```

Cena: nakon uključivanja, **host polja sa imenima prestaju da rade.** Svi nalozi moraju koristiti IP adrese, opsege ili `localhost`. Zato pre uključivanja proverite:

```sql
SELECT user, host FROM mysql.user
WHERE host NOT REGEXP '^[0-9%_.:]+$' AND host <> 'localhost';
```

Ako ovaj upit vrati redove, ti nalozi će prestati da rade. Prepravite ih pre restarta.

### Autentikacioni plugini

```sql
SELECT user, host, plugin FROM mysql.user;
```

Šta ćete videti:

**`caching_sha2_password`** — podrazumevano u MySQL 8. Bezbedno. Zahteva da lozinka pri prvoj konekciji ide preko TLS-a ili kroz RSA razmenu ključeva; ako klijent ne podržava nijedno, konekcija pada. To je čest uzrok problema sa starijim drajverima.

**`mysql_native_password`** — stari mehanizam. Zastareo od MySQL 8.0.34 i uklonjen u novijim serijama. Ako ga koristite, imate rok trajanja — planirajte prelaz.

**`auth_socket`** — bez lozinke, autentikacija po OS korisniku. Odlično za lokalne administrativne i skriptne naloge.

**`sha256_password`** — prethodnik `caching_sha2_password`-a. Nema razloga za nove naloge.

Promena plugina za postojeći nalog:

```sql
ALTER USER 'app'@'10.0.1.%'
  IDENTIFIED WITH caching_sha2_password BY 'nova-lozinka';
```

Kada je `caching_sha2_password` problem, ispravan redosled rešavanja je: prvo pokušajte da nadogradite drajver u aplikaciji, pa tek ako to nije moguće razmotrite `mysql_native_password` kao privremenu meru sa zapisanim rokom.

### Nalog za lokalne skripte bez lozinke

Praktična primena `auth_socket`-a koja vam se isplati:

```sql
CREATE USER 'backup'@'localhost' IDENTIFIED WITH auth_socket AS 'backup';
```

Deo `AS 'backup'` određuje koji **Linux** korisnik se mapira na ovaj nalog. Ako napravite sistemskog korisnika `backup` i pod njim pokrećete backup skripte, one se povezuju bez ijedne lozinke u ijednom fajlu.

To rešava celu klasu problema sa čuvanjem lozinki u skriptama (Modul 11).

### Upravljanje lozinkama i nalozima

MySQL 8 ima ugrađene mehanizme koje vredi znati:

```sql
-- lozinka ističe za 90 dana
ALTER USER 'app'@'10.0.1.%' PASSWORD EXPIRE INTERVAL 90 DAY;

-- ne sme ponovo koristiti poslednjih 5 lozinki
ALTER USER 'app'@'10.0.1.%' PASSWORD HISTORY 5;

-- posle 3 neuspela pokušaja, nalog se zaključava na 2 sata
ALTER USER 'app'@'10.0.1.%'
  FAILED_LOGIN_ATTEMPTS 3 PASSWORD_LOCK_TIME 2;

-- privremeno zaključavanje naloga
ALTER USER 'stari_app'@'%' ACCOUNT LOCK;
ALTER USER 'stari_app'@'%' ACCOUNT UNLOCK;

-- ograničenje resursa
ALTER USER 'report'@'10.0.2.%'
  WITH MAX_USER_CONNECTIONS 5 MAX_QUERIES_PER_HOUR 10000;
```

`FAILED_LOGIN_ATTEMPTS` je jednostavna zaštita od pogađanja lozinke i nema razloga da je ne koristite na nalozima koji nisu kritični za rad aplikacije. Pazite samo da je ne stavite na aplikacioni nalog bez razmišljanja — pogrešna lozinka u konfiguraciji tada zaključava celu aplikaciju.

**`ACCOUNT LOCK` je vaš alat za penzionisanje naloga.** Kada ne znate da li se nalog još negde koristi, ne brišite ga — zaključajte ga, sačekajte nedelju dana, pa obrišite ako se niko nije javio. Zaključavanje je reverzibilno, brisanje nije.

### Rotacija lozinke bez ispada

Klasičan problem: menjate lozinku aplikacionog naloga, ali aplikaciju ne možete restartovati u istoj sekundi. MySQL 8 ima rešenje:

```sql
-- postavi novu lozinku, ali zadrži i staru kao važeću
ALTER USER 'app'@'10.0.1.%' IDENTIFIED BY 'nova' RETAIN CURRENT PASSWORD;

-- ... ažurirate konfiguraciju aplikacije i restartujete je ...

-- kada potvrdite da radi sa novom, uklonite staru
ALTER USER 'app'@'10.0.1.%' DISCARD OLD PASSWORD;
```

Ovim se rotacija lozinke pretvara iz rizične operacije u rutinu.

---

## 12. `GRANT` u praksi

### Nivoi privilegija

Privilegije se dodeljuju na nekoliko nivoa, od najšireg ka najužem:

```sql
GRANT SELECT ON *.*            TO ...;  -- globalno: sve baze
GRANT SELECT ON shop.*         TO ...;  -- jedna baza
GRANT SELECT ON shop.korisnici TO ...;  -- jedna tabela
GRANT SELECT (ime, email) ON shop.korisnici TO ...;  -- pojedine kolone
GRANT EXECUTE ON PROCEDURE shop.obracun TO ...;      -- rutina
```

Praktično pravilo: **nivo baze (`baza.*`) je pravo mesto za aplikacione naloge.** Nivo tabele je za posebne slučajeve, a nivo kolone je uglavnom prekomplikovan za održavanje. Globalni nivo (`*.*`) rezervišite za administrativne i infrastrukturne naloge.

### Statičke i dinamičke privilegije

MySQL 8 je uveo podelu koja vam pojednostavljuje posao.

**Statičke privilegije** su klasične: `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `CREATE`, `DROP`, `ALTER`, `INDEX`, `REFERENCES`, `LOCK TABLES`, `SHOW VIEW`, `TRIGGER`, `EVENT`, `EXECUTE`, `FILE`, `PROCESS`, `RELOAD`, `SHUTDOWN`, `REPLICATION SLAVE`, `REPLICATION CLIENT`.

**Dinamičke privilegije** su nastale razbijanjem stare `SUPER` privilegije na sitnije delove. Umesto "sme sve administrativno", sada možete dati tačno ono što treba:

| Privilegija | Za šta služi |
|---|---|
| `BACKUP_ADMIN` | XtraBackup i slični alati |
| `SYSTEM_VARIABLES_ADMIN` | `SET GLOBAL` / `SET PERSIST` |
| `CONNECTION_ADMIN` | prekidanje tuđih konekcija, `KILL` |
| `REPLICATION_APPLIER` | primena replikacije |
| `CLONE_ADMIN` | CLONE plugin |
| `SERVICE_CONNECTION_ADMIN` | povezivanje kada je server pun |
| `SYSTEM_USER` | zaštita naloga od modifikacije od strane drugih |
| `PERSIST_RO_VARIABLES_ADMIN` | `SET PERSIST_ONLY` |

**`SUPER` je zastarela i ne treba je koristiti u novim `GRANT` naredbama.** Ako je vidite na nasleđenom serveru, to je kandidat za zamenu konkretnim dinamičkim privilegijama.

Spisak svih dostupnih:

```sql
SELECT * FROM information_schema.user_privileges
WHERE grantee LIKE "'root'%";

SHOW PRIVILEGES;
```

### Osnovna sintaksa

```sql
-- kreiranje naloga
CREATE USER 'ime'@'host' IDENTIFIED BY 'lozinka';

-- dodela
GRANT SELECT, INSERT ON baza.* TO 'ime'@'host';

-- oduzimanje
REVOKE INSERT ON baza.* FROM 'ime'@'host';

-- pregled
SHOW GRANTS FOR 'ime'@'host';
SHOW CREATE USER 'ime'@'host';

-- brisanje
DROP USER 'ime'@'host';
```

Kada je potreban `FLUSH PRIVILEGES`? **Skoro nikad.** `GRANT`, `REVOKE`, `CREATE USER` i `DROP USER` primenjuju izmene odmah. `FLUSH PRIVILEGES` je potreban samo ako ste menjali tabele u `mysql` šemi direktnim `INSERT`/`UPDATE` naredbama — što ne treba da radite.

### Gotovi recepti

Ovo je deo koji ćete najviše koristiti. Svaki recept je minimalan skup privilegija za konkretnu ulogu.

#### Aplikacioni nalog — standardna web aplikacija

```sql
CREATE USER 'app_shop'@'10.0.1.%'
  IDENTIFIED BY 'duga-nasumicna-lozinka'
  WITH MAX_USER_CONNECTIONS 100;

GRANT SELECT, INSERT, UPDATE, DELETE
  ON shop.* TO 'app_shop'@'10.0.1.%';
```

Ako aplikacija koristi stored procedure ili funkcije, dodajte:

```sql
GRANT EXECUTE ON shop.* TO 'app_shop'@'10.0.1.%';
```

**Ovo je sve.** Nema `CREATE`, nema `DROP`, nema `ALTER`, nema `FILE`, nema ničega globalnog. Aplikacija u normalnom radu ne menja strukturu baze.

#### Nalog za migracije šeme

Ako aplikacioni framework sam menja šemu (Rails, Django, Laravel, Flyway), to je **odvojen nalog** koji se koristi samo pri deploy-u:

```sql
CREATE USER 'migrate_shop'@'10.0.1.%' IDENTIFIED BY '...';

GRANT SELECT, INSERT, UPDATE, DELETE,
      CREATE, DROP, ALTER, INDEX, REFERENCES
  ON shop.* TO 'migrate_shop'@'10.0.1.%';
```

Razdvajanje ova dva naloga je jedna od najkorisnijih stvari koje možete uraditi. Aplikacija u radu nema pravo da obriše tabelu; deploy proces ima, ali traje minut i pokreće se pod kontrolom.

Ako developer insistira da mu treba jedan nalog za sve — objasnite da je `DROP TABLE` kroz SQL injekciju moguć samo ako nalog ima to pravo. To je konkretan argument, a ne birokratija.

#### Nalog samo za čitanje / izveštaje

```sql
CREATE USER 'report'@'10.0.2.%'
  IDENTIFIED BY '...'
  WITH MAX_USER_CONNECTIONS 5 MAX_QUERIES_PER_HOUR 20000;

GRANT SELECT ON shop.* TO 'report'@'10.0.2.%';
```

Ograničenja resursa su ovde bitna. Nalozi za izveštavanje su tipičan izvor upita koji oboriju server (Modul 7). `MAX_USER_CONNECTIONS` sprečava da jedan analitički alat zauzme sve slotove.

Idealno, ovakav nalog uopšte ne postoji na primarnom serveru, nego na replici (Modul 8).

#### Backup nalog za `mysqldump`

```sql
CREATE USER 'backup'@'localhost' IDENTIFIED WITH auth_socket AS 'backup';

GRANT SELECT, SHOW VIEW, TRIGGER, EVENT,
      LOCK TABLES, RELOAD, PROCESS, REPLICATION CLIENT
  ON *.* TO 'backup'@'localhost';
```

Objašnjenje po stavkama, jer ćete ovo prilagođavati:

- `SELECT` — čitanje podataka, očigledno.
- `SHOW VIEW` — za dump pogleda.
- `TRIGGER` — za `--triggers` (podrazumevano uključeno).
- `EVENT` — za `--events`.
- `LOCK TABLES` — potrebno ako **ne** koristite `--single-transaction`.
- `RELOAD` — za `FLUSH TABLES` i `--flush-logs`.
- `PROCESS` — potrebno u MySQL 8 za čitanje informacija o tablespace-ovima; može se izbeći opcijom `--no-tablespaces`.
- `REPLICATION CLIENT` — za `--source-data` / `--master-data`, tj. za zapisivanje pozicije binloga, što je preduslov za point-in-time recovery.

Ako neka od ovih privilegija fali, `mysqldump` pada sa porukom koja je uglavnom jasna. Kada dobijete takvu grešku, vratite se na ovaj spisak.

#### Backup nalog za Percona XtraBackup

```sql
CREATE USER 'xtrabackup'@'localhost' IDENTIFIED BY '...';

GRANT BACKUP_ADMIN, PROCESS, RELOAD, LOCK TABLES, REPLICATION CLIENT
  ON *.* TO 'xtrabackup'@'localhost';

GRANT SELECT ON performance_schema.log_status
  TO 'xtrabackup'@'localhost';
```

Uočite `BACKUP_ADMIN` — dinamička privilegija koja je zamenila deo nekadašnjeg `SUPER`-a. Ovaj nalog **ne treba** `SELECT` na podacima, jer XtraBackup kopira fajlove direktno, a ne kroz SQL.

#### Monitoring nalog

```sql
CREATE USER 'exporter'@'localhost'
  IDENTIFIED BY '...'
  WITH MAX_USER_CONNECTIONS 3;

GRANT PROCESS, REPLICATION CLIENT ON *.* TO 'exporter'@'localhost';
GRANT SELECT ON performance_schema.* TO 'exporter'@'localhost';
```

Ovo je skup potreban za `mysqld_exporter` (Prometheus) i slične alate. `PROCESS` omogućava `SHOW PROCESSLIST`, `REPLICATION CLIENT` omogućava praćenje zaostajanja replike.

`MAX_USER_CONNECTIONS 3` je bitno: monitoring alat koji se raspomami ne sme da potroši sve konekcije i time obori aplikaciju.

Nalog **nema** `SELECT` na korisničkim podacima. Monitoring ne treba da vidi sadržaj baze.

#### Nalog za replikaciju

```sql
CREATE USER 'repl'@'10.0.1.%' IDENTIFIED BY '...' REQUIRE SSL;
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'10.0.1.%';
```

Jedna privilegija, ništa više. `REQUIRE SSL` je ovde posebno važno jer replikacioni tok sadrži **sve** vaše podatke i putuje mrežom (Modul 4).

### Uloge — kada imate više od nekoliko naloga

Ako održavate desetak aplikacija, dodeljivanje privilegija naloga po naloga postaje nepregledno. MySQL 8 ima uloge.

```sql
CREATE ROLE 'shop_rw', 'shop_ro';

GRANT SELECT, INSERT, UPDATE, DELETE ON shop.* TO 'shop_rw';
GRANT SELECT ON shop.* TO 'shop_ro';

CREATE USER 'app1'@'10.0.1.%' IDENTIFIED BY '...';
GRANT 'shop_rw' TO 'app1'@'10.0.1.%';

CREATE USER 'analitika'@'10.0.2.%' IDENTIFIED BY '...';
GRANT 'shop_ro' TO 'analitika'@'10.0.2.%';
```

**Zamka koja obara ljude prvi put:** dodeljena uloga **nije automatski aktivna** pri prijavi. Aplikacija se poveže i nema nijednu privilegiju, iako `SHOW GRANTS` izgleda ispravno.

Rešenje, po nalogu:

```sql
SET DEFAULT ROLE ALL TO 'app1'@'10.0.1.%';
```

Ili globalno, što je za aplikacione servere obično ono što želite:

```ini
[mysqld]
activate_all_roles_on_login = ON
```

Provera šta je stvarno aktivno u trenutnoj sesiji:

```sql
SELECT CURRENT_ROLE();
SHOW GRANTS FOR CURRENT_USER() USING 'shop_rw';
```

### Revizija postojećih naloga

Kada preuzimate nasleđen server, ovo su upiti od kojih počinjete.

**Ko ima globalne privilegije:**

```sql
SELECT user, host FROM mysql.user
WHERE Super_priv = 'Y' OR Grant_priv = 'Y' OR File_priv = 'Y'
   OR Shutdown_priv = 'Y' OR Create_user_priv = 'Y';
```

**Ko sme da se poveže odakle bilo:**

```sql
SELECT user, host FROM mysql.user WHERE host = '%';
```

**Ko koristi zastareli autentikacioni plugin:**

```sql
SELECT user, host, plugin FROM mysql.user
WHERE plugin = 'mysql_native_password';
```

**Nalozi bez lozinke:**

```sql
SELECT user, host FROM mysql.user
WHERE (authentication_string = '' OR authentication_string IS NULL)
  AND plugin NOT IN ('auth_socket');
```

**Kompletan izvoz svih privilegija:**

Najbrži alat je iz Percona Toolkit-a:

```bash
sudo apt install percona-toolkit
pt-show-grants > grants-$(date +%F).sql
```

Izlaz je gotov SQL koji možete pregledati, staviti u Git i iskoristiti za obnavljanje naloga na novom serveru. **Ovo treba da bude deo vašeg backup procesa** — `mysqldump` po podrazumevanim podešavanjima ne obuhvata naloge.

Alternativa bez dodatnih alata, u MySQL 8.0.20 i novijim:

```bash
mysqldump --system=users > korisnici-$(date +%F).sql
```

Vratićemo se na ovo u Modulu 5, jer je „vratili smo bazu ali se aplikacija ne može prijaviti" čest ishod nepotpunog backupa.

---

## 13. Zaboravljena root lozinka

### Prvo: proverite da li uopšte imate problem

Na Ubuntuu sa paketom iz distribucije, `root@localhost` koristi `auth_socket`. Lozinka nije ni potrebna:

```bash
sudo mysql
```

Ako ovo radi — nemate problem, samo ste tražili pogrešnu stvar.

### Drugo: probajte održavateljski nalog

Ubuntu paket kreira nalog `debian-sys-maint` sa punim privilegijama i lozinkom zapisanom u fajlu:

```bash
sudo cat /etc/mysql/debian.cnf
sudo mysql --defaults-file=/etc/mysql/debian.cnf
```

Ako se povežete, možete odmah rešiti stvar bez ikakvog restartovanja:

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'nova-lozinka';
```

**Ovaj put pokušajte pre svega ostalog.** Ne zahteva prekid rada servisa, što na produkciji znači sve.

### Treće: `--init-file` (preporučena metoda)

Ako prethodna dva ne uspeju, ovo je bezbedniji od dva klasična načina, jer ni u jednom trenutku ne ostavlja server bez provere privilegija.

**Korak 1 — zaustavite server**

```bash
sudo systemctl stop mysql
```

**Korak 2 — napravite SQL fajl**

Lokacija je bitna. Koristimo `/var/lib/mysql-files/` jer je taj direktorijum već dozvoljen u AppArmor profilu (Modul 2), pa nema dodatnog posla.

```bash
sudo tee /var/lib/mysql-files/reset.sql > /dev/null <<'EOF'
ALTER USER 'root'@'localhost' IDENTIFIED BY 'nova-jaka-lozinka';
EOF

sudo chown mysql:mysql /var/lib/mysql-files/reset.sql
sudo chmod 600 /var/lib/mysql-files/reset.sql
```

Ako je root nalog na `auth_socket` a vi želite lozinku:

```sql
ALTER USER 'root'@'localhost'
  IDENTIFIED WITH caching_sha2_password BY 'nova-jaka-lozinka';
```

**Korak 3 — privremeni systemd override**

```bash
sudo systemctl edit mysql
```

```ini
[Service]
ExecStart=
ExecStart=/usr/sbin/mysqld --init-file=/var/lib/mysql-files/reset.sql
```

Prazna linija `ExecStart=` je obavezna — ona poništava originalnu vrednost. Bez nje systemd pokušava da izvrši obe komande i javlja grešku.

```bash
sudo systemctl daemon-reload
sudo systemctl start mysql
```

**Korak 4 — proverite**

```bash
mysql -u root -p
```

**Korak 5 — počistite**

Ovo je korak koji se zaboravlja, a ostavlja lozinku u fajlu na disku:

```bash
sudo rm /var/lib/mysql-files/reset.sql
sudo rm /etc/systemd/system/mysql.service.d/override.conf
sudo systemctl daemon-reload
sudo systemctl restart mysql
```

Potvrda da je override zaista uklonjen:

```bash
systemctl cat mysql
```

### Četvrto: `--skip-grant-tables` (poslednja opcija)

Ovaj način startuje server **bez ikakve provere privilegija**. Svako ko se poveže postaje pun administrator. Zato ga koristite samo kad `--init-file` ne prolazi i uvek uz `--skip-networking`.

```bash
sudo systemctl stop mysql
sudo systemctl edit mysql
```

```ini
[Service]
ExecStart=
ExecStart=/usr/sbin/mysqld --skip-grant-tables --skip-networking
```

```bash
sudo systemctl daemon-reload
sudo systemctl start mysql
mysql -u root
```

U klijentu — obratite pažnju na prvi red:

```sql
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'nova-jaka-lozinka';
```

`FLUSH PRIVILEGES` je ovde obavezan. Bez njega server nije učitao tabele privilegija i `ALTER USER` javlja grešku. Ovo je detalj zbog koga ljudi odustaju od ove metode misleći da ne radi.

Zatim isto čišćenje kao u prethodnoj metodi.

**`--skip-networking` nije opcionalno.** Bez njega, u periodu dok server radi u ovom režimu, bilo ko sa mrežnim pristupom portu 3306 ima pun administratorski pristup vašoj bazi, bez lozinke. Ako morate raditi ovo na serveru izloženom mreži, prvo zatvorite port na firewall-u.

### Beleška o oporavku

Nakon svakog reseta root lozinke, proverite da niste ostavili trag:

```bash
systemctl cat mysql | grep -A2 ExecStart
ls -la /var/lib/mysql-files/
history | grep -i "identified by"
```

Poslednja komanda je podsetnik da lozinke ne kucate u shell-u. Ako jeste, očistite `~/.bash_history`.

---

## 14. Zašto `GRANT ALL PRIVILEGES ON *.*` skoro nikad nije tačan odgovor

Do sada smo pričali o tome kako se radi. Ovo poglavlje je o tome zašto se isplati.

### Šta `ALL PRIVILEGES ON *.*` zapravo znači

To nije "sme sve u svojoj bazi". To je:

- sme da čita **svaku** bazu na serveru, uključujući `mysql` šemu sa heševima lozinki svih naloga,
- sme da obriše bilo koju bazu, uključujući baze drugih aplikacija,
- ima `FILE` privilegiju,
- može da menja globalne promenljive,
- može da kreira nove korisnike (ako je uz to i `WITH GRANT OPTION`).

### Tri konkretna scenarija

**SQL injekcija.** Aplikacija ima ranjivost. Sa `SELECT, INSERT, UPDATE, DELETE ON shop.*`, napadač može da čita i menja podatke te aplikacije — loše, ali ograničeno i oporavljivo iz backupa. Sa `ALL ON *.*`, isti propust omogućava `DROP DATABASE` na svakoj bazi na serveru i čitanje heševa lozinki iz `mysql.user`.

Razlika između ta dva ishoda je jedan red u `GRANT` naredbi.

**`FILE` privilegija.** Ovo je najpodcenjenija stavka u celom spisku. Nalog sa `FILE` može:

```sql
SELECT LOAD_FILE('/etc/passwd');
SELECT '<?php system($_GET["c"]); ?>' INTO OUTFILE '/var/www/html/x.php';
```

Dakle: čitanje fajlova sa servera i **pisanje web shell-a**. Iz kompromitovanog naloga baze dobija se izvršavanje koda na serveru.

Jedina odbrana koja ostaje je `secure_file_priv` (Modul 2), a on je često pogrešno podešen. Nemojte se oslanjati na njega — jednostavno ne dajte `FILE`.

```sql
SELECT @@secure_file_priv;
SELECT user, host FROM mysql.user WHERE File_priv = 'Y';
```

**Nevidljive izmene.** Nalog sa `SYSTEM_VARIABLES_ADMIN` (ili starim `SUPER`) može da isključi binarno logovanje za svoju sesiju:

```sql
SET sql_log_bin = 0;
```

Sve što uradi nakon toga **ne ulazi u binlog**. To znači: ne stiže do replika i ne postoji u point-in-time recovery istoriji. Vaš forenzički trag nestaje, a replika i primarni server tiho se razilaze.

### Praktičan pristup razgovoru sa developerom

Kada dobijete zahtev "treba mi pun pristup bazi", ne odbijajte — pitajte:

1. **Koje baze aplikacija dodiruje?** Odgovor je gotovo uvek jedna.
2. **Da li menja strukturu tabela u toku rada, ili samo pri deploy-u?** Ako samo pri deploy-u — dva naloga.
3. **Sa kojih mašina se povezuje?** To vam popunjava host polje.
4. **Koristi li stored procedure?** Ako da — `EXECUTE`.
5. **Koristi li `LOAD DATA INFILE`?** Ako da, razgovarajte o alternativi, jer to vodi ka `FILE` privilegiji.

Pet pitanja i imate tačan `GRANT`. Ovo traje pet minuta i rešava problem za godine unapred.

Ako aplikacija stvarno padne zbog nedostajuće privilegije, greška je konkretna i lako se čita:

```
ERROR 1142 (42000): CREATE command denied to user 'app_shop'@'10.0.1.50'
for table 'shop.temp_import'
```

Piše tačno šta fali, kom nalogu i na kojoj tabeli. Dodate tu jednu privilegiju i idete dalje. To je mnogo bolja pozicija od one u kojoj ste dali sve unapred i ne znate šta se stvarno koristi.

### Postupak sređivanja nasleđenog servera

Ako preuzimate server na kome sve radi kao root, ne menjajte ništa naglo. Radite ovako:

1. **Popišite** — `pt-show-grants` i sačuvajte izlaz u Git.
2. **Napravite paralelni nalog** sa restriktivnim privilegijama, pored postojećeg.
3. **Prebacite jednu instancu aplikacije** na nov nalog i pustite je da radi.
4. **Pratite greške** u aplikacionom logu nekoliko dana. Svaka nedostajuća privilegija se javi kao jasna poruka.
5. **Doterajte** `GRANT` na osnovu stvarnih grešaka.
6. **Prebacite ostatak** aplikacije.
7. **Zaključajte stari nalog** sa `ACCOUNT LOCK`, ne brišite ga.
8. **Obrišite** nakon mesec dana bez incidenta.

Ovaj postupak nema tačku u kojoj rušite produkciju, a svaki korak je reverzibilan.

---

## Kontrolna lista na kraju modula

```sql
-- 1. Pregled svih naloga
SELECT user, host, plugin, account_locked FROM mysql.user ORDER BY user;

-- 2. Nema naloga sa host = '%' bez opravdanja
SELECT user, host FROM mysql.user WHERE host = '%';

-- 3. Nema anonimnih naloga
SELECT user, host FROM mysql.user WHERE user = '';

-- 4. Nema FILE privilegije na aplikacionim nalozima
SELECT user, host FROM mysql.user WHERE File_priv = 'Y';

-- 5. Nema zastarelog plugina
SELECT user, host FROM mysql.user WHERE plugin = 'mysql_native_password';

-- 6. Aplikacioni nalog ima samo ono što mu treba
SHOW GRANTS FOR 'app_shop'@'10.0.1.%';

-- 7. Znate šta je aktivno u trenutnoj sesiji
SELECT USER(), CURRENT_USER(), CURRENT_ROLE();
```

```bash
# 8. DNS razrešavanje je isključeno
mysql -e "SELECT @@skip_name_resolve;"

# 9. Privilegije su izvezene i pod verzionisanjem
pt-show-grants > /backup/grants-$(date +%F).sql

# 10. Backup i monitoring nalozi postoje i nisu root
mysql -e "SHOW GRANTS FOR 'backup'@'localhost';"
mysql -e "SHOW GRANTS FOR 'exporter'@'localhost';"
```

---

## Vežbe

**Vežba 1 — Poklapanje hostova**
Napravite `'test'@'%'` sa `GRANT SELECT ON shop.*` i lozinkom `aaa`, pa `'test'@'localhost'` sa lozinkom `bbb` i bez ijedne privilegije. Povežite se sa `-h localhost` i sa `-h 127.0.0.1`, obe lozinke. Za svaki od četiri ishoda zabeležite šta vraćaju `USER()` i `CURRENT_USER()` i objasnite zašto.

**Vežba 2 — `auth_socket` za skripte**
Napravite Linux korisnika `backup` i MySQL nalog `'backup'@'localhost'` sa `auth_socket`. Pokrenite `mysqldump` kao taj korisnik, bez ijedne lozinke u komandi ili fajlu. Zatim pokušajte isto kao drugi Linux korisnik i objasnite grešku.

**Vežba 3 — Minimalni `GRANT` metodom pokušaja**
Napravite nalog sa samo `SELECT ON shop.*`. Pokušajte redom `INSERT`, `UPDATE`, `DELETE`, `CREATE TABLE`, `DROP TABLE`. Zabeležite tačan tekst svake greške. Ovo je vežba prepoznavanja poruka — sledeći put kad je vidite u aplikacionom logu, znaćete šta fali za pola sekunde.

**Vežba 4 — Uloge i zamka sa aktivacijom**
Napravite ulogu `shop_rw`, dodelite je nalogu, povežite se i pokušajte `SELECT`. Objasnite grešku. Zatim rešite problem na dva načina — sa `SET DEFAULT ROLE` i sa `activate_all_roles_on_login` — i objasnite kada koji bira.

**Vežba 5 — Reset root lozinke, tri puta**
Namerno "zaboravite" root lozinku i povratite pristup na sva tri načina: preko `debian.cnf`, preko `--init-file` i preko `--skip-grant-tables`. Izmerite koliko svaki traje i koliko dugo je servis nedostupan. Zaključite koji biste koristili na produkciji u tri ujutru.

**Vežba 6 — Opasnost `FILE` privilegije**
Na **izolovanoj test mašini** dajte nalogu `FILE` privilegiju i proverite `SELECT LOAD_FILE('/etc/passwd')`. Zatim promenite `secure_file_priv` i ponovite. Oduzmite `FILE` i ponovite. Ovu vežbu radite samo na mašini koju možete da bacite.

**Vežba 7 — Revizija nasleđenog servera**
Napravite pet naloga sa namerno lošim postavkama (host `%`, `ALL ON *.*`, prazna lozinka, `mysql_native_password`, `FILE`). Zatim napišite jedan SQL skript koji ih sve pronalazi i ispisuje izveštaj. Sačuvajte taj skript — koristićete ga.

---

## Šta sledi

U **Modulu 4** izlazimo na mrežu. Objasnićemo `bind-address` i zašto MySQL podrazumevano sluša samo na `localhost`, kako se konfiguriše firewall i zašto port 3306 nikada ne ide direktno na internet, kako se pravi bezbedan udaljeni pristup preko SSH tunela i kada je VPN bolji izbor, i kako se podešava pravi TLS između aplikacije i baze — sa sertifikatima koje klijent može da verifikuje, umesto onih koje je MySQL sam generisao.

Host polje iz ovog modula i firewall iz narednog su dva sloja iste odbrane. Tek zajedno imaju smisla.
