# Poglavlje 7: Kontejneri (LXC)

> **Cilj poglavlja:** Kada završiš ovo poglavlje, znaćeš kada je kontejner bolji izbor od VM-a, kako se prave i podešavaju LXC kontejneri, koja je razlika između privilegovanih i neprivilegovanih kontejnera i kako da kontejnere bezbedno koristiš u produkciji.

---

## 7.1 Kada koristiti kontejner?

Podsetnik iz Poglavlja 1: kontejner **deli kernel hosta**, pa je mnogo lakši od VM-a, ali može da pokreće **samo Linux**.

| Koristi **kontejner** za | Koristi **VM** za |
|---|---|
| Web servere (nginx, Apache) | Windows |
| Baze (PostgreSQL, MariaDB), uz pažnju na storage | Sisteme kojima treba poseban kernel ili kernel moduli |
| DNS, DHCP, NTP, reverse proxy | **Docker / Kubernetes** (preporuka Proxmox-a) |
| Interne alate, monitoring, male servise | Aplikacije od kojih se traži jaka izolacija (npr. višekorisnička okruženja) |
| Mnogo malih instanci na ograničenom hardveru | Sisteme koji moraju da se migriraju **uživo** |

> **Pravilo:** Ako servis radi na Linuxu, ne traži poseban kernel i ne treba mu live migracija, **kontejner** je odličan izbor. U suprotnom koristi **VM**.

---

## 7.2 Šabloni (templates)

Kontejner se pravi iz **šablona**, arhive sa minimalnim root fajl sistemom distribucije.

### Preko web interfejsa

**Storage `local` → CT Templates → Templates**: izaberi distribuciju (Debian, Ubuntu, Alpine, Rocky, Alma, Fedora, Arch...) i klikni **Download**.

### Preko komandne linije

```bash
pveam update                         # osveži listu dostupnih šablona
pveam available --section system     # sistemski šabloni distribucija
pveam available --section turnkeylinux  # gotove aplikacije (TurnKey Linux)
pveam download local <ime-šablona>   # npr. debian-13-standard_..._amd64.tar.zst
pveam list local                     # preuzeti šabloni
```

> Tačna imena šablona (sa verzijom) vidiš u izlazu `pveam available`.

### OCI slike

Od verzije **9.1** Proxmox VE može da napravi kontejner i iz **OCI slike** (formata koji koriste Docker i Podman registri), za takozvane **aplikacione kontejnere**. Ovo je još novija funkcija. Pre upotrebe u produkciji proveri njen trenutni status i ograničenja u zvaničnoj dokumentaciji za 9.3.

---

## 7.3 Pravljenje kontejnera: čarobnjak

Klikni **Create CT** u gornjem desnom uglu.

### General

| Polje | Preporuka |
|---|---|
| **CT ID** | Po šemi (npr. `3xx` za kontejnere) |
| **Hostname** | `dns01` |
| **Unprivileged container** | **Uključeno** (podrazumevano). Vidi odeljak 7.4 |
| **Nesting** | Uključeno (podrazumevano). Potrebno za moderne distribucije sa systemd-om |
| **Password / SSH public key** | Za `root` u kontejneru. **Preporučuje se SSH ključ** |

### Template

Izaberi preuzeti šablon.

### Disks

- **rootfs**: storage i veličina (npr. `local-lvm`, 8 GB),
- dodatni **mount point**-ovi (odeljak 7.6).

### CPU i Memory

| Polje | Opis |
|---|---|
| **Cores** | Koliko jezgara kontejner vidi i koristi |
| **CPU limit / units** | Kao kod VM-a (Poglavlje 6) |
| **Memory** | Ograničenje RAM-a. Kontejner zauzima samo onoliko koliko stvarno koristi |
| **Swap** | Ograničenje swap-a za kontejner |

### Network

| Polje | Primer |
|---|---|
| **Name** | `eth0` |
| **Bridge** | `vmbr0` ili SDN VNet |
| **VLAN Tag** | Po planu |
| **IPv4** | **Static**: `10.0.100.30/24`, gateway `10.0.100.1`, ili **DHCP** |
| **IPv6** | Static / DHCP / SLAAC, po potrebi |
| **Firewall** | Po potrebi |

### DNS

Ako ostaviš prazno, kontejner preuzima DNS podešavanja hosta.

### Iz komandne linije

```bash
pct create 300 local:vztmpl/<debian-13-šablon>.tar.zst \
  --hostname dns01 \
  --unprivileged 1 --features nesting=1 \
  --cores 1 --memory 512 --swap 512 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=10.0.100.30/24,gw=10.0.100.1 \
  --ssh-public-keys /root/.ssh/authorized_keys \
  --onboot 1 --start 1
```

---

## 7.4 Privilegovani i neprivilegovani kontejneri

Ovo je **najvažnija bezbednosna odluka** kod kontejnera.

### Neprivilegovani (unprivileged): preporučeno

Korisnici u kontejneru se **mapiraju na neprivilegovane korisnike na hostu**:

```
U kontejneru          Na hostu
root (UID 0)    →     UID 100000
UID 1000        →     UID 101000
```

Ako napadač "pobegne" iz kontejnera kao root, na hostu je samo **običan korisnik bez prava**.

### Privilegovani (privileged)

`root` u kontejneru je **pravi root na hostu**. Zaštitu pružaju samo AppArmor profili i ograničenja mogućnosti (*capabilities*). Bekstvo iz kontejnera znači **potpunu kontrolu nad hostom**.

| | Neprivilegovani | Privilegovani |
|---|---|---|
| Bezbednost | **Visoka** | Niska |
| Root u kontejneru | UID 100000 na hostu | UID 0 (root) na hostu |
| Montiranje NFS/CIFS u kontejneru | Ne (montira se na hostu pa prosledi kao bind mount) | Da |
| Pristup uređajima hosta | Ograničen | Lakši |
| Upotreba u produkciji | **Da** | Samo uz jak razlog i samo za pouzdan softver |

> **Produkcija:** Koristi **isključivo neprivilegovane kontejnere**, osim ako imaš konkretan, dokumentovan razlog. Ako servis "ne radi" u neprivilegovanom kontejneru, prvo probaj sa odgovarajućim *feature*-om (odeljak 7.5) ili ga prebaci u VM.

---

## 7.5 Features: dodatne mogućnosti

**CT → Options → Features**:

| Feature | Namena |
|---|---|
| **nesting** | Dozvoljava ugnježdene namespace-ove. Potrebno za **systemd** u novijim distribucijama |
| **keyctl** | Pristup keyctl sistemskom pozivu (neki servisi ga traže, npr. Docker u neprivilegovanom kontejneru) |
| **fuse** | Dozvoljava FUSE fajl sisteme (npr. `sshfs`) |
| **mount** | Dozvoljava montiranje određenih tipova fajl sistema (npr. `nfs`, `cifs`), uglavnom za privilegovane kontejnere |
| **create device nodes** | Pravljenje uređaja u kontejneru |

> Uključuj samo ono što servisu stvarno treba. Svaki feature smanjuje izolaciju.

---

## 7.6 Mount points: dodatni diskovi i deljeni direktorijumi

### Mount point na storage-u (preporučeno)

Dodatni disk kontejnera, koji Proxmox vodi kao i rootfs:

**CT → Resources → Add → Mount Point**: storage, veličina, putanja u kontejneru (npr. `/var/lib/postgresql`).

```bash
pct set 300 -mp0 local-lvm:20,mp=/data
```

Ovakav mount point ulazi u **backup**, može da ima **snapshot** i radi sa **migracijom**.

### Bind mount: direktorijum sa hosta

Prosleđuje postojeći direktorijum hosta u kontejner:

```bash
pct set 300 -mp1 /mnt/pve/nas-share/www,mp=/var/www
```

Bind mount-ovi imaju ozbiljna ograničenja:

- **ne ulaze u backup** (`vzdump` ih preskače),
- **sprečavaju migraciju** (osim ako putanja postoji na svim čvorovima i označiš je kao `shared=1`),
- kod neprivilegovanih kontejnera fajlovi moraju da pripadaju **mapiranim UID-ovima** (npr. 100000 za root u kontejneru). U suprotnom kontejner ne može da piše u njih.

Primer: kontejner treba da piše u `/srv/data` na hostu kao korisnik sa UID 1000 u kontejneru:

```bash
chown -R 101000:101000 /srv/data
```

> **Produkcija:** Koristi **mount point-ove na storage-u** gde god možeš. Bind mount koristi samo za deljene podatke koji se čuvaju na drugom mestu (NAS), i dokumentuj ih.

---

## 7.7 Rad sa kontejnerima iz komandne linije

```bash
pct list                       # svi kontejneri na čvoru
pct start 300                  # pokretanje
pct shutdown 300               # uredno gašenje
pct stop 300                   # prinudno zaustavljanje
pct enter 300                  # shell u kontejneru (kao root)
pct exec 300 -- apt update     # izvrši komandu u kontejneru
pct push 300 ./nginx.conf /etc/nginx/nginx.conf   # kopiraj fajl u kontejner
pct pull 300 /var/log/syslog ./syslog-300         # kopiraj fajl iz kontejnera
pct resize 300 rootfs +10G     # povećaj disk
pct config 300                 # konfiguracija
pct snapshot 300 pre-update    # snapshot
```

### Konfiguracioni fajl

`/etc/pve/lxc/300.conf`:

```
arch: amd64
cores: 1
features: nesting=1
hostname: dns01
memory: 512
mp0: local-lvm:vm-300-disk-1,mp=/data,size=20G
net0: name=eth0,bridge=vmbr0,gw=10.0.100.1,hwaddr=BC:24:11:AA:BB:CC,ip=10.0.100.30/24,type=veth
onboot: 1
ostype: debian
rootfs: local-lvm:vm-300-disk-0,size=8G
swap: 512
unprivileged: 1
```

---

## 7.8 Docker u kontejneru?

Ovo je često pitanje. Docker **može** da radi u LXC kontejneru (uz `nesting` i `keyctl`), ali:

- dobijaš **dva sloja kontejnera** (LXC i Docker), koji ponekad ne rade dobro zajedno, naročito sa storage drajverima i AppArmor-om,
- **ažuriranje Proxmox-a, LXC-a ili Docker-a** može da pokvari postavku,
- zvanična preporuka Proxmox-a je da se **Docker pokreće u virtuelnoj mašini**.

> **Produkcija:** Docker, Podman i Kubernetes pokreći **u VM-u**. Ako želiš male aplikacione kontejnere direktno u Proxmox-u, pogledaj **OCI podršku** (odeljak 7.2) i njen trenutni status.

---

## 7.9 Migracija i backup kontejnera

### Migracija

Kontejneri **ne podržavaju live migraciju**. Koristi se **restart migracija**: kontejner se ugasi, prenese i ponovo pokrene na drugom čvoru.

```bash
pct migrate 300 pve2 --restart
```

Zastoj je kratak, obično nekoliko sekundi, ali postoji. Za servise kojima ni to nije prihvatljivo, napravi dva kontejnera na dva čvora (aktivni i rezervni, ili iza load balancer-a) ili koristi VM.

### Backup

Kontejneri se čuvaju istim backup sistemom kao VM-ovi (Poglavlje 10). Režimi:

| Režim | Kako radi | Zastoj |
|---|---|---|
| **Snapshot** | Snapshot storage-a, pa backup iz snapshota | Nema (traži storage sa snapshotovima: LVM-thin, ZFS, Ceph) |
| **Suspend** | Kratka pauza kontejnera | Kratak |
| **Stop** | Gašenje za vreme backup-a | Tokom celog backup-a |

---

## 7.10 Ažuriranje kontejnera

Kontejner je poseban sistem i **ažurira se iznutra**, nezavisno od hosta:

```bash
pct exec 300 -- bash -c "apt update && apt full-upgrade -y"
```

Za više kontejnera napravi skriptu ili koristi alat za automatizaciju (npr. Ansible), uz snapshot pre ažuriranja.

> **Pažnja:** Kada izađe nova verzija distribucije (npr. Debian 14), stari šabloni i kontejneri ostaju na staroj verziji. Nadogradnju verzije unutar kontejnera planiraj kao i na fizičkom serveru. Proveri i da li verzija Proxmox-a podržava novu distribuciju u kontejneru.

---

## 7.11 Kontrolna lista: produkcioni kontejner

- [ ] **Neprivilegovan** (`unprivileged: 1`)
- [ ] Samo potrebni features (najčešće samo `nesting`)
- [ ] Ime i CT ID po šemi, tagovi, **Notes**
- [ ] Root prijava preko **SSH ključa**, ne lozinke
- [ ] Ograničenja memorije, swap-a i CPU-a podešena
- [ ] Podaci na **mount point-ovima na storage-u**, a ne na bind mount-ovima
- [ ] **Start at boot**, redosled pokretanja i **Protection**
- [ ] U **backup** poslu, restore isproban
- [ ] Redovno ažuriranje unutar kontejnera
- [ ] Docker, Kubernetes i nepouzdan softver su u **VM**-u, a ne u kontejneru

---

## 7.12 Česte greške u produkciji

| Greška | Posledica | Kako izbeći |
|---|---|---|
| Privilegovani kontejner "jer je lakše" | Bekstvo iz kontejnera daje root na hostu | Neprivilegovani + potrebni features |
| Docker u LXC-u | Kvarovi posle ažuriranja | Docker u VM-u |
| Važni podaci na bind mount-u | Ne ulaze u backup | Mount point na storage-u ili poseban backup NAS-a |
| Pogrešni UID-ovi na bind mount-u | Kontejner ne može da piše | `chown` na mapirane UID-ove (100000+) |
| Očekivanje live migracije | Kratak prekid servisa pri migraciji | Plan za restart migraciju ili VM |
| Kontejneri se ne ažuriraju | Ranjivi servisi | Redovno ažuriranje, automatizacija |
| Bez ograničenja memorije i CPU-a | Jedan kontejner troši resurse svih ostalih | Ograničenja za svaki kontejner |

---

## 7.13 Vežba

1. Preuzmi šablon **Debian 13** i **Alpine** sa `pveam`.
2. Napravi **neprivilegovani** Debian kontejner `web01` sa 1 jezgrom, 512 MB RAM-a i statičkom IP adresom. Instaliraj `nginx` i otvori stranicu iz pregledača.
3. Uporedi potrošnju RAM-a ovog kontejnera i Linux VM-a iz Poglavlja 6 (stranica **Summary**).
4. Dodaj **mount point** od 5 GB na `/var/www` i premesti sajt na njega.
5. Napravi direktorijum na hostu, prosledi ga kao **bind mount** i probaj da pišeš u njega iz kontejnera. Zatim popravi dozvole sa `chown 100000:100000`.
6. Pogledaj `/etc/pve/lxc/<id>.conf` i objasni svaku liniju.
7. Iz komandne linije napravi kontejner komandom `pct create` (odeljak 7.3).
8. Na hostu pokreni `ps aux | grep nginx` i pogledaj pod kojim UID-om rade procesi kontejnera.

---

## 7.14 Rezime

- Kontejneri **dele kernel hosta**. Lakši su i brži, ali rade samo za Linux i nemaju live migraciju.
- Šabloni se preuzimaju sa `pveam` ili iz GUI-ja. Od 9.1 postoji podrška za **OCI slike**.
- **Neprivilegovani kontejneri** mapiraju root na UID 100000 i jedini su pravi izbor za produkciju.
- Features (`nesting`, `keyctl`, `fuse`...) uključuj samo po potrebi.
- **Mount point na storage-u** ulazi u backup i radi sa migracijom. **Bind mount** ne ulazi u backup i traži ispravne UID-ove.
- **Docker pokreći u VM-u**, a ne u LXC-u.
- Kontejneri se ažuriraju **iznutra**, kao zasebni sistemi.

---

## 7.15 Pitanja za proveru znanja

1. Navedi dva slučaja kada je kontejner bolji izbor od VM-a i dva kada nije.
2. Na koji UID na hostu se mapira root iz neprivilegovanog kontejnera i zašto je to važno?
3. Zašto je `nesting` potreban u većini modernih kontejnera?
4. Koja je razlika između mount point-a na storage-u i bind mount-a kod backup-a?
5. Kontejner ne može da piše u bind mount direktorijum. Šta je najverovatniji uzrok i kako ga rešavaš?
6. Kako se migrira kontejner i koliki je zastoj?
7. Zašto Proxmox preporučuje Docker u VM-u, a ne u LXC-u?

---

## Izvori

- [Proxmox VE Administration Guide (9.3.1): Proxmox Container Toolkit](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)
- [Proxmox VE dokumentacija, indeks](https://pve.proxmox.com/pve-docs/)
- [What's new in Proxmox VE 9.3](https://www.proxmox.com/en/services/training-courses/videos/proxmox-virtual-environment/whats-new-in-proxmox-ve-9-3)

---

**Prethodno poglavlje:** [Poglavlje 6: Virtuelne mašine (QEMU/KVM)](06-virtuelne-masine.md)
**Sledeće poglavlje:** [Poglavlje 8: Klaster i visoka dostupnost (HA)](08-klaster-i-ha.md)
