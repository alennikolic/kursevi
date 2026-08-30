# Modul 4: Mreža i pristup spolja

*MySQL za Linux administratore*

---

## Uvod: šta zapravo branimo

U prethodnom modulu smo videli prvi sloj odbrane — host polje u nalogu. Ovaj modul dodaje ostale slojeve, i to redom od najgrubljeg ka najfinijem:

1. **Da li server uopšte sluša na mreži?** (`bind-address`)
2. **Ko sme da dođe do porta?** (firewall)
3. **Kojim kanalom putuje saobraćaj?** (SSH tunel, VPN)
4. **Da li je saobraćaj šifrovan i da li znamo sa kim pričamo?** (TLS)
5. **Sa koje adrese nalog sme da se prijavi?** (host polje — Modul 3)
6. **Šta taj nalog sme da uradi?** (privilegije — Modul 3)

Šest slojeva. Nijedan nije dovoljan sam.

Pre nego što krenemo, vredi da bude jasno protiv čega se štitimo, jer to određuje koliko truda gde ulažete.

**Automatizovano skeniranje.** Ovo nije teorija. Server sa otvorenim portom 3306 na javnoj IP adresi biva pronađen za nekoliko minuta, ne dana. Boti neprekidno skeniraju ceo IPv4 prostor i probaju uobičajene lozinke. Odbrana: firewall i `bind-address`.

**Presretanje saobraćaja.** MySQL protokol bez TLS-a prenosi upite i rezultate u čitljivom obliku. Ko god vidi saobraćaj — vidi vaše podatke. Odbrana: TLS ili tunel.

**Napadač koji je već ušao na jedan server u mreži.** Ovo je scenario koji se najčešće potceni. Ako je kompromitovan aplikacioni server, napadač je sada *unutar* vaše mreže. Zato "interna mreža je bezbedna" nije stav koji drži vodu. Odbrana: TLS i unutar LAN-a, uz restriktivne host vrednosti u nalozima.

**Slučajna izloženost.** Neko je "privremeno" otvorio port da testira i zaboravio. Odbrana: revizija i monitoring, ne dobra volja.

---

## 15. `bind-address` — da li server uopšte sluša

### Trenutno stanje

```sql
SELECT @@bind_address, @@port, @@mysqlx_bind_address, @@mysqlx_port;
```

```bash
sudo ss -lntp | grep -E '3306|33060'
```

Tipičan izlaz na Ubuntu instalaciji:

```
LISTEN 0 151 127.0.0.1:3306  0.0.0.0:* users:(("mysqld",pid=1234,fd=22))
LISTEN 0  70 127.0.0.1:33060 0.0.0.0:* users:(("mysqld",pid=1234,fd=25))
```

Dva porta, ne jedan. O drugom za trenutak.

### Vrednosti i njihovo značenje

| Vrednost | Ko se može povezati |
|---|---|
| `127.0.0.1` | samo procesi na istoj mašini, preko IPv4 |
| `::1` | samo lokalno, preko IPv6 |
| `10.0.1.10` | samo preko tog konkretnog interfejsa |
| `10.0.1.10,127.0.0.1` | preko oba (MySQL 8.0.13+) |
| `0.0.0.0` | svi IPv4 interfejsi |
| `::` | svi IPv6 interfejsi (i IPv4 ako je dual-stack) |
| `*` | svi interfejsi, IPv4 i IPv6 |

**Podrazumevana vrednost razlikuje se po paketu.** Ubuntu paket postavlja `127.0.0.1` — server nije dostupan spolja dok to ne promenite. Oracle paket podrazumevano sluša na svim interfejsima. Ako ste instalirali iz Oracle repozitorijuma, ovo proverite odmah.

### Kada šta koristiti

**`127.0.0.1` — aplikacija i baza na istom serveru.**

Ovo je najbolji slučaj i mnogo je češći nego što ljudi misle. Klasičan LAMP stek, WordPress, jedan VPS. Ako je tako kod vas, ostavite kako jeste i pređite na sledeće poglavlje — nemate mrežni problem.

Možete ići i korak dalje:

```ini
[mysqld]
skip_networking = ON
```

Ovim potpuno gasite TCP. Server prihvata konekcije **isključivo** preko Unix socket-a. Nijedan port se ne otvara, pa ne postoji ni mogućnost mrežnog napada na bazu.

Cena: aplikacija se mora povezivati preko `localhost` (socket), ne `127.0.0.1`. Podsetnik iz Modula 1 — to su za MySQL dve različite stvari. Neki drajveri i ORM-ovi ovo ne podržavaju najbolje, pa testirajte pre nego što uključite na produkciji.

**Konkretna adresa interfejsa — aplikacija na drugom serveru.**

```ini
[mysqld]
bind-address = 10.0.1.10
```

Server sluša samo na internom interfejsu. Ako mašina ima i javnu adresu, baza na njoj **nije dostupna** — bez obzira na firewall. To je jak, jednostavan sloj odbrane.

Ako vam treba i lokalni pristup preko TCP-a:

```ini
bind-address = 10.0.1.10,127.0.0.1
```

**`0.0.0.0` ili `*` — izbegavajte kad god možete.**

Postoji legitiman slučaj: server sa dinamičkim adresama, kontejnerska okruženja, orkestratori. Ako morate, tada firewall postaje **jedini** mrežni sloj odbrane i mora biti besprekoran.

### Port 33060 — X Protocol

Ovo je deo koji se redovno previdi.

MySQL 8 podrazumevano učitava X Plugin, koji sluša na portu **33060** i implementira poseban protokol (za MySQL Shell, dokumentni model, neke novije konektore).

Praktično: **ako ste otvorili firewall za 3306 a zaboravili 33060, možda ste ostavili otvoren drugi ulaz u istu bazu.** Ili obrnuto — otvorili ste 33060 misleći da nije bitan.

Ako ga ne koristite, isključite ga:

```ini
[mysqld]
mysqlx = 0
```

Provera nakon restarta:

```bash
sudo ss -lntp | grep 33060
```

Ako ga koristite, tretirajte ga potpuno isto kao 3306 — isti `bind-address` pristup, isto pravilo na firewall-u, isti TLS zahtev.

### Promena porta

```ini
[mysqld]
port = 3307
```

Da odmah raščistimo: **ovo nije bezbednosna mera.** Ko god skenira portove naći će bazu i na 3307. Ne štiti od usmerenog napada.

Ono što promena porta stvarno donosi je manje šuma. Automatizovani boti masovno gađaju 3306; na drugom portu error log neće biti pun neuspelih pokušaja prijave, pa je lakše uočiti pravi problem. To je operativna korist, ne bezbednosna.

Cena je što svaki klijent, svaki backup skript i svaki monitoring alat mora znati za nov port. Ako ćete to raditi, uradite dosledno i zabeležite.

### Restart je obavezan

`bind_address` je statička promenljiva. Nema `SET GLOBAL`, nema `SET PERSIST` sa trenutnim dejstvom. Menja se u fajlu i traži restart.

```bash
sudo nano /etc/mysql/mysql.conf.d/99-custom.cnf
sudo mysqld --validate-config
sudo systemctl restart mysql
sudo ss -lntp | grep mysqld
```

Poslednja komanda je potvrda. Nemojte pretpostaviti da je promena primenjena — pogledajte na čemu proces zaista sluša.

---

## 16. Firewall

### Osnovno pravilo

**Port 3306 nikada ne ide na internet.** Bez izuzetka koji vredi pominjati.

Ako mislite da vam treba, ono što vam zapravo treba je VPN ili SSH tunel — o oba u sledećem poglavlju.

### UFW u praksi

Ubuntu isporučuje UFW kao nadgradnju nad `nftables`/`iptables`. Za većinu servera je sasvim dovoljan.

```bash
sudo ufw status verbose
```

Ako je neaktivan, pažljivo pre uključivanja:

```bash
# PRVO ovo — inače gubite SSH pristup serveru
sudo ufw allow 22/tcp

sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
```

Redosled nije stilsko pitanje. `ufw enable` sa `default deny incoming` i bez dozvoljenog SSH-a zatvara vam vrata dok ste unutra.

### Pravila za MySQL

**Samo iz podmreže aplikacionih servera:**

```bash
sudo ufw allow from 10.0.1.0/24 to any port 3306 proto tcp
```

**Samo sa konkretnih adresa — bolje kada ih znate:**

```bash
sudo ufw allow from 10.0.1.50 to any port 3306 proto tcp comment 'app01'
sudo ufw allow from 10.0.1.51 to any port 3306 proto tcp comment 'app02'
sudo ufw allow from 10.0.1.60 to any port 3306 proto tcp comment 'backup server'
```

Koristite `comment`. Za godinu dana nećete znati čija je adresa `10.0.1.60`, a `ufw status` će vam reći.

**Ako koristite X Protocol:**

```bash
sudo ufw allow from 10.0.1.0/24 to any port 33060 proto tcp comment 'mysqlx'
```

**Provera:**

```bash
sudo ufw status numbered
```

```
[ 1] 22/tcp                     ALLOW IN    Anywhere
[ 2] 3306/tcp                   ALLOW IN    10.0.1.50    # app01
[ 3] 3306/tcp                   ALLOW IN    10.0.1.51    # app02
```

Brisanje po broju:

```bash
sudo ufw delete 3
```

Napomena o redosledu: UFW primenjuje pravila **redom** i prvo poklapanje pobeđuje. Ako imate opšte `deny` pravilo iznad specifičnog `allow`, ovo drugo nikada neće doći na red. Umetanje na određeno mesto:

```bash
sudo ufw insert 1 allow from 10.0.1.50 to any port 3306 proto tcp
```

### Docker zaobilazi UFW — pročitajte ovo

Ovo je zamka koja je izložila veliki broj baza i zaslužuje poseban odeljak.

**Docker upisuje sopstvena `iptables` pravila u lanac `DOCKER`, koji se obrađuje pre UFW lanaca.** Posledica: ako pokrenete kontejner sa

```bash
docker run -p 3306:3306 mysql:8.0
```

port 3306 je **otvoren ka svetu**, iako `ufw status` prikazuje `deny incoming`. UFW pravila jednostavno nisu na putu tog saobraćaja.

Provera stvarnog stanja, uvek sa druge mašine:

```bash
sudo iptables -L DOCKER -n
sudo ss -lntp | grep 3306
```

Rešenje je da port vežete za lokalni interfejs:

```bash
docker run -p 127.0.0.1:3306:3306 mysql:8.0
```

Ili u `docker-compose.yml`:

```yaml
ports:
  - "127.0.0.1:3306:3306"
```

Ili da uopšte ne objavljujete port napolje, nego koristite Docker mrežu između kontejnera. O MySQL-u u kontejnerima detaljnije u Modulu 11.

### Firewall u cloudu

Ako je server kod cloud provajdera, imate **dva** firewall-a: onaj na samoj mašini (UFW) i onaj na nivou platforme (security group, cloud firewall, network ACL).

Koristite oba. Cloud pravila štite i kada je host kompromitovan; UFW štiti i kada neko pogreši u cloud konzoli. To su nezavisni slojevi i cena održavanja je mala.

Česta greška: podešen je samo cloud firewall, a na hostu je sve otvoreno. Onda neko doda mašinu u pogrešnu security grupu i baza je javna.

### Provera spolja — jedina koja se računa

Ne verujte `ufw status`. Testirajte sa druge mašine:

```bash
# sa udaljene mašine
nc -zv <ip-baze> 3306
nmap -Pn -p 3306,33060 <ip-baze>
```

Očekivan rezultat sa mašine koja **ne sme** da pristupi:

```
Connection to <ip> 3306 port [tcp/mysql] failed: Connection timed out
```

`timed out` je bolje od `connection refused` — znači da paketi bivaju odbačeni bez odgovora, pa skener ne dobija ni potvrdu da tu nešto postoji.

Ovu proveru uvrstite u rutinu nakon svake promene mrežne konfiguracije.

### `fail2ban` — skromna, ali korisna dopuna

Ako iz nekog razloga MySQL ipak mora biti dostupan sa šireg opsega, `fail2ban` može da blokira adrese sa kojih se ponavljaju neuspele prijave.

```bash
sudo apt install fail2ban
```

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[mysqld-auth]
enabled  = true
port     = 3306
logpath  = /var/log/mysql/error.log
maxretry = 5
bantime  = 3600
findtime = 600
```

Da bi ovo radilo, neuspele prijave moraju stvarno pisati u error log. Proverite:

```bash
sudo grep -i "access denied" /var/log/mysql/error.log | tail
```

Budite realni oko dometa: `fail2ban` je korisna dopuna, ali nije zamena za firewall. Ako vam je `fail2ban` glavna odbrana MySQL-a, arhitektura je pogrešna.

Podsetnik iz Modula 3: ugrađeni `FAILED_LOGIN_ATTEMPTS` na nivou naloga radi sličan posao, na nivou naloga umesto na nivou IP adrese. Ta dva se lepo dopunjuju.

---

## 17. Bezbedan udaljeni pristup

Vama, kao administratoru, s vremena na vreme treba pristup bazi sa laptopa. Rešenje **nije** otvaranje porta.

### SSH tunel — osnova

Ideja: SSH veza do servera, kroz koju se provuče TCP saobraćaj do MySQL-a. Baza ostaje vezana za `127.0.0.1`, firewall ostaje zatvoren, a vi ipak radite sa laptopa.

```bash
ssh -L 3307:127.0.0.1:3306 vas_nalog@db-server
```

Čitanje ove komande: *otvori na mom lokalnom portu 3307 tunel do porta 3306 na adresi 127.0.0.1, gledano iz perspektive db-servera.*

Dok veza traje, u drugom terminalu:

```bash
mysql -h 127.0.0.1 -P 3307 -u admin -p
```

**Obavezno `-h 127.0.0.1`, nikako `-h localhost`.** Ovo je najčešća greška pri radu sa tunelom. `localhost` tera klijent na Unix socket, a socket ne prolazi kroz tunel — dobijate grešku o socket fajlu koji ne postoji i pomislite da tunel ne radi. Radi; samo mu se niste obratili.

Ovo je isti koncept iz Modula 1, sada u praktičnoj primeni.

### Tunel u pozadini

```bash
ssh -f -N -L 3307:127.0.0.1:3306 vas_nalog@db-server
```

- `-N` — ne izvršavaj komandu, samo prosleđuj portove.
- `-f` — idi u pozadinu.

Zatvaranje:

```bash
pkill -f "ssh -f -N -L 3307"
```

### Trajno rešenje u `~/.ssh/config`

Da ne biste pamtili komandu:

```
Host db-tunel
    HostName db-server.interno.local
    User vas_nalog
    LocalForward 3307 127.0.0.1:3306
    ServerAliveInterval 30
    ServerAliveCountMax 3
```

```bash
ssh -N db-tunel
```

`ServerAliveInterval` sprečava da NAT ili firewall tiho ubiju neaktivnu vezu, što je najčešći uzrok tunela koji "puca posle par minuta".

### Stabilan tunel: `autossh`

Za tunele koji treba da rade neprekidno (npr. monitoring ili backup sa udaljene lokacije):

```bash
sudo apt install autossh

autossh -M 0 -f -N \
  -o "ServerAliveInterval 30" \
  -o "ServerAliveCountMax 3" \
  -o "ExitOnForwardFailure yes" \
  -L 3307:127.0.0.1:3306 vas_nalog@db-server
```

`autossh` sam podiže vezu kada padne. Kao systemd servis:

```bash
sudo nano /etc/systemd/system/mysql-tunel.service
```

```ini
[Unit]
Description=SSH tunel do MySQL baze
After=network-online.target
Wants=network-online.target

[Service]
User=tunel
Environment="AUTOSSH_GATETIME=0"
ExecStart=/usr/bin/autossh -M 0 -N \
  -o "ServerAliveInterval 30" \
  -o "ServerAliveCountMax 3" \
  -o "ExitOnForwardFailure yes" \
  -o "StrictHostKeyChecking yes" \
  -L 3307:127.0.0.1:3306 vas_nalog@db-server
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mysql-tunel
```

### SSH nalog koji sme samo tunel

Ako nekome treba pristup bazi, ne mora da dobije shell na serveru. U `~/.ssh/authorized_keys` na db-serveru:

```
command="/bin/false",no-pty,no-agent-forwarding,no-X11-forwarding,permitopen="127.0.0.1:3306" ssh-ed25519 AAAAC3... analiticar@laptop
```

Šta ovo daje: nalog može da otvori tunel isključivo ka `127.0.0.1:3306` i ništa drugo. Nema shell, nema drugih portova, nema prosleđivanja agenta.

Ovo je odličan način da nekom analitičaru date pristup bazi bez pristupa serveru.

### GUI klijenti

DBeaver, MySQL Workbench, TablePlus, HeidiSQL i slični imaju **ugrađenu podršku za SSH tunel**. U podešavanjima veze postoji zaseban tab za SSH.

Recite ovo ljudima u timu koji traže da im otvorite port. U devet od deset slučajeva rešenje je pet polja u njihovom klijentu, a ne promena na firewall-u.

### Kada VPN umesto tunela

SSH tunel je odličan za pojedinačan, povremen pristup. VPN je bolji kada:

- pristup treba većem broju ljudi,
- pristup treba **aplikaciji**, a ne čoveku (tunel je krhak za produkcijski saobraćaj),
- povezujete više lokacija ili više servisa, ne samo bazu,
- želite da mrežni pristup bude trajno rešenje sa centralnim upravljanjem.

WireGuard je danas najjednostavniji izbor. Nije predmet ovog kursa, ali ono što jeste relevantno je kako se MySQL uklapa:

```ini
[mysqld]
bind-address = 10.8.0.1
```

Server sluša **samo** na VPN interfejsu. Bez VPN-a ne postoji mrežni put do baze — ni sa interne mreže, ni sa interneta. Uz to, host polja u nalozima koriste VPN opseg:

```sql
CREATE USER 'admin'@'10.8.0.%' IDENTIFIED BY '...';
```

Ovo je čista, jednostavna arhitektura i vredi je razmotriti pre nego što počnete da komplikujete firewall pravila.

---

## 18. TLS između aplikacije i baze

### Šta MySQL 8 već ima

Pri prvoj inicijalizaciji MySQL 8 sam generiše sertifikate:

```bash
sudo ls -la /var/lib/mysql/*.pem
```

```
ca-key.pem  ca.pem  client-cert.pem  client-key.pem
private_key.pem  public_key.pem  server-cert.pem  server-key.pem
```

Provera da je TLS aktivan:

```sql
SHOW VARIABLES LIKE 'ssl_%';
SHOW GLOBAL STATUS LIKE 'Ssl_%';
```

Provera za **trenutnu** konekciju:

```sql
SHOW SESSION STATUS LIKE 'Ssl_cipher';
\s
```

Ako `Ssl_cipher` ima vrednost, ova veza je šifrovana.

### Ključna zamka: šifrovano nije isto što i provereno

Ovo je najvažnija rečenica u poglavlju.

Podrazumevani režim klijenta je `--ssl-mode=PREFERRED`: ako server nudi TLS, koristi ga; ako ne nudi, nastavi bez njega. Klijent pritom **ne proverava sertifikat servera.**

Posledice su dve, obe ozbiljne:

1. **Nema autentikacije servera.** Napadač koji je u poziciji da preusmeri saobraćaj može se predstaviti kao vaš MySQL server, ponuditi sopstveni sertifikat, i klijent će ga prihvatiti. Saobraćaj je šifrovan — prema napadaču.
2. **Tiha degradacija.** Ako TLS iz bilo kog razloga otkaže, veza se uspostavlja u čistom obliku i niko ništa ne primećuje.

Automatski generisani sertifikati imaju CN vrednost tipa `MySQL_Server_8.0.x_Auto_Generated_Server_Certificate` i potpisani su sopstvenim CA-om koji nijedan klijent ne poznaje. Zato sa njima ne možete koristiti pravu verifikaciju.

Zaključak: **auto-generisani sertifikati su bolji nego ništa, ali nisu rešenje.** Za produkciju napravite svoj CA.

### Režimi klijenta

| Režim | Šifrovanje | Provera CA | Provera imena hosta |
|---|---|---|---|
| `DISABLED` | ne | — | — |
| `PREFERRED` (podrazumevano) | ako može | ne | ne |
| `REQUIRED` | obavezno | ne | ne |
| `VERIFY_CA` | obavezno | **da** | ne |
| `VERIFY_IDENTITY` | obavezno | **da** | **da** |

Za produkciju ciljajte `VERIFY_IDENTITY`. `VERIFY_CA` je prihvatljiv kompromis kada imate problem sa imenima hostova.

### Sopstveni CA i sertifikati

Radimo ručno preko `openssl`, jer je `mysql_ssl_rsa_setup` zastareo.

**Korak 1 — pripremite direktorijum**

```bash
sudo mkdir -p /etc/mysql/ssl
cd /etc/mysql/ssl
```

**Korak 2 — CA**

```bash
sudo openssl genrsa -out ca-key.pem 4096

sudo openssl req -new -x509 -nodes -days 3650 \
  -key ca-key.pem -out ca.pem \
  -subj "/C=RS/O=Moja Firma/CN=Moja Firma MySQL CA"
```

**Korak 3 — serverski ključ i zahtev**

```bash
sudo openssl genrsa -out server-key.pem 2048

sudo openssl req -new -key server-key.pem -out server-req.pem \
  -subj "/C=RS/O=Moja Firma/CN=db01.interno.local"
```

CN mora biti **ime pod kojim se klijenti povezuju.** Ako se aplikacija povezuje na `db01.interno.local`, to ide u CN.

**Korak 4 — SAN i potpisivanje**

Moderni klijenti proveravaju SAN, ne CN, pa ga moramo dodati:

```bash
sudo tee /etc/mysql/ssl/server-ext.cnf > /dev/null <<'EOF'
subjectAltName = DNS:db01.interno.local, DNS:db01, IP:10.0.1.10
extendedKeyUsage = serverAuth
EOF

sudo openssl x509 -req -days 825 \
  -in server-req.pem \
  -CA ca.pem -CAkey ca-key.pem -CAcreateserial \
  -extfile server-ext.cnf \
  -out server-cert.pem
```

Navedite sva imena i adrese pod kojima se serveru pristupa. Ako fali ono koje aplikacija koristi, `VERIFY_IDENTITY` će pucati.

**Korak 5 — klijentski sertifikat (opciono, za `REQUIRE X509`)**

```bash
sudo openssl genrsa -out client-key.pem 2048

sudo openssl req -new -key client-key.pem -out client-req.pem \
  -subj "/C=RS/O=Moja Firma/CN=app01"

sudo tee /etc/mysql/ssl/client-ext.cnf > /dev/null <<'EOF'
extendedKeyUsage = clientAuth
EOF

sudo openssl x509 -req -days 825 \
  -in client-req.pem \
  -CA ca.pem -CAkey ca-key.pem -CAcreateserial \
  -extfile client-ext.cnf \
  -out client-cert.pem
```

**Korak 6 — prava**

```bash
sudo chown -R mysql:mysql /etc/mysql/ssl
sudo chmod 700 /etc/mysql/ssl
sudo chmod 600 /etc/mysql/ssl/*-key.pem
sudo chmod 644 /etc/mysql/ssl/ca.pem /etc/mysql/ssl/server-cert.pem
```

Privatni ključevi moraju biti 600 i u vlasništvu korisnika `mysql`. Ako nisu, MySQL odbija da ih učita — i to prijavljuje u error logu, ne pri validaciji konfiguracije.

**Korak 7 — AppArmor**

Podsetnik iz Modula 2: `/etc/mysql/ssl/` je nova putanja i profil je možda ne pokriva.

```bash
sudo nano /etc/apparmor.d/local/usr.sbin.mysqld
```

```
/etc/mysql/ssl/ r,
/etc/mysql/ssl/** r,
```

```bash
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.mysqld
```

Ako preskočite ovaj korak, dobićete `Permission denied` na fajl sa savršeno ispravnim pravima — tačno simptom koji smo opisali u Modulu 2.

**Korak 8 — konfiguracija servera**

```ini
[mysqld]
ssl_ca   = /etc/mysql/ssl/ca.pem
ssl_cert = /etc/mysql/ssl/server-cert.pem
ssl_key  = /etc/mysql/ssl/server-key.pem

tls_version = TLSv1.2,TLSv1.3
```

```bash
sudo mysqld --validate-config
sudo systemctl restart mysql
```

**Korak 9 — provera**

```sql
SHOW VARIABLES LIKE 'ssl_%';
```

Sa klijenta:

```bash
mysql -h db01.interno.local -u admin -p \
  --ssl-mode=VERIFY_IDENTITY \
  --ssl-ca=/putanja/do/ca.pem
```

Zatim u klijentu:

```sql
\s
```

Tražite liniju sa `SSL:` i nazivom šifre. Ako je ima, veza je i šifrovana i verifikovana.

### Obnavljanje sertifikata bez restarta

MySQL 8 ima korisnu naredbu koju vredi zapamtiti:

```sql
ALTER INSTANCE RELOAD TLS;
```

Ponovo učitava sertifikate sa diska, bez restarta i bez prekidanja postojećih veza. Nove veze koriste nove sertifikate.

Ovo pretvara obnovu sertifikata iz planiranog ispada u rutinsku operaciju.

### Istek sertifikata — postavite alarm sada

Najčešći uzrok ispada vezanih za TLS je istekao sertifikat. Provera:

```bash
sudo openssl x509 -in /etc/mysql/ssl/server-cert.pem -noout -subject -dates
```

Iz same baze:

```sql
SHOW STATUS LIKE 'Ssl_server_not_after';
```

Ovo obavezno uključite u monitoring (Modul 6). Alarm na 30 dana pre isteka. Sertifikat koji istekne u subotu uveče je klasičan scenario.

### Prisila na serverskoj strani

Do sada je klijent birao da koristi TLS. Sada to postaje obavezno.

**Po nalogu:**

```sql
ALTER USER 'app_shop'@'10.0.1.%' REQUIRE SSL;

-- strože: obavezan validan klijentski sertifikat od našeg CA
ALTER USER 'admin'@'10.0.1.%' REQUIRE X509;

-- najstrože: tačno određen sertifikat
ALTER USER 'repl'@'10.0.1.%'
  REQUIRE SUBJECT '/C=RS/O=Moja Firma/CN=replika01';
```

**Globalno:**

```ini
[mysqld]
require_secure_transport = ON
```

Ovim server odbija svaku konekciju koja nije preko TLS-a ili preko Unix socket-a. Socket je izuzet jer ne napušta mašinu.

**Uključujte ovo tek nakon što ste proverili da svi klijenti rade sa TLS-om.** Redosled je: prvo prebacite klijente na `REQUIRED`/`VERIFY_CA`, potvrdite da rade, pa tek onda uključite prisilu na serveru. Obrnut redosled znači potpun ispad.

Bezbedan način provere ko se još povezuje bez TLS-a:

```sql
SELECT DISTINCT
       t.processlist_user  AS korisnik,
       t.processlist_host  AS host,
       sbt.variable_value  AS sifra
FROM performance_schema.threads t
LEFT JOIN performance_schema.status_by_thread sbt
       ON t.thread_id = sbt.thread_id
      AND sbt.variable_name = 'Ssl_cipher'
WHERE t.processlist_user IS NOT NULL;
```

Prazna vrednost u koloni `sifra` znači konekciju bez šifrovanja. Nađite čija je i sredite je pre nego što uključite `require_secure_transport`.

### Konfiguracija na strani aplikacije

Podešavanja se razlikuju po jeziku i drajveru, ali princip je isti — navesti CA i tražiti verifikaciju.

**PHP (PDO):**

```php
$pdo = new PDO($dsn, $user, $pass, [
    PDO::MYSQL_ATTR_SSL_CA => '/etc/ssl/mysql/ca.pem',
    PDO::MYSQL_ATTR_SSL_VERIFY_SERVER_CERT => true,
]);
```

**Python (mysql-connector):**

```python
conn = mysql.connector.connect(
    host='db01.interno.local',
    ssl_ca='/etc/ssl/mysql/ca.pem',
    ssl_verify_identity=True,
)
```

**Java (JDBC):**

```
jdbc:mysql://db01.interno.local:3306/shop?sslMode=VERIFY_IDENTITY&trustCertificateKeyStoreUrl=...
```

Vaš deo posla je da CA sertifikat dostavite na aplikacione servere i da developeru kažete koji parametar treba. Sve ostalo je njihova konfiguracija.

### Šta TLS košta

Realno, malo. Šifrovanje samog toka podataka je jeftino na modernim procesorima sa AES-NI. Trošak je u **uspostavljanju veze** — TLS handshake je nekoliko dodatnih milisekundi.

To znači da je uticaj primetan samo kod aplikacija koje otvaraju novu konekciju za svaki zahtev. Rešenje nije isključiti TLS, nego uvesti connection pooling — što je ionako dobra praksa iz drugih razloga.

TLS 1.3 dodatno skraćuje handshake, pa ako svi klijenti podržavaju samo njega, možete i to podesiti.

---

## Kontrolna lista na kraju modula

```bash
# 1. Server sluša samo tamo gde treba — i 3306 i 33060
sudo ss -lntp | grep -E '3306|33060'
mysql -e "SELECT @@bind_address, @@mysqlx_bind_address;"

# 2. Firewall je aktivan i pravila su komentarisana
sudo ufw status verbose

# 3. Provereno je SPOLJA, sa mašine koja ne sme da pristupi
nmap -Pn -p 3306,33060 <ip-baze>

# 4. Ako koristite Docker — port nije objavljen na 0.0.0.0
sudo iptables -L DOCKER -n

# 5. TLS je konfigurisan sa sopstvenim CA
mysql -e "SHOW VARIABLES LIKE 'ssl_%';"

# 6. Sertifikat ne ističe uskoro
sudo openssl x509 -in /etc/mysql/ssl/server-cert.pem -noout -enddate

# 7. Klijent može da verifikuje identitet servera
mysql -h db01.interno.local --ssl-mode=VERIFY_IDENTITY \
      --ssl-ca=/etc/mysql/ssl/ca.pem -u admin -p -e "\s"

# 8. Zna se ko se povezuje bez TLS-a
# (upit nad performance_schema iz poglavlja 18)

# 9. Administrativni pristup ide kroz tunel ili VPN, ne kroz otvoren port
# 10. Istek sertifikata je u monitoringu sa alarmom na 30 dana
```

---

## Vežbe

**Vežba 1 — `bind-address` u tri stanja**
Postavite redom `127.0.0.1`, adresu interfejsa i `0.0.0.0`. Posle svake promene proverite sa `ss` lokalno i sa `nmap` sa druge mašine. Zabeležite razliku između `refused` i `timed out` i objasnite šta svaki odgovor otkriva napadaču.

**Vežba 2 — Zaboravljeni port**
Ostavite `mysqlx` uključen, otvorite firewall samo za 3306 i pokušajte povezivanje na 33060 sa dozvoljene i sa nedozvoljene mašine. Zatim isključite X Plugin i ponovite. Ovo je vežba koja se pamti.

**Vežba 3 — Docker zamka**
Na test mašini uključite UFW sa `default deny incoming`, pa pokrenite MySQL kontejner sa `-p 3306:3306`. Sa druge mašine proverite dostupnost porta. Objasnite rezultat, pa popravite sa `-p 127.0.0.1:3306:3306` i proverite ponovo.

**Vežba 4 — SSH tunel i `localhost` zamka**
Otvorite tunel i pokušajte povezivanje sa `-h localhost`, pa sa `-h 127.0.0.1`. Objasnite obe poruke. Zatim napravite `~/.ssh/config` unos i systemd servis sa `autossh`, pa testirajte oporavak tako što ćete namerno prekinuti mrežu.

**Vežba 5 — SSH nalog samo za tunel**
Napravite nalog sa ograničenim `authorized_keys` unosom. Potvrdite da tunel radi, da shell **ne** radi i da prosleđivanje drugog porta (npr. 22) ne prolazi.

**Vežba 6 — TLS od nule**
Napravite sopstveni CA i sertifikate, konfigurišite server, i povežite se sa sva četiri režima: `DISABLED`, `REQUIRED`, `VERIFY_CA`, `VERIFY_IDENTITY`. Zatim namerno pogrešite SAN (izostavite ime hosta) i ponovite — objasnite koja dva režima i dalje rade i zašto je to problem.

**Vežba 7 — Obnova sertifikata bez ispada**
Napravite nov serverski sertifikat, zamenite fajlove i primenite ih sa `ALTER INSTANCE RELOAD TLS`. Potvrdite da postojeće veze nisu prekinute i da nove koriste nov sertifikat.

**Vežba 8 — Prisila na TLS, ispravnim redosledom**
Pronađite upitom sve konekcije bez TLS-a. Prebacite ih. Tek zatim uključite `require_secure_transport = ON`. Zatim namerno uradite obrnuto na test instanci i vidite šta se dešava kad prisilite server pre nego što ste sredili klijente.

---

## Šta sledi

Sa **Modulom 5** ulazimo u srce kursa: backup i restore.

To je modul u kome se meri da li ste dobar administrator baze ili ne, jer sve ostalo — konfiguracija, bezbednost, performanse — može da se ponovo podesi. Podaci ne mogu.

Obradićemo tri tipa backupa i kada koji koristiti, `mysqldump` sa opcijama koje su obavezne a podrazumevano isključene, `mydumper` kada dump traje predugo, Percona XtraBackup za fizički backup živog servera, snapshot pristup i njegova ograničenja, point-in-time recovery pomoću binlogova, i — najvažnije — proceduru restore-a koja je testirana i izmerena, a ne pretpostavljena.
