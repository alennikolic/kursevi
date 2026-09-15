# Poglavlje 2: Hardver fabrike — switchevi, SFP moduli, kablovi i licence

> **Cilj poglavlja:** Prepoznati sa kojim hardverom radite, razumeti optički deo linka (moduli, kablovi, snaga signala) i znati koliko portova na switchu je zaista licencirano i upotrebljivo.

---

## 2.1 Vrste Brocade uređaja

Brocade FC portfolio se deli u dve kategorije:

**Fixed-port switchevi** — kompaktni uređaji, 1U ili 2U, sa fiksnim brojem portova. Ovo je ono što se sreće u 90% instalacija.

**Directori (chassis)** — modularna kućišta sa slotovima za port blejdove, dvostrukim kontrolnim procesorima i hot-swap komponentama. Koriste se u velikim data centrima gde je broj portova veliki, a prekid rada nedopustiv.

| Generacija | Brzina | Tipični fixed-port modeli | Directori |
|---|---|---|---|
| Gen 5 | 16G | 6505, 6510, 6520 | DCX 8510 |
| Gen 6 | 32G | G610, G620, G630 | X6-4, X6-8 |
| Gen 7 | 64G | G720, G730 | X7-4, X7-8 |

### OEM preprodaja — isti uređaj, drugo ime

Brocade svoje switcheve prodaje kroz partnere, koji ih rebrendiraju. Hardver i firmware su isti, menja se samo nalepnica i deo broj:

| Brocade | Dell EMC | HPE |
|---|---|---|
| G610 | DS-6610B | SN3600B |
| G620 | DS-6620B | SN6600B |
| G720 | DS-7720B | SN6700B |

**Praktična posledica:** kada tražite dokumentaciju ili firmware, uvek prevedite OEM oznaku u Brocade model. Dell-ov `DS-7720B` u Brocade dokumentaciji figurira isključivo kao `G720`. Firmware (FOS) preuzimate od OEM proizvođača od koga ste kupili uređaj — Dell i HPE objavljuju sopstvene, sertifikovane verzije.

> Napomena o brzini: G720 je Gen 7 uređaj, ali se u praksi najčešće koristi sa 32G modulima jer HBA kartice i storage portovi na 64G tek postaju uobičajeni. Switch to podržava bez problema — modul određuje brzinu porta.

---

## 2.2 Fizički pregled uređaja

Na prednjoj ili zadnjoj strani switcha nalaze se:

- **FC portovi** sa LC konektorima, numerisani sleva nadesno
- **Serijski konzolni port** (RJ-45, RS-232) — jedini pristup kada nema mrežne konfiguracije
- **Management Ethernet port** (`eth0`) — za SSH i web pristup
- **USB port** — za firmware i backup konfiguracije
- **Napajanja i ventilatori** — na modelima za produkciju redundantni i hot-swap

### Provera stanja hardvera

```
SAN_A_SW01:admin> chassisshow
FAN Unit: 1
Time Awake:                 412 days

FAN Unit: 2
Time Awake:                 412 days

POWER SUPPLY Unit: 1
Manufacturer:               DELL
Serial Num:                 CN0XXXXX1234
Part Num:                   0XXXX
Time Awake:                 412 days

POWER SUPPLY Unit: 2
Manufacturer:               DELL
Serial Num:                 CN0XXXXX5678
Part Num:                   0XXXX
Time Awake:                 412 days

CHASSIS/WWN Unit: 1
Header Version:             2
Power Consume Factor:       -200
Brocade Part Num:           40-1000675-04
Serial Num:                 FTX0000X00J
Manufacture:                Day: 14  Month: 3  Year: 2023
Update:                     Day: 15  Month: 9  Year: 2026
Time Alive:                 1276 days
Time Awake:                 412 days
```

Serijski broj iz `CHASSIS/WWN Unit` sekcije je onaj koji tražite kada otvarate slučaj kod podrške.

```
SAN_A_SW01:admin> psshow
Power Supply #1 is OK
Power Supply #2 is OK

SAN_A_SW01:admin> fanshow
Fan 1 is Ok, speed is 7420 RPM
Fan 2 is Ok, speed is 7350 RPM

SAN_A_SW01:admin> tempshow
Sensor ID   Category    Status     Temperature(C)
---------------------------------------------------
      1     Switch        Ok             38
      2     Switch        Ok             41
      3     Switch        Ok             36
```

Ako `psshow` prijavi `Faulty` ili `Absent` na jednom napajanju, switch i dalje radi — ali ste ostali bez redundanse i to treba rešiti isti dan.

### LED indikatori na portovima

| LED stanje | Značenje |
|---|---|
| Ugašen | Nema SFP-a ili nema signala |
| Zeleno, stalno | Port online, link uspostavljen |
| Zeleno, treperi | Saobraćaj u toku |
| Žuto, stalno | Port prima signal, ali nema sinhronizacije |
| Žuto, treperi sporo | Port disabled |
| Žuto, treperi brzo | Greška na portu ili neuspeo dijagnostički test |
| Naizmenično zeleno/žuto | Port u beacon režimu ili neuspeo POST |

---

## 2.3 SFP moduli

SFP (Small Form-factor Pluggable) modul je optički primopredajnik koji se umeće u port. On, a ne switch, određuje domet i talasnu dužinu linka.

### Tipovi po dometu

| Oznaka | Puni naziv | Talasna dužina | Vlakno | Domet |
|---|---|---|---|---|
| **SW** | Short Wave | 850 nm | Multimode (MMF) | do ~100 m na 32G |
| **LW** | Long Wave | 1310 nm | Singlemode (SMF) | do 10 km |
| **ELW** | Extended Long Wave | 1550 nm | Singlemode | do 25–80 km |

**Obe strane linka moraju imati istu talasnu dužinu.** SW modul na jednoj i LW na drugoj strani ne uspostavljaju link, bez obzira na to što se konektor uklapa.

### Brzine i unazadna kompatibilnost

32G FC SFP podržava 32G, 16G i 8G. 16G modul podržava 16G, 8G i 4G. Pravilo je „tekuća generacija plus dve unazad". To znači da 32G modul **neće** raditi sa 4G uređajem.

Ako host ima 16G HBA, a switch 32G modul, link će se auto-negotiate na 16G i raditi normalno.

### Zamka koju treba znati napamet

**32G FC SFP i 25G Ethernet SFP28 modul imaju fizički identičan konektor i identično kućište.** Isto važi za kablove. Nema mehaničke zaštite koja bi vas sprečila da FC kabl ubodete u Ethernet port na serveru — i obrnuto. Rezultat je link koji „nikako da proradi", a sva merenja na switchu izgledaju ispravno.

Ako na switchu vidite `No_Light` na portu, a `sfpshow` pokazuje da modul na switch strani normalno emituje, uvek fizički proverite u koji slot na serveru je kabl zaista utaknut — pre nego što krenete da menjate kablove i module.

### `sfpshow` — najkorisnija dijagnostička komanda u ovom poglavlju

```
SAN_A_SW01:admin> sfpshow 0
Identifier:  3    SFP
Connector:   7    LC
Transceiver: 7004404000000000 800,1600,3200_MB/s M5 sw Short_dist
Encoding:    6    64B66B
Baud Rate:   285  (units 100 megabaud)
Length 50u (OM3): 7  (units 10 meters)
Length 50u (OM2): 3  (units 10 meters)
Vendor Name: BROCADE
Vendor OUI:  00:05:1e
Vendor PN:   57-1000335-01
Vendor Rev:  A
Wavelength:  850  (units nm)
Options:     003a Loss_of_Sig,Tx_Fault,Tx_Disable
Serial No:   HAF3203400001J
Date Code:   230815
DD Type:     0x68
Status/Ctrl: 0x30
Pwr On Time: 2.31 years (20264 hours)
Alarm flags[0,1] = 0x0, 0x0
Warn  flags[0,1] = 0x0, 0x0

                                            Alarm                  Warn
                                       low        high        low       high
Temperature: 42      Centigrade        -5         85          0         75
Current:     7.712   mAmps             1.000      12.000      2.000     11.000
Voltage:     3341.5  mVolts            3000.0     3600.0      3100.0    3500.0
Rx Power:    -2.4    dBm (575.1uW)     -13.9      0.0         -12.0     -1.0
Tx Power:    -1.8    dBm (660.0uW)     -11.4      1.0         -10.0      0.0

State transitions: 3
```

**Šta se čita iz ovog izlaza:**

- **Wavelength: 850** — SW modul, ide na multimode vlakno
- **Transceiver: ... 800,1600,3200_MB/s** — podržane brzine: 8G, 16G, 32G
- **Vendor Name / PN** — proizvođač i deo broj modula
- **Status/Ctrl** — bitovi stanja; vrednost `0x0` na portu gde očekujete aktivan link je sumnjiva, dok `0x30` znači da su rate-select biti postavljeni i modul radi normalno
- **Pwr On Time** — koliko dugo je modul u pogonu; kod starih modula deo objašnjenja degradacije
- **Alarm / Warn flags** — nule znače da nijedna vrednost nije van dozvoljenog opsega

### Pregled svih modula odjednom

```
SAN_A_SW01:admin> sfpshow -all
Port  0: id (sw)  Vndr: BROCADE   Serial No: HAF3203400001J  Speed: 8,16,32_Gbps
   Temp: 42 C   Volt: 3341.5 mV   Curr: 7.712 mA   RXP: -2.4 dBm   TXP: -1.8 dBm
Port  1: id (sw)  Vndr: BROCADE   Serial No: HAF3203400002K  Speed: 8,16,32_Gbps
   Temp: 43 C   Volt: 3339.8 mV   Curr: 7.688 mA   RXP: -2.6 dBm   TXP: -1.9 dBm
Port  4: id (sw)  Vndr: BROCADE   Serial No: HAF3203400005M  Speed: 8,16,32_Gbps
   Temp: 41 C   Volt: 3340.1 mV   Curr: 7.701 mA   RXP: -40.0 dBm  TXP: -1.8 dBm
Port  5: No SFP module
```

Port 4 je udžbenički primer: modul emituje normalno (`TXP: -1.8 dBm`), ali ne prima ništa (`RXP: -40.0 dBm`). Problem je na putu od druge strane ka nama — kabl, konektor ili uređaj na drugom kraju.

---

## 2.4 Optički budžet — kako čitati snagu signala

Snaga se izražava u **dBm**, logaritamskoj skali u odnosu na 1 mW. Nula dBm je 1 mW; negativne vrednosti znače manje od milivata, što je normalno za FC linkove.

Orijentacione vrednosti za 32G SW link:

| Rx Power | Tumačenje |
|---|---|
| od -1 do -6 dBm | Zdrav link |
| od -6 do -9 dBm | Prihvatljivo, ali vredi proveriti konektore |
| od -9 do -12 dBm | Marginalno; očekujte CRC greške |
| ispod -14 dBm | Ispod praga osetljivosti, link neće raditi pouzdano |
| -40 dBm | Praktično nula — ne stiže nikakvo svetlo |

Osnovno pravilo: **razlika između Tx snage na jednoj strani i Rx snage na drugoj je gubitak na putanji.** Ako jedna strana šalje -1.8 dBm, a druga prima -2.4 dBm, gubitak je 0.6 dB — odlično za kratak patch kabl. Gubitak od 4–5 dB na kablu od 10 metara znači prljav ili oštećen konektor.

Najčešći uzroci prevelikog gubitka, po učestalosti:

1. Prljav konektor (prašina, otisak prsta) — rešava se namenskim alatom za čišćenje optike
2. Loše ubačen konektor — ne čuje se „klik"
3. Previše spojeva na putanji (patch paneli)
4. Savijen kabl ispod dozvoljenog poluprečnika
5. Star ili degradiran modul

---

## 2.5 Kablovi

### Multimode (za SW module)

| Tip | Boja omotača | Domet na 16G | Domet na 32G |
|---|---|---|---|
| OM2 | oranžna | 35 m | 20 m |
| OM3 | tirkizna | 100 m | 70 m |
| OM4 | tirkizna ili ljubičasta | 125 m | 100 m |
| OM5 | zeleno-žuta | 125 m | 100 m |

**Domet opada sa porastom brzine.** Kabl koji je bio sasvim u redu za 8G link može biti prekratak za 32G ako je duži i lošije kategorije. U data centru se praktično uvek koristi OM4.

### Singlemode (za LW module)

OS2, žuti omotač, domet do 10 km sa standardnim LW modulom. Koristi se za veze između lokacija.

### Konektori i polaritet

FC koristi **LC duplex** konektor — dva vlakna u jednom kućištu, jedno za slanje, jedno za prijem. Polaritet mora biti ukršten: Tx sa jedne strane ide na Rx sa druge. Kod fabrički napravljenih duplex patch kablova to je već rešeno.

Problem nastaje kod strukturnog kabliranja preko patch panela, gde je moguće da je negde u putanji polaritet zamenjen. Simptom je karakterističan: **obe strane emituju, nijedna ne prima.** Ako na oba kraja vidite normalan Tx i -40 dBm Rx, prvo posumnjajte na polaritet, a ne na kvar.

### Higijena kablova

- Ne savijati ispod poluprečnika od oko 30 mm
- Držati zaštitne kapice na neiskorišćenim konektorima i modulima
- Ne oslanjati se na duvanje komprimovanim vazduhom — koristiti namenske olovke ili kasete za čišćenje
- Obeležiti oba kraja svakog kabla pre postavljanja

---

## 2.6 Licenciranje portova (Ports on Demand)

Brocade switchevi se isporučuju sa fizički prisutnim portovima kojih je više nego što je licencirano. Broj upotrebljivih portova se kupuje — to je **POD** model.

**Base allowance** je broj portova uključen u cenu uređaja. Dodatne portove otključava POD licenca.

Postoje dva načina dodele:

- **Dynamic POD** (podrazumevano) — licenca se automatski dodeljuje portu koji prvi dođe online. Ako se port isključi i drugi port se poveže, rezervacija može preći na njega.
- **Static POD** — administrator ručno vezuje licencu za konkretan port broj.

Dynamic je pogodniji u većini slučajeva jer ne zahteva intervenciju pri prekabliranju.

### Provera licenci

```
SAN_A_SW01:admin> licenseshow
bQebzbTbRdDtfTQST:
    Fabric Vision license
eRRSzSQdSceTcQfS:
    Trunking license
RdcSSbQRSdTtdQcT:
    Extended Fabric license
```

```
SAN_A_SW01:admin> licenseport --show
  24 ports are available in this switch
  Full POD license is not installed
  Dynamic POD method is in use

  24 port assignments are provisioned for use in this switch:
     24 port assignments are provisioned by the base switch allowance
      0 port assignments are provisioned by a full POD license

  8 ports are assigned to installed licenses:
      8 ports are assigned to the base switch allowance
      0 ports are assigned to the full POD license

  Ports assigned to the base switch allowance:
    0, 1, 2, 3, 4, 5, 6, 7

  Ports assigned to the full POD license:
    None

  Ports not assigned to a license:
    8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23

  16 license reservations are still available for use by unassigned ports
```

Ovaj izlaz čitajte odozdo nagore: 24 porta je licencirano, 8 ih je trenutno u upotrebi, još 16 rezervacija je slobodno. Sve dok taj poslednji broj nije nula, novi uređaj koji priključite dobiće licencu automatski.

**Simptom iscrpljene licence:** port je fizički ispravan, SFP prisutan, svetlo stiže — ali port ostaje disabled sa razlogom koji upućuje na licencu. Provera:

```
SAN_A_SW01:admin> portshow 24 | grep portDisableReason
portDisableReason: No POD License
```

### Instalacija licence

```
SAN_A_SW01:admin> licenseadd "bQebzbTbRdDtfTQST"
adding license-key [bQebzbTbRdDtfTQST]

SAN_A_SW01:admin> licenseshow
bQebzbTbRdDtfTQST:
    Ports on Demand license - additional 16 port upgrade license
```

Licenca se generiše na osnovu **License ID** uređaja, koji se dobija sa:

```
SAN_A_SW01:admin> licenseidshow
10:00:38:ba:b0:fc:9f:b0
```

Neke licence zahtevaju restart switcha da bi stupile na snagu — `licenseadd` će to eksplicitno prijaviti ako je slučaj.

---

## 2.7 Rezime poglavlja

- Uvek prevedite OEM oznaku uređaja u Brocade model pre traženja dokumentacije; firmware uzimajte od svog OEM proizvođača.
- SFP modul određuje talasnu dužinu i domet — obe strane linka moraju biti iste vrste (SW sa SW, LW sa LW).
- 32G FC i 25G Ethernet SFP moduli su fizički nerazlučivi; pogrešan slot na serveru je realna i teško uočljiva greška.
- `sfpshow` daje Tx i Rx snagu — razlika između njih na dva kraja je gubitak na putanji.
- Tx normalan a Rx na -40 dBm znači da problem nije u ovom modulu, nego na putu ka njemu.
- Domet multimode kabla opada sa porastom brzine; za 32G planirati OM4.
- Broj upotrebljivih portova je licencno pitanje — `licenseport --show` pre nego što obećate kolegama slobodne portove.

---

## 2.8 Provera znanja

1. `sfpshow` na portu 3 pokazuje `Tx Power: -1.9 dBm` i `Rx Power: -40.0 dBm`. Na drugoj strani linka, isti port pokazuje identične vrednosti. Koji uzrok je najverovatniji?
2. Link između dva switcha na udaljenosti od 600 metara ne radi, iako su na obe strane SW moduli i OM4 kabl. Šta je problem?
3. Rx snaga na portu je -10.5 dBm. Link radi, ali u `porterrshow` rastu CRC greške. Šta uraditi pre nego što se naruči novi modul?
4. Switch ima 24 licencirana porta, svi su zauzeti, a treba priključiti još jedan server sa dva porta. Koje su vam dve opcije?
5. Zašto 32G SFP modul ne može da radi sa 4G uređajem?

---

**Sledeće poglavlje:** Prvi pristup switchu i inicijalna konfiguracija — konzola, IP adresiranje, ime, Domain ID, vreme i korisnički nalozi.
