# DEO IV — INTEGRACIJA SA NETWORKER-om

**Dell PowerProtect Cyber Recovery 20.3 — Uputstvo za instalaciju i integraciju**
Verzija dokumenta: 0.1 | Namena: System Engineer

> **Izvori:**
> - *Dell PowerProtect Cyber Recovery 20.3 Installation and Upgrade Guide*
> - *Dell PowerProtect Cyber Recovery 20.3 Product Guide*
> - *Dell NetWorker Server Disaster Recovery and Availability 19.13 and later — Best Practices Guide* (Rev. 01, jun 2025)
>
> **Preduslov:** završen Deo II (preduslovi) i Deo III (instalacija), uključujući dodat vault storage i funkcionalnu politiku.

---

## Sadržaj

- [16. Planiranje integracije](#16-planiranje-integracije)
- [17. Priprema produkcijskog NetWorker okruženja](#17-priprema-produkcijskog-networker-okruženja)
- [18. Priprema NetWorker instance u vault-u](#18-priprema-networker-instance-u-vault-u)
- [19. Konfiguracija NetWorker-a u Cyber Recovery](#19-konfiguracija-networker-a-u-cyber-recovery)
- [20. Kreiranje DD Boost korisnika i UID-a u vault-u](#20-kreiranje-dd-boost-korisnika-i-uid-a-u-vault-u)
- [21. Izvršavanje NetWorker recovery-ja](#21-izvršavanje-networker-recovery-ja)
- [22. Recovery Check i validacija](#22-recovery-check-i-validacija)
- [23. Čišćenje i post-recovery koraci](#23-čišćenje-i-post-recovery-koraci)
- [24. Referenca `nsrdr` komande](#24-referenca-nsrdr-komande)
- [Dodatak A: Checklist NetWorker integracije](#dodatak-a-checklist-networker-integracije)
- [Dodatak B: Otvorena pitanja](#dodatak-b-otvorena-pitanja)

---

# 16. Planiranje integracije

## 16.1 Kako Cyber Recovery oporavlja NetWorker

Cyber Recovery ne replicira NetWorker aplikaciju — replicira **MTree** na kojem NetWorker čuva svoju bazu i podatke. U vault-u se zatim pokreće automatizovana procedura koja:

1. Kreira **recovery sandbox** — jedinstvenu lokaciju u vault-u sa read/write kopijom zaključanih podataka.
2. Popunjava sandbox izabranom point-in-time (PIT) kopijom.
3. Izlaže sandbox NetWorker aplikativnom hostu u vault-u.
4. Pokreće NetWorker `nsrdr` proceduru koja oporavlja bootstrap i client file indekse.

> **Zašto root:** Cyber Recovery koristi NetWorker komande poput `nsrdr`, koje zahtevaju root privilegije. Zato se NetWorker na Linux-u u vault-u dodaje kao `root` korisnik.

## 16.2 Šta sadrži bootstrap

Prema *NetWorker Server Disaster Recovery Best Practices Guide*, bootstrap save set sadrži pet komponenti koje se nalaze na NetWorker serveru:

| Komponenta | Sadržaj |
|---|---|
| **Media database** | Lokacija volumena za svaki save set |
| **Resource files** | Svi resursi definisani na NetWorker serveru (klijenti, backup grupe itd.) |
| **License server files** | `dpa.lic` i `licspec.properties` (`dpa.lic` se nalazi u `/nsr/lic`) |
| **NetWorker Authentication Service database** | `authcdb.h2.db` |
| **Lockboxes** | Poverljive informacije u enkriptovanom formatu (npr. Oracle client lozinke, DD Boost lozinka) |

**Client file indexes (CFI)** su odvojeni save set-ovi. Za svaki klijent postoji direktorijum indeksa u `nsr/index` na NetWorker serveru. CFI sadrži ime fajla, tip fajla, save time, veličinu (samo UNIX) i atribute fajla. CFI nije uvek neophodan za oporavak podataka, ali bitno utiče na brzinu vraćanja NetWorker servera u potpuno funkcionalno stanje.

> **Bootstrap backup je jedini garantovani način** da se konfiguracija NetWorker servera bezbedno i konzistentno uhvati. Repliciranje konfiguracionih fajlova može dati crash-consistent stanje.

## 16.3 Podržane verzije i ograničenja

| Stavka | Vrednost |
|---|---|
| Podržana verzija NetWorker-a (produkcija i vault) | **19.10 i novije** |
| Minimalna verzija za Recovery Check | **19.9 i novija** |
| Platforme | Linux i Windows (Windows uz Cygwin) |
| Verzija u vault-u | Mora biti **identična** verziji na produkciji |

### Ograničenja koja treba komunicirati kupcu unapred

| Ograničenje | Detalj |
|---|---|
| **Više MTree-ova** | Cyber Recovery **ne podržava automatski** oporavak NetWorker servera sa više od jednog MTree-a. Ručni oporavak je moguć — kontaktirati Dell Support. |
| **Paralelni recovery** | Može se izvršavati **samo jedan recovery job po aplikaciji istovremeno**. |
| **Postojeći sandbox** | Padajuća lista u UI-u **ne prikazuje** NetWorker aplikativni host za koji već postoji sandbox. |
| **Mount na Windows-u** | Za NetWorker na Windows-u **mount operacija nad sandbox-om nije podržana**. |
| **vDisk i VTL** | Podržana je samo Sync operacija. |
| **jobsdb** | Server Protection politika **ne backup-uje** `jobsdb`. Posle DR-a `jobsdb` je prazan, svi prethodni statusi workflow-ova i akcija su izgubljeni, status prelazi u `Never Run`. |
| **NMC baza** | NMC ima **zasebnu bazu**, izolovanu od serverdb. `nsrdr` je ne oporavlja — potrebno je posebno pokrenuti `recoverpsm`. |

## 16.4 Tok integracije — pregled

```
PRODUKCIJA                          VAULT
──────────                          ─────
NetWorker server                    NetWorker server (neinicijalizovan,
  │                                   ista verzija, isti hostname)
  │ backup + bootstrap                       ▲
  ▼                                          │ nsrdr
Produkcioni DD                               │
  │ MTree                          Recovery sandbox
  │                                          ▲
  │  MTree replikacija                       │ populate
  └──────────────────────────────►  Vault DD (PIT kopija, locked)
                                             ▲
                                             │ Sync → Copy → Lock
                                    Cyber Recovery politika
```

---

# 17. Priprema produkcijskog NetWorker okruženja

Ovo je najčešće previđen deo implementacije. Ako produkcijska strana nije ispravno pripremljena, kopija u vault-u je neupotrebljiva za oporavak.

## 17.1 DD Boost korisnik na produkcionom DD sistemu

```bash
# role za NetWorker je "admin"
sysadmin@dd-prod# user add <networker_ddboost_user> role admin
sysadmin@dd-prod# ddboost user assign <networker_ddboost_user>
```

**Zabeležiti UID ovog naloga** — biće potreban u vault-u (poglavlje 20):

```bash
sysadmin@dd-prod# user show list
```

## 17.2 Konfiguracija Server Protection politike

Pri instalaciji ili nadogradnji NetWorker servera automatski se kreira podrazumevana **Server Protection** politika koja backup-uje NetWorker server i NMC bazu.

Podrazumevano ponašanje **Server backup** workflow-a:

| Parametar | Podrazumevana vrednost |
|---|---|
| Vreme pokretanja | 10:00 |
| Prvi dan u mesecu | Full backup |
| Ostali dani | Inkrementalni backup |
| Grupa | Server Protection group (dinamički generisana lista Client resursa za NetWorker server i NMC server) |

> **Provera koju SE mora izvršiti:** da li je politika omogućena, da li se izvršava uspešno i da li piše na MTree koji se replicira u vault. Podrazumevani raspored od jednom dnevno je minimum — proveriti da odgovara RPO zahtevu kupca.

### Kreiranje Server Backup akcije (ako je potrebna nova)

Server Backup akcija izvršava bootstrap backup NetWorker media i resource baza i može uključiti i client file indekse. Akcija mora biti **prva akcija u workflow-u**.

Ključna polja u **Policy Action** čarobnjaku:

| Polje | Napomena za CR implementaciju |
|---|---|
| **Action Type** | `Server Backup` |
| **Destination Storage Node** | Storage node sa uređajima na kojima se čuva backup |
| **Destination Pool** | **Namenski pool za bootstrap** (vidi 17.3) |
| **Retention** | Trajanje čuvanja backup podataka |
| **Perform CFI** | Uključuje client file indekse u server backup |
| **Perform Bootstrap** | Uključuje bootstrap backup |

> **Obavezno:** mora biti označen **Perform CFI**, **Perform Bootstrap**, ili oba. U suprotnom server backup akcija **ne backup-uje ništa**.

> NetWorker podržava **samo jednu akciju posle** server backup akcije (npr. clone ili expire).

## 17.3 Preporuke za bootstrap (iz Best Practices Guide-a)

| Preporuka | Obrazloženje |
|---|---|
| **Bootstrap backup najmanje jednom u 24 sata** | Minimalni preduslov za uspešan DR |
| **Namenski (dedicated) pool za bootstrap** | Ubrzava oporavak i sprečava zavisnost od klijentskih volumena sa neodgovarajućim politikama |
| **Ne mešati bootstrap save set sa klijentskim backup podacima** | Isti razlog |
| **Redovno klonirati bootstrap volumene** | Otkaz ili gubitak jednog medija ne sme onemogućiti oporavak |
| **Čuvati zapis o bootstrap save set-u** | Datum i vreme backup-a, ime i lokacija volumena, Save set ID (SSID), početni file i record broj na volumenu |
| **Backup OS konfiguracije servera redovno** | |
| **Beležiti i održavati podatke o SAN-u, IP-u i svim storage komponentama** | |
| **Čuvati status i sadržaj svakog bootstrap backup-a na fizički odvojenoj lokaciji** | |

> Za Cyber Recovery deployment, replikacija u vault delimično preuzima ulogu offsite kopije — ali **ne zamenjuje** disciplinu vođenja evidencije o bootstrap-u, jer je SSID potreban i pri ručnom oporavku.

## 17.4 Ručni backup NetWorker servera

Pre prve validacije integracije korisno je pokrenuti backup ručno:

```bash
# Full backup backup servera
nsrpolicy start -p <server_protection> -w <server_backup>

# Praćenje statusa politike
nsrpolicy monitor -p <server_protection> -w <server_backup>

# Preuzimanje informacija o najnovijem bootstrap-u
mminfo -B
```

> Najnovije bootstrap informacije čuvati na sigurnom mestu radi kasnije upotrebe u DR-u.

## 17.5 Kako pronaći bootstrap informacije

Tri načina, prema Best Practices Guide-u:

**1. Iz log fajla notifikacija**

```
nsr/logs/policy_notifications.log
```

Sekcija *Server backup Action report* sadrži informacije o bootstrap i CFI backup-ima. Primer strukture izlaza:

```
---Server backup Action report---
Policy name:Server Protection
Workflow name:Server backup
Action name:Server db backup
Action status:succeeded
--- Successful Server backup Save Sets ---
<ssid>/<savetime> <server>: index:<client> level=1, ...
<ssid>/<savetime> <server>: bootstrap level=full, ...
--- Bootstrap backup report ---
date time level ssid file record volume
```

Poslednji red u sekciji *Bootstrap backup report* daje SSID i volumen najnovijeg bootstrap-a.

**2. Preko `mminfo`** (ako media baza nije izgubljena i lista volumena je dostupna)

```bash
mminfo -av -B -s <server_name>
```

**3. Preko `nsrdr`** — komanda skenira uređaj i za postojeće uređaje detektuje najnoviji bootstrap na volumenu.

## 17.6 Usklađivanje rasporeda replikacije

> **Kritično pravilo iz CR dokumentacije:** početak replikacionog prozora mora biti **posle** završetka aplikativnog backup-a na produkcionom sistemu, uključujući potreban metadata backup. Ako se koristi međureplikacija na produkcionoj strani, i njen početak mora biti posle završetka produkcionog backup-a.

Praktično znači: pre definisanja Sync rasporeda u Cyber Recovery politici, utvrditi kada Server Protection workflow tipično završava (uz rezervu za varijacije trajanja) i tek onda postaviti Sync.

**Radni list:**

| Stavka | Vrednost |
|---|---|
| Vreme pokretanja Server backup workflow-a | |
| Tipično trajanje | |
| Najduže zabeleženo trajanje | |
| Predloženo vreme početka Sync operacije | |
| Rezerva | |

## 17.7 Bootstrap device folder

Pri pokretanju recovery-ja u CR UI-u može se opciono uneti ime foldera koji sadrži poslednje bootstrap backup-e. Ako se polje ne popuni, softver skenira **sve volumene u MTree-u**, što je znatno sporije.

**Zabeležiti u fazi pripreme:** ime foldera sa bootstrap backup-ima na produkcionom uređaju.

---

# 18. Priprema NetWorker instance u vault-u

## 18.1 Zahtevi za instancu

| Zahtev | Detalj |
|---|---|
| **Verzija** | Identična verziji na produkcionom sistemu |
| **Stanje** | **Neinicijalizovana** instanca |
| **Dimenzionisanje** | Ispravno dimenzionisana za izvršavanje `nsrdr` operacije nad podacima repliciranim sa produkcionog na vault DD |
| **Namena** | Isključivo za oporavak u vault-u |

> **Neinicijalizovana** znači sveža instalacija bez konfigurisanih klijenata, uređaja i politika. Recovery procedura zamenjuje resource i media baze podacima iz bootstrap-a.

## 18.2 Instalacija operativnog sistema i NetWorker-a

1. Instalirati istu verziju operativnog sistema koja je na produkcionom hostu.
2. Konfigurisati OS sa istom IP adresom i hostname-om (shortname i FQDN) — videti 18.3.
3. Instalirati sve zakrpe / nadograditi OS na isti nivo kao pre.
4. Instalirati **istu verziju** NetWorker Server softvera na **originalnu lokaciju**.
   - Obavezno instalirati pakete: **NetWorker client, storage node i Authentication service**.
5. Pokrenuti konfiguracionu skriptu:

```bash
/opt/nsr/authc-server/scripts/authc_configure.sh
```

6. Instalirati sve NetWorker zakrpe koje su bile instalirane pre.
7. Imenovati NetWorker server **istim imenom** — nova instalacija mora imati isti fully qualified name.
8. Imenovati shortname uređaja isto kao pre.

> Na Linux-u nije potrebno ponovo učitavati license enabler-e ako NetWorker konfiguracioni fajlovi postoje. Podrazumevano se nalaze u `/nsr/res/nsrdb`.

## 18.3 IP adresa i hostname

**Preporuka:** NetWorker server host u vault-u treba da ima **istu IP adresu i hostname** kao produkcioni NetWorker host.

> Nije obavezno da IP adresa bude ista. Međutim, ako se koristi različita IP adresa, mogu se javiti problemi sa komponentama i agentima koji se na NetWorker server pozivaju preko IP adrese — što zahteva ručnu intervenciju. Ista IP adresa te probleme izbegava.

**Napomena za SE-a:** ovo je dizajnerska odluka koja utiče na mrežnu segmentaciju vault-a. Ako se koristi ista IP adresa kao u produkciji, mora se osigurati da vault mreža nikada nije istovremeno rutabilna ka produkciji — inače nastaje konflikt adresa. Uskladiti sa air-gap dizajnom.

## 18.4 Rešavanje hostname-ova

Iz Best Practices Guide-a: problemi sa rezolucijom hostname-ova mogu učiniti NetWorker server neodgovarajućim ili vrlo sporim pri pokretanju. U vault-u, gde DNS često nije dostupan ili je ograničen, ovo je realan scenario.

**Zaobilazno rešenje:**

1. Onemogućiti DNS lookup za host koji se oporavlja i koristiti lokalni `hosts` fajl:
   - Linux: `/etc`
   - Windows: `C:\Windows\System32\Drivers\etc\`

2. Izmeniti `/etc/nsswitch.conf` tako da se `hosts` fajlovi konsultuju pre DNS-a:

```
# Umesto:
hosts: dns files

# Postaviti:
hosts: files
```

3. Popuniti lokalni `hosts` fajl poznatim, važećim IP adresama klijenata. Za klijente čija je IP adresa nepoznata koristiti `127.0.0.1`.

> `127.0.0.1` je standardna loopback adresa. Kada se NetWorker server podigne, izvršava se DNS provera za svakog klijenta. Za klijente koji su offline ili nedostupni, server se povezuje na `127.0.0.1` što se odmah vraća na isti host. Time server postaje dostupan brže, umesto da čeka rezoluciju svih klijenata.

4. Format `hosts` fajla: FQDN prvi, zatim shortname.

> Ako je client name NetWorker servera originalno bio shortname, onda shortname mora biti prvi, pa longname.

5. Kada DNS server bude dostupan, ponovo omogućiti DNS lookup i ukloniti detalje klijenata iz lokalnog `hosts` fajla.

**Alternativa — onemogućavanje popunjavanja DNS keša:**

`nsrdr` kreira fajl `nsr_disaster_recovery_mode` u direktorijumu `/nsr/debug`. Ako NetWorker server pronađe taj fajl, preskače popunjavanje DNS keša i RAP consistency provera za klijente. Kada `nsrdr` završi ili izađe zbog greške, fajl briše sam. **Ako dođe do pada i `nsrdr` ne izađe uspešno, fajl treba obrisati ručno** iz `/nsr/debug`. Posle završenog DR-a restartovati NetWorker server da se DNS keš ispravno popuni.

## 18.5 NetWorker na Linux-u

- Aplikacija se u Cyber Recovery dodaje kao **`root`** korisnik.
- Uveriti se da je SSH pristup sa CR management hosta ka NetWorker hostu funkcionalan (port 22).

> **Napomena iz Best Practices Guide-a:** pri pokretanju `nsrdr` ka **drugom** NetWorker serveru na Linux platformi, korisnički ID koji koristi `nsrtomcat` korisnik mora biti **isti** na originalnom i na ciljnom NetWorker serveru. Proveriti pre implementacije:

```bash
id nsrtomcat
```

## 18.6 NetWorker na Windows-u

### 18.6.1 Osnovni zahtevi

- Aplikacija se u Cyber Recovery dodaje kao **`Admin`** korisnik.
- Potreban je Windows host u vault-u sa omogućenim Remote Desktop-om sa lokalnog servera.
- **Cygwin je obavezan** za automatizovani DR NetWorker-a na Windows-u.
- Ako deployment već uključuje SSH, onemogućiti ga da se ne pokreće automatski (izbegava se konflikt sa Cygwin sshd).
- Mount operacija nad NetWorker sandbox-om nije podržana.

### 18.6.2 Instalacija i konfiguracija Cygwin-a

**Korak 1 — instalacija**

Instalirati Cygwin i konfigurisati SSH prema uputstvu *Installing and Updating Cygwin Packages*. Dodati paket **OpenSSH**.

**Korak 2 — konfiguracija sshd servisa**

Otvoriti Cygwin terminal i pokrenuti:

```bash
$ ssh-host-config
```

Odgovori na upite prema primeru iz dokumentacije:

```
*** Info: Generating missing SSH host keys
*** Info: Creating default /etc/ssh_config file
*** Info: Creating default /etc/sshd_config file
*** Query: Should StrictModes be used? (yes/no)  no
*** Query: Do you want to install sshd as a service?
*** Query: (Say "no" if it is already installed as a service) (yes/no)  yes
*** Query: Enter the value of CYGWIN for the daemon: []
*** Info: The sshd service has been installed under the LocalSystem account
```

> `StrictModes` se postavlja na `no` jer podrazumevane Windows dozvole za home direktorijum montiran sa `noacl` opcijom nisu kompatibilne sa strogim režimom.

**Korak 3 — verifikacija ključeva**

```bash
$ ls -tlr /etc/ssh_*
```

Očekuju se privatni i javni ključevi (`ssh_host_rsa_key`, `ssh_host_dsa_key`, `ssh_host_ecdsa_key`, `ssh_host_ed25519_key` i odgovarajući `.pub` fajlovi) i `ssh_config`.

**Korak 4 — pokretanje servisa**

Pokrenuti Cygwin `sshd` servis:

```
net start cygsshd
```

**Korak 5 — otvaranje porta 22 na Windows firewall-u**

Iz PowerShell sesije:

```powershell
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
```

**Korak 6 — provera SSH pristupa**

Pristupiti Windows hostu preko SSH sa CR management hosta.

**Korak 7 — podešavanje putanje do NetWorker instalacije**

U fajlu `~/.bashrc`, **pre** linije `# If not running interactively, don't do anything`, dodati:

```bash
cd <NetWorker installation path>
```

> Ova linija saopštava Cyber Recovery softveru lokaciju NetWorker instalacije na Windows hostu pri pokretanju SSH-a bez interaktivne ljuske. Bez nje automatizovani recovery ne uspeva.

## 18.7 Podešavanje retention policy za NetWorker server klijent

Iz Best Practices Guide-a, korak koji se izvršava u vault-u pre recovery-ja:

1. U NetWorker Administration prozoru → **Protection** → **Clients**.
2. Pod tabom **View** izabrati **Diagnostic mode** da bi se prikazao atribut `Retention policy`.
3. Desni klik na client resurs NetWorker servera → **Modify Client Properties**.
4. Na tabu **General**, atribut **Retention policy** postaviti na **Decade**.

> Podrazumevana retencija je mesec dana. Postavljanjem na `Decade` omogućava se oporavak **svih** zapisa u fajlovima baze. Ako se retencija ne promeni, save set-ovi NetWorker servera sa retencijom dužom od mesec dana se odbacuju, jer je podrazumevana browse politika mesec dana.

5. Na tabu **Globals (1 of 2)** proveriti da atribut **Aliases** sadrži ispravne hostname-ove NetWorker servera — i shortname i FQDN.

## 18.8 NetWorker 19.8 — stanje disaster recovery

Za deployment koji koristi NetWorker verziju 19.8, stanje NetWorker servera mora biti postavljeno na **disaster recovery**.

> Napomena: CR 20.3 zvanično podržava NetWorker 19.10+, pa se ovaj korak odnosi na zatečena starija okruženja.

Iz Best Practices Guide-a, o disaster recovery stanju uopšte: da bi se media baza sigurno oporavila, server se postavlja u disaster recovery stanje izmenom server state u NSR resursu. Nakon prelaska u to stanje:
- Svi workflow-ovi u toku se zaustavljaju; nastavljaju se samo ručno pokrenuti udaljeni poslovi.
- Ne mogu se izvršavati aktivnosti zaštite podataka (save, clone, index management, `nsrmm` operacije nad save set-ovima).
- Recover operacija nastavlja da radi. `nsrmmdbasm` se može pokretati u disaster recovery stanju.

## 18.9 Nadogradnja NetWorker-a u vault-u

Kada je potrebno uskladiti verziju sa produkcijom:

| Platforma | Postupak |
|---|---|
| **Linux** | Preuzeti željenu `tar.gz` verziju NetWorker softvera, raspakovati je, pa pokrenuti `rpm -U` za nadogradnju NetWorker binarnih fajlova |
| **Windows** | Preuzeti i raspakovati novu verziju, pokrenuti instalater i izabrati **Upgrade** |

**Uticaj na Cyber Recovery:**
- Nadogradnja je disruptivna za NetWorker server — servisi se zaustavljaju i pokreću.
- Cyber Recovery softver nastavlja da radi uz ograničenu smetnju.
- **Automatizovani NetWorker recovery proces ne radi** tokom nadogradnje; sve ostale CR operacije rade normalno.

> **Redosled:** ako se planira i nadogradnja Cyber Recovery-ja, prvo nadograditi Dell backup aplikacije i komponente trećih strana, pa tek onda Cyber Recovery softver. Pre toga: nema aktivnih job-ova, zaustaviti sve CR servise, izvršiti nadogradnju, pokrenuti CR servise, sinhronizovati vreme (chrony).

---

# 19. Konfiguracija NetWorker-a u Cyber Recovery

## 19.1 Replication kontekst

Kreirati MTree replication kontekst sa produkcionog na vault DD sistem za NetWorker MTree.

- Za novi replikacioni par, port **3009** mora biti otvoren između produkcionog i vault DD sistema. **Zatvoriti ga nakon dodavanja para.**
- Izvršiti **inicijalnu replikaciju** pre definisanja Cyber Recovery politike.
- Ako je FIPS omogućen na bilo kom od DD sistema, konfigurisati dvosmernu autentikaciju za kontekst.

## 19.2 Dodavanje NetWorker aplikacije kao asset-a

**Main Menu → Infrastructure → Assets → Applications → Add**

### Stranica 1 — Application Information

| Polje | Vrednost za NetWorker |
|---|---|
| **Application Type** | `NetWorker` |
| **Nickname** | Ime aplikativnog objekta |
| **FQDN or IP Address** | FQDN ili IP adresa NetWorker hosta u vault-u |
| **Tags** | Opciono. **Za NetWorker recovery: dodati tag koji označava DD Boost username konfigurisan za produkcionu aplikaciju.** |

**Napomene:**
- Ako se unese FQDN ili IP adresa koja pripada drugom tipu aplikacije, procedura ne uspeva. Host mora odgovarati tipu aplikacije.
- **Polje Application Type se ne može menjati za postojeću aplikaciju.**
- Ako se izmeni postojeći FQDN ili IP, sistem traži ponovni unos lozinke.
- Tag duži od 24 znaka prikazuje se skraćen na 21 znak sa tri tačke.

> **Zašto je tag sa DD Boost username-om bitan:** pri kreiranju UID-a u vault-u (poglavlje 20) tag služi kao referenca za utvrđivanje produkcionog DD Boost korisničkog imena. Bez njega SE mora tražiti taj podatak po produkcionoj dokumentaciji.

### Stranica 2 — Host Authentication

| Polje | Vrednost |
|---|---|
| **Username** | Korisničko ime administratora operativnog sistema hosta u vault-u.<br>Linux: **`root`**<br>Windows: **`Admin`** |
| **Password** | Lozinka host administratora |
| **SSH Port Number** | SSH port aplikacije (podrazumevano 22) |
| **Reset Host Fingerprint** | Samo Security Admin. Označiti ako je promenjen FQDN/IP hosta — CR tada šalje alert. |

> Ako se izmeni username, sistem traži ponovni unos lozinke.

Dovršiti čarobnjak i sačuvati.

## 19.3 Kreiranje Standard politike za NetWorker

**Main Menu → Policies → Add**

| Stranica | Vrednosti za NetWorker |
|---|---|
| **Policy Information** | Policy Type: **`Standard`** (obuhvata NetWorker, Avamar, Filesystem, Other). Izabrati Storage Target sa NetWorker replication kontekstom. |
| **Replication** | Izabrati NetWorker replication kontekst i **namenski replikacioni Ethernet port**. Ne birati data ili management interfejse. Standard politika koristi **jedan** data replication kontekst. |
| **Retention Lock** | `Compliance`, `Governance` ili `None` — prema dizajnu. Kod DD sistema sa Secure Snapshot podrškom ove opcije se ne prikazuju za nove politike. |
| **Scheduling** | Min Retention Lock Period (min. 12 sati), Max (max. 5 godina za RL / 180 dana za Secure Snapshot), Retain for. Dodati rasporede. |
| **Storage Security Credentials** | Samo za Compliance — kredencijali DD Security Officer-a. |

### Preporučeni raspored operacija

| Operacija | Napomena |
|---|---|
| **Sync** | **Posle** završetka Server Protection workflow-a na produkciji (videti 17.6) |
| **Copy** | Posle završetka Sync-a |
| **Lock** | Nad kreiranom PIT kopijom |
| **Recovery Check** | Opciono, zakazano — videti poglavlje 22 |
| **Analyze** | Ako je Cyber Detect u obimu |

## 19.4 Konfiguracija zakazanog Recovery Check-a

Videti poglavlje 22.1.

---

# 20. Kreiranje DD Boost korisnika i UID-a u vault-u

> **Ovo je najčešći uzrok neuspelog NetWorker recovery-ja.** UID DD Boost korisnika u vault-u mora biti **identičan** UID-u produkcionog DD Boost korisnika.

## 20.1 Utvrđivanje potrebnog UID-a

Na CR management hostu:

```bash
# Prijava u CRCLI
crcli login -u <Cyber_Recovery_korisnik>

# Prikaz informacija o kopiji
crcli policy list-copy --policyname <ime_politike> -c <ime_kopije>
```

U izlazu potražiti:

```
Source Storage UID: 503
```

Zabeležiti tu vrednost.

## 20.2 Provera postojanja naloga u vault-u

Prijaviti se na DD sistem u vault-u:

```bash
sysadmin@dd-vault# user show list
```

- Ako izlaz **sadrži** traženi UID → nastaviti sa recovery procedurom (poglavlje 21).
- Ako izlaz **ne sadrži** UID → nastaviti sa 20.3.

## 20.3 Kreiranje naloga sa tačnim UID-om

1. Utvrditi produkciono DD Boost korisničko ime — ako je pri dodavanju aplikacije definisan tag, iskoristiti ga kao referencu.
2. Kreirati nalog:

```bash
sysadmin@dd-vault# user add <NetWorker_ddboostname> uid <UID_iz_user_show_list>
```

## 20.4 Tehnika privremenih naloga (starije verzije)

Kod starijih verzija DD OS-a, komanda `user add` dodeljuje UID-ove sekvencijalno počev od **500**. Ako je potreban npr. UID 510, može biti neophodno kreirati do devet privremenih naloga da bi se došlo do željene vrednosti.

> **Nakon dostizanja željenog UID-a obrisati privremene naloge.**

## 20.5 Preporuka za planiranje UID opsega

Iz CR dokumentacije, dobra praksa pri kreiranju DD naloga:

| UID opseg | Namena |
|---|---|
| **501–599** | **Rezervisati** za korisnike koji se automatski kreiraju tokom recovery operacija ili koji moraju odgovarati UID-ovima produkcionih DD Boost korisnika |
| **600–700** | Ručno kreirani DD korisnici specifični za Cyber Recovery: `cradmin`, DD Security Officer, Cyber Detect korisnici, DD admin nalozi za auditing |

> Ako se ova konvencija primeni od početka, izbegava se kolizija između CR administrativnih naloga i UID-ova koje recovery procedura mora da reprodukuje.

---

# 21. Izvršavanje NetWorker recovery-ja

## 21.1 Checklist preduslova

Proveriti **sve** stavke pre pokretanja:

| # | Preduslov |
|---|---|
| 1 | Prijavljeni ste kao **Admin** ili **Vault Operator** korisnik |
| 2 | Za NetWorker 19.8: stanje NetWorker servera postavljeno na **disaster recovery** |
| 3 | Imate kredencijale za vault host na kojem je NetWorker instaliran **i** za samu NetWorker aplikaciju |
| 4 | NetWorker server host u vault-u ima **istu IP adresu i hostname** kao produkcioni host (preporuka — videti 18.3) |
| 5 | NetWorker aplikacija je instalirana u vault-u i definisana kao application asset u Cyber Recovery |
| 6 | DD Boost korisnik u vault-u ima **isti UID** kao produkcioni DD Boost korisnik |
| 7 | Politika je kreirala **PIT kopiju** koja će se koristiti za oporavak |
| 8 | UID povezan sa tom kopijom je kreiran na vault DD sistemu |
| 9 | Za NetWorker na Windows-u: Windows host i Cygwin instalirani u vault-u, Cygwin OpenSSH omogućen |
| 10 | NetWorker server u vault-u ima samo **jedan MTree** (automatski recovery) |
| 11 | Nema drugog recovery job-a u toku za istu aplikaciju |

## 21.2 Pokretanje recovery-ja iz CR UI

1. Iz **Main Menu** izabrati **Recovery**.
2. U **Recovery** panelu kliknuti **Application**.

   > **Ako birate Windows kopiju, obavezno izaberite aplikaciju NetWorker on Windows.** Ako izaberete kopiju koja ne odgovara operativnom sistemu, recovery operacija ne uspeva.

3. Iz padajuće liste izabrati **NetWorker** aplikaciju.

   > Padajuća lista **ne prikazuje** NetWorker aplikativni host za koji već postoji sandbox. Ako aplikacija nije na listi, proveriti postojeće sandbox-ove i po potrebi ih očistiti.

4. Uneti **DD Boost username** i **password**.
5. Opciono uneti ime foldera koji sadrži poslednje bootstrap backup-e.

   > Ako se polje ne popuni, softver skenira sve volumene u MTree-u. Popunjavanjem ovog polja automatizovani NetWorker recovery je **brži**.

6. Kliknuti **Apply**.

Status kopije prelazi u **In Progress**.

## 21.3 Praćenje job-a

Cyber Recovery pokreće job koji:
1. Kreira recovery sandbox.
2. Popunjava ga izabranom kopijom.
3. Izlaže sandbox aplikativnom hostu.

Koraci praćenja:

1. Sačekati da se recovery application job završi sa kreiranjem sandbox-a.
2. Kliknuti na ime job-a **`recoverapp_<ID>`** i pogledati **Status Detail**.
   - Status Detail sadrži ime novokreiranog sandbox-a.
3. Kliknuti **Recovery Sandboxes** na vrhu Recovery panela.

Rezultat: recovery sandbox je kreiran za NetWorker aplikaciju; oporavljena je najnovija NetWorker konfiguracija.

## 21.4 Rad sa Recovery Sandbox-ovima

**Main Menu → Recovery → Recovery Sandboxes**

| Radnja | Opis |
|---|---|
| Izabrati `recoverapp_<ID>` | Prikaz detalja oporavka |
| **Launch App** | Pristup NetWorker UI-u u vault-u radi validacije. **Dugme je aktivno samo ako je oporavak uspešno završen.** |
| **Cleanup** | Brisanje sandbox-a |
| **Export** | Izvoz informacija o svim recovery sandbox-ovima u `recoverysandboxes.csv` |

> **Napomena:** ako je aplikacija povezana sa recovery sandbox-om, ne može se obrisati. Pokušaj brisanja daje poruku o grešci.

## 21.5 Opcioni ručni koraci

Ovi koraci **nisu deo** automatizovane recovery procedure, ali mogu biti potrebni u određenim scenarijima:

```bash
# Popunjavanje oporavljene media baze najnovijim save set-ovima
# Pokrenuti na SVAKOM uređaju kreiranom tokom oporavka
scanner -i <device name>

# Ponovna izgradnja client file indeksa
# Neophodno za pregledanje fajlova i oporavak baza podataka
nsrck -L7
```

> Za detalje o tome kada su ove komande potrebne videti *NetWorker Server Disaster Recovery Best Practices Guide*.

### Kada je `scanner -i` zaista potreban

Iz Best Practices Guide-a: ručna save operacija je **jedini** način da save set bude backup-ovan bez pokretanja backup-a CFI podataka. Ako je ručni backup izvršen pre sledećeg zakazanog backup-a (koji uvek backup-uje bootstrap i CFI), poslednji CFI **neće imati zapis** o ručno backup-ovanim save set-ovima.

> `scanner -i` može trajati **veoma dugo**, posebno na velikom disk volumenu. Za volumene za koje ne sumnjate da imaju save set-ove backup-ovane posle poslednjeg bootstrap-a, korak se može preskočiti.

### Rad sa Scan Needed zastavicom

Ako je korišćena `nsrdr -N` opcija, svi appendable (non read-only) volumeni u oporavljenoj media bazi se označavaju kao **Scan Needed**.

**Za AFTD i DD volumene:**

```bash
# 1. Ako ne znate ime uređaja koje odgovara volumenu
nsrmm -C
# Izlaz oblika:
# 32916:nsrmm: file disk <volume_name> mounted on <device_name>, write enabled

# 2. Popunjavanje CFI i media baze informacijama o save set-ovima
scanner -i <device_name>
#    <device_name> je ime AFTD ili DD uređaja, NE ime volumena
```

Zatim u NMC-u:
3. **Unmount** uređaja (Administration → Devices → Devices → desni klik → Unmount). Zabeležiti volumen povezan sa uređajem.
4. Uklanjanje Scan Needed statusa: Administration → Media → **Disk Volumes** → desni klik na volumen → **Mark Scan Needed** → izabrati **Scan is NOT needed** → OK.
5. **Mount** uređaja nazad.

**Za trakaste uređaje:**

```bash
# Zabeležiti file i record broj iz poruke koju NetWorker prikaže
scanner -f <file> -r <record> -i <device>

# Uklanjanje Scan Needed zastavice sa volumena
nsrmm -o notscan <volume_name>
```

> **Lažne greške u `nsrdr.log`:** ako je oporavljeni NetWorker server štitio virtuelne cluster klijente ili NMM zaštićeni virtuelni DAG Exchange server, `nsrdr.log` sadrži lažne poruke o grešci koje se odnose na CFI oporavak osnovnih fizičkih hostova. Te poruke se mogu **ignorisati**, jer NetWorker ne backup-uje osnovni fizički host u virtuelnom okruženju.

## 21.6 Interpretacija rezultata

Nakon NetWorker recovery-ja status kopije se označava kao:

| Status | Značenje |
|---|---|
| **Recoverable** | Oporavak je uspešan |
| **Failed** | Oporavak nije uspeo |

## 21.7 Validacija u NetWorker UI

Nakon uspešnog oporavka, preko **Launch App** otvoriti NetWorker UI i proveriti (prema Best Practices Guide-u):

| Sekcija | Provera |
|---|---|
| **Protection** | Svi resursi izgledaju kao pre oporavka |
| **Devices** | Svi resursi izgledaju kao pre oporavka |
| **Media** | Svi resursi izgledaju kao pre oporavka |
| **Media → Tape Volumes / Disk Volumes** | Svi volumeni imaju isti režim kao pre oporavka; svi uređaji na koje se piše su u **appendable** režimu |

> **Podsetnik:** `jobsdb` neće sadržati istoriju — svi workflow-ovi imaće status `Never Run`. To je očekivano, nije greška.

> **NMC baza:** ako želite da NMC prikaže sve resurse posle `nsrdr`-a, potrebno je oporaviti i NMC bazu komandom `recoverpsm`. Primer: ako je Data Domain dodat pre Server DR-a, da bi bio prikazan posle DR-a mora se pokrenuti `recoverpsm`.

---

# 22. Recovery Check i validacija

Recovery check potvrđuje da se kopija može oporaviti, bez trajnog zauzimanja NetWorker instance.

**Kako radi:** procedura oporavi NetWorker server i zatim **automatski očisti** oporavak, bez obzira na to da li je uspeo ili ne. Cyber Recovery vraća NetWorker u početno stanje iz kojeg se može pokrenuti pravi oporavak.

**Ograničenja:**
- Ne pruža opciju pristupa NetWorker UI-u.
- **Ne može se oporaviti VM**, ali kopija ostaje oporavljiva kada zatreba.
- Zahteva NetWorker **19.9 ili noviji**, na Linux-u ili Windows-u.
- Za NetWorker na Windows-u verzije **19.18 ili starije**, ili NetWorker stariji od 19.9, Cyber Recovery generiše poruku o grešci.

## 22.1 Zakazani recovery check

1. **Main Menu → Policies**
2. Izabrati politiku → **Edit**
3. Na stranici **Summary** Edit Policy čarobnjaka:
   - Ako politika ima postojeće rasporede → **Edit** u sekciji **Schedule Information**
   - Ako nema → **Edit** u sekciji **Retention**, pa **Next** do stranice **Scheduling**
4. Na stranici **Scheduling** kliknuti **Add Schedule**
5. U polju **Run a** izabrati **Recovery Check**
6. U polju **App Host** izabrati NetWorker host
7. Odrediti učestalost izvršavanja
8. Uneti **storage username** i **password**
9. Opciono uneti ime foldera sa poslednjim bootstrap backup-ima
10. U poljima **Start Date** izabrati datum i vreme početka
11. Osigurati da je opcija **Enabled** izabrana
12. **Next** → pregledati **Summary** → **Finish**

## 22.2 On-demand recovery check

1. **Main Menu → Recovery**
2. Pod **Copies** izabrati kopiju
3. Kliknuti **Recovery Check**
4. Izabrati NetWorker host iz padajuće liste **Application Host**
5. Popuniti polja i kliknuti **Apply**:

| Polje | Opis |
|---|---|
| **Application Host** | Prikazan host izabran u prethodnom koraku |
| **Storage User** | Korisničko ime |
| **Storage Password** | Lozinka storage korisnika |
| **Bootstrap Device Folder** | Opciono — folder sa poslednjim bootstrap backup-ima |

Recovery check se izvršava odmah.

## 22.3 Preporuka za operativni režim

Zakazan recovery check je najbolji način da kupac ima kontinuiranu potvrdu oporavljivosti bez ručnog rada. Predložiti:

| Stavka | Preporuka |
|---|---|
| Učestalost | Uskladiti sa RPO/RTO zahtevima; tipično jednom sedmično ili mesečno |
| Vreme | Van prozora Sync/Copy/Lock operacija |
| Notifikacije | Konfigurisati email alerte za neuspešan check |
| Evidencija | Izvoz rezultata u CSV za potrebe revizije |

---

# 23. Čišćenje i post-recovery koraci

## 23.1 Unmount sandbox-a sa CR management hosta

```bash
umount /opt/dellemc/cr/mnt/cr-rec-<networker_sandbox>_1604
```

## 23.2 Brisanje objekata kreiranih tokom oporavka

Posle automatizovanog NetWorker recovery-ja, ručno obrisati uređaje koje je procedura oporavila na NetWorker server.

> **PAŽNJA:** NetWorker server može sadržati i druge uređaje koji su postojali pre CR recovery job-a. **Brisati isključivo objekte koje je Cyber Recovery kreirao.** Voditi računa da se ne obrišu objekti koji moraju ostati.

U NetWorker UI-u:

**Tab Protection:**
- Obrisati novododate klijente
- Obrisati novododate politike
- Obrisati novododate grupe
- Obrisati sve druge novododate tipove zaštite

**Tab Devices:**
- Obrisati novododate uređaje
- Obrisati novododate DD sisteme
- Obrisati novododate storage node-ove (ako je potrebno)

**Tab Media:**
- Obrisati novododate disk volumene
- Obrisati novododate media pool-ove
- Obrisati sve druge novododate tipove medija

## 23.3 Brisanje recovery sandbox-a

**Main Menu → Recovery → Recovery Sandboxes** → izabrati sandbox → **Cleanup**

## 23.4 Ručno čišćenje posle prekinutog oporavka

Ako se tokom automatizovanog recovery procesa javi problem i oporavak se ne završi čisto, izvršiti ručno čišćenje:

```bash
# 1. Zaustaviti NetWorker
/etc/init.d/networker stop

# 2. Resource baza
#    a. Pronaći najnoviji /nsr/res.cr.<timestamp> direktorijum
#    b. Ukloniti tekući /nsr/res direktorijum
#    c. Vratiti prethodnu resource bazu preimenovanjem
mv /nsr/res.cr.1554828308 /nsr/res

# 3. Media baza
#    a. Pronaći najnoviji /nsr/mm.cr.<timestamp> direktorijum
#    b. Ukloniti tekući /nsr/mm direktorijum
#    c. Vratiti prethodnu media bazu
mv /nsr/mm.cr.155512814 /nsr/mm

# 4. Index baza
#    a. Pronaći najnoviji /nsr/index.cr.<timestamp> direktorijum
#    b. Ukloniti tekući /nsr/index direktorijum
#    c. Vratiti prethodni index direktorijum
mv /nsr/index.cr.151231326 /nsr/index

# 5. Pokrenuti NetWorker
/etc/init.d/networker start
```

> Vrednosti timestamp-a u primerima su ilustrativne — koristiti stvarne vrednosti sa sistema.

## 23.5 Oporavak aplikativnih i korisničkih podataka

Nakon što je NetWorker server u vault-u oporavljen i validiran, oporavak stvarnih podataka izvršava se standardnim NetWorker procedurama:

1. Prijaviti se kao `root`.
2. Učitati i inventarisati uređaje, da NetWorker server prepozna lokaciju svakog volumena.

   > Ako učitavate klonirani volumen, ili obrišite originalni volumen iz media baze, ili označite željeni save set kao suspect. Ako koristite klonirani volumen, on se koristi do kraja procesa oporavka.

3. Iz komandne linije pokrenuti `recover`.
4. Označiti direktorijume ili fajlove za oporavak. Opcija `a` bira ili dodaje sve fajlove.

   > **Prepisivanje fajlova operativnog sistema može dovesti do nepredvidivih rezultata.**

5. Uneti `recover` da bi oporavak počeo.

---

# 24. Referenca `nsrdr` komande

Cyber Recovery automatizovano poziva `nsrdr`. Ova referenca je za scenarije kada SE mora intervenisati ručno — npr. kod NetWorker servera sa više MTree-ova, ili kada automatizovana procedura ne uspe.

> `nsrdr` zamenjuje zastarelu komandu `mmrecov` (deprecated od NetWorker 9.0). Koristiti `nsrdr` za oporavak NetWorker 9.x i novijih baza. Za rollback na raniju verziju NetWorker softvera kontaktirati Dell Support.

**Lokacije logova:**
- Linux: `/nsr/logs/nsrdr.log`
- Windows: `<NetWorker_install_path>\nsr\logs\nsrdr.log`

## 24.1 Tabela opcija

| Opcija | Opis |
|---|---|
| `-a` | Neinteraktivni režim. Minimalno moraju biti navedene `-B` i `-d`. Bez validnog bootstrap ID-a uz `-B`, čarobnjak izlazi kao da je otkazan, bez opisne poruke o grešci. |
| `-B <bootstrap_ID>` | Save set ID bootstrap-a koji se oporavlja |
| `-d <device_name>` | Uređaj sa kojeg se oporavlja bootstrap |
| `-K` | Koristi originalne resource fajlove umesto oporavljenih |
| `-v` | Verbose režim |
| `-q` | Quiet režim — samo poruke o greškama |
| `-c` | Oporavlja samo client file indekse. Uz `-a` mora se navesti i `-I`. |
| `-I [client1 client2 ...]` | Određuje koje CFI-jeve oporaviti. Imena klijenata razdvojena razmakom. Bez imena — oporavljaju se svi CFI-jevi. **Mora biti poslednja opcija u komandi**, jer se sve posle nje tumači kao imena klijenata. |
| `-f <path/file_name>` | CFI-jevi se određuju preko ASCII tekst fajla, jedno ime po liniji. Koristi se uz `-I`. Nema validacije imena klijenata. |
| `-t <date/time>` | Oporavlja CFI-jeve od navedenog datuma/vremena. Format prema `nsr_getdate`. |
| `-N` | Ako trakasti volumeni imaju save set-ove novije od onoga što je zapisano u oporavljenom bootstrap-u, označavaju se kao Scan Needed. Za AFTD uređaje sprečava recover space operacije dok se Scan Needed ne ukloni. |
| `-F` | Postavlja Scan Needed samo na File type, AFTD i Cloud uređaje; trakaste volumene ne označava. **Zahteva `-N`.** |
| `-l` | Alternativna putanja za preimenovanje NetWorker resursa, u slučaju neuspeha preimenovanja u podrazumevanoj putanji `/nsr`. Mora biti lokacija foldera; ako folder ne postoji, kreira se. |

## 24.2 Primeri

```bash
# Oporavak bootstrap podataka i izabranih CFI-jeva
nsrdr -I client1 client2 client3

# Oporavak bootstrap-a i izabranih CFI-jeva preko ulaznog fajla
nsrdr -f <path>/<file_name> -I

# Preskakanje bootstrap oporavka, oporavak izabranih CFI-jeva iz fajla
nsrdr -c -f <path>/<file_name> -I

# Preskakanje bootstrap-a, oporavak SVIH CFI-jeva
nsrdr -c -I

# Preskakanje bootstrap-a, oporavak izabranih CFI-jeva
nsrdr -c -I client1 client2

# Preskakanje bootstrap-a, CFI-jevi od određenog datuma
nsrdr -c -t <date/time> -I client1 client2

# Neinteraktivni režim, bootstrap + svi CFI-jevi
nsrdr -a -B <bootstrap_ID> -d <device> -I

# Zaštita od prepisivanja ručnih backup-a posle poslednjeg bootstrap-a
nsrdr -N
```

## 24.3 Parametri podešavanja (`nsrdr.conf`)

Za veliki broj klijenata, podrazumevanih 5 paralelnih niti može biti usko grlo.

```bash
# Kreirati ASCII tekst fajl nsrdr.conf
# Linux:   /nsr/debug/nsrdr.conf
# Windows: <NW_install_path>\nsr\debug\nsrdr.conf
```

Sadržaj:

```
NSRDR_SERVICES_PATH = /non_default_path/nsr
NSRDR_NUM_THREADS = 10
```

| Parametar | Opis |
|---|---|
| `NSRDR_SERVICES_PATH` | Putanja do NetWorker servisa, ako podrazumevana putanja nije korišćena pri instalaciji |
| `NSRDR_NUM_THREADS` | Broj paralelnih niti pri oporavku CFI-jeva. **Podrazumevano 5.** Vrednost mora biti veća od 1; nula ili negativna vrednost vraća podrazumevanih 5. Povećanje skraćuje vreme DR-a kod velikog broja klijenata. |

> Obavezno razmak pre i posle znaka `=`. Ako se navode oba parametra, svaki mora biti u zasebnoj liniji. Neki editori dodaju `.txt` na kraj imena — ukloniti ekstenziju.

Parametri stupaju na snagu pri sledećem pokretanju `nsrdr`-a.

## 24.4 Provera pre pokretanja

> Pre ručnog DR-a NetWorker baza proveriti da direktorijum authentication baze **ne sadrži oporavljeni fajl baze noviji od bootstrap-a** koji se oporavlja. Ime takvog fajla je oblika `authcdb.h2.db.<timestamp>`.

## 24.5 Lokalni uređaj je obavezan

NetWorker server zahteva **lokalni device resurs** za oporavak podataka iz bootstrap backup-a. U DR situaciji resource baza je izgubljena, pa se lokalni uređaj mora ponovo kreirati.

| Pravilo | Detalj |
|---|---|
| **Ne označavati (label) volumen ponovo** | Ponovno označavanje volumena sa bootstrap-om ili bilo kojim backup-om čini podatke neoporavljivim |
| **Za disk uređaje (AFTD)** | Ne dozvoliti čarobnjaku da označi disk volumen. Opcija **Label and Mount** je podrazumevano izabrana u prozoru **Device Label and Mount** — **isključiti je**. |
| **Putanja** | U prozoru **Select Storage Node** navesti lokalnu putanju do AFTD, DD ili trakastog volumena. Mora biti ista putanja na kojoj se nalaze bootstrap podaci. |
| **AFTD na lokalnom serveru** | Ako je AFTD kreiran na samom serveru, podaci mogu biti izgubljeni pri padu servera. Preporučuje se DD/trakasti uređaj ili AFTD koji nije na lokalnom disku (npr. AFTD na NFS share-u). |

> **Bootstrap na udaljenom uređaju:** NetWorker **ne podržava** oporavak bootstrap-a sa udaljenog uređaja. Bootstrap sa kloniranog save set-a na udaljenom uređaju mora se prvo klonirati na uređaj lokalan za NetWorker server.

---

# Dodatak A: Checklist NetWorker integracije

## A. Produkciona strana

| # | Stavka | Vrednost / potvrda | Status |
|---|---|---|---|
| A1 | NetWorker verzija na produkciji | | |
| A2 | Platforma (Linux / Windows) | | |
| A3 | DD Boost korisnik kreiran, `role admin` | | |
| A4 | DD Boost korisnik dodeljen (`ddboost user assign`) | | |
| A5 | **UID DD Boost korisnika zabeležen** | | |
| A6 | Server Protection politika omogućena i uspešna | | |
| A7 | Perform Bootstrap i/ili Perform CFI označeno | | |
| A8 | Namenski pool za bootstrap | | |
| A9 | Bootstrap backup najmanje jednom u 24 sata | | |
| A10 | Bootstrap volumeni se kloniraju | | |
| A11 | Bootstrap piše na MTree koji se replicira u vault | | |
| A12 | Vreme završetka Server backup workflow-a zabeleženo | | |
| A13 | Ime bootstrap device foldera zabeleženo | | |
| A14 | NetWorker server ima **samo jedan MTree** | | |
| A15 | Notifikacije politike konfigurisane i praćene | | |

## B. Vault strana — NetWorker instanca

| # | Stavka | Vrednost / potvrda | Status |
|---|---|---|---|
| B1 | Ista verzija OS-a kao na produkciji | | |
| B2 | Ista verzija NetWorker-a kao na produkciji | | |
| B3 | Instalirani paketi: client, storage node, Authentication service | | |
| B4 | `authc_configure.sh` pokrenut | | |
| B5 | Sve NetWorker zakrpe instalirane | | |
| B6 | Instanca je **neinicijalizovana** | | |
| B7 | Isti FQDN kao produkcioni server | | |
| B8 | Ista IP adresa (preporuka) — ili prihvaćen rizik | | |
| B9 | Shortname uređaja isti kao pre | | |
| B10 | `nsswitch.conf` podešen (`hosts: files`) | | |
| B11 | Lokalni `hosts` fajl popunjen | | |
| B12 | Retention policy klijenta postavljena na **Decade** | | |
| B13 | Aliases atribut sadrži shortname i FQDN | | |
| B14 | `nsrtomcat` UID isti kao na produkciji (Linux) | | |
| B15 | SSH sa CR hosta ka NetWorker hostu radi | | |
| B16 | Instanca dimenzionisana za `nsrdr` | | |

## C. Vault strana — Windows / Cygwin (ako je primenljivo)

| # | Stavka | Status |
|---|---|---|
| C1 | Windows host u vault-u sa omogućenim Remote Desktop-om | |
| C2 | Postojeći SSH onemogućen (ne pokreće se automatski) | |
| C3 | Cygwin instaliran | |
| C4 | OpenSSH paket dodat | |
| C5 | `ssh-host-config` izvršen, sshd instaliran kao servis | |
| C6 | StrictModes = no | |
| C7 | Ključevi verifikovani u `/etc/` | |
| C8 | Cygwin sshd servis pokrenut | |
| C9 | Port 22 otvoren (`New-NetFirewallRule`) | |
| C10 | SSH pristup sa CR hosta radi | |
| C11 | `cd <NetWorker path>` dodat u `~/.bashrc` na ispravnom mestu | |

## D. Cyber Recovery konfiguracija

| # | Stavka | Vrednost / potvrda | Status |
|---|---|---|---|
| D1 | Replication kontekst kreiran | | |
| D2 | Inicijalna replikacija završena | | |
| D3 | Port 3009 zatvoren posle dodavanja para | | |
| D4 | Aplikacija dodata: Application Type = NetWorker | | |
| D5 | Host Authentication: `root` (Linux) / `Admin` (Windows) | | |
| D6 | Tag sa produkcionim DD Boost username-om dodat | | |
| D7 | Standard politika kreirana | | |
| D8 | Sync zakazan **posle** završetka produkcionog backup-a | | |
| D9 | Copy i Lock zakazani | | |
| D10 | Retention Lock režim i trajanja podešeni | | |
| D11 | Prva Sync operacija uspešna | | |
| D12 | Prva Copy operacija uspešna, kopija vidljiva | | |
| D13 | Lock operacija uspešna | | |

## E. UID usklađivanje

| # | Stavka | Vrednost / potvrda | Status |
|---|---|---|---|
| E1 | `crcli policy list-copy` izvršen, Source Storage UID zabeležen | | |
| E2 | `user show list` na vault DD proveren | | |
| E3 | DD Boost nalog kreiran u vault-u sa tačnim UID-om | | |
| E4 | Privremeni nalozi obrisani (ako su korišćeni) | | |
| E5 | UID konvencija primenjena (501–599 rezervisano, 600–700 za CR) | | |

## F. Validacija

| # | Stavka | Status |
|---|---|---|
| F1 | Test recovery pokrenut iz CR UI | |
| F2 | Job `recoverapp_<ID>` uspešno završen | |
| F3 | Recovery sandbox kreiran | |
| F4 | **Launch App** aktivan, NetWorker UI dostupan | |
| F5 | Protection resursi izgledaju kao pre | |
| F6 | Devices resursi izgledaju kao pre | |
| F7 | Media resursi i režimi volumena ispravni | |
| F8 | Status kopije = **Recoverable** | |
| F9 | Sandbox očišćen (**Cleanup**) | |
| F10 | Objekti kreirani tokom oporavka obrisani iz NetWorker-a | |
| F11 | Sandbox unmount-ovan sa CR management hosta | |
| F12 | Recovery Check zakazan | |
| F13 | Rezultati zabeleženi u zapisniku o primopredaji | |

---

# Dodatak B: Otvorena pitanja

Stavke koje nisu pokrivene priloženom dokumentacijom i zahtevaju potvrdu pre finalizacije dokumenta:

| # | Pitanje | Zašto je bitno |
|---|---|---|
| 1 | **vProxy u vault-u.** *NetWorker and VMware Integration Guide* opisuje zaštitu VM-ova preko vProxy appliance-a, ali CR dokumentacija ne govori o ponašanju vProxy-ja pri oporavku u vault-u. Kod PPDM-a je eksplicitno navedeno da se protection engine, Search i Reporting engine **ne oporavljaju**, jer su zasebni VM-ovi sa drugačijim hostname-om/IP-om u vault-u. Za NetWorker vProxy analogna izjava ne postoji. | Ako kupac koristi NetWorker VMware zaštitu, treba znati da li se vProxy mora ručno deploy-ovati i registrovati u vault-u posle oporavka. **Preporuka: potvrditi kod Dell-a ili testirati.** |
| 2 | **NMC server u vault-u.** `recoverpsm` procedura za oporavak NMC baze je dokumentovana u Best Practices Guide-u, ali Cyber Recovery je ne automatizuje. | Treba odlučiti da li je NMC u obimu implementacije i ko izvršava `recoverpsm`. |
| 3 | **NetWorker Installation Guide** za konkretnu verziju | Za precizne korake instalacije neinicijalizovane instance u vault-u (poglavlje 18.2) trenutno se oslanjamo na uopšten opis iz Best Practices Guide-a. |
| 4 | **NetWorker Administration Guide** | Za detalje o konfiguraciji autochanger-a, notifikacija politika i `Recovering expired save sets` procedure. |
| 5 | **Više MTree-ova** | Ako kupčev NetWorker server ima više od jednog MTree-a, automatski recovery nije podržan. Ručna procedura zahteva angažman Dell Support-a i treba je predvideti u planu. |

---

*Kraj Dela IV. Sledeći deo: **Deo V — Integracija sa PowerProtect Data Manager-om** (poglavlja 25–31).*
