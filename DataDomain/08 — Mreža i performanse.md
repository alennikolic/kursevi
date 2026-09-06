# 08 — Mreža i performanse

Mrežna konfiguracija, merenje opterećenja i dijagnostika kada je backup ili
replikacija sporija nego što bi trebalo.

---

## 8.1 Pregled mrežne konfiguracije

```
net show settings                        # IP adrese, maske, MTU, stanje interfejsa
net show config                          # konfiguracija po interfejsu
net show hardware                        # fizički portovi: brzina, duplex, link status
net show all                             # sve zajedno
net show domainname
net show hostname
net show dns
net hosts show                           # statička mapiranja imena
route show table
route show gateway
```

`net show hardware` je komanda koja najbrže otkriva fizički problem:
port koji je pao na 1 Gb umesto 10 Gb, ili port bez linka.

| Kolona u `net show hardware` | Na šta paziti |
|---|---|
| `Link` | `Yes` / `No` — nema linka znači kabl, SFP ili port na svič-u |
| `Speed` | Mora odgovarati očekivanoj brzini porta |
| `Duplex` | `Full`. `Half` je uvek problem |
| `Enabled` | Interfejs može biti konfigurisan a isključen |

---

## 8.2 Konfiguracija interfejsa

⚠️ Sve u ovoj sekciji može prekinuti pristup uređaju. Ako radite preko SSH-a
na interfejsu koji menjate, obezbedite drugi put (serijska konzola, IPMI).

```
net config <interfejs> <ip> netmask <maska>
net config <interfejs> up
net config <interfejs> down
net enable <interfejs>
net disable <interfejs>
net config <interfejs> mtu <vrednost>
net reset <interfejs>
```

Gateway i rute:

```
route show table
route add net <mreza> netmask <maska> gateway <ip>       # ⚠️
route del net <mreza> netmask <maska> gateway <ip>       # ⚠️
route set gateway <ip>                                   # ⚠️
```

### VLAN i virtuelni interfejsi

```
net config <interfejs>.<vlan-id> <ip> netmask <maska>    # ⚠️  npr. eth1a.100
net config <interfejs>:<n> <ip> netmask <maska>          # ⚠️  alias interfejs
net show settings
```

### MTU i jumbo frames

```
net show settings                        # trenutni MTU
net config <interfejs> mtu 9000          # ⚠️
```

> **Najčešća greška u okruženju:** jumbo frames uključen na DD-u, a nije na
> svič-u ili na klijentu. Rezultat je saobraćaj koji radi za male pakete i
> pada za velike — backup krene pa stane. MTU mora biti isti **na celom putu**.
>
> Provera end-to-end:
> ```
> net ping <cilj> ping-count 5 packet-size 8972 df-set
> ```
> Ako prolazi sa `df-set` (bez fragmentacije), jumbo je ispravno postavljen
> celim putem.

---

## 8.3 Agregacija i failover

Dva različita mehanizma koja se često mešaju:

| | **Aggregate** | **Failover** |
|---|---|---|
| Svrha | Propusnost + redundansa | Samo redundansa |
| Koristi sve portove | Da | Ne — jedan aktivan, ostali čekaju |
| Traži podršku svič-a | Da, za LACP | Ne |

```
net aggregate show
net aggregate add <virt-iface> interfaces <lista> mode <mod>     # ⚠️
net aggregate del <virt-iface>                                   # ⚠️

net failover show
net failover add <virt-iface> interfaces <lista>                 # ⚠️
net failover del <virt-iface>                                    # ⚠️
```

Modovi agregacije: `lacp`, `roundrobin`, `balanced`. Za `lacp` mrežni tim
mora konfigurisati odgovarajući port-channel na svič-u — inače link neće raditi
ili će raditi nepredvidivo.

---

## 8.4 Alati za dijagnostiku

```
net ping <cilj>
net ping <cilj> ping-count 10 packet-size 8972 df-set
net traceroute <cilj>
net lookup <ime-ili-ip>                  # DNS provera
net hosts add <ime> <ip>                 # ⚠️ statičko mapiranje
```

### Merenje stvarne propusnosti — iperf

Ovo je jedini način da se razdvoji "DD je spor" od "mreža je spora".

```
# Na jednom uređaju:
net iperf server

# Na drugom:
net iperf client <ip-servera>
```

Rezultat poredite sa nominalnom brzinom linka. Ako iperf daje 200 Mbps
na 10 Gb linku, problem nije na DD-u.

Koristi se i za proveru replikacionog linka pre nego što se optuži replikacija
(**poglavlje 07**).

### Hvatanje saobraćaja

```
net tcpdump <opcije>                     # ⚠️ troši resurse
net tcpdump-stop
```

Koristi se samo uz konkretnu hipotezu i po pravilu uz Dell support case.
Ne ostavljajte pokrenuto.

---

## 8.5 Mrežna statistika

```
net show stats                           # kumulativno
net show stats interval 2 count 10       # osvežavanje na 2 sekunde
net show stats type interfaces
net show stats type tcp
net show stats type ip
```

**Šta se traži:**

| Pokazatelj | Značenje |
|---|---|
| `errors` raste | Fizički problem: kabl, SFP, port na svič-u |
| `dropped` raste | Zagušenje ili neusklađen MTU |
| `collisions` | Duplex mismatch — praktično uvek konfiguracija |
| `overruns` | Uređaj ne stiže da obradi dolazni saobraćaj |

Rastuće greške su bitne, apsolutan broj nije — brojači se kumuliraju od
poslednjeg reboot-a. Uporedite dva očitanja u razmaku od par minuta.

---

## 8.6 Performanse sistema

```
system show performance                  # throughput po protokolima
system show stats                        # sistemski pokazatelji
system show stats interval 2             # uživo, sysstat pogled
system show stats view sysstat
```

`system show stats interval 2` je glavni alat za "šta uređaj upravo radi".
Prikazuje CPU, disk I/O, mrežni promet i replikaciju u realnom vremenu.

Po komponentama:

```
mtree show performance                   # promet po MTree-ju
ddboost show stats                       # DD Boost promet
nfs show detailed-stats                  # NFS statistika
replication show performance             # brzina replikacije
disk show performance                    # disk I/O
```

---

## 8.7 Streams — ograničenje koje se zaboravi

Svaki DD model ima ograničen broj istovremenih tokova (write, read, replication).
Kada se dostigne limit, novi backup ne pada odmah — čeka, i sve deluje sporo.

```
system show stats
quota streams show
mtree show performance
ddboost show connections
```

Ako je broj aktivnih tokova blizu limita modela, rešenje nije u mreži nego u
rasporedu backup-a — treba razmaknuti poslove ili postaviti stream kvote
po MTree-ju da jedan klijent ne pojede sve (**poglavlje 05**).

---

## 8.8 DD Boost ifgroup — raspodela opterećenja

Kod DD Boost-a raspodela po interfejsima se ne radi mrežnom agregacijom nego
`ifgroup` mehanizmom, koji klijentima dodeljuje interfejse na nivou aplikacije.

```
ifgroup show config
ifgroup show detailed
ifgroup status
```

⚠️

```
ifgroup create <ime>
ifgroup add <ime> interface <ip>
ifgroup add <ime> client <klijent>
ifgroup enable <ime>
ifgroup disable <ime>
```

Detaljno o DD Boost-u → **poglavlje 09**.

---

## 8.9 Playbook: "backup je spor"

Redosled eliminacije, od najjeftinije provere ka najskupljoj:

1. **Fizički sloj**
   ```
   net show hardware
   net show stats
   ```
   Link speed, duplex, rastuće greške. Ako je ovde problem — dalje se ne ide.

2. **Da li je uređaj uopšte opterećen**
   ```
   system show stats interval 2
   system show performance
   ```
   Ako CPU i disk nisu opterećeni, DD nije usko grlo.

3. **Da li nešto drugo radi u isto vreme**
   ```
   filesys clean status
   replication show performance
   ddboost show connections
   ```
   Cleaning na throttle 100 tokom backup prozora je čest uzrok.

4. **Streams limit**
   ```
   quota streams show
   ddboost show connections
   ```

5. **Mreža između klijenta i DD-a**
   ```
   net iperf server        # na DD-u
   ```
   Sa klijenta pustiti iperf klijent. Poređenje sa nominalnom brzinom.

6. **MTU neusklađenost**
   ```
   net ping <klijent> packet-size 8972 df-set
   ```

7. **Tip podataka**
   ```
   filesys show compression last 24 hours
   ```
   Pad faktora dedupe-a znači da stižu već komprimovani ili enkriptovani podaci —
   uređaj tada radi više posla za istu količinu. To je razgovor sa backup timom,
   ne problem na DD-u (**poglavlje 04**).

8. **Kapacitet**
   ```
   filesys show space
   ```
   File system blizu popunjenosti radi sporije. Preko 90% se to primeti.

---

## 8.10 Odvajanje saobraćaja po nameni

Dobra praksa je da menadžment, backup i replikacija ne dele isti interfejs.

```
net show settings
net hosts show
route show table
```

Replikacija se usmerava na zaseban interfejs mapiranjem imena parnjaka
na IP adresu replikacione mreže:

```
net hosts add <ime-parnjaka> <ip-replikacione-mreze>     # ⚠️
```

Za CR vault okruženja izolacija mreže nije preporuka nego zahtev —
vidi **poglavlje 12**.

---

## Reference

- DDOS Administration Guide — poglavlje o mrežnoj konfiguraciji
- DDOS Command Reference Guide — sekcije `net`, `route`, `ifgroup`, `system show stats`
- DD Security Configuration Guide — lista mrežnih portova
