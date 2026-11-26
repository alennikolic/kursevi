# Poglavlje 5: Skladišta podataka (storage)

> **Cilj poglavlja:** Kada završiš ovo poglavlje, razumećeš koje vrste skladišta Proxmox VE podržava, po čemu se razlikuju i koje da izabereš za koji scenario. Znaćeš da podesiš **ZFS**, **NFS** i **deljeni LVM preko SAN-a** (sa novim SAN prikazom iz 9.3) i kako da izbegneš greške zbog kojih se u produkciji gube podaci.

---

## 5.1 Osnovni pojmovi

U Proxmox VE-u **storage** je bilo koje mesto gde se čuvaju podaci. Skladišta se definišu **na nivou data centra** (`/etc/pve/storage.cfg`), pa ih vide svi čvorovi u klasteru.

### Vrste sadržaja (content)

Za svako skladište biraš šta sme da se čuva na njemu:

| Content | Šta je | Primer upotrebe |
|---|---|---|
| **Disk image** (`images`) | Diskovi virtuelnih mašina | local-lvm, ZFS, Ceph |
| **Container** (`rootdir`) | Diskovi kontejnera | local-lvm, ZFS, Ceph |
| **ISO image** (`iso`) | Instalacioni ISO fajlovi | local, NFS |
| **Container template** (`vztmpl`) | Šabloni za LXC | local, NFS |
| **VZDump backup file** (`backup`) | Backup fajlovi | NFS, poseban disk |
| **Snippets** (`snippets`) | Hookscript-ovi, cloud-init fajlovi | local |
| **Import** (`import`) | Fajlovi za uvoz VM-ova (npr. OVA) | local, NFS |

### Fajl i blok skladišta

- **Fajl skladišta** (*file level*): Directory, NFS, CIFS, CephFS. Diskovi su fajlovi u formatu **qcow2**, **raw** ili **vmdk**.
- **Blok skladišta** (*block level*): LVM, LVM-thin, ZFS, Ceph RBD, iSCSI. Svaki disk je poseban blok uređaj (logički volumen, zvol, RBD image).

### qcow2 i raw

| | **raw** | **qcow2** |
|---|---|---|
| Performanse | Najbolje | Malo slabije |
| Snapshotovi | Zavise od skladišta | Ugrađeni u format |
| Thin provisioning | Zavisi od skladišta | Da |
| Preporuka | Na blok skladištima (ZFS, LVM-thin, Ceph) | Na fajl skladištima (Directory, NFS) kada ti trebaju snapshotovi |

---

## 5.2 Pregled tipova skladišta

| Tip | Nivo | Deljeno u klasteru | Snapshotovi | Napomena |
|---|---|---|---|---|
| **Directory** | fajl | ne | qcow2 | Najjednostavnije, lokalni direktorijum |
| **LVM** | blok | da, preko SAN-a | **da, od 9.3 za produkciju** (volume chain) | Thick provisioning. Osnova za deljeni SAN |
| **LVM-thin** | blok | ne | da | Podrazumevano posle instalacije (`local-lvm`) |
| **ZFS (lokalni)** | blok/fajl | ne | da | Integritet podataka, kompresija, replikacija |
| **NFS** | fajl | da | qcow2 | Popularan za ISO i backup |
| **CIFS/SMB** | fajl | da | qcow2 | Windows deljeni folderi |
| **iSCSI** | blok | da | ne | Obično samo kao osnova za LVM na vrhu |
| **ZFS over iSCSI** | blok | da | da | Zahteva podržan storage sistem |
| **Ceph RBD** | blok | da | da | Distribuirano skladište (Poglavlje 9) |
| **CephFS** | fajl | da | da | Za ISO, šablone, backup |
| **Proxmox Backup Server** | backup | da | – | Samo za backup (Poglavlje 10) |
| **BTRFS** | fajl | ne | da | Tehnološki pregled |

> **"Deljeno" (shared)** znači da svi čvorovi vide **isti podatak**. Samo sa deljenim skladištem live migracija prenosi samo memoriju, a ne i disk, pa je mnogo brža. Uz to i HA može da pokrene gosta na drugom čvoru kada jedan otkaže.

### Primer `/etc/pve/storage.cfg`

```
dir: local
        path /var/lib/vz
        content iso,vztmpl,backup

lvmthin: local-lvm
        thinpool data
        vgname pve
        content rootdir,images

zfspool: tank
        pool tank
        content rootdir,images
        sparse 1

nfs: nas-backup
        server 10.0.30.50
        export /export/pve-backup
        path /mnt/pve/nas-backup
        content backup
        options vers=4.2
```

Komande za pregled stanja:

```bash
pvesm status          # svi storage-i i popunjenost
pvesm list local-lvm  # sadržaj jednog storage-a
```

---

## 5.3 LVM-thin (podrazumevano)

Posle instalacije sa ext4/xfs dobijaš `local-lvm`, **LVM-thin pool**. Thin znači da disk od 100 GB zauzima onoliko prostora koliko je stvarno upisano.

### Opasnost: overprovisioning

Možeš da napraviš VM-ove sa ukupno 2 TB diskova na pool-u od 500 GB. Sve radi dok se pool ne napuni. **Kada se thin pool napuni, gosti staju sa I/O greškama, a fajl sistemi unutra mogu da se oštete.**

Prati popunjenost (i podataka i metapodataka):

```bash
lvs -a pve
```

```
  LV              VG  Attr       LSize   Pool Data%  Meta%
  data            pve twi-aotz-- 350.00g      62.15  3.10
```

> **Produkcija:** Podesi upozorenje kada `Data%` ili `Meta%` pređe **80%**. Uključi i **discard/TRIM** na diskovima gostiju (odeljak 5.8), da se obrisani podaci vraćaju u pool.

---

## 5.4 ZFS

**ZFS** je fajl sistem i menadžer volumena u jednom. U produkciji se najčešće koristi za lokalna skladišta zbog:

- **zaštite integriteta podataka:** svaki blok ima kontrolnu sumu, pa ZFS otkriva oštećene podatke (*bit rot*) i popravlja ih iz kopije,
- **ugrađenog RAID-a** (mirror, RAIDZ),
- **trenutnih snapshotova** i kloniranja,
- **kompresije** (podrazumevano `lz4`, koja najčešće i ubrzava rad),
- **replikacije** između čvorova (Poglavlje 9).

### Nivoi zaštite

| Nivo | Minimum diskova | Može da otkaže | Iskorišćenost | Za VM-ove |
|---|---|---|---|---|
| **Mirror** (RAID1) | 2 | 1 (po paru) | 50% | Odlično |
| **RAID10** (mirror parovi) | 4 | 1 po paru | 50% | **Najbolje za VM-ove** (IOPS) |
| **RAIDZ1** | 3 | 1 | (n-1)/n | Slabije IOPS |
| **RAIDZ2** | 4 | 2 | (n-2)/n | Dobro za velike diskove i backup |
| **RAIDZ3** | 5 | 3 | (n-3)/n | Arhive |

> **Pravilo:** Za diskove virtuelnih mašina koristi **mirror ili RAID10**. RAIDZ ima dobar odnos kapaciteta i cene, ali slabije performanse za nasumičan I/O, kakav prave VM-ovi.

### Pravljenje ZFS pool-a

**Čvor → Disks → ZFS → Create: ZFS**:

- Name: `tank`
- RAID Level: `Mirror`
- Compression: `on` (lz4)
- ashift: `12` (za diskove sa sektorom od 4K, što je danas skoro uvek slučaj)
- izaberi diskove i uključi **Add Storage**

Iz komandne linije:

```bash
zpool create -o ashift=12 tank mirror /dev/disk/by-id/ata-DISK1 /dev/disk/by-id/ata-DISK2
zfs set compression=lz4 tank
pvesm add zfspool tank --pool tank --content images,rootdir --sparse 1
```

> **Koristi `/dev/disk/by-id/...`**, a ne `/dev/sda`. Imena tipa `sda` mogu da se promene posle restarta.

### Memorija: ZFS ARC

ZFS koristi RAM kao keš (**ARC**). Na novim instalacijama Proxmox ograničava ARC na **10% RAM-a (najviše 16 GB)**. Proveri trenutno stanje i po potrebi promeni ograničenje:

```bash
arc_summary | head -30
```

Primer: ograničenje na 8 GiB (8 × 1024³ bajtova):

```bash
echo "options zfs zfs_arc_max=8589934592" > /etc/modprobe.d/zfs.conf
update-initramfs -u -k all
# primenjuje se posle restarta, ili odmah:
echo 8589934592 > /sys/module/zfs/parameters/zfs_arc_max
```

Okvirno pravilo: **2 GB + 1 GB po TB** skladišta za ARC. Ostatak ostavi gostima.

### Održavanje

```bash
zpool status          # stanje pool-a i diskova
zpool list            # kapacitet
zpool scrub tank      # provera svih podataka
```

Debian automatski pokreće **scrub** jednom mesečno. Proveri rezultate u `zpool status`, a notifikacije o ZFS greškama stižu preko **ZED** servisa.

### Zamena neispravnog diska (mirror)

```bash
zpool status tank                     # nađi disk u stanju FAULTED/UNAVAIL
zpool replace tank <stari-disk> /dev/disk/by-id/<novi-disk>
zpool status tank                     # prati resilver
```

Ako je pool **sistemski** (`rpool`, sa kojeg host butuje), novi disk mora dobiti i particije za butovanje:

```bash
sgdisk /dev/<ispravan-disk> -R /dev/<novi-disk>   # kopiraj tabelu particija
sgdisk -G /dev/<novi-disk>                        # novi GUID-ovi
zpool replace -f rpool <stara-particija> <nova-particija-3>
proxmox-boot-tool format /dev/<novi-disk-particija-2>
proxmox-boot-tool init /dev/<novi-disk-particija-2>
```

> **Vežbaj ovo u laboratoriji pre nego što ti zatreba u produkciji.** Zamena diska u sistemskom pool-u je postupak koji se radi retko i pod stresom.

### Šta NE raditi sa ZFS-om

- **ne koristi hardverski RAID** ispod ZFS-a (kontroler mora da bude u HBA/IT režimu),
- **ne uključuj deduplikaciju** (`dedup`) bez ozbiljnog razloga i planiranja, jer troši ogromne količine RAM-a,
- **ne koristi jeftine "consumer" SSD-ove** za intenzivan rad. ZFS i VM-ovi brzo troše SSD bez zaštite od nestanka struje (PLP), a performanse padaju pri dužem upisu,
- **ne puni pool preko 80%**, jer performanse primetno padaju.

---

## 5.5 NFS i CIFS

NFS je najčešći izbor za **deljeni prostor za ISO fajlove, šablone i backup**. Može da čuva i diskove VM-ova, ali u produkciji to zavisi od kvaliteta NAS-a i mreže.

**Datacenter → Storage → Add → NFS**:

| Polje | Primer |
|---|---|
| ID | `nas-backup` |
| Server | `10.0.30.50` |
| Export | `/export/pve-backup` (bira se iz liste) |
| Content | `VZDump backup file` |
| NFS Version | `4.2` |

Na NAS-u dozvoli pristup **IP adresama svih čvorova** (sa `no_root_squash` ako Proxmox treba da piše kao root).

> **Produkcija:** NFS za backup treba da bude **na drugom fizičkom uređaju**, ne na istom serveru čije goste čuvaš. Za backup je još bolje rešenje **Proxmox Backup Server** (Poglavlje 10).

**CIFS/SMB** se dodaje na isti način (**Add → SMB/CIFS**), sa korisničkim imenom i lozinkom za deljeni folder.

---

## 5.6 Deljeni storage preko SAN-a (iSCSI, FC, NVMe-oF)

U firmama koje prelaze sa VMware-a najčešće već postoji **SAN** (storage sistem povezan preko iSCSI-ja, Fibre Channel-a ili NVMe over Fabrics). Proxmox ga koristi kao **deljeni LVM**:

```
   SAN storage ── LUN ──┬── putanja 1 ──┐
                        └── putanja 2 ──┴─→ multipath uređaj → LVM VG (shared) → diskovi VM-ova
                                             (na svakom čvoru)
```

1. **Multipath:** svaki čvor vidi LUN preko više putanja (dve kartice, dva sviča, dva kontrolera storage-a). `multipathd` ih spaja u jedan uređaj i prebacuje saobraćaj kada jedna putanja otkaže.
2. **LVM** na multipath uređaju, označen kao **shared**. Svi čvorovi vide istu volume grupu, a Proxmox se stara da u svakom trenutku samo jedan čvor koristi određeni disk.

### Novo u 9.3: SAN integracija

- **vođeno podešavanje** deljenog LVM-a sa multipath-om za iSCSI, FC i NVMe-oF, iz web interfejsa,
- **multipath se podešava na nivou klastera**, a ne ručno na svakom čvoru,
- novi prikaz **Datacenter → SAN** pokazuje stanje putanja za svaki LUN, na svim čvorovima. Odmah vidiš ako je neki čvor izgubio jednu putanju.

### Snapshotovi na deljenom LVM-u (produkcija od 9.3)

Klasičan deljeni (thick) LVM ranije **nije podržavao snapshotove**, i to je bio veliki nedostatak u odnosu na VMware VMFS. Proxmox VE 9.0 je uveo snapshotove preko **volume chain** mehanizma kao tehnološki pregled, a u **9.3 su spremni za produkciju**.

Uključuju se opcijom **"Allow Snapshots as Volume-Chain"** u podešavanjima LVM storage-a. Diskovi tada koriste **qcow2 format na LVM volumenima**, pa svaki snapshot postaje poseban volumen u lancu.

> **Napomena:** Pre upotrebe u produkciji pročitaj u zvaničnoj dokumentaciji ograničenja ove funkcije: uticaj na performanse kod dugih lanaca snapshotova, zauzeće prostora i postojeće diskove u raw formatu.

### Ručno podešavanje (iSCSI, za razumevanje)

```bash
# 1. iSCSI storage (samo kao osnova, bez direktnog korišćenja)
pvesm add iscsi san1 --portal 10.0.40.10 --target iqn.2026-01.rs.firma:storage.lun1 --content none

# 2. Provera multipath uređaja
multipath -ll

# 3. LVM grupa na multipath uređaju (na jednom čvoru)
pvcreate /dev/mapper/mpatha
vgcreate vg_san1 /dev/mapper/mpatha

# 4. Deljeni LVM storage
pvesm add lvm san1-lvm --vgname vg_san1 --shared 1 --content images
```

U 9.3 ove korake obavlja vođeno podešavanje u GUI-ju, ali je korisno da znaš šta se dešava ispod.

---

## 5.7 Storage pluginovi trećih strana

Proizvođači storage sistema (i zajednica) prave **pluginove** koji Proxmox-u omogućavaju direktnu integraciju sa njihovim uređajima (snapshotovi na samom storage-u, kloniranje...). **Od 9.3** pluginovi trećih strana mogu da se dodaju i uređuju **kroz web interfejs**, a ne samo ručnom izmenom `storage.cfg`.

> **Produkcija:** Plugin treće strane je kod koji radi sa root privilegijama na svakom čvoru. Koristi samo pluginove od proizvođača kojima veruješ, proveri da li podržavaju tvoju verziju Proxmox-a i **testiraj ih posle svakog ažuriranja**.

---

## 5.8 Opcije diskova gostiju

**VM → Hardware → Hard Disk → Edit**:

| Opcija | Preporuka | Zašto |
|---|---|---|
| **Bus/Device** | **SCSI** sa kontrolerom **VirtIO SCSI single** | Najbolje performanse i podrška za sve opcije |
| **Cache** | **No cache** (podrazumevano) | Bezbedno i brzo. `writeback` je brži, ali rizikuje gubitak podataka pri nestanku struje |
| **Discard** | **Uključeno** | TRIM iz gosta vraća prostor u thin pool, ZFS ili Ceph |
| **IO thread** | **Uključeno** | Poseban thread za I/O diska, uz VirtIO SCSI single |
| **SSD emulation** | Uključeno na SSD storage-u | Gost zna da je disk SSD i šalje TRIM |
| **Backup** | Uključeno (osim za diskove koje ne treba čuvati) | Disk ulazi u backup |
| **Async IO** | `io_uring` (podrazumevano) | Za većinu slučajeva najbolje |
| **Bandwidth limits** | Po potrebi | Da jedan gost ne zauzme ceo storage |

---

## 5.9 Kako izabrati storage za produkciju

| Scenario | Preporuka |
|---|---|
| Jedan server, bez klastera | **ZFS mirror/RAID10** na enterprise SSD/NVMe diskovima |
| 2 do 3 servera, bez SAN-a, mali budžet | Lokalni **ZFS + replikacija** (Poglavlje 9). Kratak gubitak podataka pri otkazu (od poslednje replikacije) |
| 3 ili više servera, bez SAN-a | **Ceph** (Poglavlje 9), sa najmanje 10 Gbit mrežom, a bolje 25 Gbit |
| Postojeći SAN (iSCSI/FC/NVMe-oF) | **Deljeni LVM + multipath**, sa volume chain snapshotovima (9.3) |
| Postojeći kvalitetan NAS | NFS za diskove VM-ova (ako NAS ima redundansu) |
| ISO i šabloni | NFS ili CephFS, deljeno za sve čvorove |
| Backup | **Proxmox Backup Server** na posebnom hardveru, a kao alternativa NFS na posebnom NAS-u |

---

## 5.10 Česte greške u produkciji

| Greška | Posledica | Kako izbeći |
|---|---|---|
| Pun LVM-thin pool | Gosti staju, moguće oštećenje fajl sistema | Nadzor `Data%`/`Meta%`, discard, nema velikog overprovisioning-a |
| ZFS na hardverskom RAID-u | ZFS ne vidi greške diskova i ne može da popravi podatke | HBA/IT režim |
| Consumer SSD za ZFS/Ceph | Brzo trošenje, pad performansi, gubitak podataka pri nestanku struje | Enterprise SSD sa PLP-om |
| ARC bez ograničenja na hostu sa malo RAM-a | VM-ovi ostaju bez memorije | Ograniči `zfs_arc_max` |
| RAIDZ za intenzivne VM-ove | Slab IOPS | Mirror / RAID10 |
| Backup na istom serveru ili storage-u | Kvar uništava i podatke i backup | Backup na posebnom uređaju (PBS) |
| Diskovi dodati kao `/dev/sdX` | Pogrešan disk posle restarta | `/dev/disk/by-id/` |
| Neisprobana zamena sistemskog diska | Duži zastoj kada disk otkaže | Vežbaj u laboratoriji, dokumentuj postupak |
| SAN bez multipath-a | Jedan kabl ili svič ruši sve VM-ove | Multipath i nadzor putanja (Datacenter → SAN u 9.3) |
| Cache `writeback` bez UPS-a | Gubitak podataka pri nestanku struje | `No cache`, UPS |

---

## 5.11 Vežba

1. Pokreni `pvesm status` i `lvs -a pve`. Zapiši popunjenost `local-lvm`.
2. Pogledaj `/etc/pve/storage.cfg` i objasni svaku stavku.
3. *(Ako imaš dodatne diskove, ili dodaj 2 virtuelna diska u nested VM)* Napravi **ZFS mirror** pool `tank` i dodaj ga kao storage.
4. Proveri `zpool status`, pokreni `zpool scrub tank` i pogledaj rezultat.
5. Simuliraj otkaz diska: u nested VM ukloni jedan disk, pogledaj stanje pool-a (`DEGRADED`), dodaj novi disk i uradi `zpool replace`.
6. Ograniči ZFS ARC na 2 GiB i proveri sa `arc_summary`.
7. *(Ako imaš NAS ili drugi Linux računar)* Dodaj NFS storage za ISO fajlove i backup.
8. Uključi **Discard**, **IO thread** i **SSD emulation** na disku jedne VM.

---

## 5.12 Rezime

- Storage se definiše na nivou data centra (`storage.cfg`). Svaki storage ima **content** tipove.
- **Blok** skladišta (LVM, ZFS, Ceph) koriste raw diskove, a **fajl** skladišta (NFS, Directory) qcow2 ili raw.
- **LVM-thin** je podrazumevan, ali pazi na **popunjenost** pool-a.
- **ZFS** pruža integritet podataka, snapshotove i replikaciju. Za VM-ove koristi **mirror/RAID10**, ograniči **ARC**, i nikad ne stavljaj ZFS na hardverski RAID.
- **Deljeni storage** omogućava brzu live migraciju i HA.
- **Novo u 9.3:** vođeno podešavanje **SAN-a sa multipath-om**, prikaz **Datacenter → SAN**, **snapshotovi na deljenom LVM-u spremni za produkciju** i **pluginovi trećih strana u GUI-ju**.
- Backup nikad ne čuvaj na istom mestu kao originalne podatke.

---

## 5.13 Pitanja za proveru znanja

1. Koja je razlika između fajl i blok skladišta?
2. Šta se dešava kada se LVM-thin pool napuni i kako to sprečiti?
3. Zašto je RAID10 bolji izbor od RAIDZ1 za diskove virtuelnih mašina?
4. Zašto ZFS ne treba koristiti preko hardverskog RAID kontrolera?
5. Šta je ZFS ARC i kako ga ograničavaš?
6. Šta znači da je storage "shared" i zašto je to važno za HA?
7. Šta je multipath i koju novinu za SAN donosi verzija 9.3?
8. Koje tri opcije diska gosta treba uključiti na SSD storage-u?

---

## Izvori

- [Proxmox VE Administration Guide (9.3.1): Proxmox VE Storage, ZFS on Linux](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)
- [What's new in Proxmox VE 9.3](https://www.proxmox.com/en/services/training-courses/videos/proxmox-virtual-environment/whats-new-in-proxmox-ve-9-3)
- [StorageReview: Proxmox VE 9.3 SAN management and LVM snapshots](https://www.storagereview.com/news/proxmox-ve-9-3-cluster-wide-san-management-lvm-snapshots?amp=)

---

**Prethodno poglavlje:** [Poglavlje 4: Mrežna podešavanja i SDN](04-mreza-i-sdn.md)
**Sledeće poglavlje:** [Poglavlje 6: Virtuelne mašine (QEMU/KVM)](06-virtuelne-masine.md)
