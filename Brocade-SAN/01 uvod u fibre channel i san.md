# Poglavlje 1: Uvod u Fibre Channel i SAN

> **Cilj poglavlja:** Razumeti šta je Fibre Channel mreža, od kojih se delova sastoji, kako se uređaji u njoj adresiraju i identifikuju, i kako se sve to vidi kroz osnovne komande na Brocade switchu.

---

## 1.1 Zašto Fibre Channel?

Fibre Channel (FC) je mrežna tehnologija napravljena isključivo za jednu svrhu — prenos blokovskog storage saobraćaja između servera i diskovnih nizova. Za razliku od Etherneta, koji je opštenamenski, FC je projektovan oko pretpostavke da se **paket ne sme izgubiti**.

Ključne razlike u odnosu na IP/Ethernet storage (iSCSI, NFS):

| Karakteristika | Fibre Channel | Ethernet / iSCSI |
|---|---|---|
| Kontrola protoka | Buffer-to-Buffer krediti (lossless po dizajnu) | TCP retransmisija, PFC opciono |
| Latencija | Tipično 1–2 µs po switchu | Veća i promenljiva |
| Mreža | Fizički odvojena, namenska | Deljena sa LAN saobraćajem |
| Kontrola pristupa | Zoning + LUN masking | VLAN, CHAP, ACL |
| Brzine u produkciji | 16G, 32G, 64G | 10G, 25G, 100G |

FC ne koristi TCP/IP stek. Nema IP adresa, nema rutiranja u klasičnom smislu, nema retransmisije na nivou protokola. Ako dođe do gubitka okvira (frame), storage operacija propada i mora je ponoviti sloj iznad — zato se ceo dizajn vrti oko toga da se gubitak ne dogodi.

**Praktična posledica za administratora:** FC mreža je nemilosrdna prema fizičkim greškama. Prljav konektor, savijen kabl ili neodgovarajući SFP ne daju „sporiju vezu", nego CRC greške i prekide I/O operacija. Zato dijagnostika u FC svetu skoro uvek počinje od fizičkog sloja.

---

## 1.2 Komponente SAN-a

Svaki FC SAN se sastoji od tri grupe elemenata:

**1. Inicijatori (initiators)**
Serveri, odnosno njihovi HBA (Host Bus Adapter) portovi. Inicijator je onaj koji pokreće I/O operaciju — traži da se blok pročita ili upiše. U praksi: QLogic ili Emulex kartica u serveru, ili FC port na backup serveru.

**2. Targeti (targets)**
Portovi na storage nizu koji odgovaraju na zahteve inicijatora. Na modernom all-flash nizu to su FC portovi na kontrolerima (nodovima). Trakne biblioteke i VTL uređaji su takođe targeti.

**3. Fabric**
Jedan ili više međusobno povezanih FC switcheva koji prenose saobraćaj i pružaju servise: registraciju uređaja, name server, kontrolu pristupa (zoning) i obaveštavanje o promenama.

```
   ┌──────────┐                                   ┌──────────────┐
   │ ESXi-01  │──HBA port 1──┐         ┌──────────│  Storage     │
   │          │──HBA port 2──┼──┐      │  ┌───────│  Node A / B  │
   └──────────┘              │  │      │  │       └──────────────┘
                       ┌─────┴──┴──┐ ┌─┴──┴─────┐
                       │ SWITCH A  │ │ SWITCH B │   ← dve nezavisne fabrike
                       └───────────┘ └──────────┘
```

### Zašto dve odvojene fabrike (A i B)?

Ovo je standard u svakoj ozbiljnoj produkciji. Svaki server ima najmanje dva FC porta — jedan ide u fabriku A, drugi u fabriku B. Fabrike se **ne povezuju međusobno**.

Razlog nije samo redundansa hardvera. Fabric je jedan logički entitet sa zajedničkom bazom zona i zajedničkim servisima. Greška u konfiguraciji, bagovita firmware verzija ili „fabric merge" incident mogu oboriti celu fabriku odjednom. Ako su fabrike dve i potpuno nezavisne, takav događaj ostavlja drugu polovinu puteva živom, a multipathing na hostu preuzima saobraćaj.

---

## 1.3 Topologije

**Point-to-Point (P2P)** — direktna veza inicijatora i targeta, bez switcha. Sreće se u malim instalacijama, ali ne skalira i ne daje deljenje resursa.

**Arbitrated Loop (FC-AL)** — istorijska topologija u kojoj uređaji dele medijum u prstenu. **Nije podržana na brzinama iznad 8G.** Ovo je važno zapamtiti: ako na modernoj 16G/32G kartici ostavite Connection Mode na „Loop Preferred", link ili neće doći online ili će se ponašati nepredvidivo. Ispravna vrednost je uvek **Point To Point Only**.

**Switched Fabric (FC-SW)** — jedini relevantan dizajn danas. Svaki uređaj ima namensku vezu ka switchu, saobraćaj se prosleđuje kroz fabric.

Kada fabric prevaziđe jedan switch, koriste se dva pristupa:

- **Core-Edge** — serveri se kače na edge switcheve, storage na core switcheve, edge i core su povezani ISL vezama. Standard za veće instalacije.
- **Full Mesh** — svaki switch povezan sa svakim. Praktično do 4 switcha po fabrici.

---

## 1.4 Adresiranje: WWN, WWPN, WWNN, FCID i Domain ID

Ovo je deo koji početnicima najviše smeta, a zapravo je jednostavan kada se razdvoje dva nivoa: **trajni identitet** i **adresa u fabrici**.

### World Wide Name (WWN)

64-bitni, globalno jedinstven identifikator, upisan u hardver — analogno MAC adresi. Zapisuje se kao osam heksadecimalnih bajtova razdvojenih dvotačkom:

```
21:00:f4:e9:d4:56:7a:b1
```

Postoje dve vrste:

- **WWNN** (World Wide Node Name) — identifikuje ceo uređaj (celu HBA karticu, ceo storage niz)
- **WWPN** (World Wide Port Name) — identifikuje pojedinačni port

Dvoportna HBA kartica ima jedan WWNN i dva WWPN-a. **U zoniranju se gotovo uvek koristi WWPN** — zona mora da se odnosi na konkretan port, ne na ceo uređaj.

Prvi nibble govori o formatu adrese, a proizvođač se prepoznaje po OUI delu. Nekoliko orijentira koje je korisno prepoznati na prvi pogled:

| Početak WWPN-a | Tipično znači |
|---|---|
| `21:00:` / `20:00:` | QLogic HBA port |
| `10:00:00:00:c9:` | Emulex HBA port |
| `10:00:` (Brocade OUI) | WWN samog switcha |
| `50:` / `58:` | Storage target port |

### FCID (Fibre Channel Address Identifier)

Za razliku od WWPN-a, FCID je **24-bitna adresa koju switch dodeljuje portu kada se uređaj prijavi u fabric**. Ona se koristi u zaglavlju svakog FC okvira za prosleđivanje — isto kao IP adresa u IP mreži. Sastoji se iz tri bajta:

```
      01      0e      00
      │       │       │
      │       │       └── Port ID   (identifikuje uređaj na tom portu)
      │       └────────── Area ID   (identifikuje port na switchu)
      └────────────────── Domain ID (identifikuje switch u fabrici)
```

Dakle FCID `010e00` znači: switch sa Domain ID 1, area 0x0e (port 14), uređaj 0.

### Domain ID

Broj od 1 do 239 koji jedinstveno identifikuje switch unutar jedne fabrike. **Dva switcha u istoj fabrici ne smeju imati isti Domain ID** — ako se pokušaju povezati, ISL link se segmentira i fabric se ne formira.

> **Napomena iz prakse:** novi switchevi iz fabrike najčešće dolaze sa Domain ID 1 i istim podrazumevanim imenom. Ako planirate da ikada povežete dva switcha u istu fabriku, preimenujte ih i dodelite različite Domain ID-jeve **pre** povezivanja. Promena Domain ID-a zahteva da switch bude disabled, što znači prekid saobraćaja.

---

## 1.5 Tipovi portova

Tip porta govori šta je na drugoj strani kabla. Brocade switch ga određuje automatski, tokom pregovaranja pri uspostavljanju linka.

| Tip | Gde se nalazi | Šta znači |
|---|---|---|
| **N_Port** | Na hostu/storage-u | Node port — port krajnjeg uređaja |
| **F_Port** | Na switchu | Fabric port — switch port na koji je zakačen N_Port |
| **E_Port** | Na switchu | Expansion port — veza ka drugom switchu (ISL) |
| **U_Port** | Na switchu | Universal — port još nije pregovarao ulogu |
| **G_Port** | Na switchu | Generic — link postoji, uloga se određuje |
| **D_Port** | Na switchu | Diagnostic — port u režimu testiranja, ne prenosi podatke |
| **EX_Port** | Na switchu | Veza ka drugoj fabrici preko FC rutera |

Redosled u praksi: port je `U_Port` dok je prazan → postaje `G_Port` kada se pojavi svetlo i počne pregovaranje → završava kao `F_Port` (kačen host/storage) ili `E_Port` (kačen drugi switch).

Ako port ostane zaglavljen kao `G_Port`, pregovaranje ne uspeva — to je jasan signal problema na linku ili nekompatibilnog podešavanja na drugoj strani.

---

## 1.6 Kako uređaj ulazi u fabric

Kada se server upali ili se kabl priključi, odvija se sledeći niz:

1. **Link inicijalizacija** — fizički sloj uspostavlja vezu, pregovara se brzina (auto-negotiation)
2. **FLOGI** (Fabric Login) — N_Port se prijavljuje switchu; switch mu dodeljuje FCID
3. **PLOGI / registracija u Name Server** — uređaj upisuje svoj WWPN, tip (inicijator/target) i podržane protokole u Simple Name Server (SNS)
4. **Provera zoninga** — switch inicijatoru vraća samo one uređaje sa kojima je u istoj aktivnoj zoni
5. **PLOGI ka targetu** — inicijator se prijavljuje targetu i počinje razmena podataka
6. **RSCN** (Registered State Change Notification) — kad god se nešto u fabrici promeni, switch obaveštava sve zainteresovane uređaje

Ključna tačka za razumevanje zoninga: **Name Server ne laže, ali filtrira.** Uređaj je registrovan u fabrici čim uradi FLOGI — vidljiv je administratoru kroz `nsshow` čak i ako nije ni u jednoj zoni. Ali drugi uređaji ga neće videti dok zoning to ne dozvoli. Zato je „host vidi switch, ali ne vidi storage" skoro uvek problem zoninga, a ne fizičkog sloja.

---

## 1.7 Prve komande na switchu

Sve komande koje slede izvršavaju se iz FOS CLI (SSH ili serijska konzola), po pravilu kao korisnik `admin`.

### `switchshow` — najvažnija komanda na switchu

Daje kompletnu sliku: identitet switcha, njegovu ulogu u fabrici, stanje zoninga i status svakog porta.

```
SAN_A_SW01:admin> switchshow
switchName:     SAN_A_SW01
switchType:     183.0
switchState:    Online
switchMode:     Native
switchRole:     Principal
switchDomain:   1
switchId:       fffc01
switchWwn:      10:00:38:ba:b0:fc:9f:b0
zoning:         ON (PROD_CFG)
switchBeacon:   OFF
FC Router:      OFF
Fabric Name:    FABRIC_A
HIF Mode:       OFF
Address Mode:   0

Index Port Address Media Speed State     Proto
=================================================
   0   0   010000   id    N32     Online      FC  F-Port  21:00:f4:e9:d4:56:7a:b1
   1   1   010100   id    N32     Online      FC  F-Port  21:00:f4:e9:d4:56:7a:b2
   2   2   010200   id    N32     Online      FC  F-Port  21:00:f4:e9:d4:56:8c:03
   3   3   010300   id    N16     Online      FC  F-Port  50:06:01:60:88:60:2a:11
   4   4   010400   id    --      No_Light    FC
   5   5   010500   --   --       No_Module   FC
   6   6   010600   id    N32     Online      FC  E-Port  10:00:38:ba:b0:fc:aa:20 "SAN_A_SW02"
   7   7   010700   id    --      No_Sync     FC   Disabled (Persistent)
```

**Kako se čita tabela portova:**

- **Index / Port** — redni broj porta (kod većine 1U modela su identični)
- **Address** — FCID dodeljen tom portu; prvi bajt je Domain ID
- **Media** — `id` znači da je SFP prisutan i prepoznat, `--` da modula nema
- **Speed** — `N32` = auto-negotiated na 32G; `32G` bi značilo ručno fiksirano
- **State** — stanje linka (tabela ispod)
- **Proto / tip porta** — uloga porta i WWPN uređaja na drugoj strani

Stanja porta koja ćete najčešće videti:

| State | Značenje | Šta proveriti |
|---|---|---|
| `Online` | Sve radi, port prenosi saobraćaj | — |
| `No_Module` | Nema SFP modula u portu | Da li je SFP uopšte ubačen |
| `No_Light` | SFP postoji, ali ne prima svetlo | Kabl, druga strana linka, da li je kabl u pravom slotu |
| `No_Sync` | Prima svetlo, ali ne može da se sinhronizuje | Neusklađena brzina, loš kabl, prljav konektor |
| `In_Sync` | Sinhronizovan, ali FC login nije završen | Zaglavljeno pregovaranje |
| `Laser_Flt` | Kvar lasera na SFP-u | Zameniti SFP |
| `Disabled` | Port administrativno isključen | `portenable` |

> Primetite port 4 u primeru: modul je prisutan (`id`), ali nema svetla. To je klasična situacija „kabl nije priključen na drugoj strani" — ili je priključen, ali u pogrešan port. Konektori 32G FC SFP-a i 25G Ethernet SFP28 modula su fizički identični, pa je zamena slotova na serveru realna i neprijatno česta greška.

### `fabricshow` — ko je sve u fabrici

```
SAN_A_SW01:admin> fabricshow
Switch ID   Worldwide Name           Enet IP Addr    FC IP Addr      Name
--------------------------------------------------------------------------------
 1: fffc01  10:00:38:ba:b0:fc:9f:b0  10.10.10.11     0.0.0.0     >"SAN_A_SW01"
 2: fffc02  10:00:38:ba:b0:fc:aa:20  10.10.10.12     0.0.0.0      "SAN_A_SW02"

The Fabric has 2 switches
Fabric Name: FABRIC_A
```

Znak `>` označava **principal switch** — onaj koji je izabran da dodeljuje Domain ID-jeve i koordinira fabric servise. Ako komanda vrati samo jedan switch iako ih je povezano više, fabric se nije formirao (segmentacija).

### `nsshow` — Name Server, lokalno

Prikazuje sve uređaje prijavljene na **ovom** switchu.

```
SAN_A_SW01:admin> nsshow
{
 Type Pid    COS     PortName                NodeName                 TTL(sec)
 N    010000;      3;21:00:f4:e9:d4:56:7a:b1;20:00:f4:e9:d4:56:7a:b1; na
    FC4s: FCP
    PortSymb: [39] "QLE2772 FW:v9.12.01 DVR:v5.2.11.0 ESXi-01"
    NodeSymb: [24] "ESXi-01.domain.local"
    Fabric Port Name: 20:00:38:ba:b0:fc:9f:b0
    Permanent Port Name: 21:00:f4:e9:d4:56:7a:b1
    Port Index: 0
    Share Area: No
    Device Shared in Other AD: No
    Redirect: No
    Partial: No
    LSAN: No
 N    010300;      3;50:06:01:60:88:60:2a:11;50:06:01:60:08:60:2a:11; na
    FC4s: FCP
    PortSymb: [28] "Storage-NodeA-Port0"
    Fabric Port Name: 20:03:38:ba:b0:fc:9f:b0
    Port Index: 3
The Local Name Server has 4 entries }
```

Za svaki uređaj ovde vidite ono što je potrebno za zoniranje: **Pid** (FCID), **PortName** (WWPN) i simbolično ime koje uređaj sam prijavljuje — često dovoljno da prepoznate o kom serveru ili kontroleru je reč.

Za pregled cele fabrike, uključujući uređaje na drugim switchevima, koristi se `nscamshow`.

### `portshow` — detalji jednog porta

```
SAN_A_SW01:admin> portshow 0
portIndex:  0
portName:   ESXi-01_HBA1
portHealth: HEALTHY
Authentication: None
portDisableReason: None
portCFlags: 0x1
portType:  24.0
portState: 1    Online
portPhys:  6    In_Sync
portScn:   32   F_Port
port generation number:    18
state transition count:    3
portId:    010000
portIfId:  4302000a
portWwn:   20:00:38:ba:b0:fc:9f:b0
Distance:  normal
portSpeed: N32Gbps
LE domain: 0
Interrupts:   0        Link_failure: 0      Frjt: 0
Unknown:      0        Loss_of_sync: 1      Fbsy: 0
Lli:          58       Loss_of_sig:  2
Proc_rqrd:    112      Protocol_err: 0
```

Brojači na dnu su prvi indikator zdravlja linka. Nule ili vrlo male vrednosti su normalne. Brojač koji **raste dok gledate** znači aktivan problem na fizičkom sloju.

### `switchstatusshow` — brzi pregled zdravlja

```
SAN_A_SW01:admin> switchstatusshow
Switch Health Report                 Report time: 09/15/2026 10:14:22
Switch Name: SAN_A_SW01
IP address:  10.10.10.11
SwitchState: HEALTHY

Contributing factors:
------------------------------------
PowerSupplies monitor           HEALTHY
Temperatures monitor            HEALTHY
Fans monitor                    HEALTHY
Flash monitor                   HEALTHY
Marginal ports monitor          HEALTHY
Faulty ports monitor            HEALTHY
Missing SFPs monitor            HEALTHY
Error ports monitor             HEALTHY
```

Ako je `SwitchState` nešto drugo osim `HEALTHY`, pogledajte koji monitor nije zelen — to odmah sužava pretragu.

---

## 1.8 Rezime poglavlja

- FC je namenska, lossless mreža za blokovski storage; fizički sloj je kritičan i dijagnostika počinje odatle.
- Produkcijski SAN se gradi kao **dve nezavisne fabrike** (A i B) koje se nikada ne spajaju.
- **WWPN** je trajni identitet porta i osnovna jedinica zoniranja; **FCID** je adresa koju switch dodeljuje pri prijavi.
- **Domain ID** mora biti jedinstven unutar fabrike.
- Tip porta (`F_Port`, `E_Port`, `G_Port`) govori šta je na drugoj strani i da li je pregovaranje uspelo.
- Uređaj koji je u Name Serveru, a nedostupan drugom uređaju — to je problem zoninga, ne kabla.
- Četiri komande pokrivaju 80% svakodnevnog rada: `switchshow`, `fabricshow`, `nsshow`, `portshow`.

---

## 1.9 Provera znanja

1. Port u `switchshow` izlazu prikazuje `id` u koloni Media, ali stanje `No_Light`. Šta to konkretno znači i gde se traži uzrok?
2. FCID porta je `020a00`. Na kom switchu se port nalazi i koji je redni broj porta?
3. Host je vidljiv u `nsshow` izlazu, ali na ESXi hostu nema nijednog LUN-a. Da li je uzrok verovatnije fizički sloj ili konfiguracija? Zašto?
4. Zašto se u zonama koristi WWPN a ne WWNN?
5. Dva nova switcha treba povezati u jednu fabriku. Oba imaju Domain ID 1. Šta se dešava ako ih povežete ISL kablom bez izmena?

---

**Sledeće poglavlje:** Hardver fabrike — SFP moduli, talasne dužine, optički kablovi i licenciranje portova.
