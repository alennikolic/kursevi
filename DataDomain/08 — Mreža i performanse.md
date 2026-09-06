# 08 — Mreža i performanse

Mrežna konfiguracija, merenje opterećenja i dijagnostika kada je backup ili
replikacija sporija nego što bi trebalo.

---

## 8.1 Pregled konfiguracije

```
net show settings
net show config [<ifname>]
net show hardware                        # brzina, duplex, link status
net show all
net show hostname
net show domainname
net show dns
net hosts show
net route show config [routing-table-name <ime>]
net route show gateways [ipversion {ipv4 | ipv6}] [detailed]
net route show tables [<table-name-list> | ipversion {ipv4 | ipv6}]
```

`net show hardware` najbrže otkriva fizički problem:

| Kolona | Na šta paziti |
|---|---|
| `Link` | `No` znači kabl, SFP ili port na svič-u |
| `Speed` | Mora odgovarati očekivanoj brzini porta |
| `Duplex` | Mora biti `Full`. `Half` je uvek problem |
| `Enabled` | Interfejs može biti konfigurisan a isključen |

---

## 8.2 Konfiguracija interfejsa

⚠️ Može prekinuti pristup uređaju. Ako radite preko SSH-a na interfejsu koji
menjate, obezbedite drugi put (serijska konzola, IPMI).

```
net config <interfejs> <ip> netmask <maska>
net config <interfejs> up
net config <interfejs> down
net config <interfejs> mtu <vrednost>
net enable <interfejs>
net disable <interfejs>
net reset <interfejs>
```

### Rute

Komanda je **`net route`**, ne `route`.

```
net route show tables
net route show gateways
net route add [ipversion {ipv4 | ipv6}] [type {fixed | floating}] <route-spec>   # ⚠️
net route add gateway <ipv4address> [interface <ime>]                            # ⚠️
net route del [ipversion {ipv4 | ipv6}] [type {fixed | floating}] <route-spec>   # ⚠️
net route del gateway <ipv4address> [interface <ime>]                            # ⚠️
```

Primeri iz dokumentacije:

```
net route add 192.168.1.0 netmask 255.255.255.0 gw srvr12
net route add 192.168.1.0 netmask 255.255.255.0 gw srvr12 table teth5a
net route add user24 gw srvr12
net route add gateway 192.168.1.2 interface eth0b
```

### VLAN i alias interfejsi

```
net config <interfejs>.<vlan-id> <ip> netmask <maska>    # ⚠️  npr. eth1a.100
net config <interfejs>:<n> <ip> netmask <maska>          # ⚠️  alias
net show settings
```

> `net alias` adrese se **ne mogu** dodati u `net filter` niti u iptables pravila.

### MTU i jumbo frames

```
net show settings                        # trenutni MTU
net config <interfejs> mtu 9000          # ⚠️
```

> **Najčešća greška:** jumbo frames uključen na DD-u, a nije na svič-u ili na
> klijentu. Saobraćaj radi za male pakete i pada za velike — backup krene pa stane.
> MTU mora biti isti **na celom putu**.

Provera end-to-end:

```
net ping <cilj> count 5 packet-size 8972 path-mtu do
```

`path-mtu do` znači "ne fragmentiraj". Ako prolazi, jumbo je ispravno
postavljen celim putem. Opcije su `{do | dont | want}`.

---

## 8.3 Agregacija i failover

| | **Aggregate** | **Failover** |
|---|---|---|
| Svrha | Propusnost + redundansa | Samo redundansa |
| Koristi sve portove | Da | Ne — jedan aktivan |
| Traži podršku svič-a | Da, za LACP | Ne |

```
net aggregate show [detailed]
net failover show
```

⚠️

```
net aggregate add <virt-ifname> interfaces <lista> [mode {roundrobin | balanced hash {xor-L2 | xor-L3L4 | ...} | lacp}]
net aggregate modify <virt-ifname> [mode {...}]
net aggregate del <virt-ifname> interfaces {<lista> | all}
net failover add <virt-ifname> interfaces <lista>
net failover del <virt-ifname>
```

Za `lacp` mrežni tim mora konfigurisati port-channel na svič-u — inače link
neće raditi ili će raditi nepredvidivo.

---

## 8.4 Alati za dijagnostiku

```
net ping {<ipaddr> | <hostname>} [count <n>] [interface <ifname>] \
    [packet-size <bytes>] [path-mtu {do | dont | want}] [numeric] [verbose]

net route trace {<ipaddr> | <hostname>} [no-resolve]

net lookup <ime-ili-ip>
net hosts add <ime> <ip>                 # ⚠️
net hosts del <ime>                      # ⚠️
```

> **`net traceroute` ne postoji.** Komanda je `net route trace`.
> Alijas `traceroute` je mapiran na nju.

### Merenje propusnosti — iperf

Jedini način da se razdvoji "DD je spor" od "mreža je spora".

```
# Na jednom uređaju:
net iperf server [run] [ipversion {ipv4 | ipv6}] [bind <ipaddr>] [iperf-version {v2 | v3}]

# Na drugom:
net iperf client {<ipaddr> | <hostname>} [iperf-version {v2 | v3}]
```

Rezultat poredite sa nominalnom brzinom linka. Ako iperf daje 200 Mbps
na 10 Gb linku, problem nije na DD-u.

### Hvatanje saobraćaja

```
net tcpdump <opcije>                     # ⚠️ troši resurse
net tcpdump-stop
```

Samo uz konkretnu hipotezu i po pravilu uz Dell support case. Ne ostavljati pokrenuto.

---

## 8.5 Mrežna statistika

```
net show stats [[ipversion {ipv4 | ipv6}] [[all | listening] [detailed] | route | statistics] | interfaces]
net show stats interfaces
net show stats all detailed
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
system show stats [view {cifs | repl | net | iostat | sysstat | ddboost}] \
    [custom-view <spec>] [interval <sec>] [count <n>]

system show performance [raw | fsop | view {legacy | default}] \
    [custom-view {state | throughput | protocol | compression | streams | utilization | mtree-active}] \
    [duration <n> {hr | min} [interval <n> {hr | min}]]
```

Primeri:

```
system show stats interval 2                          # uživo, na 2 sekunde
system show stats view sysstat interval 2
system show stats view cifs interval 2
system show stats custom-view cpu state nfs disk
system show performance duration 30 min
system show performance duration 30 min interval 5 min
system show performance custom-view state streams
```

`custom-view` za `system show stats` prima sekcije: `cpu`, `state`, `nfs`,
`cifs`, `net`, `disk`, `nvram`, `repl`.

Ako je `interval` naveden a `count` nije, izlaz ide u nedogled do `Ctrl-C`.
Podrazumevani interval je 5 sekundi.

Po komponentama:

```
mtree show performance /data/col1/<mtree> [interval <n> {mins | hrs}]
ddboost show stats [interval <sec>] [count <n>]
ddboost show histogram
nfs show detailed-stats
nfs show histogram
replication show performance {<obj-spec> | all} [interval <sec>]
disk show performance
```

---

## 8.7 Streams — ograničenje koje se zaboravi

Svaki DD model ima ograničen broj istovremenih tokova. Kada se dostigne limit,
novi backup ne pada odmah — čeka, i sve deluje sporo.

```
system show performance custom-view streams
ddboost show connections detailed
quota streams show all
mtree show performance /data/col1/<mtree>
```

Ako je broj aktivnih tokova blizu limita modela, rešenje nije u mreži nego u
rasporedu backup-a — razmaknuti poslove ili postaviti stream kvote.

> Stream kvote (`quota streams set`) rade **samo nad DD Boost storage unit-ovima**,
> ne nad MTree-jevima. Vidi **poglavlje 05**.

Za replikaciju postoji i ograničenje po kontekstu:

```
replication modify <destination> max-repl-streams <n>    # ⚠️
```

---

## 8.8 DD Boost ifgroup — raspodela opterećenja

Kod DD Boost-a raspodela po interfejsima se ne radi mrežnom agregacijom nego
`ifgroup` mehanizmom.

```
ifgroup show config [<group>] {all | summary | interfaces | clients | replication}
ifgroup show connections
```

⚠️

```
ifgroup create <group>
ifgroup add <group> {interface {<ipaddr> | <ipv6addr>} | client <host>}
ifgroup del <group> {interface <ipaddr> | client <host>}
ifgroup enable <group>
ifgroup disable <group>
ifgroup rename <staro> <novo>
ifgroup destroy <group>
ifgroup option set {disable-file-replication | enforce-client-interface}
ifgroup option reset {disable-file-replication | enforce-client-interface}
```

> **`ifgroup status` ne postoji.** Stanje se vidi kroz `ifgroup show config`
> i `ifgroup show connections`.

Detaljno o DD Boost-u → **poglavlje 09**.

---

## 8.9 Ograničavanje pristupa — net filter

```
net filter show
net filter config show
net filter clear stats
```

⚠️

```
net filter add [seq-id <n>] operation {allow | block} protocol tcp [ports <port>] [except-clients <ip1>,<ip2>]
net filter del <seq-id>
net filter reset
net filter auto-list add ports {all}
net filter config set admin-interface <ifname> [client <host>] [ports <port>]
net filter config reset [admin-interface | mss-minimum] [ipversion {ipv4 | ipv6}]
```

Ovo je mehanizam za IP restrikcije — **ne `adminaccess`**. Vidi **poglavlje 01**.

---

## 8.10 Playbook: "backup je spor"

Redosled eliminacije, od najjeftinije provere ka najskupljoj:

1. **Fizički sloj**
   ```
   net show hardware
   net show stats interfaces
   ```
   Link speed, duplex, rastuće greške. Ako je ovde problem — dalje se ne ide.

2. **Da li je uređaj opterećen**
   ```
   system show stats interval 2
   system show performance duration 60 min
   ```

3. **Da li nešto drugo radi u isto vreme**
   ```
   filesys clean status
   replication show performance all
   ddboost show connections detailed
   ```
   Cleaning na throttle 100 tokom backup prozora je čest uzrok.

4. **Streams limit**
   ```
   system show performance custom-view streams
   quota streams show all
   ```

5. **Mreža između klijenta i DD-a**
   ```
   net iperf server
   ```
   Sa klijenta pustiti iperf klijent i uporediti sa nominalnom brzinom.

6. **MTU neusklađenost**
   ```
   net ping <klijent> count 5 packet-size 8972 path-mtu do
   ```

7. **Tip podataka**
   ```
   filesys show compression last 24 hours
   ```
   Pad faktora dedupe-a znači da stižu već komprimovani ili enkriptovani podaci.
   Razgovor sa backup timom, ne problem na DD-u (**poglavlje 04**).

8. **Kapacitet**
   ```
   filesys show space
   ```
   File system blizu popunjenosti radi sporije.

---

## 8.11 Odvajanje saobraćaja po nameni

Menadžment, backup i replikacija ne bi trebalo da dele isti interfejs.

```
net show settings
net hosts show
net route show tables
```

Replikacija se usmerava mapiranjem imena parnjaka na IP replikacione mreže:

```
net hosts add <ime-parnjaka> <ip-replikacione-mreze>     # ⚠️
```

Za CR vault okruženja izolacija mreže nije preporuka nego zahtev —
**poglavlje 12**.

---

## Reference

- DD OS 8.6 Command Reference Guide — poglavlja `net`, `ifgroup`, `system` (sekcije `show stats` i `show performance`)
- DD OS 8.6 Administration Guide — mrežna konfiguracija
- Dell KB 000004184 — port requirements
