# DEO I — UVOD I PLANIRANJE

**Dell PowerProtect Cyber Recovery 20.3 — Uputstvo za instalaciju i integraciju**
Verzija dokumenta: 0.1 | Namena: System Engineer

> **Izvori:** *Dell PowerProtect Cyber Recovery 20.3 Installation and Upgrade Guide* (Rev. 01), *Dell PowerProtect Cyber Recovery 20.3 Product Guide* (Rev. 01, avgust 2026).

---

## Sadržaj

- [1. O ovom dokumentu](#1-o-ovom-dokumentu)
- [2. Pregled rešenja](#2-pregled-rešenja)
- [3. Planiranje implementacije](#3-planiranje-implementacije)
- [Dodatak: Radni list za planiranje](#dodatak-radni-list-za-planiranje)

---

# 1. O ovom dokumentu

## 1.1 Namena i ciljna grupa

Dokument je namenjen **System Engineer-ima** koji izvode instalaciju, konfiguraciju i integraciju Dell PowerProtect Cyber Recovery rešenja verzije 20.3 kod korisnika.

Pretpostavlja se da čitalac poznaje:
- Osnove Linux administracije (RHEL ili SLES)
- PowerProtect DD sisteme i DD OS
- VMware vSphere okruženje
- Backup aplikacije koje se integrišu (NetWorker, PowerProtect Data Manager)
- Mrežnu segmentaciju i osnove bezbednosti

## 1.2 Obim dokumenta

**Pokriveno:**

| Deo | Sadržaj |
|---|---|
| **Deo I** | Uvod, pregled rešenja, planiranje |
| **Deo II** | Preduslovi (management host, DD, mreža, aplikacije, Podman, priprema DD-a) |
| **Deo III-a** | Instalacija na RHEL/SLES + prva prijava + osnovna konfiguracija u UI |
| **Deo III-b** | Deployment virtuelnog appliance-a (OVA) na VMware |
| **Deo IV** | Integracija sa NetWorker-om |
| **Deo V** | Integracija sa PowerProtect Data Manager-om |
| **Deo VI** | Validacija, primopredaja, operativne procedure, nadogradnja, migracija, DR, troubleshooting |

**Nije pokriveno (van dogovorenog obima):**

| Tema | Napomena |
|---|---|
| Integracija sa **Avamar**-om | Podržano kroz Standard politiku; dokumentacija postoji ako se obim proširi |
| **Cyber Detect / CyberSense** | Pominjano usput; nema zasebnog poglavlja |
| **Sheltered Harbor** | Samo kratak pregled u odeljku 2.7 |
| Deployment u **AWS / Azure / GCP** | Postoje zasebni Dell cloud deployment vodiči |
| **Cyber Recovery CLI (CRCLI)** — puna referenca | Koriste se samo komande potrebne za opisane procedure |
| **REST API** | Van obima |

## 1.3 Konvencije u dokumentu

| Oznaka | Značenje |
|---|---|
| **CAUTION** | Rizik od gubitka podataka ili nemogućnosti oporavka |
| **VAŽNO / KRITIČNO** | Korak koji se ne sme preskočiti |
| `monospace` | Komande, putanje, imena fajlova, izlaz sistema |
| `<vrednost>` | Promenljiva koju zamenjuje stvarna vrednost |
| **Bold** | Elementi korisničkog interfejsa (dugmad, polja, meniji) |

Tehnički termini se ostavljaju u originalu (engleski) kada je to standard u struci: *replication context*, *sandbox*, *retention lock*, *MTree*, *bootstrap*, *point-in-time copy*.

## 1.4 Referentna Dell dokumentacija

### Korišćeno pri izradi ovog dokumenta

| Dokument | Verzija |
|---|---|
| Dell PowerProtect Cyber Recovery Installation and Upgrade Guide | 20.3 |
| Dell PowerProtect Cyber Recovery Product Guide | 20.3 |
| Dell NetWorker Server Disaster Recovery and Availability Best Practices Guide | 19.13+ |
| Dell NetWorker and VMware Integration Guide | 19.13 |
| PowerProtect Data Manager Administrator Guide | 19.22 |
| PowerProtect Data Manager Deployment Guide | 19.22 |

### Potrebno za dopunu (videti Dodatke B u Delovima IV i V)

| Dokument | Za šta |
|---|---|
| E-Lab Navigator / Support Compatibility Matrix | Tačne verzije Podman-a, DDOS-a, pretraživača |
| KB **000205512** | Minimalne, preporučene i najnovije verzije koda |
| KB **198370** | Root squash podešavanja na DD-u |
| KB **205800** | Port 3009 i replikacioni parovi |
| Dell PowerProtect Data Domain Operating System Administration Guide | Replication contexts, MTree, Retention Lock, ifGroups, net filter |
| Dell PowerProtect Cyber Recovery Security Configuration Guide | Sertifikati, hardening, FIPS, audit logging |
| Dell PowerProtect Cyber Recovery Command-Line Interface Reference Guide | Puna CRCLI referenca |
| PowerProtect Data Manager 20.3 Administrator / Deployment Guide | Ako se ide na PPDM 20.3+ |
| NetWorker Installation Guide, NetWorker Administration Guide | Deployment neinicijalizovane instance, autochanger, notifikacije |

## 1.5 Rečnik pojmova

| Pojam | Objašnjenje |
|---|---|
| **Cyber Recovery vault** | Kupčeva bezbedna lokacija koja predstavlja odredište DD MTree replikacije. Obuhvata DD sistem, management host, backup i analitičke aplikacije. Fizički ili virtuelno izolovana od produkcije. |
| **Air gap** | Princip po kojem je vault povezan sa produkcijom samo onoliko dugo koliko traje replikacija; u svim ostalim trenucima je odvojen |
| **CR management host** | Fizički server ili VM na kojem radi Cyber Recovery softver |
| **MTree** | Logička particija fajl sistema na PowerProtect DD sistemu |
| **Replication context** | Definisana veza između izvornog MTree-a na produkcionom DD i odredišnog MTree-a u vault-u |
| **PIT copy** | Point-in-time kopija — tačka vraćanja kreirana iz poslednje replikacije |
| **Repository MTree** | MTree u kojem se čuvaju PIT kopije kod repository-based politika. Konvencija imenovanja: `/data/col1/cr-policy-<policyID>-repo` |
| **Retention Lock** | DD tehnologija nepromenljivosti (DDOS ≤ 8.7). Aktivira se po MTree-u, retencija se postavlja po fajlu. |
| **Secure Snapshot** | DD tehnologija nepromenljivosti (DDOS ≥ 8.8). Aktivira se na nivou sistema, kreira nepromenljive snapshot-ove na replikacionom target-u. |
| **Sandbox** | Jedinstvena lokacija u vault-u sa read/write kopijom zaključanih podataka, nad kojom se izvode operacije bez uticaja na original |
| **Recovery sandbox** | Sandbox kreiran u sklopu automatizovanog oporavka aplikacije |
| **Politika (policy)** | Kombinacija objekata (DD storage, aplikacije) i job-ova (sync, copy, lock) koja orkestrira tok između produkcije i vault-a |
| **crso** | Podrazumevani Security Admin nalog kreiran pri instalaciji |
| **Lockbox** | Enkriptovano skladište tajni Cyber Recovery softvera, zaštićeno passphrase-om |
| **DD Boost** | Dell protokol za optimizovanu komunikaciju backup aplikacija sa DD sistemom |
| **Bootstrap** | NetWorker save set sa ključnim konfiguracionim informacijama servera |
| **Server DR** | PPDM mehanizam periodičnog backup-a sopstvene konfiguracije i baza |

---

# 2. Pregled rešenja

## 2.1 Šta je Cyber Recovery

Cyber Recovery održava poslovno kritične podatke i tehnološke konfiguracije u **bezbednom, air-gapped „vault" okruženju**, koje se može koristiti za oporavak ili analizu. Vault je fizički ili virtuelno izolovan od produkcionog sistema ili mreže, u zavisnosti od tipa deploymenta.

**Osnovni princip rada:**

Rešenje omogućava pristup vault-u **samo onoliko dugo koliko je potrebno da se podaci repliciraju** sa produkcionog sistema. U svim ostalim trenucima vault je obezbeđen i odvojen od produkcione mreže.

> **Zašto DD deduplikacija ovde nije samo optimizacija:** deduplikacija se izvršava u produkcionom okruženju kako bi se ubrzao proces replikacije — **da bi vreme konekcije ka vault-u bilo što kraće**. Kraća konekcija znači manju površinu izloženosti.

Unutar vault-a, Cyber Recovery softver kreira **point-in-time, retention-locked kopije** koje se mogu validirati i zatim iskoristiti za oporavak produkcionog sistema.

> Vault se može deploy-ovati i na Amazon Web Services, Microsoft Azure ili Google Cloud Platform.

## 2.2 Arhitektura

```
   PRODUKCIONO OKRUŽENJE                    CYBER RECOVERY VAULT
   ─────────────────────                    ────────────────────

   NetWorker / PPDM / Avamar                CR management host
            │                                  (CR softver)
            │ backup                                 │
            ▼                                        │ upravlja
   Produkcioni DD sistem                             ▼
            │                              ┌──────────────────┐
            │  MTree replikacija           │  Vault DD sistem │
            │  (namenski replikacioni      │                  │
            └──── data link) ─────────────►│  • repl. MTree   │
                                           │  • repository    │
   [veza otvorena samo tokom               │  • recovery      │
    replikacije]                           └──────────────────┘
                                                     │
                                           NetWorker / PPDM / Avamar
                                           (za oporavak aplikacija)
                                                     │
                                           Cyber Detect (opciono)
```

### Produkciono okruženje

Aplikacije kao što su Avamar, NetWorker i PowerProtect Data Manager upravljaju backup operacijama i skladište backup podatke u MTree-ovima na DD sistemima. Produkcioni DD sistem je konfigurisan da replicira podatke ka odgovarajućem DD sistemu u vault-u.

### Vault okruženje

Vault obuhvata **CR management host** (na kojem radi Cyber Recovery softver) i **DD sistem**. Ako je potrebno za oporavak aplikacija, vault može uključivati i NetWorker, Avamar, PowerProtect Data Manager i druge aplikacije.

Vault je kupčeva bezbedna lokacija koja predstavlja odredište DD MTree replikacije. Zahteva **namenske resurse, uključujući mrežu**, a — iako nije obavezno, snažno se preporučuje — servis imena (DNS) i izvor vremena.

> Vault može biti na drugoj lokaciji, na primer kod service provider-a.

### Kako se kontroliše protok podataka

> Cyber Recovery softver **omogućava i onemogućava replikacioni Ethernet interfejs i replication context** na DD sistemu u vault-u, čime kontroliše protok podataka iz produkcije ka vault-u. Na kratke periode vault je povezan sa produkcijom preko tog namenskog interfejsa radi replikacije.
>
> **Management interfejs je uvek omogućen**, pa se ostale Cyber Recovery operacije izvršavaju i dok je vault obezbeđen.

Ovo je ključna arhitekturna činjenica: air gap se odnosi na **replikacioni put**, ne na upravljanje. CR može da radi svoj posao u vault-u i kada je veza sa produkcijom zatvorena.

## 2.3 Nepromenljivost (immutability)

Nepromenljivost je **obavezna** za vault okruženje. PowerProtect DD je obezbeđuje kroz dve tehnologije, u zavisnosti od DDOS verzije:

| DDOS | Tehnologija | Nivo aktivacije | Postavljanje retencije |
|---|---|---|---|
| **≤ 8.7** | PowerProtect DD Retention Lock | Po MTree-u | Po fajlu |
| **≥ 8.8** | PowerProtect DD Secure Snapshot | Na nivou sistema | Nepromenljivo od trenutka kreiranja; retencija se može produžiti |

### Automatska migracija na Secure Snapshot

Cyber Recovery **automatski migrira** repository-based politike na Secure Snapshot kada DD sistem ispuni uslove.

> **Migracija je automatska i trajna.** Kada politika pređe na Secure Snapshot, **ne može se vratiti** na repository-based politiku.

**Preduslovi na DD sistemu:**
- DDOS 8.8 ili noviji
- Retention Lock Compliance omogućen
- MTree Scaling omogućen (može zahtevati restart fajl sistema)

**Uslovi koje politika mora ispuniti:**
- Ima najmanje jedan lock-related raspored
- **Nema Copy ni Sync Copy rasporede**
- Procenjena upotreba snapshot-ova ne prelazi podržani DD limit
- Min i max vrednosti retention lock-a su u podržanom opsegu za Secure Snapshot
- **Sheltered Harbor politike se ne migriraju**

**Kada CR proverava mogućnost migracije:**
- Pri nadogradnji CR-a, ako je DD već Secure Snapshot capable
- Kada DD postane Secure Snapshot capable posle nadogradnje CR-a
- Kada se repository-based politika izmeni na način koji utiče na podobnost
- Za PPDM politike — kada se izmene politike koje dele isti Server DR replication context

> **Prednosti Secure Snapshot-a:** brže kreiranje kopija, kreiranje sandbox-ova, produžavanje retencije i čišćenje isteklih kopija, jer se te operacije obavljaju na nivou snapshot-a umesto kroz operacije nad fajlovima u repository-ju.

> **Postojeće repository kopije se ne konvertuju** — ostaju dostupne za oporavak i analizu do isteka.

## 2.4 Cyber Recovery operacije

| Operacija | Opis |
|---|---|
| **Replication (Sync)** | DD MTree replikacija sa produkcionog na vault DD. Svaka replikacija koristi DD deduplikaciju za inkrementalno usklađivanje podataka u vault-u. |
| **Copy** | Kreira se PIT kopija iz najnovije replikacije. Služi kao tačka vraćanja. Može se održavati više PIT kopija radi optimalnog broja tačaka vraćanja. |
| **Lock** | Zaštita PIT kopije od izmene u zadatom trajanju. |
| **Analyze** | Analiza zaključanih ili nezaključanih kopija alatima koji traže indikatore kompromitacije, sumnjive fajlove ili potencijalni malver. Anomalije mogu označiti kopiju kao neispravan izvor za oporavak. |
| **Recovery** | Upotreba podataka iz PIT kopije za oporavak. |
| **Recovery Check** | Zakazana ili on-demand provera da se kopija može oporaviti. |

> Sve operacije osim oporavka mogu se **zakazati** ili pokrenuti **ručno**.

### Ključna napomena o Sync operaciji

> **Cyber Recovery softver sam ne inicira replikaciju.** On čeka da produkcioni DD sistem sinhronizuje svoje podatke preko replikacionog interfejsa, a zatim validira timestamp repliciranih podataka na vault DD sistemu.
>
> Zbog toga pri Sync akciji može doći do **kašnjenja do 15 minuta**, u zavisnosti od replikacionog ciklusa na produkcionom DD sistemu.

Ovo objašnjava zašto se rasporedi u CR-u ne mogu postaviti „uz sam kraj" produkcionog backup-a — treba ostaviti rezervu.

## 2.5 Akcije politike

| Akcija | Šta radi |
|---|---|
| **Sync** | Replicira MTree sa produkcije u vault, sinhronizujući se sa prethodnom replikacijom |
| **Copy** | Kreira PIT kopiju najnovije replikacije i skladišti je u replication archive |
| **Sync Copy** | Sync + Copy u jednom zahtevu |
| **Copy Lock** | Repository-based: kreira PIT kopiju u repository MTree-u i primenjuje retention lock.<br>Secure Snapshot: ne kreira repository kopiju — identifikuje najnoviji snapshot na replication kontekstu i obezbeđuje ga; ako je već obezbeđen, referencira postojeći. |
| **Secure Copy** | Repository-based: Sync + Copy + Lock.<br>Secure Snapshot: replikacija + kreiranje nepromenljivog Secure Snapshot-a na replikacionom target-u. |
| **Secure Copy Analyze** | Sync + Copy + Lock svih fajlova + Analyze nad rezultujućom kopijom. Dostupno **samo** ako je Cyber Detect instaliran. |
| **Sheltered Harbor Copy** | Samo ako je Sheltered Harbor omogućen. Kreira enkriptovanu retention-locked kopiju prema preporučenoj proceduri. Jedina dostupna opcija za Sheltered Harbor politiku. |

**Ograničenja:**

| Ograničenje | Detalj |
|---|---|
| **Secure Snapshot politike** | Podržavaju **samo akcije koje kreiraju zaključane ili nepromenljive kopije**. Copy i Sync Copy nisu dostupni. |
| **Automatic retention lock (ARL)** | Postojeće ARL politike podržavaju samo Secure Copy Analyze, Secure Copy, Copy Lock i Sync akcije |
| **Konkurentnost** | **Ne mogu se izvršavati istovremene Sync ni Lock akcije za istu politiku.** Ako pokrenete politiku pa je ponovo pokrenete sa akcijom koja izvršava sync ili lock, CR prikazuje informativnu poruku i ne kreira job. |

## 2.6 Politike i kopije

### Politike

| Činjenica | Detalj |
|---|---|
| Politika može upravljati jednim ili više DD MTree-ova | **Samo PPDM tip politike može upravljati sa više od jednog MTree-a** |
| Maksimalan broj politika | Do **50 politika** za maksimalno **10 DD sistema** u vault-u, u zavisnosti od DD modela i drugih faktora |
| Politike se mogu | Kreirati, menjati, brisati, izvoziti, onemogućiti |
| Onemogućena politika | Njeni replication konteksti se mogu iskoristiti za novu politiku. **Ako iskoristite kontekste onemogućene politike, tu politiku više ne možete omogućiti.** Kopija onemogućene politike se i dalje može koristiti za oporavak. |

### Kopije

Kopije su PIT tačke vraćanja. U zavisnosti od tipa nepromenljivosti politike, kopija se čuva u repository MTree-u ili se kreira kao Secure Snapshot na replikacionom target-u.

| Tip kopije | Mogućnosti |
|---|---|
| **Repository-based** | Može se analizirati, primeniti retention lock, obrisati ako je otključana |
| **Secure Snapshot** | Nepromenljiva od trenutka kreiranja, **ne može se ručno obrisati** |

### Tipovi politika

| Tip | Obuhvata |
|---|---|
| **Standard** | NetWorker, Avamar, Filesystem, Other |
| **PPDM** | PowerProtect Data Manager — zahteva **najmanje dva MTree-a** |
| **Sheltered Harbor** | Prikazuje se samo ako je funkcija omogućena |

## 2.7 Sheltered Harbor — kratak pregled

Rešenje koje ispunjava tehničke zahteve *Sheltered Harbor Turnkey Data Vaulting Solution* specifikacije, namenjeno finansijskom sektoru.

- Vault obezbeđuje izolovano i enkriptovano okruženje koje ispunjava zahteve bezbednosti, poverljivosti i integriteta standarda
- Podaci su dostupni za brzo vraćanje poznato dobre kopije radi oporavka kritičnih sistema i nastavka pružanja finansijskih usluga
- Automatski sinhronizuje podatke između produkcionih/backup sistema i vault-a, i čuva nepromenljive kopije
- Učesnici moraju slati **dnevnu attestation poruku** koja potvrđuje uspešan završetak dnevnog procesa arhiviranja

**Omogućavanje funkcije:**

```bash
crsetup.sh --shenable
```

Komanda zaustavlja CR kontejnerske servise, omogućava Sheltered Harbor servise, pa ponovo pokreće CR kontejnerske servise. Traži lockbox passphrase.

> Svaka finansijska institucija koju planirate da dodate zahteva **zasebnu Sheltered Harbor licencu**. Licencni fajl se traži od Sheltered Harbor-a.

## 2.8 Korisničke uloge

| Uloga | Ključna ovlašćenja |
|---|---|
| **Security Admin** | Korisnički nalozi (kreiranje, izmena, omogućavanje, onemogućavanje, brisanje), reset lozinki, password politika, MFA za druge korisnike, mail server podešavanja, telemetrija, **obezbeđivanje i oslobađanje vault-a**, log bundle-ovi |
| **Admin** | Asset-i (kreiranje, upravljanje, izvoz), politike i rasporedi (kreiranje, upravljanje, pokretanje, izvoz), izveštaji, alerti, **obezbeđivanje vault-a**, oporavci i sandbox-ovi, job-ovi (pregled, izvoz, **otkazivanje**), maintenance rasporedi, **DR backup-i**, log podešavanja |
| **Vault Operator** | Asset-i (pregled i izvoz), pokretanje politika, rasporedi (pregled i izvoz), izveštaji, alerti, **obezbeđivanje vault-a**, oporavci i sandbox-ovi, job-ovi (pregled i izvoz), log podešavanja i bundle-ovi |
| **Dashboard** | Samo pregled dashboard-a. **Ova uloga ne ističe (ne time-out-uje).** |

### Bitne asimetrije

| Činjenica | Implikacija |
|---|---|
| **Vault mogu obezbediti Security Admin, Admin i Vault Operator — osloboditi ga može samo Security Admin** | U incidentu bilo koji operater može zatvoriti vault; za otvaranje treba Security Admin |
| **Admin ne može dodavati korisnike; Security Admin ne može dodavati storage ni politike** | Razdvajanje dužnosti je ugrađeno u proizvod |
| **Samo Admin može konfigurisati DR backup** | Vault Operator ga ne vidi |
| **Samo Security Admin vidi telemetriju i alert notifikacije** | |
| **Samo `crso` obavlja prvu prijavu** i mora kreirati bar jednog Admin korisnika | |
| **Security Admin ne može obrisati `crso`** | Ni izmeniti/onemogućiti njegov MFA |
| Ako `crso` zaboravi lozinku | Reset preko `crsetup.sh` — zahteva lockbox passphrase |

> **Ne mešati Cyber Recovery Security Admin sa DD Security Officer-om** za DD Compliance retention locking. To su dva različita naloga na dva različita sistema.

## 2.9 Alati za upravljanje

| Alat | Opis |
|---|---|
| **Cyber Recovery UI** | Primarni alat za upravljanje i praćenje. Web aplikacija na `https://<hostname>:14777`. Omogućava definisanje i pokretanje politika, praćenje operacija, rešavanje problema i verifikaciju ishoda. |
| **CRCLI** | Komandnolinijska alternativa UI-u. `crcli help` prikazuje dostupne komande za vašu ulogu. Puna referenca: *Cyber Recovery Command-Line Interface Reference Guide*. |
| **REST API** | Predefinisan skup operacija preko HTTPS-a, za izradu prilagođene klijentske aplikacije ili integraciju CR funkcionalnosti u postojeću aplikaciju. |

**Pristup REST API dokumentaciji:**

```bash
# 1. Dohvatanje access token-a
curl -k -X POST https://<hostname>:14778/cr/v9/auth/login

# 2. Izvršavanje GET zahteva sa access token-om
```

> Prikazane opcije u UI-u zavise od dodeljene uloge. Dashboard korisnik vidi samo dashboard.

## 2.10 Opcije deploymenta — izbor RHEL vs. OVA

| Kriterijum | Softverska instalacija (RHEL/SLES) | Virtuelni appliance (OVA) |
|---|---|---|
| **Fizički server** | Da | Ne |
| **Standard OS-a kupca** | Poštuje se | Fiksiran na SLES 15 SP6 |
| **Vreme instalacije** | ~10 minuta | ~5 minuta |
| **Kontrola nad OS-om** | Puna | Ograničena |
| **OS patching** | Odgovornost kupca | Preko CR `osupdate` binary-ja |
| **Podman** | Instalira SE ručno | Uključeno |
| **Mandatory access control** | SELinux (preporučeno) | AppArmor |
| **Mrežni interfejsi** | Prema konfiguraciji OS-a | Jedan; dodatni preko vSphere + YaST |
| **4KN datastore** | Nije relevantno | **Nije podržano** |
| **Snapshot kao tačka vraćanja** | Ne | Da |
| **Administrativni napor** | Veći | Manji |

### Kada birati koju opciju

**Birati OVA kada:**
- Vault je VMware okruženje
- Kupac želi minimalan administrativni napor
- Nema zahteva za specifičnim OS standardom
- Datastore nije formatiran sa 4K blokom

**Birati softversku instalaciju kada:**
- CR ide na fizički server
- Kupac ima obavezujući OS standard (npr. RHEL sa korporativnim hardening-om)
- Datastore je VMFS-6 sa 4K blokom
- Potrebna je puna kontrola nad OS konfiguracijom i patch ciklusom

## 2.11 Aplikacije u vault-u

| Aplikacija | Uloga u vault-u |
|---|---|
| **NetWorker, Avamar, PowerProtect Data Manager** | Omogućavaju oporavak i restore podataka radi rehidracije produkcionih backup aplikacija |
| **Cyber Detect** | Analizira backup podatke na prisustvo malvera ili drugih anomalija |

> **Vault ne zahteva backup aplikacije da bi zaštitio podatke** — MTree replikacija kopira sve podatke u vault. Aplikacije se pokreću u vault-u da bi se podaci mogli oporaviti i vratiti.

> **Cyber Detect je podržan isključivo kao komponenta Cyber Recovery rešenja u vault-u; nije podržan na produkcionom sistemu.**

> **Uključiti napajanje** svih vault storage, application i vCenter asset-a **pre** nego što ih dodate u CR deployment.

---

# 3. Planiranje implementacije

## 3.1 Pre-engagement checklist

Podaci koje SE mora prikupiti **pre** dolaska na lokaciju.

### Poslovni i projektni kontekst

| # | Pitanje | Odgovor |
|---|---|---|
| 1 | Šta se štiti — koji sistemi i koji obim podataka | |
| 2 | RPO zahtev | |
| 3 | RTO zahtev | |
| 4 | Retencioni zahtev (koliko dugo se kopije čuvaju) | |
| 5 | Regulatorni zahtevi (compliance režim, Sheltered Harbor) | |
| 6 | Ko su operateri rešenja kod kupca | |
| 7 | Postoji li već vault infrastruktura ili se gradi od nule | |
| 8 | Planirani termin i trajanje maintenance prozora | |

> **Napomena o retenciji:** ako je zahtev duži od **180 dana**, a DD radi pod DDOS 8.8+, Secure Snapshot ne može ispuniti zahtev (maksimum 180 dana). To treba rešiti u fazi dizajna, ne na terenu.

### Tehnički kontekst

| # | Pitanje | Odgovor |
|---|---|---|
| 9 | Backup aplikacije u produkciji i njihove verzije | |
| 10 | NetWorker platforma (Linux / Windows / oba) | |
| 11 | PPDM verzija i način deploymenta | |
| 12 | Modeli i DDOS verzije produkcionih DD sistema | |
| 13 | Modeli i DDOS verzije vault DD sistema | |
| 14 | HA DD u vault-u (da/ne) | |
| 15 | Broj DD sistema u vault-u (1–10) | |
| 16 | Cyber Detect u obimu (da/ne) | |
| 17 | Sheltered Harbor u obimu (da/ne) | |
| 18 | Način deploymenta CR-a (RHEL / SLES / OVA) | |
| 19 | vCenter verzija u vault-u | |
| 20 | Format sektora datastore-a (4K provera) | |
| 21 | Kaskadna replikacija (vault → cleanroom) | |
| 22 | FIPS režim na DD sistemima | |
| 23 | SIEM integracija za audit logove | |
| 24 | Mail server i TLS podrška | |
| 25 | NTP izvor u vault-u | |
| 26 | DNS u vault-u | |

## 3.2 Sizing

### MTree-ovi po politici

| Tip politike | Minimalan broj MTree-ova | Namena |
|---|---|---|
| **Standard** (repository-based) | 3 | replication destination + CR repository + recovery |
| **PPDM** | 2+ | data context(-i) + ServerDR context |

> Stvarni minimum zavisi od zadataka koje treba obaviti.

### Kapacitet

| Pravilo | Detalj |
|---|---|
| **Vault DD mora imati više prostora od produkcionog DD** | Obavezno |
| Prag upozorenja | 80% — alert + email |
| Kritični prag | 90% — alert + email |
| Nedostatak prostora | **Sync job ne uspeva** |

**Radni list kapaciteta:**

| Stavka | Vrednost |
|---|---|
| Ukupan kapacitet produkcionog DD | |
| Iskorišćeno na produkcionom DD | |
| Ukupan kapacitet vault DD | |
| Broj planiranih politika | |
| Broj MTree-ova po politici | |
| Ukupan broj MTree-ova | |
| Planirana retencija | |
| Procenjena godišnja stopa rasta | |

### Ograničenja koja treba imati u vidu

| Ograničenje | Vrednost |
|---|---|
| Maksimalan broj politika | 50 |
| Maksimalan broj DD sistema u vault-u | 10 |
| Maksimalan broj vault DD sistema po produkcionom DD (replikacija) | 10 |
| Min. retencija (RL i Secure Snapshot) | 12 sati |
| Max. retencija — Retention Lock | 5 godina |
| Max. retencija — Secure Snapshot | 180 dana |

## 3.3 Planiranje mrežne segmentacije i air gap-a

### Principi

1. **Vault zahteva namenske resurse, uključujući mrežu.**
2. **Replikacioni interfejs na vault DD-u je namenski** i njime upravlja Cyber Recovery softver. Ne koristiti ga ni za šta drugo.
3. **Management interfejs je uvek omogućen** — CR operacije rade i dok je replikacioni put zatvoren.
4. Vault se otvara ka produkciji **samo tokom replikacije**.

### Odluke koje treba doneti

| # | Odluka | Napomena |
|---|---|---|
| 1 | Da li je vault na istoj ili drugoj fizičkoj lokaciji | |
| 2 | Koji VLAN / subnet za management saobraćaj u vault-u | |
| 3 | Koji VLAN / subnet za replikacioni saobraćaj | |
| 4 | Firewall pravila produkcija ↔ vault | Port 2051 (replikacija), 3009 (samo pri dodavanju parova) |
| 5 | Firewall pravila unutar vault-a | Videti Deo II, 6.1 |
| 6 | Da li NetWorker u vault-u koristi **istu IP adresu** kao produkcioni | Ako da — vault mreža **nikada** ne sme biti istovremeno rutabilna ka produkciji |
| 7 | Kako se unose update fajlovi u vault | Videti Deo VI, 34.4 |
| 8 | Pristup SIEM sistemu iz vault-a | |
| 9 | Pristup mail serveru iz vault-a | Port 25; opciono zaseban adapter na OVA |

> **Kolizija Podman subnetova.** Prikupiti spisak **svih** subnetova u vault-u i uporediti ih sa opsezima koje Podman planira da koristi. Ako postoji preklapanje, CR ne može konfigurisati asset-e dodeljene tom subnetu. Subnetovi se mogu zadati već pri instalaciji.

## 3.4 Planiranje imenovanja

Dosledna konvencija imenovanja štedi vreme pri troubleshooting-u i predaji.

### Šta treba imenovati

| Objekat | Predlog konvencije | Primer |
|---|---|---|
| CR management host | `<org>-cr-mgmt-<n>` | |
| Vault DD sistem | `<org>-dd-vault-<n>` | |
| CR nalog na DD-u | `cradmin` | Preporuka iz dokumentacije |
| DD Security Officer | `<org>-so` | |
| MTree — data | `<aplikacija>_<namena>` | |
| MTree — CR DR backup | `cr-drbackup` | |
| Replication context | `<izvorni_MTree>` → `<odredišni_MTree>` | |
| CR politika | `pol-<aplikacija>-<namena>` | |
| Aplikativni asset | `<aplikacija>-vault` | |
| Tag na aplikaciji | DD Boost username produkcione aplikacije | **Obavezno za NetWorker i PPDM** |

### Imena koja CR generiše automatski

| Objekat | Konvencija |
|---|---|
| Repository MTree | `/data/col1/cr-policy-<policyID>-repo` |
| Recovery job (NetWorker) | `recoverapp_<ID>` |
| Recovery job (PPDM) | `recoverappPPDM` |
| Linked recovery job (PPDM) | `linked-recoverapp` |
| PPDM server DR storage unit | `SysDR_<hostname>` |
| PPDM server DR replika | `SysDR-R_<hostname>` |

### Ograničenja imenovanja

| Ograničenje | Detalj |
|---|---|
| Ime PPDM VM-a u vCenter-u (PPDM < 20.3) | **Bez** znakova `% & * $ # @ ! \ / : * ? " < > [ ] \| ; '` |
| Tag u CR UI-u | Duži od 24 znaka prikazuje se skraćen na 21 znak sa tri tačke |
| PPDM DD Boost ime naloga | Najviše 32 znaka |
| Instalaciona putanja CR-a | **Bez razmaka** |

## 3.5 Planiranje naloga, UID-ova i lozinki

### UID konvencija — planirati unapred

| UID opseg | Namena |
|---|---|
| **501–599** | **Rezervisati** za korisnike koje recovery kreira automatski ili koji moraju odgovarati UID-ovima produkcionih DD Boost korisnika |
| **600–700** | Ručno kreirani DD nalozi specifični za CR: `cradmin`, DD Security Officer, Cyber Detect nalozi, DD admin nalozi za auditing |

> Ako se ova konvencija primeni od početka, izbegava se kolizija u kojoj administrativni nalog zauzme UID koji recovery procedura mora da reprodukuje. **DD dodeljuje UID-ove sekvencijalno počev od 500.**

### DD Boost uloge po aplikaciji

| Aplikacija | `role` |
|---|---|
| NetWorker | `admin` |
| Avamar | `admin` |
| **PowerProtect Data Manager** | **`none`** |

### Lozinke i passphrase-i

| Stavka | Zahtev |
|---|---|
| Dužina | 9–64 znaka |
| FIPS — lockbox passphrase | **Najmanje 14 znakova** |
| Sadržaj | Bar jedan broj, veliko slovo, malo slovo i specijalan znak |
| Razmaci | Nisu dozvoljeni |
| Postgres lozinka | **Ne koristiti dvotačku (`:`)** u ovom izdanju |

> ### CAUTION — lockbox passphrase
> **Ne može se povratiti.** Ako se izgubi, potrebna je sveža instalacija i gubi se cela konfiguracija. Neophodan je za nadogradnje, reset Security Admin lozinke i oporavak CR servera.
>
> **Planirati gde će se čuvati pre nego što se instalacija započne** — u sistemu za upravljanje tajnama kupca ili fizičkom sefu, redundantno, van CR servera.

## 3.6 Redosled radova

Preporučeni redosled aktivnosti, sa naznakom šta se može raditi paralelno.

| Faza | Aktivnost | Zavisnost | Deo |
|---|---|---|---|
| **1** | Prikupljanje preduslova i popunjavanje radnih listova | — | I, II |
| **2** | Priprema mreže i firewall pravila | 1 | II |
| **3** | Osnovna konfiguracija vault DD sistema | 2 | II, 9 |
| **4** | Kreiranje MTree-ova i DD Boost naloga | 3 | II, 9 |
| **5** | Konfiguracija Retention Lock / Secure Snapshot | 3 | II, 9 |
| **6** | Kreiranje replication konteksta | 4 | II, 9.8 |
| **7** | **Inicijalna replikacija** | 6 | II, 9.9 |
| **8** | Priprema produkcijskih backup aplikacija | paralelno sa 3–7 | IV, V |
| **9** | Deployment backup aplikacija u vault-u | paralelno sa 3–7 | IV, V |
| **10** | Instalacija Podman-a i CR softvera | 2 | III |
| **11** | Prva prijava, korisnici, licence, sertifikati | 10 | III, 14 |
| **12** | Dodavanje vault storage-a | 11, 7 | III, 15.1 |
| **13** | Dodavanje vCenter-a i aplikacija | 12, 9 | III, 15.2–15.3 |
| **14** | Kreiranje politika | 13 | III, 15.4 |
| **15** | Prvo izvršavanje Sync / Copy / Lock | 14 | III, 15.6 |
| **16** | Usklađivanje UID-ova | 15 | IV, 20 |
| **17** | Test recovery | 16 | IV, V |
| **18** | Konfiguracija DR backup-a CR konfiguracije | 12 | VI, 33.4 |
| **19** | Maintenance raspored, notifikacije, telemetrija | 11 | VI, 33 |
| **20** | Acceptance testiranje i primopredaja | 17, 18, 19 | VI, 32 |

### Kritični putevi

> **Inicijalna replikacija (korak 7) je preduslov za definisanje politika.** Ako je količina podataka velika, ona može trajati danima. **Planirati je što ranije** — idealno pre nego što SE dođe na lokaciju za konfiguraciju.

> **Usklađivanje UID-ova (korak 16) zavisi od postojanja kopije**, jer se UID čita komandom `crcli policy list-copy`. Zato se ne može uraditi pre prvog uspešnog Copy-ja.

> **Redosled 8 i 9 u odnosu na 14:** rasporedi u CR politici se ne mogu ispravno postaviti dok se ne zna kada završavaju produkcioni backup-i. Prikupiti te podatke u fazi 8.

## 3.7 Rizici i tipične greške

| Rizik | Posledica | Kako izbeći |
|---|---|---|
| Datastore sa 4K blokom | OVA se ne može deploy-ovati | Proveriti u fazi 1 |
| Retencioni zahtev > 180 dana uz DDOS 8.8+ | Secure Snapshot ne može ispuniti zahtev | Rešiti u dizajnu |
| Zauzet UID/GID 14999 | CR direktorijume poseduje pogrešan korisnik | Proveriti pre instalacije |
| Kolizija Podman subnetova | CR ne može konfigurisati asset-e | Prikupiti subnetove u fazi 1 |
| Novija verzija Podman-a od podržane | Instalacija se prekida | Pinovati verzije |
| Docker uklonjen pre nadogradnje sa < 20.2 | Nadogradnja ne uspeva | Poštovati redosled |
| Docker nije uklonjen pri svežoj 20.3 na bare-metal | Instalacija ne uspeva | Poštovati redosled |
| Prva prijava preko CLI-ja | `Failed to authenticate user` | Prva prijava uvek kroz UI |
| Izgubljen lockbox passphrase | Gubitak cele konfiguracije | Planirati čuvanje unapred |
| Neusklađeni UID-ovi | Recovery ne uspeva | Zabeležiti produkcione UID-ove u fazi 8 |
| Sync zakazan prerano | Kopija bez kompletnog backup-a | Ostaviti rezervu + 15 min za DD replikacioni ciklus |
| Port 3009 ostavljen otvoren | Narušen air gap | Zatvoriti odmah po dodavanju para |
| NFS v4 bez v3 uz PPDM | PPDM recovery ne uspeva | Omogućiti v3 ili koristiti DD Boost |
| ifGroups nisu konfigurisani (PPDM) | Server DR backup ne uspeva | Proveriti na produkciji u fazi 8 |
| NetWorker sa više MTree-ova | Automatski recovery nije podržan | Utvrditi u fazi 1 |

---

# Dodatak: Radni list za planiranje

## A. Projekat

| Stavka | Vrednost |
|---|---|
| Kupac | |
| Lokacija vault-a | |
| System Engineer | |
| Kontakt kod kupca | |
| Broj projekta / SR | |
| Planirani datum početka | |
| Planirani datum primopredaje | |

## B. Zahtevi

| Stavka | Vrednost |
|---|---|
| RPO | |
| RTO | |
| Retencija kopija | |
| Regulatorni zahtevi | |
| Compliance ili Governance režim | |
| Sheltered Harbor | |

## C. Arhitekturne odluke

| # | Odluka | Izbor | Obrazloženje |
|---|---|---|---|
| 1 | Način deploymenta CR-a (RHEL / SLES / OVA) | | |
| 2 | Broj DD sistema u vault-u | | |
| 3 | HA DD | | |
| 4 | Immutability tehnologija | | |
| 5 | Retention Lock režim | | |
| 6 | NetWorker u obimu | | |
| 7 | PPDM u obimu | | |
| 8 | Avamar u obimu | | |
| 9 | Cyber Detect u obimu | | |
| 10 | Ista IP adresa za NetWorker u vault-u | | |
| 11 | NFS v3 / v4 / oba na vault DD | | |
| 12 | Kaskadna replikacija | | |
| 13 | SIEM integracija | | |
| 14 | FIPS režim | | |

## D. Imenovanje

| Objekat | Dogovoreno ime |
|---|---|
| CR management host | |
| Vault DD sistem(i) | |
| CR nalog na DD-u | |
| DD Security Officer | |
| Konvencija za MTree-ove | |
| Konvencija za politike | |
| MTree za CR DR backup | |

## E. Mreža

| Stavka | Vrednost |
|---|---|
| Management subnet u vault-u | |
| Replikacioni subnet | |
| IP adresa CR management hosta | |
| FQDN CR management hosta | |
| DNS serveri | |
| NTP server | |
| Gateway | |
| Spisak svih subnetova u vault-u (za Podman proveru) | |
| Planirani Podman opsezi | |
| Mail server i port | |
| SIEM server | |

## F. Vremenski plan

| Faza | Planirani datum | Odgovoran | Status |
|---|---|---|---|
| Prikupljanje preduslova | | | |
| Priprema mreže | | | |
| Priprema DD sistema | | | |
| **Inicijalna replikacija** | | | |
| Priprema produkcijskih aplikacija | | | |
| Deployment aplikacija u vault-u | | | |
| Instalacija CR softvera | | | |
| Konfiguracija politika | | | |
| Usklađivanje UID-ova | | | |
| Test recovery | | | |
| Acceptance testiranje | | | |
| Primopredaja | | | |

---

*Kraj Dela I. Sledeći deo: **Deo II — Preduslovi** (`02-Preduslovi.md`).*
