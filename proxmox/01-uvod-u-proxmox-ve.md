# Poglavlje 1: Uvod u Proxmox VE 9.3

> **Cilj poglavlja:** Kada završiš ovo poglavlje, znaćeš šta je Proxmox VE, kako rade dve vrste virtuelizacije koje podržava (KVM i LXC), šta donosi verzija 9.3 i kako da napraviš laboratoriju za vežbu kroz ostatak kursa.

---

## 1.1 Šta je Proxmox VE?

**Proxmox Virtual Environment (Proxmox VE)** je platforma otvorenog koda za virtuelizaciju servera. Na jednom mestu objedinjuje:

- **KVM** hipervizor za pune virtuelne mašine (VM),
- **LXC** za lagane Linux kontejnere,
- **softverski definisano skladište** (ZFS, LVM, Ceph i drugo),
- **softverski definisanu mrežu** (Linux bridge, VLAN, SDN).

Svim tim upravljaš kroz **web interfejs**, **komandnu liniju** ili **REST API**.

Proxmox VE razvija austrijska firma **Proxmox Server Solutions GmbH**. Softver je objavljen pod licencom **GNU AGPLv3**, pa je potpuno besplatan za korišćenje. Firma zarađuje na **pretplatama (subscription)**, koje donose pristup *Enterprise* repozitorijumu sa dodatno testiranim paketima i tehničkoj podršci. Za učenje i kućnu laboratoriju pretplata ti ne treba.

### Gde se Proxmox VE koristi?

- u malim i srednjim firmama kao zamena za VMware vSphere ili Microsoft Hyper-V,
- u kućnim laboratorijama (homelab) za učenje i testiranje,
- u hosting kompanijama i data centrima,
- za hiperkonvergentnu infrastrukturu (računanje i skladište na istim serverima, uz Ceph).

### Ključne osobine

| Osobina | Opis |
|---|---|
| Web interfejs | Upravljanje celim klasterom iz pregledača, bez dodatnog softvera |
| Klasterovanje | Više servera (čvorova, *nodes*) radi kao jedna celina, a konfiguracija se deli preko **pmxcfs** |
| Live migracija | Premeštanje VM-a sa jednog čvora na drugi bez gašenja |
| Visoka dostupnost (HA) | Automatsko pokretanje gostiju na drugom čvoru ako jedan otkaže |
| Ugrađeni backup | `vzdump` alat i integracija sa **Proxmox Backup Server**-om |
| Firewall | Ugrađen firewall na nivou data centra, čvora i gosta |
| Otvoren kod | Bez licenci po procesoru ili jezgru, bez ograničenja funkcija |

---

## 1.2 Osnovni pojmovi

Pre nego što krenemo dalje, dogovorimo se oko termina koje ćemo koristiti u celom kursu:

- **Host / čvor (node):** fizički server na kojem je instaliran Proxmox VE.
- **Klaster (cluster):** grupa čvorova koji se zajedno upravljaju.
- **Data centar (Datacenter):** najviši nivo u web interfejsu. Predstavlja ceo klaster, ili samo jedan čvor ako klaster ne postoji.
- **Gost (guest):** zajednički naziv za virtuelnu mašinu ili kontejner.
- **VMID:** jedinstveni broj svakog gosta u klasteru (npr. `100`, `101`...).
- **Storage (skladište):** mesto gde se čuvaju diskovi gostiju, ISO fajlovi, šabloni i backup-ovi.
- **Bridge (most):** virtuelni mrežni svič na hostu (podrazumevano `vmbr0`) na koji se kače gosti.

---

## 1.3 Dve vrste virtuelizacije: KVM i LXC

Proxmox VE podržava dva načina da pokreneš izolovane sisteme. Razumevanje razlike je verovatno najvažnija stvar u ovom poglavlju.

### KVM / QEMU: pune virtuelne mašine

**KVM** (*Kernel-based Virtual Machine*) je modul Linux kernela koji koristi hardversku podršku procesora za virtuelizaciju (Intel VT-x / AMD-V). **QEMU** emulira ostatak hardvera: diskove, mrežne kartice, BIOS/UEFI i slično.

Svaka VM ima **sopstveni kernel i operativni sistem**. Unutra možeš da pokreneš bilo šta: Windows, Linux, BSD...

```
+--------------------------------------------------+
|                 Fizički server                   |
|  +--------------------------------------------+  |
|  |        Proxmox VE (Debian + KVM)           |  |
|  |  +------------+  +------------+            |  |
|  |  |   VM 100   |  |   VM 101   |            |  |
|  |  |  Windows   |  |   Ubuntu   |            |  |
|  |  | (svoj      |  | (svoj      |            |  |
|  |  |  kernel)   |  |  kernel)   |            |  |
|  |  +------------+  +------------+            |  |
|  +--------------------------------------------+  |
+--------------------------------------------------+
```

### LXC: Linux kontejneri

**LXC** (*Linux Containers*) ne emulira hardver. Kontejneri **dele kernel hosta**, a izolacija se postiže mehanizmima kernela (*namespaces*, *cgroups*). Zato su kontejneri:

- **mnogo lakši:** troše manje RAM-a i diska,
- **brži:** pokreću se za par sekundi,
- **ograničeni na Linux:** pošto dele kernel hosta, unutra može da radi samo Linux distribucija.

### Poređenje

| | KVM virtuelna mašina | LXC kontejner |
|---|---|---|
| Operativni sistem | Bilo koji (Windows, Linux, BSD...) | Samo Linux |
| Kernel | Sopstveni | Deli kernel hosta |
| Potrošnja resursa | Veća | Mala |
| Vreme pokretanja | Desetine sekundi | Par sekundi |
| Izolacija | Jaka (hardverska) | Dobra, ali slabija od VM-a |
| Live migracija | Da | Ne (koristi se *restart* migracija) |
| Tipična upotreba | Windows serveri, sistemi sa posebnim kernelom, Docker hostovi | Web serveri, baze, DNS, mali Linux servisi |

> **Pravilo za početak:** Ako ti treba Windows, poseban kernel ili maksimalna izolacija, koristi **VM**. Ako ti treba lagan Linux servis, koristi **kontejner**.

---

## 1.4 Arhitektura Proxmox VE

Proxmox VE nije poseban operativni sistem napravljen od nule. To je **Debian GNU/Linux** sa dodatnim paketima. Zato na hostu imaš pun pristup shell-u i alatima koje već znaš iz Debiana (`apt`, `systemctl`, `ip`...).

Glavni delovi sistema:

| Komponenta | Uloga |
|---|---|
| **Debian 13 "Trixie"** | Osnovni operativni sistem |
| **Proxmox kernel** | Prilagođen Linux kernel sa podrškom za KVM, ZFS i drugo |
| **QEMU/KVM** | Pokretanje virtuelnih mašina |
| **LXC** | Pokretanje kontejnera |
| **pmxcfs** | Klasterski fajl sistem montiran na `/etc/pve`, gde se čuva konfiguracija |
| **Corosync** | Komunikacija između čvorova u klasteru i kvorum |
| **pveproxy** | Web server za GUI i API (port **8006**) |
| **pvedaemon** | Servis koji izvršava API zahteve |
| **pvestatd** | Prikupljanje statistike o gostima i skladištima |

Web interfejs i komandna linija koriste **isti REST API**. Sve što uradiš klikom u GUI-ju može da se uradi i komandom ili skriptom.

---

## 1.5 Šta je novo u Proxmox VE 9.3

Verzija **9.3** objavljena je **25. novembra 2026**. Ovaj kurs prati zvaničnu dokumentaciju za verziju **9.3.1**.

### Verzije glavnih komponenti

| Komponenta | Verzija u 9.3 |
|---|---|
| Debian | 13.8 "Trixie" |
| Linux kernel | 7.0 (podrazumevano), 7.3 opciono (*opt-in*) |
| QEMU | 11.1 |
| LXC | 7.0 |
| OpenZFS | 2.4 |
| Ceph | Tentacle 20.2 (stabilno), Umbrella 21.2 (tehnološki pregled) |

### Najvažnije novine

- **Bond i VLAN u instaleru:** mreža za upravljanje može da se podesi sa bond-om i VLAN-om već tokom instalacije (grafički, tekstualni i automatski instaler).
- **SAN prikaz za celi klaster:** novi prikaz *Datacenter → SAN* pokazuje stanje putanja (LUN) na svim čvorovima. Tu je i vođeno podešavanje deljenog LVM-a sa **multipath**-om za iSCSI, Fibre Channel i NVMe over Fabrics.
- **Snapshotovi na deljenom LVM-u za produkciju:** snapshotovi na *thick* deljenom LVM-u (preko *volume chain* mehanizma) više nisu tehnološki pregled kao u 9.0, nego su spremni za produkciju.
- **Trajni identitet gosta:** identitet gosta ostaje isti kroz migracije, backup i restore.
- **HA režim održavanja:** čvor može da se stavi u režim održavanja direktno iz GUI-ja i API-ja.
- **Bolji logovi:** strukturiran sistemski log sa filterima, novi log pristupa API-ju i panel sa istorijom APT ažuriranja.
- **Notifikacije:** pravila za obaveštenja mogu da se ugnježdavaju i slažu prevlačenjem (*drag-and-drop*).
- **Storage pluginovi trećih strana:** dodaju se i menjaju kroz web interfejs.
- **SDN:** *Simple* zone mogu da koriste VRF sa statičkim rutama, a status WireGuard fabrica vidi se u GUI-ju.

### Kratka istorija serije 9.x

| Verzija | Datum | Neke novine |
|---|---|---|
| 9.0 | 5. avgust 2025. | Prelazak na Debian 13 "Trixie", snapshotovi na deljenom LVM-u (tehnološki pregled), SDN fabrics |
| 9.2 | 21. maj 2026. | Kernel 7.0, QEMU 11.0, Ceph Tentacle kao podrazumevani, Dynamic Load Balancer, WireGuard u SDN-u, HA arm/disarm |
| 9.3 | 25. novembar 2026. | SAN prikaz, LVM snapshotovi za produkciju, bond/VLAN u instaleru, HA režim održavanja |

---

## 1.6 Laboratorija za vežbu

Za ovaj kurs ti treba okruženje u kojem možeš slobodno da praviš i brišeš stvari. Imaš tri opcije.

### Opcija A: Poseban fizički računar (preporučeno)

Najbolje iskustvo dobijaš na pravom hardveru. Dovoljan je i stariji desktop ili mini PC.

**Minimum za vežbu:**

- 64-bitni procesor (Intel ili AMD) sa podrškom za virtuelizaciju (**Intel VT-x / AMD-V**, uključeno u BIOS-u/UEFI-ju),
- **8 GB RAM-a** (16 GB ili više je mnogo udobnije),
- SSD od najmanje **128 GB**,
- jedna mrežna kartica (žičana mreža, Wi-Fi se ne preporučuje za host).

> **Napomena:** Instalacija **briše ceo disk** na koji instaliraš Proxmox VE. Ne instaliraj ga na računar sa podacima koji ti trebaju.

### Opcija B: Ugnježdena virtuelizacija (nested)

Proxmox VE možeš da instaliraš kao virtuelnu mašinu u VirtualBox-u, VMware Workstation-u, Hyper-V-u ili na postojećem Proxmox-u. Uslov je da uključiš **ugnježdenu virtuelizaciju** (*nested virtualization*), kako bi KVM radio i unutar te VM.

Ovo je odlično rešenje za vežbanje **klastera**: na jednom jačem računaru možeš da podigneš **3 Proxmox VM-a** i od njih napraviš klaster. Gosti unutra će biti sporiji, ali za učenje je to sasvim dovoljno.

### Opcija C: Kombinacija

Jedan fizički Proxmox host i na njemu nekoliko ugnježdenih Proxmox VM-ova za vežbe sa klasterom, HA i Ceph-om.

### Preporučena laboratorija za ovaj kurs

```
           Kućna mreža (npr. 192.168.1.0/24)
                         |
     +-------------------+-------------------+
     |                   |                   |
+---------+         +---------+         +---------+
|  pve1   |         |  pve2   |         |  pve3   |
| .1.11   |         | .1.12   |         | .1.13   |
+---------+         +---------+         +---------+
```

- Poglavlja **2 do 7** možeš da radiš na **jednom čvoru** (`pve1`).
- Za poglavlja **8 i 9** (klaster, HA, Ceph) trebaju ti **tri čvora**, fizička ili ugnježdena.

### Šta da pripremiš

1. Preuzmi **Proxmox VE 9.3 ISO** sa zvaničnog sajta: <https://www.proxmox.com/en/downloads>
2. Pripremi USB fleš od najmanje 2 GB (za fizičku instalaciju). Alati za pravljenje butabilnog USB-a: `dd` na Linuxu, Rufus ili balenaEtcher na Windowsu.
3. Odredi **statičke IP adrese** za svoje čvorove, gateway i DNS server.
4. Smisli **imena čvorova** (FQDN), npr. `pve1.lab.local`.

---

## 1.7 Kako pristupiti Proxmox VE

Posle instalacije (detaljno u Poglavlju 2) Proxmox VE-u pristupaš na tri načina:

1. **Web interfejs:** u pregledaču otvori `https://IP-ADRESA:8006`. Prijavljuješ se kao `root` sa realm-om **Linux PAM**.
2. **SSH:** `ssh root@IP-ADRESA`, za rad u shell-u.
3. **API:** `https://IP-ADRESA:8006/api2/json/...`, za automatizaciju (alat `pvesh` na samom hostu).

> Pregledač će prijaviti da sertifikat nije pouzdan. To je normalno, jer Proxmox posle instalacije koristi samopotpisani sertifikat. Kasnije ćemo videti kako da podesiš pravi sertifikat (npr. preko Let's Encrypt-a).

---

## 1.8 Rezime

- Proxmox VE je besplatna platforma otvorenog koda za virtuelizaciju, zasnovana na **Debianu**.
- Podržava **KVM virtuelne mašine** (bilo koji OS) i **LXC kontejnere** (samo Linux, lakši i brži).
- Svim se upravlja kroz **web interfejs na portu 8006**, CLI ili REST API.
- Verzija **9.3** (novembar 2026) zasniva se na Debianu 13.8 i kernelu 7.0, a donosi SAN prikaz, produkcione snapshotove na deljenom LVM-u i HA režim održavanja.
- Za kurs ti treba jedan host za osnove i tri čvora (mogu biti ugnježdeni) za klaster.

---

## 1.9 Pitanja za proveru znanja

1. Koja je glavna razlika između KVM virtuelne mašine i LXC kontejnera?
2. Da li u LXC kontejneru možeš da pokreneš Windows? Zašto?
3. Na kojoj Linux distribuciji je zasnovan Proxmox VE 9.3 i koja je to verzija?
4. Na kom portu radi web interfejs Proxmox VE-a?
5. Gde se u klasteru čuva konfiguracija i kako se zove taj fajl sistem?
6. Navedi tri novine u Proxmox VE 9.3.
7. Zašto je za vežbanje klastera potrebno najmanje tri čvora? (Nagoveštaj: kvorum, o njemu više u Poglavlju 8.)

---

## Izvori

- [Proxmox VE Administration Guide (9.3.1)](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)
- [Proxmox VE dokumentacija, indeks](https://pve.proxmox.com/pve-docs/)
- [What's new in Proxmox VE 9.3](https://www.proxmox.com/en/services/training-courses/videos/proxmox-virtual-environment/whats-new-in-proxmox-ve-9-3)
- [Proxmox VE 9.2 najava na forumu](https://forum.proxmox.com/threads/proxmox-virtual-environment-9-2-available.183741/latest)

---

**Sledeće poglavlje:** [Poglavlje 2: Instalacija i prvo podešavanje](02-instalacija-i-prvo-podesavanje.md)
