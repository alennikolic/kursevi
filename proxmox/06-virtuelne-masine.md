# Poglavlje 6: Virtuelne mašine (QEMU/KVM)

> **Cilj poglavlja:** Kada završiš ovo poglavlje, znaćeš da napraviš i podesiš virtuelne mašine za Linux i Windows po najboljim praksama. Naučićeš da praviš šablone sa **cloud-init**-om, da koristiš snapshotove i live migraciju, da uvoziš VM-ove sa VMware-a i da pripremiš VM za produkciju.

---

## 6.1 Pravljenje VM-a: čarobnjak korak po korak

Klikni **Create VM** u gornjem desnom uglu. Čarobnjak ima nekoliko kartica.

### General

| Polje | Preporuka |
|---|---|
| **Node** | Čvor na kojem se VM pravi |
| **VM ID** | Broj gosta. U produkciji uvedi **šemu** (npr. `1xx` web, `2xx` baze, `9xxx` šabloni) |
| **Name** | Ime hosta gosta, npr. `web01` (bez razmaka) |
| **Resource Pool** | Opciono, za grupisanje i dozvole |

### OS

- izaberi **ISO image** sa storage-a (prethodno ga učitaj: storage → **ISO Images → Upload** ili **Download from URL**),
- **Guest OS type**: `Linux` ili `Microsoft Windows` (sa verzijom). Proxmox na osnovu ovoga bira podrazumevana podešavanja.

### System

| Polje | Preporuka |
|---|---|
| **Graphic card** | Default |
| **Machine** | **q35** (moderan čipset, PCIe) |
| **BIOS** | **OVMF (UEFI)** za moderne sisteme. Traži i **EFI Storage** (mali disk za UEFI promenljive) |
| **Pre-Enroll keys** | Uključeno, za **Secure Boot** |
| **SCSI Controller** | **VirtIO SCSI single** |
| **Qemu Agent** | **Uključeno** (agent instaliraš u gostu) |
| **Add TPM** | Uključi za **Windows 11** i Windows Server 2025 (TPM v2.0) |

### Disks

| Polje | Preporuka |
|---|---|
| **Bus/Device** | SCSI |
| **Storage** | Tvoj storage za diskove (local-lvm, ZFS, Ceph...) |
| **Disk size** | Prema potrebi. Kasnije može samo da se poveća |
| **Discard** | Uključeno |
| **IO thread** | Uključeno |
| **SSD emulation** | Uključeno na SSD storage-u |

(Detalji u Poglavlju 5, odeljak 5.8.)

### CPU

| Polje | Preporuka |
|---|---|
| **Sockets** | `1` (osim ako imaš poseban razlog) |
| **Cores** | Koliko vCPU-a treba. **Počni sa malo**, pa povećaj po potrebi |
| **Type** | Vidi odeljak 6.2 |

### Memory

| Polje | Preporuka |
|---|---|
| **Memory** | Potrebna memorija u MiB |
| **Minimum memory / Ballooning** | Vidi odeljak 6.3 |

### Network

- **Bridge**: `vmbr0` ili SDN VNet,
- **VLAN Tag** po planu mreže,
- **Model**: **VirtIO**,
- **Firewall**: po potrebi.

### Confirm

Proveri podešavanja i uključi **Start after created** ako želiš da se VM odmah pokrene.

---

## 6.2 Tip procesora (CPU type)

Ovo je jedna od **najvažnijih odluka za klaster**.

| Tip | Opis | Live migracija |
|---|---|---|
| **x86-64-v2-AES** (podrazumevano) | Generički model, podržan na skoro svim modernim procesorima | Da, između različitih procesora |
| **x86-64-v3** | Noviji skup instrukcija (AVX2...) | Da, ako svi čvorovi podržavaju v3 |
| **x86-64-v4** | AVX-512 | Samo na novijim serverskim procesorima |
| **host** | Gost vidi tačno procesor hosta. Najbolje performanse | **Samo između identičnih procesora** |
| Imenovani modeli (`EPYC-Genoa`, `SapphireRapids`...) | Određena generacija | Između iste ili novije generacije |

> **Pravilo za produkciju:** Izaberi **najviši nivo koji podržavaju SVI čvorovi** u klasteru (najčešće `x86-64-v2-AES` ili `x86-64-v3`). Ako staviš `host`, a klaster ima različite procesore, live migracija će pući ili će gost pasti posle migracije.

**Novo u 9.2:** prilagođeni (custom) modeli procesora mogu da se prave i menjaju iz web interfejsa, pa možeš da napraviš model tačno po meri svojih čvorova.

### Ostala CPU podešavanja

- **CPU limit**: ograničava koliko procesorskog vremena VM može da troši (npr. `2` = najviše 2 jezgra, bez obzira na broj vCPU-a),
- **CPU units**: relativan prioritet (veći broj dobija više procesora kada je host opterećen),
- **NUMA**: uključi na velikim VM-ovima na serverima sa više procesora (više NUMA čvorova).

> **Ne dodeljuj previše vCPU-a.** VM sa 16 vCPU-a koja koristi 2 može da radi **sporije** od VM sa 4 vCPU-a, jer host mora da rasporedi sve virtuelne procesore istovremeno. Ukupan broj vCPU-a na hostu sme da bude veći od broja fizičkih jezgara (overcommit), ali to prati.

---

## 6.3 Memorija

### Ballooning

Kada je uključen **Ballooning Device** i podešena **Minimum memory**, Proxmox može da "uzme" deo memorije od gosta kada hostu ponestaje RAM-a. Spušta je najniže do minimalne vrednosti.

- radi uz **balloon drajver** u gostu (u Linuxu je ugrađen, a za Windows je deo VirtIO drajvera),
- ako su **Memory** i **Minimum memory** iste, ballooning ne smanjuje memoriju, ali i dalje daje tačnije podatke o potrošnji.

### KSM (Kernel Samepage Merging)

KSM spaja identične stranice memorije između VM-ova (npr. više Windows VM-ova sa istim sistemskim fajlovima). Uključuje se automatski kada je host opterećen memorijom.

> **Produkcija:** Ne oslanjaj se na ballooning i KSM kao na rešenje za nedostatak RAM-a. Planiraj tako da zbir memorije gostiju, uz **rezervu za host** (sistem, ZFS ARC, Ceph), stane u fizički RAM. Ako host ostane bez memorije, **OOM killer** ubija procese, a najveći procesi su obično VM-ovi.

---

## 6.4 Linux VM: QEMU guest agent

Agent omogućava hostu da:

- prikaže IP adrese gosta u GUI-ju,
- uradi **ispravno gašenje** (*shutdown*),
- "zamrzne" fajl sistem pre backup-a i snapshota (**fs-freeze**), za konzistentne podatke.

Instalacija u gostu:

```bash
# Debian / Ubuntu
apt install qemu-guest-agent
systemctl enable --now qemu-guest-agent

# RHEL / Rocky / Alma
dnf install qemu-guest-agent
systemctl enable --now qemu-guest-agent
```

Na hostu proveri da li agent odgovara:

```bash
qm agent 100 ping
qm agent 100 network-get-interfaces
```

---

## 6.5 Windows VM

### Šta ti treba

1. Windows ISO,
2. **VirtIO drajveri** (`virtio-win.iso`) sa zvaničnog Fedora/virtio-win projekta. Učitaj ga na storage kao drugi ISO.

### Postupak

1. U čarobnjaku izaberi **Guest OS: Microsoft Windows** sa odgovarajućom verzijom.
2. **System**: q35, OVMF (UEFI), EFI disk, **TPM v2.0**, Qemu Agent.
3. **Disks**: SCSI, VirtIO SCSI single, Discard, IO thread.
4. **Network**: VirtIO.
5. Posle pravljenja: **Hardware → Add → CD/DVD Drive** i ubaci `virtio-win.iso`.
6. Pokreni VM. Kada instaler kaže da **ne vidi disk**, klikni **Load driver** i izaberi drajver sa VirtIO CD-a (`vioscsi\<verzija Windowsa>\amd64`).
7. Posle instalacije pokreni **`virtio-win-guest-tools.exe`** sa VirtIO CD-a. On instalira sve drajvere (mreža, balloon...) i **QEMU guest agent**.

### Secure Boot sertifikati 2023

Microsoft menja sertifikate koji se koriste za Secure Boot (stari sertifikati iz 2011. ističu). **Od 9.2** Proxmox omogućava da se **Microsoft UEFI CA 2023** sertifikati upišu u EFI disk postojećih VM-ova preko GUI-ja i API-ja. U produkciji proveri Windows VM-ove sa uključenim Secure Boot-om, da ne bi prestali da se pokreću posle ažuriranja.

---

## 6.6 Šabloni i kloniranje

### Šablon (template)

Šablon je VM koja služi samo kao osnova za pravljenje novih. Desni klik na VM → **Convert to template**. Šablon više ne može da se pokrene.

### Kloniranje

Desni klik na šablon → **Clone**:

| Tip | Opis | Kada koristiti |
|---|---|---|
| **Full clone** | Potpuno nezavisna kopija diska | **Produkcija** |
| **Linked clone** | Deli disk sa šablonom i čuva samo razlike. Brzo i štedi prostor | Laboratorija, test okruženja. Šablon ne sme da se obriše dok postoje klonovi |

### Cloud-init: automatska podešavanja

**Cloud-init** omogućava da pri prvom pokretanju VM-a automatski podesiš ime hosta, korisnika, SSH ključ, IP adresu i drugo. Većina Linux distribucija nudi gotove **cloud slike** (*cloud images*).

Primer: Ubuntu 24.04 šablon.

```bash
# 1. Preuzmi cloud sliku
cd /root
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

# 2. Napravi VM
qm create 9000 --name ubuntu-2404-tmpl --memory 2048 --cores 2 \
  --cpu x86-64-v2-AES --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-single \
  --ostype l26 --agent enabled=1

# 3. Uvezi sliku kao disk
qm set 9000 --scsi0 local-lvm:0,import-from=/root/noble-server-cloudimg-amd64.img,discard=on,iothread=1,ssd=1

# 4. Cloud-init disk, redosled butovanja i serijska konzola
qm set 9000 --ide2 local-lvm:cloudinit --boot order=scsi0 --serial0 socket --vga serial0

# 5. Cloud-init podešavanja
qm set 9000 --ciuser admin --sshkeys /root/.ssh/authorized_keys --ipconfig0 ip=dhcp

# 6. Pretvori u šablon
qm template 9000
```

Pravljenje nove VM iz šablona:

```bash
qm clone 9000 120 --name web01 --full
qm set 120 --ipconfig0 ip=10.0.100.20/24,gw=10.0.100.1 --nameserver 10.0.10.1
qm resize 120 scsi0 +18G
qm start 120
```

Ista podešavanja vidiš i menjaš u GUI-ju, na kartici **VM → Cloud-Init**.

> **Napomena:** Cloud slike ne sadrže uvek `qemu-guest-agent`. Instaliraj ga posle prvog pokretanja ili preko prilagođenog cloud-init *snippet*-a.

---

## 6.7 Snapshotovi

**VM → Snapshots → Take Snapshot**. Snapshot čuva stanje diskova (i opciono **RAM-a**, ako uključiš *Include RAM*) u tom trenutku.

```bash
qm snapshot 120 pre-update --description "Pre ažuriranja paketa"
qm listsnapshot 120
qm rollback 120 pre-update
qm delsnapshot 120 pre-update
```

> **Snapshot NIJE backup.** Snapshot se nalazi na istom storage-u kao i VM. Ako storage otkaže, nestaju i VM i snapshot.

**Produkcija:**

- pravi snapshot **pre rizičnih izmena** (ažuriranje, promena konfiguracije), a **obriši ga** kada potvrdiš da je sve u redu,
- ne čuvaj snapshotove danima i nedeljama, jer usporavaju rad (posebno dugi lanci) i zauzimaju prostor,
- snapshot sa RAM-om je zgodan, ali je velik i pravi se sporije.

---

## 6.8 Live migracija

Live migracija premešta **pokrenutu** VM na drugi čvor u klasteru bez prekida rada (samo kratka pauza na kraju, obično ispod sekunde).

**Desni klik na VM → Migrate** ili iz komandne linije:

```bash
qm migrate 120 pve2 --online
```

### Uslovi

- čvorovi su u **istom klasteru** (Poglavlje 8),
- **CPU tip** je kompatibilan na oba čvora (odeljak 6.2),
- isti **bridge/VNet** postoji na odredišnom čvoru,
- nema lokalnih ISO fajlova u CD drajvu, niti PCI passthrough uređaja.

### Deljeni i lokalni diskovi

| Storage | Šta se prenosi | Brzina |
|---|---|---|
| **Deljeni** (Ceph, NFS, SAN) | Samo memorija | Brzo |
| **Lokalni** (LVM-thin, ZFS) | Memorija **i ceo disk** (`--with-local-disks`) | Sporo za velike diskove |

**Novo u 9.3:** gost zadržava **trajni identitet** kroz migracije, backup i restore, što je važno za softver koji je licenciran ili vezan za identitet mašine.

---

## 6.9 Uvoz VM-ova sa VMware-a

Mnoge firme prelaze sa VMware-a na Proxmox. Proxmox ima ugrađen **ESXi import**:

1. **Datacenter → Storage → Add → ESXi**: unesi adresu ESXi hosta (ili vCenter-a) i korisnički nalog.
2. Novi storage prikazuje sve VM-ove sa ESXi-ja.
3. Izaberi VM → **Import**. Proxmox predlaže podešavanja (CPU, memorija, diskovi, mreža) koja možeš da izmeniš.
4. Posle uvoza:
   - za **Windows**: prvo pokreni sa SATA diskom, instaliraj VirtIO drajvere, pa prebaci disk na SCSI/VirtIO,
   - **ukloni VMware Tools** i instaliraj **QEMU guest agent**,
   - proveri mrežnu karticu (u gostu se pojavljuje kao nova kartica, pa IP adresu treba ponovo podesiti).

Uvoz **OVA/OVF** fajlova radi preko storage-a sa content tipom **Import** (storage → **Import** → izaberi fajl).

> **Produkcija:** Napravi **plan migracije** sa listom VM-ova, redosledom i vremenom zastoja. Prvo probaj na manje važnim VM-ovima. Proveri i **licence**, jer neki softver se veže za hardver ili hipervizor.

---

## 6.10 PCI passthrough (ukratko)

PCI passthrough daje VM-u **direktan pristup fizičkom uređaju**: grafičkoj kartici, mrežnoj kartici, HBA kontroleru.

Osnovni uslovi:

- **IOMMU** uključen u BIOS-u (Intel **VT-d** / AMD-Vi), i u kernelu ako nije podrazumevano uključen,
- uređaj u posebnoj IOMMU grupi.

**Datacenter → Resource Mappings** omogućava da isti uređaj definišeš na više čvorova, pa Proxmox zna gde VM sme da se pokrene.

> **Ograničenje:** VM sa PCI passthrough uređajem **ne može da se migrira uživo** i ne može da se automatski pokrene na drugom čvoru preko HA, osim ako i taj čvor ima mapiran isti tip uređaja.

---

## 6.11 Pokretanje i gašenje

**VM → Options**:

| Opcija | Preporuka za produkciju |
|---|---|
| **Start at boot** | Uključeno za sve servise koji treba da rade posle restarta hosta |
| **Start/Shutdown order** | Redosled pokretanja (npr. prvo DNS i baze, pa aplikacije), uz **Startup delay** |
| **Protection** | **Uključeno** za važne VM-ove. Sprečava slučajno brisanje VM-a i njenih diskova |
| **QEMU Guest Agent** | Uključeno, da gašenje bude ispravno |

U **VM → Summary** postoji polje **Notes** (podržava Markdown). Upiši vlasnika, namenu, kontakt i važne napomene. Budućim kolegama (i tebi za 6 meseci) ovo mnogo znači.

### Shutdown i Stop

- **Shutdown**: šalje gostu zahtev za gašenje (ACPI ili preko agenta). Gost se gasi uredno,
- **Stop**: isto kao izvlačenje kabla iz struje. Koristi samo kada gost ne odgovara.

---

## 6.12 Kontrolna lista: produkciona VM

- [ ] Ime i VMID po šemi, tagovi (`prod`, uloga), **Notes** sa vlasnikom i namenom
- [ ] q35, UEFI (OVMF), Secure Boot po potrebi, TPM za Windows 11 / Server 2025
- [ ] VirtIO SCSI single, disk sa Discard, IO thread i SSD emulation
- [ ] VirtIO mrežna kartica u ispravnom VLAN-u/VNet-u, firewall po potrebi
- [ ] CPU tip kompatibilan sa **svim** čvorovima klastera
- [ ] Razuman broj vCPU-a i memorije (bez preteranog dodeljivanja)
- [ ] **QEMU guest agent** instaliran i uključen
- [ ] **Start at boot** i redosled pokretanja podešeni
- [ ] **Protection** uključen
- [ ] VM je u **backup** poslu (Poglavlje 10) i restore je **isproban**
- [ ] VM je u **HA** ako je kritična (Poglavlje 8)
- [ ] Nema starih snapshotova i nema ISO fajla u CD drajvu

---

## 6.13 Česte greške u produkciji

| Greška | Posledica | Kako izbeći |
|---|---|---|
| CPU tip `host` u klasteru sa različitim procesorima | Migracija ne radi ili gost pada | Generički tip (`x86-64-v2-AES`/`v3`) |
| Bez guest agenta | Neuredno gašenje, nekonzistentan backup | Agent u svakoj VM |
| Snapshotovi koji stoje nedeljama | Sporiji rad, pun storage | Briši posle potvrde, prati |
| Snapshot umesto backup-a | Gubitak podataka kada storage otkaže | Pravi backup na drugom uređaju |
| Linked clone u produkciji | Ne možeš da obrišeš šablon, zavisnost | Full clone |
| ISO ostao u CD drajvu | Migracija ne radi ako ISO nije na deljenom storage-u | Izbaci ISO posle instalacije |
| Previše vCPU-a i RAM-a "za svaki slučaj" | Lošije performanse i manje mesta za druge goste | Kreni od manjeg, prati i povećaj |
| Windows bez VirtIO drajvera (IDE/E1000) | Loše performanse | VirtIO disk i mreža |
| Bez Protection opcije | Slučajno brisanje važne VM | Protection za produkcione VM-ove |

---

## 6.14 Vežba

1. Napravi **Linux VM** (Debian ili Ubuntu) iz ISO-a po preporukama iz ovog poglavlja. Instaliraj **qemu-guest-agent** i proveri `qm agent <id> ping`.
2. Napravi **Ubuntu cloud-init šablon** (odeljak 6.6) i iz njega napravi dve VM sa **full clone**: jednu sa DHCP-om, a drugu sa statičkom adresom.
3. Napravi **snapshot**, obriši neki fajl u VM-u, pa vrati snapshot (rollback). Na kraju obriši snapshot.
4. *(Ako imaš Windows ISO)* Napravi **Windows VM** sa VirtIO drajverima, TPM-om i UEFI-jem.
5. Uključi **Protection** na jednoj VM i pokušaj da je obrišeš.
6. Podesi **Start at boot** i redosled pokretanja za dve VM, restartuj host i proveri redosled.
7. *(Ako imaš 2 čvora, posle Poglavlja 8)* Uradi **live migraciju** sa `qm migrate --online`.

---

## 6.15 Rezime

- Moderna VM: **q35, UEFI, VirtIO SCSI single, VirtIO mreža, guest agent**, a za Windows 11 i TPM.
- **CPU tip** biraj prema **najslabijem** čvoru u klasteru. `host` samo za identične procesore.
- Ne preteruj sa vCPU-ima i memorijom. Ballooning i KSM nisu zamena za planiranje.
- **Cloud-init šabloni** i **full clone** su osnova brzog i ponovljivog pravljenja VM-ova.
- **Snapshot nije backup.** Pravi ga pre izmena i briši posle.
- Live migracija zahteva klaster, kompatibilan CPU i istu mrežu. Najbrža je sa deljenim storage-om.
- **ESXi import** olakšava prelazak sa VMware-a.
- **Novo:** custom CPU modeli i Secure Boot sertifikati 2023 iz GUI-ja (9.2), trajni identitet gosta (9.3).

---

## 6.16 Pitanja za proveru znanja

1. Zašto je CPU tip `host` problematičan u klasteru sa različitim procesorima?
2. Šta omogućava QEMU guest agent?
3. Šta je potrebno da bi Windows instaler video VirtIO SCSI disk?
4. Koja je razlika između full i linked clone-a i koji se koristi u produkciji?
5. Zašto snapshot nije backup?
6. Šta se prenosi pri live migraciji sa deljenim, a šta sa lokalnim storage-om?
7. Koje korake treba uraditi posle uvoza Windows VM-a sa ESXi-ja?
8. Čemu služi opcija **Protection**?

---

## Izvori

- [Proxmox VE Administration Guide (9.3.1): QEMU/KVM Virtual Machines](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)
- [What's new in Proxmox VE 9.3](https://www.proxmox.com/en/services/training-courses/videos/proxmox-virtual-environment/whats-new-in-proxmox-ve-9-3)
- [Proxmox VE 9.2 najava na forumu](https://forum.proxmox.com/threads/proxmox-virtual-environment-9-2-available.183741/latest)

---

**Prethodno poglavlje:** [Poglavlje 5: Skladišta podataka (storage)](05-skladista-podataka.md)
**Sledeće poglavlje:** [Poglavlje 7: Kontejneri (LXC)](07-kontejneri-lxc.md)
