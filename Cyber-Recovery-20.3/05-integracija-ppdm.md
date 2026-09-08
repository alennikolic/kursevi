# DEO V — INTEGRACIJA SA POWERPROTECT DATA MANAGER-om

**Dell PowerProtect Cyber Recovery 20.3 — Uputstvo za instalaciju i integraciju**
Verzija dokumenta: 0.1 | Namena: System Engineer

> **Izvori:**
> - *Dell PowerProtect Cyber Recovery 20.3 Installation and Upgrade Guide*
> - *Dell PowerProtect Cyber Recovery 20.3 Product Guide*
> - *PowerProtect Data Manager 19.22 Administrator Guide* — poglavlje *Preparing for and recovering from a disaster*
> - *PowerProtect Data Manager 19.22 Deployment Guide*
>
> **Preduslov:** završeni Deo II (preduslovi) i Deo III (instalacija).

> ### ⚠ Napomena o verziji — pročitati pre svega ostalog
> Cyber Recovery 20.3 se **bitno različito ponaša** prema PPDM verzijama starijim od 20.3 i verzijama 20.3+. Razlike obuhvataju VM snapshot, kredencijale, potrebu za vCenter asset-om i mogućnost ponovne upotrebe appliance-a.
>
> **Priložena PPDM dokumentacija je za verziju 19.22**, koja spada u kategoriju **„starija od 20.3"**. Svi PPDM-specifični koraci u ovom dokumentu odgovaraju toj grani. Gde postoji razlika za PPDM 20.3+, ona je eksplicitno označena, ali **nije potkrepljena PPDM dokumentacijom za tu verziju** — videti Dodatak B.

---

## Sadržaj

- [25. Planiranje integracije](#25-planiranje-integracije)
- [26. Priprema produkcijskog PPDM okruženja](#26-priprema-produkcijskog-ppdm-okruženja)
- [27. Priprema PPDM instance u vault-u](#27-priprema-ppdm-instance-u-vault-u)
- [28. Konfiguracija PPDM-a u Cyber Recovery](#28-konfiguracija-ppdm-a-u-cyber-recovery)
- [29. Izvršavanje PPDM recovery-ja](#29-izvršavanje-ppdm-recovery-ja)
- [30. Post-recovery koraci](#30-post-recovery-koraci)
- [31. Recovery Check i čišćenje](#31-recovery-check-i-čišćenje)
- [Dodatak A: Checklist PPDM integracije](#dodatak-a-checklist-ppdm-integracije)
- [Dodatak B: Otvorena pitanja](#dodatak-b-otvorena-pitanja)

---

# 25. Planiranje integracije

## 25.1 Kako Cyber Recovery oporavlja PPDM

Za razliku od NetWorker-a, PPDM oporavak zahteva **dva** replikaciona toka:

| Tok | Sadržaj | Uloga |
|---|---|---|
| **Data replication context** | Policy MTree-ovi sa backup podacima assets-a | Podaci koje treba oporaviti |
| **ServerDR context** | Server DR backup PPDM sistema | Konfiguracija, baze i metapodaci PPDM-a |

Zato **PPDM politika u Cyber Recovery zahteva najmanje dva MTree-a**.

Kada pokrenete oporavak, Cyber Recovery priprema okruženje tako da možete pokrenuti PPDM restore iz aplikativne konzole. U sklopu tog procesa softver:

1. Kreira produkciono DD Boost korisničko ime i lozinku na DD sistemu.
2. Restartuje PPDM appliance.
3. Kreira recovery sandbox i popunjava ga izabranom kopijom.

## 25.2 Šta je Server DR na PPDM strani

Iz *PPDM Administrator Guide*: proces oporavka sistema kreira periodične backup-e PPDM sistema. **Svaki backup se smatra full backup-om**, iako se kreira inkrementalno.

Server DR backup obuhvata:
- Lockbox
- PPDM baze podataka
- File Search indekse
- Ostale sistemske komponente

**Tip protection storage-a:** PPDM podržava **DD Boost** kao tip protection storage-a za server DR. DD Boost obezbeđuje autentikaciju zaštićenu lozinkom, server DR replikaciju i automatsku server DR konfiguraciju.

**Konvencija imenovanja** — ovo je ključno za konfiguraciju CR politike:

| Objekat | Ime |
|---|---|
| Lokalna storage unit i nalog | `SysDR_<hostname>` |
| Replica storage unit i nalog | `SysDR-R_<hostname>` |

> `<hostname>` je hostname PPDM sistema. **Imena naloga mogu imati najviše 32 znaka.** Lozinka DD Boost korisnika se automatski generiše sigurnosnim algoritmom pri kreiranju storage unit-a.

Cyber Recovery iz liste **data** replication konteksta izuzima kontekste koji odgovaraju ovoj konvenciji — čime sprečava da se ServerDR kontekst greškom izabere kao data kontekst.

## 25.3 Obavezni način deploymenta u vault-u

> **Za automatski recovery, PPDM u vault-u mora biti deploy-ovan kao OVA na vCenter.**

| Okruženje | Podrška za automatski recovery |
|---|---|
| On-premises | Da — **isključivo OVA na vCenter** |
| AWS | Da — odgovarajući PPDM cloud image deploy-ovan u vault |
| Azure | Da — odgovarajući PPDM cloud image deploy-ovan u vault |
| GCP | **Nije podržano** |

> Na **produkcionoj** strani PPDM može biti deploy-ovan i preko install package-a na Linux, i preko OVA na vCenter, kao i na DM5500 / DM5510 uređajima. Ograničenje se odnosi samo na vault.

> **Dodatno ograničenje sa PPDM strane:** DR backup snimljen sa vCenter OVA ili cloud machine image deploymenta **ne može** se vratiti na install package deployment, i obrnuto. To pojačava zahtev da vault instanca bude OVA — ako je produkcija OVA, vault mora biti OVA.

## 25.4 Šta se NE oporavlja

Ako izvršite automatski PPDM oporavak u vault-u, sledeći engine-i **nisu** oporavljeni:

| Komponenta | Napomena |
|---|---|
| **Protection engine** | Uključuje NAS protection engine i TSDM data movers |
| **PowerProtect Search** | Koristi se za pretragu fajlova i restore |
| **Reporting engine** | Za backup izveštaje |

**Razlog:** to su zasebni VM-ovi koje PPDM deploy-uje na produkcionom sistemu. Pošto se hostname/IP adresa vault PPDM sistema razlikuje, Cyber Recovery ne može da ih deploy-uje tokom oporavka.

> **Potvrda sa PPDM strane:** *Administrator Guide* navodi isto — proces oporavka **ne redeploy-uje automatski protection engine-e**; treba ih ponovo deploy-ovati posle oporavka. Za vCenter OVA deployment, Search Engine i reporting engine node-ove iz prethodnog deploymenta treba **obrisati iz vCenter-a pre** oporavka, a zatim kreirati nove — dovoljno konfigurisane da ih proces oporavka može kontaktirati i prepisati konfiguracijom iz backup-a.

## 25.5 Razlike PPDM < 20.3 vs. PPDM ≥ 20.3

| Aspekt | PPDM **< 20.3** (uključujući 19.22) | PPDM **≥ 20.3** |
|---|---|---|
| **VM snapshot appliance-a** | CR uzima snapshot; koristi se za vraćanje PPDM-a posle oporavka | CR **ne uzima** i **ne vraća** snapshot |
| **Ponovna upotreba appliance-a** | Appliance mora biti u svežem/podrazumevanom stanju | Može se koristiti appliance korišćen za raniji oporavak; **ne mora** biti svež |
| **vCenter asset u CR** | **Obavezan** za on-premises deployment | Nije potreban |
| **Kredencijali pri dodavanju aplikacije** | Kredencijali produkcione PPDM aplikacije; **stranica Host Authentication** | Trenutni application administrator kredencijali PPDM appliance-a **u vault-u**; **stranica Vault Application Authentication** + zasebno definisani post-recovery kredencijali |
| **Podrazumevani kredencijali** | CR koristi podrazumevane kredencijale za validaciju FQDN/IP — **ne menjati ih pre dodavanja aplikacije** | Appliance se dodaje u stanju **Operational Running**; nije potrebno Pending ili Maintenance Ready stanje |
| **`sshd_config`** | Mora se omogućiti `PasswordAuthentication yes` | Nije navedeno kao zahtev |
| **FQDN** | Mora odgovarati DNS imenu prikazanom u vCenter UI (CR ga koristi za kreiranje VM snapshot-a) | Nije navedeno kao zahtev |
| **Postojeći VM snapshot-ovi** | Ne sme ih biti na PPDM VM-u u vCenter-u | Nije navedeno kao zahtev |
| **Post-recovery lozinke** | `root` i `admin` na host serveru dobijaju lozinku uneta u admin password polje pri dodavanju aplikacije | Prijava sa post-recovery kredencijalima; nije potrebno usklađivati sa produkcijom |
| **Cleanup** | Softver koristi cleanup job da vrati sistem, pa briše sandbox-ove; **čekati ~10 min** na VM rollback | Softver briše sandbox-ove bez vraćanja snapshot-a |

## 25.6 Ostale mogućnosti i ograničenja

| Stavka | Detalj |
|---|---|
| **Linked recovery** | Zahteva PPDM **19.19 ili noviji** na produkciji i u vault-u. Omogućava istovremeni oporavak više kopija. |
| **MFA na produkciji** | CR podržava oporavak za produkcioni deployment sa PPDM **19.14+** i uključenom multifactor autentikacijom. |
| **Više DD sistema u vault-u** | Moguć je istovremeni (concurrent) PPDM oporavak. Videti *Replication PowerProtect Data Manager Backups to the Cyber Recovery Vault — A User Journey* na PowerProtect Cyber Recovery Info Hub. |
| **One-to-many** | Za CR 19.19 deployment, konfiguracija u kojoj se kopije repliciraju sa jednog produkcionog DD na **dva** vault DD sistema **nije podržana**. Kada izaberete kopiju na jednom vault DD, kopija na drugom je onemogućena. Opcija **Select Latest Copies** je takođe onemogućena. |
| **Verzija u vault-u** | Mora biti **ista** kao verzija produkcionog sistema. Sa PPDM strane: verzija backup-a mora odgovarati verziji PPDM-a u produkciji — **odnosi se samo na major verzije, ne na patch izdanja**. |
| **DDOS na vault DD** | Mora biti **7.13 ili 8.x** |

## 25.7 Tok integracije — pregled

```
PRODUKCIJA                                  VAULT
──────────                                  ─────
PPDM (OVA ili install package)              PPDM (OVA na vCenter — obavezno)
  │                                                    ▲
  ├─ protection policies ──► policy MTree-ovi          │ restore iz konzole
  │                              │                     │
  └─ Server DR ──► SysDR_<host>  │              Recovery sandbox(-ovi)
       (DD Boost storage unit)   │                     ▲
                                 │                     │ populate
Produkcioni DD ──────────────────┴──────────►  Vault DD
   │  data replication context(s)                      ▲
   │  ServerDR replication context                     │
   └───────────────────────────────────────►  CR politika tipa PPDM
                                               (≥ 2 MTree-a)
```

---

# 26. Priprema produkcijskog PPDM okruženja

## 26.1 DD Boost korisnik za policy MTree-ove

```bash
# role za PowerProtect Data Manager je "none"
sysadmin@dd-prod# user add <ppdm_ddboost_user> role none
sysadmin@dd-prod# ddboost user assign <ppdm_ddboost_user>
```

> **Pažnja:** za PPDM je `role` **`none`**, za razliku od NetWorker-a i Avamar-a gde je `admin`.

## 26.2 Konfiguracija Server DR backup-a

### Automatska konfiguracija

Novi PPDM deployment-i **automatski konfigurišu i omogućavaju server DR** uz minimalan unos. Mehanizam detektuje kada prvi put dodate protection storage sistem i koristi preporučeni DD Boost tip i podrazumevana podešavanja da kreira upravljanu storage unit za server DR.

> Automatska konfiguracija bira **prvi** protection storage sistem koji dodate u PPDM. Ako to nije DD sistem koji se replicira u vault, konfiguraciju treba promeniti ručno.

### Ručna konfiguracija

**Preduslov:** DD sistem je dodat kao protection storage. Ako se planira replikacija server DR backup-a, replikacioni target mora biti **drugi** protection storage sistem.

1. Prijaviti se u PPDM UI kao korisnik sa **Administrator** ulogom.
2. **System Settings → Disaster Recovery → Configuration**
3. Izabrati **Enable backup**.
4. Iz padajuće liste **Protection Storage** izabrati backup destinaciju, ili **Add** za dodavanje novog sistema.

   > Pri inicijalnoj konfiguraciji polje **Storage Unit** je prazno. Ako je server DR već konfigurisan, polje prikazuje ime storage unit-a sa server DR backup-ima.

5. Konfigurisati učestalost i trajanje:
   - **Interval između server DR backup-a, u satima** — dozvoljene vrednosti **1 do 24**
   - **Broj dana čuvanja server DR backup-a** — dozvoljene vrednosti **2 do 30**

6. Opciono, za server DR replikaciju:
   - Označiti **Enable Replication**
   - Iz **Replicate Backup To** izabrati target (ne sme biti isti kao backup destinacija)
   - Učestalost i retencija replikacije su iste kao za backup

   > Ako izvorna storage unit ima uključen compliance mode retention locking, morate uneti korisničko ime i lozinku **security officer-a** povezanog sa izvornim i odredišnim protection storage sistemima.

7. **Save**

Rezultat: PPDM kreira sistemske job-ove za pripremu storage unit-a i konfiguraciju server DR protection politike, i job za prvi server DR backup. **Proveriti da su sistemski job-ovi uspeli.**

### ifGroups — obavezno

> **Kritično:** proveriti da su **ifGroups omogućeni i ispravno konfigurisani** na DD sistemu. Ako ifGroups nisu omogućeni ili su pogrešno konfigurisani, **server DR backup-i se ne mogu uspešno završiti**. Za detalje videti *Dell PowerProtect Data Domain Operating System Administration Guide*.

## 26.3 Retencija server DR backup-a

| Tip backup-a | Retencija | Napomene |
|---|---|---|
| **Hourly** | 3 sata | |
| **Hourly sa compliance mode retention locking** | 12 sati | Compliance mode RL se automatski omogućava kada je retention locking uključen na asset-u u protection politici. **Backup-i ne podržavaju governance mode retention locking.** |
| **Daily** | 7 dana | Poslednji hourly backup koji se završi pre ponoći GMT automatski se promoviše u daily backup. |
| **Manual** | 30 dana | Manualni backup-i se mogu obrisati **samo ručno**. |

**Brisanje kopija:**
- System protection servis radi dnevno u ponoć GMT i automatski briše sve istekle kopije.
- Istekle kopije mogu se obrisati i ručno u bilo kom trenutku.
- Servis **ne briše** i ne mogu se ručno obrisati: najnovija kopija označena kao `FULL` i najnovija kopija označena kao `PARTIAL`.

> **Implikacija za Cyber Recovery:** ako je server DR retencija na produkciji podešena na minimum (2 dana), a Sync operacija u vault-u se izvršava ređe, može se desiti da u trenutku Sync-a više ne postoji odgovarajući server DR backup. **Uskladiti PPDM server DR retenciju sa učestalošću CR Sync operacije.**

## 26.4 Ograničenja server DR backup-a

- Ne mogu se izvršavati **više server DR backup operacija istovremeno**.
- Ne može se izvršiti **component DR recovery operacija tokom server DR backup operacije**.
- DR backup sa OVA/cloud image deploymenta **ne može** se vratiti na install package deployment, i obrnuto.

## 26.5 Konvencija imenovanja VM-a u vCenter-u

> **Za PPDM verzije starije od 20.3, on-premises deployment:** u imenu VM-a u vCenter-u **ne smeju** se koristiti sledeći specijalni znaci, jer Cyber Recovery tada ne može da detektuje PPDM VM:

```
%  &  *  $  #  @  !  \  /  :  *  ?  "  <  >  [  ]  |  ;  '
```

**Proveriti pre implementacije** i po potrebi preimenovati VM.

## 26.6 UID-ovi policy MTree-ova

> UID-ovi povezani sa **produkcionim policy MTree-ovima** moraju biti dostupni na DD sistemu u Cyber Recovery vault-u.

Postupak utvrđivanja i kreiranja UID-ova je isti kao za NetWorker — videti Deo IV, poglavlje 20.

**Utvrđivanje UID-a server DR DD Boost korisnika (PPDM procedura):**

1. Povezati se na PPDM konzolu.
2. Za deployment koji koristi vCenter OVA ili cloud machine image — preći na `root` korisnika.
3. Za deployment koji koristi install package:
   ```bash
   sudo ppdmadmin shell -c core --user root
   ```
4. Promeniti direktorijum:
   ```bash
   cd /usr/local/brs/puppet/scripts
   ```
5. Povezati se na konzolu protection storage sistema i preuzeti UID:
   ```bash
   sysadmin@dd-prod# user show list
   ```

## 26.7 Evidencija podataka za DR — obavezno

*PPDM Administrator Guide* propisuje beleženje sledećih podataka **na lokalni disk van PowerProtect Data Manager-a**:

| Sistem | Podatak | Primer | Zabeležena vrednost |
|---|---|---|---|
| PowerProtect Data Manager | Verzija i build | 19.22 | |
| PowerProtect Data Manager | FQDN | `server1.example.com` | |
| PowerProtect Data Manager | Backup protokol | DD Boost | |
| Server DR replika | FQDN | `dd-replica.example.com` | |
| Protection storage sistem | FQDN | `dd.example.com` | |
| Protection storage sistem | DD Boost username | `SysDR_server1` | |
| Protection storage sistem | DD Boost password | | |
| Protection storage sistem | **DD Boost UID** | 501 | |

**Dodatno, ako je PPDM deploy-ovan na vSphere:** zabeležiti **port group** podešavanja dodeljena PPDM-u (vSphere client → desni klik na virtuelni appliance → Edit Settings). Ovaj podatak je koristan pri vraćanju u isto VMware okruženje.

> **CAUTION iz PPDM dokumentacije:** *nikada ne menjajte DD Boost lozinku naloga koji koristi server DR storage unit, osim ako se upravo spremate da izvršite server DR recovery operaciju.*
>
> Ova lozinka se **ne može** promeniti iz PPDM UI-a — menja se na DD sistemu. Promene predefinisane administratorske lozinke PPDM-a povlače odgovarajuća ažuriranja DD Boost lozinke; ako je konfigurisana server DR replikacija, i kredencijala na replikacionom target-u.

## 26.8 Usklađivanje rasporeda replikacije

Isto pravilo kao za NetWorker: replikacioni prozor mora početi **posle** završetka aplikativnog i DR backup-a na produkciji.

Za PPDM to znači:
1. Pokrenuti aplikativne (protection policy) backup-e.
2. Pokrenuti DR backup u PPDM produkcionom okruženju.
3. Tek onda izvršiti Secure Copy operaciju koja kopira podatke u vault.

**Radni list:**

| Stavka | Vrednost |
|---|---|
| Interval server DR backup-a (1–24 h) | |
| Retencija server DR backup-a (2–30 dana) | |
| Tipično vreme završetka protection policy backup-a | |
| Predloženo vreme početka CR Sync operacije | |
| Da li je server DR replikacija omogućena | |

## 26.9 Ručni server DR backup

Pre prve validacije integracije korisno je pokrenuti backup ručno:

1. PPDM UI → **System Settings → Disaster Recovery → Manage Backups**
2. **Backup Now**
3. Opciono uneti ime backup-a.

   > Ako se ime ostavi prazno, PPDM koristi konvenciju `UserDR-`. Ako unesete ime po konvenciji koju PPDM koristi za zakazane backup-e (`SystemDR`), prikazuje se greška.

4. **Start Backup**

Praćenje: **Jobs → System Jobs**, job pod imenom **`Protect the server datastore`**.

> Ako je Search Engine deploy-ovan, PPDM backup-uje i njega. Detalji backup-a prikazuju status Search Engine backup-a.

---

# 27. Priprema PPDM instance u vault-u

## 27.1 Osnovni zahtevi

| Zahtev | Detalj |
|---|---|
| **Način deploymenta** | **OVA na vCenter** — obavezno za automatski recovery |
| **Verzija** | Ista kao verzija produkcionog sistema (major verzija) |
| **Namena** | **Isključivo** za izvršavanje oporavka u vault-u |
| **DDOS na vault DD** | 7.13 ili 8.x |
| **Stanje appliance-a (< 20.3)** | Bez postojećih VM snapshot-ova |
| **Stanje appliance-a (≥ 20.3)** | Operational Running; ne mora biti svež |

## 27.2 Resursi za PPDM appliance

Iz *PPDM 19.22 Deployment Guide*, minimalni sistemski zahtevi u VMware okruženju:

| Resurs | Vrednost |
|---|---|
| CPU | **10 jezgara** (preporučeno 14) |
| Memorija | **36 GB RAM** |
| Swap | 8 GB |
| NIC | 1 GB |
| Disk 1 | 100 GB |
| Disk 2 | 500 GB |
| Diskovi 3 i 4 | 10 GB svaki |
| Diskovi 5 do 7 | 5 GB svaki |

**Sa Cloud DR Add-On:** 14 CPU jezgara, 40 GB memorije.

> **Preporuka:** konfigurisati swap memoriju na **SSD** i koristiti SSD datastore pri deploymentu PPDM servera. Većina PPDM servisa je memorijski intenzivna; kada raspoloživa fizička memorija padne ispod praga, servisi počinju da koriste swap. Ako swap leži na sporom disku, to značajno utiče na Java Garbage Collection aktivnost.

> **Vrednost memorije mora biti umnožak 4 GB** ako je naknadno menjate.

## 27.3 Deployment OVA u vault vCenter

1. Preuzeti OVA paket sa Dell Support sajta i preneti ga u vault.
2. Prijaviti se u **vSphere Client** u vault-u.
3. **Actions → Deploy OVF Template**
4. **Select an OVF template** — izabrati **Local File** i OVA paket.
5. **Select a name and folder** — ime virtuelnog appliance-a i lokacija.

   > **Proveriti da ime ne sadrži zabranjene specijalne znake** (videti 26.5), ako je PPDM verzija starija od 20.3.

6. **Select a compute resource** — odredišni compute resurs.
7. **Review details** — verifikovati detalje OVF šablona.
8. **Configuration** — izabrati konfiguraciju koja odgovara okruženju.
9. **Select storage:**
   - Izabrati datastore
   - **Select virtual disk format: `Thick provision lazy zeroed`**

   > Ovo je suprotno preporuci za sam Cyber Recovery OVA (thin provisioning). Za PPDM appliance dokumentacija izričito navodi thick provision lazy zeroed.

10. **Select networks** — izabrati odredišnu mrežu za svaku izvornu mrežu.

    > Podrazumevano je izabrana **prva** mreža na listi. Proveriti da je izabrana ispravna — može ne biti prva.

11. **Customize template:**
    - Network IP address
    - Default gateway
    - Network netmask
    - DNS server (do tri)
    - **Fully qualified domain name** (oblik `hostname.domain`)
12. **Ready to Complete** → verifikovati → **Finish**

> **Preporuka iz PPDM dokumentacije:** deploy-ovati OVA na **vCenter server**, a ne direktno na ESXi server.

## 27.4 FQDN i hostname

> **Zahtev iz PPDM dokumentacije:** FQDN PowerProtect Data Manager-a mora biti **isti kao hostname**.

**Za PPDM verzije starije od 20.3, on-premises deployment:** koristiti **isti FQDN ili hostname koji je prikazan pod DNS name u vCenter korisničkom interfejsu**. Cyber Recovery koristi tu informaciju za kreiranje VM snapshot-a tokom oporavka.

**Ako se FQDN vault sistema razlikuje od produkcionog** — što je čest slučaj — tokom oporavka može doći do **mount greške**. Zaobilazno rešenje iz PPDM dokumentacije:

1. Na DD sistemu na kojem se nalazi backup, obrisati replikacioni par i montirati ga za PowerProtect Data Manager.
2. Za vCenter OVA deployment, po završetku oporavka obnoviti sertifikate:
   ```bash
   sudo -H -u admin /usr/local/brs/puppet/scripts/generate_certificates.sh -c
   ```
3. Restartovati sistem i izabrati URL primarnog PPDM sistema.
   Prikazuje se stranica `https://<PPDM_IP>/#/progress` i oporavak se nastavlja.
4. Prijaviti se u primarni PPDM UI koristeći originalnu IP adresu.

   > PPDM VM vCenter konzola prikazuje grešku koju možete ignorisati.

## 27.5 Omogućavanje password autentikacije (PPDM < 20.3)

Za PPDM verzije starije od 20.3, izmeniti `/etc/ssh/sshd_config`:

```bash
# Promeniti vrednost polja PasswordAuthentication sa no na yes
PasswordAuthentication yes
```

Zatim restartovati servis:

```bash
service sshd restart
```

## 27.6 Kredencijali (PPDM < 20.3)

Za PPDM verzije starije od 20.3, potrebni su **host kredencijali** i **aplikativni kredencijali**.

**Host username i host password:**
- Host username je korisničko ime administratora operativnog sistema za **`admin`** nalog PPDM servera **u vault-u**. Tipično `admin`.
- **Lozinka za host administratora ne mora da odgovara lozinci produkcionog naloga.** Može biti bilo koja vrednost po izboru.
- Ovi kredencijali se koriste u **drugoj fazi** recovery procedure, radi zaštite oporavljenih podataka od potencijalnih napada.
- **Posle oporavka, `admin` i `root` nalozi na host serveru koriste ovu lozinku.**

> **Ne menjati podrazumevane kredencijale pre dodavanja aplikacije.** Prvi korak čarobnjaka koristi podrazumevane kredencijale da verifikuje FQDN/IP i validira PPDM server. Ako se podrazumevani kredencijali izmene, čarobnjak prikazuje grešku i PPDM se ne može dodati u CR deployment.

**Aplikativni kredencijali (application administrator):**
- Application administrator **username mora odgovarati** administratorskom korisničkom imenu na produkcionom PPDM sistemu.
- Application administrator **password ne mora** odgovarati lozinci na produkcionom sistemu.

## 27.7 NFS v3 vs. DD Boost za ServerDR MTree

> **Kritično ograničenje:** ako je na DD sistemu u vault-u omogućen **samo NFS v4**, PPDM oporavak **ne uspeva**. PPDM server DR podržava isključivo **NFS v3**.

**Dva rešenja:**

| Rešenje | Kako |
|---|---|
| **DD Boost za ServerDR MTree** (preporučeno) | Koristiti DD Boost protokol umesto NFS-a. PPDM dokumentacija takođe preporučuje DD Boost kao efikasniji protokol. |
| **Omogućiti NFS v3 i NFS v4** | Na vault DD: `Default Export Version` i `Default Servers Version` postaviti na NFSv3 **i** NFSv4, a `NFSv4 ID Map Out Numeric` na `always` |

> Ako se koristi NFS put oporavka, DD sistem mora dozvoliti PPDM hostname-u ili IP adresi da izvrši NFS client mount.

## 27.8 Search Engine i reporting engine u vault-u

Za vCenter OVA deployment, **pre** oporavka:

1. Ako su Search Engine i reporting engine node-ovi iz prethodnog PPDM deploymenta i dalje hostovani na vCenter serveru — **obrisati ih sa vCenter servera**. Proces oporavka ih ponovo deploy-uje.
2. Kreirati **novi Search Engine** i **novi reporting engine**. Dovoljno je da budu konfigurisani toliko da ih proces oporavka može kontaktirati i prepisati konfiguracijom iz backup-a.

> **Podsetnik:** Cyber Recovery navodi da se ove komponente **ne oporavljaju** u vault-u (videti 25.4). Redosled radnji sa PPDM strane treba uskladiti sa realnim obimom oporavka u vault-u — pre implementacije potvrditi sa Dell-om koji je očekivani ishod u vault scenariju (Dodatak B).

## 27.9 DM5500 / DM5510

Ako se oporavlja sa DM5500 ili DM5510 uređaja:

1. Kreirati Cyber Recovery politiku sa **PPDM** tipom iz DM5500 ili DM5510 backup-a.
2. Posle oporavka PowerProtect Data Manager-a u CR UI-u, za prijavu u PPDM UI koristiti kredencijale:
   ```
   admin / Abcd!2345
   ```

---

# 28. Konfiguracija PPDM-a u Cyber Recovery

## 28.1 Replication konteksti

Kreirati **dva** tipa replication konteksta sa produkcionog na vault DD sistem:

| Kontekst | Sadržaj |
|---|---|
| **Data replication context** | Policy MTree-ovi. PPDM politika može imati **više** data konteksta. |
| **ServerDR replication context** | MTree sa server DR backup-om (`SysDR_<hostname>`) |

- Port **3009** mora biti otvoren pri dodavanju replikacionog para; **zatvoriti ga posle**.
- Izvršiti **inicijalnu replikaciju** pre definisanja politike.
- Ako je FIPS omogućen na bilo kom DD sistemu, konfigurisati dvosmernu autentikaciju za svaki kontekst.

## 28.2 Dodavanje vCenter servera (samo PPDM < 20.3)

Za on-premises vault sa PPDM verzijom starijom od 20.3, vCenter asset je **obavezan** — bez njega se PPDM aplikacija ne može dodati u CR deployment.

**Main Menu → Infrastructure → Assets → vCenters → Add**

| Polje | Opis |
|---|---|
| **Nickname** | Naziv za vCenter server |
| **FQDN or IP Address** | FQDN ili IP adresa. Pri izmeni morate ponovo uneti sve lozinke. |
| **Username** | Administratorsko korisničko ime |
| **Password** | Administratorska lozinka |
| **Tags** | Opciono |

> Ako je vCenter pridružen PPDM aplikaciji verzije starije od 20.3, nadogradnja aplikacije na 20.3+ **ne uklanja** postojeću asocijaciju.

> Za CR deployment na AWS-u vCenter asset **nije potreban**.

## 28.3 Dodavanje PPDM aplikacije kao asset-a

**Main Menu → Infrastructure → Assets → Applications → Add**

### Stranica 1 — Application Information

| Polje | Vrednost |
|---|---|
| **Application Type** | `PPDM` |
| **Nickname** | Ime aplikativnog objekta |
| **FQDN or IP Address** | FQDN ili IP adresa PPDM appliance-a u vault-u |
| **Tags** | **Dodati tag koji označava DD Boost username konfigurisan za produkcionu aplikaciju** |
| **Security Group Tag** | Samo AWS deployment |
| **Resource Group** | Samo Azure deployment |

**Validaciono ponašanje po verziji:**

| Verzija | Ponašanje |
|---|---|
| **< 20.3** | CR validira da IP adresa pripada PPDM serveru i proverava da su podešeni podrazumevani kredencijali. **Ne menjati podrazumevanu lozinku pre dodavanja aplikacije** — inače čarobnjak prikazuje grešku. |
| **≥ 20.3** | Appliance se može dodati kada je u stanju **Operational Running**. Nije potrebno Pending ili Maintenance Ready stanje. Ako se appliance ne može validirati, čarobnjak prikazuje grešku. |

> **Application Type se ne može menjati za postojeću aplikaciju.**

### Stranica 2 — Host Authentication (osim za PPDM 20.3+)

| Polje | Vrednost |
|---|---|
| **Username** | Korisničko ime administratora OS-a hosta u vault-u — tipično `admin` |
| **Password** | Lozinka host administratora. **Ne mora** odgovarati lozinci produkcionog naloga. |
| **SSH Port Number** | SSH port aplikacije |
| **Reset Host Fingerprint** | Samo Security Admin |

### Stranica — vCenter Name (samo on-premises, PPDM < 20.3)

Izabrati vCenter server dodat u koraku 28.2. Nije potrebno za PPDM 20.3+.

### Stranica — Vault Application Authentication (samo PPDM 20.3+)

| Polje | Vrednost |
|---|---|
| **Application Username** | Korisničko ime aplikativnog korisnika. **Mora odgovarati** application username-u na produkcionom sistemu. |
| **Application Password** | Lozinka aplikativnog korisnika. **Mora odgovarati** lozinci na produkcionom sistemu. |
| **Reset Authentication** | Označiti ako: sertifikat poverenja između CR i PPDM ističe ili je istekao; ili je promenjena IP adresa CR hosta. Ako se promeni hostname PPDM aplikacije, checkbox se označava automatski. |
| **Certificate Validation** | Ako sertifikat nije uspostavljen: **Verify** → označiti **Accept** → **Done** |

### Stranica — Post-Recovery Application Authentication (samo PPDM 20.3+)

Zasebno se definišu kredencijali koji se dodeljuju **posle** oporavka. Njima se prijavljujete u oporavljenu PPDM aplikaciju.

> **Application administrator username mora odgovarati** administratorskom korisničkom imenu na produkcionom PPDM sistemu. **Password ne mora** odgovarati.

### Obnavljanje sertifikata (PPDM 20.3+)

Kada sertifikat za autentikaciju između CR i PPDM ističe, CR šalje obaveštenje. Postupak:

1. **Infrastructure → Assets → Applications**
2. Izabrati PPDM aplikaciju → **Edit**
3. Otići na stranicu **Vault Application Authentication**
4. Označiti **Reset Authentication**
5. Uneti trenutne **Application Username** i **Application Password** za PPDM sistem u vault-u
6. **Next** → na stranici **Post-Recovery Application Authentication** izmeniti kredencijale ako je potrebno → **Next**
7. Pregledati **Summary** → **Finish**

> Postojeća podešavanja autentikacije se zadržavaju. Ako se kredencijali nisu menjali, na tim stranicama nije potrebna nikakva radnja.

## 28.4 Kreiranje politike tipa PPDM

**Main Menu → Policies → Add**

### Stranica 1 — Policy Information

| Polje | Vrednost |
|---|---|
| **Policy Name** | Naziv politike |
| **Policy Type** | **`PPDM`** |
| **Storage Target** | Vault storage sa replication kontekstima |
| **Tags** | Opciono |

> **PPDM politika zahteva najmanje dva MTree-a za konfiguraciju.**

### Stranica 2 — Replication

| Polje | Vrednost |
|---|---|
| **Replication Context** | Izabrati data replication kontekst i **namenski replikacioni Ethernet port**. Za dodatne kontekste: **Add Replication Context**. |
| **ServerDR Context** | Izabrati kontekst sa server disaster recovery informacijama i odgovarajući Ethernet port |

**Napomene:**
- **PPDM je jedini tip politike koji može imati više data replication konteksta.**
- Cyber Recovery iz liste data konteksta **izuzima** kontekste koji odgovaraju standardnoj PPDM Server DR konvenciji imenovanja (`SysDR_<hostname>`, `SysDR-R_<hostname>`), a te kontekste prikazuje **na vrhu liste** za ServerDR Context.
- **Ne birati data ili management Ethernet interfejse.**

### Stranice 3–6

Retention Lock, Scheduling, Storage Security Credentials i Summary — identično kao za Standard politiku (videti Deo III-a, poglavlje 15.4).

---

# 29. Izvršavanje PPDM recovery-ja

## 29.1 Checklist preduslova

| # | Preduslov |
|---|---|
| 1 | Prijavljeni ste kao **Admin** ili **Vault Operator** korisnik |
| 2 | PPDM u vault-u deploy-ovan kao **OVA na vCenter** |
| 3 | Verzija u vault-u ista kao produkciona (major verzija) |
| 4 | Vault DD radi pod DDOS 7.13 ili 8.x |
| 5 | NFS v3 dostupan **ili** DD Boost se koristi za ServerDR MTree |
| 6 | UID-ovi produkcionih policy MTree-ova postoje na vault DD |
| 7 | PPDM definisan kao application asset u Cyber Recovery |
| 8 | CR politika tipa PPDM kreirana, sa data i ServerDR kontekstom |
| 9 | Aplikativni i DR backup-i izvršeni na produkciji |
| 10 | Secure Copy operacija izvršena, PIT kopija postoji |
| 11 | **< 20.3:** vCenter asset dodat u CR |
| 12 | **< 20.3:** nema postojećih VM snapshot-ova PPDM VM-a |
| 13 | **< 20.3:** ime VM-a bez zabranjenih specijalnih znakova |
| 14 | **< 20.3:** `PasswordAuthentication yes` u `sshd_config` |
| 15 | **< 20.3:** podrazumevani kredencijali nisu menjani pre dodavanja aplikacije |
| 16 | **19.19+:** za linked recovery, i produkcija i vault na 19.19 ili novijem |

## 29.2 Pokretanje recovery-ja iz CR UI

1. **Main Menu → Recovery**
   Prikazuje se sadržaj taba **Copies**, dugme **Application** je aktivno.

2. Opciono izabrati prikaz:
   - Hijerarhijski prikaz — politike i pridružene kopije
   - Prikaz liste kopija

3. Ako želite da aktivirate dugmad **Sandbox**, **Alternate Recovery** i **Recovery Check** (podrazumevano neaktivna), izaberite kopiju.

   > Time se **onemogućava** dugme **Application**. Za ponovno aktiviranje kliknuti **Clear Selected** pored imena politike.

4. Za pokretanje oporavka kliknuti **Application**.
   Otvara se **Application Recovery** panel sa padajućom listom PPDM aplikacija.

5. Izabrati PPDM aplikativni host iz liste **Application Host**.

6. Iz liste imena produkcionih PPDM hostova otvoriti listu politika sa kopijama koje se mogu oporaviti.

7. Otvoriti listu kopija pridruženih svakoj politici.

## 29.3 Izbor kopija

Tri načina pokretanja oporavka na neoporavljenoj PPDM aplikaciji:

| Način | Ponašanje |
|---|---|
| **Jedna kopija** → **Apply** | Pokreće **full recovery job** (`recoverappPPDM`). Job kreira sandbox, zatim se izvršava pun oporavak. |
| **Više kopija** → **Apply** | Pokreće full recovery job (`recoverappPPDM`) koristeći najnoviju kopiju pridruženu politici, i pridružene **linked recovery** job-ove (`linked-recoverapp`) koristeći najnoviju dostupnu kopiju među izabranima. Linked job-ovi kreiraju odgovarajuće sandbox-ove i nastavljaju **posle** završetka full recovery-ja. |
| **Select Latest Copies** slider ulevo → **Apply** | Ako postoji više kopija pridruženih raznim politikama a želite samo najnovije. Linked job-ovi kreiraju odgovarajuće sandbox-ove i nastavljaju posle full recovery-ja. |

> **Ograničenje verzija kopija:** ako je ovo prvi izbor kopije uz PPDM verziju 19.19 ili noviju u vault-u, dostupne su **samo kopije verzije 19.19 ili novije**. Kopije kreirane pre CR verzije 19.19 nemaju copy version i **nisu dostupne za izbor**.

> **One-to-many ograničenje (CR 19.19):** kada izaberete kopiju na jednom vault DD sistemu, kopija na drugom vault DD sistemu je onemogućena. Razlog: obe kopije imaju storage unit-e sa istim DD izvorom ali različitim DD odredištem. Opcija **Select Latest Copies** je takođe onemogućena.

## 29.4 Šta se dešava tokom oporavka

Cyber Recovery:
1. Kreira produkciono DD Boost korisničko ime i lozinku na DD sistemu.
2. **Restartuje PPDM appliance.**
3. **< 20.3:** uzima VM snapshot PPDM appliance-a (koristi se za vraćanje posle oporavka).
   **≥ 20.3:** ne uzima snapshot.
4. Kreira recovery sandbox(-ove) i popunjava ih izabranom kopijom.

**Napomene o lozinkama (PPDM < 20.3):**
- Recovery procedura postavlja `root` i `admin` lozinke operativnog sistema na vrednosti unete u polje admin password pri dodavanju PPDM aplikacije u CR deployment.
- Ako procedura ne uspe da vrati `admin` ili `root` lozinku, CR prikazuje status job-a kao **Warning**. Opciono resetovati lozinku radi dodatne bezbednosti.

## 29.5 Praćenje

Pratiti job-ove i sandbox-ove:
- **Main Menu → Jobs** — tab **Running**, zatim **Completed**
- **Main Menu → Recovery → Recovery Sandboxes**

> U tabeli sandbox-ova red **ne prikazuje** child sandbox-ove pridružene PPDM sandbox-u. Panel **Details** daje informacije o child kopijama.

---

# 30. Post-recovery koraci

## 30.1 Pristup PPDM UI-u

1. Kliknuti **Launch App** za pristup PPDM UI-u u vault-u.
   Otvara se prozor **Welcome to PowerProtect Data Manager**.

2. > **VAŽNO:** za deployment verzije 19.5 i novije, PPDM oporavak **onemogućava sve servise**. Kada pristupite PPDM UI-u posle oporavka, prikazuje se alert koji ukazuje da servisi ne rade. **Ne klikati na alert da biste omogućili servise.**

## 30.2 Prijava

| Verzija | Kredencijali |
|---|---|
| **< 20.3** | `admin` / `root` sa lozinkom definisanom pri dodavanju aplikacije |
| **≥ 20.3** | Application username i password sa stranice **Post-Recovery Application Authentication**. Nije potrebno menjati kredencijale oporavljene PPDM aplikacije da odgovaraju produkciji. |
| **DM5500 / DM5510** | `admin` / `Abcd!2345` |

## 30.3 Restricted mode (PPDM strana)

Ako je pri deploymentu izabrano **After restore, keep the product in recovery mode**, PPDM aktivira **restricted mode**:

- Na vrhu PPDM UI-a prikazuje se baner koji označava da je sistem operativan, ali da se zakazani backup-i protection politika, event-based backup-i i manualni backup-i **ne mogu pokrenuti** dok se ne omogući **Return to full operational mode**. To obuhvata sve zakazane job-ove definisane protection politikama koji menjaju backup storage (kreiranje i brisanje backup-a, PPDM server DR job-ovi).
- **Sve operacije koje pišu na backup storage su onemogućene.**

Za povratak u pun operativni režim kliknuti **Return to full operational mode**.

> **Preporuka za vault:** u vault okruženju restricted mode je poželjan podrazumevani izbor. PPDM u vault-u služi **isključivo za oporavak** — ne treba da piše na backup storage niti da pokreće protection politike.

## 30.4 Validacija kada se IP/FQDN poklapa

Ako se IP adresa ili FQDN oporavljenog PPDM-a **poklapa** sa produkcionim PPDM sistemom, možete izvršiti centralizovani restore i oporavak aplikacije direktno, bez ručne intervencije.

## 30.5 Validacija kada se IP/FQDN razlikuje

Ako se IP adresa ili FQDN **razlikuje**, potrebno je verifikovati oporavak za SQL, Oracle i file system workload-e.

**Windows deployment:**

```
1. Otići u C:\Program Files\DPSAPPS\AgentService
2. Pokrenuti unregister.bat da se agent odjavi
3. Po potrebi obrisati ssl folder iz Agent Service foldera
4. Pokrenuti register.bat da se host ponovo registruje kod PPDM-a
5. Odobriti host u PPDM: Main Menu → Infrastructure → Application Agents
6. Pokrenuti ručni discovery agenta
```

> **Sa PPDM strane, isti postupak važi za zaštitu klijenata registrovanih kod primarnog sistema.** Pre bilo koje backup ili restore operacije: odjaviti klijenta sa primarnog sistema (`unregister.bat`), pa ga registrovati na oporavljeni sistem (`register.bat`). Ako je primarni sistem i dalje dostupan, ukloniti asset source iz PowerProtect Data Manager-a.

**Linux, SQL, Oracle i file system workload-i:** videti *PowerProtect Data Manager Administration and User Guide*, *PowerProtect Data Manager for Oracle RMAN Agent User Guide* i *PowerProtect Data Manager for Microsoft Application Agent SQL Server User Guide*.

## 30.6 Ponovni deployment engine-a

Pošto se protection engine, Search Engine i reporting engine **ne oporavljaju** automatski (videti 25.4), po potrebi ih deploy-ovati ručno.

**Ručni oporavak Search Engine-a (vCenter OVA deployment)** — ako ga PPDM nije oporavio automatski:

```bash
sudo ppdmadmin install --server-dr-only --component search
sudo ppdmadmin shell -c core

backupId=`grep /var/log/brs/serverdr/ -Rna -e 'Provided payload' --include=*.log -A 60 \
  | grep '"name" : "PPDM",' -A 7 | grep '"id" :' | awk '{print$4}' \
  | sed "s/\"//g" | sed 's/,//g'`

/usr/local/brs/puppet/scripts/mrestore.py -c SearchCluster -b <backupId>
```

> Opcija `<backupId>` određuje DR backup iz kojeg se vrši restore. Ako se ne navede, koristi se najnoviji DR backup.
> Praćenje: PPDM UI → **Jobs → System Jobs**, job sa opisom **`Restoring backup Search Node`**.

**Linux deployment — Search Engine i reporting engine** se **ne oporavljaju automatski**:

```bash
# Search Engine
sudo ppdmadmin install --server-dr-only --component search
sudo ppdmadmin shell -c core
backupId=`...`   # ista komanda kao gore
/usr/local/brs/puppet/scripts/mrestore.py -c SearchCluster "${backupId}"

# Reporting engine
sudo ppdmadmin install --server-dr-only --component reporting
sudo ppdmadmin shell -c core
backupId=`...`
/usr/local/brs/puppet/scripts/mrestore.py -c REPORTING "${backupId}"
```

> Posle svakog koraka sačekati završetak. **Svaki korak može trajati do 5 minuta.**

## 30.7 Poznato ponašanje posle oporavka

| Ponašanje | Objašnjenje |
|---|---|
| **DR backup job prikazan kao critical failed** | Ako oporavite DR backup, posle oporavka DR backup job se prikazuje kao kritično neuspeo job. **Ignorisati taj status.** |
| **Vremenska zona** | Vremenska zona PPDM instance se postavlja na vremensku zonu backup-a |
| **Preloaded nalozi (vCenter OVA)** | Svi preloaded nalozi se resetuju na podrazumevane lozinke. **UI administrator nalog je izuzetak** i zadržava svoju lozinku. Promeniti lozinke svih preloaded naloga što pre. |
| **Copy discovery** | Ako je konfigurisana VMware VM, Kubernetes, PowerStore Block Volume, PowerMax Block Volume ili NAS protection politika, automatski se izvršava copy discovery operacija. Usklađuje backup-e nastale između početka i završetka oporavka. Može trajati minutima ili satima. Loguje se kao **Post Restore Copy Discovery** grupa job-ova u **System Jobs**. |
| **Vraćanje sa replike** | Ako se vraća sa replike, originalni replikacioni target se automatski konfiguriše kao novi primarni backup target. Originalni primarni izvorni DD se uklanja iz server DR konfiguracije, jer se pretpostavlja da je nedostupan. **Replikacija se mora ručno ponovo omogućiti**, koristeći protection storage sistem različit od novog primarnog. |

---

# 31. Recovery Check i čišćenje

## 31.1 Recovery Check

Recovery check potvrđuje da se kopija može oporaviti.

**Tokom provere:** stanje backup kopije prikazuje se kao **In-progress**.

**Rezultat:**

| Status | Značenje |
|---|---|
| **Recoverable** | Provera uspešno završena — kopija se može oporaviti |
| **Failed** | Provera nije uspela |

Ako provera ne uspe, alerti na dashboard-u i email poruka obaveštavaju o stanju:

| Nivo alerta | Značenje |
|---|---|
| **Warning** | Backup kopija je **delimično** oporavljiva |
| **Critical** | Backup kopija je **neoporavljiva** |

Postupak zakazivanja i pokretanja on-demand provere je isti kao za NetWorker — videti Deo IV, poglavlje 22.

## 31.2 Čišćenje sandbox-ova

**Main Menu → Recovery → Recovery Sandboxes** → izabrati sandbox → **Cleanup**

**Ponašanje po verziji:**

| Verzija | Ponašanje |
|---|---|
| **< 20.3** | Softver koristi jedan od cleanup job-ova da **vrati sistem** (VM rollback), pa zatim briše preostale sandbox-ove |
| **≥ 20.3** | Softver briše sandbox-ove **bez vraćanja** PPDM appliance-a na snapshot |

**Statusi posle brisanja sandbox-a:**

| Situacija | Status |
|---|---|
| Sandbox uspešno završenog punog oporavka je obrisan | **Recoverable** |
| Cleanup sandbox-a nije uspeo | **Unknown** |

> **Za PPDM < 20.3:** posle cleanup-a sačekati **približno 10 minuta** da se završi PPDM VM rollback i da se PPDM servisi pokrenu.

> Kada obrišete PPDM sandbox koji ima pridružene child sandbox-ove, i ti child sandbox-ovi se brišu.

Rezultat: sistem je spreman za sledeću recovery operaciju.

## 31.3 Napomena o brisanju aplikacije

> Ako je aplikacija povezana sa recovery sandbox-om, **ne može se obrisati**. Pokušaj brisanja daje poruku o grešci. Prvo očistiti sandbox.

## 31.4 Redosled uklanjanja DD storage-a iz CR-a

Ako se PPDM integracija uklanja iz deploymenta, redosled je obavezan:

1. Prvo obrisati kopije — brisanjem svih sandbox-ova pridruženih kopiji.
2. Zatim obrisati sve politike — brisanjem svih kopija pridruženih svakoj politici.
3. Na kraju obrisati vault storage — brisanjem svih politika pridruženih tom storage-u.

---

# Dodatak A: Checklist PPDM integracije

## A. Verzija i arhitektura

| # | Stavka | Vrednost / potvrda | Status |
|---|---|---|---|
| A1 | PPDM verzija na produkciji | | |
| A2 | PPDM verzija u vault-u (mora biti ista) | | |
| A3 | Grana ponašanja: **< 20.3** ili **≥ 20.3** | | |
| A4 | Način deploymenta na produkciji (OVA / install package / DM55xx) | | |
| A5 | Način deploymenta u vault-u — **mora biti OVA na vCenter** | | |
| A6 | Okruženje (on-prem / AWS / Azure) — **GCP nije podržan** | | |
| A7 | Linked recovery u obimu (zahteva 19.19+) | | |

## B. Produkciona strana

| # | Stavka | Vrednost / potvrda | Status |
|---|---|---|---|
| B1 | DD Boost korisnik kreiran, **`role none`** | | |
| B2 | DD Boost korisnik dodeljen (`ddboost user assign`) | | |
| B3 | UID-ovi policy MTree-ova zabeleženi | | |
| B4 | Server DR omogućen (automatski ili ručno) | | |
| B5 | Server DR target je DD koji se replicira u vault | | |
| B6 | Storage unit `SysDR_<hostname>` postoji | | |
| B7 | **ifGroups omogućeni i ispravno konfigurisani na DD** | | |
| B8 | Interval server DR backup-a (1–24 h) | | |
| B9 | Retencija server DR backup-a (2–30 dana) | | |
| B10 | Retencija usklađena sa učestalošću CR Sync-a | | |
| B11 | Server DR replikacija (da/ne) i target | | |
| B12 | Security officer kredencijali (ako compliance mode RL) | | |
| B13 | Sistemski job-ovi server DR konfiguracije uspešni | | |
| B14 | Protection politike konfigurisane i uspešne | | |
| B15 | Ime PPDM VM-a bez zabranjenih znakova (< 20.3) | | |
| B16 | Evidencija DR podataka popunjena (26.7) | | |
| B17 | DD Boost UID server DR korisnika zabeležen | | |
| B18 | Port group podešavanja zabeležena (vSphere) | | |
| B19 | Predefinisana administratorska lozinka poznata | | |

## C. Vault strana — PPDM instanca

| # | Stavka | Vrednost / potvrda | Status |
|---|---|---|---|
| C1 | vCenter u vault-u dostupan | | |
| C2 | Datastore ima kapacitet za 7 diskova (100+500+10+10+5+5+5 GB) | | |
| C3 | 10 CPU jezgara (preporučeno 14) | | |
| C4 | 36 GB RAM (40 GB sa Cloud DR) | | |
| C5 | Swap na SSD datastore-u | | |
| C6 | Disk format: **Thick provision lazy zeroed** | | |
| C7 | Ispravna mreža izabrana (ne podrazumevano prva) | | |
| C8 | IP, gateway, netmask, DNS, FQDN konfigurisani | | |
| C9 | **FQDN = hostname** | | |
| C10 | FQDN odgovara DNS imenu u vCenter UI (< 20.3) | | |
| C11 | Nema postojećih VM snapshot-ova (< 20.3) | | |
| C12 | `PasswordAuthentication yes` u `sshd_config` (< 20.3) | | |
| C13 | Podrazumevani kredencijali **nisu** menjani (< 20.3) | | |
| C14 | Search Engine i reporting engine node-ovi obrisani iz vCenter-a | | |
| C15 | Novi Search Engine i reporting engine kreirani | | |
| C16 | Restricted mode odabran pri deploymentu (preporuka) | | |

## D. Vault DD i NFS

| # | Stavka | Vrednost / potvrda | Status |
|---|---|---|---|
| D1 | DDOS verzija 7.13 ili 8.x | | |
| D2 | **NFS v3 omogućen ILI DD Boost se koristi za ServerDR MTree** | | |
| D3 | `nfs4-idmap-out-numeric` = `always` (ako NFSv4) | | |
| D4 | `force-minimum-root-squash-default` = `disabled` | | |
| D5 | UID-ovi produkcionih policy MTree-ova kreirani u vault-u | | |
| D6 | UID konvencija primenjena (501–599 / 600–700) | | |

## E. Cyber Recovery konfiguracija

| # | Stavka | Vrednost / potvrda | Status |
|---|---|---|---|
| E1 | vCenter asset dodat u CR (samo < 20.3, on-prem) | | |
| E2 | Data replication kontekst(i) kreirani | | |
| E3 | **ServerDR replication kontekst kreiran** | | |
| E4 | Inicijalna replikacija završena za sve kontekste | | |
| E5 | Port 3009 zatvoren posle dodavanja parova | | |
| E6 | Aplikacija dodata: Application Type = **PPDM** | | |
| E7 | Tag sa produkcionim DD Boost username-om dodat | | |
| E8 | Host Authentication popunjena (< 20.3) | | |
| E9 | Vault Application Authentication popunjena (≥ 20.3) | | |
| E10 | Sertifikat verifikovan i prihvaćen (≥ 20.3) | | |
| E11 | Post-Recovery Application Authentication popunjena (≥ 20.3) | | |
| E12 | Politika tipa **PPDM** kreirana | | |
| E13 | **Politika ima najmanje dva MTree-a** | | |
| E14 | ServerDR Context izabran u politici | | |
| E15 | Sync zakazan **posle** završetka produkcionih backup-a | | |
| E16 | Copy i Lock zakazani | | |
| E17 | Prva Sync / Copy / Lock operacija uspešna | | |

## F. Validacija

| # | Stavka | Status |
|---|---|---|
| F1 | Test recovery pokrenut iz CR UI | |
| F2 | Job `recoverappPPDM` uspešno završen | |
| F3 | Linked recovery job-ovi uspešni (ako se koriste) | |
| F4 | Recovery sandbox(-ovi) kreirani | |
| F5 | **Launch App** aktivan, PPDM UI dostupan | |
| F6 | Alert o onemogućenim servisima **nije** kliknut | |
| F7 | Prijava sa odgovarajućim kredencijalima uspešna | |
| F8 | Test restore workload-a izvršen | |
| F9 | Agenti ponovo registrovani (ako se IP/FQDN razlikuje) | |
| F10 | Status kopije = **Recoverable** | |
| F11 | Sandbox očišćen (**Cleanup**) | |
| F12 | VM rollback završen, ~10 min sačekano (< 20.3) | |
| F13 | Recovery Check zakazan | |
| F14 | Rezultati zabeleženi u zapisniku o primopredaji | |

---

# Dodatak B: Otvorena pitanja

| # | Pitanje | Zašto je bitno |
|---|---|---|
| 1 | **PPDM 20.3 dokumentacija.** Priloženi vodiči su za 19.22. CR 20.3 se bitno drugačije ponaša prema PPDM 20.3+, ali PPDM-strana dokumentacija za tu verziju nedostaje. | Ako kupac planira PPDM 20.3+, poglavlja 26 i 27 treba proveriti prema *PPDM 20.3 Administrator Guide* i *Deployment Guide*. Trenutni sadržaj za tu granu potiče isključivo iz CR dokumentacije. |
| 2 | **Search Engine i reporting engine u vault-u.** CR dokumentacija kaže da se **ne oporavljaju**. PPDM dokumentacija propisuje da se pre oporavka obrišu iz vCenter-a i kreiraju novi, jer ih proces oporavka prepisuje. Dve izjave se ne poklapaju očigledno. | Treba potvrditi koji je stvarni očekivani postupak u vault scenariju: da li se u vault-u uopšte kreiraju prazni Search/reporting node-ovi pre oporavka, ili se korak preskače. **Preporuka: potvrditi kod Dell-a ili testirati.** |
| 3 | **PPDM Security Configuration Guide.** Dokumentacija upućuje na njega za spisak podrazumevanih lozinki preloaded naloga koje se resetuju posle oporavka (vCenter OVA). | Potrebno za post-recovery hardening korak u vault-u. |
| 4 | **DD OS Administration Guide** — poglavlje o ifGroups | ifGroups su tvrd preduslov za server DR backup, a konfiguracija nije opisana ni u jednom priloženom fajlu. |
| 5 | **Quick recovery.** PPDM podržava quick recovery preko sekundarnog PPDM sistema. CR dokumentacija ga ne pominje. | Ako kupac koristi quick recovery u produkciji, treba razjasniti odnos prema PPDM instanci u vault-u (da li vault PPDM sme biti secondary system). |
| 6 | **Više vault DD sistema.** CR dokumentacija upućuje na dokument *Replication PowerProtect Data Manager Backups to the Cyber Recovery Vault — A User Journey* na Info Hub-u za konfiguraciju sa više DD sistema u vault-u i istovremenim PPDM oporavcima. | Ako je u obimu, taj dokument treba pribaviti. |
| 7 | **Provisioning diskova.** PPDM Deployment Guide traži `Thick provision lazy zeroed`, dok CR OVA preporučuje thin provisioning. | Različiti proizvodi, ali vredi potvrditi da nema izuzetka za PPDM u vault-u, zbog planiranja kapaciteta datastore-a (~635 GB thick po PPDM instanci). |

---

*Kraj Dela V. Sledeći deo: **Deo VI — Validacija, predaja i održavanje** (poglavlja 32–37).*
