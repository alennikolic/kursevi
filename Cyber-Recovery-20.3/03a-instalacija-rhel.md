# DEO III-a — INSTALACIJA NA RED HAT ENTERPRISE LINUX

**Dell PowerProtect Cyber Recovery 20.3 — Uputstvo za instalaciju i integraciju**
Verzija dokumenta: 0.1 | Namena: System Engineer

> **Izvori:** *Dell PowerProtect Cyber Recovery 20.3 Installation and Upgrade Guide*, *Dell PowerProtect Cyber Recovery 20.3 Product Guide*.
> **Preduslov:** pre početka mora biti u potpunosti izvršen **Deo II — Preduslovi** (`02-Preduslovi.md`), uključujući popunjen radni list.
> **Napomena o obimu:** ovaj fajl pokriva instalaciju na **softverskom** deploymentu (RHEL; procedura je identična i za SLES uz razlike u imenima paketa). Za OVA deployment videti `03b-Instalacija-OVA.md`.
> **Zajednička poglavlja:** poglavlja **14** (prva prijava) i **15** (osnovna konfiguracija u CR UI) važe za oba načina instalacije i nalaze se u ovom fajlu.

---

## Sadržaj

- [10. Preuzimanje i priprema softvera](#10-preuzimanje-i-priprema-softvera)
- [11. Referenca komande crsetup.sh](#11-referenca-komande-crsetupsh)
- [12. Instalacija na Red Hat Enterprise Linux](#12-instalacija-na-red-hat-enterprise-linux)
- [13. Post-install koraci (RHEL)](#13-post-install-koraci-rhel)
- [14. Prva prijava i inicijalna konfiguracija — zajedničko](#14-prva-prijava-i-inicijalna-konfiguracija--zajedničko)
- [15. Osnovna konfiguracija vault-a u CR UI — zajedničko](#15-osnovna-konfiguracija-vault-a-u-cr-ui--zajedničko)
- [Dodatak: Post-install checklist za RHEL](#dodatak-post-install-checklist-za-rhel)

---

# 10. Preuzimanje i priprema softvera

## 10.1 Preuzimanje

Sa **Dell Online Support** preuzeti jedno od:

| Paket | Format | Namena |
|---|---|---|
| Cyber Recovery installation package | `.tar.gz` | Softverska instalacija na RHEL / SLES |
| Cyber Recovery virtual appliance file | `.ova` | Instalacija u VMware ESXi okruženju |
| Cyber Recovery virtual appliance update package | `.bin` | Nadogradnja OVA deploymenta |

---

## 10.2 Prenos paketa u vault

Vault je izolovano okruženje. Prenos instalacionog paketa mora se izvesti u skladu sa procedurom kupca za unos podataka u vault (npr. preko kontrolisanog jump hosta ili prenosivog medija odobrenog bezbednosnom politikom).

> Za nadogradnje postoji posebna procedura *Moving software and operating system update files to the Cyber Recovery vault* — videti Deo VI.

---

## 10.3 Raspakivanje

```bash
# 1. Prijaviti se na management host kao root
ssh root@<cr-management-host>

# 2. Preuzeti/kopirati paket u direktorijum sa najmanje 5 GB slobodnog prostora
cd /opt/staging-cr

# 3. Raspakovati instalacioni paket
tar -xzvf <ime_instalacionog_paketa>.tar.gz

# 4. Preći u staging direktorijum
cd staging
```

Raspakivanje kreira poddirektorijum `staging` u tekućem direktorijumu. Ekstrakcija uključuje i komandu `crsetup.sh`.

---

# 11. Referenca komande `crsetup.sh`

Komanda `crsetup.sh` služi za instalaciju, upravljanje, verifikaciju i uklanjanje Cyber Recovery softvera.

**Sintaksa:**

```bash
./crsetup.sh <opcija>
```

## 11.1 Tabela opcija

| Opcija | Kratka | Namena |
|---|---|---|
| `--addcustcert` | `-y` | Dodavanje custom CA-potpisanih sertifikata u CR sistem |
| `--address` | `-a` | Izmena IP adrese CR management hosta |
| `--changepassword` | `-w` | Promena passphrase-a / lozinki za lockbox, Postgres bazu i `crso`; kreira i CR backup |
| `--check` | `-c` | Provera konfiguracije i preduslova za instalaciju |
| `--deploy` | `-d` | Konfiguracija Cyber Recovery softvera (**samo OVA**) |
| `--forcerecreate` | `-f` | Prinudno ponovno kreiranje CR kontejnera |
| `--gencertrequest` | `-j` | Generisanje `CRSERVICE.csr` fajla (certificate signing request) |
| `--getdbpassword` | — | Pokretanje procedure za preuzimanje Postgres lozinke (nakon `--quick-install`) |
| `--help` | `-h` | Prikaz pomoći |
| `--install` | `-i` | Instalacija CR softvera (interaktivno, sa svim lozinkama) |
| `--quick-install` | `-Q` | Automatska instalacija CR softvera (**samo bare metal**) |
| `--recover` | `-r` | Oporavak CR softvera iz backup paketa |
| `--restart` | `-e` | Zaustavljanje pa pokretanje svih servisa |
| `--rootreset` | — | Reset CR root sertifikata za sve servise |
| `--save` | `-b` | Snimanje konfiguracije CR softvera |
| `--securereset` | `-l` | Reset root sertifikata i enkripcionih ključeva (stop → regeneracija → start) |
| `--shenable` | `-m` | Omogućavanje Sheltered Harbor funkcije |
| `--start` | `-s` | Pokretanje svih servisa ili pojedinačnog uređaja |
| `--stop` | `-p` | Zaustavljanje CR softvera |
| `--uninstall` | `-x` | Deinstalacija CR softvera |
| `--upgcheck` | `-k` | Provera spremnosti za nadogradnju (preupdate readiness check) |
| `--upgrade` | `-u` | Nadogradnja CR softvera |
| `--verifypassword` | `-v` | Verifikacija lockbox passphrase-a, Postgres i `crso` lozinki |
| `--changeiprange` | — | Promena opsega IP adresa Podman mreža (koristi se uz `--forcerecreate`) |

## 11.2 Opšte napomene

- Za unos lozinke ili passphrase-a imate **tri pokušaja**.
- Ako se CR ne instalira, ne nadograđuje ili ne radi ispravno, pokrenuti `./crsetup.sh --check` — prikazuje nedostajuće ili nekompatibilne sistemske zahteve i instalirane verzije Podman-a i Podman Compose-a.
- `--addcustcert` validira lanac sertifikata. Potreban je **pun lanac** — leaf, root i svi intermediate sertifikati. Ako postojeći custom sertifikati nemaju pun lanac, `--upgcheck` i `--upgrade` prikazuju grešku i traže dodavanje novog sertifikata.

---

# 12. Instalacija na Red Hat Enterprise Linux

## 12.1 Preduslovi neposredno pre instalacije

Proveriti sledeće (detalji u Delu II):

| # | Preduslov | Provera |
|---|---|---|
| 1 | Podržana verzija RHEL-a (8.10, 9.4, 9.6) sa svim zakrpama | `cat /etc/redhat-release` |
| 2 | `umask` = 022 | `umask` |
| 3 | `libstdc++` instaliran | `rpm -q libstdc++` |
| 4 | `rsyslog` i `rsyslog-gnutls` instalirani (ako se koristi audit forwarding) | `rpm -q rsyslog rsyslog-gnutls` |
| 5 | Docker i docker-compose uklonjeni, sistem rebootovan | `rpm -qa \| grep -i docker` |
| 6 | Firewall konfigurisan **pre** instalacije Podman-a | `firewall-cmd --list-all` |
| 7 | Podman i Podman Compose instalirani globalno | `podman --version` / `podman-compose --version` |
| 8 | ≥ 5 GB za raspakivanje, ≥ 20 GB za instalaciju | `df -h` |
| 9 | DNS rezolucija radi | `nslookup <fqdn>` |

Podešavanje `umask` ako je potrebno:

```bash
echo "umask 022" >> ~/.bashrc
source ~/.bashrc
umask   # očekivano: 0022
```

---

## 12.2 SELinux

Za softversku instalaciju **preporučuje se korišćenje SELinux-a** kao Linux Security Module radi dodatne bezbednosti OS-a. Pošto SELinux obezbeđuje granularnu kontrolu pristupa, treba biti pažljiv pri konfigurisanju bezbednosnih konteksta za CR fajlove i direktorijume.

```bash
# Provera trenutnog režima
getenforce
```

> **Ako omogućite SELinux**, nakon instalacije je obavezno izmeniti SELinux kontekst za CR binarne fajlove — videti odeljak 13.2. U suprotnom Cyber Recovery se možda neće pokrenuti posle reboota.

---

## 12.3 Provera preduslova skriptom

```bash
cd staging
./crsetup.sh --check
```

**Ponašanje:**
- Ako neki preduslov nije zadovoljen, provera **ne uspeva i izlazi**. Ispraviti problem i ponoviti proveru dok ne prođe.
- Komanda prikazuje i instalirane verzije Podman-a i Podman Compose-a.

**Upozorenje o rsyslog paketima:**
Za slanje audit logova ka eksternom SIEM serveru deployment mora sadržati rsyslog pakete. Ako nisu instalirani, provera prikazuje **upozorenje**. Instalacija se nastavlja i uspeva, ali SIEM server se ne može integrisati.

---

## 12.4 Provera IP adresa management hosta

Ako management host ima više IP adresa, mora se eksplicitno zadati adresa koju Cyber Recovery koristi za komunikaciju sa DD storage-om u vault-u.

```bash
# Provera broja IP adresa
hostname -i

# Ako komanda vrati više adresa, zadati adresu za komunikaciju sa DD storage-om
export PodmanHost=<IP_adresa>
```

> Ovaj korak se lako previdi na serverima sa više NIC-ova i uzrokuje probleme u komunikaciji sa DD sistemom.

---

## 12.5 Izbor metode instalacije

| Metoda | Komanda | Ponašanje |
|---|---|---|
| **Interaktivna** | `./crsetup.sh --install` | Traže se sve lozinke: lockbox passphrase, Postgres lozinka, `crso` lozinka. Bira se i instalacioni direktorijum. |
| **Brza** | `./crsetup.sh --quick-install` | Traži se **samo lockbox passphrase**. Za `crso` se generiše podrazumevana lozinka koja se mora promeniti pri prvoj prijavi. Lozinka baze se generiše nasumično i čuva lokalno — mora se promeniti nakon instalacije. Samo bare metal. |

> **Preporuka za produkcione implementacije:** koristiti `--install`. Kontrola nad lozinkama i putanjama je bitna za dokumentovanje i predaju kupcu. `--quick-install` je pogodan za laboratorijske i demo instalacije.

**Trajanje instalacije:** približno 10 minuta.

---

## 12.6 Procedura instalacije

### Korak 1 — Pokretanje instalacije

```bash
# Interaktivna instalacija
./crsetup.sh --install

# ILI brza instalacija
./crsetup.sh --quick-install
```

### Korak 2 — Validacija Podman-a

Tokom instalacije instalater validira instalirane verzije Podman-a i Podman Compose-a:

| Situacija | Ponašanje |
|---|---|
| Verzije su podržane | Instalacija se nastavlja bez pitanja |
| Verzije nisu podržane (`--install`) | Instalater pita da li da instalira/nadogradi Podman offline repozitorijume:<br>`y` — instalira/nadograđuje i nastavlja<br>`n` — Podman provera ne uspeva, **instalacija se prekida** |
| Verzije nisu podržane (`--quick-install`) | Podman i Podman Compose se automatski instaliraju/nadograđuju, bez pitanja |
| Instalirana verzija je **viša** od podržane | **Instalacija se prekida** |

> Poslednji red je važan: novija verzija Podman-a nije prednost — CR je odbija. Držati se verzija iz matrice kompatibilnosti (RHEL: Podman 4.9.4, Podman Compose 1.5.0).

### Korak 3 — EULA

- Pritisnuti `Enter` za prikaz End User License Agreement.
- `q` za izlazak iz prikaza u bilo kom trenutku.
- `y` za prihvatanje.

Ako se EULA odbije, instalacija se zaustavlja.

### Korak 4 — Kreiranje `cyber-recovery-admin` korisnika

Instalacija pokušava da kreira Linux korisnika `cyber-recovery-admin` na management hostu, sa rezervisanim **UID:GID 14999**.

Ako se prikaže upozorenje o kreiranju ovog korisnika, izabrati nastavak ili otkazivanje instalacije.

> Ako se instalacija završi uprkos upozorenju, Cyber Recovery radi ispravno, ali neke instalacione direktorijume može posedovati korisnik različit od `cyber-recovery-admin`.
>
> **Preporuka:** proveriti da li UID/GID 14999 već postoji na sistemu **pre** instalacije i osloboditi ga ako je zauzet.

### Korak 5 — Instalacioni direktorijumi

> Ovaj korak se preskače kod `--quick-install`.

| Prompt | Podrazumevana vrednost |
|---|---|
| Direktorijum za instalaciju CR softvera | `/opt/dellemc/cr` |
| Direktorijum za bazu podataka | `/opt/dellemc/cr/postgresdata` |

Pritisnuti `Enter` za prihvatanje podrazumevane vrednosti ili uneti sopstvenu putanju.

> **Putanja ne sme sadržati razmak.**

Instalacija prikazuje izlaz o kreiranju direktorijuma, učitavanju Podman kontejnera i pokretanju Postgres baze. Kreira i interne IP adrese koje omogućavaju komunikaciju između Podman kontejnera.

### Korak 6 — Lozinke i passphrase

Uneti i potvrditi:

1. **Lockbox passphrase**
2. **Lozinku baze podataka (Postgres)**
3. **Lozinku `crso` naloga**

> Kod `--quick-install` traži se samo lockbox passphrase.

`crso` je podrazumevani Security Admin i može se smatrati superuser-om. Prva prijava u Cyber Recovery obavlja se kao `crso`.

**Zahtevi za passphrase i lozinke:**

| Zahtev | Detalj |
|---|---|
| Dužina | 9–64 znaka (podrazumevani minimum od 9 znakova je promenljiv) |
| FIPS usklađenost | Lockbox passphrase mora imati **najmanje 14 znakova** |
| Numerički znak | Najmanje jedan (0–9) |
| Veliko slovo | Najmanje jedno (A–Z) |
| Malo slovo | Najmanje jedno (a–z) |
| Specijalni znak | Najmanje jedan: `~!@#$%^&*_-+={}[]|:;"?<>,.'\`\/` |
| Razmaci | Nisu dozvoljeni |
| Korisničko ime | Lozinka ne sme sadržati korisničko ime |
| Ponovna upotreba | Definisan broj prethodnih lozinki se ne može ponovo koristiti |

> **Za ovo izdanje: ne koristiti dvotačku (`:`) u lozinci Postgres baze.**

> ### CAUTION — lockbox passphrase
> **Lockbox passphrase se ne sme zaboraviti; ne može se povratiti.** Ako se zaboravi ili izgubi, potrebna je **sveža instalacija** Cyber Recovery softvera.
> Lockbox passphrase je neophodan za:
> - izvršavanje nadogradnji
> - reset lozinke Security Admin korisnika
>
> Čuvati ga na siguran i redundantan način, **van** Cyber Recovery servera.

---

## 12.7 Rezultat instalacije

Instalaciona procedura pokreće Cyber Recovery servise i zatim izlazi.

- Učitava se fajl `cyber-recovery.service`. Ako se management host restartuje posle gašenja, ovaj fajl usmerava host da automatski pokrene CR servise.
- **Napomena:** pune opcije system control-a nisu konfigurisane. Ako pokrenete `systemctl` za `cyber-recovery.service`, status će biti prikazan kao `inactive` — to je očekivano ponašanje.
- Na kraju instalacione skripte ispisuje se **URL za pristup UI-u** i **spisak portova** koji moraju biti otvoreni na firewall-u. Zabeležiti oba.

---

## 12.8 Preuzimanje Postgres lozinke nakon `--quick-install`

Procedura `--quick-install` generiše podrazumevanu lozinku za Postgres bazu. Za njeno preuzimanje koristi se Dell Support attestation.

```bash
# 1. Generisati string
$ crsetup.sh --getdbpassword
```

Primer izlaza:

```
26.01.14 07_15_20 : Using timezone 'America/New_York' for Cyber Recovery service containers...
26.01.14 07_15_20 : Determining the Podman bridge network IP...
26.01.14 07_15_20 : Using IP Address x.x.x.x to send mail from the notifications container...
26.01.14 07_15_20 : |===============================================================
26.01.14 07_15_20 : | You are about to retrieve the Postgress password.
26.01.14 07_15_20 : |===============================================================

Provide the following string to Dell Support to get it signed.
xZ4HcztjbxxxxxxxxxxXL7POc2IMhkbP

Enter the base64-encoded signature:
```

**Koraci:**
1. Generisani string proslediti Dell Support-u.
2. Dell Support vraća base64-kodiran potpis.
3. Uneti potpis — sistem prikazuje trenutnu Postgres lozinku.

> Nakon preuzimanja **promeniti lozinku** komandom `./crsetup.sh --changepassword`.

---

# 13. Post-install koraci (RHEL)

## 13.1 Verifikacija Podman mreža

Ako su tokom instalacije readresirane Podman mreže, proveriti da su bridge interfejsi i mreže `cr_back` i `cr_front` na **odvojenim subnetovima**.

```bash
podman network inspect cr_back  | grep -E 'subnet|gateway'
podman network inspect cr_front | grep -E 'subnet|gateway'

# Provera da kontejneri rade
podman ps --format 'table {{.Names}}\t{{.Status}}' | grep cr_
```

Ako je potrebno naknadno readresiranje, videti proceduru *Readdressing the Podman network* u Delu VI.

---

## 13.2 Izmena SELinux konteksta

> **Ne primenjuje se na OVA deployment** — tamo se koristi AppArmor.

U SELinux okruženju Cyber Recovery se možda neće pokrenuti posle reboota, zbog neoznačenog tipa konteksta i custom politika.

```bash
# Pod pretpostavkom da je CR instaliran u /opt/dellemc/cr
chcon -u system_u -t bin_t /opt/dellemc/cr/bin/cradmin
chcon -u system_u -t bin_t /opt/dellemc/cr/bin/crcli
chcon -u system_u -t bin_t /opt/dellemc/cr/bin/crsetup.sh
chcon -u system_u -t bin_t /opt/dellemc/cr/bin/crshutil
chcon -u system_u -t bin_t /opt/dellemc/cr/bin/crsshutil
```

Zatim **rebootovati sistem**.

**Verifikacija:**

```bash
ls -Z /opt/dellemc/cr/bin/
```

Očekivani izlaz:

```
-rwxr-----. root root system_u:object_r:bin_t:s0 cradmin
-rwxr-----. root root system_u:object_r:bin_t:s0 crcli
-rwxr-----. root root system_u:object_r:bin_t:s0 crsetup.sh
-rwxr-----. root root system_u:object_r:bin_t:s0 crshutil
-rwxr-----. root root system_u:object_r:bin_t:s0 crsshutil
```

> Installation Guide navodi tri fajla (`cradmin`, `crcli`, `crsetup.sh`); Product Guide u poglavlju za rešavanje problema navodi pet, uključujući `crshutil` i `crsshutil`. Koristiti verziju sa pet fajlova.

---

## 13.3 Konfiguracija NTP-a preko chrony

Preporučuje se korišćenje NTP protokola za sinhronizaciju **svih komponenti** u vault-u.

### Provera i instalacija chrony-ja

```bash
# Provera da li je chrony instaliran
rpm -qa chrony
```

Ako paket nije instaliran:

```bash
# RHEL 9.4 i 9.6 — sistem sa pristupom mreži i RHEL repozitorijumu
yum install chrony
```

> Za sisteme bez pristupa mreži (tipično za vault): preuzeti chrony RPM paket sa RHEL sajta i instalirati ga lokalno.
> Za SLES koristiti `zypper install chrony`. Za OVA, ako verzija chrony-ja nije ažurna, instalirati najnoviji Cyber Recovery `osupdate` binary.

### Konfiguracija

```bash
# 1. Omogućiti chronyd servis
systemctl enable chronyd

# 2. Pokrenuti servis
systemctl start chronyd

# 3. Proveriti da chronyd radi
systemctl status chronyd

# 4. Dodati NTP server u konfiguraciju
vi /etc/chrony.conf
#    Dodati liniju:
#    server <IP_adresa_NTP_servera> iburst

# 5. Restartovati servis
systemctl restart chronyd.service

# 6. Provera da je servis aktivan
systemctl is-active chronyd.service
```

### Verifikacija sinhronizacije

```bash
# Provera da je sistemski sat sinhronizovan sa NTP izvorom i tačnost sinhronizacije
chronyc tracking

# Provera koji NTP server chrony koristi i da li je dostupan
chronyc sources

# Prinudno usklađivanje sata — izlaz "200 OK" znači uspeh
chronyc -a makestep

# Potvrda da je NTP servis aktivan
timedatectl
```

> **Podsetnik:** ako se vreme CR hosta razlikuje od vremena authenticator-a za više od ±60 sekundi, multifactor authentication se ne može omogućiti. Ako naknadno menjate vreme hosta, zaustavite pa ponovo pokrenite CR servise.

---

## 13.4 Konfiguracija kaskadne replikacije

Primeniti **samo** ako je deployment konfigurisan za kaskadnu replikaciju — produkcioni podaci se repliciraju u jedan Cyber Recovery vault, pa zatim u drugi vault (npr. cleanroom).

Na management serveru u **drugom** vault-u, kao `root`:

```bash
# Za softversku instalaciju
touch /opt/dellemc/cr/etc/config/airgap_from_vault
```

Komanda obaveštava Cyber Recovery softver o kaskadnoj konfiguraciji.

> **Napomena o retenciji:** ako i CR sistem u vault-u i CR sistem u cleanroom-u koriste Secure Snapshot-ove, trajanje retencije u cleanroom politikama mora biti **veće ili jednako** trajanju retencije u odgovarajućim vault politikama.

---

## 13.5 Firewall

Otvoriti portove koje je instalaciona skripta ispisala na kraju izvršavanja. Minimalno:

```bash
firewall-cmd --permanent --add-port=14777/tcp   # CR UI
firewall-cmd --permanent --add-port=14778/tcp   # CR REST API
firewall-cmd --permanent --add-port=14780/tcp   # CR API dokumentacija (opciono)
firewall-cmd --permanent --add-port=25/tcp      # SMTP (ako se koristi Postfix lokalno)
firewall-cmd --reload
firewall-cmd --list-all
```

> Puna tabela portova je u Delu II, odeljak 6.1.

---

## 13.6 Vremenska zona

Podesiti vremensku zonu tako da vremena u logovima budu tačna. Cyber Recovery koristi vremensku zonu hosta za servisne kontejnere (vidljivo u izlazu `crsetup.sh` komandi: `Using timezone '<zona>' for Cyber Recovery service containers...`).

```bash
timedatectl set-timezone <Region/Grad>
timedatectl
```

> Za naknadnu promenu vremenske zone na postojećem deploymentu postoji posebna procedura (*Changing time zones*) — videti Deo VI.

---

# 14. Prva prijava i inicijalna konfiguracija — zajedničko

> Ovo poglavlje važi jednako za RHEL/SLES softversku instalaciju i za OVA deployment.

## 14.1 Pristup korisničkom interfejsu

Otvoriti podržan pretraživač i otići na URL koji je ispisan na kraju instalacione skripte:

```
https://<host>:14777
```

gde je `<host>` hostname ili IP adresa management hosta.

**Podržani pretraživači:** Google Chrome, Microsoft Edge, Mozilla Firefox. Za najažurnije informacije videti E-Lab Navigator.

> **Ako se stranica ne otvara:** proveriti da su portovi 14777, 14778 i 14780 otvoreni na firewall-u i da DNS podešavanja rade. Za dijagnostiku pogledati logove `edge` i `users` servisa.

---

## 14.2 Prijava kao `crso`

Instalaciona procedura kreira podrazumevanog korisnika `crso` i dodeljuje mu ulogu **Security Admin**. Prvu prijavu mora obaviti `crso`.

| Način instalacije | Korisničko ime | Lozinka |
|---|---|---|
| `crsetup.sh --install` | `crso` | lozinka koju ste kreirali tokom instalacije |
| `crsetup.sh --quick-install` | `crso` | `ChangeMe@1` — sistem odmah traži promenu |
| OVA (`crsetup.sh --deploy`) | `crso` | lozinka koju ste kreirali tokom deploymenta |

> **VAŽNO:** ne prijavljivati se prvo preko CLI-ja. To rezultira greškom:
> `Failed to authenticate user. Please re-login to run commands.`
> Prva prijava mora biti preko UI-ja.

---

## 14.3 Getting Started čarobnjak

Nakon prijave prikazuje se **Getting Started** čarobnjak.

1. Proći kroz čarobnjak i pregledati zahteve.
2. Dodati **najmanje jednog Admin korisnika**.
3. Odjaviti se i prijaviti kao Admin korisnik.
4. Kao Admin, dovršiti čarobnjak: dodati vault storage i kreirati politiku (videti poglavlje 15).

> Čarobnjak se može ponovo pozvati u bilo kom trenutku: ikona na masthead navigaciji → **Getting Started**.

---

## 14.4 Multifactor authentication

MFA je **podrazumevano omogućen**. Opciono se može zaobići, čime se pri narednim prijavama ne traži bezbednosni kod.

**Preduslov za MFA:** vreme CR hosta ne sme odstupati više od ±60 sekundi od vremena authenticator-a. Ako odstupa, MFA se ne može omogućiti. Zato je interna NTP konfiguracija preporučena.

> Ako se `crso` lozinka menja komandom `crsetup.sh --changepassword`, MFA se **onemogućava** i mora se ponovo uključiti.

---

## 14.5 Licenciranje

Za licenciranje Cyber Recovery-ja koristi se **Cyber Recovery Software Instance ID**. Detalji su u *Dell PowerProtect Cyber Recovery Product Guide*.

---

## 14.6 Sertifikati

Za zamenu podrazumevanog sertifikata CA-potpisanim:

```bash
# Generisanje certificate signing request-a
./crsetup.sh --gencertrequest
# Kreira se fajl CRSERVICE.csr

# Dodavanje custom CA-potpisanog sertifikata
./crsetup.sh --addcustcert
```

> **Potreban je pun lanac sertifikata** — leaf, root i svi intermediate sertifikati. Komanda validira lanac. Ako lanac nije potpun, kasnije komande `--upgcheck` i `--upgrade` prijavljuju grešku i traže novi sertifikat.

---

## 14.7 Backup Cyber Recovery konfiguracije

Odmah nakon završene inicijalne konfiguracije kreirati backup:

```bash
# Backup preko CLI-ja
./crsetup.sh --save
```

Backup se čuva u `/opt/dellemc/cr-configs`. **Kopirati ga na lokaciju van Cyber Recovery servera.**

> Preferirani način je backup preko CR UI-ja, koji postavlja server DR backup na konfigurisani DD MTree.

---

# 15. Osnovna konfiguracija vault-a u CR UI — zajedničko

> Ovo poglavlje važi jednako za oba načina instalacije. Radnje se izvršavaju kao **Admin** korisnik.

## 15.1 Dodavanje vault storage-a (DD sistem)

**Preduslovi:**
- DD sistem radi pod DDOS 7.13 ili 8.x
- Opcija `force-minimum-root-squash-default` je `disabled` (videti Deo II, 5.6)
- Prijavljeni ste kao Admin korisnik (Vault Operator može samo da pregleda i izvozi informacije)

Ako se DD sistem definiše prvi put, koristiti **Getting Started** čarobnjak.

**Napomene:**
- **Ne konfigurisati više Cyber Recovery servera da koriste isti DD sistem.**
- Polje **Secure Snapshot Capable** u tabeli Systems pokazuje da li DD podržava Secure Snapshot-ove (videti Deo II, 5.3).
- Podrazumevani pragovi popunjenosti su 80% (warning) i 90% (critical); pri dostizanju se generišu alerti i email notifikacije.

---

## 15.2 Dodavanje vCenter servera

Potrebno **samo** ako se za recovery koristi PowerProtect Data Manager verzije **starije od 20.3** u on-premises vault-u. Za PPDM 20.3+ i za AWS deployment vCenter asset nije potreban.

**Main Menu → Infrastructure → Assets → vCenters → Add**

| Polje | Opis |
|---|---|
| Nickname | Naziv za vCenter server |
| FQDN or IP Address | FQDN ili IP adresa vCenter servera. Pri izmeni ove vrednosti morate ponovo uneti sve lozinke. |
| Username | Administratorsko korisničko ime za prijavu na vCenter |
| Password | Administratorska lozinka |
| Tags | Opciono. Tag duži od 24 znaka prikazuje se skraćen na 21 znak sa tri tačke. |

> Ako je vCenter pridružen PPDM aplikaciji verzije starije od 20.3, nadogradnja aplikacije na 20.3+ **ne uklanja** postojeću vCenter asocijaciju.

---

## 15.3 Dodavanje aplikativnih asset-a

**Main Menu → Infrastructure → Assets → Applications → Add**

**Preduslov na DD sistemu** — kreirati i dodeliti DD Boost korisnika:

```bash
# role: admin za NetWorker i Avamar, none za PowerProtect Data Manager
sysadmin@dd-vault# user add <username> role <admin|none>
sysadmin@dd-vault# ddboost user assign <username>
```

**Korisnik pod kojim se aplikacija dodaje u vault-u:**

| Aplikacija / platforma | Korisnik |
|---|---|
| NetWorker na Linux-u | `root` (CR koristi komande poput `nsrdr` koje zahtevaju root privilegije) |
| NetWorker na Windows-u | `Admin` |
| Avamar | `Admin` |
| PowerProtect Data Manager | `Admin` |
| Cyber Detect | `Admin`, uz važeći sertifikat na Cyber Detect serveru |

> Detaljne procedure za NetWorker i PPDM su u Delu IV i Delu V.

---

## 15.4 Kreiranje politike — Add Policy čarobnjak

**Main Menu → Policies → Add**

### Stranica 1 — Policy Information

| Polje | Opis |
|---|---|
| **Policy Name** | Naziv politike |
| **Description** | Opciono, opis politike |
| **Policy Type** | `PPDM`, `Sheltered Harbor` ili `Standard`.<br>`Standard` obuhvata NetWorker, Avamar, Filesystem i Other.<br>**PPDM politika zahteva najmanje dva MTree-a.**<br>Ako Sheltered Harbor funkcija nije omogućena, ta opcija se ne prikazuje. |
| **Storage Target** | Vault storage koji sadrži replication kontekst koji politika štiti. **Ne može se menjati za postojeću politiku.** |
| **Tags** | Opciono. Tag duži od 24 znaka prikazuje se skraćen. |

### Stranica 2 — Replication

| Polje | Opis |
|---|---|
| **Replication Context** | Izabrati replication kontekst sa podacima koje treba zaštititi i koristiti za recovery. Izabrati Ethernet port na storage instanci konfigurisan za replikaciju. Dodatni konteksti se dodaju preko **Add Replication Context**. |
| **ServerDR Context** | Za PPDM deployment: izabrati kontekst sa server disaster recovery informacijama i odgovarajući Ethernet port. |

**Napomene:**
- Politici se dodeljuje **jedan** data replication kontekst — **osim** za PPDM tip, koji može imati **više** data konteksta.
- **Ne birati data ili management Ethernet interfejse** — samo namenski replikacioni.
- Za PPDM politiku, Cyber Recovery iz liste data konteksta **izuzima** kontekste koji odgovaraju standardnoj PPDM Server DR konvenciji imenovanja, čime sprečava slučajan izbor Server DR konteksta kao data konteksta.

### Stranica 3 — Retention Lock

| Opcija | Opis |
|---|---|
| **Compliance** | Stroža zaštita. Zaključani fajlovi se ne mogu obrisati ni prepisati dok retencija ne istekne. |
| **Governance** | Fleksibilna retencija zasnovana na politici. Ovlašćeni korisnici mogu menjati podešavanja. |
| **None** | Bez retention lock-a. Podaci se mogu obrisati ili prepisati u bilo kom trenutku. **Zaobilazi dodatni sloj zaštite — pažljivo razmotriti rizike.** |

> Ako se kreira nova politika na DD sistemu koji podržava Secure Snapshot-ove, ove opcije **se ne prikazuju**. Prikazuju se samo pri izmeni postojeće repository-based politike.

### Stranica 4 — Scheduling

| Polje | Opis |
|---|---|
| **Min Retention Lock Period** | Minimalno trajanje retencije koje politika može primeniti. **Ne može biti kraće od 12 sati.** Važi i za Retention Lock i za Secure Snapshot politike. |
| **Max Retention Lock Period** | Maksimalno trajanje retencije. **Najviše 5 godina** za Retention Lock politike, **180 dana** za Secure Snapshot politike. |
| **Retain for** | Podrazumevano trajanje retencije za PIT kopije, u opsegu između minimuma i maksimuma. |
| **Add Schedule** | Dodavanje rasporeda za politiku (Sync, Copy, Lock, Recovery Check, Analyze) |
| **No Schedules** | Označiti ako se raspored ne dodaje |

### Stranica 5 — Storage Security Credentials

Prikazuje se **samo** ako je izabran Retention Lock Compliance kontekst ili Compliance tip retention lock-a.

Uneti korisničko ime i lozinku **DD Security Officer-a** (nalog kreiran na DD sistemu — videti Deo II, 9.4).

### Stranica 6 — Summary

- **Finish** — dodaje politiku
- **Back** — povratak na prethodne stranice radi izmene

Nakon toga kliknuti **Launch**. Prikazuje se pano *What's New in Cyber Recovery*; klikom na **Close** otvara se Cyber Recovery dashboard.

---

## 15.5 Pregled Cyber Recovery operacija

| Operacija | Opis |
|---|---|
| **Replication (Sync)** | MTree replikacija sa produkcionog na vault DD sistem, uz DD deduplikaciju |
| **Copy** | Kreiranje point-in-time (PIT) kopije iz najnovije replikacije. U zavisnosti od tipa immutability-ja, kopija se čuva u repository MTree-u ili se kreira kao Secure Snapshot na replikacionom target-u. |
| **Lock** | Zaštita PIT kopije od izmene u zadatom trajanju. Repository kopije koriste Retention Lock Governance ili Compliance; Secure Snapshot kopije su nepromenljive od trenutka kreiranja, a retencija im se može produžiti. |
| **Analyze** | Analiza zaključanih ili nezaključanih kopija radi otkrivanja indikatora kompromitacije, sumnjivih fajlova ili malvera |
| **Recovery** | Korišćenje podataka iz PIT kopije za oporavak |
| **Recovery Check** | Zakazana ili on-demand provera da se kopija može oporaviti (PPDM politike i NetWorker 19.9+) |
| **Sheltered Harbor Copy** | Samo ako je Sheltered Harbor omogućen — generiše validirane, enkriptovane i retention-locked arhivske volumene sa izveštajem i attestation-om |

---

## 15.6 Prvo izvršavanje i validacija

1. Ručno pokrenuti Sync operaciju za novokreiranu politiku.
2. Pratiti job na **Main Menu → Jobs**, tab **Running**.
3. Nakon završetka Sync-a, pokrenuti Copy operaciju.
4. Proveriti da je kopija kreirana i vidljiva na **Recovery → Copies**.
5. Proveriti Lock operaciju i stanje retencije kopije.
6. Proveriti dashboard i da nema kritičnih alerta.

> Ako Sync ne uspeva: proveriti da je inicijalna replikacija između produkcionog i vault DD sistema završena pre definisanja politike, da DD sistem nije ostao bez prostora i da su NFS opcije ispravno podešene (Deo II, 5.6).

---

# Dodatak: Post-install checklist za RHEL

| # | Stavka | Status | Napomena |
|---|---|---|---|
| 1 | `./crsetup.sh --check` prošao bez grešaka | | |
| 2 | Podman i Podman Compose u podržanim verzijama | | |
| 3 | `PodmanHost` postavljen (ako host ima više IP adresa) | | |
| 4 | Instalacija završena bez upozorenja o `cyber-recovery-admin` | | |
| 5 | Instalacioni direktorijum | | podrazumevano `/opt/dellemc/cr` |
| 6 | Direktorijum baze | | podrazumevano `/opt/dellemc/cr/postgresdata` |
| 7 | Lockbox passphrase zabeležen i pohranjen van CR servera | | **kritično** |
| 8 | Postgres lozinka zabeležena (i promenjena ako `--quick-install`) | | |
| 9 | `crso` lozinka zabeležena (i promenjena ako `--quick-install`) | | |
| 10 | Podman mreže `cr_back` i `cr_front` na odvojenim subnetovima | | |
| 11 | Nema kolizije Podman subnetova sa vault mrežom | | |
| 12 | SELinux kontekst izmenjen za svih 5 binarnih fajlova | | ako je SELinux uključen |
| 13 | Sistem rebootovan nakon `chcon` komandi | | |
| 14 | CR servisi pokrenuti posle reboota | | |
| 15 | chrony konfigurisan i sinhronizovan (`chronyc tracking`) | | |
| 16 | Vremenska zona podešena | | |
| 17 | Portovi sa kraja instalacione skripte otvoreni na firewall-u | | |
| 18 | `airgap_from_vault` fajl kreiran (samo kaskadna replikacija) | | |
| 19 | UI dostupan na `https://<host>:14777` | | |
| 20 | Prva prijava kao `crso` uspešna (preko UI-ja, ne CLI-ja) | | |
| 21 | Admin korisnik kreiran | | |
| 22 | MFA konfigurisan po dogovoru sa kupcem | | |
| 23 | Custom CA sertifikat dodat (pun lanac) | | ako se koristi |
| 24 | SMTP i email notifikacije konfigurisane | | ako se koriste |
| 25 | Licenca aktivirana (Software Instance ID) | | |
| 26 | Vault storage dodat | | |
| 27 | vCenter dodat | | samo ako PPDM < 20.3 |
| 28 | Prva politika kreirana i uspešno izvršena | | |
| 29 | CR backup kreiran i pohranjen van servera | | `crsetup.sh --save` ili UI |

---

*Kraj Dela III-a. Videti i: `03b-Instalacija-OVA.md` (OVA deployment), Deo IV (integracija sa NetWorker-om), Deo V (integracija sa PPDM-om).*
