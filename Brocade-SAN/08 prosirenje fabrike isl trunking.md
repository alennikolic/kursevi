# Poglavlje 8: Proširenje fabrike — ISL, trunking i fabric merge

> **Cilj poglavlja:** Bezbedno povezati dva switcha u jednu fabriku, znati šta se mora uskladiti pre povezivanja, prepoznati i rešiti segmentaciju, i razumeti trunking i planiranje propusnosti između switcheva.

---

## 8.1 Kada se fabrika širi

Razlozi su obično prozaični: potrošeni su portovi, ili se oprema nalazi u dva rack-a odnosno dve prostorije. Rešenje je dodavanje switcha i povezivanje ISL vezom (Inter-Switch Link).

Dva uobičajena rasporeda:

**Core-Edge** — serveri na edge switcheve, storage na core switcheve, edge i core povezani ISL vezama. Skalira dobro i lako se proširuje dodavanjem edge uređaja.

**Full mesh** — svaki switch povezan sa svakim. Praktično do četiri switcha po fabrici; iznad toga broj potrebnih veza postaje neracionalan.

> **Podsetnik koji se ne sme zaboraviti:** proširuje se fabrika A ili fabrika B — nikada se ne spajaju međusobno. Sve u ovom poglavlju odnosi se na povezivanje switcheva **unutar iste fabrike**.

---

## 8.2 Kako nastaje ISL

Kada se dva switcha povežu kablom, portovi na oba kraja pregovaraju i postaju E_Port. Zatim sledi proces izgradnje fabrike:

1. **Build Fabric** — switchevi razmenjuju parametre i proveravaju kompatibilnost
2. **Izbor principal switcha** — jedan switch preuzima ulogu koordinatora
3. **Dodela Domain ID-jeva** — principal potvrđuje ili dodeljuje Domain ID-jeve
4. **Razmena zoning baza** — baze se spajaju ili se veza segmentira
5. **Izgradnja rutirajućih tabela** — protokolom FSPF, po principu najkraćeg puta

Ako bilo koji korak ne uspe, port prelazi u stanje **segmented** i fabrika se ne formira.

Provera principal switcha:

```
SAN_A_SW01:admin> fabricshow
Switch ID   Worldwide Name           Enet IP Addr    FC IP Addr      Name
--------------------------------------------------------------------------------
 1: fffc01  10:00:38:ba:b0:fc:9f:b0  10.10.10.11     0.0.0.0     >"SAN_A_SW01"
 2: fffc02  10:00:38:ba:b0:fc:aa:20  10.10.10.12     0.0.0.0      "SAN_A_SW02"

The Fabric has 2 switches
Fabric Name: FABRIC_A
```

Znak `>` označava principal switch. Uloga se bira automatski; može se uticati prioritetom, ali u većini okruženja za tim nema potrebe.

---

## 8.3 Kontrolna lista pre povezivanja

Ovo je najvažniji odeljak poglavlja. Provera pre spajanja traje petnaest minuta; rešavanje segmentacije posle spajanja traje mnogo duže i često se radi u produkciji.

### 1. Domain ID mora biti jedinstven

```
SAN_A_SW01:admin> switchshow | grep -i domain
switchDomain:   1

SAN_A_SW02:admin> switchshow | grep -i domain
switchDomain:   2
```

### 2. Ime switcha mora biti jedinstveno

```
SAN_A_SW01:admin> switchname
SAN_A_SW01
```

Formalno, isto ime ne sprečava formiranje fabrike, ali čini `fabricshow` izlaz beskorisnim i vodi u greške pri radu.

### 3. Tajmeri moraju biti identični

```
SAN_A_SW01:admin> configshow | grep -iE "edtov|ratov"
fabric.ops.E_D_TOV:2000
fabric.ops.R_A_TOV:10000
```

Ove vrednosti moraju biti iste na oba switcha. Razlika je čest uzrok segmentacije, a nastaje kada je neko na jednom uređaju „probao nešto" kroz `configure` dijalog.

### 4. FOS verzije moraju biti kompatibilne

```
SAN_A_SW01:admin> version | grep "Fabric OS"
Fabric OS:  v9.2.0c3
```

Pravilo je da se podržava razlika od jedne veće verzije. Idealno je da su verzije identične. Matrica kompatibilnosti proizvođača je merodavna.

### 5. Zoning baze moraju biti spojive

Ovo je najosetljivija stavka i obrađuje se u narednom odeljku.

### 6. Podrazumevana zona mora biti ista

```
SAN_A_SW01:admin> defzone --show
Default Zone Access Mode
  committed - No Access
```

Različiti režimi podrazumevane zone na dva switcha izazivaju segmentaciju.

### 7. Portovi moraju biti licencirani i E_Port dozvoljen

```
SAN_A_SW01:admin> licenseport --show | head -3
SAN_A_SW01:admin> portcfgshow 6 | grep -i eport
Disabled E_Port           OFF
```

Ako ste u poglavlju 5 zabranili E_Port na svim portovima, ne zaboravite da ga dozvolite na onom koji koristite za ISL.

---

## 8.4 Fabric merge — spajanje zoning baza

Kada se dva switcha povežu, njihove zoning baze moraju postati jedna. Mogući ishodi:

| Situacija | Ishod |
|---|---|
| Jedan switch ima praznu bazu | Čisto spajanje — prazan preuzima bazu drugog |
| Baze su identične | Čisto spajanje |
| Baze imaju različite zone, bez sukoba imena | Spajanje — unija obe baze |
| Isto ime zone, različit sadržaj | **Segmentacija** |
| Isto ime alias-a, različit sadržaj | **Segmentacija** |
| Različite aktivne konfiguracije | **Segmentacija** |
| Objekat istog imena, ali drugog tipa (zona i alias) | **Segmentacija** |

Pravilo koje pojednostavljuje život: **najbezbednije spajanje je ono gde novi switch ima potpuno praznu zoning bazu.**

Kod novog uređaja to je već slučaj. Kod uređaja koji je negde radio, bazu treba očistiti pre spajanja — sa prethodno uzetim backup-om:

```
SAN_A_SW02:admin> configupload
...
SAN_A_SW02:admin> cfgdisable
SAN_A_SW02:admin> cfgclear
SAN_A_SW02:admin> cfgsave
```

Posle povezivanja, novi switch preuzima bazu iz postojeće fabrike.

Ako oba switcha imaju bazu koju treba sačuvati, upoređivanje se radi unapred:

```
SAN_A_SW01:admin> cfgshow > (pregled i poređenje ručno)
SAN_A_SW02:admin> cfgshow
```

Tražite: ista imena zona sa različitim članovima, ista imena alias-a sa različitim WWPN-ovima, i različita imena aktivnih konfiguracija.

---

## 8.5 Povezivanje i provera

Posle priključivanja kabla:

```
SAN_A_SW01:admin> switchshow | grep E-Port
   6   6   010600   id    N32     Online      FC  E-Port  10:00:38:ba:b0:fc:aa:20 "SAN_A_SW02"
```

```
SAN_A_SW01:admin> islshow
  1:  6-> 6 10:00:38:ba:b0:fc:aa:20   2 SAN_A_SW02    sp: 32.000G bw: 32.000G
```

Čitanje `islshow` izlaza: lokalni port 6 ide na port 6 udaljenog switcha sa Domain ID 2, imena SAN_A_SW02, brzinom 32G.

```
SAN_A_SW01:admin> fabricshow
Switch ID   Worldwide Name           Enet IP Addr    FC IP Addr      Name
--------------------------------------------------------------------------------
 1: fffc01  10:00:38:ba:b0:fc:9f:b0  10.10.10.11     0.0.0.0     >"SAN_A_SW01"
 2: fffc02  10:00:38:ba:b0:fc:aa:20  10.10.10.12     0.0.0.0      "SAN_A_SW02"

The Fabric has 2 switches
```

Ako `fabricshow` prikazuje samo jedan switch, fabrika nije formirana.

Provera da li se uređaji sa drugog switcha vide:

```
SAN_A_SW01:admin> nscamshow
nscam show for remote switches:
Switch entry for 2
  state rev   owner
 known v723  0xfffc01
  Device list: count 2
  Type Pid    COS     PortName                NodeName
  N    020000;    3;21:00:f4:e9:d4:56:9d:21;20:00:f4:e9:d4:56:9d:21;
     FC4s: FCP
     PortSymb: [31] "QLE2772 ESXi-03 port0"
```

Provera puta do uređaja na drugom switchu:

```
SAN_A_SW01:admin> fcping --number 3 21:00:f4:e9:d4:56:9d:21
Pinging 21:00:f4:e9:d4:56:9d:21 [0x020000] with 12 bytes of data:
received reply from 21:00:f4:e9:d4:56:9d:21: 12 bytes time:1234 usec
received reply from 21:00:f4:e9:d4:56:9d:21: 12 bytes time:1180 usec
received reply from 21:00:f4:e9:d4:56:9d:21: 12 bytes time:1201 usec
3 frames sent, 3 frames received, 0 frames rejected, 0 frames timeout
Round-trip min/avg/max = 1180/1205/1234 usec
```

---

## 8.6 Segmentacija — prepoznavanje

Segmentiran port izgleda ovako:

```
SAN_A_SW01:admin> switchshow | grep "^   6"
   6   6   010600   id    N32     Online      FC  Disabled (Segmented)
```

Razlog se traži u logu:

```
SAN_A_SW01:admin> errdump | grep -i segment
2026/09/15-11:04:12, [FABR-1001], 8841, FID 128, WARNING, SAN_A_SW01,
port 6, Segmented, reason: Domain ID Conflict

2026/09/15-11:04:12, [ESM-1011], 8842, FID 128, WARNING, SAN_A_SW01,
S0,P6: Segmentation, reason: Zone Conflict
```

Tabela razloga i rešenja:

| Razlog u logu | Uzrok | Rešenje |
|---|---|---|
| `Domain ID Conflict` | Oba switcha imaju isti Domain ID | `switchdisable`, `configure`, promeniti Domain, `switchenable` |
| `Zone Conflict` | Nespojive zoning baze | Uskladiti ili očistiti bazu na novom switchu |
| `E_D_TOV mismatch` / `Fabric Parameter` | Različiti tajmeri | Uskladiti kroz `configure` |
| `Incompatible Fabric Parameters` | Različit PID format ili drugi fabric parametar | Uskladiti |
| `Security Violation` | Fabric binding politika ne dozvoljava uređaj | Dodati WWN u politiku ili je isključiti |
| `ELP rejected` | Nekompatibilne FOS verzije ili licenca | Provera verzija i licenci |
| `Default Zone mismatch` | `defzone` različit na dva switcha | Uskladiti na `No Access` |

Detaljnije o samom portu:

```
SAN_A_SW01:admin> portshow 6 | grep -i segment
Segmentation Reason: Zone Conflict
```

---

## 8.7 Postupak rešavanja segmentacije

Redosled koji vodi do rešenja bez lutanja:

1. **Utvrdite razlog** — `errdump | grep -i segment` i `portshow <port>`
2. **Odlučite koja strana se menja** — po pravilu se menja novi switch, ne produkcijski
3. **Uskladite parametar** na strani koja se menja
4. **Resetujte port** da bi se ponovo pokušalo spajanje:

```
SAN_A_SW01:admin> portdisable 6
SAN_A_SW01:admin> portenable 6
```

5. **Proverite ishod** — `switchshow`, `fabricshow`, `islshow`

Kod sukoba zoning baza, najbrži i najsigurniji put je očistiti bazu na novom switchu (uz backup) i pustiti ga da preuzme bazu iz fabrike.

> **Kritična napomena:** kada rešavate segmentaciju, radite na jednoj fabrici. Proverite pre početka da druga fabrika radi normalno i da multipathing na hostovima drži saobraćaj. Ako obe fabrike imaju problem istovremeno, ne dirajte ništa dok ne stabilizujete jednu.

---

## 8.8 Trunking

Kada je jedan ISL nedovoljan, dodaju se novi. Bez trunkinga, više paralelnih ISL veza radi kao odvojeni linkovi po kojima se saobraćaj raspoređuje po razmenama (exchange). To radi, ali raspodela nije savršeno ravnomerna.

**Trunking** spaja više ISL veza u jednu logičku vezu sa zbirnom propusnošću i raspodelom na nivou pojedinačnog okvira. Rezultat je ravnomerno opterećenje i to da otkaz jednog člana ne prekida saobraćaj.

### Uslovi za trunk

- Trunking licenca na oba switcha
- Portovi u istoj grupi portova (oktetu) na oba kraja
- Ista brzina na svim članovima
- Približno jednaka dužina kablova (razlika u kašnjenju mora biti u dozvoljenom opsegu)
- Trunking uključen na portovima

```
SAN_A_SW01:admin> licenseshow | grep -i trunk
    Trunking license

SAN_A_SW01:admin> portcfgshow 6 | grep -i trunk
Trunk Port                ON
```

Uključivanje na portu:

```
SAN_A_SW01:admin> portcfgtrunkport 6 1
SAN_A_SW01:admin> portcfgtrunkport 7 1
```

### Provera trunka

```
SAN_A_SW01:admin> trunkshow
  1:  6-> 6 10:00:38:ba:b0:fc:aa:20   2 deskew 15 MASTER
      7-> 7 10:00:38:ba:b0:fc:aa:20   2 deskew 16
```

Dva porta u jednoj grupi, jedan je master. Vrednost `deskew` odražava razliku u dužini putanje; velika razlika sprečava formiranje trunka.

```
SAN_A_SW01:admin> islshow
  1:  6-> 6 10:00:38:ba:b0:fc:aa:20   2 SAN_A_SW02   sp: 32.000G bw: 64.000G TRUNK QOS CR_RECOV
```

Oznaka `TRUNK` i `bw: 64.000G` potvrđuju da dva linka od 32G rade kao jedan od 64G.

---

## 8.9 Rutiranje i raspodela opterećenja

```
SAN_A_SW01:admin> aptpolicy
Current Policy: 3 0(ap shared link)

3 0(ap shared link) : Default Policy
1: Port Based Routing Policy
2: Device Based Routing Policy
3: Exchange Based Routing Policy
```

Podrazumevana politika je **Exchange Based Routing** — saobraćaj se raspoređuje po razmenama, što daje najbolje iskorišćenje više putanja. Menja se retko i samo kada to traži proizvođač storage niza.

Povezane postavke:

```
SAN_A_SW01:admin> dlsshow
DLS is set with Lossless enabled

SAN_A_SW01:admin> iodshow
IOD is not set
```

**DLS** (Dynamic Load Sharing) preraspoređuje saobraćaj kada se putanje promene. **IOD** (In-Order Delivery) garantuje redosled okvira po ceni propusnosti. Podrazumevane vrednosti odgovaraju gotovo svim okruženjima.

---

## 8.10 Planiranje propusnosti

Ključni pojam je **oversubscription** — odnos između ukupne propusnosti uređaja koji šalju saobraćaj kroz ISL i propusnosti samog ISL-a.

Primer: deset hostova na 32G portovima na edge switchu pristupa storage nizu preko jednog ISL-a od 32G. Odnos je 10:1. To u praksi može biti sasvim prihvatljivo, jer hostovi retko istovremeno koriste punu propusnost — ali samo ako to važi za vaš saobraćaj.

Orijentacija:

| Tip opterećenja | Prihvatljiv odnos |
|---|---|
| Virtuelizacija, mešano opterećenje | 10:1 do 20:1 |
| Baze podataka, transakcioni sistemi | 6:1 do 10:1 |
| Backup i sekvencijalno opterećenje | 3:1 do 6:1 |

Merenje umesto nagađanja:

```
SAN_A_SW01:admin> portperfshow 6
     6
  Bps
  2.1g
```

```
SAN_A_SW01:admin> portstatsshow 6 | grep tim_txcrd_z
tim_txcrd_z                    128441
```

Ako `tim_txcrd_z` na ISL portu stalno raste, link je zagušen — potreban je dodatni ISL, ne zamena kabla.

---

## 8.11 Veze na velikim udaljenostima

Na dugom vlaknu okvir putuje dovoljno dugo da pošiljalac ostane bez kredita pre nego što stignu potvrde. Tada se ne gubi link, nego propusnost.

```
SAN_A_SW01:admin> portcfglongdistance 6 LS 1 100
```

Režimi:

| Režim | Namena |
|---|---|
| `L0` | Normalna veza unutar data centra (podrazumevano) |
| `LE` | Do 10 km |
| `LD` | Automatsko merenje udaljenosti |
| `LS` | Ručno zadata udaljenost u kilometrima |

```
SAN_A_SW01:admin> portbuffershow
User  Port Lx Max/Resv  Buffer   Needed    Link   Remaining
Port  Type Mode Buffers Usage    Buffers Distance Buffers
----  ---- ---- ------- ------   ------- -------- ---------
  6     E   LS    550     550       550     100km      3410
```

Veći broj kredita zahteva i odgovarajuću licencu (Extended Fabric) na uređajima koji je traže.

---

## 8.12 Rezime poglavlja

- Pre povezivanja proverite: Domain ID, ime, tajmere, FOS verziju, zoning bazu, `defzone` i dozvolu E_Porta.
- Najbezbednije spajanje je ono u kojem novi switch ima praznu zoning bazu.
- Segmentacija se prepoznaje po `Disabled (Segmented)` u `switchshow`, a razlog se čita iz `errdump`.
- Posle usklađivanja parametra, port treba resetovati sa `portdisable` / `portenable`.
- Trunking zahteva licencu, portove u istoj grupi, istu brzinu i približno jednake kablove.
- `tim_txcrd_z` koji raste na ISL portu znači zagušenje, a ne kvar.
- Rad na segmentaciji se izvodi na jednoj fabrici, uz prethodnu proveru da druga radi.

---

## 8.13 Provera znanja

1. Povezali ste novi switch i `fabricshow` prikazuje samo lokalni uređaj. Koja su vam prva dva koraka?
2. Zašto je poželjno obrisati zoning bazu na switchu koji dodajete u postojeću fabriku?
3. Dva ISL-a od 32G povezuju switcheve, ali `islshow` ne prikazuje oznaku `TRUNK`. Navedite tri moguća uzroka.
4. Na edge switchu je petnaest hostova i jedan ISL ka core switchu. `tim_txcrd_z` na ISL portu raste. Šta je rešenje, a šta sigurno nije?
5. Kolega predlaže da se fabrike A i B povežu jednim ISL-om „radi redundanse". Objasnite zašto je to loša ideja.

---

**Sledeće poglavlje:** Održavanje — backup i restore konfiguracije, nadogradnja firmware-a, `supportsave` i rutinske provere.
