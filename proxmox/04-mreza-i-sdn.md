# Poglavlje 4: Mrežna podešavanja i SDN

> **Cilj poglavlja:** Kada završiš ovo poglavlje, razumećeš kako Proxmox VE povezuje goste sa mrežom (bridge, VLAN, bond) i kako se mreže razdvajaju u produkciji. Znaćeš i kako da koristiš **Software-Defined Network (SDN)** za upravljanje mrežama na nivou celog klastera.

---

## 4.1 Osnovni pojmovi

Proxmox VE koristi standardnu Linux mrežu. Konfiguracija se nalazi u `/etc/network/interfaces`, a primenjuje je alat **ifupdown2**.

| Pojam | Šta je | Ime u sistemu (primer) |
|---|---|---|
| **Fizički interfejs** | Mrežna kartica | `eno1`, `enp1s0`, `ens18` |
| **Bridge** | Virtuelni svič na hostu. Na njega se kače fizička kartica i gosti | `vmbr0`, `vmbr1` |
| **Bond** | Više fizičkih kartica spojenih u jednu logičku, radi redundanse ili brzine | `bond0` |
| **VLAN** | Logička podela jedne fizičke mreže pomoću oznaka (tagova) 1 do 4094 | `vmbr0.10`, `bond0.20` |
| **tap / veth** | Virtuelna kartica gosta (VM ima `tap`, kontejner ima `veth`) | `tap100i0`, `veth200i0` |

### Kako izgleda tok saobraćaja

```
             Fizički svič
                  |
              [ eno1 ]           ← fizička kartica (bez IP adrese)
                  |
   +-------------[vmbr0]-------------+   ← bridge (IP hosta: 192.168.1.11)
   |              |                  |
[tap100i0]    [tap101i0]        [veth200i0]
   |              |                  |
 VM 100        VM 101            CT 200
```

Bridge radi kao običan svič: gosti dobijaju adrese u **istoj mreži** kao host i direktno komuniciraju sa ostatkom mreže.

---

## 4.2 Upravljanje mrežom kroz web interfejs

**Čvor → System → Network**.

1. Klikni **Create** i izaberi tip (Linux Bridge, Linux Bond, Linux VLAN, OVS...).
2. Popuni polja i klikni **Create**.
3. Promene se **ne primenjuju odmah**. Upisuju se u `/etc/network/interfaces.new`, a na dnu stranice vidiš razliku (*Pending changes*).
4. Klikni **Apply Configuration**. Promene se primenjuju bez restarta hosta.

Iz komandne linije, posle ručne izmene fajla `/etc/network/interfaces`:

```bash
ifreload -a
```

> **Upozorenje:** Greška u mreži za upravljanje može da te **zaključa van servera**. Pre izmena u produkciji obezbedi pristup konzoli preko IPMI/iDRAC/iLO, ili radi sa nekog drugog interfejsa.

---

## 4.3 Bond: redundansa i brzina

Bond spaja dve ili više kartica. Ako jedna kartica (ili kabl, ili port na sviču) otkaže, saobraćaj nastavlja preko druge.

### Najčešći režimi

| Režim | Opis | Zahteva podešavanje sviča |
|---|---|---|
| **active-backup** | Radi samo jedna kartica, druga čeka kao rezerva | Ne |
| **802.3ad (LACP)** | Sve kartice rade istovremeno. Najbolje rešenje za produkciju | **Da** (LACP na sviču, a za dva sviča MLAG/stack) |
| **balance-alb** | Raspodela opterećenja bez podrške sviča | Ne |
| balance-rr, balance-xor, broadcast | Ređe se koriste | Zavisi |

### Primer: LACP bond sa VLAN-aware bridge-om

```
auto eno1
iface eno1 inet manual

auto eno2
iface eno2 inet manual

auto bond0
iface bond0 inet manual
        bond-slaves eno1 eno2
        bond-miimon 100
        bond-mode 802.3ad
        bond-xmit-hash-policy layer3+4

auto vmbr0
iface vmbr0 inet manual
        bridge-ports bond0
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes
        bridge-vids 2-4094

# Mreža za upravljanje na VLAN-u 10
auto vmbr0.10
iface vmbr0.10 inet static
        address 10.0.10.11/24
        gateway 10.0.10.1
```

Šta se ovde dešava:

- `bond0` spaja `eno1` i `eno2` preko LACP-a,
- `vmbr0` je **VLAN-aware** bridge na bond-u, pa gosti mogu da budu u bilo kom VLAN-u od 2 do 4094,
- IP adresa hosta je na **VLAN-u 10** (`vmbr0.10`), a ne direktno na bridge-u.

> **Novo u 9.3:** Bond i VLAN za mrežu za upravljanje mogu da se podese već u instaleru (videti Poglavlje 2), pa ovakvu konfiguraciju dobijaš odmah posle instalacije.

---

## 4.4 VLAN: razdvajanje mreža

U produkciji gosti obično ne treba da budu u istoj mreži kao host. VLAN-ovi to rešavaju preko istog fizičkog kabla.

### Način 1: VLAN-aware bridge (preporučeno)

Bridge podesiš sa `bridge-vlan-aware yes` (kao u primeru iznad). Zatim u mrežnoj kartici gosta upišeš **VLAN Tag**:

**VM → Hardware → Network Device → Edit → VLAN Tag: `20`**

Gost se tako nalazi u VLAN-u 20, a port na fizičkom sviču mora da bude **trunk** koji propušta taj VLAN.

### Način 2: Poseban bridge po VLAN-u

Stariji pristup, gde za svaki VLAN praviš poseban bridge (`vmbr20` na `bond0.20`). Radi, ali je teže za održavanje. Ako ti trebaju imenovani VLAN-ovi za celi klaster, bolje je da koristiš **SDN** (odeljak 4.8).

---

## 4.5 Imena mrežnih kartica

Linux daje karticama "predvidiva" imena (`eno1`, `enp3s0f0`...) na osnovu položaja na hardveru. Ta imena mogu da se **promene** posle ažuriranja kernela ili firmware-a, ili posle dodavanja nove kartice. Tada `/etc/network/interfaces` pokazuje na nepostojeći interfejs i **host ostaje bez mreže**.

Ovo je čest problem u produkciji posle nadogradnje. Zaštita:

- Proxmox VE 9 ima alat **`proxmox-network-interface-pinning`**, koji karticama dodeljuje stalna imena vezana za MAC adresu,
- alternativa je ručno pravljenje `.link` fajlova u `/etc/systemd/network/`,
- posle svake veće nadogradnje proveri imena kartica sa `ip link` **pre** restarta, ako release notes upozoravaju na promene.

```bash
# Primer: dodeli stalna imena svim karticama
proxmox-network-interface-pinning generate
```

> Pre upotrebe pročitaj pomoć alata (`--help`) i dokumentaciju, jer alat menja i mrežnu konfiguraciju.

---

## 4.6 Mrežna kartica gosta

**VM → Hardware → Add → Network Device**:

| Opcija | Preporuka |
|---|---|
| **Bridge** | `vmbr0` ili SDN VNet |
| **VLAN Tag** | Prema planu mreže |
| **Model** | **VirtIO (paravirtualized)**: najbrži. Za Windows su potrebni VirtIO drajveri (Poglavlje 6) |
| **Firewall** | Uključi, ako koristiš Proxmox firewall za goste |
| **Rate limit** | Ograničenje brzine u MB/s, korisno da jedan gost ne zauzme celu vezu |
| **Multiqueue** | Za VM-ove sa velikim mrežnim saobraćajem, postavi na broj vCPU-a |
| **MTU** | Ostavi isto kao bridge (vrednost `1` znači "preuzmi od bridge-a") |

---

## 4.7 Razdvajanje mreža u produkciji

Ovo je **najvažniji deo poglavlja** za produkciju. Produkcioni klaster ima nekoliko vrsta saobraćaja, i oni **ne smeju da smetaju jedni drugima**.

| Mreža | Namena | Zahtevi |
|---|---|---|
| **Upravljanje (management)** | Web interfejs, SSH, API | Dostupna samo administratorima. Može 1 Gbit |
| **Klaster (corosync)** | Komunikacija čvorova i kvorum | **Veoma malo kašnjenje**, stabilna. Najbolje posebne kartice. Propusni opseg nije bitan |
| **Storage** | Ceph, NFS, iSCSI | Velika brzina (10/25 Gbit i više), često jumbo frames (MTU 9000) |
| **Migracija** | Prenos memorije pri live migraciji | Velika brzina. Može da deli mrežu sa storage-om |
| **Gosti** | Saobraćaj VM-ova i kontejnera | Prema potrebama, obično VLAN-ovi na bond-u |

### Primer rasporeda za čvor sa 4 kartice

```
eno1 ─────────────── corosync link 0 (10.0.1.0/24), bez bond-a
eno2 ─────────────── corosync link 1 (10.0.2.0/24), bez bond-a
ens1f0 ┐
       ├─ bond0 (LACP) ─ vmbr0 (VLAN-aware)
ens1f1 ┘                 ├─ VLAN 10: upravljanje
                         ├─ VLAN 30: storage / migracija (MTU 9000)
                         └─ VLAN 100-199: gosti
```

### Zašto corosync ne ide preko istog bond-a?

Corosync je **veoma osetljiv na kašnjenje**. Ako backup ili migracija zaguše mrežu, corosync gubi vezu između čvorova. Čvor tada može da izgubi kvorum, a uz HA se čak **sam restartuje** (fencing). Zato:

- corosync ide na **posebne kartice** (ili barem poseban VLAN sa QoS-om),
- umesto bond-a za corosync koristi **više corosync linkova** (corosync sam prelazi na drugi link ako prvi otkaže). Podešava se pri pravljenju klastera (Poglavlje 8).

### Mreža za migraciju

**Datacenter → Options → Migration Settings**: izaberi mrežu (npr. storage VLAN) preko koje ide live migracija. Tako migracija ne opterećuje mrežu za upravljanje.

### Jumbo frames (MTU 9000)

Za storage mreže veći MTU smanjuje opterećenje procesora i povećava brzinu. **MTU mora da bude isti na celom putu**: na karticama, bond-u, bridge-u, VLAN-u i na **svim portovima sviča**. Ako se negde razlikuje, dešavaju se čudne greške (manji paketi prolaze, a veliki ne).

Provera:

```bash
# 8972 = 9000 - 28 bajtova zaglavlja; -M do zabranjuje fragmentaciju
ping -M do -s 8972 10.0.30.12
```

---

## 4.8 Software-Defined Network (SDN)

Do sada smo mrežu podešavali **na svakom čvoru posebno**. Kod klastera od 5 ili 10 čvorova to postaje teško i podložno greškama. **SDN** omogućava da mreže definišeš **jednom, na nivou data centra**, a Proxmox ih primenjuje na sve čvorove.

SDN je od verzije 8.1 instaliran podrazumevano. Konfiguracija se čuva u `/etc/pve/sdn/`, pa je ista na celom klasteru.

### Delovi SDN-a

| Deo | Opis |
|---|---|
| **Zona (Zone)** | Tip mreže i način na koji se saobraćaj prenosi između čvorova |
| **VNet** | Virtuelna mreža u zoni. Gosti se kače na VNet kao na bridge |
| **Subnet** | IP opseg u VNet-u, sa gateway-om, opcionim SNAT-om i DHCP-om |
| **IPAM** | Upravljanje IP adresama (ugrađeni PVE IPAM, NetBox, phpIPAM) |
| **DNS** | Automatsko upisivanje imena gostiju u DNS (PowerDNS) |
| **Kontroleri (Controllers)** | EVPN, BGP, IS-IS, za napredno rutiranje |
| **Fabrics** | Mreža između čvorova (underlay): OpenFabric, OSPF, a od 9.2 i **WireGuard** |

### Tipovi zona

| Zona | Opis | Kada koristiti |
|---|---|---|
| **Simple** | Izolovan bridge na svakom čvoru, bez veze između čvorova. Podržava SNAT i DHCP | Laboratorija, izolovane mreže, NAT za goste. **Od 9.3 podržava VRF sa statičkim rutama** |
| **VLAN** | VNet je VLAN na postojećem bridge-u | **Najčešći izbor u produkciji** sa klasičnim svičevima |
| **QinQ** | VLAN u VLAN-u (802.1ad) | Provajderi, veliki broj izolovanih mreža |
| **VXLAN** | Layer 2 mreža preko Layer 3 (UDP tunel) | Čvorovi u različitim mrežama ili lokacijama |
| **EVPN** | VXLAN sa BGP kontrolerom i rutiranjem | Veliki klasteri, multi-tenant okruženja. Od 9.2 ima filtriranje sa *route maps* i *prefix lists* |

### Primer 1: VLAN zona (produkcija)

Cilj: VLAN-ovi 100 (web) i 110 (baze), dostupni na svim čvorovima pod istim imenima.

1. **Datacenter → SDN → Zones → Add → VLAN**
   - ID: `prod`
   - Bridge: `vmbr0` (mora da postoji na svim čvorovima i da bude VLAN-aware)
2. **Datacenter → SDN → VNets → Create**
   - Name: `web`, Zone: `prod`, Tag: `100`
3. Ponovi za `db` sa tagom `110`.
4. **Datacenter → SDN → Apply**. Ovo je obavezan korak, bez njega se ništa ne primenjuje!
5. Kod gosta, u mrežnoj kartici izaberi bridge **`web`** ili **`db`**. VLAN tag više ne upisuješ ručno.

Prednosti: imena mreža su jasna (`web` umesto "VLAN 100"), ista su na svim čvorovima, a dozvole možeš da daješ po VNet-u (npr. tim za baze vidi samo `db`).

### Primer 2: Simple zona sa NAT-om i DHCP-om (laboratorija)

Cilj: izolovana mreža `10.99.0.0/24` u kojoj gosti dobijaju adrese preko DHCP-a i izlaze na internet preko NAT-a na hostu.

1. Instaliraj dnsmasq (potreban za DHCP) i isključi njegov podrazumevani servis:

   ```bash
   apt install dnsmasq
   systemctl disable --now dnsmasq
   ```

2. **Zones → Add → Simple**: ID `lab`, IPAM `pve`, uključi **automatic DHCP**.
3. **VNets → Create**: Name `labnet`, Zone `lab`.
4. Izaberi `labnet` → **Subnets → Create**:
   - Subnet: `10.99.0.0/24`, Gateway: `10.99.0.1`, uključi **SNAT**,
   - DHCP Ranges: `10.99.0.100` – `10.99.0.200`.
5. **SDN → Apply**.

### Status i provera

- **Datacenter → SDN** prikazuje status zona po čvorovima. Ako primena ne uspe, tu vidiš na kom čvoru i zašto.
- **Od 9.3** u GUI-ju se vidi i **status WireGuard fabric-a**.
- Fajl `/etc/network/interfaces` mora da sadrži liniju `source /etc/network/interfaces.d/*`, jer tamo SDN upisuje svoju konfiguraciju. Na novim instalacijama ova linija već postoji, ali je proveri na starijim, nadograđenim sistemima.

---

## 4.9 Česte greške u produkciji

| Greška | Posledica | Kako izbeći |
|---|---|---|
| Izmena mreže za upravljanje bez pristupa konzoli | Zaključan si van servera | IPMI/iDRAC/iLO pristup pre svake izmene |
| Corosync na istoj mreži sa storage-om i backup-om | Gubitak kvoruma, HA restartuje čvorove | Posebne kartice i više corosync linkova |
| LACP podešen na hostu, ali ne i na sviču | Gubitak paketa ili mreže | Uskladi podešavanja sa mrežnim timom, testiraj izvlačenjem kabla |
| Različit MTU na putu | Spor ili nepouzdan storage | MTU 9000 svuda na putu, provera sa `ping -M do` |
| Imena kartica promenjena posle nadogradnje | Host bez mreže posle restarta | Pinning imena, čitanje release notes |
| Zaboravljen **SDN → Apply** | Nova mreža ne postoji na čvorovima | Uvek Apply i provera statusa |
| Bridge nije VLAN-aware, a gost ima VLAN tag | Gost nema mrežu | `bridge-vlan-aware yes` |
| Trunk na sviču ne propušta VLAN | Gost nema mrežu samo na nekim čvorovima | Isti trunk na portovima **svih** čvorova |
| Svi gosti u mreži za upravljanje | Kompromitovan gost ima direktan pristup hipervizoru | Gosti u svojim VLAN-ovima |

---

## 4.10 Vežba

1. Pogledaj `/etc/network/interfaces` na svom čvoru i nacrtaj šemu mreže.
2. Napravi novi bridge **`vmbr1` bez fizičke kartice** (izolovana interna mreža). Pogledaj *Pending changes* pa primeni sa **Apply Configuration**.
3. Uključi **VLAN aware** na `vmbr0` (ako tvoj kućni svič/ruter ne podržava VLAN, to i dalje radi za netagovan saobraćaj).
4. Napravi **Simple** SDN zonu `lab` sa VNet-om `labnet`, subnetom `10.99.0.0/24`, SNAT-om i DHCP-om, prema Primeru 2. Primeni i proveri status.
5. *(Ako imaš 2 kartice ili nested okruženje)* Napravi **active-backup** bond i testiraj ga: dok traje `ping`, isključi jednu karticu.
6. Kao vežbu na papiru, napravi plan mreže za produkcioni klaster od 3 čvora sa po 4 kartice: koje mreže, koji VLAN-ovi, koji MTU i gde ide corosync.

---

## 4.11 Rezime

- Gosti se kače na **bridge** (`vmbr0`), koji radi kao virtuelni svič. IP adresa hosta je na bridge-u ili VLAN interfejsu, a ne na fizičkoj kartici.
- **Bond** daje redundansu. **LACP (802.3ad)** je najbolji izbor, ali zahteva podešavanje sviča.
- **VLAN-aware bridge** i VLAN tag na kartici gosta su osnova razdvajanja mreža.
- U produkciji razdvoji **upravljanje, corosync, storage, migraciju i goste**. **Corosync** traži posebnu, stabilnu mrežu i više linkova.
- **MTU 9000** pomaže storage-u, ali mora da bude isti na celom putu.
- **SDN** definiše mreže jednom za ceo klaster: **zone, VNet-ovi i subneti**. Ne zaboravi **Apply**.
- **Novo:** WireGuard fabric i BGP/EVPN filtriranje (9.2), VRF u Simple zonama i status WireGuard fabric-a u GUI-ju (9.3).

---

## 4.12 Pitanja za proveru znanja

1. Zašto IP adresa hosta nije na fizičkoj kartici?
2. Koja je razlika između režima bond-a `active-backup` i `802.3ad`?
3. Šta znači `bridge-vlan-aware yes` i šta se podešava na sviču da bi to radilo?
4. Zašto corosync ne treba da deli mrežu sa storage-om i backup-om?
5. Kako proveravaš da li jumbo frames rade na celom putu?
6. Koja je razlika između zona tipa Simple, VLAN i VXLAN?
7. Šta se desi ako napraviš VNet, a ne klikneš **Apply**?
8. Zašto je promena imena mrežne kartice opasna i kako se od nje zaštititi?

---

## Izvori

- [Proxmox VE Administration Guide (9.3.1): Network Configuration, Software-Defined Network](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)
- [What's new in Proxmox VE 9.3](https://www.proxmox.com/en/services/training-courses/videos/proxmox-virtual-environment/whats-new-in-proxmox-ve-9-3)
- [Proxmox VE 9.2 najava na forumu](https://forum.proxmox.com/threads/proxmox-virtual-environment-9-2-available.183741/latest)

---

**Prethodno poglavlje:** [Poglavlje 3: Administracija hosta i web interfejs](03-administracija-hosta-i-web-interfejs.md)
**Sledeće poglavlje:** [Poglavlje 5: Skladišta podataka (storage)](05-skladista-podataka.md)
