# DEO VI — VALIDACIJA, PREDAJA I ODRŽAVANJE

**Dell PowerProtect Cyber Recovery 20.3 — Uputstvo za instalaciju i integraciju**
Verzija dokumenta: 0.1 | Namena: System Engineer

> **Izvori:** *Dell PowerProtect Cyber Recovery 20.3 Installation and Upgrade Guide*, *Dell PowerProtect Cyber Recovery 20.3 Product Guide*.

---

## Sadržaj

- [32. Validacija implementacije](#32-validacija-implementacije)
- [33. Operativne procedure](#33-operativne-procedure)
- [34. Nadogradnja deploymenta](#34-nadogradnja-deploymenta)
- [35. Migracija deploymenta](#35-migracija-deploymenta)
- [36. Disaster recovery Cyber Recovery servera](#36-disaster-recovery-cyber-recovery-servera)
- [37. Troubleshooting](#37-troubleshooting)
- [38. Deinstalacija](#38-deinstalacija)
- [Dodatak A: Zapisnik o primopredaji](#dodatak-a-zapisnik-o-primopredaji)
- [Dodatak B: Operativni kalendar](#dodatak-b-operativni-kalendar)

---

# 32. Validacija implementacije

## 32.1 Korisničke uloge — provera pre predaje

Cyber Recovery koristi četiri uloge. Proveriti da su dodeljene u skladu sa dogovorom sa kupcem i principom najmanjih privilegija.

| Uloga | Ovlašćenja |
|---|---|
| **Security Admin** (`crso`) | Kreiranje i upravljanje korisničkim nalozima, postavljanje password politike, konfiguracija mail servera, praćenje alerta, **obezbeđivanje (secure) i oslobađanje (release) vault-a**, zakazivanje telemetrije, upravljanje support podešavanjima |
| **Admin** | Puna konfiguracija: storage, aplikacije, politike, recovery operacije |
| **Vault Operator** | Pregled i izvoz informacija, obezbeđivanje vault-a, ograničeni zadaci i operacije, generisanje support bundle-ova |
| **Dashboard** | Samo pregled dashboard-a, bez izvršavanja zadataka. **Ova uloga ne ističe (ne time-out-uje).** |

> **Bitna asimetrija:** vault mogu **obezbediti** Security Admin, Admin i Vault Operator, ali ga može **osloboditi samo Security Admin**. Ovo treba objasniti kupcu — u incidentu bilo koji operater može zatvoriti vault, ali za otvaranje je potreban Security Admin.

## 32.2 Status vault-a — referenca

Status se vidi na **Main Menu → Dashboard**, pod **Status**.

| Status | Značenje |
|---|---|
| **Locked** | Sve konfigurisane replikacione konekcije su zatvorene jer se replikacija ne izvršava. Ako se pokrene replikaciona politika, CR otvara konekciju i status prelazi u Unlocked. |
| **Unlocked** | Jedna ili više replikacionih konekcija je otvoreno jer je replikacija u toku. Tajmer pokazuje koliko dugo je vault otvoren. Status se vraća u Locked po završetku. |
| **Secured** | Sve replikacione konekcije su obezbeđene jer je Security Admin, Admin ili Vault Operator ručno zatvorio konekciju zbog bezbednosnog incidenta. Ne mogu se pokretati replikacione politike. Tajmer pokazuje trajanje. CR generiše alert sa informacijom **koji korisnik** je obezbedio vault. |
| **Degraded** | Ima više DD sistema u vault-u, a **jedan** ne može da komunicira sa CR softverom. Tipičan uzrok: promenjen FQDN ili IP adresa DD sistema. |
| **Unknown** | Ima više DD sistema u vault-u, a **nijedan** ne može da komunicira. Tipični uzroci: nisu kreirane politike posle instalacije, ili su promenjeni FQDN/IP DD sistema. |

> **Očekivano ponašanje posle instalacije:** vault može biti **Unlocked**. To je po dizajnu — inicijalizacija može biti u toku dok konfigurišete okruženje, pa port mora biti otvoren. CR kreira job za inicijalnu Sync operaciju. Kada se inicijalizacija završi, port se zatvara automatski.
>
> **Ne može se kreirati drugi Sync job dok inicijalni Sync traje.**

## 32.3 Acceptance checklist — infrastruktura i mreža

| # | Provera | Očekivano | Status |
|---|---|---|---|
| 1 | Verzija CR softvera | 20.3 | |
| 2 | Verzija DDOS na svim DD sistemima | 7.13 ili 8.x | |
| 3 | Verzija OS-a management hosta | Podržana | |
| 4 | Podman i Podman Compose verzije | Prema matrici | |
| 5 | `./crsetup.sh --check` prolazi bez grešaka | Da | |
| 6 | CR servisi rade posle reboota | Da | |
| 7 | NTP sinhronizacija svih komponenti vault-a | `chronyc tracking` OK | |
| 8 | DNS rezolucija | Radi | |
| 9 | Portovi otvoreni prema tabeli | Da | |
| 10 | Port 3009 zatvoren posle dodavanja replikacionih parova | Da | |
| 11 | Podman subnetovi bez kolizije sa vault mrežom | Da | |
| 12 | Replikacioni interfejs je namenski | Da | |

## 32.4 Acceptance checklist — bezbednost

| # | Provera | Očekivano | Status |
|---|---|---|---|
| 1 | Podrazumevane lozinke promenjene (`root`, `admin` na OVA) | Da | |
| 2 | Lockbox passphrase pohranjen van CR servera, na dva mesta | Da | |
| 3 | Password politika podešena | Prema zahtevu kupca | |
| 4 | MFA konfigurisan | Prema dogovoru | |
| 5 | Uloge dodeljene po principu najmanjih privilegija | Da | |
| 6 | Custom CA sertifikat sa punim lancem | Ako se koristi | |
| 7 | TLS za email omogućen | Da (TLS 1.2/1.3) | |
| 8 | Retention Lock režim odgovara dizajnu | Da | |
| 9 | DD Security Officer nalog kreiran (ako Compliance) | Da | |
| 10 | SELinux Enforcing / AppArmor aktivan | Da | |
| 11 | Audit log forwarding ka SIEM-u | Ako je u obimu | |
| 12 | `sysadmin` nalog **nije** korišćen za CR | Potvrđeno | |

## 32.5 Acceptance checklist — politike i replikacija

| # | Provera | Očekivano | Status |
|---|---|---|---|
| 1 | Sve politike kreirane i omogućene | Da | |
| 2 | Inicijalna replikacija završena za svaki kontekst | Da | |
| 3 | Sync operacija uspešna | Da | |
| 4 | Copy operacija uspešna, PIT kopija vidljiva | Da | |
| 5 | Lock operacija uspešna, retencija primenjena | Da | |
| 6 | Rasporedi zakazani **posle** produkcionih backup-a | Da | |
| 7 | Min/Max Retention Lock Period u dozvoljenom opsegu | Da | |
| 8 | Vault status se vraća u **Locked** po završetku Sync-a | Da | |
| 9 | Nema kritičnih alerta na dashboard-u | Da | |
| 10 | Pragovi kapaciteta (80% / 90%) — trenutna popunjenost | Zabeleženo | |

## 32.6 Acceptance checklist — zaštita same CR konfiguracije

| # | Provera | Očekivano | Status |
|---|---|---|---|
| 1 | MTree za DR backup kreiran na vault DD | Da | |
| 2 | DR backup konfigurisan i omogućen | Da | |
| 3 | Učestalost DR backup-a definisana | Zabeleženo | |
| 4 | DR backup uspešno izvršen bar jednom | Da | |
| 5 | DR backup **nije** zakazan u isto vreme kad drugi job-ovi | Potvrđeno | |
| 6 | `crsetup.sh --save` kopija pohranjena van servera | Da | |
| 7 | Maintenance (cleaning) raspored podešen | Da | |

## 32.7 Test recovery scenario

Test oporavka je **obavezan** deo primopredaje. Bez njega se ne može tvrditi da je rešenje funkcionalno.

| Korak | Radnja | Rezultat |
|---|---|---|
| 1 | Izabrati PIT kopiju za test | |
| 2 | Pokrenuti Application recovery | |
| 3 | Pratiti job do uspešnog završetka | |
| 4 | Otvoriti aplikaciju preko **Launch App** | |
| 5 | Validirati sadržaj u aplikaciji | |
| 6 | Izvršiti test restore workload-a | |
| 7 | Očistiti sandbox (**Cleanup**) | |
| 8 | Verifikovati status kopije = **Recoverable** | |
| 9 | Zabeležiti trajanje svakog koraka | |

> **Trajanja zabeležiti** — služe kao osnova za realan RTO koji se saopštava kupcu.

## 32.8 Izvoz dokazne dokumentacije

Cyber Recovery omogućava izvoz u CSV za potrebe revizije i zapisnika:

| Objekat | Fajl | Gde |
|---|---|---|
| Job-ovi | `jobsList.csv` | Main Menu → Jobs → Export |
| Sandbox-ovi | `sandboxes.csv` | Recovery → Sandboxes → Export |
| Recovery sandbox-ovi | `recoverysandboxes.csv` | Recovery → Recovery Sandboxes → Export |
| vCenter serveri | `vcenter.csv` | Infrastructure → Assets → vCenters → Export |
| Aplikacije | `.csv` | Infrastructure → Assets → Applications → Export |

> **Dugme Export je onemogućeno** ako su primenjeni filteri kolona, filteri pretrage ili je označen checkbox. Očistiti filtere da bi se dugme aktiviralo.

---

# 33. Operativne procedure

## 33.1 Gašenje i pokretanje komponenti vault-a

> **Redosled je bitan.** Cyber Recovery softver kontroliše sve operacije u vault-u.

### Priprema (kao Admin korisnik)

1. Prijaviti se u CR UI i proveriti:
   - **Nema protection, system ni recovery job-ova u toku.**

     > Analyze job **sme** biti u toku. Kada se Cyber Recovery ponovo pokrene, softver restartuje Analyze operaciju i ažurira analizirane job-ove.

   - **Svi rasporedi su onemogućeni.**

2. **Main Menu → Recovery** — proveriti da nema oporavka u toku i da se nijedan sandbox ne koristi.

   > Ako se recovery sandbox-ovi koriste, po završetku rada sa oporavljenom backup aplikacijom pokrenuti cleanup operaciju na svakom sandbox-u.

3. Proveriti da **DD file system cleanup operacija nije u toku.**

   > Pošto CR kontroliše sve operacije u vault-u, jedine aktivne operacije su DD sistemski job-ovi kao što je file system cleanup, koji se podrazumevano izvršava **svakog utorka**.

### Gašenje

4. Ugasiti **DD, Cyber Detect i backup aplikacije — bilo kojim redosledom**.
5. Izvršiti radove zbog kojih je vault gašen.

### Pokretanje

6. Pokrenuti **sve komponente osim Cyber Recovery softvera**.
7. Kada sve komponente rade, pokrenuti **Cyber Recovery**.
8. **Ponovo omogućiti sve rasporede** koji su bili onemogućeni.

### Posle pokretanja

> Sinhronizovati vreme svih komponenti preko NTP-a (chrony). U suprotnom mogu nastati problemi.

## 33.2 Upravljanje CR servisima

```bash
# Zaustavljanje CR softvera
./crsetup.sh --stop

# Pokretanje svih servisa
./crsetup.sh --start

# Zaustavljanje pa pokretanje svih servisa
./crsetup.sh --restart
```

> **Pojedinačni CR servis se ne može zasebno zaustaviti i pokrenuti.**

## 33.3 Ručno obezbeđivanje i oslobađanje vault-a

Koristi se pri sumnji na bezbednosni incident.

**Obezbeđivanje** (Security Admin, Admin ili Vault Operator):

1. Na dashboard-u otići na **Status** tile.
2. Kliknuti **Secure Vault** — status prelazi iz **Locked** u **Secured**.

**Posledice:**
- Sve Sync operacije politika se **odmah zaustavljaju**.
- Nove Sync operacije se ne mogu pokrenuti.
- Tajmer pokazuje koliko dugo je vault obezbeđen.
- CR generiše alert sa dodatnim informacijama i **identifikuje korisnika** koji je obezbedio vault.

**Oslobađanje** — **samo Security Admin**, i to tek kada je pouzdano utvrđeno da bezbednosna pretnja više ne postoji.

## 33.4 Zaštita Cyber Recovery konfiguracije (DR backup)

> **Snažno se preporučuje konfiguracija DR backup-a** radi zaštite CR konfiguracije i politika u slučaju otkaza management servera.

**Preduslovi:**
- Prijavljeni kao Admin korisnik
- **Kreiran MTree na vault DD sistemu** koji CR koristi za DR backup

**Konfiguracija:**

1. Masthead navigacija → **System Settings → DR Backups**
2. **Configuration**:
   - Slider udesno za omogućavanje DR backup-a (**podrazumevano je onemogućen**)
   - Izabrati DD sistem za čuvanje backup podataka
   - Odrediti MTree
   - Postaviti učestalost i datum/vreme sledećeg izvršavanja
   - **Save**

> CR UI koristi istu vremensku zonu kao CR management host za zakazano vreme.

**Kritična ograničenja:**

| Ograničenje | Detalj |
|---|---|
| **Konflikt sa drugim job-ovima** | Osim Analyze job-a, ako bilo koji drugi job radi u vreme zakazanog ili ručno pokrenutog DR backup-a, **DR backup se ne izvršava**. Ne zakazivati druge job-ove u isto vreme kao DR backup. |
| **Stale sandbox-ovi** | Ako se DR backup izvrši dok Analyze job radi, obrisati nastale stale sandbox-ove. **U suprotnom se ne može pokrenuti sledeći Analyze job.** |

**CLI alternativa:**

```bash
./crsetup.sh --save
```

Backup se čuva u `/opt/dellemc/cr-configs`. **Kopirati ga na lokaciju van CR servera.**

> UI opcija je preferirana jer postavlja server DR backup na konfigurisani DD MTree.

## 33.5 Preuzimanje pohranjene konfiguracije

DR backup-i se čuvaju na zasebnom MTree-u na vault DD sistemu.

```bash
# 1. Na DD sistemu kreirati NFS export mapiran na CR management host
#    OBAVEZNO koristiti no_root_squash opciju
sysadmin@dd-vault# nfs add /data/col1/drbackups <hostname>(no_root_squash)

# 2. Na CR management hostu montirati NFS export
mount <DD_hostname>:/data/col1/drbackups /mnt/drbackups

# 3. Pristupiti backup podacima i izvršiti oporavak

# 4a. Posle oporavka — odmontirati
umount /mnt/drbackups

# 4b. Na DD sistemu ukloniti NFS export
sysadmin@dd-vault# nfs del /data/col1/drbackups <hostname>
```

## 33.6 Maintenance (cleaning) raspored

Konfiguriše brisanje alerta, događaja, isteklih i otključanih kopija, DR backup-a i job-ova kada više nisu potrebni. **Postavljanjem rasporeda izbegava se usporavanje sistema.**

**Preduslovi:**
- Prijavljeni kao Admin (Vault Operator može samo da pregleda raspored)
- Da bi DR backup bio u rasporedu čišćenja, omogućiti i konfigurisati DR backup pod **System Settings → DR Backups**

**Konfiguracija:**

1. Masthead navigacija → **System Settings → Maintenance**
2. Tab **Cleaning Schedule**:
   - Učestalost izvršavanja
   - Datum i vreme sledećeg izvršavanja
   - Starost objekata za brisanje

**Ponašanje polja `Delete Unlocked Copies Older Than`:**

| Tip kopije | Ponašanje |
|---|---|
| **Otključana kopija** | Briše se posle zadatog broja dana |
| **Zaključana kopija** | Briše se zadati broj dana **posle isteka retention lock-a** |

> **Primer:** kopija je retention-locked 14 dana, a polje je postavljeno na 7 dana. Posle 14 dana fajl se otključava, pa se posle još 7 dana briše. **Ukupno 21 dan.**

> **Zaštita:** ako je jedini preostali backup istekao, CR ga **ne briše** — uvek je dostupan bar jedan DR backup.

## 33.7 Monitoring

### Kapacitet

**Main Menu → Infrastructure → Storage → tab Capacity**

Prikazuje informacije o alertima i fizičkom i logičkom vault storage-u za DD sisteme.

| Prag | Podrazumevana vrednost | Posledica |
|---|---|---|
| Warning | 80% | Alert na dashboard-u + email |
| Critical | 90% | Alert na dashboard-u + email |
| Nedostatak prostora | — | **Sync job ne uspeva** |

> **MTree limit:** obrisani MTree-ovi se **ne računaju** u limit aktivnih MTree-ova. Time CR podržava više politika koje izvršavaju analyze i recovery check operacije. CR šalje alert sa zahtevom za restart DD sistema radi omogućavanja te funkcije. **Ako je omogućite, ne možete je kasnije onemogućiti.** Maksimalan broj podržanih MTree-ova ostaje isti.

### Job-ovi

**Main Menu → Jobs** — tabovi **Running** i **Completed**.

**Otkazivanje job-a u toku:**
1. Tab **Running**
2. Radio dugme pored imena job-a
3. **Cancel** i potvrda
4. CR generiše alert za zahtev otkazivanja; prikazuje se napredak i korak procesa otkazivanja
5. Detalje videti na tabu **Step Log** — na kom koraku je proces otkazan
6. Po završetku, job više nije u panelu Running; proveriti tab **Completed** za status **Canceled**

### Logovi

| Stavka | Vrednost |
|---|---|
| Maksimalna veličina log fajla | **50 MB** |
| Broj arhivskih fajlova | **10** (FIFO — najstariji se briše) |
| Nivoi logovanja | **Info** (podrazumevano) i **Debug** |
| Podešavanje nivoa | System Settings → Support → Log Settings |

> `Info` daje kontekstualne detalje o stanju i konfiguraciji softvera. `Debug` daje granularne detalje za analizu i dijagnostiku.

### Support bundle

**System Settings → Support → Support Bundles → Generate Log Bundle**

- Log fajlovi se prikupljaju u `.tar` fajl u `/opt/dellemc/cr/var/log`
- CR pokreće prikupljanje logova i **na svim pridruženim DD sistemima** u vault-u
- Za pregled DD kolekcija: PowerProtect DD Management Center → Settings → System → Support → Support Bundles
- Preuzeti bundle sadrži log fajlove i **checksum fajl** (SHA-256)

> Bundle se **ne može preuzeti** ako je status `Generation in Progress` ili `Failed`. Briše se **jedan po jedan**.

### Telemetrija

**System Settings → Support → Telemetry Reports** (Security Admin)

**Preduslovi:** važeća email adresa, mail server omogućen za prijem poruka.

Izveštaj sadrži: broj po korisničkoj ulozi i broj uloga sa omogućenim MFA, politike, aplikacije, CR verzije instalacija i nadogradnji, CR servise, vault DD storage, mail server, nedostajuće ili istekle TLS sertifikate.

| Parametar | Vrednost |
|---|---|
| Učestalost | Minimum 1 dan, maksimum 30 dana |
| Odredište | `dataprotection-telemetry@emc.com` |

> Ako su omogućena ograničenja domena, izveštaj se ipak šalje — ograničeni domeni se ignorišu.

---

# 34. Nadogradnja deploymenta

## 34.1 Redosled nadogradnje komponenti

> **Prvo nadograditi Dell backup aplikacije i komponente trećih strana, pa tek onda Cyber Recovery softver.**

Postupak za komponente:
1. Proveriti da nema aktivnih job-ova.
2. Zaustaviti sve CR servise.
3. Nadograditi Dell backup aplikacije i komponente trećih strana.
4. Pokrenuti sve CR servise.
5. Sinhronizovati vreme svih komponenti (chrony).

**Uticaj nadogradnje po komponenti:**

| Komponenta | Postupak i uticaj |
|---|---|
| **DD softver** | Nadogradnja preko `.rpm` paketa iz UI-a ili CLI-ja na DD sistemu. Disruptivno za DD, ali CR nastavlja da radi. Operacije su ograničene jer se DD ne može kontaktirati. Nema negativnog uticaja na CR; po završetku sve radi normalno. Novije DD verzije mogu otključati funkcije poput **ARL** koje CR ranije nije podržavao. |
| **NetWorker** | Linux: `rpm -U`. Windows: instalater → **Upgrade**. Disruptivno za NetWorker. CR radi uz ograničenu smetnju. **Automatizovani NetWorker recovery ne radi** tokom nadogradnje. |
| **PPDM** | Videti PPDM dokumentaciju. |
| **Avamar** | Nema programskog načina za čišćenje Avamar servera, pa metapodaci ostaju. AVE: **preporuka je obrisati i ponovo deploy-ovati** Avamar server u verziji koja odgovara produkciji. Fizički Avamar: **samo Avamar support tim** može pokrenuti nadogradnju — postoji KB članak. |

## 34.2 Putanje nadogradnje CR softvera

> **Pre nadogradnje na 20.3 obavezno instalirati OS ažuriranja.**
> **Minimalna podržana verzija DDOS-a za CR 20.3 je 7.13 ili 8.x.** Ako nadograđujete sa verzije starije od 19.17, prvo nadograditi DDOS.

| Trenutna verzija | Putanja nadogradnje |
|---|---|
| **19.17 ili novija** | Direktno na 20.3 |
| **19.16** | → 19.20 → 20.3 |
| **19.8 do 19.15** | → 19.16 → 19.20 → 20.3 |
| **19.1 do 19.7** | → 19.8 → 19.16 → 19.20 → 20.3 |
| **18.1.1.7** | → 19.1.0.9 → 19.8 → 19.16 → 19.20 → 20.3 |
| **Starija od 18.1.1.7** (osim 18.1.0-529) | → 18.1.1.7 → 19.1.0.9 → 19.8 → 19.16 → 19.20 → 20.3 |
| **18.1.0-529 ili 18.1.0-532** | → 18.1.1.4 → 19.1.0.9 → 19.8 → 19.16 → 19.20 → 20.3 |

**Dodatne napomene:**
- Ako okruženje uključuje **virtuelni appliance stariji od 20.3**, primeniti bezbednosne zakrpe virtuelnog appliance-a **pre** nadogradnje na 20.3.
- Ako CR radi na **SLES 12 u AWS, Azure ili GCP**, **ne može** se direktno nadograditi na 20.2 — prvo nadograditi OS na SLES 15.
- Za **CR verzije 18.x** backup i kopiju sistema treba napraviti **ručno** — te verzije nemaju automatizovan proces.

## 34.3 Putanje nadogradnje virtuelnog appliance-a

| Scenario | Postupak |
|---|---|
| **VA na verziji 19.17+ sa SLES 15** | Direktno na 20.3 |
| **VA nije na SLES 15** | 1. Pokrenuti najnoviji SLES 12 OS update<br>2. Pokrenuti SLES 15 SP4 OS update (SP6 je opcion; prvo SP4, pa SP6)<br>3. Nadograditi na 20.3 |
| **VA na verziji 19.16 ili starijoj** | 1. Najnoviji SLES 12 OS update<br>2. SLES 15 SP4 OS update<br>3. 19.16 → 19.20; verzije 19.15 i starije → 19.16 → 19.20<br>4. Nadograditi na 20.3 |
| **Samo OS na SLES 15 SP4, ostati na trenutnoj CR verziji** | Za VA na 19.13+: najnoviji SLES 12 update, pa SLES 15 SP4 update |

> **Kritične napomene:**
> - Nadogradnja **ne uspeva** ako CR radi na SLES 12.
> - VA na SLES 12 **ne može** se direktno nadograditi na SLES 15 SP6 — prvo SP4.
> - Kada nadograđujete VA na **SLES 15 SP6, morate nadograditi i CR na 20.3**, zbog migracije sa SuSEfirewall2 na firewalld.
> - SLES 15 SP4 je podržan samo na VA sa CR verzijom 19.13 ili novijom.

## 34.4 Prenos update fajlova u vault

Procedura za slučaj kada nemate fizički pristup vault-u ili ne želite da unosite laptop/eksterni disk.

**Preduslovi:**
- Na produkcionom DD sistemu kreiran **namenski MTree**
- Na produkcionom i vault DD sistemu kreirana i inicijalizovana DD replikacija
- U CR kreirana politika sa replication kontekstom pridruženim tom MTree-u
- Opciono, za VA: backup podataka i VM snapshot, sačuvani van appliance-a

**Koraci:**

1. Nabaviti softverski ili OS fajl.
2. U produkcionom okruženju postaviti update softver na host.
3. Na produkcionom DD sistemu eksportovati namenski MTree ka tom hostu.
4. Sa tog hosta NFS montirati produkcioni MTree.
5. Preuzeti update softver na NFS lokaciju.
6. **Izvršiti checksum i pokrenuti scanner** da se potvrdi da preuzeti softver nije oštećen.
7. Opciono testirati update na test sistemu.
8. U CR izvršiti **Sync Copy** operaciju da se replicira MTree sa update softverom.
9. Po završetku Sync Copy job-a kreirati **CR sandbox** kopije i eksportovati ga ka hostu na kojem želite pristup softveru.
10. Opciono: pokrenuti scanner ili izvršiti analizu preko Cyber Detect-a.
11. Iz CLI-ja pokrenuti `crsetup.sh --save` za kreiranje backup kopije.
12. Kopirati update na CR management host u direktorijum po izboru.

> **Ovo je operativno elegantno rešenje** — koristi sam Cyber Recovery mehanizam da bezbedno unese fajlove u vault, sa mogućnošću skeniranja i analize pre izvršavanja.

## 34.5 Priprema pre maintenance prozora

Zadaci koje treba obaviti **unapred**, radi ranog otkrivanja problema (npr. nedovoljno prostora):

| # | Zadatak |
|---|---|
| 1 | **Bare-metal, sveža instalacija 20.3/20.2 ili nadogradnja sa 20.2:** ukloniti Docker i docker-compose i **rebootovati** |
| 2 | **OVA:** uklanjanje Docker-a nije potrebno |
| 3 | Obezbediti **najmanje 20 GB** prostora za update |
| 4 | **VA deploy-ovan pre 19.22:** proveriti slobodan prostor na root particiji; po potrebi proširiti root particiju i fajl sistem |
| 5 | **VA:** kreirati **snapshot** pre nadogradnje |
| 6 | Kreirati backup kopiju (UI DR backup ili `crsetup.sh --save`), pohraniti **van** CR servera |
| 7 | `umask` = 022 |
| 8 | **RHEL 8.10 / 9.6, CR verzije starije od 20.2:** ako je SELinux u Enforcing, postaviti na Permissive (`setenforce 0`) |
| 9 | Nabaviti update paket, preneti ga u vault i raspakovati u staging |
| 10 | **Znati lockbox passphrase** |
| 11 | **Nadogradnja sa verzija starijih od 20.2: NE uklanjati Docker** — proces nadogradnje ga zahteva za PostgreSQL backup i migraciju |
| 12 | **Nadogradnja sa verzija starijih od 19.14:** deployment mora imati **omogućenog Admin korisnika**, inače preupdate provera ne uspeva |

## 34.6 Izmene podešavanja pri nadogradnji na 20.3

| Podešavanje | Ponašanje |
|---|---|
| **Mail servis** | Ako je mail server konfigurisan pre nadogradnje — bez promena. Ako je **Postfix** konfigurisan na CR hostu — instalacija potvrđuje da radi; po završetku mail/relay server se postavlja na **CR hostname**. Ako Postfix **nije** konfigurisan — po završetku je **email podrška onemogućena**; prijaviti se i omogućiti je po potrebi. |
| **TLS** | Nadogradnja sa 19.12 ili starije — instalacija pita da li omogućiti TLS. Sa 19.13 ili novije — TLS je podrazumevano omogućen, bez pitanja. |
| **Trajanje rasporeda izveštaja** | Rasporedi duži od 365 dana se **skraćuju na 365 dana**. Kraći zadržavaju podešavanje. |

## 34.7 Procedura nadogradnje

> **Proces je disruptivan** — svi Podman kontejneri se zaustavljaju i ponovo pokreću.
> **Nadogradnja nema uticaja** na postojeće asset-e, politike i druge CR objekte.

**Preduslovi:**
- Zadovoljeni svi sistemski zahtevi
- Update fajl prenet u vault
- **Zaustavljeni CR servisi i primenjene najnovije bezbednosne zakrpe OS-a / VA pre nadogradnje**
- Backup kopija sačuvana van CR servera

**Koraci:**

```bash
# 1. Prijaviti se na management host kao root

# 2. Preuzeti update paket u direktorijum sa ~5 GB slobodnog prostora
#    CR paket = tar.gz ; CR virtuelni appliance paket = .bin

# 3. Raspakovati
tar -xzvf <ime_fajla>

# 4. Preći u staging direktorijum
cd staging

# 5. Provera spremnosti (ako nije rađena ranije ili se deployment promenio)
./crsetup.sh --upgcheck
```

> Ako provera prijavi neispunjene preduslove, ispraviti ih i ponoviti dok ne prođe. **Ako je neki potreban servis ugašen, provera generiše alert i ne uspeva.**
>
> Tokom provere i nadogradnje CR validira verzije Podman-a i Podman Compose-a. Ako su nepodržane, instalater pita:
> `Do you want proceed with the Podman offline install/upgrade (y/n)`

Nastavak: pokrenuti `./crsetup.sh --upgrade` i pratiti upite.

> **Upozorenje o Cyber Detect-u:** ako deployment koristi Cyber Detect, nastavak se potvrđuje samo ako nema analyze operacija u toku ni zakazanih, i ako su DD Boost korisnici konfigurisani na Cyber Detect serveru za svaki DD sistem koji dostavlja podatke za analizu. U suprotnom slediti uputstva iz poruke upozorenja.

## 34.8 Koraci posle nadogradnje

**RHEL 8.10 / 9.6, deployment stariji od 20.2** — ako ste postavili SELinux na Permissive pre nadogradnje:

```bash
getenforce
setenforce 1

# Primena SELinux politika i labela
./crsetup.sh --forcerecreate
```

**SLES 16, CR verzija 20.1 sa SELinux u Permissive** — postaviti na Enforcing posle nadogradnje:

```bash
getenforce
setenforce 1
```

## 34.9 Primena bezbednosnih zakrpa OS-a

```bash
# 1. Nabaviti fajl sa Dell Online Support
#    cyber-recovery-osupdate-<current release>.bin

# 2. Preneti fajl u vault (videti 34.4)

# 3. Zaustaviti CR softver
./crsetup.sh --stop

# 4. Pokrenuti update
./cyber-recovery-osupdate-<current release>.bin

# 5. Potvrditi automatski reboot na upit
```

> **Reboot je obavezan**, inače mogu nastati problemi. Po rebootu se CR servisi pokreću automatski.
> Posle reboota **verifikovati da softver radi**.

> Isti `osupdate` binary se koristi i za sinhronizaciju Podman verzija sa najnovijim dostupnim, kao i za ažuriranje paketa na OVA (npr. chrony, rsyslog).

---

# 35. Migracija deploymenta

> **CentOS nije podržan.** Postojeće CentOS deployment-e migrirati na RHEL ili SLES.

Tri scenarija migracije. Sva tri prate isti obrazac: **sačuvati konfiguraciju → instalirati novo okruženje → izvršiti recovery**.

## 35.1 Migracija na virtuelni appliance

```bash
# 1. Zaustaviti sve job-ove u toku

# 2. Verifikovati Postgres i lockbox passphrase
crsetup.sh --verifypassword

# 3. Zabeležiti podatke o deploymentu:
#    IP adresa, DNS, FQDN, mail server (Postfix), firewall podešavanja

# 4. Kreirati backup kopiju (UI DR backup ILI CLI)
crsetup.sh --save
#    Backup se čuva u /opt/dellemc/cr-configs — sačuvati ga VAN CR servera
```

5. Deploy-ovati Cyber Recovery virtuelni appliance u vault.

   > **Verzija VA mora biti ista** kao CR verzija iz koje je napravljen backup.

6. Kopirati novokreirani backup fajl na virtuelni appliance.
7. Izvršiti recovery:
   - Ako je backup napravljen preko UI-a: montirati CR backup MTree na CR host
   - Ako je napravljen preko CLI-ja: kopirati fajl u `/opt/dellemc/cr-configs` i pokrenuti:

```bash
crsetup.sh --recover
```

## 35.2 Migracija na drugi OS na istom serveru

```bash
# 1. Verifikovati passphrase
./crsetup.sh --verifypassword

# 2. Zabeležiti podatke o deploymentu:
#    IP adrese, DNS, FQDN, mail server (Postfix), firewall podešavanja
#    Podman i Podman Compose verzije

# 3. Zaustaviti sve job-ove u toku

# 4. Kreirati backup kopiju
crsetup.sh --save
```

5. Deploy-ovati RHEL ili SLES na server, pa:
   - a. Konfigurisati mrežu, DNS, FQDN, mail server (Postfix) i firewall na **iste vrednosti** kao pre
   - b. Deploy-ovati Podman i Podman Compose
   - c. Uskladiti deployment sa preporukama iz *Installation Guide*-a

6. Instalirati **istu verziju** softvera koja je radila na originalnom deploymentu.

   > **Preporuka:** postaviti Postgres i lockbox passphrase na **iste** vrednosti kao na originalnom deploymentu. Time je oporavak znatno jednostavniji.

7. Izvršiti recovery kao u 35.1, korak 7.

## 35.3 Migracija na drugi server

**Preduslov:** server mora raditi pod Red Hat Enterprise Linux ili SUSE Linux Enterprise Server.

> Ovi koraci se mogu koristiti i za migraciju **sa virtuelnog appliance-a na softversku instalaciju**.

```bash
# 1. Na trenutnom serveru verifikovati passphrase
crsetup.sh --verifypassword

# 2. Zabeležiti podatke o deploymentu:
#    Mrežna podešavanja (IP, DNS, FQDN, mail server, firewall)
#    Podman i Podman Compose verzije

# 3. Kreirati backup kopiju
crsetup.sh --save
#    Sačuvati na eksterni disk
```

4. Instalirati **istu verziju** CR softvera koja je radila na originalnom deploymentu, sa **istim passphrase-ima**.
5. Izvršiti recovery kao u 35.1, korak 7.

## 35.4 Sažetak — šta obavezno zabeležiti pre migracije

| # | Podatak |
|---|---|
| 1 | IP adresa(e) |
| 2 | DNS serveri |
| 3 | FQDN |
| 4 | Mail server / Postfix konfiguracija |
| 5 | Firewall podešavanja |
| 6 | Podman verzija |
| 7 | Podman Compose verzija |
| 8 | Instalacione lokacije |
| 9 | Lockbox passphrase |
| 10 | Postgres lozinka |
| 11 | `crso` lozinka |
| 12 | CR verzija |

---

# 36. Disaster recovery Cyber Recovery servera

## 36.1 Restore softverske instalacije

**Preduslovi:**
- Postoji CR backup tar paket kreiran **pre** incidenta. **Bez njega se procedura ne može izvršiti.**
- Obrisan CR instalacioni direktorijum

**Koraci:**

1. Ponovo instalirati CR softver.

   > **Preporuka:** koristiti **iste lozinke** kao na originalnoj instalaciji — time je procedura oporavka jednostavnija. Takođe koristiti **iste instalacione lokacije**.

2. Po završetku instalacije pokrenuti UI i **potvrditi da je konfiguracija prazna**.
3. Zatvoriti UI.
4. Pokrenuti proceduru vraćanja:

```bash
crsetup.sh --recover
```

```
Do you want to continue [y/n]:            → y
Are you REALLY sure you want to continue [y/n]:  → y
```

5. Uneti punu putanju do CR backup tar paketa, na primer:

```
/tmp/cr_backups/cr.19.2.1.0-3.2019-09-19.08_02_09.tar.gz
```

6. Uneti **lockbox passphrase originalne instalacije** — one pre incidenta:

```
Enter the previously saved lockbox passphrase:
```

Uspešan završetak:

```
Cyber Recovery has been successfully recovered onto this system
```

7. Prijaviti se u CR UI ili CRCLI i **validirati da je prethodna instalacija vraćena**.

## 36.2 Restore virtuelnog appliance deploymenta

**Preduslovi:** isti kao u 36.1.

1. Ponovo deploy-ovati virtuelni appliance. Dve opcije:
   - Preuzeti i deploy-ovati verziju VA koju želite da koristite
   - Deploy-ovati verziju VA koja je trenutno u vault-u; po potrebi nadograditi na noviju

2. Pokrenuti proceduru vraćanja:

```bash
crsetup.sh --recover
```

```
Do you want to continue [y/n]:            → y
Are you REALLY sure you want to continue [y/n]:  → y
```

3. Uneti punu putanju do backup tar paketa.
4. Uneti lockbox passphrase originalne instalacije.
5. Validirati vraćenu konfiguraciju.

## 36.3 Napomena o lockbox passphrase-u

> Cela procedura oporavka CR servera zavisi od **lockbox passphrase-a originalne instalacije**. Ako je izgubljen, backup je neupotrebljiv i potrebna je sveža instalacija sa gubitkom cele konfiguracije.
>
> **Ovo je najkritičniji podatak u celom deploymentu.** Obezbediti da bude pohranjen redundantno, van CR servera, u sistemu za upravljanje tajnama kupca ili u fizičkom sefu.

---

# 37. Troubleshooting

## 37.1 Instalacija i pokretanje

| Simptom | Provera i rešenje |
|---|---|
| **CR softver se ne instalira** | • `crsetup.sh --check` mora proći sve preduslove<br>• Koristiti **stabilnu** verziju Podman-a<br>• Podman socket automatski omogućava `crsetup.sh`; ručno: `systemctl enable podman.socket`<br>• `crsetup.sh` logovi su u direktorijumu iz kojeg je komanda pokrenuta<br>• Otvoriti portove 14777, 14778, 14780 na firewall-u |
| **Ne može se prijaviti u CR UI** | • Proveriti `edge` i `users` servisne logove<br>• DNS podešavanja moraju biti rezolvabilna<br>• Otvoriti portove 14777, 14778, 14780 |
| **CR se ne pokreće posle reboota (SELinux)** | Promeniti SELinux kontekst za `cradmin`, `crcli`, `crsetup.sh`, `crshutil`, `crsshutil` (`chcon -u system_u -t bin_t ...`) i rebootovati |
| **Ne može se omogućiti MFA** | Vreme CR hosta ne sme odstupati više od ±60 sekundi od vremena authenticator-a. Podesiti vreme; ako ga menjate, **zaustaviti pa pokrenuti CR servise**. Preporučena je interna NTP konfiguracija. |

## 37.2 Mreža i NFS

| Simptom | Provera i rešenje |
|---|---|
| **Politika, sandbox ili recovery ne rade zbog mount grešaka sa DD sistema** | DD UI → **Protocols → NFS → Options**:<br>• Samo NFSv3: `Default Export Version` i `Default Servers Version` = NFSv3<br>• Samo NFSv4: obe = NFSv4 i `NFSv4 ID Map Out Numeric` = `always`<br>• Oba: obe = NFSv3 i NFSv4, `NFSv4 ID Map Out Numeric` = `always` |
| **Kolizija Podman subnetova** | Readresirati mreže: `crsetup.sh --forcerecreate --changeiprange` |
| **Vault status Degraded ili Unknown** | Promenjen FQDN ili IP DD sistema. Proveriti komunikaciju CR-a sa svakim DD sistemom. |

## 37.3 PowerProtect Data Manager

| Simptom | Provera i rešenje |
|---|---|
| **PPDM recovery ne uspeva** | Ako je na vault DD omogućen samo NFS v4 — ili ga onemogućiti i koristiti NFS v3, ili koristiti **DD Boost za server DR MTree** |
| **DR backup job prikazan kao critical failed posle oporavka DR backup-a** | **Ignorisati taj status** — očekivano ponašanje |

## 37.4 NetWorker

| Simptom | Rešenje |
|---|---|
| **NetWorker recovery se ne završava čisto** | Ručno čišćenje: zaustaviti NetWorker, vratiti prethodne `res`, `mm` i `index` baze preimenovanjem `.cr.<timestamp>` direktorijuma, pokrenuti NetWorker. Puna procedura u Delu IV, poglavlje 23.4. |

## 37.5 Sertifikati i OIDC

| Simptom | Rešenje |
|---|---|
| **Sertifikat se ne validira pri dodavanju Cyber Detect 8.12 kao aplikacije** | Pokrenuti `generate_cert` komandu na Cyber Detect sistemu, ako se ne koristi custom sertifikat |
| **Ne može se dohvatiti servis** | Proveriti logove. **Ako je Keycloak ugašen, svi CR servisi ne mogu da komuniciraju.** |
| **Problemi sa OIDC komunikacijom** | Pogledati `app.log`. Proveriti da je **TCP port 443 otvoren** između CR management hosta i Cyber Detect servera. |

## 37.6 Uklanjanje DD storage-a iz Cyber Recovery

> **Redosled je obavezan:**
>
> 1. Prvo obrisati **kopije** — brisanjem svih sandbox-ova pridruženih kopiji
> 2. Zatim obrisati **sve politike** — brisanjem svih kopija pridruženih svakoj politici
> 3. Na kraju obrisati **vault storage** — brisanjem svih politika pridruženih tom storage-u

## 37.7 Onemogućavanje SSH pristupa replikacionom interfejsu

Opciona procedura za dodatno ograničavanje pristupa. CR komunicira sa svim DD sistemima preko SSH-a.

```bash
# 1. Na management hostu utvrditi hostname
hostname

# 2. Prijaviti se na DD host i dodati hostname u listu dozvoljenih
sysadmin@dd-vault# adminaccess ssh add <hostname>
```

**Rezultat:** SSH je blokiran na svim interfejsima osim management interfejsa.

> Za dodatnu kontrolu koristiti DD **net filter** funkcionalnost — videti DD dokumentaciju.

## 37.8 Prikupljanje podataka za Dell Support

1. Generisati support bundle: **System Settings → Support → Support Bundles → Generate Log Bundle**
2. Preuzeti bundle i checksum fajl
3. Preuzeti i DD kolekcije iz PowerProtect DD Management Center-a
4. Po potrebi privremeno prebaciti nivo logovanja na **Debug** i reprodukovati problem
5. Priložiti izlaz `./crsetup.sh --check`
6. Priložiti verzije: CR, DDOS, OS, Podman, Podman Compose, backup aplikacije

---

# 38. Deinstalacija

> Deinstalacija **briše sve CR komponente**, uključujući bazu, korisničke interfejse i log fajlove.
> **Nije potrebno zaustaviti CR servise pre deinstalacije.**

```bash
./crsetup.sh --uninstall
```

1. Na upit za potvrdu uneti `y`.
2. Na upit da li želite da sačuvate konfiguraciju — izabrati opciju.

   > Ako potvrdite, procedura koristi `tar` da sačuva Postgres fajlove, log fajlove i lockbox fajlove kao komprimovani `.gz` fajl. Konfiguracija se može sačuvati i nezavisno od deinstalacije, komandom `./crsetup.sh --save`.

> **Preporuka:** uvek sačuvati konfiguraciju, čak i kada je deinstalacija planirana kao trajna. Sačuvana konfiguracija omogućava kasniji oporavak.

---

# Dodatak A: Zapisnik o primopredaji

## A.1 Osnovni podaci

| Stavka | Vrednost |
|---|---|
| Kupac | |
| Lokacija vault-a | |
| Datum implementacije | |
| System Engineer | |
| Kontakt kod kupca | |
| Broj projekta / SR | |

## A.2 Implementirano rešenje

| Komponenta | Verzija / model | Napomena |
|---|---|---|
| Cyber Recovery | 20.3 | |
| Način deploymenta | RHEL / SLES / OVA | |
| Operativni sistem management hosta | | |
| Podman / Podman Compose | | |
| Vault DD sistem(i) | | |
| DDOS verzija | | |
| Produkcioni DD sistem(i) | | |
| Immutability tehnologija | Retention Lock / Secure Snapshot | |
| NetWorker | | |
| PowerProtect Data Manager | | |
| Avamar | | |
| Cyber Detect | | |
| vCenter u vault-u | | |

## A.3 Konfigurisane politike

| Ime politike | Tip | Storage target | Broj konteksta | Retencija | Raspored |
|---|---|---|---|---|---|
| | | | | | |
| | | | | | |
| | | | | | |

## A.4 Rezultati validacije

| Test | Rezultat | Trajanje | Napomena |
|---|---|---|---|
| Inicijalna replikacija | | | |
| Sync operacija | | | |
| Copy operacija | | | |
| Lock operacija | | | |
| Test recovery — NetWorker | | | |
| Test recovery — PPDM | | | |
| Recovery Check | | | |
| DR backup CR konfiguracije | | | |
| Restore CR konfiguracije (opciono) | | | |

## A.5 Predati podaci i pristupi

| # | Stavka | Predato | Način predaje |
|---|---|---|---|
| 1 | **Lockbox passphrase** | | |
| 2 | Postgres lozinka | | |
| 3 | `crso` lozinka | | |
| 4 | Admin korisnik i lozinka | | |
| 5 | Vault Operator korisnik i lozinka | | |
| 6 | DD `cradmin` nalog | | |
| 7 | DD Security Officer nalog | | |
| 8 | Spisak DD Boost naloga sa UID-ovima | | |
| 9 | OS `root` lozinka management hosta | | |
| 10 | Lokacija CR backup kopije | | |
| 11 | Ova dokumentacija | | |

> **Lockbox passphrase se predaje kontrolisano i evidentirano.** Bez njega kupac ne može izvršiti nadogradnju, reset Security Admin lozinke ni oporavak CR servera.

## A.6 Otvorene stavke i ograničenja

| # | Stavka | Vlasnik | Rok |
|---|---|---|---|
| | | | |
| | | | |

## A.7 Preporuke za kupca

| # | Preporuka |
|---|---|
| 1 | Lockbox passphrase čuvati redundantno, van CR servera |
| 2 | Redovno proveravati status DR backup-a CR konfiguracije |
| 3 | Pratiti popunjenost vault DD sistema (pragovi 80% / 90%) |
| 4 | Zakazati periodični Recovery Check |
| 5 | Pratiti alerte i konfigurisati email notifikacije |
| 6 | Pre svake nadogradnje: snapshot (VA) i backup konfiguracije |
| 7 | Poštovati redosled gašenja i pokretanja komponenti vault-a |
| 8 | Držati verzije backup aplikacija usklađene između produkcije i vault-a |
| 9 | Ne koristiti `sysadmin` nalog za Cyber Recovery operacije |
| 10 | Periodično testirati pun scenario oporavka |

**Potpisi:**

| Uloga | Ime | Potpis | Datum |
|---|---|---|---|
| System Engineer | | | |
| Predstavnik kupca | | | |

---

# Dodatak B: Operativni kalendar

Predlog periodičnih zadataka za kupca.

## Dnevno

| Zadatak | Gde |
|---|---|
| Pregled dashboard-a i alerta | Main Menu → Dashboard |
| Provera statusa job-ova | Main Menu → Jobs |
| Provera da se vault status vraća u **Locked** | Dashboard → Status |
| Provera uspešnosti DR backup-a CR konfiguracije | System Settings → DR Backups |

## Sedmično

| Zadatak | Napomena |
|---|---|
| Provera popunjenosti vault storage-a | Infrastructure → Storage → Capacity |
| Pregled isteklih i otključanih kopija | |
| Provera da DD file system cleanup ne kolidira sa CR job-ovima | DD cleanup podrazumevano utorkom |
| Provera NTP sinhronizacije svih komponenti | `chronyc tracking` |

## Mesečno

| Zadatak | Napomena |
|---|---|
| Recovery Check nad reprezentativnom kopijom | Zakazano ili on-demand |
| Izvoz job-ova i sandbox-ova u CSV za evidenciju | |
| Provera isteka sertifikata (TLS, PPDM OIDC) | |
| Provera bezbednosnih zakrpa OS-a | |
| Provera slobodnog prostora za nadogradnje (≥ 20 GB) | |

## Kvartalno

| Zadatak | Napomena |
|---|---|
| Pun test oporavka aplikacije u vault-u | Sa evidentiranjem trajanja |
| Provera i po potrebi rotacija lozinki | `crsetup.sh --changepassword` |
| Verifikacija dostupnosti lockbox passphrase-a | Bez otkrivanja vrednosti |
| Pregled korisničkih naloga i uloga | |
| Provera usklađenosti verzija produkcija ↔ vault | |
| Provera matrice kompatibilnosti (E-Lab Navigator) | |

## Godišnje ili po potrebi

| Zadatak | Napomena |
|---|---|
| Nadogradnja CR softvera | Prema putanjama iz poglavlja 34.2 |
| Nadogradnja DDOS-a | Prvo komponente, pa CR |
| Nadogradnja backup aplikacija | Uskladiti produkciju i vault |
| Pregled retencionih politika u odnosu na poslovne zahteve | |
| Revizija air-gap dizajna i firewall pravila | |

---

*Kraj Dela VI. Ovim je pokriven ceo obim dokumenta prema strukturi iz `CR-20.3-Uputstvo-Struktura.md`.*
