# Poglavlje 10: Bezbednost, backup i notifikacije

> **Cilj poglavlja:** Kada završiš ovo poglavlje, znaćeš da podesiš korisnike, uloge i dozvole po principu najmanjih privilegija i da bezbedno uključiš ugrađeni firewall. Naučićeš i da napraviš pouzdan sistem za backup sa **Proxmox Backup Server**-om i da podesiš notifikacije, da za probleme saznaš pre korisnika. Na kraju je **završna kontrolna lista za produkciju** koja objedinjuje ceo kurs.

---

## 10.1 Korisnici, grupe i dozvole

### Realm-ovi (izvori korisnika)

| Realm | Opis | Kada koristiti |
|---|---|---|
| **Linux PAM** (`@pam`) | Linux korisnici hosta | Samo `root@pam`, za hitne slučajeve |
| **Proxmox VE** (`@pve`) | Korisnici koje vodi sam Proxmox, a važe na celom klasteru | Mala okruženja, servisni nalozi |
| **LDAP** | Spoljni LDAP direktorijum | Postojeći LDAP |
| **Active Directory** | Microsoft AD | Firme sa AD-om |
| **OpenID Connect** | Prijava preko SSO provajdera (Keycloak, Entra ID, Authentik...) | **Centralizovan SSO sa 2FA** |

**Datacenter → Permissions → Realms → Add**. Za LDAP i AD možeš da uključiš **sinhronizaciju** korisnika i grupa (**Sync**).

### Kako rade dozvole

Dozvola je kombinacija tri stvari:

```
KO (korisnik / grupa / API token)  +  ŠTA (uloga)  +  GDE (putanja)
```

**Putanje** (paths):

| Putanja | Odnosi se na |
|---|---|
| `/` | Ceo data centar |
| `/vms/100` | VM ili CT sa ID-om 100 |
| `/storage/vm-pool` | Jedan storage |
| `/pool/tim-web` | Resource pool (grupa gostiju i storage-a) |
| `/nodes/pve1` | Jedan čvor |
| `/sdn/zones/prod` | SDN zona |
| `/mapping/pci/...` | Mapirani uređaji |

Opcija **Propagate** znači da dozvola važi i za sve ispod te putanje.

**Ugrađene uloge** (izbor):

| Uloga | Šta sme |
|---|---|
| **Administrator** | Sve |
| **PVEAdmin** | Skoro sve, osim nekih sistemskih podešavanja |
| **PVEVMAdmin** | Potpuno upravljanje VM-ovima |
| **PVEVMUser** | Korišćenje VM-ova: konzola, start/stop, backup |
| **PVEAuditor** | **Samo čitanje** |
| **PVEDatastoreUser** | Korišćenje storage-a (pravljenje diskova, backup) |
| **NoAccess** | Eksplicitna zabrana |

Možeš da napraviš i **sopstvene uloge** sa tačno određenim privilegijama (**Permissions → Roles → Create**).

### Primer: tim administratora i tim za web

```bash
# Grupa administratora
pveum group add admins --comment "Sistem administratori"
pveum acl modify / --groups admins --roles Administrator

# Lični nalog
pveum user add marko@pve --groups admins --email marko@firma.rs
pveum passwd marko@pve

# Tim za web: vidi i upravlja samo gostima u svom pool-u
pveum pool add tim-web
pveum group add web-tim
pveum acl modify /pool/tim-web --groups web-tim --roles PVEVMAdmin
```

### Resource pool-ovi

**Datacenter → Permissions → Pools**. Pool grupiše goste i storage-e. Dozvole daješ na pool, pa novi gost dodat u pool automatski dobija iste dozvole. Ovo je osnova za okruženja gde **više timova deli klaster**.

### API tokeni

Za automatizaciju (Terraform, Ansible, monitoring, skripte) **nikad ne koristi lozinku stvarnog korisnika**, nego **API token**:

```bash
# Servisni korisnik i token za monitoring
pveum user add monitor@pve --comment "Monitoring servis"
pveum user token add monitor@pve zabbix --privsep 1
pveum acl modify / --tokens 'monitor@pve!zabbix' --roles PVEAuditor
```

- **Privilege separation** (`--privsep 1`): token ima **samo dozvole koje su njemu dodeljene**, a ne sva prava korisnika,
- vrednost tokena (*secret*) prikazuje se **samo jednom**. Sačuvaj je u sistemu za tajne (Vault, password manager),
- tokenu možeš da postaviš **datum isteka**.

### Najbolje prakse za produkciju

- **lični nalozi** za svakog administratora, a `root@pam` samo za hitne slučajeve (*break glass*), sa jakom lozinkom i 2FA,
- **2FA za sve** sa administratorskim pravima (Poglavlje 3), ili SSO (OpenID Connect) koji zahteva 2FA,
- **najmanje privilegije**: svako dobija samo ono što mu treba,
- dozvole se daju **grupama**, ne pojedincima,
- **API tokeni sa privsep-om** za automatizaciju, svaki sa svojom namenom,
- redovno pregledaj **API Access Log** (novo u 9.3) i listu korisnika i tokena, i ukloni one koji nisu potrebni.

---

## 10.2 Firewall

Proxmox VE ima **ugrađen firewall** zasnovan na pravilima koja se čuvaju u `/etc/pve/firewall/` (isto na celom klasteru).

### Nivoi

| Nivo | Fajl | Šta štiti |
|---|---|---|
| **Datacenter** | `/etc/pve/firewall/cluster.fw` | Glavni prekidač, pravila za sve čvorove, alias-i, IPSet-ovi, security grupe |
| **Čvor (Host)** | `/etc/pve/nodes/<čvor>/host.fw` | Saobraćaj ka samom hostu |
| **Gost (VM/CT)** | `/etc/pve/firewall/<VMID>.fw` | Saobraćaj ka i od gosta |

> **Važno:** Firewall je podrazumevano **isključen** na nivou data centra. Kada ga uključiš, podrazumevana politika za dolazni saobraćaj je **DROP**. Proxmox automatski dozvoljava pristup web interfejsu i SSH-u iz **lokalne mreže** klastera, ali ne i iz drugih mreža.

### Gradivni elementi

| Element | Opis | Primer |
|---|---|---|
| **Alias** | Ime za IP adresu ili mrežu | `admin_vpn = 10.8.0.0/24` |
| **IPSet** | Skup adresa | `management` sa svim admin mrežama |
| **Security Group** | Skup pravila koji se dodeljuje gostima | `web-server`: dozvoli 80, 443 |
| **Macro** | Gotova pravila za poznate servise | `SSH`, `HTTPS`, `DNS`, `Ceph` |

> **IPSet `management`**: ako napraviš IPSet sa ovim imenom, Proxmox iz tih adresa dozvoljava pristup upravljačkim portovima (8006, 22...). Pre uključivanja firewall-a proveri u dokumentaciji tačno ponašanje.

### Bezbedno uključivanje: korak po korak

1. **Datacenter → Firewall → Alias**: dodaj `admin_net` (npr. `10.0.10.0/24`) i `admin_vpn`.
2. **Datacenter → Firewall → IPSet**: napravi `management` i u njega dodaj `admin_net` i `admin_vpn`.
3. **Datacenter → Firewall → Add** (pravila), na primer:
   - `IN ACCEPT -source +management -p tcp -dport 8006`
   - `IN ACCEPT -source +management -p tcp -dport 22`
   - `IN ACCEPT -source admin_net -p icmp`
4. **Imaj pristup konzoli** (IPMI/iDRAC/iLO) za slučaj da se zaključaš.
5. **Datacenter → Firewall → Options → Firewall: Yes**.
6. Proveri da i dalje imaš pristup web interfejsu i SSH-u **iz nove sesije**.

Primer `cluster.fw`:

```
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT

[ALIASES]
admin_net 10.0.10.0/24
admin_vpn 10.8.0.0/24

[IPSET management]
admin_net
admin_vpn

[RULES]
IN ACCEPT -source +management -p tcp -dport 8006 -log nolog
IN ACCEPT -source +management -p tcp -dport 22 -log nolog
IN ACCEPT -source admin_net -p icmp

[group web-server]
IN HTTP(ACCEPT)
IN HTTPS(ACCEPT)
```

### Firewall za goste

1. **VM → Hardware → Network Device**: uključi **Firewall** na kartici.
2. **VM → Firewall → Options**: **Firewall: Yes**, podesi **Input Policy** (npr. DROP).
3. Dodaj pravila ili **security grupu** (`web-server`).
4. Uključi **IP filter** i **MAC filter**, da gost ne bi mogao da lažira IP ili MAC adresu (*spoofing*).

Komande za proveru:

```bash
pve-firewall status
pve-firewall compile     # prikaz generisanih pravila
```

> **Napomena:** Proxmox ima i noviji **nftables** backend za firewall (`proxmox-firewall`). Pre upotrebe u produkciji proveri u dokumentaciji za 9.3 da li je podrazumevan i koje su razlike.

---

## 10.3 Backup: osnove

### Pravilo 3-2-1

- **3** kopije podataka (original + 2 backup-a),
- na **2** različita medija ili sistema,
- **1** kopija van lokacije (*offsite*), po mogućstvu i **nepromenjiva** (*immutable*) ili odvojena od mreže, zbog ransomware-a.

### vzdump: ugrađeni alat

| Režim | Kako radi | Zastoj gosta |
|---|---|---|
| **Snapshot** | VM: QEMU pravi konzistentan snapshot dok VM radi. CT: snapshot storage-a | **Nema** |
| **Suspend** | Kratka pauza gosta | Kratak |
| **Stop** | Gašenje gosta za vreme backup-a | Najduži, ali najveća konzistentnost |

Za VM-ove sa **guest agentom** Proxmox pre snapshot backup-a radi **fs-freeze**, pa su fajl sistemi konzistentni.

```bash
vzdump 100 --storage nas-backup --mode snapshot --compress zstd
```

### Backup poslovi (jobs)

**Datacenter → Backup → Add**:

| Polje | Preporuka |
|---|---|
| **Node** | `All` |
| **Storage** | PBS (preporučeno) ili NFS |
| **Schedule** | Npr. `21:00` ili `mon..fri 22:30` |
| **Selection mode** | **All** (svi gosti, i novi), sa izuzecima, ili **Pool based** |
| **Mode** | Snapshot |
| **Compression** | ZSTD (za fajl backup) |
| **Retention** | Npr. `keep-daily=7, keep-weekly=4, keep-monthly=6` |
| **Notification** | Uvek, ili barem pri grešci |
| **Note template** | `{{guestname}} - {{node}}`, za lakše snalaženje |

> **Produkcija:** Koristi **Selection mode: All**, sa izuzecima. Kada neko napravi novu VM i zaboravi da je doda u backup, ona je i dalje pokrivena.

**Fleecing**: ako je backup storage spor, upisi gosta mogu da se uspore za vreme backup-a. Opcija **fleecing** (Advanced) privremeno čuva te blokove na brzom lokalnom storage-u.

---

## 10.4 Proxmox Backup Server (PBS)

**Proxmox Backup Server** je poseban proizvod (takođe otvorenog koda) napravljen za backup Proxmox gostiju. U produkciji je **preporučeno rešenje**.

### Zašto PBS, a ne NFS?

| Mogućnost | NFS + vzdump | **PBS** |
|---|---|---|
| Inkrementalni backup | Ne (uvek pun) | **Da**, uz *dirty bitmap* za VM-ove koji rade: šalju se samo promenjeni blokovi |
| Deduplikacija | Ne | **Da**, isti blokovi se čuvaju jednom |
| Enkripcija na klijentu | Ne | **Da** (AES-256-GCM). Server ne vidi podatke |
| Provera integriteta (verify) | Ne | **Da**, zakazana |
| Vraćanje pojedinačnih fajlova | Teško | **Da**, iz GUI-ja |
| Live restore | Ne | **Da**, VM se pokreće dok se još vraća |
| Sinhronizacija na drugu lokaciju | Ručno | **Da**, *sync jobs* na udaljeni PBS |
| Zaštita od brisanja | Ne | **Da**, *protected* backup-ovi, odvojene dozvole |

### Arhitektura

```
  Proxmox VE klaster              Lokacija A                   Lokacija B (offsite)
 +------------------+         +-----------------+   sync     +-----------------+
 | pve1  pve2  pve3 | ──────→ |   PBS (lokalni) | ─────────→ |  PBS (udaljeni) |
 +------------------+ backup  +-----------------+   (pull)   +-----------------+
```

- PBS treba da bude na **posebnom hardveru**, **ne kao VM u istom klasteru**, jer bi pri otkazu klastera izgubio i backup,
- udaljeni PBS **povlači** (*pull*) podatke, pa kompromitovan glavni sistem ne može da obriše offsite kopije.

### Povezivanje PBS-a sa Proxmox VE

Na PBS-u napravi **datastore**, korisnika (ili API token) sa ulogom **DatastoreBackup** i pronađi **fingerprint** sertifikata (PBS **Dashboard → Show Fingerprint**).

Na Proxmox VE: **Datacenter → Storage → Add → Proxmox Backup Server**:

| Polje | Primer |
|---|---|
| ID | `pbs01` |
| Server | `pbs01.firma.rs` |
| Datastore | `vm-backup` |
| Username | `pve-backup@pbs` ili API token |
| Fingerprint | (sa PBS-a) |
| **Encryption** | **Upload/generate key**. Proxmox šifruje backup pre slanja |

> **Ključ za enkripciju:** Bez njega **ne možeš da vratiš backup**. Sačuvaj ga (i *paper key* verziju) **van klastera**, na najmanje dva bezbedna mesta. Izgubljen ključ znači izgubljene backup-ove.

### Održavanje na PBS-u

Na PBS-u zakaži:

- **Prune**: brisanje starih backup-ova po pravilima čuvanja,
- **Garbage Collection**: oslobađanje prostora posle prune-a,
- **Verify**: provera integriteta backup-ova,
- **Sync**: kopiranje na udaljeni PBS.

---

## 10.5 Vraćanje (restore)

> **Backup koji nisi probao da vratiš ne postoji.**

### Vraćanje celog gosta

**Storage → Backups** (ili **VM → Backup**) → izaberi backup → **Restore**:

- vrati na **novi VMID** za test, ili preko postojećeg (oprezno!),
- za PBS uključi **Live restore**, pa se VM pokreće odmah, a podaci se prenose u pozadini.

```bash
qmrestore pbs01:backup/vm/100/2026-11-25T21:00:00Z 9100 --storage local-lvm
pct restore 9300 pbs01:backup/ct/300/2026-11-25T21:00:00Z --storage local-lvm
```

### Vraćanje pojedinačnih fajlova (PBS)

**Storage pbs01 → Backups** → izaberi backup → **File Restore** → pronađi fajl → **Download**.

### Test vraćanja: redovna procedura

- jednom mesečno (ili češće) vrati **nasumično izabranu VM** na novi VMID, u izolovanoj mreži,
- proveri da se pokreće i da su **podaci ispravni** (npr. baza radi),
- zabeleži **koliko je trajalo**. To je tvoj stvarni **RTO** (vreme oporavka),
- obriši test VM.

---

## 10.6 Notifikacije

**Datacenter → Notifications**. Proxmox šalje obaveštenja o backup-u, replikaciji, ažuriranjima, HA događajima (fencing), greškama na diskovima (ZFS, SMART) i drugim događajima.

### Targets: kuda se šalje

| Target | Opis |
|---|---|
| **Sendmail** | Lokalni mail sistem hosta (podrazumevano, ka e-mail-u `root@pam`) |
| **SMTP** | Direktno preko SMTP servera (sa autentifikacijom i TLS-om). **Preporučeno** |
| **Gotify** | Push notifikacije preko Gotify servera |
| **Webhook** | HTTP zahtev ka bilo kom servisu: Slack, Microsoft Teams, Mattermost, ntfy, sistem za tikete... |

### Matchers: pravila

Matcher određuje **koja obaveštenja idu na koji target**, po polju (tip događaja, čvor, backup posao...), **ozbiljnosti** (info, notice, warning, error) i **vremenu** (kalendar).

**Novo u 9.3:** pravila mogu da se **ugnježdavaju** (AND/OR uslovi unutar drugih uslova) i uređuju **prevlačenjem** (*drag-and-drop*), pa su složena pravila mnogo preglednija.

### Primer rasporeda za produkciju

| Matcher | Uslov | Target |
|---|---|---|
| `kriticno` | ozbiljnost `error` (bilo koji izvor), ili fencing | Webhook ka timskom chat-u + SMTP dežurnom |
| `backup` | tip = backup, ozbiljnost `warning` ili `error` | SMTP ka timu |
| `info` | ostalo | SMTP ka zajedničkom sandučetu |

Posle podešavanja klikni **Test** na svakom target-u i proveri da poruka stvarno stiže.

> **Produkcija:** Notifikacije o **uspešnom** backup-u lako se ignorišu. Važnije je da stigne obaveštenje o **neuspelom**. Uz to spoljni monitoring treba da javi ako backup **uopšte nije** pokrenut.

---

## 10.7 Česte greške u produkciji

| Greška | Posledica | Kako izbeći |
|---|---|---|
| Svi rade kao `root@pam` | Nema traga ko je šta uradio, a ukradena lozinka znači pristup svemu | Lični nalozi, grupe, 2FA |
| API skripte sa lozinkom administratora | Curenje lozinke daje potpunu kontrolu | API tokeni sa privsep-om i minimalnim pravima |
| Firewall uključen bez pravila za administratore | Zaključan si van klastera | IPSet `management`, pravila **pre** uključivanja, pristup konzoli |
| Backup na istom klasteru ili storage-u | Kvar ili ransomware briše i podatke i backup | PBS na posebnom hardveru + offsite sync |
| PBS kao VM u istom klasteru | Pri otkazu klastera nema backup-a | Poseban fizički PBS |
| Izgubljen ključ za enkripciju | Backup-ovi su neupotrebljivi | Ključ na dva bezbedna mesta van klastera |
| Restore nikad testiran | Backup ne radi kada zatreba | Mesečni test vraćanja |
| Backup posao sa ručnim izborom VM-ova | Nove VM-ove niko ne čuva | Selection mode **All** |
| Notifikacije idu na nečitano sanduče | Neuspeli backup-ovi prolaze neprimećeno | Webhook ka chat-u, posebna pravila za greške |

---

## 10.8 Završna kontrolna lista za produkciju

Ovo je rezime celog kursa. Pre nego što produkcioni klaster pustiš u rad, proveri:

### Hardver i instalacija (Poglavlje 2)
- [ ] Serverski hardver sa ECC RAM-om, redundantnim napajanjem i IPMI/iDRAC/iLO
- [ ] Sistemski diskovi u ogledalu, a diskovi za goste enterprise SSD/NVMe sa PLP-om
- [ ] Firmware ažuriran, **Enterprise** repozitorijum, ista verzija na svim čvorovima

### Host (Poglavlje 3)
- [ ] Pravi sertifikat (ACME ili interni CA)
- [ ] SSH samo ključem, `PermitRootLogin prohibit-password`
- [ ] Na hostu nema dodatnih servisa
- [ ] Postupak ažuriranja čvor po čvor je dokumentovan
- [ ] Metric Server i spoljni monitoring podešeni
- [ ] Backup konfiguracije hosta

### Mreža (Poglavlje 4)
- [ ] Odvojene mreže: upravljanje, corosync (2 linka), storage, migracija, gosti
- [ ] Bond (LACP) testiran izvlačenjem kabla
- [ ] MTU proveren na celom putu
- [ ] Imena mrežnih kartica zaključana (pinning)

### Storage (Poglavlje 5)
- [ ] Nadzor popunjenosti (thin pool, ZFS, Ceph), upozorenje na 70–80%
- [ ] ZFS: HBA režim, ograničen ARC, scrub, isprobana zamena diska
- [ ] SAN: multipath, nadzor putanja (Datacenter → SAN)

### Gosti (Poglavlja 6 i 7)
- [ ] CPU tip kompatibilan sa svim čvorovima
- [ ] Guest agent u svim VM-ovima, VirtIO drajveri
- [ ] Neprivilegovani kontejneri, Docker u VM-u
- [ ] Protection, Start at boot, Notes, tagovi

### Klaster i HA (Poglavlja 8 i 9)
- [ ] Najmanje 3 glasa (ili QDevice)
- [ ] Kapacitet po pravilu N+1
- [ ] HA pravila za redundantne parove (**odvojeno**)
- [ ] Failover **testiran**
- [ ] Ceph: size 3 / min_size 2, posebna mreža, `noout` postupak dokumentovan
- [ ] RPO za servise sa replikacijom dogovoren i dokumentovan

### Bezbednost, backup i notifikacije (Poglavlje 10)
- [ ] Lični nalozi, grupe, najmanje privilegije, 2FA za sve administratore
- [ ] `root@pam` samo za hitne slučajeve
- [ ] API tokeni sa privsep-om
- [ ] Firewall uključen, pristup upravljanju samo iz mreže za upravljanje
- [ ] PBS na posebnom hardveru, enkripcija, ključ sačuvan van klastera
- [ ] Offsite kopija (sync na udaljeni PBS)
- [ ] Mesečni test vraćanja, sa izmerenim RTO-om
- [ ] Notifikacije podešene i testirane, greške idu tamo gde ih neko čita

---

## 10.9 Vežba

1. Napravi grupu `admins` sa ulogom Administrator i lični nalog `ime@pve` u njoj. Uključi mu **TOTP 2FA** i prijavi se njime.
2. Napravi pool `tim-web`, grupu `web-tim` sa ulogom **PVEVMUser** na pool-u i korisnika u njoj. Dodaj jednu VM u pool, prijavi se kao taj korisnik i proveri šta vidi.
3. Napravi **API token** `monitor@pve!test` sa privsep-om i ulogom **PVEAuditor**. Testiraj ga:
   ```bash
   curl -k -H "Authorization: PVEAPIToken=monitor@pve!test=<SECRET>" \
     https://IP:8006/api2/json/nodes
   ```
4. Podesi alias-e, IPSet `management` i pravila, pa **bezbedno uključi firewall** na data centru (odeljak 10.2). Proveri pristup iz nove sesije.
5. Napravi security grupu `web-server` (HTTP, HTTPS) i dodeli je VM-u sa nginx-om. Proveri da li drugi portovi stvarno ne prolaze.
6. *(Ako imaš resurse)* Instaliraj **Proxmox Backup Server** u posebnu VM **van klastera** (ili na poseban računar), napravi datastore i poveži ga sa enkripcijom.
7. Napravi **backup posao** za sve goste u 21:00, sa retention-om `keep-daily=7, keep-weekly=4`.
8. **Vrati** jednu VM na novi VMID i proveri da radi. Vrati i **jedan fajl** preko File Restore-a (PBS).
9. Podesi **SMTP** ili **webhook** target i matcher za greške. Klikni **Test** i proveri da je poruka stigla.

---

## 10.10 Rezime

- Dozvole = **ko + uloga + putanja**. Daj ih **grupama**, po principu najmanjih privilegija.
- Koristi **lične naloge, 2FA ili SSO**, a `root@pam` samo za hitne slučajeve.
- Za automatizaciju koristi **API tokene sa privsep-om**.
- **Firewall** ima nivo data centra, čvora i gosta. Pre uključivanja obezbedi pristup za administratore (IPSet `management`).
- **Backup po pravilu 3-2-1**, a **PBS** donosi inkrementalni backup, deduplikaciju, enkripciju, verifikaciju i offsite sync.
- **Ključ za enkripciju** čuvaj van klastera. **Testiraj restore** redovno.
- **Notifikacije** (SMTP, webhook, Gotify) sa pravilima po ozbiljnosti. U 9.3 pravila mogu da se ugnježdavaju.

---

## 10.11 Pitanja za proveru znanja

1. Šta znači opcija **Propagate** kod dozvola?
2. Koju ulogu i na kojoj putanji dodeljuješ API tokenu za monitoring?
3. Zašto API token treba da ima **privilege separation**?
4. Koje korake radiš **pre** uključivanja firewall-a na data centru?
5. Objasni pravilo 3-2-1.
6. Navedi četiri prednosti PBS-a u odnosu na backup na NFS.
7. Zašto PBS ne treba da bude VM u istom klasteru koji čuva?
8. Šta se dešava ako izgubiš ključ za enkripciju backup-a?
9. Šta su RPO i RTO i kako ih meriš?

---

## Zaključak kursa

Prešao si ceo put: od instalacije jednog čvora do produkcionog klastera sa HA, Ceph-om, firewall-om i backup-om. Sledeći koraci:

1. **Ponovi vežbe** na laboratoriji od tri čvora, dok ti postupci ne postanu rutina.
2. **Simuliraj kvarove:** disk, čvor, mreža, gubitak kvoruma, vraćanje iz backup-a. Najviše se nauči kada nešto "pukne" u laboratoriji.
3. **Automatizuj:** Ansible za podešavanje hostova, Terraform/OpenTofu (Proxmox provider) za goste, cloud-init šabloni.
4. **Prati izmene:** čitaj release notes svake nove verzije i forum.proxmox.com.
5. **Razmisli o zvaničnoj obuci i pretplati** kada sistem pređe u produkciju.

Srećno!

---

## Izvori

- [Proxmox VE Administration Guide (9.3.1): User Management, Proxmox VE Firewall, Backup and Restore, Notifications](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)
- [What's new in Proxmox VE 9.3](https://www.proxmox.com/en/services/training-courses/videos/proxmox-virtual-environment/whats-new-in-proxmox-ve-9-3)
- [Proxmox VE dokumentacija, indeks](https://pve.proxmox.com/pve-docs/)

---

**Prethodno poglavlje:** [Poglavlje 9: Ceph i replikacija](09-ceph-i-replikacija.md)
**Nazad na početak:** [Sadržaj kursa](README.md)
