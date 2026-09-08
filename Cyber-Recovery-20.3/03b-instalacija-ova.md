# DEO III-b — INSTALACIJA VIRTUELNOG APPLIANCE-a (OVA) NA VMware

**Dell PowerProtect Cyber Recovery 20.3 — Uputstvo za instalaciju i integraciju**
Verzija dokumenta: 0.1 | Namena: System Engineer

> **Izvori:** *Dell PowerProtect Cyber Recovery 20.3 Installation and Upgrade Guide*, *Dell PowerProtect Cyber Recovery 20.3 Product Guide*.
> **Preduslov:** pre početka mora biti u potpunosti izvršen **Deo II — Preduslovi** (`02-Preduslovi.md`), posebno odeljak 4.3 (zahtevi za virtuelni appliance) i provera 4KN ograničenja.
> **Zajednička poglavlja:** poglavlja **14** (prva prijava i inicijalna konfiguracija) i **15** (osnovna konfiguracija vault-a u CR UI) identična su za oba načina instalacije i nalaze se u fajlu `03a-Instalacija-RHEL.md`. Nakon završetka poglavlja 13 u ovom fajlu, nastaviti tamo.

---

## Sadržaj

- [10b. Preuzimanje OVA fajla](#10b-preuzimanje-ova-fajla)
- [11b. Referenca komande crsetup.sh (OVA)](#11b-referenca-komande-crsetupsh-ova)
- [12b. Preddeployment provere](#12b-preddeployment-provere)
- [13. Deployment virtuelnog appliance-a](#13-deployment-virtuelnog-appliance-a)
- [13b. Post-deploy koraci (OVA)](#13b-post-deploy-koraci-ova)
- [Razlike u odnosu na softversku instalaciju](#razlike-u-odnosu-na-softversku-instalaciju)
- [Dodatak: Post-deploy checklist za OVA](#dodatak-post-deploy-checklist-za-ova)

---

# 10b. Preuzimanje OVA fajla

Sa **Dell Online Support** preuzeti **Cyber Recovery virtual appliance** fajl za instalaciju u VMware ESXi okruženju.

| Paket | Format | Namena |
|---|---|---|
| Cyber Recovery virtual appliance | `.ova` | Novi deployment |
| Cyber Recovery virtual appliance update package | `.bin` | Nadogradnja postojećeg OVA deploymenta |
| Cyber Recovery `osupdate` binary | — | Ažuriranje OS paketa na appliance-u (uključujući chrony i rsyslog pakete) |

**Prenos u vault:** OVA fajl mora se preneti u izolovano vault okruženje u skladu sa procedurom kupca za unos podataka (kontrolisani jump host ili odobreni prenosivi medij).

---

# 11b. Referenca komande `crsetup.sh` (OVA)

Puna tabela opcija je u fajlu `03a-Instalacija-RHEL.md`, poglavlje 11. Opcije koje se koriste u OVA toku rada:

| Opcija | Kratka | Namena u OVA deploymentu |
|---|---|---|
| `--deploy` | `-d` | **Konfiguracija Cyber Recovery softvera — samo OVA.** Ovo je komanda kojom se pokreće instalacija na novodeploy-ovanom appliance-u. |
| `--check` | `-c` | Provera konfiguracije i preduslova |
| `--start` / `--stop` / `--restart` | `-s` / `-p` / `-e` | Upravljanje servisima |
| `--save` | `-b` | Snimanje konfiguracije (backup) |
| `--recover` | `-r` | Oporavak iz backup paketa |
| `--changepassword` | `-w` | Promena lockbox passphrase-a, Postgres i `crso` lozinke |
| `--verifypassword` | `-v` | Verifikacija passphrase-a i lozinki |
| `--addcustcert` | `-y` | Dodavanje custom CA-potpisanog sertifikata |
| `--gencertrequest` | `-j` | Generisanje CSR fajla |
| `--forcerecreate` | `-f` | Prinudno ponovno kreiranje kontejnera |
| `--changeiprange` | — | Promena Podman subnetova (uz `--forcerecreate`) |
| `--upgcheck` / `--upgrade` | `-k` / `-u` | Provera spremnosti i nadogradnja |
| `--shenable` | `-m` | Omogućavanje Sheltered Harbor funkcije |

> **Opcija `--quick-install` (`-Q`) ne postoji za OVA** — namenjena je isključivo bare-metal instalacijama.
> **Opcija `--install` (`-i`) se ne koristi za OVA** — koristi se `--deploy`.

**Opšte:** za unos lozinke ili passphrase-a imate **tri pokušaja**.

---

# 12b. Preddeployment provere

Pre pokretanja Deploy OVF Template čarobnjaka proveriti sledeće.

## 12b.1 Infrastrukturne provere

| # | Stavka | Zahtev |
|---|---|---|
| 1 | Verzija hipervizora | VMware vCenter ili ESXi **8.0.x ili 9.0** |
| 2 | Slobodan prostor za OVA deployment | ~10 GB |
| 3 | Slobodan prostor za tri diska | ~195 GB (48 + 48 + 96 GB) |
| 4 | Provisioning | Preporučen **thin provisioning** |
| 5 | Resursi VM-a | 2 vCPU (jedan core po socket-u), 8 GB RAM |
| 6 | Format sektora datastore-a | **Ne sme biti 4KN** — videti odeljak 12b.2 |
| 7 | Port grupa / VLAN za CR appliance | Definisan i dostupan iz vault mreže |

## 12b.2 Provera 4KN ograničenja — kritično

Cyber Recovery i Cyber Detect OVA fajlovi **ne podržavaju 4KN emulaciju diska** i koriste format sektora od 512 bajtova. Nisu kompatibilni sa okruženjima koja zahtevaju 4KN podršku, kao što su VMFS-6 datastore-ovi formatirani sa blokom veličine 4K.

> **Ako je ciljni datastore formatiran sa 4K blokom, OVA opcija se ne sme koristiti.** Alternativa: softverska instalacija na RHEL ili SLES (`03a-Instalacija-RHEL.md`), ili obezbeđivanje datastore-a sa 512-bajtnim sektorima.

Ovu proveru izvršiti **pre** dogovaranja termina implementacije — naknadno otkrivanje zahteva ponovno planiranje.

## 12b.3 Mrežni podaci koje treba prikupiti

Pre deploymenta obezbediti:

| Podatak | Vrednost |
|---|---|
| IP adresa VM-a | |
| Subnet maska | |
| Default gateway | |
| DNS serveri | |
| FQDN | |
| Vremenska zona | |
| NTP server | |

> Vremensku zonu podesiti tako da vremena u logovima budu tačna.

## 12b.4 Poznata ograničenja OVA deploymenta

| Ograničenje | Detalj | Zaobilaženje |
|---|---|---|
| Jedan mrežni interfejs | Appliance je podrazumevano konfigurisan sa jednim interfejsom | Dodatni virtuelni Ethernet adapteri mogu se dodati nakon deploymenta — **testirani samo za SMTP komunikaciju** (videti 13b.5) |
| 4KN diskovi | Nisu podržani | Softverska instalacija ili datastore sa 512-bajtnim sektorima |
| `--quick-install` | Nije dostupan za OVA | Koristiti `--deploy` |
| Operativni sistem | Fiksiran na SUSE Linux Enterprise Server 15 SP6 | Ako kupac zahteva RHEL standard, koristiti softversku instalaciju |
| Mandatory access control | AppArmor umesto SELinux-a | Procedura izmene SELinux konteksta se **ne primenjuje** |

---

# 13. Deployment virtuelnog appliance-a

Cyber Recovery virtuelni appliance je predkonfigurisana virtuelna mašina koja radi pod SUSE Linux Enterprise Server 15 SP6 i spremna je za deployment na VMware hipervizor.

**Trajanje procedure:** približno pet minuta.

> **Napomena o AppArmor-u:** u OVA deploymentu je podrazumevano instalirana aplikacija AppArmor. Cyber Recovery primenjuje prilagođenu AppArmor politiku na Podman kontejnerske servise, radi kompatibilnosti i dodatne bezbednosti.

---

## 13.1 Deployment OVF šablona

1. U vSphere Client-u **u Cyber Recovery vault-u**, pokrenuti čarobnjak **Deploy OVF Template**.
2. Izabrati preuzeti Cyber Recovery OVA fajl.
3. Izabrati ciljni cluster/host, datastore i port grupu.
4. Za provisioning diskova izabrati **thin provisioning** (preporuka).
5. Dovršiti čarobnjak i sačekati završetak deploymenta.

---

## 13.2 Prvo pokretanje i prijava

1. Kada je deployment završen, otvoriti **vCenter konzolu** novodeploy-ovanog appliance-a.
2. Prijaviti se kao `root` korisnik sa podrazumevanom lozinkom **`changeme`**.

---

## 13.3 Pokretanje instalacije

```bash
# Pokretanje konfiguracije Cyber Recovery softvera
./crsetup.sh --deploy
```

---

## 13.4 Lozinke i passphrase

Na zahtev sistema uneti i potvrditi:

1. **Lockbox passphrase**
2. **Lozinku baze podataka**
3. **Lozinku Security Admin (`crso`) naloga**

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

> ### CAUTION — lockbox passphrase
> **Lockbox passphrase se ne sme zaboraviti; ne može se povratiti.** Ako se zaboravi ili izgubi, potrebna je **sveža instalacija** Cyber Recovery softvera.
> Lockbox passphrase je neophodan za izvršavanje nadogradnji i za reset lozinke Security Admin korisnika.
> Čuvati ga na siguran i redundantan način, **van** Cyber Recovery appliance-a.

---

## 13.5 Rezultat instalacije

Instalaciona procedura pokreće Cyber Recovery servise i zatim izlazi.

- Učitava se fajl `cyber-recovery.service`. Ako se management host restartuje posle gašenja, ovaj fajl usmerava host da automatski pokrene CR servise.
- **Napomena:** u ovom trenutku pune opcije system control-a nisu konfigurisane. Ako pokrenete `systemctl` za `cyber-recovery.service`, status će biti prikazan kao `inactive` — to je očekivano.
- Na kraju instalacione skripte ispisuje se **URL za pristup UI-u** i **spisak portova** koji moraju biti otvoreni na firewall-u. Zabeležiti oba.

---

## 13.6 Obavezna promena podrazumevanih lozinki

> **Ovaj korak se ne sme preskočiti.** Appliance se isporučuje sa podrazumevanim lozinkama `changeme` za `admin` i `root` naloge.

1. Odjaviti se sa Cyber Recovery virtuelnog appliance-a.
2. Preko SSH pristupiti appliance-u kao korisnik **`admin`** sa lozinkom **`changeme`**.
3. Na zahtev sistema **promeniti lozinku**.
4. Komandom `su` preći na `root`.
5. Uneti root lozinku **`changeme`**.
6. Na zahtev sistema **promeniti lozinku**.

```bash
# Sa radne stanice u vault-u
ssh admin@<ip_adresa_appliance-a>
# lozinka: changeme → sistem traži promenu

# Prelazak na root
su
# lozinka: changeme → sistem traži promenu
```

---

# 13b. Post-deploy koraci (OVA)

## 13b.1 Verifikacija Podman mreža

Ako su tokom deploymenta readresirane Podman mreže, proveriti da su bridge interfejsi i mreže `cr_back` i `cr_front` na **odvojenim subnetovima**.

```bash
podman network inspect cr_back  | grep -E 'subnet|gateway'
podman network inspect cr_front | grep -E 'subnet|gateway'

# Provera da kontejneri rade
podman ps --format 'table {{.Names}}\t{{.Status}}' | grep cr_
```

Ako je potrebno naknadno readresiranje, videti proceduru *Readdressing the Podman network* u Delu VI.

> Podman subnetovi se mogu zadati i **tokom samog deploymenta** virtuelnog appliance-a, čime se izbegava naknadno readresiranje. Ako je u fazi preduslova utvrđena kolizija sa vault subnetovima, iskoristiti tu mogućnost.

---

## 13b.2 SELinux — ne primenjuje se

Procedura izmene SELinux konteksta (`chcon`) **ne primenjuje se na OVA deployment**. U virtuelnom appliance-u se kao mehanizam obavezne kontrole pristupa koristi **AppArmor**, sa prilagođenom politikom koju Cyber Recovery primenjuje na Podman kontejnerske servise.

---

## 13b.3 Konfiguracija NTP-a preko chrony

Preporučuje se korišćenje NTP protokola za sinhronizaciju **svih komponenti** u vault-u.

### Provera i instalacija

```bash
# Provera da li je chrony instaliran
rpm -qa chrony
```

> **Za OVA:** ako verzija chrony-ja nije ažurna, instalirati najnoviji Cyber Recovery **`osupdate`** binary da bi se ažurirali paketi na appliance-u.

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
# Provera sinhronizacije sistemskog sata i tačnosti
chronyc tracking

# Provera koji NTP server chrony koristi i da li je dostupan
chronyc sources

# Prinudno usklađivanje — izlaz "200 OK" znači uspeh
chronyc -a makestep

# Potvrda da je NTP servis aktivan
timedatectl
```

> **Podsetnik:** multifactor authentication ne može se omogućiti ako vreme CR hosta odstupa više od ±60 sekundi od vremena authenticator-a. Ako naknadno menjate vreme hosta, zaustavite pa ponovo pokrenite CR servise.

---

## 13b.4 Konfiguracija kaskadne replikacije

Primeniti **samo** ako je deployment konfigurisan za kaskadnu replikaciju — produkcioni podaci se repliciraju u jedan vault, pa zatim u drugi (npr. cleanroom).

Na management serveru u **drugom** vault-u, kao `root`:

```bash
# Za Cyber Recovery virtuelni appliance
touch /var/lib/dellemc/cr/etc/config/airgap_from_vault
```

> **Pažnja:** putanja se razlikuje od softverske instalacije. Za softversku instalaciju koristi se `/opt/dellemc/cr/etc/config/airgap_from_vault`.

> **Napomena o retenciji:** ako i CR sistem u vault-u i CR sistem u cleanroom-u koriste Secure Snapshot-ove, trajanje retencije u cleanroom politikama mora biti **veće ili jednako** trajanju retencije u odgovarajućim vault politikama.

---

## 13b.5 Dodavanje dodatnih virtuelnih Ethernet adaptera (opciono)

Dodatni adapteri se koriste za upravljanje mrežom, razdvajanje saobraćaja i slično. **Testirani su samo za SMTP komunikaciju.**

**Preduslovi:**
- Virtuelni appliance je deploy-ovan na ESXi host u vault-u
- Poznavanje mreža i gateway-a je izričito preporučeno

### Dodavanje adaptera u vSphere-u

1. Pristupiti **VMware vSphere Web Client**-u.
2. **Ugasiti** (power off) virtuelnu mašinu Cyber Recovery appliance-a.
3. Desni klik na VM → **Edit Settings**.
4. U gornjem desnom uglu panela Edit Settings otvoriti padajuću listu **ADD NEW DEVICE**.
5. Pod **Network** izabrati **Network Adapter**. Novi NIC se dodaje u panel.
6. Kliknuti strelicu desno od **New Network** za prikaz opcija konfiguracije.
7. Za **Adapter Type** izabrati **VMXNET** i kliknuti **OK**.
8. **Uključiti** VM.
9. Prijaviti se kao `admin` i komandom `su` preći na `root`.

### Konfiguracija adaptera preko YaST-a

1. Otvoriti **YaST** alat.
2. Otići na **System → Network Settings → Overview**.
3. Izabrati dodatni virtuelni Ethernet adapter i kliknuti **Add**.
4. Uneti detalje adaptera i kliknuti **Next**.
5. Izaći iz YaST-a.
6. Verifikovati da je adapter dodat:

```bash
ip route list
```

> **VAŽNO:** kada se doda dodatni virtuelni Ethernet adapter, **default ruta se postavlja na novi adapter**. Da biste se i dalje mogli prijaviti na originalni sistem, postavite default rutu nazad na originalni virtuelni Ethernet adapter.

---

## 13b.6 Razdvajanje mail i management saobraćaja (opciono)

Opciono, u OVA deploymentu se mail saobraćaj može odvojiti od management saobraćaja, tako da konfiguracija sadrži zasebnu IP adresu za SMTP komunikaciju. Nakon dodavanja adaptera, konfiguracija sadrži dve IP adrese za SMTP komunikaciju.

**Preduslovi:**
- Dodat je drugi virtuelni Ethernet adapter (13b.5)
- Dodatni adapter je konfigurisan preko YaST-a (13b.5)
- Najmanje 20 GB slobodnog prostora nakon raspakivanja CR paketa
- Poznavanje mreža i gateway-a

**Postavljanje rutiranja:**

1. Otvoriti **YaST** alat.
2. Otići na **System → Network Settings → Routing**.
3. U tabeli rutiranja dodati rute ka mail serveru **isključivo preko dodatnog virtuelnog Ethernet adaptera**.

Sav email saobraćaj se preusmerava na port ka mail serveru.

> Za omogućavanje i konfiguraciju podrške za mail server videti *Dell PowerProtect Cyber Recovery Product Guide*. Ako se koristi Postfix, potrebno je otvoriti port 25:
> ```bash
> firewall-cmd --permanent --zone=public --add-port=25/tcp
> firewall-cmd --reload
> ```
> (Na SLES 15 SP6 OVA koristi se `firewalld`, a ne SuSEfirewall2.)

---

## 13b.7 Firewall

Otvoriti portove koje je instalaciona skripta ispisala na kraju izvršavanja. Minimalno:

```bash
firewall-cmd --permanent --add-port=14777/tcp   # CR UI
firewall-cmd --permanent --add-port=14778/tcp   # CR REST API
firewall-cmd --permanent --add-port=14780/tcp   # CR API dokumentacija (opciono)
firewall-cmd --permanent --add-port=25/tcp      # SMTP (ako se koristi Postfix)
firewall-cmd --reload
firewall-cmd --list-all
```

> Puna tabela portova je u Delu II, odeljak 6.1.

---

## 13b.8 Snapshot appliance-a

Nakon završene inicijalne konfiguracije preporučuje se kreiranje snapshot-a virtuelne mašine, kao brza tačka povratka.

> **Obavezno pre svake nadogradnje:** za Cyber Recovery virtuelni appliance kreirati snapshot pre izvršavanja update-a (videti Deo VI).

---

## 13b.9 Backup Cyber Recovery konfiguracije

```bash
./crsetup.sh --save
```

Backup se čuva u `/opt/dellemc/cr-configs`. **Kopirati ga na lokaciju van Cyber Recovery appliance-a.**

> Preferirani način je backup preko CR UI-ja, koji postavlja server DR backup na konfigurisani DD MTree.

---

## 13b.10 Nastavak

Deployment je završen. Nastaviti sa:

- **Poglavlje 14 — Prva prijava i inicijalna konfiguracija** → `03a-Instalacija-RHEL.md`
- **Poglavlje 15 — Osnovna konfiguracija vault-a u CR UI** → `03a-Instalacija-RHEL.md`

Prijava se obavlja na `https://<host>:14777`, korisničko ime `crso`, sa lozinkom kreiranom u koraku 13.4.

> **VAŽNO:** ne prijavljivati se prvo preko CLI-ja — to rezultira greškom `Failed to authenticate user. Please re-login to run commands.` Prva prijava mora biti preko UI-ja.

---

# Razlike u odnosu na softversku instalaciju

Sažetak za brzu orijentaciju:

| Aspekt | Softverska instalacija (RHEL/SLES) | Virtuelni appliance (OVA) |
|---|---|---|
| Operativni sistem | RHEL 8.10/9.4/9.6 ili SLES 16, 15 SP7/SP6/SP5 | SLES 15 SP6 (fiksirano) |
| Komanda za instalaciju | `crsetup.sh --install` ili `--quick-install` | `crsetup.sh --deploy` |
| Trajanje | ~10 minuta | ~5 minuta |
| Mandatory access control | SELinux (preporučeno) | AppArmor (podrazumevano) |
| `chcon` izmena konteksta | **Potrebna** ako je SELinux uključen | **Ne primenjuje se** |
| `libstdc++` | Mora se instalirati ručno | Već uključena |
| rsyslog paketi | Instaliraju se ručno | Preko `osupdate` binary-ja |
| `forwardAuditLogs.conf` | Dodaje se **ručno** | Kopira se automatski u `/etc/rsyslog.d` |
| Podman / Podman Compose | Instalira SE, globalno, nakon firewall-a | Uključeno u appliance |
| Uklanjanje Docker-a | **Potrebno** za bare-metal (20.2/20.3) | Nije potrebno — obavljeno tokom 20.2 |
| Putanja za kaskadnu replikaciju | `/opt/dellemc/cr/etc/config/airgap_from_vault` | `/var/lib/dellemc/cr/etc/config/airgap_from_vault` |
| Mrežni interfejsi | Prema konfiguraciji OS-a | Jedan podrazumevano; dodatni preko vSphere + YaST |
| Podrazumevane lozinke | Nema — postavlja ih SE | `root` i `admin` = `changeme`, **obavezna promena** |
| Instalacioni direktorijum | Izbor SE-a, podrazumevano `/opt/dellemc/cr` | Fiksiran |
| Snapshot pre nadogradnje | N/A | **Obavezno** |
| 4KN datastore | Nije relevantno | **Nije podržano** |

---

# Dodatak: Post-deploy checklist za OVA

| # | Stavka | Status | Napomena |
|---|---|---|---|
| 1 | Verzija vCenter/ESXi je 8.0.x ili 9.0 | | |
| 2 | Datastore **nije** formatiran sa 4K blokom | | **kritično** |
| 3 | Datastore ima ≥ 10 GB za OVA i ≥ 195 GB za diskove | | |
| 4 | Thin provisioning primenjen | | |
| 5 | VM ima 2 vCPU (single core per socket) i 8 GB RAM | | |
| 6 | Deploy OVF Template uspešno završen | | |
| 7 | Mrežni parametri konfigurisani (IP, maska, GW, DNS, FQDN) | | |
| 8 | Vremenska zona podešena | | |
| 9 | `./crsetup.sh --deploy` uspešno završen | | |
| 10 | Lockbox passphrase zabeležen i pohranjen van appliance-a | | **kritično** |
| 11 | Lozinka baze zabeležena | | |
| 12 | `crso` lozinka zabeležena | | |
| 13 | Lozinka `admin` naloga promenjena sa `changeme` | | **obavezno** |
| 14 | Lozinka `root` naloga promenjena sa `changeme` | | **obavezno** |
| 15 | Podman mreže `cr_back` i `cr_front` na odvojenim subnetovima | | |
| 16 | Nema kolizije Podman subnetova sa vault mrežom | | |
| 17 | chrony konfigurisan i sinhronizovan (`chronyc tracking`) | | |
| 18 | `osupdate` primenjen ako je chrony/rsyslog zastareo | | |
| 19 | Portovi sa kraja instalacione skripte otvoreni na firewall-u | | |
| 20 | `airgap_from_vault` fajl kreiran na `/var/lib/...` putanji | | samo kaskadna replikacija |
| 21 | Dodatni Ethernet adapter dodat i konfigurisan | | ako se koristi |
| 22 | Default ruta vraćena na originalni adapter | | ako je dodat adapter |
| 23 | Rutiranje mail saobraćaja podešeno u YaST-u | | ako se razdvaja saobraćaj |
| 24 | Snapshot VM-a kreiran nakon konfiguracije | | |
| 25 | CR backup kreiran i pohranjen van appliance-a | | `crsetup.sh --save` ili UI |
| 26 | UI dostupan na `https://<host>:14777` | | |
| 27 | Prva prijava kao `crso` uspešna (preko UI-ja, ne CLI-ja) | | |

> Nakon ove tačke nastaviti sa poglavljima **14** i **15** u `03a-Instalacija-RHEL.md`, gde se nalazi checklist za inicijalnu konfiguraciju u CR UI (stavke 21–29 tamošnjeg dodatka).

---

*Kraj Dela III-b. Videti i: `03a-Instalacija-RHEL.md` (softverska instalacija, poglavlja 14 i 15), Deo IV (integracija sa NetWorker-om), Deo V (integracija sa PPDM-om).*
