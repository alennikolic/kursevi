# Poglavlje 2: Instalacija i prvo podešavanje

> **Cilj poglavlja:** Kada završiš ovo poglavlje, imaćeš instaliran Proxmox VE 9.3 na svom računaru ili u VM-u, podešene repozitorijume za ažuriranje i ažuriran sistem spreman za rad.

---

## 2.1 Hardverski zahtevi

Zvanična dokumentacija razlikuje minimalne zahteve, dovoljne samo za probu, i preporučene zahteve za ozbiljan rad.

### Minimum (samo za probu i učenje)

- 64-bitni procesor (Intel 64 ili AMD64) sa podrškom za **Intel VT-x / AMD-V**,
- najmanje **1 GB RAM-a** za sam sistem, plus memorija za goste,
- disk (preporučuje se SSD),
- jedna mrežna kartica.

### Preporučeno (za produkciju)

- procesor sa više jezgara i podrškom za virtuelizaciju,
- **2 GB RAM-a** za sistem i servise Proxmox-a, plus memorija za goste. Ako koristiš **ZFS** ili **Ceph**, računaj dodatno oko **1 GB RAM-a po 1 TB** skladišta,
- brzi i redundantni diskovi, najbolje **SSD/NVMe**. Za ZFS koristi diskove bez hardverskog RAID kontrolera (ili kontroler u HBA/IT režimu),
- **više mrežnih kartica** (1 Gbit minimalno, 10 Gbit ili brže za klaster i Ceph).

> **Za ovaj kurs** je dovoljno ono što smo naveli u Poglavlju 1: 8 GB RAM-a (bolje 16 GB), SSD od 128 GB i jedna žičana mrežna kartica.

### Provera podrške za virtuelizaciju

Ako na računaru već imaš Linux, proveri podršku za virtuelizaciju ovom komandom:

```bash
grep -E -c '(vmx|svm)' /proc/cpuinfo
```

Ako je rezultat veći od `0`, procesor podržava virtuelizaciju. `vmx` označava Intel, a `svm` AMD. Ako je rezultat `0`, uključi virtuelizaciju u BIOS-u/UEFI-ju. Opcija se obično zove *Intel Virtualization Technology*, *VT-x*, *SVM Mode* ili *AMD-V*.

---

## 2.2 Priprema instalacionog medija

### Preuzimanje ISO fajla

1. Otvori <https://www.proxmox.com/en/downloads>.
2. Izaberi **Proxmox Virtual Environment → ISO Images**.
3. Preuzmi **Proxmox VE 9.3 ISO Installer**. Postoji i posebna verzija za **ARM64**, ali u ovom kursu koristimo standardnu x86-64 verziju.
4. Proveri **SHA256** sumu preuzetog fajla u odnosu na onu objavljenu na sajtu:

```bash
sha256sum proxmox-ve_9.3-*.iso
```

### Pravljenje butabilnog USB-a

> **Upozorenje:** Svi podaci na USB fleš uređaju biće obrisani.

**Linux** (zameni `/dev/sdX` pravim uređajem. Proveri ga sa `lsblk`, jer greška briše pogrešan disk!):

```bash
dd bs=1M conv=fdatasync if=./proxmox-ve_9.3-*.iso of=/dev/sdX
```

**Windows:** koristi **Rufus** ili **balenaEtcher**. Ako Rufus ponudi izbor, izaberi **DD režim**, jer ISO režim ne radi ispravno sa Proxmox instalerom.

**macOS:** koristi `dd` slično kao na Linuxu (uređaj je `/dev/rdiskN`, a proveravaš ga sa `diskutil list`) ili balenaEtcher.

### Instalacija u virtuelnu mašinu (nested)

Ako vežbaš u VirtualBox-u, VMware-u ili Hyper-V-u:

- napravi VM tipa **Linux / Debian (64-bit)**,
- dodeli joj najmanje **2 vCPU, 4 GB RAM-a i disk od 32 GB ili više**,
- **uključi ugnježdenu virtuelizaciju** (u VirtualBox-u: *Enable Nested VT-x/AMD-V*, u VMware-u: *Virtualize Intel VT-x/EPT or AMD-V/RVI*, u Hyper-V-u: `Set-VMProcessor -VMName <ime> -ExposeVirtualizationExtensions $true`),
- mrežnu karticu stavi u **Bridged** režim da bi Proxmox dobio adresu u tvojoj kućnoj mreži,
- ISO fajl ubaci kao virtuelni CD/DVD.

---

## 2.3 Pre instalacije: planiranje

Pre nego što pokreneš instaler, zapiši sledeće podatke:

| Podatak | Primer | Napomena |
|---|---|---|
| Ime hosta (FQDN) | `pve1.lab.local` | Mora da bude puno ime sa domenom |
| IP adresa / maska | `192.168.1.11/24` | Statička adresa, van DHCP opsega rutera |
| Gateway | `192.168.1.1` | Obično adresa rutera |
| DNS server | `192.168.1.1` ili `1.1.1.1` | |
| Lozinka za `root` | (jaka lozinka) | Koristi se za web interfejs i SSH |
| E-mail adresa | `admin@primer.rs` | Na nju stižu obaveštenja sistema |
| Disk za instalaciju | `/dev/sda` ili `/dev/nvme0n1` | **Biće potpuno obrisan** |

> **Savet:** Kasnije promene imena hosta i IP adrese moguće su, ali nisu jednostavne, naročito u klasteru. Zato ih dobro isplaniraj odmah.

---

## 2.4 Instalacija korak po korak

### Korak 1: Pokretanje instalera

Butuj računar sa USB-a. Obično pritiskaš `F11`, `F12`, `F8` ili `Esc` za meni za butovanje. Pojaviće se meni sa opcijama:

| Opcija | Kada se koristi |
|---|---|
| **Install Proxmox VE (Graphical)** | Standardna grafička instalacija. **Ovo biramo.** |
| **Install Proxmox VE (Terminal UI)** | Tekstualni instaler, ako grafički ne radi sa tvojom grafičkom karticom |
| **Install Proxmox VE (Terminal UI, Serial Console)** | Za servere bez monitora, preko serijske konzole |
| **Advanced Options** | Debug režim, instalacija bez nekih drajvera i slično |
| **Install Proxmox VE (Automated)** | Automatska instalacija iz unapred pripremljenog fajla odgovora (*answer file*) |

### Korak 2: Licenca (EULA)

Pročitaj i prihvati licencni ugovor klikom na **I agree**.

### Korak 3: Izbor diska i fajl sistema

Izaberi ciljni disk (*Target Harddisk*). Klikom na **Options** biraš fajl sistem:

| Fajl sistem | Opis | Kada koristiti |
|---|---|---|
| **ext4** (podrazumevano) | Klasičan Linux fajl sistem, uz LVM | Jedan disk, jednostavno i pouzdano |
| **xfs** | Alternativa za ext4, uz LVM | Slično kao ext4 |
| **ZFS (RAID0/1/10/RAIDZ-1/2/3)** | Napredni fajl sistem sa snapshotovima, kompresijom i proverom integriteta podataka | Dva ili više diskova, ozbiljniji rad |
| **BTRFS** (tehnološki pregled) | Fajl sistem sa sličnim mogućnostima kao ZFS | Samo za testiranje |

> **Za kurs:** Ako imaš **jedan disk**, izaberi **ext4**. Ako imaš **dva diska iste veličine**, probaj **ZFS (RAID1)**, gde se diskovi ogledaju jedan u drugom. Detaljnije o ZFS-u govorimo u Poglavlju 5.

#### Šta se dešava sa diskom kod ext4/xfs

Instaler pravi LVM grupu `pve` sa tri logička volumena:

- `root`: sistem (`/`),
- `swap`: swap prostor,
- `data`: **LVM-thin** pool za diskove virtuelnih mašina i kontejnera.

U **Advanced LVM Configuration Options** možeš ručno da podesiš veličine (`hdsize`, `swapsize`, `maxroot`, `minfree`, `maxvz`). Za početak ostavi podrazumevane vrednosti.

### Korak 4: Lokacija i vremenska zona

- **Country:** Serbia
- **Time zone:** Europe/Belgrade
- **Keyboard Layout:** po želji (npr. *U.S. English*, jer je za rad u shell-u najpraktičniji)

### Korak 5: Lozinka i e-mail

- unesi **lozinku za `root`** dva puta (najmanje 8 znakova, a preporučuje se mnogo duža),
- unesi **e-mail adresu** za sistemska obaveštenja.

### Korak 6: Mreža za upravljanje (Management Network)

Ovo je najvažniji korak. Ovde podešavaš adresu na kojoj ćeš pristupati web interfejsu.

| Polje | Primer |
|---|---|
| Management Interface | `enp1s0` (tvoja mrežna kartica) |
| Hostname (FQDN) | `pve1.lab.local` |
| IP Address (CIDR) | `192.168.1.11/24` |
| Gateway | `192.168.1.1` |
| DNS Server | `192.168.1.1` |

#### Novo u 9.3: bond i VLAN u instaleru

Od verzije 9.3 instaler omogućava da mrežu za upravljanje odmah podesiš kao:

- **bond:** spajanje više mrežnih kartica u jednu logičku vezu, radi redundanse ili veće brzine,
- **VLAN:** mreža za upravljanje se nalazi na određenom VLAN tag-u.

Ovo je korisno na serverima u data centru, gde mreža za upravljanje često radi baš preko bond-a i VLAN-a. Do sada se to radilo ručno, posle instalacije. **Za kućnu laboratoriju sa jednom karticom ovo ne koristimo**, nego biramo jednu karticu bez VLAN-a. Bond i VLAN detaljno obrađujemo u Poglavlju 4.

> **Savet:** Ako imaš više mrežnih kartica i nisi siguran koja je povezana, instaler obično prikazuje status veze (*link*) pored svake kartice.

### Korak 7: Pregled i instalacija

Na ekranu **Summary** proveri sve podatke. Opcija **Automatically reboot after successful installation** neka ostane uključena. Klikni **Install**.

Instalacija traje nekoliko minuta. Kada se završi, računar se restartuje. **Izvadi USB** da se instaler ne bi ponovo pokrenuo.

### Korak 8: Prvo pokretanje

Posle restarta, na konzoli će pisati poruka sličnu ovoj:

```
Welcome to the Proxmox Virtual Environment. Please use your web browser to
configure this server - connect to:

  https://192.168.1.11:8006/
```

---

## 2.5 Prvo prijavljivanje na web interfejs

1. Na **drugom računaru** u istoj mreži otvori pregledač i idi na `https://192.168.1.11:8006`.
2. Pregledač će upozoriti da **sertifikat nije pouzdan**. Prihvati rizik i nastavi, jer je to samopotpisani sertifikat koji je Proxmox sam napravio.
3. Prijavi se:
   - **User name:** `root`
   - **Password:** lozinka iz instalacije
   - **Realm:** `Linux PAM standard authentication`
   - **Language:** po želji (postoji i srpski, ali u kursu koristimo **English**, da bi se nazivi poklapali sa dokumentacijom)
4. Pojaviće se poruka **"No valid subscription"**. To je normalno kada nemaš pretplatu. Klikni **OK**. Sistem radi potpuno isto, a poruka samo podseća da postoji plaćena podrška.

---

## 2.6 Podešavanje repozitorijuma

Posle instalacije Proxmox VE je podešen da koristi **Enterprise** repozitorijum, kome bez pretplate nemaš pristup. Ako tako ostane, `apt update` će prijavljivati grešku `401 Unauthorized`. Zato prelazimo na **No-Subscription** repozitorijum.

### Vrste repozitorijuma

| Repozitorijum | Namena | Potrebna pretplata |
|---|---|---|
| **pve-enterprise** | Najstabilniji, dodatno testirani paketi, za produkciju | Da |
| **pve-no-subscription** | Besplatan, za testiranje i nekritične sisteme. Paketi su testirani, ali ne kao u enterprise | Ne |
| **pve-test** | Najnoviji paketi pre izlaska u ostale repozitorijume | Ne (ne koristi u produkciji) |

Isto važi i za **Ceph** repozitorijume (`enterprise`, `no-subscription`, `test`).

### Način 1: Preko web interfejsa (preporučeno)

1. U levom stablu izaberi svoj čvor (npr. **pve1**).
2. Idi na **Updates → Repositories**.
3. Označi **pve-enterprise** repozitorijum i klikni **Disable**. Isto uradi i za Ceph **enterprise** repozitorijum.
4. Klikni **Add**, izaberi **No-Subscription** i potvrdi sa **Add**.
5. Na vrhu stranice proveri status. Upozorenje da se no-subscription ne preporučuje za produkciju je očekivano.

### Način 2: Preko komandne linije

Proxmox VE 9 (kao i Debian 13) koristi **novi deb822 format** za repozitorijume: fajlove sa ekstenzijom `.sources` u direktorijumu `/etc/apt/sources.list.d/`, umesto starijih `.list` fajlova.

Pogledaj koje repozitorijume imaš:

```bash
ls /etc/apt/sources.list.d/
```

Isključi enterprise repozitorijum tako što ćeš u fajlu `pve-enterprise.sources` dodati liniju `Enabled: false`:

```bash
nano /etc/apt/sources.list.d/pve-enterprise.sources
```

```
Types: deb
URIs: https://enterprise.proxmox.com/debian/pve
Suites: trixie
Components: pve-enterprise
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
Enabled: false
```

Napravi fajl za no-subscription repozitorijum:

```bash
nano /etc/apt/sources.list.d/proxmox.sources
```

```
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
```

Na isti način isključi i Ceph enterprise repozitorijum u fajlu `ceph.sources`, ako postoji.

> **Napomena:** Tačna imena fajlova mogu malo da se razlikuju. Najsigurnije je da repozitorijume menjaš kroz web interfejs (Način 1), jer on uvek piše ispravan format.

---

## 2.7 Ažuriranje sistema

### Preko web interfejsa

1. Čvor → **Updates**.
2. Klikni **Refresh**, što je isto kao `apt update`.
3. Klikni **Upgrade**. Otvoriće se konzola u kojoj potvrđuješ instalaciju paketa.

### Preko komandne linije

Otvori shell (čvor → **Shell** u GUI-ju, ili preko SSH-a) i pokreni:

```bash
apt update
apt full-upgrade
```

> **Važno:** Na Proxmox VE uvek koristi **`apt full-upgrade`** (ili `apt dist-upgrade`), a **nikad samo `apt upgrade`**. Obična `upgrade` komanda ne instalira nove zavisnosti i ne uklanja stare pakete, što može da ostavi Proxmox u nekonzistentnom stanju.

Ako je ažuriran kernel, restartuj host:

```bash
reboot
```

### Provera verzije

```bash
pveversion
```

Primer izlaza:

```
pve-manager/9.3.1/... (running kernel: 7.0.x-x-pve)
```

Za detaljan spisak verzija svih komponenti:

```bash
pveversion -v
```

Verziju vidiš i u web interfejsu, u gornjem levom uglu i na stranici **čvor → Summary**.

---

## 2.8 Kratak obilazak posle instalacije

### Podrazumevana skladišta

U levom stablu, ispod čvora, videćeš dva skladišta (ako si izabrao ext4/xfs):

| Ime | Tip | Šta čuva |
|---|---|---|
| **local** | Directory (`/var/lib/vz`) | ISO fajlove, šablone kontejnera, backup-ove |
| **local-lvm** | LVM-thin | Diskove VM-ova i kontejnera |

Ako si izabrao ZFS, umesto `local-lvm` imaćeš **local-zfs**.

### Mrežna konfiguracija

Pogledaj fajl `/etc/network/interfaces`:

```bash
cat /etc/network/interfaces
```

Primer:

```
auto lo
iface lo inet loopback

iface enp1s0 inet manual

auto vmbr0
iface vmbr0 inet static
        address 192.168.1.11/24
        gateway 192.168.1.1
        bridge-ports enp1s0
        bridge-stp off
        bridge-fd 0
```

Vidi se da IP adresa **nije na fizičkoj kartici** (`enp1s0`), nego na **bridge-u `vmbr0`**. Fizička kartica je samo "port" tog virtuelnog sviča. Na isti bridge kasnije kačimo i VM-ove, pa oni dobijaju pristup istoj mreži. O tome detaljno u Poglavlju 4.

### Fajl `/etc/hosts`

Proxmox zahteva da se ime hosta razrešava u njegovu **pravu IP adresu** (ne u `127.0.1.1`):

```bash
cat /etc/hosts
```

```
127.0.0.1 localhost.localdomain localhost
192.168.1.11 pve1.lab.local pve1
```

### Sinhronizacija vremena

Proxmox VE koristi **chrony** za sinhronizaciju vremena. Tačno vreme je posebno važno u klasteru. Proveri status:

```bash
chronyc tracking
```

---

## 2.9 Česti problemi pri instalaciji

| Problem | Moguće rešenje |
|---|---|
| Instaler javlja da virtuelizacija nije dostupna (*No support for hardware-accelerated KVM virtualization*) | Uključi VT-x/AMD-V u BIOS-u. Kod nested instalacije uključi ugnježdenu virtuelizaciju |
| Grafički instaler ne radi ili je ekran crn | Probaj **Terminal UI** instaler |
| USB ne butuje | Napravi USB ponovo u **DD režimu**. Proveri redosled butovanja i Secure Boot podešavanja |
| Web interfejs se ne otvara | Proveri da li koristiš **https** i port **8006**, da li je IP adresa ispravna i da li su računari u istoj mreži |
| `apt update` javlja `401 Unauthorized` | Enterprise repozitorijum je uključen bez pretplate. Vidi odeljak 2.6 |
| Nema interneta na hostu | Proveri gateway i DNS (`ping 1.1.1.1`, pa `ping download.proxmox.com`) |

---

## 2.10 Vežba

1. Instaliraj Proxmox VE 9.3 na fizički računar ili u VM.
2. Prijavi se na web interfejs.
3. Isključi enterprise repozitorijume i dodaj **No-Subscription** repozitorijum.
4. Ažuriraj sistem sa `apt update && apt full-upgrade` i restartuj ga ako je stigao novi kernel.
5. Pokreni `pveversion -v` i zapiši verzije kernela, QEMU-a i LXC-a.
6. Pogledaj fajlove `/etc/network/interfaces` i `/etc/hosts` i objasni šta svaka linija radi.
7. *(Dodatno)* Ako planiraš klaster, ponovi instalaciju za `pve2` i `pve3`, sa različitim imenima i IP adresama.

---

## 2.11 Rezime

- Proxmox VE se instalira sa ISO-a napravljenog kao butabilni USB (DD režim) ili u VM-u sa ugnježdenom virtuelizacijom.
- Najvažniji podaci za instalaciju su **FQDN, statička IP adresa, gateway i DNS**.
- Za jedan disk koristi **ext4**, a za dva ili više razmisli o **ZFS-u**.
- Verzija 9.3 omogućava **bond i VLAN** za mrežu za upravljanje već u instaleru.
- Bez pretplate prebaci se na **No-Subscription** repozitorijum. PVE 9 koristi **deb822 `.sources`** format.
- Sistem ažuriraj sa **`apt full-upgrade`**, nikad samo sa `apt upgrade`.

---

## 2.12 Pitanja za proveru znanja

1. Kako proveravaš da li procesor podržava hardversku virtuelizaciju?
2. Zašto Rufus treba da koristi DD režim pri pravljenju USB-a?
3. Koja tri logička volumena pravi instaler kada izabereš ext4?
4. Zašto se IP adresa hosta nalazi na `vmbr0`, a ne na fizičkoj kartici?
5. Koja je razlika između `pve-enterprise` i `pve-no-subscription` repozitorijuma?
6. Zašto na Proxmox-u ne treba koristiti `apt upgrade`?
7. Koju novinu za mrežu donosi instaler u verziji 9.3 i kada je korisna?

---

## Izvori

- [Proxmox VE Administration Guide (9.3.1): Installing Proxmox VE](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)
- [Proxmox VE dokumentacija, indeks](https://pve.proxmox.com/pve-docs/)
- [What's new in Proxmox VE 9.3](https://www.proxmox.com/en/services/training-courses/videos/proxmox-virtual-environment/whats-new-in-proxmox-ve-9-3)
- [Proxmox VE preuzimanje](https://www.proxmox.com/en/downloads)

---

**Prethodno poglavlje:** [Poglavlje 1: Uvod u Proxmox VE 9.3](01-uvod-u-proxmox-ve.md)
**Sledeće poglavlje:** [Poglavlje 3: Administracija hosta i web interfejs](03-administracija-hosta-i-web-interfejs.md)
