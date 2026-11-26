# Poglavlje 9: Ceph i replikacija

> **Cilj poglavlja:** Kada završiš ovo poglavlje, razumećeš kako radi **Ceph**, distribuirano skladište ugrađeno u Proxmox VE. Znaćeš da napraviš hiperkonvergentni Ceph klaster i da ga održavaš i nadzireš u produkciji. Naučićeš i da koristiš **ZFS replikaciju** kao jednostavniju alternativu za manje klastere.

---

## 9.1 Dva načina da podaci budu na više čvorova

Za HA i brzu migraciju podaci gostiju moraju da budu dostupni na više čvorova. Bez SAN-a imaš dve mogućnosti:

| | **Ceph** | **ZFS replikacija** |
|---|---|---|
| Kako radi | Podaci se **istovremeno** upisuju na više čvorova | Lokalni ZFS, sa **periodičnim** kopiranjem na drugi čvor |
| Gubitak podataka pri otkazu | **Nema** (sinhrono) | Podaci nastali od poslednje replikacije (npr. do 15 minuta) |
| Minimum čvorova | **3** (preporuka 4 ili više) | 2 (+ QDevice za HA) |
| Mreža | **10 Gbit minimum**, preporuka 25 Gbit ili više | 1 Gbit može, a 10 Gbit je bolje |
| Složenost | Veća | Mala |
| Performanse | Dobre, zavise od mreže i diskova | Odlične (lokalni disk) |
| Skaliranje | Dodaš čvor ili disk, kapacitet raste | Ograničeno na čvor |

---

## 9.2 Ceph: osnovni pojmovi

**Ceph** je distribuirani sistem za skladištenje. Podatke deli na objekte i raspoređuje ih na diskove svih čvorova, sa više kopija.

| Komponenta | Uloga | Koliko |
|---|---|---|
| **MON** (Monitor) | Čuva "mapu" klastera: ko je gde, šta je ispravno. Odlučuje po kvorumu | **3** (ili 5 u velikim klasterima), uvek neparno |
| **MGR** (Manager) | Statistike, dashboard, moduli (npr. PG autoscaler) | Najmanje 2 (aktivni + rezervni) |
| **OSD** (Object Storage Daemon) | Jedan proces po **disku**. Čuva podatke i kopira ih na druge OSD-ove | Po jedan za svaki disk |
| **MDS** (Metadata Server) | Potreban samo za **CephFS** | 1 aktivni + rezervni |

### Pool, replike i PG

- **Pool**: logička grupa podataka (npr. `vm-pool` za diskove VM-ova),
- **size = 3**: svaki podatak postoji u **3 kopije**, na 3 različita čvora,
- **min_size = 2**: pool prima upise dok postoje najmanje **2** ispravne kopije,
- **PG** (*Placement Group*): grupe objekata koje Ceph raspoređuje po OSD-ovima. Broj PG-ova automatski podešava **PG autoscaler**,
- **CRUSH**: algoritam koji određuje gde ide koja kopija. Podrazumevano su kopije na **različitim čvorovima** (*failure domain = host*).

```
           VM disk (RBD image)
                 |
         podeljen na objekte
                 |
     +-----------+-----------+
     |           |           |
  kopija 1    kopija 2    kopija 3
  (pve1 OSD)  (pve2 OSD)  (pve3 OSD)
```

### RBD i CephFS

- **RBD** (*RADOS Block Device*): blok uređaji za **diskove VM-ova i kontejnera**. Ovo je glavna upotreba,
- **CephFS**: deljeni fajl sistem za **ISO fajlove, šablone, snippets** i (opciono) backup.

> **Pravilo:** Nikad ne podešavaj `size=2, min_size=1` u produkciji. Sa samo jednom ispravnom kopijom svaki dodatni kvar znači **gubitak podataka**.

---

## 9.3 Zahtevi za produkciju

### Čvorovi

- **najmanje 3 čvora**, a **preporučeno 4 ili više**. Kod 3 čvora i `size=3`, kada jedan čvor otkaže, Ceph **nema gde da napravi treću kopiju** i ostaje u stanju *degraded* dok se čvor ne vrati. Sa 4 i više čvorova Ceph **sam obnavlja** treću kopiju,
- **ujednačeni čvorovi**: slični diskovi i kapaciteti na svim čvorovima.

### Diskovi

- **enterprise SSD/NVMe sa zaštitom od nestanka struje (PLP)**. Consumer SSD-ovi su **veoma spori** za Ceph, jer Ceph traži potvrđen upis (*sync write*) za svaku operaciju,
- **bez RAID kontrolera**: HBA/IT režim, svaki disk je poseban OSD,
- HDD je prihvatljiv samo za veliki, sporiji kapacitet (arhive, backup), po mogućstvu sa DB/WAL na NVMe-u.

### Mreža

- **posebna mreža za Ceph**, najmanje **10 Gbit**, a za NVMe **25 Gbit ili više**,
- opciono dve mreže:
  - **public network**: komunikacija klijenata (VM-ova) sa Ceph-om,
  - **cluster network**: replikacija i oporavak između OSD-ova,
- redundantna (bond, dva sviča), sa **MTU 9000** ako ga cela mreža podržava (Poglavlje 4).

### Memorija i procesor

- svaki OSD podrazumevano koristi do **4 GiB RAM-a** (`osd_memory_target`). Za čvor sa 6 OSD-ova računaj **~24 GiB samo za Ceph**, plus MON/MGR,
- za NVMe OSD-ove računaj i nekoliko **CPU jezgara po OSD-u**.

---

## 9.4 Instalacija Ceph-a

### Verzije u Proxmox VE 9.3

| Ceph verzija | Status |
|---|---|
| **Tentacle 20.2** | **Podrazumevana, stabilna** (od 9.2) |
| Squid 19.2 | Starija stabilna, i dalje dostupna |
| **Umbrella 21.2** | **Tehnološki pregled** u 9.3, ne za produkciju |

### Korak 1: Instalacija paketa (na svakom čvoru)

**Čvor → Ceph → Install Ceph**: izaberi verziju (**Tentacle**) i repozitorijum (**Enterprise** uz pretplatu, ili **No-Subscription**).

```bash
pveceph install --repository no-subscription
```

### Korak 2: Inicijalna konfiguracija (jednom, na prvom čvoru)

Čarobnjak traži **public** i (opciono) **cluster** mrežu:

```bash
pveceph init --network 10.0.30.0/24 --cluster-network 10.0.31.0/24
```

### Korak 3: Monitori i manageri

Na **tri** čvora:

**Čvor → Ceph → Monitor → Create** (MON i MGR)

```bash
pveceph mon create
pveceph mgr create
```

### Korak 4: OSD-ovi

Na svakom čvoru, za svaki disk:

**Čvor → Ceph → OSD → Create: OSD** → izaberi disk.

```bash
pveceph osd create /dev/nvme1n1
pveceph osd create /dev/nvme2n1
```

> Disk mora da bude **prazan** (bez particija). Ako je ranije korišćen, očisti ga sa `ceph-volume lvm zap /dev/nvme1n1 --destroy` ili kroz **Disks → Wipe Disk**.

### Korak 5: Pool za VM-ove

**Čvor → Ceph → Pools → Create**:

- Name: `vm-pool`
- Size: `3`, Min. Size: `2`
- PG Autoscale Mode: `on`
- **Add as Storage**: uključeno (automatski pravi RBD storage u Proxmox-u)

```bash
pveceph pool create vm-pool --size 3 --min_size 2 --add_storages
```

### Korak 6 (opciono): CephFS

```bash
pveceph mds create          # na 2 čvora (aktivni + rezervni)
pveceph fs create --name cephfs --add-storage
```

---

## 9.5 Nadzor Ceph-a

### Web interfejs

**Čvor → Ceph** prikazuje stanje celog Ceph klastera: health, broj OSD-ova (up/in), popunjenost, performanse.

### Komandna linija

```bash
ceph -s                 # kratak status
ceph health detail      # detalji upozorenja
ceph osd tree           # raspored OSD-ova po čvorovima i njihovo stanje
ceph osd df tree        # popunjenost po OSD-u
ceph df                 # popunjenost po pool-u
ceph osd perf           # kašnjenje po OSD-u
```

### Stanja

| Stanje | Značenje | Šta raditi |
|---|---|---|
| `HEALTH_OK` | Sve je u redu | – |
| `HEALTH_WARN` | Upozorenje (npr. jedan OSD ne radi, popunjenost se približava granici) | Pročitaj `ceph health detail` i reši uzrok |
| `HEALTH_ERR` | Ozbiljan problem, mogući nedostupni podaci | **Hitno** |

### Popunjenost: najvažnija metrika

| Prag (podrazumevano) | Šta se dešava |
|---|---|
| **nearfull 85%** | Upozorenje |
| **backfillfull 90%** | Ceph prestaje da premešta podatke na taj OSD |
| **full 95%** | **Upis staje** za ceo pool, a VM-ovi se zamrzavaju |

**Iskoristivi kapacitet** sa `size=3` je **trećina** sirovog kapaciteta. Ako želiš da Ceph sam obnovi kopije posle otkaza jednog čvora, moraš da imaš **slobodno mesto i za podatke tog čvora**.

Primer: 4 čvora × 4 TB = 16 TB sirovo.

```
Sirovo:                                 16 TB
Iskoristivo (size=3):                   ~5,3 TB
Posle otkaza jednog čvora (12 TB / 3):  4 TB
Bezbedna granica (~80% od 4 TB):        ≈ 3,2 TB za podatke gostiju
```

> **Produkcija:** Podesi upozorenje na **70 do 75%** popunjenosti i planiraj proširenje na vreme.

---

## 9.6 Održavanje

### Restart čvora (ažuriranje)

Kada se čvor ugasi, Ceph posle nekoliko minuta označi njegove OSD-ove kao `out` i počinje da **premešta podatke** (velik saobraćaj). Za kratko planirano održavanje to ne želiš:

```bash
ceph osd set noout       # pre restarta
# ... ažuriranje i restart čvora ...
ceph -s                  # sačekaj da se svi OSD-ovi vrate (up)
ceph osd unset noout     # obavezno posle!
```

Kombinuj sa HA režimom održavanja (Poglavlje 8) i radi **čvor po čvor**. Na sledeći čvor pređi tek kada je `ceph -s` ponovo **HEALTH_OK**.

### Zamena neispravnog diska (OSD)

1. **Čvor → Ceph → OSD**: izaberi neispravan OSD → **Out**. Ceph premešta podatke sa njega.
2. Sačekaj da `ceph -s` pokaže da je premeštanje završeno (svi PG-ovi `active+clean`).
3. **Stop**, pa **More → Destroy**.
4. Fizički zameni disk.
5. **Create: OSD** na novom disku.

```bash
ceph osd out osd.7
# sačekaj active+clean
systemctl stop ceph-osd@7
pveceph osd destroy 7 --cleanup
pveceph osd create /dev/nvme2n1
```

### Nadogradnja verzije Ceph-a

Nadogradnja Ceph-a na novu glavnu verziju (npr. Squid → Tentacle) radi se **posebno** od ažuriranja Proxmox-a, po zvaničnom vodiču na Proxmox wiki-ju. Redosled je strogo određen: monitori, pa manageri, pa OSD-ovi, pa MDS. **Ne preskači verzije** i ne radi nadogradnju dok klaster nije u stanju HEALTH_OK.

---

## 9.7 ZFS replikacija

**Storage Replication** (`pvesr`) periodično kopira ZFS diskove gosta na jedan ili više drugih čvorova. Koristi ZFS snapshotove i šalje **samo promene**, pa je posle prve kopije veoma brza.

### Uslovi

- gost je na **lokalnom ZFS** storage-u,
- **ZFS storage sa istim imenom** postoji na ciljnom čvoru,
- čvorovi su u **istom klasteru**.

### Podešavanje

**VM → Replication → Add** (ili **Datacenter → Replication**):

| Polje | Primer |
|---|---|
| **Target** | `pve2` |
| **Schedule** | `*/15` (svakih 15 minuta). Najkraće je `*/1` (svaki minut) |
| **Rate limit** | Ograničenje brzine u MB/s, da replikacija ne zaguši mrežu |

```bash
pvesr create-local-job 100-0 pve2 --schedule "*/15" --rate 100
pvesr status
pvesr list
```

### Replikacija i HA

Ako je gost u HA i ima replikaciju, posle otkaza čvora HA ga pokreće na **čvoru sa replikom**. Gost se tada vraća na stanje iz **poslednje replikacije**, pa su **podaci nastali posle nje izgubljeni**.

> **Produkcija:** Odluči sa vlasnicima servisa koliki gubitak podataka je prihvatljiv (**RPO**). Za baze kojima ni minut nije prihvatljiv, koristi Ceph, SAN ili replikaciju na nivou aplikacije (npr. PostgreSQL streaming replication).

### Replikacija ubrzava migraciju

Kada migriraš gosta na čvor na kojem već postoji replika, prenose se **samo promene** od poslednje replikacije. Migracija gosta sa lokalnim diskom tako traje sekunde umesto sati.

---

## 9.8 Šta izabrati?

| Situacija | Preporuka |
|---|---|
| 2 čvora, mali budžet | **ZFS + replikacija** + QDevice |
| 3 čvora, 10 Gbit, prihvatljiv kratki gubitak podataka | ZFS + replikacija, ili Ceph (uz svest o ograničenju kod 3 čvora) |
| 4 ili više čvorova, 25 Gbit, NVMe sa PLP-om | **Ceph** |
| Postojeći SAN | Deljeni LVM (Poglavlje 5) |
| Baze sa nultim RPO | Ceph/SAN + replikacija na nivou baze |

---

## 9.9 Česte greške u produkciji

| Greška | Posledica | Kako izbeći |
|---|---|---|
| Ceph na 1 Gbit mreži | Veoma spor rad, dugačak oporavak | 10 Gbit minimum, a bolje 25 Gbit |
| Consumer SSD-ovi za OSD | Visoko kašnjenje, brzo trošenje | Enterprise SSD/NVMe sa PLP-om |
| `size=2, min_size=1` | Gubitak podataka pri dva kvara | `size=3, min_size=2` |
| Ceph puni preko 80% | Pri otkazu nema mesta za oporavak, a na 95% upis staje | Upozorenje na 70–75%, planiraj proširenje |
| Restart čvora bez `noout` | Nepotrebno premeštanje podataka | `ceph osd set noout` / `unset` |
| Zaboravljen `unset noout` | Ceph ne obnavlja kopije posle stvarnog kvara | Kontrolna lista za održavanje |
| Ceph i gosti na istoj mreži | Spor storage i nestabilan klaster | Posebna Ceph mreža |
| Paralelni restart dva čvora | Pool sa manje od `min_size` kopija, pa VM-ovi staju | Čvor po čvor, čekaj HEALTH_OK |
| Replikacija bez nadzora | Replika stara danima, a niko ne zna | Notifikacije za replikaciju, `pvesr status` |
| Neusklađen RPO sa vlasnicima servisa | Neočekivan gubitak podataka posle HA failover-a | Dokumentuj RPO po servisu |

---

## 9.10 Vežba

> Za vežbu sa Ceph-om ti trebaju **3 čvora** (može nested), svaki sa **najmanje jednim dodatnim praznim diskom** (npr. 32 GB), i klaster iz Poglavlja 8.

**Ceph:**

1. Instaliraj Ceph (Tentacle, no-subscription) na sva tri čvora.
2. Inicijalizuj Ceph sa posebnom mrežom (ako je imaš).
3. Napravi MON i MGR na sva tri čvora.
4. Napravi po jedan OSD na svakom čvoru.
5. Napravi pool `vm-pool` (size 3, min_size 2) i dodaj ga kao storage.
6. Napravi VM na `vm-pool` i uradi **live migraciju**. Uporedi brzinu sa migracijom lokalnog diska.
7. Pokreni `ceph -s`, `ceph osd tree` i `ceph df` i objasni izlaz.
8. Simuliraj kvar: ugasi jedan čvor i posmatraj `ceph -s` (degraded). Uključi ga ponovo i prati oporavak.
9. Ponovi korak 8 sa `ceph osd set noout` i uporedi ponašanje.

**ZFS replikacija:**

10. Na dva čvora napravi ZFS storage sa **istim imenom** (Poglavlje 5).
11. Napravi VM na ZFS-u i podesi replikaciju na drugi čvor na svaki minut.
12. Dodaj VM u HA, ugasi čvor na kojem radi i proveri na kom stanju podataka se pokrenula na drugom čvoru.

---

## 9.11 Rezime

- **Ceph** upisuje podatke **sinhrono** u više kopija na više čvorova. **ZFS replikacija** kopira periodično i jednostavnija je.
- Ceph komponente: **MON** (3), **MGR**, **OSD** (po disku), **MDS** (samo za CephFS).
- Produkcija: **size=3, min_size=2**, **4 ili više čvorova**, **posebna mreža od 10/25 Gbit**, **enterprise SSD/NVMe sa PLP-om**, bez RAID kontrolera.
- Iskoristivo je **trećina sirovog kapaciteta**, a ostavi i mesta za oporavak. Prati popunjenost.
- Za održavanje koristi **`noout`**, radi čvor po čvor i čekaj **HEALTH_OK**.
- **Verzije u 9.3:** Tentacle 20.2 je stabilna, Umbrella 21.2 je tehnološki pregled.
- **ZFS replikacija** je odlična za 2 do 3 čvora, ali posle HA failover-a gubiš podatke nastale od poslednje replikacije (**RPO**).

---

## 9.12 Pitanja za proveru znanja

1. Koja je osnovna razlika između Ceph-a i ZFS replikacije kada čvor otkaže?
2. Čemu služe MON, MGR i OSD?
3. Šta znače `size=3` i `min_size=2`?
4. Zašto je za Ceph bolje imati 4 nego 3 čvora?
5. Ceph klaster ima 30 TB sirovog kapaciteta. Koliko otprilike možeš da koristiš sa `size=3`?
6. Šta se dešava kada Ceph pool dostigne 95% popunjenosti?
7. Čemu služi `ceph osd set noout` i šta ne smeš da zaboraviš posle održavanja?
8. Šta je RPO i kako se odnosi na ZFS replikaciju sa rasporedom `*/15`?

---

## Izvori

- [Proxmox VE Administration Guide (9.3.1): Deploy Hyper-Converged Ceph Cluster, Storage Replication](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)
- [What's new in Proxmox VE 9.3](https://www.proxmox.com/en/services/training-courses/videos/proxmox-virtual-environment/whats-new-in-proxmox-ve-9-3)
- [Proxmox VE 9.2 najava na forumu](https://forum.proxmox.com/threads/proxmox-virtual-environment-9-2-available.183741/latest)

---

**Prethodno poglavlje:** [Poglavlje 8: Klaster i visoka dostupnost (HA)](08-klaster-i-ha.md)
**Sledeće poglavlje:** [Poglavlje 10: Bezbednost, backup i notifikacije](10-bezbednost-backup-notifikacije.md)
