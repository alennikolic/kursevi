# Modul 1: Uvod i instalacija

*MySQL za Linux administratore*

---

## Uvod: zašto ovaj kurs postoji

Postoji ogromna količina materijala o MySQL-u, ali skoro sav je pisan za nekoga ko se bavi podacima — za developera koji piše upite ili za DBA koji optimizuje indekse. Vi verovatno niste ni jedno ni drugo.

Vi ste čovek kome je neko dao server. Na tom serveru radi aplikacija, a ispod aplikacije radi MySQL. Vaš posao počinje onog trenutka kad se nešto pokvari, kad disk počne da se puni, kad neko traži da se baza vrati na stanje od juče u 14h, ili kad monitoring pošalje alarm u tri ujutru.

Iz tog ugla, MySQL nije "sistem za upravljanje bazama podataka". MySQL je:

- **jedan proces** (`mysqld`) koji troši RAM i CPU,
- **jedan systemd servis** (`mysql.service`) koji ume da ne startuje,
- **jedan direktorijum sa fajlovima** (`/var/lib/mysql`) koji raste dok ne popuni disk,
- **jedan mrežni socket** koji ne sme da bude otvoren ka svetu,
- i **jedna stvar koju morate umeti da vratite iz backupa**, jer ako to ne umete, sve ostalo je nebitno.

Ovaj kurs vas uči tome. Naučićete i nešto SQL-a, ali tek toliko da se snađete — SQL je alat, a ne tema.

---

## 1. Šta Linux administrator zapravo mora da zna o MySQL-u

Pre nego što išta instaliramo, vredi razgraničiti teren. Ako pokušate da naučite "sve o MySQL-u", udavićete se u temama koje vam nikad neće zatrebati.

### Šta morate da znate

**Životni ciklus servisa.** Kako se MySQL startuje, gasi, restartuje; šta radi tokom starta; zašto ponekad startuje 40 sekundi, a ponekad 8 minuta. Ovo je osnova svega.

**Gde su fajlovi.** Konfiguracija, podaci, logovi, socket, PID fajl. Kad vam neko kaže "baza je pala", prvo pitanje je "šta piše u error logu", a to znači da morate znati gde je error log.

**Model korisnika i privilegija.** MySQL ima svoje korisnike koji nemaju nikakve veze sa Linux korisnicima. Ovo je izvor ogromnog broja nesporazuma i grešaka u konfiguraciji.

**Backup i restore.** Ne backup — *restore*. Backup koji nikad niste testirali nije backup, to je fajl. Ovo je najvažniji modul celog kursa i on nosi broj 5.

**Logovi.** Četiri različita loga, svaki za drugu svrhu. Onaj koji vas najviše zanima (error log) je najmanje čitan.

**Resursi.** Koliko RAM-a MySQL zaista troši i zašto, koliko fajl deskriptora mu treba, kako se ponaša kad ostane bez diska. Ovo je klasičan sysadmin posao, samo primenjen na jedan konkretan proces.

**Mrežni pristup.** Ko sme da se poveže, odakle, preko čega. `bind-address`, firewall, TLS, SSH tunel.

**Replikaciju — na nivou održavanja.** Ne morate umeti da dizajnirate HA arhitekturu, ali morate umeti da prepoznate da replika kasni i da je vratite u rad.

### Šta slobodno možete da ne znate

**Dizajn šeme.** Normalizacija, izbor tipova kolona, strani ključevi — to je posao developera. Vi treba samo da razumete šta gledate kad otvorite tuđu bazu.

**Optimizaciju upita.** `EXPLAIN`, plan izvršavanja, izbor indeksa — korisno je, ali to nije vaša odgovornost. Vaša odgovornost je da **pronađete** spor upit (slow query log) i da ga **prosledite** onome ko ume da ga popravi. To je velika razlika i ona vam štedi mesece učenja.

**Stored procedure, trigere, view-ove.** Treba da znate da postoje i da ih backup mora obuhvatiti (zato `mysqldump --routines --triggers`). Dalje od toga — ne.

**Napredni SQL.** Prozorske funkcije, CTE, rekurzivni upiti. Zanimljivo, ali nije vaš posao.

**Interne mehanizme InnoDB-a do detalja.** Dovoljno je da razumete šta je buffer pool i šta je redo log. Ne morate znati kako izgleda B+ stablo.

### Podela odgovornosti, ukratko

| Oblast | Vi | Developer / DBA |
|---|---|---|
| Instalacija i nadogradnja | ✅ | |
| Konfiguracija servera (`my.cnf`) | ✅ | savetuje |
| Backup i restore | ✅ | |
| Monitoring i alarmi | ✅ | |
| Korisnici i privilegije | ✅ | traži |
| Dizajn tabela i indeksa | | ✅ |
| Pisanje i optimizacija upita | pronalazi problem | ✅ rešava |
| Replikacija — postavka | ✅ | |
| Replikacija — logika (šta se replicira) | | ✅ |

Zapamtite ovu tabelu. Ona je odgovor na pitanje "da li ja treba ovo da naučim".

---

## 2. MySQL, MariaDB, Percona — šta je šta i šta imate na serveru

Ovo je prva stvar koja zbunjuje. Kad kažete "MySQL", u praksi možete misliti na najmanje tri različita programa.

### Kratka istorija u pet rečenica

MySQL je nastao 1995. u švedskoj firmi MySQL AB. Sun Microsystems ga je kupio 2008, a Oracle je 2010. kupio Sun — i time MySQL. Deo originalnih autora, na čelu sa Michaelom "Monty" Wideniusom, napravio je fork pod imenom **MariaDB**, iz bojazni kako će Oracle upravljati projektom. Firma Percona napravila je svoj fork MySQL-a pod imenom **Percona Server for MySQL**, fokusiran na performanse i alate za dijagnostiku. Danas su to tri odvojena projekta koji su nekada bili isti kod.

### MySQL (Oracle)

Referentna implementacija. Community Edition je pod GPL-om i besplatan; Enterprise Edition je komercijalan i donosi dodatke (thread pool, enterprise backup, audit, TDE, podršku).

Trenutna generacija je serija 8.x. Numeracija verzija je 2023. promenjena i to je bitno da razumete:

- **8.0** — dugovečna verzija koja je bila standard godinama. Ulazi u kraj podrške sredinom 2020-ih; proverite aktuelan status na `endoflife.date/mysql` pre nego što planirate produkciju.
- **8.1, 8.2, 8.3** — *Innovation* izdanja. Nove funkcije, kratak vek podrške (do sledećeg izdanja). Nisu za produkciju osim ako vam konkretno treba nešto novo.
- **8.4** — *LTS* izdanje. Dugoročna podrška, ovo je danas razuman izbor za nov produkcijski server.
- **9.x** — nova Innovation serija.

Pravilo za sysadmina: **birate LTS, ne Innovation.** Innovation izdanja postoje da bi Oracle brže isporučivao funkcije, ne da bi vi svaka tri meseca radili nadogradnju produkcije.

### MariaDB

Fork iz 2009. koji je dugo bio "drop-in replacement" za MySQL. **To više nije tačno** i ovo je najvažnija stvar iz ovog poglavlja.

MariaDB i MySQL su se toliko razišli da danas treba da ih tretirate kao srodne, ali različite baze:

- **GTID replikacija je nekompatibilna.** Ne možete jednostavno replicirati sa MySQL 8 na MariaDB niti obrnuto.
- **Backup alati su različiti.** Percona XtraBackup radi sa MySQL-om i Percona Serverom; za MariaDB koristite `mariabackup`.
- **JSON implementacija je različita.** MySQL ima nativni binarni JSON tip; MariaDB koristi alias na `LONGTEXT` sa CHECK ograničenjem.
- **MySQL 8 ima transakcioni data dictionary** (nema više `.frm` fajlova); MariaDB je zadržala drugačiji pristup.
- **Numeracija verzija se ne poklapa.** MariaDB 10.6, 10.11, 11.4 nemaju veze sa MySQL 8.x brojevima.
- **Neke funkcije postoje samo na jednoj strani** — MariaDB ima sistemski verzionisane tabele i neke storage engine-e kojih nema u MySQL-u; MySQL ima CLONE plugin, drugačiji optimizator, drugačije prozorske funkcije u detaljima.

MariaDB je odličan proizvod. Problem nastaje kada neko pretpostavi da su isti pa u dokumentaciji piše "MySQL", a na serveru je MariaDB — ili obrnuto. To vodi u komande koje ne rade i backup procedure koje ne rade.

**Bitno za Ubuntu:** paketi `mysql-server` i `mariadb-server` se međusobno isključuju. Instalacija jednog uklanja drugi. Ako to uradite nepažljivo na produkciji, uklonićete servis (podaci u `/var/lib/mysql` obično ostaju, ali format može biti nekompatibilan). Nikad ne izvršavajte `apt install mariadb-server` na serveru gde radi MySQL "samo da probate".

### Percona Server for MySQL

Fork MySQL-a koji ostaje binarno i funkcionalno kompatibilan sa Oracle MySQL-om, ali dodaje stvari korisne u produkciji: bolji instrumenti, dodatne statistike, neke Enterprise funkcije besplatno (npr. audit log, thread pool).

Percona je posebno relevantna zbog svojih **alata**, koje ćete koristiti bez obzira na to koji server imate:

- **Percona XtraBackup** — fizički backup bez zaustavljanja servisa (Modul 5),
- **Percona Toolkit** — `pt-query-digest`, `pt-online-schema-change`, `pt-table-checksum` i još tridesetak alata (Moduli 6, 9).

Ove alate ćemo koristiti kroz ceo kurs. Vredi ih znati čak i ako nikad ne instalirate Percona Server.

### Kako da utvrdite šta imate na serveru

Ovo je prva komanda koju izvršite na nasleđenom serveru:

```bash
mysql --version
```

Primeri izlaza i njihovo tumačenje:

```
mysql  Ver 8.0.36-0ubuntu0.22.04.1 for Linux on x86_64 ((Ubuntu))
```
→ Oracle MySQL 8.0, iz Ubuntu repozitorijuma (obratite pažnju na `0ubuntu0.22.04.1`).

```
mysql  Ver 8.4.3 for Linux on x86_64 (MySQL Community Server - GPL)
```
→ Oracle MySQL 8.4 LTS, iz zvaničnog Oracle repozitorijuma.

```
mysql  Ver 15.1 Distrib 10.11.6-MariaDB, for debian-linux-gnu
```
→ MariaDB 10.11. Pazite: `Ver 15.1` je verzija *klijenta*, ne servera. Server je 10.11.6.

```
mysql  Ver 8.0.36-28 for Linux on x86_64 (Percona Server (GPL))
```
→ Percona Server.

Ali `mysql --version` vam govori samo verziju klijenta. Server može biti drugi. Provera servera:

```bash
mysql -e "SELECT VERSION(), @@version_comment;"
```

I provera odakle je paket došao:

```bash
apt policy mysql-server mariadb-server percona-server-server 2>/dev/null
dpkg -l | grep -Ei 'mysql|mariadb|percona'
```

`apt policy` će vam pokazati i koji repozitorijum je izvor — to je često važnije od same verzije, jer određuje odakle stižu bezbednosne zakrpe.

---

## 3. Instalacija MySQL-a na Ubuntu

Sada instaliramo. Prvo odluka koja se ne vraća lako.

### Ubuntu repozitorijum ili Oracle repozitorijum?

Imate dva razumna izvora paketa. Nema univerzalno tačnog odgovora, ali postoje jasni kriterijumi.

**Ubuntu repozitorijum (`apt install mysql-server`)**

Prednosti:
- Verzija je zamrznuta za ceo životni ciklus Ubuntu izdanja — dobijate samo bezbednosne zakrpe, nikad iznenadne promene ponašanja.
- Canonical isporučuje bezbednosne zakrpe kroz standardni proces, uključujući ESM za LTS izdanja.
- Integracija sa sistemom je urađena i testirana: AppArmor profil, `logrotate` konfiguracija, systemd unit, `debian-sys-maint` nalog.
- Radi sa `unattended-upgrades` bez iznenađenja.

Mane:
- Verzija je onakva kakva je bila u trenutku zamrzavanja Ubuntu izdanja. Ako vam treba novija funkcionalnost, nemate je.
- Kad Ubuntu izdanje ode iz podrške, ostajete bez zakrpa dok ne nadogradite ceo sistem.

**Oracle APT repozitorijum**

Prednosti:
- Birate seriju (npr. 8.4 LTS) nezavisno od Ubuntu izdanja.
- Novije verzije, brže zakrpe direktno od proizvođača.

Mane:
- Instalacija je manje "ubuntuovska": drugačiji podrazumevani parametri, AppArmor profil nije uvek isporučen na isti način, `logrotate` konfiguracija se razlikuje.
- Morate sami da vodite računa o repozitorijumu i njegovom GPG ključu. Oracle je u prošlosti rotirao ključ za potpisivanje, što je preko noći slomilo `apt update` na velikom broju servera. To se može ponoviti.
- Nadogradnja Ubuntua i nadogradnja MySQL-a postaju dva odvojena, ponekad sukobljena procesa.

**Preporuka:**

- Ako radite standardnu web aplikaciju i nemate poseban zahtev — **Ubuntu repozitorijum**. Manje pokretnih delova, manje iznenađenja.
- Ako aplikacija zahteva konkretnu verziju, ili vam treba 8.4 LTS a Ubuntu ga nema — **Oracle repozitorijum**, uz svest o dodatnom održavanju.
- Ako naslеđujete server, ne migrirajte samo zato što vam se ne sviđa izbor prethodnika. To nije razlog.

Kroz ovaj kurs koristićemo Ubuntu paket kao osnovu, jer je najčešći slučaj, a razlike u odnosu na Oracle paket ćemo naznačavati gde su bitne.

### Priprema sistema

```bash
sudo apt update
sudo apt upgrade
```

Proverite koju verziju Ubuntu ima za vas:

```bash
apt policy mysql-server
```

Izlaz izgleda otprilike ovako:

```
mysql-server:
  Installed: (none)
  Candidate: 8.0.36-0ubuntu0.22.04.1
  Version table:
     8.0.36-0ubuntu0.22.04.1 500
        500 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 Packages
```

Uvek proverite ovo umesto da pamtite koja verzija ide uz koje izdanje — to se menja sa `-updates` nadogradnjama i različito je po izdanjima.

Provera slobodnog prostora — MySQL ne voli pun disk i ponašaće se ružno (Modul 10):

```bash
df -h /var
free -h
```

### Instalacija iz Ubuntu repozitorijuma

```bash
sudo apt install mysql-server
```

To je sve. Bez prompta, bez pitanja za lozinku.

Ako ste navikli na starije verzije Debiana/Ubuntua gde vas je instalacija pitala za root lozinku — to više nije slučaj. Paket root nalog konfiguriše preko `auth_socket` plugina, o čemu detaljno u poglavlju 5.

Ako vam treba samo klijent (npr. na aplikacionom serveru koji se povezuje na udaljenu bazu):

```bash
sudo apt install mysql-client
```

### Instalacija iz Oracle repozitorijuma

Postupak, za slučaj da vam zatreba:

```bash
# 1. Preuzmite konfiguracioni paket sa dev.mysql.com/downloads/repo/apt/
wget https://dev.mysql.com/get/mysql-apt-config_0.8.xx-1_all.deb

# 2. Instalirajte ga — otvoriće se tekstualni meni
sudo dpkg -i mysql-apt-config_0.8.xx-1_all.deb
```

U meniju birate koju seriju želite (npr. `mysql-8.4-lts`) i potvrđujete sa **OK**. Zatim:

```bash
sudo apt update
sudo apt install mysql-server
```

Ova instalacija **hoće** da vas pita za root lozinku i za način enkripcije lozinki (`caching_sha2_password` vs `mysql_native_password`). Za nove sisteme birajte `caching_sha2_password`; stariji klijenti i drajveri ponekad zahtevaju `mysql_native_password`, ali to je danas retko i predstavlja korak unazad u bezbednosti.

Ako kasnije želite da promenite seriju:

```bash
sudo dpkg-reconfigure mysql-apt-config
sudo apt update
```

**Upozorenje:** ne prelazite iz Ubuntu paketa u Oracle paket (ili obrnuto) na produkciji bez punog backupa i testa na staging serveru. Nadogradnja formata podataka je jednosmerna — MySQL 8.4 neće otvoriti podatke natrag u 8.0.

### Šta je instalacija zapravo uradila

Ovo je deo koji većina tutorijala preskoči, a vama je najkorisniji. Nakon `apt install mysql-server` na sistemu se pojavilo sledeće:

**Sistemski korisnik i grupa**

```bash
id mysql
```
```
uid=114(mysql) gid=120(mysql) groups=120(mysql)
```

Sistemski nalog bez shell-a i bez lozinke. `mysqld` startuje kao root, odmah odbacuje privilegije i nastavlja kao `mysql`. Ovo je razlog zbog kojeg svaki direktorijum sa podacima mora biti u vlasništvu ovog korisnika — vratićemo se na to u Modulu 2 kada budemo premeštali `datadir`.

**Direktorijum sa podacima**

```bash
sudo ls -la /var/lib/mysql
```

Ovde su svi vaši podaci. `ibdata1`, `ib_logfile*` (ili `#innodb_redo/` u novijim verzijama), `mysql.ibd`, poddirektorijumi po bazama sa `.ibd` fajlovima. Detaljno u Modulu 2.

Zapamtite odmah: **ovaj direktorijum se ne kopira dok MySQL radi.** `cp -r /var/lib/mysql /backup/` na živom serveru daje nekonzistentnu kopiju iz koje se često ne može oporaviti.

**Konfiguracija**

```bash
ls -la /etc/mysql/
```

```
my.cnf -> /etc/alternatives/my.cnf
conf.d/
mysql.conf.d/
debian.cnf
```

`/etc/mysql/my.cnf` na Ubuntuu je simbolički link koji vodi (kroz `alternatives` sistem) do `/etc/mysql/mysql.cnf`. Taj fajl gotovo ništa ne sadrži osim direktiva za uključivanje:

```ini
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/
```

Prava konfiguracija servera je u `/etc/mysql/mysql.conf.d/mysqld.cnf`. **Vaše izmene ne idu tamo** — one idu u zaseban fajl u `/etc/mysql/mysql.conf.d/`, npr. `99-custom.cnf`, da vam nadogradnja paketa ne bi prepisala izmene. Ceo redosled učitavanja i pravila prioriteta obrađujemo u Modulu 2.

**Systemd servis**

```bash
systemctl status mysql
systemctl is-enabled mysql
```

Na Ubuntuu se servis zove `mysql`, ne `mysqld`. (Na RHEL-u je `mysqld`, na MariaDB sistemima `mariadb` sa `mysql` kao aliasom. Ovo je čest izvor konfuzije kad prelazite između distribucija.)

**Logovi**

```bash
sudo ls -la /var/log/mysql/
```

Error log je obično `/var/log/mysql/error.log`. Ne pretpostavljajte — proverite:

```bash
mysql -e "SELECT @@log_error;"
```

Ako je vrednost `stderr`, log ide u journald i čitate ga sa `journalctl -u mysql`.

**Socket i PID**

```bash
ls -la /run/mysqld/
```
```
mysqld.sock
mysqld.pid
mysqld.sock.lock
```

**AppArmor profil**

```bash
sudo aa-status | grep mysql
```

Ubuntu isporučuje AppArmor profil za `mysqld`. On ograničava kojim fajlovima proces sme da pristupi. Ovo je najčešći uzrok misterioznih grešaka pri premeštanju `datadir`-a ili pisanju backupa na nestandardnu lokaciju. Detaljno u Modulu 2.

**Održavateljski nalog**

```bash
sudo cat /etc/mysql/debian.cnf
```

Ubuntu paket kreira MySQL nalog `debian-sys-maint` sa nasumičnom lozinkom, koji koriste sistemske skripte (npr. `logrotate`). Ne brišite ga i ne menjajte mu lozinku ručno.

### Prva provera da sve radi

```bash
systemctl status mysql
sudo mysqladmin status
sudo mysqladmin version
```

`mysqladmin status` daje jednu liniju sa uptime-om, brojem niti i upita. Ako je dobijete, server radi i prihvata konekcije.

---

## 4. `mysql_secure_installation` — šta tačno radi i zašto odmah

Sledeći korak je skripta koja dolazi uz paket:

```bash
sudo mysql_secure_installation
```

Većina uputstava kaže "pokrenite ovo i odgovorite sa `y` na sve". To je loš savet, jer neki od tih odgovora imaju posledice koje treba razumeti. Idemo redom kroz pitanja.

### Pitanje 1: VALIDATE PASSWORD komponenta

```
Would you like to setup VALIDATE PASSWORD component?
```

Ova komponenta nameće pravila složenosti lozinki za sve MySQL naloge. Nudi tri nivoa:

- **LOW** — najmanje 8 znakova,
- **MEDIUM** — 8 znakova, brojevi, mala i velika slova, specijalni znak,
- **STRONG** — sve to plus provera da lozinka nije u rečniku.

Realan savet: **na produkciji da, na razvojnom serveru možda ne.** Razlog za oprez je što će komponenta odbiti i lozinke koje generišete alatom ako slučajno ne zadovolje pravila, i to može zbuniti automatizaciju (Ansible, skripte). Ako koristite generator lozinki od 32 znaka — a treba da koristite — komponenta vam ne donosi mnogo.

Možete je uključiti i kasnije:

```sql
INSTALL COMPONENT 'file://component_validate_password';
```

Provera trenutnog stanja:

```sql
SHOW VARIABLES LIKE 'validate_password%';
```

### Pitanje 2: Lozinka za root nalog

```
Please set the password for root here.
```

Ovde nastaje najveća zabuna na Ubuntuu. Ako je paket iz Ubuntu repozitorijuma, `root@localhost` koristi `auth_socket` autentikaciju — što znači da **lozinka nije ni relevantna**. Skripta može da postavi lozinku, ali dok je `auth_socket` aktivan, prijava se svejedno vrši preko Unix socket-a i identiteta OS korisnika.

Preporuka: **ostavite `auth_socket` kakav jeste.** To je bezbednije od lozinke. Detalje o tome zašto — u sledećem poglavlju.

Ako ste instalirali iz Oracle repozitorijuma, root već ima lozinku koju ste zadali tokom instalacije i ovo pitanje ima smisla.

### Pitanje 3: Uklanjanje anonimnih korisnika

```
Remove anonymous users? (Press y|Y for Yes)
```

**Uvek `y`.** Anonimni korisnici su nasleđe iz 1990-ih koje dozvoljava povezivanje bez korisničkog imena. Nemaju nikakvu upotrebu na modernom serveru.

Provera šta ćete ukloniti:

```sql
SELECT user, host FROM mysql.user WHERE user = '';
```

### Pitanje 4: Zabrana udaljene prijave za root

```
Disallow root login remotely? (Press y|Y for Yes)
```

**Uvek `y`.** Ovo briše naloge tipa `root@'%'`. Root nikad ne treba da se povezuje sa druge mašine. Ako vam treba administrativni pristup sa udaljene lokacije, rešenje je SSH tunel (Modul 4) ili zaseban admin nalog sa jasno određenim izvornim hostom — nikako root sa bilo koje adrese.

### Pitanje 5: Uklanjanje `test` baze

```
Remove test database and access to it? (Press y|Y for Yes)
```

**`y`.** Baza `test` istorijski dolazi sa privilegijama koje svaki korisnik može da koristi. Nema razloga da ostane.

### Pitanje 6: Ponovno učitavanje tabela privilegija

```
Reload privilege tables now? (Press y|Y for Yes)
```

**`y`.** Ekvivalent `FLUSH PRIVILEGES;` — primenjuje izmene odmah.

### Šta skripta NIJE uradila

Ovo je važno, jer mnogi misle da su nakon ove skripte "obezbedili bazu". Nisu. Skripta nije:

- promenila `bind-address` (Modul 4),
- podesila firewall (Modul 4),
- uključila TLS (Modul 4),
- podesila backup (Modul 5),
- ograničila privilegije aplikacionih naloga (Modul 3),
- podesila rotaciju logova ili monitoring (Modul 6).

`mysql_secure_installation` uklanja podrazumevane slabosti iz devedesetih. Ostatak bezbednosti je vaš posao i predmet je narednih modula.

---

## 5. Prvo povezivanje: klijent, socket, TCP i `auth_socket`

Server radi. Vreme je da se povežemo — i da razumemo šta se pri tome dešava.

### Prva konekcija

```bash
sudo mysql
```

Dobijate prompt:

```
mysql>
```

Bez lozinke. Ovo mnoge iznenadi i deluje kao propust u bezbednosti. Nije — to je `auth_socket` plugin na delu.

### Kako radi `auth_socket`

Kada se povezujete preko Unix domain socket-a (a ne preko TCP-a), kernel može da kaže serveru **koji OS korisnik** je otvorio konekciju. `auth_socket` plugin koristi tu informaciju: ako je vaše Linux korisničko ime `root`, možete se prijaviti kao MySQL korisnik `root@localhost` bez lozinke. Ako niste `root`, ne možete — bez obzira na to koju lozinku ukucate.

Proverite trenutno stanje:

```sql
SELECT user, host, plugin FROM mysql.user;
```

```
+------------------+-----------+-----------------------+
| user             | host      | plugin                |
+------------------+-----------+-----------------------+
| debian-sys-maint | localhost | caching_sha2_password |
| mysql.infoschema | localhost | caching_sha2_password |
| mysql.session    | localhost | caching_sha2_password |
| mysql.sys        | localhost | caching_sha2_password |
| root             | localhost | auth_socket           |
+------------------+-----------+-----------------------+
```

Nalozi `mysql.infoschema`, `mysql.session` i `mysql.sys` su interni sistemski nalozi. Zaključani su i ne diraju se.

**Zašto je ovo dobro:** ne postoji lozinka koju neko može da pogodi, ukrade iz skripte ili nađe u istoriji shell-a. Pristup rootu je vezan za pristup rootu na samom serveru — a ako neko već ima root na mašini, MySQL lozinka je najmanji od vaših problema.

**Kada ovo smeta:** ako neki alat zahteva prijavu sa lozinkom kao root preko TCP-a. Rešenje nije da menjate root, nego da napravite **zaseban administrativni nalog** — o tome za koji trenutak.

### Socket ili TCP — koja je razlika

MySQL na Linuxu prihvata konekcije na dva načina i to je izvor mnogih "ali ja sam se maločas povezao" situacija.

**Unix socket** (`/run/mysqld/mysqld.sock`) — fajl u fajl sistemu, bez mreže. Brži je, ne prolazi kroz mrežni stek, i omogućava `auth_socket` autentikaciju. Koristi se **kad je host `localhost`**.

**TCP** (podrazumevano port 3306) — obična mrežna konekcija. Koristi se za sve ostalo, a lokalno **kad je host `127.0.0.1`**.

Ključna zamka:

```bash
mysql -h localhost -u aplikacija -p    # ide preko socket-a
mysql -h 127.0.0.1 -u aplikacija -p    # ide preko TCP-a
```

Za MySQL klijent, `localhost` je **specijalna vrednost** koja znači "koristi socket", a ne "poveži se na 127.0.0.1". Ovo je razlog zbog kojeg ista aplikacija radi sa jednim konfiguracionim fajlom, a ne radi sa drugim, iako "pišu isto".

Ako želite TCP eksplicitno i pored `localhost`:

```bash
mysql -h localhost --protocol=TCP -u korisnik -p
```

Ova razlika ima i posledicu na privilegije: `korisnik@localhost` i `korisnik@127.0.0.1` su **dva različita MySQL korisnika**. Detaljno u Modulu 3.

Provera gde je socket i na čemu server sluša:

```bash
mysql -e "SELECT @@socket, @@port, @@bind_address;"
ss -lntp | grep 3306
ls -la /run/mysqld/mysqld.sock
```

### Kreiranje sopstvenog administrativnog naloga

Raditi sve kao `root` preko `sudo mysql` je u redu za učenje, ali u produkciji je bolje imati imenovani nalog — zbog revizije i zbog toga što alati za backup i monitoring ne treba da rade kao root.

```sql
CREATE USER 'admin'@'localhost' IDENTIFIED BY 'ovde-duga-nasumicna-lozinka';
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

Napomena: `WITH GRANT OPTION` daje pravo da taj nalog dodeljuje privilegije drugima. Za administratora je to u redu; za aplikaciju nikako. Princip najmanjih privilegija razrađujemo u Modulu 3.

Da ne biste kucali lozinku svaki put — i, još važnije, da se ne bi pojavljivala u `ps` izlazu i u istoriji shell-a — napravite lični konfiguracioni fajl:

```bash
cat > ~/.my.cnf <<'EOF'
[client]
user=admin
password=ovde-duga-nasumicna-lozinka
EOF

chmod 600 ~/.my.cnf
```

Sada je dovoljno:

```bash
mysql
```

**Nikad ne pišite lozinku direktno u komandnoj liniji** (`mysql -u admin -pTajna`). Ona ostaje u `~/.bash_history` i vidljiva je u `ps aux` svima na sistemu dok komanda traje. Bezbednim rukovanjem lozinkama u skriptama bavimo se u Modulu 11.

### Osnovno snalaženje u klijentu

Minimum koji vam treba da se orijentišete na nepoznatom serveru:

```sql
SHOW DATABASES;                 -- koje baze postoje
SELECT @@datadir;               -- gde su fajlovi
SELECT @@version;               -- verzija servera
SELECT USER(), CURRENT_USER();  -- kao ko sam se povezao / kao ko sam autentifikovan
STATUS;                         -- pregled konekcije: protokol, socket, uptime
SHOW PROCESSLIST;               -- ko je trenutno povezan i šta radi
```

Razlika između `USER()` i `CURRENT_USER()` je korisna u dijagnostici: prvo je ono što ste tražili, drugo je nalog na koji vas je server zaista mapirao.

Korisne opcije klijenta:

```bash
mysql -e "SHOW DATABASES;"           # jedna komanda, bez interaktivne sesije
mysql --table -e "..."               # tabelarni ispis i u neinteraktivnom režimu
mysql -N -B -e "..."                 # bez zaglavlja, tab-separated — za skripte
mysql --safe-updates                 # odbija UPDATE/DELETE bez WHERE
```

Ova poslednja opcija zaslužuje pažnju. `--safe-updates` (poznata i kao `-U`) sprečava `UPDATE` i `DELETE` bez `WHERE` klauzule. Ako ćete ikada raditi ručne izmene na produkciji, stavite je u `~/.my.cnf` pod sekciju `[mysql]`. Jednom kada spasi situaciju, isplatila se.

```ini
[mysql]
safe-updates
```

### Izlazak

```sql
exit;
```
ili `\q`, ili `Ctrl+D`.

---

## Kontrolna lista na kraju modula

Nakon ovog modula, na vašem serveru treba da važi sledeće. Proverite svaku stavku — ovo je stanje od kog kreće Modul 2.

```bash
# 1. Servis radi i uključen je za startovanje pri butu
systemctl is-active mysql
systemctl is-enabled mysql

# 2. Znate koji proizvod i koju verziju imate
mysql -e "SELECT VERSION(), @@version_comment;"

# 3. Znate odakle je paket došao
apt policy mysql-server

# 4. Znate gde su ključni fajlovi
mysql -e "SELECT @@datadir, @@socket, @@port, @@log_error;"

# 5. Server sluša samo na localhost (podrazumevano na Ubuntuu)
mysql -e "SELECT @@bind_address;"
ss -lntp | grep 3306

# 6. Nema anonimnih korisnika, nema udaljenog roota
mysql -e "SELECT user, host, plugin FROM mysql.user;"

# 7. Nema test baze
mysql -e "SHOW DATABASES;"

# 8. Imate imenovani administrativni nalog i ~/.my.cnf sa pravima 600
ls -l ~/.my.cnf

# 9. Error log postoji i možete ga pročitati
sudo tail -20 /var/log/mysql/error.log
```

---

## Vežbe

**Vežba 1 — Identifikacija nasleđenog sistema**
Na virtuelnoj mašini instalirajte MySQL. Zatim, pretvarajući se da je server nasleđen, utvrdite isključivo komandama: koji proizvod, koja verzija, iz kog repozitorijuma, gde su podaci, gde je error log, koliko dugo servis radi i koliko diska zauzima `datadir`.

**Vežba 2 — Razlika socket/TCP**
Kreirajte korisnika `test1@localhost` sa lozinkom. Pokušajte prijavu sa `-h localhost` i sa `-h 127.0.0.1`. Objasnite rezultat. Zatim kreirajte i `test1@127.0.0.1` i ponovite. Zapišite šta ste zaključili o tome kako MySQL bira nalog.

**Vežba 3 — `auth_socket` u praksi**
Kao neprivilegovan korisnik pokušajte `mysql -u root`. Objasnite grešku. Zatim proverite `plugin` kolonu u `mysql.user` i objasnite vezu.

**Vežba 4 — Vlastiti admin nalog**
Napravite nalog `sysadmin@localhost`, konfigurišite `~/.my.cnf` sa pravima 600, uključite `safe-updates` i potvrdite da radi tako što ćete pokušati `UPDATE` bez `WHERE` na test tabeli.

**Vežba 5 — Čitanje starta servisa**
Restartujte MySQL i odmah pročitajte error log. Pronađite liniju koja označava da je server spreman za konekcije i izmerite koliko je sekundi prošlo od početka starta. Ova veština je osnova troubleshootinga u Modulu 10.

---

## Šta sledi

U **Modulu 2** rastavljamo MySQL na sastavne delove sa stanovišta fajl sistema: kako radi systemd unit i šta znači kad `restart` visi, kako je organizovana konfiguracija i kojim redosledom se učitava, šta je tačno u `/var/lib/mysql` i kako se `datadir` premešta na drugi disk a da vas AppArmor ne blokira.

To je modul koji od vas pravi čoveka koji zna gde da traži kad nešto pođe naopako.
