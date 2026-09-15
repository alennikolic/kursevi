# Poglavlje 4: FOS CLI — snalaženje u komandnoj liniji

> **Cilj poglavlja:** Razumeti logiku po kojoj su FOS komande imenovane, znati kako doći do pomoći i filtrirati izlaz, i savladati skup komandi kojima se svakodnevno proverava stanje switcha i fabrike.

---

## 4.1 Kako FOS CLI funkcioniše

Fabric OS je zasnovan na Linuxu, i to se u komandnoj liniji oseća. Nema hijerarhije režima kao kod mrežne opreme drugih proizvođača — nema `enable`, nema `configure terminal`, nema `interface` pod-režima. Postoji jedan nivo, a prava određuje uloga naloga kojim ste prijavljeni.

Tri posledice koje treba odmah usvojiti:

**1. Komanda se izvršava u trenutku kada pritisnete Enter.** Ne postoji faza pripreme izmena koje se kasnije primenjuju odjednom. `portdisable 5` gasi port istog trenutka.

**2. Većina izmena se odmah upisuje u trajnu konfiguraciju.** Nema `write memory` koraka. Ako ste pogrešili, ispravljate novom komandom, ne restartom uređaja.

**3. Jedini značajan izuzetak je zoning.** Izmene zona žive u privremenoj radnoj kopiji sve dok ih eksplicitno ne sačuvate (`cfgsave`) i aktivirate (`cfgenable`). To je namerno — zoning je previše opasan da bi se primenjivao komandu po komandu.

---

## 4.2 Konvencija imenovanja komandi

FOS komande se grade po šablonu **objekat + akcija**:

| Objekat | Primer komande | Šta radi |
|---|---|---|
| `switch` | `switchshow` | Prikaz stanja switcha |
| `port` | `portshow 3` | Prikaz stanja porta 3 |
| `fabric` | `fabricshow` | Prikaz članova fabrike |
| `cfg` | `cfgshow` | Prikaz zoning konfiguracije |
| `license` | `licenseshow` | Prikaz licenci |

Nastavci nose značenje:

| Nastavak | Značenje | Primer |
|---|---|---|
| `show` | Prikaz stanja | `portshow`, `nsshow` |
| `cfgshow` | Prikaz **konfiguracije**, ne trenutnog stanja | `portcfgshow` |
| `set` | Postavljanje vrednosti | `ipaddrset`, `passwdcfg --set` |
| `enable` / `disable` | Uključivanje / isključivanje | `portenable`, `switchdisable` |
| `add` / `delete` | Dodavanje / brisanje objekta | `zonecreate`, `userconfig --delete` |

**Bitna razlika koju početnici stalno mešaju:** `portshow` prikazuje **stvarno stanje** porta u ovom trenutku, dok `portcfgshow` prikazuje **podešavanja** koja su na njemu postavljena. Port može biti konfigurisan za 32G, a raditi na 16G jer je to najviše što druga strana podržava.

Noviji delovi FOS-a koriste opcije sa dvostrukom crticom, stariji ne:

```
userconfig --show -a          # noviji stil
portcfgpersistentdisable 5    # stariji stil
```

Oba postoje istovremeno i to je normalno.

---

## 4.3 Pomoć

### Spisak svih komandi

```
SAN_A_SW01:admin> help
aaaConfig                    Configure RADIUS/LDAP/TACACS+ server
ag                           Access Gateway configuration
agtcfgDefault                Reset SNMP agent to factory default
alias                        Create/manage zone aliases
aliAdd                       Add member to zone alias
...
```

Spisak je dugačak, pa se u praksi kombinuje sa filtriranjem:

```
SAN_A_SW01:admin> help | grep -i zone
cfgShow                      Print zoning configuration
zoneAdd                      Add member to zone
zoneCreate                   Create zone
zoneDelete                   Delete zone
zoneObjectCopy               Copy a zone object
zoneObjectRename             Rename a zone object
zoneShow                     Print zone information
```

### Pomoć za pojedinačnu komandu

```
SAN_A_SW01:admin> help portcfgspeed

NAME
       portCfgSpeed - Sets the speed of a port

SYNOPSIS
       portcfgspeed [slot/]port speed_level

DESCRIPTION
       Use this command to set the speed of a port. The speed
       setting is persistent across power cycles and reboots.

OPERANDS
       This command has the following operands:

       speed_level
              0   Auto-negotiate (default)
              1   1 Gbps
              2   2 Gbps
              4   4 Gbps
              8   8 Gbps
              16  16 Gbps
              32  32 Gbps

EXAMPLES
       To set port 3 to auto-negotiate:
              switch:admin> portcfgspeed 3 0

SEE ALSO
       portCfgShow, portShow, switchShow
```

Sekcija `SEE ALSO` je koristan način da se otkriju srodne komande koje niste znali da postoje.

### Istorija komandi

```
SAN_A_SW01:admin> h
   21  switchshow
   22  sfpshow 4
   23  porterrshow
   24  portshow 4
   25  h
```

Strelice gore i dole prolaze kroz istoriju, `!23` ponavlja komandu pod tim brojem.

---

## 4.4 Filtriranje izlaza

FOS podržava prosleđivanje izlaza kroz standardne Linux alate. Ovo je u svakodnevnom radu neprocenjivo, jer izlazi poput `switchshow` na switchu sa 64 porta ne staju na ekran.

```
SAN_A_SW01:admin> switchshow | grep Online
   0   0   010000   id    N32     Online      FC  F-Port  21:00:f4:e9:d4:56:7a:b1
   1   1   010100   id    N32     Online      FC  F-Port  21:00:f4:e9:d4:56:7a:b2
   3   3   010300   id    N16     Online      FC  F-Port  50:06:01:60:88:60:2a:11
   6   6   010600   id    N32     Online      FC  E-Port  10:00:38:ba:b0:fc:aa:20 "SAN_A_SW02"
```

Korisni obrasci:

```
# Koji portovi nemaju svetlo
SAN_A_SW01:admin> switchshow | grep No_Light

# Koliko portova je online
SAN_A_SW01:admin> switchshow | grep -c Online
4

# Da li je određeni WWPN uopšte prijavljen
SAN_A_SW01:admin> nsshow | grep -i 21:00:f4:e9:d4:56:7a:b1

# Dug izlaz stranicu po stranicu
SAN_A_SW01:admin> cfgshow | more

# Samo E-portovi, dakle veze ka drugim switchevima
SAN_A_SW01:admin> switchshow | grep E-Port
```

Podržani su i `head`, `tail`, `wc` i `nl`. Preusmeravanje izlaza u fajl (`>`) nije podržano — za izvoz podataka koriste se namenske komande poput `configupload` i `supportsave`.

---

## 4.5 Komande za svakodnevni rad

Ovo je jezgro koje treba znati napamet, grupisano po tome šta pitate.

| Pitanje | Komanda |
|---|---|
| Kakvo je opšte stanje switcha i portova? | `switchshow` |
| Ko je sve u fabrici? | `fabricshow` |
| Koji uređaji su prijavljeni lokalno / u celoj fabrici? | `nsshow` / `nscamshow` |
| Kakvo je stanje jednog porta? | `portshow <port>` |
| Kako je port konfigurisan? | `portcfgshow <port>` |
| Ima li grešaka na portovima? | `porterrshow` |
| Kakav je optički signal? | `sfpshow <port>` |
| Koja je verzija firmware-a? | `firmwareshow` |
| Šta piše u logu? | `errdump` / `errshow` |
| Kakvo je zdravlje hardvera? | `switchstatusshow`, `sensorshow` |
| Koliko dugo uređaj radi? | `uptime` |
| Kakav je zoning? | `cfgshow`, `cfgactvshow` |

### `version` i `firmwareshow`

```
SAN_A_SW01:admin> version
Kernel:     3.14.17
Fabric OS:  v9.2.0c3
Made on:    Thu Mar 14 06:22:11 2024
Flash:      Mon Sep 14 21:04:33 2026
BootProm:   1.0.11
```

```
SAN_A_SW01:admin> firmwareshow
Appl     Primary/Secondary Versions
------------------------------------------
FOS      v9.2.0c3
         v9.2.0c3
```

Dve verzije znače dve particije — primarnu i sekundarnu. Posle uspešnog nadogradnje obe su iste. Ako se razlikuju, nadogradnja je u toku ili nije dovršena.

### `uptime`

```
SAN_A_SW01:admin> uptime
10:24:18 up 412 days, 3:12, 1 user, load average: 0.08, 0.11, 0.09
```

Uptime od nekoliko stotina dana je u FC svetu normalan i poželjan. Neočekivano nizak uptime na uređaju koji niste restartovali je znak da treba pogledati log.

### `porterrshow` — brojači grešaka po portovima

Ovo je komanda koju vredi pogledati svaki put kada nešto „čudno radi".

```
SAN_A_SW01:admin> porterrshow
          frames      enc    crc    crc    too   too    bad   enc   disc   link   loss   loss   frjt   fbsy
       tx     rx      in    err    g_eof  shrt   long   eof   out   c3     fail   sync   sig
  0:  1.2g   3.4g     0      0      0      0      0      0     0     0      0      1      2      0      0
  1:  1.1g   3.3g     0      0      0      0      0      0     0     0      0      1      2      0      0
  2:  890m   2.1g     0      0      0      0      0      0     0     0      0      0      1      0      0
  3:  3.4g   1.2g    14     12     12      0      0      0    38     0      2      4      5      0      0
  4:    0      0      0      0      0      0      0      0     0     0      0      0      0      0      0
  6:  8.9g   9.1g     0      0      0      0      0      0     0     0      0      1      2      0      0
```

Kolone koje su najvažnije:

| Kolona | Značenje | Kada je problem |
|---|---|---|
| `enc in` | Greške kodiranja unutar okvira | Raste → loš fizički sloj |
| `crc err` | Okviri sa neispravnom kontrolnom sumom | Bilo koja vrednost koja raste je ozbiljna |
| `crc g_eof` | CRC greške sa dobrim krajem okvira | Greška je nastala na **ovom** linku |
| `disc c3` | Odbačeni Class 3 okviri | Zagušenje ili problem sa kreditima |
| `link fail` | Prekidi linka | Nestabilan link, loš kabl ili SFP |
| `loss sync` | Gubici sinhronizacije | Isto kao gore |
| `loss sig` | Gubici signala | Kabl je izvučen ili druga strana ugašena |

**Ključna veština:** brojači su kumulativni od poslednjeg resetovanja. Jedna CRC greška nastala pre godinu dana pri prekabliranju nije problem. Zato se brojači **resetuju i posmatra se da li ponovo rastu**:

```
SAN_A_SW01:admin> statsclear
SAN_A_SW01:admin> slotstatsclear
```

Sačekajte nekoliko minuta pod normalnim opterećenjem, pa ponovo pogledajte `porterrshow`. Ako brojači rastu, problem je aktivan.

Razlika između `crc err` i `crc g_eof` govori gde je greška nastala: kada je `g_eof` brojač takođe uvećan, okvir je stigao ceo, ali sa lošim CRC-om — greška je nastala na tom linku. Ako je samo `crc err` uvećan, oštećenje je verovatno nastalo ranije na putanji.

### `portstatsshow` — detaljna statistika jednog porta

```
SAN_A_SW01:admin> portstatsshow 3
stat_wtx                 3412887623  4-byte words transmitted
stat_wrx                 1198234112  4-byte words received
stat_ftx                  214887623  Frames transmitted
stat_frx                   98234112  Frames received
er_enc_in                        14  Encoding errors inside of frames
er_crc                           12  Frames with CRC errors
er_trunc                          0  Frames shorter than minimum
er_toolong                        0  Frames longer than maximum
er_bad_eof                        0  Frames with bad end-of-frame
er_disc_c3                       38  Class 3 frames discarded
tim_txcrd_z                   28841  Time TX Credit Zero (2.5Us ticks)
```

`tim_txcrd_z` je vredan podatak: broji koliko je vremena port proveo bez raspoloživih kredita za slanje. Visoka i rastuća vrednost znači da druga strana ne stiže da obrađuje saobraćaj — to je zagušenje, a ne kvar.

### `nscamshow` — uređaji u celoj fabrici

Dok `nsshow` prikazuje samo lokalno prijavljene uređaje, `nscamshow` obuhvata i one na ostalim switchevima:

```
SAN_A_SW01:admin> nscamshow
nscam show for remote switches:
Switch entry for 2
  state rev   owner
 known v723  0xfffc01
  Device list: count 3
  Type Pid    COS     PortName                NodeName
  N    020000;    3;21:00:f4:e9:d4:56:8c:11;20:00:f4:e9:d4:56:8c:11;
     FC4s: FCP
     PortSymb: [31] "QLE2772 ESXi-03 port1"
  N    020300;    3;50:06:01:61:88:60:2a:12;50:06:01:60:08:60:2a:12;
     FC4s: FCP
     PortSymb: [28] "Storage-NodeB-Port1"
```

### `sensorshow`

```
SAN_A_SW01:admin> sensorshow
sensor  1: (Temperature) is Ok, value is 38 C
sensor  2: (Temperature) is Ok, value is 41 C
sensor  3: (Fan      ) is Ok, speed is 7420 RPM
sensor  4: (Fan      ) is Ok, speed is 7350 RPM
sensor  5: (Power Supply) is Ok
sensor  6: (Power Supply) is Ok
```

---

## 4.6 Logovi

### Pregled

```
SAN_A_SW01:admin> errshow
Fabric OS: v9.2.0c3

2026/09/15-10:02:14, [SEC-3020], 8821, FID 128, INFO, SAN_A_SW01,
Event: login, Status: success, Info: Successful login attempt via
REMOTE, IP Addr: 10.10.20.55.

Type <CR> to continue, Q<CR> to stop:

2026/09/15-09:48:33, [FW-1424], 8820, FID 128, WARNING, SAN_A_SW01,
Switch status changed from HEALTHY to MARGINAL.
```

`errshow` je interaktivan, stranicu po stranicu. Za obradu i filtriranje koristi se `errdump`:

```
SAN_A_SW01:admin> errdump | grep -i error
2026/09/14-22:11:05, [C2-1006], 8803, FID 128, ERROR, SAN_A_SW01,
S0,P4(0): CRC error threshold exceeded.
```

Filtriranje po nivou ozbiljnosti:

```
SAN_A_SW01:admin> errdump -s WARNING
SAN_A_SW01:admin> errdump -a
```

### Struktura poruke

```
2026/09/14-22:11:05, [C2-1006], 8803, FID 128, ERROR, SAN_A_SW01, S0,P4(0): CRC error threshold exceeded.
        │              │         │      │        │       │          │
        │              │         │      │        │       │          └── detalj (slot 0, port 4)
        │              │         │      │        │       └───────────── ime switcha
        │              │         │      │        └───────────────────── nivo ozbiljnosti
        │              │         │      └────────────────────────────── Fabric ID
        │              │         └───────────────────────────────────── redni broj poruke
        │              └──────────────────────────────────────────────── kod poruke (modul-broj)
        └─────────────────────────────────────────────────────────────── vreme
```

Kod poruke (`C2-1006`) je ono što se traži u dokumentaciji proizvođača — svaki kod ima opis, verovatan uzrok i preporučenu akciju.

### Slanje logova na syslog server

Log na switchu je ograničen po veličini i rotira se. Ozbiljan problem često ostavi trag koji nestane pre nego što stignete da ga pogledate.

```
SAN_A_SW01:admin> syslogdipadd 10.10.5.30
Syslog IP address 10.10.5.30 added

SAN_A_SW01:admin> syslogdipshow
syslog.IP.address.1: 10.10.5.30
```

Ovo uradite na svakom switchu, isti dan kada ga pustite u rad.

---

## 4.7 Komande koje prekidaju rad

Sledeće komande utiču na saobraćaj. Nijedna nema potvrdu koja bi vas zaustavila.

| Komanda | Posledica |
|---|---|
| `switchdisable` | Gasi sve portove — potpuni prekid na tom switchu |
| `portdisable <port>` | Gasi jedan port |
| `cfgdisable` | Deaktivira zoning — svi uređaji gube međusobnu vidljivost |
| `cfgenable <cfg>` | Aktivira zoning konfiguraciju u celoj fabrici |
| `reboot` / `fastboot` | Restart uređaja |
| `configdownload` | Prepisuje konfiguraciju iz fajla |
| `firmwaredownload` | Nadogradnja firmware-a |

Pravilo koje vredi usvojiti kao naviku: **pre svake od ovih komandi proverite u promptu na kom ste switchu.** U okruženju sa fabrikama A i B, sa dva otvorena SSH prozora, ovo nije teorijska opasnost.

```
SAN_A_SW01:admin>     ← fabrika A
SAN_B_SW01:admin>     ← fabrika B
```

Ako su vam prozori isto obojeni i niste sigurni:

```
SAN_A_SW01:admin> switchshow | head -3
switchName:     SAN_A_SW01
switchType:     183.0
switchState:    Online
```

### Automatsko odjavljivanje

```
SAN_A_SW01:admin> timeout
Current IDLE Timeout is 10 minutes

SAN_A_SW01:admin> timeout 15
IDLE Timeout Changed to 15 minutes
```

Zaboravljena otvorena sesija na produkcijskom switchu je bezbednosni i operativni rizik. Ne povećavajte ovu vrednost bez razloga.

---

## 4.8 Rezime poglavlja

- FOS nema režime konfiguracije — komanda deluje odmah i najčešće se odmah trajno upisuje.
- Zoning je izuzetak: izmene žive u radnoj kopiji do `cfgsave` i `cfgenable`.
- `portshow` pokazuje stanje, `portcfgshow` pokazuje podešavanje — to nije isto.
- Filtriranje kroz `grep` i `more` je osnovni alat za rad sa dugačkim izlazima.
- `porterrshow` brojači su kumulativni; resetujte ih i posmatrajte da li ponovo rastu.
- `tim_txcrd_z` iz `portstatsshow` razlikuje zagušenje od kvara.
- Logove šaljite na syslog server — lokalni log rotira i gubi tragove.
- Pre svake komande koja prekida rad, proverite u kom ste promptu.

---

## 4.9 Provera znanja

1. Port je konfigurisan komandom `portcfgspeed 5 32`, ali `switchshow` prikazuje `16G`. Da li je ovo greška? Kojom komandom proveravate šta je zaista podešeno?
2. `porterrshow` prikazuje 240 CRC grešaka na portu 7. Koje su vam sledeće dve radnje pre nego što zaključite da je kabl loš?
3. Kolega kaže da je „primenio izmene zoninga" tako što je otkucao `zonecreate` i `zoneadd`. Da li su te izmene aktivne u fabrici?
4. `portstatsshow` na E-portu pokazuje `tim_txcrd_z` koji naglo raste, ali nema nijedne CRC greške. Da li je problem u kablu?
5. Zašto je `syslogdipadd` jedna od prvih komandi koje treba izvršiti na novom switchu?

---

**Sledeće poglavlje:** Rad sa portovima — konfiguracija brzine i tipa porta, enable i disable, persistent disable, `portcfg` opcije i provera uspostavljenog linka.
