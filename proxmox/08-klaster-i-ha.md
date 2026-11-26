# Poglavlje 8: Klaster i visoka dostupnost (HA)

> **Cilj poglavlja:** Kada završiš ovo poglavlje, razumećeš kako radi Proxmox VE klaster (corosync, kvorum, pmxcfs) i znaćeš da napraviš klaster od tri čvora sa redundantnim corosync linkovima. Naučićeš i da podesiš **visoku dostupnost (HA)**, da razumeš **fencing** i da bezbedno održavaš čvorove u produkciji.

---

## 8.1 Šta je klaster i zašto ga praviš?

**Klaster** je grupa Proxmox VE čvorova kojima se upravlja kao **jednom celinom**:

- **jedan web interfejs** za sve čvorove (prijaviš se na bilo koji),
- **zajednička konfiguracija** (storage, korisnici, firewall, backup poslovi),
- **live migracija** VM-ova između čvorova,
- **visoka dostupnost (HA)**: automatsko pokretanje gostiju na drugom čvoru kada jedan otkaže,
- **replikacija** i **Ceph** (Poglavlje 9).

### Delovi klastera

| Komponenta | Uloga |
|---|---|
| **Corosync** | Komunikacija između čvorova: ko je živ, ko je u klasteru, kvorum |
| **pmxcfs** | Klasterski fajl sistem na `/etc/pve`. Promena na jednom čvoru odmah se vidi na svim |
| **Kvorum (quorum)** | Pravilo većine: klaster sme da menja stanje samo ako vidi **više od polovine** glasova |
| **pve-ha-crm / pve-ha-lrm** | HA menadžeri: jedan upravlja celim klasterom, a lokalni izvršava odluke na svakom čvoru |

---

## 8.2 Kvorum: najvažniji pojam

Svaki čvor ima **jedan glas**. Klaster ima kvorum kada vidi **više od polovine** svih glasova.

| Broj čvorova | Potrebno za kvorum | Sme da otkaže |
|---|---|---|
| 2 | 2 | **0** (zato 2 čvora nisu dovoljna bez QDevice-a) |
| 3 | 2 | 1 |
| 4 | 3 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

**Šta se dešava bez kvoruma?** Čvor bez kvoruma prebacuje `/etc/pve` u **režim samo za čitanje**: ne možeš da pokrećeš, praviš ni menjaš goste. Gosti koji **već rade** nastavljaju da rade (osim ako je uključen HA, odeljak 8.7).

### Zašto većina?

Zamisli klaster od 4 čvora koji se mrežno podeli na dve polovine po 2 čvora (*split brain*). Kada bi obe polovine nastavile da rade, obe bi mogle da pokrenu istu VM nad istim diskom, a to **uništava podatke**. Pravilo većine garantuje da **najviše jedna strana** može da nastavi.

> **Zato:** Pravi klastere sa **neparnim brojem glasova**, najmanje **3 čvora**. Ako imaš samo 2 servera, dodaj **QDevice** (odeljak 8.5).

---

## 8.3 Priprema za klaster

Pre pravljenja klastera proveri:

- [ ] svi čvorovi imaju **istu verziju** Proxmox VE-a (`pveversion`),
- [ ] **jedinstvena imena hostova** i ispravan `/etc/hosts` na svakom čvoru,
- [ ] **sinhronizovano vreme** (`chronyc tracking`),
- [ ] **posebna mreža za corosync**, po mogućstvu dve (Poglavlje 4), sa malim kašnjenjem,
- [ ] između čvorova su otvoreni portovi **UDP 5405–5412** (corosync), **TCP 22** (SSH) i **TCP 8006** (API),
- [ ] čvor koji se pridružuje klasteru **nema goste** (njegova konfiguracija u `/etc/pve` biće zamenjena konfiguracijom klastera),
- [ ] imena i IP adrese su **konačne**, jer je njihova promena kasnije komplikovana.

### Primer adresiranja

| Čvor | Upravljanje | Corosync link 0 | Corosync link 1 |
|---|---|---|---|
| pve1 | 10.0.10.11 | 10.0.1.11 | 10.0.2.11 |
| pve2 | 10.0.10.12 | 10.0.1.12 | 10.0.2.12 |
| pve3 | 10.0.10.13 | 10.0.1.13 | 10.0.2.13 |

---

## 8.4 Pravljenje klastera

### Korak 1: Pravljenje klastera na prvom čvoru

**pve1 → Datacenter → Cluster → Create Cluster**:

- **Cluster Name**: `prod-cl01` (ne može kasnije da se menja)
- **Cluster Network**: izaberi adresu za **Link 0** (`10.0.1.11`), klikni **Add** i dodaj **Link 1** (`10.0.2.11`)

Iz komandne linije:

```bash
pvecm create prod-cl01 --link0 10.0.1.11 --link1 10.0.2.11
```

### Korak 2: Pridruživanje ostalih čvorova

1. Na **pve1**: **Datacenter → Cluster → Join Information → Copy Information**.
2. Na **pve2**: **Datacenter → Cluster → Join Cluster**, nalepi informacije, unesi **root lozinku** za pve1 i izaberi **lokalne adrese** za link 0 (`10.0.1.12`) i link 1 (`10.0.2.12`).
3. Ponovi za **pve3**.

Iz komandne linije (na čvoru koji se pridružuje):

```bash
pvecm add 10.0.1.11 --link0 10.0.1.12 --link1 10.0.2.12
```

> Posle pridruživanja osveži web interfejs. Sertifikati čvora su zamenjeni, pa ćeš možda morati ponovo da prihvatiš sertifikat ili da se ponovo prijaviš.

### Korak 3: Provera

```bash
pvecm status
```

```
Cluster information
-------------------
Name:             prod-cl01
Config Version:   3
Transport:        knet
...
Votequorum information
----------------------
Expected votes:   3
Total votes:      3
Quorum:           2
Flags:            Quorate
```

Ključna linija je **`Flags: Quorate`**. Proveri i status corosync linkova:

```bash
corosync-cfgtool -s
pvecm nodes
```

---

## 8.5 Klaster od dva čvora: QDevice

Sa samo dva čvora otkaz jednog znači **gubitak kvoruma**. Rešenje je **QDevice**: mali spoljni servis (`corosync-qnetd`) koji daje **treći glas**. Može da radi na bilo kom Linux računaru van klastera, npr. na malom VM-u na drugom mestu ili na Raspberry Pi-ju.

```bash
# Na spoljnom računaru (Debian):
apt install corosync-qnetd

# Na SVIM čvorovima klastera:
apt install corosync-qdevice

# Na jednom čvoru klastera:
pvecm qdevice setup <IP-spoljnog-računara>
```

> **Napomena:** QDevice **ne sme** da radi kao VM unutar istog klastera kojem daje glas. QDevice se preporučuje za klastere sa **parnim** brojem čvorova. Kod neparnog broja ne pomaže, a može i da šteti.

---

## 8.6 Rad sa klasterom

### Uklanjanje čvora

```bash
# 1. Premesti ili obriši sve goste sa čvora pve3
# 2. Ugasi pve3 (trajno)
# 3. Na nekom od preostalih čvorova:
pvecm delnode pve3
```

> **Važno:** Uklonjeni čvor **ne sme ponovo da se uključi** u istu mrežu sa starom konfiguracijom, jer bi zbunio klaster. Reinstaliraj ga pre ponovne upotrebe.

### Hitni slučaj: gubitak kvoruma

Ako si ostao sa jednim čvorom (npr. dva od tri su trajno otkazala) i moraš hitno da pokreneš goste:

```bash
pvecm expected 1
```

Ovo privremeno smanjuje broj potrebnih glasova.

> **Opasno!** Koristi ovo **samo** kada si **siguran** da ostali čvorovi stvarno ne rade. Ako rade, a samo ih ne vidiš (pala je mreža), možeš da napraviš *split brain* i oštetiš podatke.

### Proxmox Datacenter Manager

Ako imaš **više klastera** (npr. na različitim lokacijama), pogledaj **Proxmox Datacenter Manager**. To je poseban proizvod za centralni pregled i upravljanje više klastera i samostalnih čvorova, uključujući migraciju između klastera.

---

## 8.7 Visoka dostupnost (HA)

HA prati goste i **automatski ih pokreće na drugom čvoru** kada čvor na kojem rade otkaže.

### Kako HA radi

1. Svaki čvor sa HA gostima ima **watchdog**, tajmer koji mora redovno da se "resetuje".
2. Dok čvor ima kvorum, `pve-ha-lrm` resetuje watchdog.
3. Ako čvor **izgubi kvorum** (mreža pukne, corosync ne radi), prestaje da resetuje watchdog. **Posle oko jednog minuta čvor se sam restartuje (fencing).**
4. Ostali čvorovi (koji imaju kvorum) čekaju dok ne budu sigurni da je čvor fencovan, pa onda **pokreću njegove HA goste kod sebe**.

Ceo proces traje obično **2 do 3 minuta**. HA **nije** isto što i neprekidan rad: gost se ponovo pokreće, kao posle nestanka struje.

```
pve1 ✗ (izgubio kvorum)       pve2 ✓, pve3 ✓ (kvorum)
   |                              |
   | watchdog ističe (~60 s)      | čekaju da pve1 bude fencovan
   | → restart (fence)            |
   |                              | → pokreću VM 100 i VM 101 na pve2/pve3
```

> **Zašto fencing?** Bez njega bi VM mogla da radi **istovremeno** na starom čvoru (koji je samo izgubio mrežu) i na novom, nad istim diskom. To znači oštećene podatke. Fencing garantuje da je stari čvor sigurno ugašen.

### Uslovi za HA

- najmanje **3 glasa** (3 čvora, ili 2 + QDevice),
- **deljeni storage** (Ceph, NFS, SAN) ili **ZFS replikacija**. Uz replikaciju gubiš podatke nastale od poslednje replikacije (Poglavlje 9),
- **pouzdana corosync mreža**, jer nestabilna mreža znači nepotrebne restarte čvorova,
- gost **bez lokalnih resursa** (lokalni ISO, PCI passthrough bez mapiranja, lokalni disk bez replikacije).

### Dodavanje gosta u HA

**Datacenter → HA → Resources → Add**:

| Polje | Preporuka |
|---|---|
| **VM/CT** | Izaberi gosta |
| **Max. Restart** | `1`: koliko puta da pokuša restart na istom čvoru |
| **Max. Relocate** | `1`: koliko puta da pokuša na drugom čvoru |
| **Request State** | `started` |

```bash
ha-manager add vm:100 --state started --max_restart 1 --max_relocate 1
ha-manager status
```

### HA pravila (HA rules)

Od verzije **9.0** stare **HA grupe** zamenjene su **HA pravilima** (**Datacenter → HA → Rules**):

| Pravilo | Primer upotrebe |
|---|---|
| **Node affinity** | VM baze radi **poželjno na pve1 ili pve2** (npr. zbog bržih diskova) |
| **Resource affinity: zajedno (positive)** | Aplikacija i njen keš uvek na **istom** čvoru |
| **Resource affinity: odvojeno (negative)** | Dva DNS servera ili dva čvora baze **nikad na istom čvoru**, da otkaz jednog servera ne obori oba |

> **Produkcija:** Za redundantne parove (DNS1/DNS2, DB primary/replica, load balancer-i) **obavezno** napravi pravilo **odvojeno**. Inače HA ili load balancer može da ih stavi na isti čvor.

### Šta HA radi kada **ručno** gasiš čvor

**Datacenter → Options → HA Settings → Shutdown Policy**:

| Politika | Ponašanje |
|---|---|
| **conditional** (podrazumevano) | Pri restartu se gosti "zamrznu" (freeze), a pri gašenju se prebacuju (failover) |
| **freeze** | Gosti se ne premeštaju, nego čekaju da se čvor vrati |
| **failover** | Gosti se pokreću na drugim čvorovima |
| **migrate** | Gosti se **uživo migriraju** pre gašenja |

Za planirano održavanje bolje je koristiti **režim održavanja** (odeljak 8.8).

---

## 8.8 Održavanje čvorova

### Režim održavanja (maintenance mode)

Kada čvor uđe u režim održavanja, HA **uživo migrira** sve njegove HA goste na druge čvorove i ne vraća ih dok održavanje ne završiš.

**Novo u 9.3:** režim održavanja se uključuje i isključuje **iz web interfejsa i API-ja**. Ranije je to bilo moguće samo iz komandne linije:

```bash
ha-manager crm-command node-maintenance enable pve1
# ... ažuriranje, restart, zamena hardvera ...
ha-manager crm-command node-maintenance disable pve1
```

### HA arm/disarm (9.2)

Od 9.2 HA može da se **privremeno isključi (disarm) za ceo klaster**. To je korisno za radove kod kojih znaš da će corosync mreža biti nestabilna (npr. zamena mrežnih svičeva). Tada ne želiš da HA restartuje čvorove zbog gubitka kvoruma. **Obavezno ga ponovo uključi (arm)** kada radovi završe.

> Pre upotrebe pročitaj u dokumentaciji tačno ponašanje gostiju i watchdog-a dok je HA isključen.

---

## 8.9 Raspoređivanje opterećenja

### Cluster Resource Scheduling (CRS)

**Datacenter → Options → Cluster Resource Scheduling**. Određuje na koji čvor HA stavlja gosta pri oporavku ili migraciji:

- **basic**: na osnovu broja HA gostiju po čvoru,
- **static**: na osnovu dodeljenih CPU-a i memorije gostiju.

### Dynamic Load Balancer (9.2)

Verzija 9.2 uvodi **dinamički load balancer**, koji prati **stvarnu potrošnju** resursa u klasteru i po potrebi premešta goste da bi opterećenje bilo ravnomerno.

> **Produkcija:** Pre uključivanja automatskog balansiranja definiši **HA pravila** (odvojeno/zajedno, node affinity), da balancer ne bi pomerao goste tamo gde ne treba.

---

## 8.10 Planiranje kapaciteta: N+1

Kada jedan čvor otkaže, njegovi gosti moraju **negde da stanu**. Pravilo **N+1**: klaster mora da podnese otkaz jednog čvora bez manjka resursa.

Primer: 3 čvora sa po 256 GB RAM-a.

```
Ukupno:                      768 GB
Ako otkaže jedan čvor:       512 GB
Rezerva za hostove (~10%):  - 50 GB
Maksimum za goste:          ≈ 460 GB  (a ne 768 GB!)
```

> Ako gosti zauzimaju više od ovoga, HA posle otkaza **neće moći da pokrene sve goste** (ili će hostovi ostati bez memorije).

---

## 8.11 Česte greške u produkciji

| Greška | Posledica | Kako izbeći |
|---|---|---|
| Klaster od 2 čvora bez QDevice-a | Otkaz jednog čvora zaustavlja upravljanje (i HA) | 3 čvora ili QDevice |
| Corosync na istoj mreži sa storage-om/backup-om | Gubitak kvoruma, HA restartuje čvorove bez razloga | Posebna mreža, 2 linka |
| Samo jedan corosync link | Kvar jednog sviča ruši klaster | Link 0 i link 1 na različitim svičevima |
| `pvecm expected 1` "da proradi" | *Split brain*, oštećeni podaci | Samo u hitnom slučaju, uz potvrdu da su ostali čvorovi ugašeni |
| Ponovno uključivanje uklonjenog čvora | Konfuzija u klasteru | Reinstalacija pre ponovne upotrebe |
| Redundantni par VM-ova na istom čvoru | Otkaz jednog servera obara oba servisa | HA pravilo **odvojeno** |
| Klaster bez N+1 kapaciteta | Posle otkaza gosti ne mogu da se pokrenu | Planiranje kapaciteta |
| HA nikad testiran | Iznenađenje pri prvom pravom otkazu | Redovni testovi failover-a |
| Radovi na mreži sa uključenim HA | Čvorovi se restartuju (fencing) | Režim održavanja ili HA disarm (9.2) |
| "Rastegnut" klaster na dve lokacije bez treće | Gubitak veze između lokacija ruši pola ili ceo klaster | Treća lokacija za glas (QDevice), malo kašnjenje, ili odvojeni klasteri |

---

## 8.12 Vežba

> Za ovu vežbu ti trebaju **tri čvora**: fizička ili ugnježdena (nested) VM-ovima na jednom računaru. Svaki čvor neka ima **dve mrežne kartice** (za link 0 i link 1).

1. Instaliraj Proxmox VE na `pve1`, `pve2` i `pve3` (Poglavlje 2), sa istom verzijom.
2. Napravi klaster `lab-cl01` sa **dva corosync linka**.
3. Pridruži `pve2` i `pve3`. Proveri `pvecm status` i `corosync-cfgtool -s`.
4. Napravi NFS storage (Poglavlje 5) ili koristi ZFS replikaciju (Poglavlje 9) i napravi malu VM na njemu.
5. Dodaj VM u **HA** sa `state=started`.
6. **Test failover-a:** "isključi struju" čvoru na kojem radi VM (prinudno ugasi nested VM). Meri koliko vremena prođe dok VM ne proradi na drugom čvoru. Prati `ha-manager status`.
7. Isključi **jedan corosync link** (iskopčaj karticu) i proveri da klaster i dalje ima kvorum.
8. Uključi **režim održavanja** na čvoru, proveri da su HA gosti migrirani, pa ga isključi.
9. Napravi dve VM (`dns1`, `dns2`) i **HA pravilo** da nikad ne rade na istom čvoru.

---

## 8.13 Rezime

- Klaster čine **corosync** (komunikacija i kvorum) i **pmxcfs** (`/etc/pve`, zajednička konfiguracija).
- **Kvorum** = više od polovine glasova. Najmanje **3 glasa**, a za 2 čvora dodaj **QDevice**.
- Corosync traži **posebnu mrežu sa malim kašnjenjem** i **dva linka**.
- **HA** ponovo pokreće goste na drugom čvoru za 2 do 3 minuta. **Fencing** (watchdog) sprečava da ista VM radi na dva mesta.
- Od 9.0 **HA pravila** (node affinity, resource affinity) zamenjuju HA grupe.
- **Novo:** HA arm/disarm i Dynamic Load Balancer (9.2), **režim održavanja u GUI-ju** (9.3).
- Planiraj kapacitet po pravilu **N+1** i **redovno testiraj failover**.

---

## 8.14 Pitanja za proveru znanja

1. Koliko čvorova sme da otkaže u klasteru od 5 čvorova, a da klaster zadrži kvorum?
2. Šta se dešava sa gostima koji već rade kada čvor izgubi kvorum (bez HA)?
3. Šta je QDevice i kada se koristi?
4. Objasni šta je fencing i zašto je neophodan.
5. Koliko otprilike traje oporavak HA gosta posle otkaza čvora?
6. Koje HA pravilo koristiš za dva DNS servera i zašto?
7. Kada koristiš režim održavanja, a kada HA disarm?
8. Klaster ima 4 čvora sa po 128 GB RAM-a. Koliko RAM-a najviše smeš da dodeliš gostima po pravilu N+1?

---

## Izvori

- [Proxmox VE Administration Guide (9.3.1): Cluster Manager, Proxmox Cluster File System, High Availability](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)
- [What's new in Proxmox VE 9.3](https://www.proxmox.com/en/services/training-courses/videos/proxmox-virtual-environment/whats-new-in-proxmox-ve-9-3)
- [Proxmox VE 9.2 najava na forumu](https://forum.proxmox.com/threads/proxmox-virtual-environment-9-2-available.183741/latest)

---

**Prethodno poglavlje:** [Poglavlje 7: Kontejneri (LXC)](07-kontejneri-lxc.md)
**Sledeće poglavlje:** [Poglavlje 9: Ceph i replikacija](09-ceph-i-replikacija.md)
