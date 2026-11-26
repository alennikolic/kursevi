# Poglavlje 3: Administracija hosta i web interfejs

> **Cilj poglavlja:** Kada završiš ovo poglavlje, snalazićeš se u web interfejsu i komandnoj liniji, znaćeš koji servisi pokreću Proxmox VE i gde da tražiš logove. Naučićeš i kako da host pripremiš za **produkciju**: pravi sertifikat, dvofaktorska autentifikacija, bezbedan SSH, kontrolisano ažuriranje i nadzor.

---

## 3.1 Web interfejs: snalaženje

Web interfejs otvaraš na `https://IP-ADRESA:8006`. Ekran je podeljen na četiri dela:

```
+-------------------------------------------------------------+
|  Zaglavlje: pretraga, Create VM, Create CT, korisnik        |
+----------------+--------------------------------------------+
|                |                                            |
|  Stablo        |   Panel sadržaja                           |
|  resursa       |   (zavisi od onoga što je izabrano levo)   |
|                |                                            |
|  Datacenter    |                                            |
|   └ pve1       |                                            |
|      ├ 100     |                                            |
|      ├ local   |                                            |
|      └ ...     |                                            |
+----------------+--------------------------------------------+
|  Donji panel: Tasks / Cluster log                           |
+-------------------------------------------------------------+
```

### Stablo resursa

Hijerarhija je uvek ista:

| Nivo | Šta se tu podešava |
|---|---|
| **Datacenter** | Podešavanja za ceo klaster: storage, korisnici i dozvole, firewall, backup poslovi, HA, SDN, notifikacije, ACME |
| **Čvor (node)** | Jedan fizički server: mreža, diskovi, sertifikati, ažuriranja, shell, sistemski log |
| **Gost (VM/CT)** | Hardver, opcije, konzola, snapshotovi, backup, firewall gosta |
| **Storage** | Sadržaj skladišta: diskovi, ISO, šabloni, backup-ovi |

Iznad stabla biraš **prikaz**:

- **Server View:** podrazumevani prikaz, po čvorovima,
- **Folder View:** grupisano po tipu (VM, CT, storage...),
- **Pool View:** grupisano po *resource pool*-ovima (korisno kada više timova deli klaster),
- **Tag View:** grupisano po tagovima.

### Tagovi

Gostima možeš da dodeliš **tagove** (npr. `prod`, `test`, `db`, `web`). U produkciji su vrlo korisni: odmah vidiš šta je kritično, a po tagovima možeš i da filtriraš. Tagove dodaješ u zaglavlju gosta, klikom na ikonicu olovke pored imena.

### Donji panel: Tasks i Cluster log

Svaka akcija (pokretanje VM-a, backup, migracija...) je **task**. Dvoklikom na task vidiš njegov kompletan izlaz. **Kada nešto ne radi, prvo pogledaj ovde.**

Taskovi se čuvaju i na disku, u direktorijumu `/var/log/pve/tasks/`.

---

## 3.2 Komandna linija: osnovni alati

Sve što radiš u GUI-ju možeš da uradiš i iz shell-a (čvor → **Shell** ili SSH). U produkciji je CLI neophodan za automatizaciju i za situacije kada web interfejs ne radi.

| Alat | Namena | Primer |
|---|---|---|
| `qm` | Upravljanje VM-ovima | `qm list`, `qm start 100`, `qm config 100` |
| `pct` | Upravljanje kontejnerima | `pct list`, `pct enter 200` |
| `pvesm` | Upravljanje skladištima | `pvesm status` |
| `pvecm` | Upravljanje klasterom | `pvecm status` |
| `pvesh` | Direktan pristup REST API-ju | `pvesh get /nodes` |
| `pvenode` | Podešavanja čvora (sertifikati, ACME, taskovi) | `pvenode task list` |
| `pveam` | Preuzimanje šablona za kontejnere | `pveam available` |
| `vzdump` | Backup gostiju | `vzdump 100 --storage backup` |
| `pveversion` | Verzije paketa | `pveversion -v` |
| `pveperf` | Brz test performansi diska i procesora | `pveperf /var/lib/vz` |
| `pvereport` | Kompletan izveštaj o sistemu (za podršku i dijagnostiku) | `pvereport > report.txt` |
| `pvesubscription` | Upravljanje pretplatom | `pvesubscription get` |

### `pvesh`: API iz komandne linije

`pvesh` koristi isti API kao web interfejs, samo bez mreže i prijave. Odličan je za skripte:

```bash
# Spisak čvorova
pvesh get /nodes

# Status svih gostiju u klasteru, kao JSON
pvesh get /cluster/resources --type vm --output-format json-pretty

# Pokretanje VM-a 100 na čvoru pve1
pvesh create /nodes/pve1/qemu/100/status/start
```

> **Savet:** Kada ne znaš putanju u API-ju, pogledaj **API Viewer** na `https://IP-ADRESA:8006/pve-docs/api-viewer/`.

---

## 3.3 Važni servisi i direktorijumi

### Servisi

| Servis | Uloga |
|---|---|
| `pve-cluster` | Pokreće **pmxcfs**, klasterski fajl sistem na `/etc/pve` |
| `pveproxy` | Web server za GUI i API (port 8006) |
| `pvedaemon` | Izvršava API zahteve sa root privilegijama |
| `pvestatd` | Prikuplja statistiku (CPU, RAM, storage) |
| `pvescheduler` | Pokreće zakazane poslove (backup, replikacija) |
| `pve-firewall` | Ugrađeni firewall |
| `corosync` | Komunikacija i kvorum u klasteru |
| `pve-ha-lrm`, `pve-ha-crm` | Upravljanje visokom dostupnošću (HA) |
| `spiceproxy` | Proxy za SPICE konzolu (port 3128) |

Provera statusa:

```bash
systemctl status pveproxy pvedaemon pvestatd pve-cluster
```

Ako se web interfejs "zaglavi", često pomaže restart proxy servisa. **Gosti nastavljaju da rade**, ovo utiče samo na GUI:

```bash
systemctl restart pveproxy pvedaemon
```

### Direktorijumi

| Putanja | Šta sadrži |
|---|---|
| `/etc/pve/` | **Sva konfiguracija Proxmox-a** (pmxcfs). U klasteru je ista na svim čvorovima |
| `/etc/pve/qemu-server/<VMID>.conf` | Konfiguracija VM-a |
| `/etc/pve/lxc/<VMID>.conf` | Konfiguracija kontejnera |
| `/etc/pve/storage.cfg` | Definicija skladišta |
| `/etc/pve/user.cfg` | Korisnici, grupe, dozvole |
| `/etc/pve/nodes/<ime>/` | Podešavanja konkretnog čvora (uključujući sertifikate) |
| `/var/lib/pve-cluster/config.db` | SQLite baza u kojoj pmxcfs zapravo čuva `/etc/pve` |
| `/var/lib/vz/` | Lokalni storage `local` (ISO, šabloni, backup) |
| `/var/log/pve/tasks/` | Logovi taskova |
| `/etc/network/interfaces` | Mrežna konfiguracija hosta |

> **Važno:** Fajlove u `/etc/pve` ne kopiraj i ne menjaj alatima koji prave privremene fajlove na neobičan način. Koristi `nano`, `vi` ili, još bolje, `qm set`/`pct set`. `/etc/pve` nije običan fajl sistem i ima ograničenja (npr. maksimalnu veličinu fajlova).

---

## 3.4 Logovi i dijagnostika

### Sistemski log (novo u 9.3)

Čvor → **System Log**. U verziji 9.3 log je **strukturiran i može da se filtrira** (po vremenu, servisu, prioritetu), pa je mnogo lakše naći grešku nego ranije, kada se listao ogroman tekst.

Novi paneli u 9.3:

- **API Access Log:** ko je i kada pristupao API-ju. Za produkciju je ovo važno zbog bezbednosti i revizije (*audit*).
- **APT History:** istorija instaliranih i ažuriranih paketa. Kada nešto prestane da radi posle ažuriranja, ovde odmah vidiš šta se promenilo.

### `journalctl` iz komandne linije

```bash
# Ceo log od poslednjeg pokretanja
journalctl -b

# Samo greške i ozbiljnije poruke
journalctl -p err -b

# Log jednog servisa, praćenje uživo
journalctl -u pveproxy -f

# Log od određenog vremena
journalctl --since "2026-11-26 08:00"
```

### `pvereport`

Kada otvaraš tiket kod Proxmox podrške ili pitaš na forumu, priloži izlaz `pvereport`. On sadrži verzije, mrežu, storage, klaster i hardver na jednom mestu.

> **Pažnja:** Izveštaj sadrži IP adrese, imena hostova i druge detalje infrastrukture. Pregledaj ga pre nego što ga javno objaviš.

---

## 3.5 Sertifikati: Let's Encrypt (ACME)

U produkciji ne želiš da svi korisnici "prihvataju rizik" zbog samopotpisanog sertifikata. Proxmox VE ima ugrađen **ACME** klijent, pa sertifikat od **Let's Encrypt**-a dobijaš i obnavljaš automatski.

### Uslovi

- host ima javno ime u DNS-u (npr. `pve1.firma.rs`),
- za **HTTP-01** validaciju port **80** mora da bude dostupan sa interneta,
- ako host **nije dostupan sa interneta** (što je česta i dobra praksa u produkciji), koristi **DNS-01** validaciju preko API-ja tvog DNS provajdera. Proxmox podržava veliki broj provajdera.

### Preko web interfejsa

1. **Datacenter → ACME → Accounts → Add**: registruj nalog sa svojom e-mail adresom.
2. (Za DNS-01) **Datacenter → ACME → Challenge Plugins → Add**: izaberi DNS provajdera i unesi API podatke.
3. **Čvor → System → Certificates → ACME → Add**: unesi domen (i plugin za DNS-01).
4. Klikni **Order Certificates Now**.

Obnavljanje je automatsko.

### Preko komandne linije (HTTP-01)

```bash
pvenode acme account register default admin@firma.rs
pvenode config set --acme domains=pve1.firma.rs
pvenode acme cert order
```

### Sopstveni sertifikat

Ako firma ima internu PKI (sopstveni CA), sertifikat i ključ učitavaš u **Čvor → System → Certificates → Upload Custom Certificate**.

---

## 3.6 Bezbednost hosta (hardening)

### 1. Dvofaktorska autentifikacija (2FA)

**Datacenter → Permissions → Two Factor → Add**. Podržani su:

- **TOTP:** aplikacije kao Google Authenticator, Aegis, FreeOTP,
- **WebAuthn:** hardverski ključevi (YubiKey i slični) i passkey-evi,
- **Recovery keys:** jednokratni ključevi za slučaj da izgubiš telefon ili ključ. **Obavezno ih napravi i sačuvaj na bezbednom mestu.**

> **Produkcija:** Uključi 2FA **za svakog** korisnika sa administratorskim pravima, a posebno za `root@pam`.

### 2. Ne radi svakodnevno kao `root`

Napravi lične naloge za administratore (detaljno u Poglavlju 10) i dodeli im uloge. Tako u logovima vidiš **ko je šta uradio**. Nalog `root@pam` čuvaj za hitne slučajeve.

### 3. SSH: ključevi umesto lozinki

```bash
# Na svom računaru:
ssh-copy-id root@pve1.firma.rs
```

Proveri da prijava ključem radi, pa tek onda u `/etc/ssh/sshd_config` podesi:

```
PermitRootLogin prohibit-password
PasswordAuthentication no
```

```bash
systemctl reload ssh
```

> **Pažnja:** **Nemoj** potpuno da zabraniš root prijavu (`PermitRootLogin no`) na čvorovima klastera. Proxmox koristi root SSH između čvorova za migracije, replikaciju i druge operacije. `prohibit-password` dozvoljava samo ključeve, što je i bezbedno i kompatibilno.

### 4. Odvojena mreža za upravljanje

Web interfejs (8006) i SSH (22) **ne smeju biti dostupni sa interneta**. U produkciji:

- upravljanje ide preko **posebne mreže ili VLAN-a**, kojoj imaju pristup samo administratori (preko VPN-a ili *jump* hosta),
- saobraćaj gostiju ide preko **druge mreže**,
- klaster (corosync) i storage (Ceph, NFS, iSCSI) imaju **svoje mreže** (Poglavlja 4, 8 i 9).

### 5. Firewall

Ugrađeni firewall (Poglavlje 10) omogućava da pristup portovima 8006 i 22 ograničiš samo na mrežu za upravljanje. **Pre uključivanja firewall-a na nivou data centra** dodaj pravilo koje dozvoljava tvoj pristup, inače možeš da se zaključaš.

### 6. Host je samo hipervizor

- **ne instaliraj dodatne servise na host** (web server, Docker, bazu...). Za to služe VM-ovi i kontejneri,
- svaki dodatni paket je potencijalni problem pri ažuriranju i dodatni rizik za bezbednost,
- izuzetak su alati za nadzor i upravljanje (npr. agent za monitoring).

---

## 3.7 Ažuriranje u produkciji

U Poglavlju 2 smo naučili kako se ažurira. U produkciji je važnije **kada i kojim redom**.

### Pravila

1. **Koristi Enterprise repozitorijum** (uz pretplatu). Paketi u njemu su dodatno testirani i stižu nešto kasnije, ali su stabilniji.
2. **Pročitaj release notes** pre svakog ažuriranja, a naročito pre prelaska na novu verziju (npr. 9.2 → 9.3).
3. **Ne ažuriraj automatski** (bez `unattended-upgrades` za Proxmox pakete). Ažuriranje radi ručno, u dogovorenom terminu za održavanje.
4. **Imaj test okruženje.** Makar jedan test čvor ili ugnježdeni klaster na kojem prvo probaš ažuriranje.
5. **Ažuriraj jedan po jedan čvor** u klasteru.
6. **Proveri da imaš sveže backup-ove** pre početka.

### Postupak za jedan čvor u klasteru

```
1. Uključi HA režim održavanja za čvor (od 9.3 i iz GUI-ja)
   → HA gosti se automatski premeštaju na druge čvorove
2. Ručno migriraj ostale goste koji ne treba da stoje
3. apt update && apt full-upgrade
4. reboot (ako je stigao novi kernel)
5. Proveri: pveversion -v, journalctl -p err -b, status servisa
6. Isključi režim održavanja
7. Pređi na sledeći čvor
```

Iz komandne linije režim održavanja se uključuje ovako:

```bash
ha-manager crm-command node-maintenance enable pve1
# ... ažuriranje i restart ...
ha-manager crm-command node-maintenance disable pve1
```

### Kerneli

Verzija 9.3 podrazumevano koristi **kernel 7.0**, a kernel **7.3** je dostupan kao **opt-in**. U produkciji ostani na podrazumevanom kernelu, osim ako ti noviji ne treba za određeni hardver. Ako probaš novi kernel, uvek možeš da se vratiš na stari:

```bash
# Spisak instaliranih kernela
proxmox-boot-tool kernel list

# Zaključaj (pin) kernel koji je proveren i radi
proxmox-boot-tool kernel pin <verzija>
```

---

## 3.8 Nadzor (monitoring)

Proxmox ima ugrađene grafike (čvor/gost → **Summary**), ali one čuvaju ograničenu istoriju i ne šalju upozorenja. U produkciji ti treba više.

### 1. Metric Server

**Datacenter → Metric Server → Add**. Proxmox šalje metrike (CPU, RAM, mreža, disk, za svaki čvor, gost i storage) u spoljnu bazu:

- **InfluxDB**, najčešće u kombinaciji sa **Grafana**-om za grafike,
- **Graphite**.

### 2. Notifikacije

Proxmox šalje obaveštenja o backup-ovima, ažuriranjima, HA događajima, replikaciji i greškama na diskovima. Podesi ih **odmah** (detaljno u Poglavlju 10). Od 9.3 pravila (*matchers*) mogu da se ugnježdavaju i uređuju prevlačenjem. Najmanje što treba da uradiš:

- proveri da li `root@pam` ima ispravnu e-mail adresu (Datacenter → Permissions → Users),
- proveri da li e-mail sa hosta zaista stiže (pogledaj i spam folder).

### 3. Zdravlje diskova (SMART)

Čvor → **Disks** prikazuje SMART status svih diskova. Proxmox instalira `smartmontools` i šalje upozorenje kada disk počne da otkazuje.

```bash
smartctl -a /dev/sda
```

### 4. Spoljni monitoring

Za ozbiljna okruženja koristi se i alat za nadzor koji već imaš (Zabbix, Prometheus, Checkmk...). Svi oni mogu da čitaju podatke iz Proxmox API-ja. Za pristup im napravi **poseban API token** sa ulogom koja dozvoljava samo čitanje (`PVEAuditor`), nikako sa root pravima.

### Šta pratiti

| Metrika | Zašto |
|---|---|
| Popunjenost storage-a (posebno **LVM-thin** i **ZFS**) | Pun thin pool ili ZFS pool može da ošteti podatke gostiju ili zaustavi VM-ove |
| RAM hosta i swap | Ako host ostane bez memorije, kernel ubija procese, a to mogu biti VM-ovi |
| Opterećenje procesora (*IO delay*) | Visok IO delay znači da je storage usko grlo |
| SMART i status RAID/ZFS pool-a | Rano otkrivanje kvarova diskova |
| Kvorum klastera | Bez kvoruma ne možeš da pokrećeš ni menjaš goste |
| Uspešnost backup-ova | Backup koji ne radi otkriješ tek kada ti zatreba |
| Datum isteka sertifikata | Za slučaj da automatska obnova ne uspe |

---

## 3.9 Backup konfiguracije hosta

Backup gostiju radiš sa `vzdump`/PBS (Poglavlje 10). **Konfiguracija samog hosta** je posebna stvar. Redovno čuvaj (npr. skriptom na drugi server):

```bash
tar czf /root/pve-host-config-$(hostname)-$(date +%F).tar.gz \
    /etc/pve \
    /var/lib/pve-cluster/config.db \
    /etc/network/interfaces \
    /etc/hosts \
    /etc/hostname \
    /etc/resolv.conf \
    /etc/apt/sources.list.d \
    /etc/ssh/sshd_config
```

> **Napomena:** U klasteru je `/etc/pve` isti na svim čvorovima, ali **lokalni fajlovi** (mreža, hosts) se razlikuju, pa ih čuvaj za **svaki** čvor.

Bolje rešenje od ručnog backup-a je da konfiguraciju hosta vodiš u alatu za automatizaciju (Ansible ili sličnom). Tako novi čvor podižeš iz koda za nekoliko minuta.

---

## 3.10 Česte greške u produkciji

| Greška | Posledica | Kako izbeći |
|---|---|---|
| Web interfejs dostupan sa interneta | Napadi grubom silom (brute force), iskorišćavanje ranjivosti | Posebna mreža za upravljanje, VPN, firewall |
| `apt upgrade` umesto `full-upgrade` | Nekonzistentni paketi, pokvaren sistem | Uvek `apt full-upgrade` |
| Ažuriranje svih čvorova odjednom | Prekid rada svih gostiju | Čvor po čvor, uz režim održavanja |
| Pun LVM-thin ili ZFS pool | Gosti staju, moguć gubitak podataka | Nadzor popunjenosti, upozorenje na 80% |
| Overcommit RAM-a bez nadzora | Host ubija VM-ove (OOM) | Ostavi rezervu RAM-a za host, ZFS ARC i Ceph |
| Bez 2FA na admin nalozima | Jedna ukradena lozinka znači pristup celoj infrastrukturi | 2FA za sve admine |
| Dodatni servisi instalirani na host | Problemi pri ažuriranju, veća površina za napad | Sve u VM/CT |
| Ručne izmene u `/etc/pve` bez backup-a | Pokvarena konfiguracija gosta ili klastera | `qm set`/`pct set`, backup konfiguracije |
| Notifikacije nisu podešene | Neuspeli backup ili disk koji otkazuje otkriješ prekasno | Notifikacije i spoljni monitoring od prvog dana |

---

## 3.11 Vežba

1. Prođi kroz sve nivoe u stablu (Datacenter, čvor, storage) i zapiši koje opcije postoje na svakom nivou.
2. Iz shell-a pokreni `pvesh get /nodes` i `pvesh get /cluster/resources`. Uporedi rezultate sa onim što vidiš u GUI-ju.
3. Pogledaj **System Log**, **API Access Log** i **APT History** na svom čvoru.
4. Pronađi u `journalctl` sve greške od poslednjeg pokretanja (`journalctl -p err -b`).
5. Uključi **TOTP 2FA** za `root@pam` i napravi **recovery keys**. Odjavi se i ponovo prijavi.
6. Podesi SSH prijavu ključem i `PermitRootLogin prohibit-password`. Proveri da prijava ključem i dalje radi **pre nego što zatvoriš postojeću sesiju**.
7. Napravi backup konfiguracije hosta komandom iz odeljka 3.9 i pogledaj sadržaj arhive.
8. *(Ako imaš javni domen)* Pribavi Let's Encrypt sertifikat preko ACME-a.

---

## 3.12 Rezime

- Web interfejs je organizovan kao **Datacenter → čvor → gost/storage**. Taskovi u donjem panelu su prvo mesto za dijagnostiku.
- `qm`, `pct`, `pvesm`, `pvecm` i `pvesh` pokrivaju sve što radiš u GUI-ju.
- Sva konfiguracija je u **`/etc/pve`** (pmxcfs), a čuva se u bazi `config.db`.
- Verzija 9.3 donosi **strukturiran System Log, API Access Log i APT History**.
- Za produkciju: **pravi sertifikat (ACME), 2FA, SSH ključevi, odvojena mreža za upravljanje**, host bez dodatnih servisa.
- Ažuriraj **ručno, čvor po čvor, uz režim održavanja**, posle čitanja release notes.
- Nadzor: **Metric Server, notifikacije, SMART i spoljni monitoring** sa API tokenom koji dozvoljava samo čitanje.

---

## 3.13 Pitanja za proveru znanja

1. Gde se nalazi konfiguracija VM-a sa VMID `105`?
2. Šta se dešava sa gostima kada restartuješ `pveproxy`?
3. Zašto na čvorovima klastera ne treba podesiti `PermitRootLogin no`?
4. Kada koristiš DNS-01, a kada HTTP-01 validaciju za ACME sertifikat?
5. Opiši postupak ažuriranja jednog čvora u produkcionom klasteru.
6. Koje tri metrike su najvažnije za nadzor i zašto?
7. Kakva prava treba da ima API token za spoljni monitoring?
8. Zašto ne treba instalirati Docker direktno na Proxmox host?

---

## Izvori

- [Proxmox VE Administration Guide (9.3.1): Host System Administration, Graphical User Interface, Certificate Management](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)
- [Proxmox VE API Viewer](https://pve.proxmox.com/pve-docs/api-viewer/)
- [What's new in Proxmox VE 9.3](https://www.proxmox.com/en/services/training-courses/videos/proxmox-virtual-environment/whats-new-in-proxmox-ve-9-3)

---

**Prethodno poglavlje:** [Poglavlje 2: Instalacija i prvo podešavanje](02-instalacija-i-prvo-podesavanje.md)
**Sledeće poglavlje:** [Poglavlje 4: Mrežna podešavanja i SDN](04-mreza-i-sdn.md)
