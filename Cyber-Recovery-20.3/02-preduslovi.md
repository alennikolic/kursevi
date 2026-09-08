# DEO II — PREDUSLOVI

**Dell PowerProtect Cyber Recovery 20.3 — Uputstvo za instalaciju i integraciju**
Verzija dokumenta: 0.1 | Namena: System Engineer

> **Izvori:** *Dell PowerProtect Cyber Recovery 20.3 Installation and Upgrade Guide*, *Dell PowerProtect Cyber Recovery 20.3 Product Guide*.
> **Napomena:** Za konačnu potvrdu verzija uvek proveriti **E-Lab Navigator** i **KB članak 000205512** (minimalne, preporučene i najnovije verzije koda). Vrednosti u ovom dokumentu odgovaraju stanju u dokumentaciji za verziju 20.3.

---

## Sadržaj

- [4. Zahtevi za Cyber Recovery management host](#4-zahtevi-za-cyber-recovery-management-host)
- [5. Zahtevi za PowerProtect DD sisteme](#5-zahtevi-za-powerprotect-dd-sisteme)
- [6. Mrežni preduslovi](#6-mrežni-preduslovi)
- [7. Podržane backup aplikacije u vault-u](#7-podržane-backup-aplikacije-u-vault-u)
- [8. Kontejnerski runtime — Podman](#8-kontejnerski-runtime--podman)
- [9. Preduslovna priprema vault DD sistema](#9-preduslovna-priprema-vault-dd-sistema)
- [Dodatak: Radni list preduslova](#dodatak-radni-list-preduslova)

---

# 4. Zahtevi za Cyber Recovery management host

Management host je fizički server ili virtuelna mašina na kojoj se izvršava Cyber Recovery softver. Postoje dva načina isporuke:

| Način isporuke | Opis | Kada se bira |
|---|---|---|
| **Softverska instalacija** | Instalacija CR paketa na kupčev RHEL ili SLES sistem | Fizički server, postojeći OS standard kupca, potreba za punom kontrolom nad OS-om |
| **Virtuelni appliance (OVA)** | Predkonfigurisana VM sa SLES 15 SP6, deploy na VMware | VMware okruženje u vault-u, brži deployment (~5 minuta), manje OS administracije |

---

## 4.1 Podržani operativni sistemi

Za **softversku instalaciju** management hosta podržani su sledeći operativni sistemi, sa najnovijim ispravkama, zakrpama i bezbednosnim zakrpama:

| Operativni sistem | Verzije |
|---|---|
| Red Hat Enterprise Linux | 8.10, 9.4, 9.6 |
| SUSE Linux Enterprise Server | 16, 15 SP7, 15 SP6, 15 SP5, 12 SP5 (EOL) |
| SUSE Linux Enterprise Server (za virtuelni appliance) | 15 SP6 |

> **UPOZORENJE — CentOS nije podržan.**
> CentOS nije podržan ni za nove instalacije ni za nadogradnju postojećih Cyber Recovery deploymenta. Ako pokušate instalaciju ili nadogradnju na CentOS sistemu, instalacija se prekida sa greškom. Postojeći CentOS deployment treba migrirati na RHEL ili SLES (videti Deo VI, poglavlje 34).

---

## 4.2 Hardverski zahtevi — softverska instalacija (RHEL / SLES)

| Resurs | Minimalni zahtev |
|---|---|
| RAM | 4 GB |
| Slobodan prostor za raspakivanje instalacionog paketa | 5 GB |
| Slobodan prostor za instalaciju CR softvera | 20 GB ili više |
| Slobodan prostor za nadogradnju (update) | najmanje 20 GB |

> **Preporuka SE-u:** 4 GB RAM je apsolutni minimum iz dokumentacije. Za produkcione implementacije sa više politika, DD sistema i aktivnim Cyber Detect-om planirajte više memorije i prostora — konsultovati *Dell PowerProtect Cyber Recovery Solution Guide* za sizing.

---

## 4.3 Zahtevi za Cyber Recovery virtuelni appliance (OVA)

Virtuelni appliance je predkonfigurisana virtuelna mašina koja radi pod SUSE Linux Enterprise Server 15 SP6 i spremna je za deployment na VMware hipervizor.

| Resurs | Zahtev |
|---|---|
| Hipervizor | VMware vCenter ili ESXi, verzija 8.0.x i 9.0 |
| Prostor za deployment OVA fajla | ~10 GB |
| Ukupan prostor za diskove | ~195 GB |
| Disk 1 | 48 GB |
| Disk 2 | 48 GB |
| Disk 3 | 96 GB |
| CPU | 2 vCPU, jedan core po socket-u |
| Memorija | 8 GB |

**Preporuka:** thin provisioning za sve diskove.

### Ograničenja OVA deploymenta

- **Mrežni interfejsi:** virtuelni appliance je podrazumevano konfigurisan sa **jednim** interfejsom. Dodatni virtuelni Ethernet adapteri mogu se dodati nakon deploymenta, ali su testirani samo za SMTP komunikaciju.
- **4KN diskovi nisu podržani:** Cyber Recovery i Cyber Detect OVA fajlovi ne podržavaju 4KN emulaciju diska i koriste format sektora od 512 bajtova. Nisu kompatibilni sa okruženjima koja zahtevaju 4KN podršku, kao što su VMFS-6 datastore-ovi formatirani sa blokom od 4K.

> **Provera pre deploymenta:** utvrditi format sektora datastore-a u vault vCenter-u. Ako je datastore formatiran za 4K blok, OVA se ne sme koristiti — potrebna je softverska instalacija na RHEL/SLES.

---

## 4.4 Zahtevani paketi i biblioteke

### libstdc++

Za **softversku instalaciju ili nadogradnju** Cyber Recovery-ja, biblioteka `libstdc++` mora biti instalirana. Kod deploymenta virtuelnog appliance-a `libstdc++` je već uključena.

### rsyslog paketi (za prosleđivanje audit logova)

Ako se koristi funkcionalnost prosleđivanja audit logova preko rsyslog-a, potrebni su sledeći paketi:

**Red Hat Enterprise Linux:**

| Paket | Namena |
|---|---|
| `rsyslog` | Sistem za obradu logova — prikupljanje, obrada i prosleđivanje log poruka |
| `rsyslog-gnutls` | Modul za rsyslog koji omogućava bezbedan prenos logova preko GnuTLS biblioteke |

**SUSE Linux Enterprise Server:**

| Paket | Namena |
|---|---|
| `rsyslog` | Sistem za obradu logova |
| `rsyslog-module-gtls` | Modul za bezbedan prenos logova preko GnuTLS-a |

**Napomene:**
- Kod virtuelnog appliance-a ove pakete obezbeđuje OS update Cyber Recovery appliance-a.
- Ako paketi nisu instalirani, precheck korak prikazuje **upozorenje**. Instalacija ili nadogradnja se može uspešno završiti, ali funkcija auditinga neće biti upotrebljiva.
- Kod softverske instalacije konfiguracioni fajl `forwardAuditLogs.conf` mora se dodati **ručno** (videti KB članak na Dell Online Support).

**Provera prisustva paketa:**

```bash
# Provera konfiguracije i preduslova za instalaciju
./crsetup.sh --check

# Provera spremnosti za nadogradnju
./crsetup.sh --upgcheck
```

---

## 4.5 Zahtevi za pristup korisničkom interfejsu

| Stavka | Zahtev |
|---|---|
| URL | `https://<host>:14777` |
| Podržani pretraživači | Google Chrome, Microsoft Edge, Mozilla Firefox |

> Za najažurniji spisak podržanih verzija pretraživača videti **E-Lab Navigator**.

---

## 4.6 Ostali OS preduslovi

| Stavka | Zahtev | Napomena |
|---|---|---|
| `umask` | 022 | Dodati `umask 022` u `~/.bashrc` ili `~/.bash_profile`, zatim `source ~/.bashrc` |
| SELinux (RHEL) | Preporučeno kao Linux Security Module | Zbog granularne kontrole pristupa, pažljivo konfigurisati bezbednosne kontekste za CR fajlove i direktorijume |
| NTP | Obavezno u praksi | Interna NTP konfiguracija je preporučena; MFA ne radi ako se vreme hosta razlikuje više od ±60 sekundi od authenticator-a |
| DNS | Rezolucija mora raditi | Neispravan DNS je čest uzrok neuspele prijave u UI |
| Docker | Mora biti uklonjen | Za bare-metal, pre sveže instalacije 20.3/20.2 ili nadogradnje sa verzija starijih od 20.2 — videti odeljak 8.5 |

---

# 5. Zahtevi za PowerProtect DD sisteme

## 5.1 Produkcioni DD sistemi

Produkciono okruženje mora imati najmanje jedan PowerProtect DD sistem konfigurisan za replikaciju ka DD sistemu u Cyber Recovery vault-u.

| Sistem za skladištenje | Napomene |
|---|---|
| **PowerProtect DD sa DDOS 7.13 ili 8.x** | Cyber Recovery ne podržava DD sa Cloud DR i Cloud Tier u vault-u. **Vault DD sistem mora imati više prostora od produkcionog DD sistema.** |
| **Dell DP4400 IDPA** | Može biti replikacioni target. Osim DDOS-a i Avamar Virtual Edition (AVE), Cyber Recovery ne podržava druge funkcije IDPA uređaja u vault-u. Za DP4400 verzije 2.7.0 i novije, CR rešenje može biti integrisano u uređaj; za starije verzije CR je eksterno. Ako produkcioni DP4400 replicira ka AVE i DD sistemu u vault-u, Avamar geometrija produkcionog i target Avamar servera mora biti identična. |
| **Dell DP5300 i DP5800 IDPA** | Mogu biti replikacioni target. Za verzije 2.7.0+ CR se može integrisati u uređaj; za starije verzije CR je eksterno. |
| **Dell DP8300 i DP8800 IDPA** | **Nisu podržani** ni u produkcionom ni u vault okruženju, zbog ograničenja podrške za Avamar Grid. Replikacija preko single node ili Virtual Edition instance jeste podržana — za detalje kontaktirati Dell predstavnika. |
| **Dell Disk Library for mainframe (DLm)** | Podržano — za detalje kontaktirati Dell servisnog predstavnika. |

> **Skalabilnost:** kada je u produkciji raspoređeno više DD sistema, mogu se konfigurisati za replikaciju ka **najviše 10** DD sistema u Cyber Recovery vault-u.

---

## 5.2 Vault DD sistemi — sistemski zahtevi

Vault storage okruženje uključuje **minimum jedan, maksimum 10** fizičkih ili virtuelnih DD sistema, na istoj mreži kao i Cyber Recovery softver.

### Zahtevi po svakom DD sistemu u vault-u

| Zahtev | Detalj |
|---|---|
| **DDOS verzija** | 7.13 ili 8.x i novije |
| **Ethernet interfejsi** | Dva: primarni za DD management, drugi **namenski** za replikaciju, kojim upravlja Cyber Recovery softver |
| **Nalog za CR** | DD nalog sa ulogom `admin`. Preporučeno ime: `cradmin` (ime je proizvoljno). **Nalog `sysadmin` se ne sme koristiti.** |
| **Licence** | Važeće licence za **DD Boost**, **Replication**, **Retention Lock Governance** i **Retention Lock Compliance** |

> **Važno:** ne konfigurisati više Cyber Recovery servera da koriste isti DD sistem.

---

## 5.3 Immutability — Retention Lock i Secure Snapshot

Immutability (nepromenljivost podataka) je **obavezna** za vault okruženje. PowerProtect DD je obezbeđuje na dva načina, u zavisnosti od verzije DDOS-a:

| DDOS verzija | Tehnologija | Kako radi |
|---|---|---|
| **8.7 i starije** | PowerProtect DD Retention Lock | Aktivira se **po MTree-u**; vreme retencije se postavlja **po fajlu** |
| **8.8 i novije** | PowerProtect DD Secure Snapshot | Aktivira se **na nivou sistema**; kreira nepromenljive snapshot-ove na replikacionom target-u nakon prenosa podataka u vault |

**Ponašanje pri migraciji:** Secure Snapshot se koristi za nove politike i za postojeće politike koje ispunjavaju uslove za migraciju. Ako politika ne može da migrira na Secure Snapshot, nastavlja da koristi postojeću DD Retention Lock konfiguraciju.

### Režimi Retention Lock-a

| Režim | Opis |
|---|---|
| **Governance** | Fleksibilna retencija zasnovana na politici. Ovlašćeni korisnici mogu menjati podešavanja retencije. Namenjena relativno kratkim trajanjima. |
| **Compliance** | Stroži režim. Zaključani fajlovi se ne mogu obrisati ni prepisati **ni pod kojim okolnostima** dok period retencije ne istekne. Preporučen za zaštitu od ozbiljnijih pretnji. |
| **None** | Bez retention lock-a. Podaci se mogu obrisati ili prepisati u bilo kom trenutku. **Zaobilazi dodatni sloj zaštite — pažljivo razmotriti rizike.** |

### Ograničenja Retention Lock-a

- **Indefinite Retention Hold nije podržan** ni u Governance ni u Compliance režimu.
- **Retention Lock Compliance nije podržan** na sledećim platformama:
  - Dell PowerProtect DD3300 uređaji i DD Virtual Edition (DDVE) sa DDOS verzijom starijom od 7.10
  - Dell DP4400 IDPA — podržan je **samo Retention Lock Governance** režim

### Trajanja retencije

| Parametar | Retention Lock (Governance/Compliance) | Secure Snapshot |
|---|---|---|
| Minimalni period retencije | ne manje od **12 sati** | ne manje od **12 sati** |
| Maksimalni period retencije | najviše **5 godina** | najviše **180 dana** |

### Uslovi za Secure Snapshot podršku

U CR UI, polje **Secure Snapshot Capable** za svaki konfigurisani DD sistem prikazuje:

| Vrednost | Značenje |
|---|---|
| `Yes` | DDOS 8.8+, Retention Lock Compliance omogućen, MTree Scaling omogućen |
| `No — Update DDOS` | DDOS stariji od 8.8 |
| `No — RLC not enabled` | DDOS 8.8+, ali Retention Lock Compliance nije omogućen za hardening |
| `No — Filesystem restart required` | DDOS 8.8+ i RLC omogućen, ali je potreban restart fajl sistema da bi se omogućio MTree Scaling |

> **Napomena:** Retention Lock Compliance je takođe potreban za hardening DD sistema radi upotrebe Secure Snapshot-ova. Secure Snapshot-ovi ne koriste Retention Lock Compliance za nametanje retencije kopija — nepromenljivi su u trenutku kreiranja.

---

## 5.4 Zahtevi za kapacitet i MTree-ove

- Za **svaku Cyber Recovery politiku** u standardnom deploymentu potreban je kapacitet za najmanje **tri MTree-a**:
  1. Replication destination
  2. Cyber Recovery repository
  3. Recovery
- Stvarni minimum zavisi od zadataka koje treba obaviti.
- **PPDM politika zahteva najmanje dva MTree-a** za konfiguraciju (data + ServerDR).

### Pragovi popunjenosti

| Prag | Podrazumevana vrednost | Posledica |
|---|---|---|
| Warning | 80% | Alert na CR dashboard-u + email notifikacija (ako je email konfigurisan) |
| Critical | 90% | Alert na CR dashboard-u + email notifikacija |
| Nedostatak prostora | — | **Sync job ne uspeva** |

---

## 5.5 High Availability (HA) deployment

Ako se u vault-u koristi HA DD sistem:

- Potrebne su **floating IP adrese** na svakom DD sistemu:
  - jedna za Cyber Recovery management host
  - jedna za replikaciju
  - za sve multilink interfejse
- Aktivni i standby DD sistem moraju imati **identičnu konfiguraciju interfejsa**.

> **Ograničenje:** Direct Connect nije podržan za HA PowerProtect DD sistem u Cyber Recovery vault-u.

---

## 5.6 NFS podešavanja na DD sistemu

Ovo je čest uzrok neuspeha politika, sandbox-ova i recovery operacija — proveriti pre instalacije.

### Obavezno podešavanje: root squash

Opcija `force-minimum-root-squash-default` mora biti postavljena na `disabled`.

```bash
sysadmin@dd-vault# nfs option show all
```

Očekivani izlaz (relevantni redovi):

```
Option                              Value
---------------------------------   --------------
default-export-version              3:4
default-server-version              3:4
nfs4-idmap-out-numeric              always
default-root-squash                 enabled
force-minimum-root-squash-default   disabled
---------------------------------   --------------
```

> Za detalje videti **KB članak 198370**.

### NFS verzije — matrica podešavanja

Iz DD UI: **Protocols > NFS > Options**

| Željeni režim | Default Export Version | Default Servers Version | NFSv4 ID Map Out Numeric |
|---|---|---|---|
| Samo NFSv3 | NFSv3 | NFSv3 | — |
| Samo NFSv4 | NFSv4 | NFSv4 | `always` |
| NFSv3 i NFSv4 | NFSv3 i NFSv4 | NFSv3 i NFSv4 | `always` |

> **KRITIČNO ZA PPDM:** ako je na vault DD sistemu omogućen **samo NFS v4**, PowerProtect Data Manager recovery **ne uspeva** — PPDM server DR podržava isključivo NFS v3. Rešenje: ili koristiti DD Boost za Server DR MTree, ili omogućiti i NFS v3 i NFS v4.

---

## 5.7 FIPS režim i autentikacija replikacije

Ako je FIPS režim omogućen na **bilo kom** od dva DD sistema (produkcionom ili vault):

- Svaki replication context između ta dva DD sistema mora biti konfigurisan za **dvosmernu (two-way) autentikaciju**.
- FIPS ne mora biti omogućen na oba DD sistema da bi ovaj zahtev važio.
- Režim autentikacije se konfiguriše **zasebno za svaki** replication context. Replikacija ne uspeva ako bilo koji kontekst koristi drugačiji režim.
- Konfigurisati dvosmernu autentikaciju za sve relevantne kontekste **pre** sledeće Sync operacije.
- Zahtev se odnosi **samo** na replication kontekste između produkcionog i vault DD sistema. Cyber Detect Analyze operacije i PPDM recovery operacije nisu pogođene.

---

# 6. Mrežni preduslovi

## 6.1 Tabela mrežnih portova

Sledeći portovi na management hostu moraju biti rezervisani za Cyber Recovery softver.

| Port | Protokol | Namena | Opis | Smer | Obavezan |
|---|---|---|---|---|---|
| **22** | TCP | SSH | Komunikacioni kanal između CR SSH klijenta i udaljenih sistema u vault-u | Inbound | Da |
| **25** | TCP | Notifikacije | SMTP email notifikacije o alertima i događajima | Outbound | Opciono |
| **111** | TCP | NFS Client | NFS mount između DD sistema i CR management hosta | Bi-directional | Da |
| **123** | UDP | NTP | Sinhronizacija vremena sa referentnim izvorom | Bi-directional | Ne |
| **443** | TCP | OIDC | Komunikacija između Cyber Recovery Manager-a i Cyber Detect-a kada je OIDC omogućen | Bi-directional | Da |
| **2049** | TCP | NFS Client | NFS mount između DD sistema i CR management hosta | Bi-directional | Da |
| **2051** | TCP | Replication | Podrazumevani replikacioni port za vault storage konfiguraciju | Bi-directional | Ne |
| **2052** | TCP | NFS Client | Mount ka DD sistemu | Bi-directional | Da |
| **3009** | TCP | Replication / REST | CR može izdavati REST API pozive ka DD sistemima u vault-u | Inbound | Da |
| **14777** | TCP | Nginx | HTTPS pristup Cyber Recovery UI-u iz pretraživača | Inbound | Da |
| **14778** | TCP | REST API | HTTPS konekcija za korisnički i UI REST interfejs | Inbound | Da |
| **14780** | TCP | Swagger | Pristup dokumentaciji Cyber Recovery REST API-ja | Inbound | Opciono |

> Tabela sadrži opšte preporuke zasnovane na standardnim portovima i servisima. Vaše okruženje može koristiti nestandardne portove.

### Posebne napomene po portovima

**Port 2051 — replikacija**
Podrazumevani replikacioni port. **Snažno se preporučuje da se ova vrednost ne menja.** Svaka promena može uticati na DD i DM5500 sisteme i može zahtevati dodatne izmene konfiguracije.

**Port 3009 — replikacija i Cyber Detect**
- Mora biti dostupan između Cyber Recovery-ja i Dell Cyber Detect-a, inače dolazi do problema.
- Mora biti otvoren između **produkcionog i vault DD sistema** prilikom dodavanja novih replikacionih parova.
- **Onemogućiti port 3009 nakon što je replikacioni par dodat.**
- Za detalje videti **KB članak 205800**.

**Portovi 14777, 14778, 14780 — firewall**
Ako je na sistemu aktivan firewall, ova tri porta moraju biti otvorena, inače instalacija ili prijava u UI ne uspevaju.

---

## 6.2 Interne Podman mreže i kolizija adresa

Pri instalaciji Cyber Recovery kreira dve Podman mreže koje izoluju kontejnere deploymenta:

| Mreža | Namena |
|---|---|
| `cr_back` | Backend kontejneri |
| `cr_front` | Frontend kontejneri |

Podman podrazumevano kreira bridge interfejse (`podman0`, `podman1`, `podman2`) i dodeljuje im opsege adresa koji **mogu doći u koliziju sa adresama u vault okruženju**. Ako subnet u vault-u dolazi u konflikt sa Podman subnetom, Cyber Recovery ne može da konfiguriše asset-e dodeljene tom subnetu.

**Provera trenutnih opsega:**

```bash
podman network inspect cr_back  | grep -E 'subnet|gateway'
podman network inspect cr_front | grep -E 'subnet|gateway'
podman network inspect cr_back cr_front podman
```

Primer podrazumevane konfiguracije:

```
cr_back:   subnet 10.89.0.0/24, gateway 10.89.0.1, interface podman1
cr_front:  subnet 10.89.1.0/24, gateway 10.89.1.1, interface podman2
podman:    subnet 10.88.0.0/16, gateway 10.88.0.1, interface podman0
```

> Podman u zavisnosti od verzije i okruženja može dodeliti i opsege iz 172.17.x / 172.18.x serije. Ne pretpostavljati unapred — proveriti komandom.

**Preduslovna radnja SE-a:** pre instalacije prikupiti spisak svih subnetova u vault-u i uporediti ih sa opsezima koje Podman planira da koristi. Ako postoji preklapanje, **subnet-ovi se mogu zadati već pri instalaciji** ili se readresiraju naknadno (`crsetup.sh --forcerecreate --changeiprange`). Procedura je opisana u Delu III.

---

## 6.3 DNS, FQDN i NTP

| Stavka | Zahtev |
|---|---|
| **DNS** | Nije formalno obavezan, ali je **snažno preporučen** za vault okruženje. DNS podešavanja moraju biti rezolvabilna — u suprotnom prijava u CR UI ne uspeva. |
| **FQDN** | Potreban za management host; kod PPDM verzija starijih od 20.3 mora se poklapati sa DNS imenom prikazanim u vCenter UI |
| **NTP / izvor vremena** | Nije formalno obavezan, ali je **snažno preporučen**. Interna NTP konfiguracija je preporučena. |

**Zašto je NTP praktično obavezan:**
- Multifactor authentication je vremenski zasnovan mehanizam. Ako se vreme CR hosta razlikuje od vremena authenticator-a za više od **±60 sekundi**, MFA se ne može omogućiti.
- Ako se vreme CR hosta izmeni, potrebno je zaustaviti pa ponovo pokrenuti CR servise.
- Za Cyber Detect: ako nema NTP servera, drift je oko nekoliko sekundi, ali **sistemski sat Cyber Detect-a mora biti ispred sistemskog sata Cyber Recovery-ja**.

---

## 6.4 Email preduslovi

| Stavka | Preporuka / zahtev |
|---|---|
| TLS | **Preporučen** za svu email komunikaciju iz vault-a ka mail/relay serveru |
| Verzija TLS-a | Mail server treba da podržava **TLS 1.2 ili TLS 1.3** |
| Sertifikat | Ako je TLS omogućen, **mora** se koristiti sertifikat mail servera |
| Podrazumevano stanje | TLS je podrazumevano omogućen kod instalacije verzije 19.21 i novijih, kao i pri nadogradnji sa 19.13+ |

> Ako se TLS onemogući, sva email komunikacija iz Cyber Recovery-ja se šalje **nešifrovano**.

---

## 6.5 Preporuke za izolaciju i air gap

- Vault je kupčeva bezbedna lokacija koja predstavlja odredište DD MTree replikacije. Zahteva **namenske resurse, uključujući mrežu**.
- Pristup vault-u se otvara samo onoliko dugo koliko je potrebno da se podaci repliciraju iz produkcije; u svim ostalim trenucima vault je zatvoren i odvojen od produkcione mreže.
- Vault može biti i na drugoj lokaciji (npr. kod service provider-a).
- Replikacioni interfejs na vault DD sistemu je **namenski** i njime upravlja Cyber Recovery softver — ne koristiti ga za druge namene.
- Opciono, SSH pristup replikacionom interfejsu se može ograničiti (procedura u Delu VI, poglavlje 36).

---

# 7. Podržane backup aplikacije u vault-u

## 7.1 Produkciona strana — podržane aplikacije

| Aplikacija | Podržane verzije | Napomene |
|---|---|---|
| **Avamar** | 19.9 i novije | Single-node fizički uređaj ili Avamar Virtual Edition (AVE). **Avamar grid nije podržan.** Validirani Avamar checkpoint-ovi se čuvaju na DD sistemu. |
| **NetWorker** (Linux i Windows) | 19.10 i novije | NetWorker server baza i data uređaji se čuvaju na DD sistemu. |
| **PowerProtect Data Manager** | 19.19 i novije | DDOS mora biti 7.13 ili 8.x i noviji. PPDM server backup-ovi i policy podaci se čuvaju na DD sistemu. |

---

## 7.2 Vault strana — zahtevi po aplikaciji

| Aplikacija | Zahtevi u vault-u |
|---|---|
| **Avamar** (19.9+) | • Ista verzija kao na produkciji<br>• Single-node ili AVE server (grid nije podržan)<br>• **Neinicijalizovana** i ispravno dimenzionisana instanca, ekvivalentna produkcionoj<br>• Hostname koji se poklapa sa produkcionim<br>• DD sistem ima isto DD Boost ime naloga i isti UID |
| **NetWorker** (19.10+) | • Ista verzija kao na produkciji<br>• **Neinicijalizovana** i ispravno dimenzionisana instanca za izvršavanje `nsrdr` operacije nad podacima repliciranim sa produkcionog na vault DD |
| **PowerProtect Data Manager** (19.19+) | • Omogućava VM recovery ili file system recovery<br>• Za automatski recovery: PPDM mora biti deployovan u vault preko **OVA na vCenter**<br>• Ista verzija kao na produkciji |

> **Napomena:** Vault ne zahteva ove aplikacije da bi zaštitio podatke — MTree replikacija kopira sve podatke u vault. Međutim, pokretanje aplikacija u vault-u omogućava oporavak i restore podataka radi rehidracije produkcionih backup aplikacija.

---

## 7.3 Ograničenja

| Ograničenje | Detalj |
|---|---|
| **Windows aplikacije** | Osim NetWorker-a na Windows-u, Cyber Recovery **ne podržava** backup aplikacije koje rade na Windows-u, niti dodavanje Windows aplikacija u CR okruženje |
| **NetWorker na Windows-u** | Zahteva **Cygwin** sa omogućenim OpenSSH servisom. Mount operacija nad NetWorker sandbox-om nije podržana. |
| **vDisk i VTL formati** | Ograničena podrška — jedina podržana CR operacija je **Sync**. Nijedna druga operacija zaštite na DD sistemu u vault-u nije podržana. |
| **NetWorker sa više MTree-ova** | Automatski recovery NetWorker servera sa više od jednog MTree-a **nije podržan**. Ručni recovery je moguć — kontaktirati Dell Support. |
| **Aplikacije trećih strana** | Cyber Recovery može štititi podatke aplikacija trećih strana, ali su te aplikacije van obima Dell dokumentacije. Koristiti smernice proizvođača aplikacije. |

---

## 7.4 Uloge DD Boost korisnika po aplikaciji

Ovo je čest izvor grešaka pri konfiguraciji — vrednost uloge se razlikuje po aplikaciji.

| Aplikacija | Vrednost `role` pri kreiranju DD Boost korisnika |
|---|---|
| NetWorker | `admin` |
| Avamar | `admin` |
| PowerProtect Data Manager | `none` |

> `role` se odnosi na korisnika koji izvršava backup-ove na **produkcionom** DD sistemu.

---

## 7.5 Uloge pod kojima se aplikacija dodaje u Cyber Recovery

| Aplikacija / platforma | Korisnik pod kojim se dodaje u vault-u |
|---|---|
| NetWorker na Linux-u | `root` — CR koristi komande poput `nsrdr` koje zahtevaju root privilegije |
| NetWorker na Windows-u | `Admin` |
| Avamar | `Admin` |
| PowerProtect Data Manager | `Admin` |
| Cyber Detect | `Admin` — uz važeći sertifikat instaliran na Cyber Detect serveru |

---

# 8. Kontejnerski runtime — Podman

Instalacija Cyber Recovery softvera zahteva kontejnerski runtime. Cyber Recovery 20.3 koristi **Podman**.

## 8.1 Podržane verzije

| Komponenta | RHEL | SLES |
|---|---|---|
| **Podman** | 4.9.4 | 4.9.5 |
| **Podman Compose** | 1.5.0 | 1.5.0 |

> Za najažurnije verzije videti **support compatibility matrix** i **E-Lab Navigator**. Određene verzije Podman Compose-a zahtevaju određene verzije Podman-a — videti Podman Compose release notes.

**Referentna dokumentacija:**
- Podman engine overview: *What is Podman?*
- Podman instalacija: zvanično uputstvo *Podman Installation*
- Podman Compose instalacija: *Install Podman-Compose*

---

## 8.2 Pravila instalacije

| Pravilo | Obrazloženje |
|---|---|
| Instalirati **stabilne** verzije Podman-a i Podman Compose-a | Nestabilne verzije uzrokuju neuspeh instalacije CR-a |
| Instalirati **globalno**, ne za pojedinačnog korisnika | Korisničke instalacije uzrokuju probleme pri rebootu management hosta |
| Ako se koristi firewall — **instalirati Podman tek nakon podešavanja firewall-a** | Podman se pri instalaciji integriše sa firewall konfiguracijom |
| Omogućiti da se Podman automatski restartuje i konfiguriše firewall pri rebootu | Bez ovoga CR servisi ne startuju posle reboota |

**Podman socket:**
`crsetup.sh` automatski omogućava Podman socket tokom instalacije. Ako iz nekog razloga nije omogućen:

```bash
systemctl enable podman.socket
```

---

## 8.3 Uklanjanje Docker-a

| Scenario | Radnja |
|---|---|
| **Bare-metal deployment**, sveža instalacija 20.3 ili 20.2, ili nadogradnja na 20.3 sa 20.2 | Ukloniti **Docker i docker-compose** i **rebootovati sistem** pre nastavka |
| **OVA deployment** | Uklanjanje Docker-a **nije potrebno** — obavljeno je tokom 20.2 instalacije/nadogradnje |
| **Nadogradnja sa verzija starijih od 20.2** | **NE uklanjati** Docker ili docker-compose pre početka nadogradnje. Starije CR verzije koriste Docker servise, a proces nadogradnje ih zahteva za PostgreSQL backup i migraciju. |

> Ovo je jedan od najčešćih uzroka neuspele nadogradnje — redosled je bitan.

---

## 8.4 Verifikacija

```bash
# Prikazuje instalirane verzije Podman-a i Podman Compose-a
# i proverava sve preduslove za instalaciju
./crsetup.sh --check
```

---

# 9. Preduslovna priprema vault DD sistema

Ovo poglavlje opisuje radnje koje se izvršavaju **pre** instalacije Cyber Recovery softvera. Bez ovih koraka `crsetup.sh --check` ili prvo pokretanje politike neće uspeti.

## 9.1 Osnovna konfiguracija DD sistema u vault-u

1. Konfigurisati mrežu i oba Ethernet interfejsa (management + namenski replikacioni).
2. Konfigurisati DNS i NTP.
3. Proveriti DDOS verziju (7.13 ili 8.x+).
4. Proveriti da su instalirane sve potrebne licence:

```bash
sysadmin@dd-vault# license show
```

Očekivane licence: DD Boost, Replication, Retention Lock Governance, Retention Lock Compliance.

---

## 9.2 Kreiranje naloga za Cyber Recovery

```bash
# Kreiranje naloga sa admin ulogom koji Cyber Recovery koristi
# za upravljanje DD operacijama
sysadmin@dd-vault# user add cradmin role admin
```

> Ime naloga je proizvoljno, ali se preporučuje `cradmin`. **Nalog `sysadmin` se ne sme koristiti za Cyber Recovery.**

Isti postupak ponoviti na produkcionom DD sistemu ako to zahteva dizajn rešenja.

---

## 9.3 Konfiguracija NFS opcija

Videti odeljak 5.6 za matricu podešavanja. Minimalna provera:

```bash
sysadmin@dd-vault# nfs option show all
```

Proveriti:
- `force-minimum-root-squash-default` = `disabled`
- `nfs4-idmap-out-numeric` = `always` (ako se koristi NFSv4)
- `default-export-version` i `default-server-version` u skladu sa planiranim režimom

---

## 9.4 Konfiguracija Retention Lock Compliance režima

Izvršiti samo ako se planira Compliance režim ili ako je potreban hardening za Secure Snapshot-ove (DDOS 8.8+).

**Preduslov:** vault DD sistem mora imati Retention Lock Compliance licencu.

```bash
# 1. Prijaviti se kao admin korisnik i kreirati security officer nalog
#    (ako već ne postoji)
sysadmin@dd-vault# user add <account_name> role security

# 2. Odjaviti se i prijaviti kao security officer

# 3. Omogućiti security autorizaciju
so_user@dd-vault# authorization policy set security-officer enabled

# 4. Odjaviti se i prijaviti ponovo kao admin korisnik

# 5. Konfigurisati Retention Lock Compliance
sysadmin@dd-vault# system retention-lock compliance configure

# 6. Na zahtev sistema uneti kredencijale security officer-a
```

> Za sveobuhvatnu proceduru videti *Dell PowerProtect Data Domain Operating System Administration Guide*.
> **Zabeležiti kredencijale security officer-a** — biće potrebni pri kreiranju politike sa Compliance retention lock-om (stranica *Storage Security Credentials* u Add Policy čarobnjaku).

---

## 9.5 Kreiranje MTree-ova

Kreirati MTree-ove prema planu iz odeljka 5.4:

- Po standardnoj politici: replication destination + CR repository + recovery
- Po PPDM politici: najmanje dva MTree-a (data + ServerDR)

Primeniti dogovorenu konvenciju imenovanja (videti Deo I, poglavlje 3.4).

---

## 9.6 Kreiranje i dodela DD Boost korisnika

```bash
# 1. Kreiranje DD Boost korisnika
#    role: admin za NetWorker i Avamar, none za PowerProtect Data Manager
sysadmin@dd-vault# user add <username> role <admin|none>

# 2. Dodela DD Boost korisnika
sysadmin@dd-vault# ddboost user assign <username>
```

---

## 9.7 Usklađivanje UID-ova

Ovo je kritičan i često previđen korak.

**Pravilo:** UID-ovi povezani sa produkcionim policy MTree-ovima moraju postojati na DD sistemu u vault-u.

```bash
# Provera postojećih naloga i njihovih UID-ova na vault DD sistemu
sysadmin@dd-vault# user show list

# Kreiranje naloga sa tačno određenim UID-om
sysadmin@dd-vault# user add <ddboost_username> uid <UID>
```

**Napomene:**
- Na DD sistemu dodela UID-ova kreće od **500**.
- Kod starijih verzija, ako je potreban npr. UID 510, moguće je da treba kreirati do devet privremenih naloga da bi se došlo do željene vrednosti.
- Za Avamar: DD sistem u vault-u mora imati **isto ime DD Boost naloga i isti UID** kao produkcioni.
- Detaljna procedura utvrđivanja potrebnog UID-a preko `crcli policy list-copy` opisana je u Delu IV (NetWorker) i Delu V (PPDM).

**Šta zabeležiti u fazi preduslova:** za svaki produkcioni DD Boost nalog — ime naloga, UID i aplikaciju kojoj pripada.

---

## 9.8 Konfiguracija replication konteksta

1. Kreirati MTree replication kontekste sa produkcionog na vault DD sistem.
2. Pri kreiranju replikacionih parova, port **3009** mora biti otvoren između produkcionog i vault DD sistema. **Zatvoriti ga nakon dodavanja para.**
3. Ako je FIPS omogućen na bilo kom od dva DD sistema, konfigurisati **dvosmernu autentikaciju** za svaki kontekst (videti odeljak 5.7).
4. Za PPDM: pridržavati se standardne konvencije imenovanja za Server DR replication kontekste — Cyber Recovery ih na osnovu imena prepoznaje i izdvaja iz liste data konteksta.

---

## 9.9 Inicijalna replikacija

> **Obavezno:** za svaki replication context mora se izvršiti **inicijalna replikacija** između produkcionog i vault DD sistema **pre** definisanja Cyber Recovery politika.

Bez završene inicijalne replikacije politika se ne može ispravno definisati niti izvršiti.

---

## 9.10 Cascading replication okruženje

Ako dizajn uključuje kaskadnu replikaciju (produkcija → međukorak → vault), postoje dodatna razmatranja:

- Za međukorak replikacije na produkcionoj strani, vreme početka te replikacije mora biti **posle** završetka produkcionog backup-a.
- Detalji su opisani u Delu III (odeljak *About a cascading replication environment*).

---

# Dodatak: Radni list preduslova

Popuniti pre početka implementacije. Kolonu **Status** označiti sa OK / NIJE OK / N/A.

## A. Opšte

| # | Stavka | Vrednost / potvrda | Status |
|---|---|---|---|
| A1 | Verzija Cyber Recovery-ja koja se instalira | 20.3 | |
| A2 | Provereno u E-Lab Navigator / KB 000205512 | | |
| A3 | Tip deploymenta (RHEL softverska instalacija / OVA) | | |
| A4 | Nabavljen instalacioni paket / OVA fajl | | |
| A5 | Licenca za Cyber Recovery obezbeđena | | |

## B. Management host

| # | Stavka | Vrednost / potvrda | Status |
|---|---|---|---|
| B1 | Operativni sistem i verzija | | |
| B2 | Nije CentOS | | |
| B3 | RAM ≥ 4 GB | | |
| B4 | Slobodno ≥ 5 GB za raspakivanje | | |
| B5 | Slobodno ≥ 20 GB za instalaciju | | |
| B6 | `libstdc++` instaliran (softverska instalacija) | | |
| B7 | `rsyslog` + `rsyslog-gnutls` / `rsyslog-module-gtls` | | |
| B8 | `umask` = 022 | | |
| B9 | Docker uklonjen (bare-metal, po potrebi) | | |
| B10 | Hostname i FQDN | | |
| B11 | IP adresa / maska / gateway | | |
| B12 | DNS serveri, rezolucija provereno | | |
| B13 | NTP izvor konfigurisan | | |
| B14 | Vremenska zona podešena | | |
| B15 | SELinux režim (RHEL) | | |

## C. OVA deployment (ako je primenljivo)

| # | Stavka | Vrednost / potvrda | Status |
|---|---|---|---|
| C1 | vCenter / ESXi verzija (8.0.x ili 9.0) | | |
| C2 | Datastore ima ≥ 10 GB za OVA | | |
| C3 | Datastore ima ≥ 195 GB za diskove | | |
| C4 | Datastore **nije** formatiran sa 4K blokom | | |
| C5 | Port grupa / VLAN za CR appliance | | |
| C6 | Thin provisioning odobren | | |

## D. Vault DD sistem(i)

| # | Stavka | Vrednost / potvrda | Status |
|---|---|---|---|
| D1 | Broj DD sistema u vault-u (1–10) | | |
| D2 | Model i DDOS verzija (7.13 ili 8.x+) | | |
| D3 | Vault DD ima više prostora od produkcionog | | |
| D4 | Dva Ethernet interfejsa (mgmt + replikacija) | | |
| D5 | Nalog `cradmin` (role admin) kreiran | | |
| D6 | Licenca DD Boost | | |
| D7 | Licenca Replication | | |
| D8 | Licenca Retention Lock Governance | | |
| D9 | Licenca Retention Lock Compliance | | |
| D10 | Immutability tehnologija (RL / Secure Snapshot) | | |
| D11 | Retention Lock Compliance konfigurisan (ako se koristi) | | |
| D12 | Security officer nalog i kredencijali zabeleženi | | |
| D13 | `force-minimum-root-squash-default` = disabled | | |
| D14 | NFS verzije podešene prema planu | | |
| D15 | `nfs4-idmap-out-numeric` = always (ako NFSv4) | | |
| D16 | Kapacitet za 3 MTree-a po standardnoj politici | | |
| D17 | HA konfiguracija i floating IP-ovi (ako se koristi) | | |
| D18 | FIPS režim — dvosmerna autentikacija konteksta | | |

## E. Produkcioni DD sistem(i)

| # | Stavka | Vrednost / potvrda | Status |
|---|---|---|---|
| E1 | Model, DDOS verzija | | |
| E2 | Nije DP8300 / DP8800 | | |
| E3 | Nema Cloud DR / Cloud Tier u vault-u | | |
| E4 | Replication kontekst(i) kreirani | | |
| E5 | Inicijalna replikacija završena | | |
| E6 | Port 3009 otvoren za dodavanje parova, pa zatvoren | | |
| E7 | Spisak DD Boost naloga sa UID-ovima zabeležen | | |

## F. Mreža

| # | Stavka | Vrednost / potvrda | Status |
|---|---|---|---|
| F1 | Portovi 22, 111, 2049, 2052 otvoreni | | |
| F2 | Portovi 14777, 14778 otvoreni na firewall-u | | |
| F3 | Port 14780 (opciono, Swagger) | | |
| F4 | Port 2051 za replikaciju (nepromenjen) | | |
| F5 | Port 443 (ako se koristi Cyber Detect / OIDC) | | |
| F6 | Port 25 (ako se koriste email notifikacije) | | |
| F7 | Port 123 UDP (NTP) | | |
| F8 | Spisak subnetova u vault-u prikupljen | | |
| F9 | Provereno preklapanje sa Podman opsezima | | |
| F10 | Plan za Podman subnetove (ako ima kolizije) | | |

## G. Backup aplikacije

| # | Stavka | Vrednost / potvrda | Status |
|---|---|---|---|
| G1 | NetWorker — verzija na produkciji | | |
| G2 | NetWorker — verzija u vault-u (identična) | | |
| G3 | NetWorker — platforma (Linux / Windows) | | |
| G4 | NetWorker — instanca neinicijalizovana i dimenzionisana | | |
| G5 | NetWorker na Windows-u — Cygwin + OpenSSH | | |
| G6 | PPDM — verzija na produkciji | | |
| G7 | PPDM — verzija u vault-u (identična) | | |
| G8 | PPDM — deployovan kao OVA na vCenter u vault-u | | |
| G9 | PPDM — NFS v3 dostupan ili DD Boost za ServerDR | | |
| G10 | vCenter u vault-u (ako PPDM < 20.3) | | |
| G11 | Cyber Detect u obimu (da / ne) | | |

## H. Nalozi i kredencijali

| # | Stavka | Zabeleženo | Status |
|---|---|---|---|
| H1 | Lockbox passphrase (min. 9 znakova, FIPS min. 14) | | |
| H2 | Postgres lozinka | | |
| H3 | `crso` lozinka | | |
| H4 | Admin korisnik CR-a | | |
| H5 | Vault Operator korisnik CR-a | | |
| H6 | DD `cradmin` nalog i lozinka | | |
| H7 | DD security officer nalog i lozinka | | |
| H8 | DD Boost nalozi (ime + UID + aplikacija) | | |
| H9 | vCenter administratorski nalog | | |
| H10 | NetWorker `root` / `Admin` kredencijali u vault-u | | |
| H11 | PPDM `admin` kredencijali u vault-u | | |

> **CAUTION — lockbox passphrase.** Lockbox passphrase se **ne može povratiti**. Ako se zaboravi ili izgubi, potrebna je sveža instalacija Cyber Recovery softvera. Passphrase je neophodan za nadogradnje i za reset lozinke Security Admin korisnika. Čuvati ga na siguran i redundantan način van CR servera.

---

*Kraj Dela II. Sledeći deo: **Deo III — Instalacija** (poglavlja 10–15).*
