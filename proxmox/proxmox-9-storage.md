# Proxmox VE 9.x — Storage: kada se koja opcija koristi

Beleška o tipovima storage-a u Proxmox VE 9.x (testirano na 9.3). Pokriva
`Datacenter → Storage → Add` dijalog, content tipove i izbor storage-a po scenariju.

---

## 1. Osnovni model

Sva konfiguracija storage-a živi u **jednom fajlu**:

```
/etc/pve/storage.cfg
```

Pošto je `/etc/pve` cluster filesystem (pmxcfs), fajl se automatski replicira na
sve nodove u klasteru. **Storage se definiše jednom, sa bilo kog noda** — ne
ponavlja se na svakom nodu.

Primer:

```
lvm: LUN23
        vgname VG_LUN23
        base ISCSI_ID:0.0.0.scsi-360060160...
        content images,rootdir
        shared 1
```

Definicija storage-a ≠ dostupnost storage-a. Definicija je cluster-wide, ali
nod mora **fizički da vidi uređaj** da bi storage na njemu bio `active`.

---

## 2. Content tipovi — najvažniji koncept

Svaki storage ima listu dozvoljenih **content** tipova. Ovo određuje šta se na
njega sme smestiti:

| Content | U GUI-ju | Šta je |
|---|---|---|
| `images` | Disk image | Diskovi VM-ova i VM state |
| `rootdir` | Container | Volumeni LXC kontejnera |
| `vztmpl` | Container template | Šabloni za LXC |
| `iso` | ISO image | ISO fajlovi za instalaciju |
| `backup` | VZDump backup file | vzdump backupi |
| `snippets` | Snippets | hook skripte, cloud-init userdata |
| `import` | Import | OVA i disk image fajlovi za import |

### Zamka koja svakoga uhvati prvi put

U **Create VM** wizardu:

- korak **OS** → bira se ISO → tu se vide **samo** storage-i sa `iso` contentom
- korak **Disks** → bira se mesto za disk → tu se vide storage-i sa `images`

Ako si napravio LVM preko iSCSI LUN-a i ne vidiš ga u koraku OS — **to nije
kvar**. Blok storage ne može da drži fajlove, pa ni ISO. ISO ostaje na `local`
(ili NFS/CIFS), a samo disk VM-a ide na LUN.

Isto važi i za backup: ne možeš vzdump da pišeš direktno na LVM.

---

## 3. Fajl nivo vs blok nivo

Ovo je linija koja deli sve ostalo.

**Fajl nivo** (dir, NFS, CIFS, BTRFS, CephFS, ZFS dataset)
- disk VM-a je fajl (`.qcow2` ili `.raw`) u direktorijumu
- može da drži ISO, šablone, backupe, snippets
- snapshot postoji ako je format `qcow2`

**Blok nivo** (LVM, LVM-Thin, ZFS zvol, RBD, iSCSI, NVMe-oF)
- disk VM-a je logički volumen / zvol / LUN, nema fajl sistema između
- **samo** `images` i `rootdir`
- obično brže, bez overhead-a fajl sistema

---

## 4. Tabela svih tipova

Prema zvaničnoj PVE dokumentaciji:

| Opis | Plugin | Nivo | Shared | Snapshots | Status |
|---|---|---|---|---|---|
| ZFS (local) | `zfspool` | fajl i blok | Ne | Da | Stabilno |
| Directory | `dir` | fajl | Ne | Da (samo qcow2) | Stabilno |
| BTRFS | `btrfs` | fajl | Ne | Da | Technology preview |
| NFS | `nfs` | fajl | Da | Da (samo qcow2) | Stabilno |
| CIFS/SMB | `cifs` | fajl | Da | Da (samo qcow2) | Stabilno |
| Proxmox Backup Server | `pbs` | fajl i blok | Da | n/a | Stabilno |
| CephFS | `cephfs` | fajl | Da | n/a | Stabilno |
| LVM | `lvm` | blok | Ne (Da uz iSCSI/FC/SAS/NVMe-oF) | Da | Stabilno |
| LVM-Thin | `lvmthin` | blok | **Ne** | Da | Stabilno |
| iSCSI (kernel) | `iscsi` | blok | Da | Da | Stabilno |
| iSCSI (libiscsi) | `iscsidirect` | blok | Da | Da | Stabilno |
| NVMe-oF | `nvmeof` | blok | Da | Da | Stabilno |
| FC / SAS | nema plugin | blok | Da | Da | Stabilno |
| Ceph / RBD | `rbd` | blok | Da | Da | Stabilno |
| ZFS over iSCSI | `zfs` | blok | Da | Da | Stabilno |

Napomene iz dokumentacije:
- Kod fajl storage-a snapshot radi preko `qcow2`. Interni qcow2 snapshotovi
  blokiraju VM dok traju i loše se ponašaju na mrežnom storage-u.
- FC i SAS nemaju svoj plugin — host ih vidi kao blok uređaj, pa se preko njih
  pravi običan LVM sa `shared` flagom.

---

## 5. Kada se koja opcija koristi

### Network

**NFS** — opšte rešenje za deljeni storage kad nemaš SAN. Drži sve content
tipove, deli se između nodova, snapshot radi preko qcow2. Sporije od blok
storage-a, ali najjednostavnije za setup. Dobar izbor za ISO/backup repozitorijum
čak i kad VM diskovi idu negde drugde.

**SMB/CIFS** — isto kao NFS, ali kad je izvor Windows share ili NAS koji ne nudi
NFS. Po pravilu ide NFS ako postoji izbor.

**RBD (Ceph)** — hiperkonvergentno rešenje. Tri ili više nodova, disk svakog
noda ulazi u pool, replikacija na nivou Ceph-a. Shared, snapshot, thin, sve radi
nativno. Cena je kompleksnost i zahtev za bar 3 noda i zasebnom mrežom.

**CephFS** — fajl sloj nad Ceph-om. Ne drži VM diskove (`images` nije podržan),
koristi se za ISO, šablone, backupe i snippets u Ceph klasteru.

### SAN

**Shared LVM on iSCSI** — wizard. Tvoj izbor kad imaš klaster i jedan LUN koji
vidi više hostova. U jednom prolazu: iSCSI konekcija, multipath, kreiranje VG-a,
`shared` flag. Rezultat u `storage.cfg` je isti kao ručna varijanta.

**Shared LVM on Fibre Channel** — isto, samo što LUN dolazi preko FC HBA. Nema
iSCSI storage objekta, LUN se prosto pojavi kao blok uređaj.

**Shared LVM on NVMe over Fabrics** — isto nad NVMe/TCP namespace-om.

**iSCSI** — sirova konekcija na target. Dve upotrebe:
- `content: none` → služi samo kao baza za LVM odozgo (uobičajeno)
- `content: images` („Use LUNs directly") → LUN se dodeljuje direktno jednoj VM
  kao raw disk, bez LVM-a. Retko, uglavnom za specifične performanse.

> iSCSI storage u GUI-ju često pokazuje status `unknown`. To je po dizajnu —
> PVE ne proverava dostupnost targeta kao kod NFS-a. Status koji te zanima je
> LVM iznad njega.

**NVMe over Fabrics** — povezivanje na NVMe/TCP subsystem. Koristi **kernel
multipath**, ne `multipath-tools`. Zahteva `pve-manager 9.2.38+` na svim
nodovima pre dodavanja prvog takvog storage-a i `nvme-cli` na nodovima.
Konekcije idu bez autentikacije i bez TLS-a, a PVE ne pravi niti briše
namespace-ove — to radiš na storage strani.

**ZFS over iSCSI** — specijalan slučaj: storage server koji vrti ZFS (TrueNAS,
Solaris, Linux sa ZFS-om) i kome PVE pristupa preko SSH-a da bi pravio zvol-ove
po VM-u. Daje snapshot i thin preko mreže, ali zahteva poverenje i SSH pristup
ka storage serveru.

### Backup

**Proxmox Backup Server** — namenski backup target. Deduplikacija, inkrementalni
backup, verifikacija. Jedini content je `backup`.

### Import

**ESXi** — read-only, za migraciju sa VMware-a. Doda se vCenter/ESXi host i
guestovi se importuju direktno, bez ručnog konvertovanja diskova.

### Filesystems and Volume Groups

**Directory** — direktorijum na nodu. Default `local` je upravo ovo. Najfleksibilniji
po contentu (drži sve), ali lokalan. Koristi se za ISO, backupe i lab VM-ove.

**LVM** — volume group nad lokalnim diskom ili nad LUN-om. Thick provisioning.
Nad deljenim LUN-om uz `shared 1` je klasično SAN rešenje.

**LVM-Thin** — thin pool, default `local-lvm` posle instalacije. Thin
provisioning i brzi snapshotovi, ali **isključivo lokalno** (vidi dole).

**ZFS** — lokalni ZFS pool. Snapshot, kompresija, checksum, replikacija između
nodova preko `pvesr`. Najbolji izbor za lokalni storage bez SAN-a. Traži RAM.

**BTRFS** — i dalje **technology preview** sa best-effort podrškom. Ne koristiti
`cache mode none` jer BTRFS poštuje `O_DIRECT`, što dovodi do checksum grešaka.

---

## 6. LVM-Thin se NIKAD ne stavlja na deljeni LUN

Najčešća i najskuplja greška u SAN setupu.

- **Thick LVM** radi kao shared storage, ali tradicionalno nema snapshot
- **LVM-Thin** ima snapshot, ali je lokalan

Razlog je mehanički: thin pool drži svoje metapodatke **unutar samog pool-a** i
nema cluster-aware zaključavanje. Dva noda koja istovremeno drže thin pool
aktivnim pišu u iste metapodatke bez koordinacije → korupcija podataka.

Nijedna „pažljiva" konfiguracija to ne rešava. Za deljeni LUN ide **thick LVM**.

### Kako PVE čuva thick LVM od istog problema

Model je **single-writer / multiple-reader**. Pojedinačni LV je aktivan samo na
nodu gde VM radi, a `/etc/pve` locking sprečava da dva noda startuju istu VM.

**Ovo radi samo ako su nodovi u istom Proxmox klasteru.** Dva nepovezana hosta
nad istim LUN-om nemaju corosync, nemaju lock — i to se završi isto kao thin
pool. Prvo klaster, pa onda shared storage.

---

## 7. Snapshots as Volume Chains (PVE 9+)

Novina u PVE 9 koja rešava istorijski nedostatak thick LVM-a.

Svaki snapshot živi u zasebnom thick-provisioned volumenu, slojevito preko
qcow2 backing-chain mehanizma, **bez** qcow2 internih snapshotova.

Uključuje se u `Add: LVM` dijalogu, pod **Advanced** → *Allow Snapshots as
Volume-Chain*. U konfiguraciji se vidi kao:

```
snapshot-as-volume-chain 1
```

Ograničenja:
- važi **samo za novokreirane diskove**
- ne može da se isključi dok na storage-u postoje qcow2 volumeni
- za racionalno trošenje prostora traži thin provisioning i discard na SAN strani

---

## 8. Multipath (iSCSI / FC)

Ako postoji više putanja do LUN-a, multipath se podešava **pre** kreiranja VG-a.
PV mora da bude `/dev/mapper/<wwid>`, nikako `/dev/sdX`.

```bash
apt install multipath-tools
/lib/udev/scsi_id -g -u -d /dev/sdb        # WWID uređaja
```

`/etc/multipath.conf` — skelet:

```
defaults {
    polling_interval 2
    find_multipaths  yes
}

blacklist {
    wwid .*
}

blacklist_exceptions {
    wwid "3600601600a30100012345678"
}
```

```bash
systemctl restart multipathd
multipath -ll
```

Parametre u `devices {}` sekciji uzeti iz dokumentacije proizvođača storage-a
(EMC, NetApp, Pure i ostali imaju preporučene vrednosti).

---

## 9. CLI

```bash
# status svih storage-a na nodu
pvesm status

# lista sadržaja storage-a
pvesm list LUN23

# discovery iSCSI targeta
pvesm scan iscsi 10.14.8.253

# dodavanje iSCSI storage-a (samo kao baza, bez direktnih LUN-ova)
pvesm add iscsi ISCSI_ID \
  --portal 10.14.8.253 \
  --target iqn.1992-04.com.emc:cx.virt23169hfwrw.a2 \
  --content none

# LVM nad tim LUN-om, deljen
pvesm add lvm LUN23 \
  --vgname VG_LUN23 \
  --base ISCSI_ID:0.0.0.scsi-3600601600a301000 \
  --content images,rootdir \
  --shared 1

# izmena postojećeg
pvesm set LUN23 --content images,rootdir

# uklanjanje definicije (ne dira podatke)
pvesm remove LUN23
```

iSCSI ručno:

```bash
cat /etc/iscsi/initiatorname.iscsi          # IQN ovog hosta
iscsiadm -m discovery -t sendtargets -p 10.14.8.253
iscsiadm -m node --login
iscsiadm -m session                          # aktivne sesije
rescan-scsi-bus.sh                           # novi LUN-ovi bez reboota
```

---

## 10. Brza dijagnostika

Storage je `enabled` ali VM ne može da se napravi na njemu:

```bash
pvesm status          # da li je active na TOM nodu
vgs                   # da li VG uopšte postoji ovde
lsblk -S              # da li kernel vidi LUN
iscsiadm -m session   # da li postoji sesija
```

| Simptom | Uzrok |
|---|---|
| Storage se ne vidi u koraku **OS** (ISO) | Normalno — blok storage nema `iso` content |
| `vgs` prazan na jednom nodu | Nema iSCSI sesije ili LUN nije mapiran na taj IQN |
| `lsblk -S` ne pokazuje LUN | Mapiranje na storage strani (igroup / storage group) |
| Storage `inactive` iako postoji | `vgchange -ay VG_IME`, pa `pvesm status` |
| Storage se ne nudi na nodu 2 | Polje **Nodes** u definiciji storage-a je ograničeno |
| iSCSI status `unknown` | Očekivano, iSCSI je samo transport |

Ako je LUN mapiran naknadno, rescan bez reboota:

```bash
rescan-scsi-bus.sh
vgscan --mknodes
pvesm status
```

---

## 11. Izbor u jednoj rečenici

| Scenario | Izbor |
|---|---|
| Jedan nod, bez SAN-a | ZFS (local) ili LVM-Thin |
| Klaster, bez SAN-a, 3+ noda | Ceph / RBD |
| Klaster, postojeći iSCSI SAN | Shared LVM on iSCSI |
| Klaster, FC SAN | Shared LVM on Fibre Channel |
| Klaster, NVMe/TCP | Shared LVM on NVMe over Fabrics |
| Deljeni storage bez SAN-a | NFS |
| ISO i šabloni | `local` (dir), NFS ili CephFS |
| Backup | Proxmox Backup Server |
| Migracija sa VMware-a | ESXi import |

---

## Izvori

- [Proxmox VE Storage dokumentacija](https://pve.proxmox.com/pve-docs/chapter-pvesm.html)
- [Proxmox VE Wiki — Storage: iSCSI](https://pve.proxmox.com/wiki/Storage:_iSCSI)
- [Proxmox VE Roadmap / Release notes](https://pve.proxmox.com/wiki/Roadmap)
