# Poglavlje 6: Zoniranje I — koncepti

> **Cilj poglavlja:** Razumeti šta zoning zaista radi, od kojih se objekata sastoji, koje vrste članstva postoje i po kojim se pravilima projektuje zoning koji je bezbedan i održiv. Praktično kreiranje zona sledi u narednom poglavlju.

---

## 6.1 Zašto zoning postoji

Zamislite fabric bez zoninga. Svaki inicijator vidi svaki target, i svaki inicijator vidi svaki drugi inicijator. Za mrežnog administratora to zvuči kao jedan veliki VLAN — neuredno, ali ne i katastrofalno. U FC svetu posledice su ozbiljnije.

**1. Rizik po podatke.** Storage niz LUN-ove prezentuje inicijatorima. Ako pogrešan host vidi LUN koji nije njegov, a taj host je operativni sistem koji rutinski upisuje potpis na svaki novootkriveni disk — podaci su uništeni. Ovo nije hipotetički scenario; to je klasičan način da se obori produkcijska baza.

**2. RSCN lavina.** Svaka promena u fabrici generiše obaveštenje (RSCN) koje ide svim uređajima koji su u istoj zoni sa uređajem koji se promenio. Bez zoninga, restart jednog servera šalje obaveštenje svima. Svaki HBA na to reaguje ponovnim skeniranjem. U okruženju sa stotinu portova, jedan restart može izazvati talas ponovnih skeniranja koji privremeno uspori ceo SAN.

**3. Inicijatori međusobno.** Dva HBA porta nemaju nikakvog razloga da se vide. Neki stariji drajveri se ponašaju nepredvidivo kada u fabrici zateknu druge inicijatore.

**4. Ograničavanje posledica kvara.** Uređaj koji se ponaša loše — port koji stalno pada i diže se, HBA sa bagovitim firmware-om — utiče samo na uređaje sa kojima deli zonu.

Zoning je dakle istovremeno mera bezbednosti, mera stabilnosti i alat za izolaciju kvarova.

> **Važno razgraničenje:** zoning kontroliše **ko koga vidi u fabrici**. On ne određuje koji LUN je dodeljen kom hostu — to je LUN masking i radi se na storage nizu. Oba sloja moraju biti ispravno podešena. Host koji je u zoni sa storage portom, ali kome nije dodeljen nijedan LUN, videće target i neće videti nijedan disk.

---

## 6.2 Objekti zoninga

Zoning je hijerarhija od tri nivoa:

```
   Zone Configuration (cfg)          ← skup zona; samo jedna je aktivna u fabrici
        │
        ├── Zone                     ← skup članova koji smeju da se vide
        │     │
        │     ├── Alias              ← simboličko ime za jedan ili više WWPN-ova
        │     └── WWPN               ← može i direktno, bez alias-a
        │
        └── Zone
              └── ...
```

### Alias

Simboličko ime za WWPN. Nije obavezan, ali je u praksi neophodan — bez njega zoning baza postaje nečitljiva lista heksadecimalnih brojeva.

```
alias: ESXi01_HBA1_P0
       21:00:f4:e9:d4:56:7a:b1
```

Kada zamenite HBA karticu u serveru, menjate WWPN na jednom mestu — u alias-u — a sve zone koje ga koriste ostaju netaknute. Bez alias-a, isti posao znači izmenu svake zone pojedinačno.

### Zone

Skup članova koji smeju međusobno da komuniciraju. Članstvo je simetrično: ako su A i B u istoj zoni, oba vide jedan drugog.

```
zone: ESXi01_P0__STG_NodeA_P0
      ESXi01_HBA1_P0; STG_NodeA_P0
```

Jedan uređaj može biti član većeg broja zona — i to je normalno. ESXi host koji pristupa sa dva porta ka dva storage kontrolera biće u više zona.

### Zone konfiguracija (cfg)

Skup zona. U fabrici može postojati više definisanih konfiguracija, ali je **u svakom trenutku aktivna najviše jedna**.

```
cfg: PROD_CFG
     zona1; zona2; zona3; ...
```

Mogućnost da postoji više konfiguracija koristi se za pripremu izmena unapred ili za brzo vraćanje na prethodno stanje.

### Defined naspram Effective

Ovo je razlika koju treba usvojiti odmah:

- **Defined configuration** — sve što je definisano i sačuvano u zoning bazi, bez obzira na to da li je aktivno
- **Effective configuration** — ono što je trenutno na snazi u celoj fabrici

```
SAN_A_SW01:admin> cfgshow
Defined configuration:
 cfg:   PROD_CFG
        ESXi01_P0__STG_NodeA_P0; ESXi01_P1__STG_NodeB_P0;
        ESXi02_P0__STG_NodeA_P0; ESXi02_P1__STG_NodeB_P0
 cfg:   TEST_CFG
        TEST_HOST__STG_NodeA_P0
 zone:  ESXi01_P0__STG_NodeA_P0
        ESXi01_HBA1_P0; STG_NodeA_P0
 zone:  ESXi01_P1__STG_NodeB_P0
        ESXi01_HBA1_P1; STG_NodeB_P0
 alias: ESXi01_HBA1_P0
        21:00:f4:e9:d4:56:7a:b1
 alias: ESXi01_HBA1_P1
        21:00:f4:e9:d4:56:7a:b2
 alias: STG_NodeA_P0
        50:06:01:60:88:60:2a:11
 alias: STG_NodeB_P0
        50:06:01:61:88:60:2a:12

Effective configuration:
 cfg:   PROD_CFG
 zone:  ESXi01_P0__STG_NodeA_P0
        21:00:f4:e9:d4:56:7a:b1
        50:06:01:60:88:60:2a:11
 zone:  ESXi01_P1__STG_NodeB_P0
        21:00:f4:e9:d4:56:7a:b2
        50:06:01:61:88:60:2a:12
```

Obratite pažnju: u efektivnoj konfiguraciji **alias-i su razrešeni u WWPN-ove**. To je zato što je efektivna konfiguracija ono što switch zaista sprovodi, a sprovođenje se radi nad adresama, ne nad imenima.

Odatle sledi praktična posledica koja iznenađuje: **izmena alias-a ne deluje dok se konfiguracija ponovo ne aktivira.** Ako zamenite WWPN u alias-u i sačuvate, efektivna konfiguracija i dalje sadrži stari WWPN.

Samo aktivna konfiguracija:

```
SAN_A_SW01:admin> cfgactvshow
Effective configuration:
 cfg:   PROD_CFG
 zone:  ESXi01_P0__STG_NodeA_P0
        21:00:f4:e9:d4:56:7a:b1
        50:06:01:60:88:60:2a:11
```

---

## 6.3 Vrste članstva u zoni

Član zone može biti definisan na nekoliko načina.

### WWPN zoning (preporučeno)

Član je WWPN porta:

```
zone: ESXi01_P0__STG_NodeA_P0
      21:00:f4:e9:d4:56:7a:b1; 50:06:01:60:88:60:2a:11
```

Zona prati uređaj. Ako server premestite u drugi port na switchu — čak i na drugi switch u istoj fabrici — zona i dalje važi.

### Port zoning (Domain, Index)

Član je fizička pozicija: domain broj i indeks porta.

```
zone: ESXi01_P0__STG_NodeA_P0
      1,0; 1,3
```

Zona prati port, ne uređaj. Ako neko priključi drugi server u taj port, taj server automatski nasleđuje pristup.

### Poređenje

| | WWPN zoning | Port zoning |
|---|---|---|
| Premeštanje kabla u drugi port | Zona i dalje radi | Zona prestaje da važi |
| Zamena HBA kartice | Mora se izmeniti zona | Zona i dalje radi |
| Neko priključi tuđi server u port | Nema pristup | Dobija pun pristup |
| Čitljivost baze | Dobra uz alias-e | Dobra |
| Preporuka | **Standard u većini okruženja** | Retko, u posebnim slučajevima |

WWPN zoning je izbor u gotovo svim modernim instalacijama. Argument bezbednosti prevladava: pristup je vezan za identitet uređaja, a ne za mesto u rack-u.

### WWNN zoning — ne koristiti

Tehnički je moguće u zonu staviti WWNN (ime čvora, ne porta). Rezultat je da svi portovi tog uređaja dobijaju pristup. To ruši kontrolu koju zoning treba da pruži i onemogućava precizno razdvajanje putanja. Izbegavajte.

### Mešanje tipova članstva

Moguće je u istoj zoni imati i WWPN i port člana. **Nemojte.** Osim što je nečitljivo, u nekim kombinacijama sprovođenje zoninga pada sa hardverskog na sesijski nivo, što je slabija zaštita.

---

## 6.4 Kako switch sprovodi zoning

**Hardverski sprovedeno (frame-level)** — ASIC čip u switchu proverava svaki okvir i odbacuje one koji krše zoning. Ovo je potpuna zaštita i podrazumevano stanje kada je zoning uredno napravljen.

**Sesijski sprovedeno (soft zoning)** — switch filtrira odgovore Name Servera, tako da inicijator jednostavno ne sazna da target postoji. Ali ako inicijator nekim putem već zna adresu, okviri prolaze. Ovo je slabija zaštita i javlja se u rubnim slučajevima, najčešće pri mešanju tipova članstva.

Praktično pravilo: **držite se isključivo WWPN članstva i sprovođenje će biti hardversko.**

---

## 6.5 Podrazumevana zona (Default Zone)

Ovo je koncept koji često izmakne, a može da izazove ozbiljan incident.

Pitanje: šta se dešava sa uređajima koji **nisu ni u jednoj zoni**, ili šta se dešava kada zoning uopšte nije aktiviran?

Odgovor zavisi od podešavanja podrazumevane zone:

```
SAN_A_SW01:admin> defzone --show
Default Zone Access Mode
  committed - No Access
  transaction - No Transaction
```

Dve moguće vrednosti:

| Režim | Ponašanje |
|---|---|
| **No Access** | Uređaj koji nije ni u jednoj aktivnoj zoni ne vidi nikoga |
| **All Access** | Uređaj koji nije ni u jednoj aktivnoj zoni vidi **sve ostale** |

```
SAN_A_SW01:admin> defzone --noaccess
SAN_A_SW01:admin> cfgsave
```

**Zašto je ovo važno:** ako je podrazumevana zona na `All Access`, a neko izvrši `cfgdisable` — fabric u tom trenutku postaje potpuno otvoren. Svaki host vidi svaki storage port. Umesto da izgubite pristup, dobijate scenario iz uvoda ovog poglavlja, sa svim rizicima za podatke.

Sa `No Access`, `cfgdisable` znači da niko nikoga ne vidi. To jeste prekid rada, ali je prekid rada uvek bolji od tihog oštećenja podataka.

**Postavite `defzone --noaccess` na svakom produkcijskom switchu.**

---

## 6.6 Pravila dizajna

### Single Initiator — Single Target

Zlatno pravilo: **jedna zona sadrži tačno jedan inicijator i tačno jedan target.**

```
zone: ESXi01_P0__STG_NodeA_P0
      21:00:f4:e9:d4:56:7a:b1     ← jedan HBA port
      50:06:01:60:88:60:2a:11     ← jedan storage port
```

Prednosti:

- RSCN obaveštenja idu samo jednom uređaju
- Kvar jednog uređaja ne dotiče druge
- Iz imena zone se odmah vidi koja je putanja u pitanju
- Dijagnostika je trivijalna: jedna zona, jedna putanja

Cena: broj zona raste. Četiri hosta sa po dva porta i dva storage kontrolera sa po jednim portom po fabrici daju osam zona po fabrici. To je normalno i prihvatljivo.

### Varijanta: Single Initiator — Multiple Target

Neki proizvođači dozvoljavaju jedan inicijator i više targeta iste storage familije u jednoj zoni. To smanjuje broj zona uz zadržavanje ključnog principa — **nikada dva inicijatora u istoj zoni**.

```
zone: ESXi01_P0__STG_FabricA
      21:00:f4:e9:d4:56:7a:b1     ← jedan inicijator
      50:06:01:60:88:60:2a:11     ← target 1
      50:06:01:61:88:60:2a:12     ← target 2
```

Pre nego što se odlučite za ovu varijantu, proverite šta preporučuje proizvođač vašeg storage niza. Njihova matrica podržanih konfiguracija je merodavna.

### Šta nikada ne raditi

- **Jedna velika zona sa svim uređajima.** Poništava svrhu zoninga.
- **Dva inicijatora u istoj zoni.** Ni u kom slučaju.
- **Zone bez imenske konvencije.** `zone1`, `test`, `novo2` — za pola godine niko neće znati šta je to.

### Konvencija imenovanja

Predlog koji radi u praksi:

```
alias:  <uredjaj>_<kartica>_<port>        →  ESXi01_HBA1_P0
alias:  <storage>_<nod>_<port>            →  STG_NodeA_P0
zone:   <inicijator>__<target>            →  ESXi01_P0__STG_NodeA_P0
cfg:    <namena>_<fabrika>                →  PROD_FABA_CFG
```

Dvostruka donja crta između inicijatora i targeta u imenu zone čini da se granica vidi na prvi pogled. Imena su ograničena na 64 znaka, dozvoljeni su slova, cifre, donja crta — ime mora početi slovom.

---

## 6.7 Peer zone

U velikim okruženjima broj zona po principu SI-ST postaje težak za održavanje. **Peer zone** su odgovor na to.

U peer zoni postoji **principal member** (tipično storage port) i obični članovi (inicijatori). Pravilo je:

- Svaki obični član vidi principal-a
- **Obični članovi se međusobno ne vide**

Time se u jednoj zoni okuplja dvadeset hostova i jedan storage port, a da nijedan host ne vidi drugog. Rezultat je ista izolacija kao kod SI-ST, uz drastično manju bazu.

```
SAN_A_SW01:admin> zone --show STG_NodeA_P0_PEER
 zone:  STG_NodeA_P0_PEER
        Property Member: 00:02:00:00:00:00:00:00
        Principal Member(s):
                50:06:01:60:88:60:2a:11
        Member(s):
                21:00:f4:e9:d4:56:7a:b1
                21:00:f4:e9:d4:56:7a:b2
                21:00:f4:e9:d4:56:8c:11
```

Peer zone zahtevaju odgovarajuću FOS verziju na svim switchevima u fabrici. Za okruženja sa nekoliko hostova, klasične SI-ST zone su i dalje jednostavnije i sasvim dovoljne.

---

## 6.8 Traffic Isolation zone

Poseban tip zone koji ne kontroliše pristup, nego **rutiranje** — usmerava saobraćaj između određenih uređaja na konkretan ISL link. Koristi se kada treba garantovati propusnost za kritičnu aplikaciju ili razdvojiti backup saobraćaj od produkcijskog.

Retko se sreće u manjim okruženjima. Dovoljno je znati da postoji i da TI zona nije zamena za običnu zonu — pristup se i dalje kontroliše klasičnim zonama.

---

## 6.9 Veličina i ograničenja baze

Zoning baza se replicira na sve switcheve u fabrici, pa je njena veličina ograničena.

```
SAN_A_SW01:admin> cfgsize
Configuration Size:
 zone db max size - 1045274 bytes
 available - 1043018 bytes
 committed - 2256 bytes
 transaction - 0 bytes
```

| Stavka | Značenje |
|---|---|
| `zone db max size` | Maksimalna veličina baze na ovom switchu |
| `committed` | Trenutno zauzeće sačuvane konfiguracije |
| `transaction` | Veličina izmena koje još nisu sačuvane |
| `available` | Preostali prostor |

U tipičnom okruženju sa nekoliko desetina uređaja zauzeće je nekoliko kilobajta i o ograničenju ne treba razmišljati. Postaje relevantno u fabrikama sa stotinama portova, i to je jedan od razloga zašto se tamo prelazi na peer zone.

**Važno kod spajanja switcheva:** maksimalna veličina baze je određena najslabijim switchem u fabrici. Ako u fabriku sa velikom bazom dodate stariji uređaj sa manjim ograničenjem, spajanje može propasti.

---

## 6.10 Pregled postojećeg zoninga

```
SAN_A_SW01:admin> zoneshow ESXi01_P0__STG_NodeA_P0
 zone:  ESXi01_P0__STG_NodeA_P0
        ESXi01_HBA1_P0; STG_NodeA_P0
```

```
SAN_A_SW01:admin> alishow ESXi01_HBA1_P0
 alias: ESXi01_HBA1_P0
        21:00:f4:e9:d4:56:7a:b1
```

Provera u kojim je zonama određeni uređaj — komanda koja se koristi stalno pri dijagnostici:

```
SAN_A_SW01:admin> cfgshow | grep -B2 21:00:f4:e9:d4:56:7a:b1
```

Provera da li switch uopšte sprovodi zoning:

```
SAN_A_SW01:admin> switchshow | grep -i zoning
zoning:         ON (PROD_CFG)
```

Ako piše `zoning: OFF`, nijedna konfiguracija nije aktivna — i tada je ponašanje fabrike određeno isključivo podrazumevanom zonom iz odeljka 6.5.

---

## 6.11 Rezime poglavlja

- Zoning određuje ko koga vidi u fabrici; LUN masking na storage nizu određuje ko šta koristi. Oba sloja su neophodna.
- Hijerarhija je alias → zone → cfg; aktivna je najviše jedna konfiguracija.
- Defined je ono što je definisano, Effective je ono što je na snazi — izmena alias-a ne deluje dok se konfiguracija ponovo ne aktivira.
- Koristite WWPN članstvo; port zoning i WWNN zoning izbegavajte.
- `defzone --noaccess` postavite na svakom produkcijskom switchu.
- Pravilo Single Initiator — Single Target; nikada dva inicijatora u istoj zoni.
- Imenska konvencija nije estetika, nego preduslov za dijagnostiku.
- Peer zone rešavaju problem veličine baze u velikim fabrikama.

---

## 6.12 Provera znanja

1. Administrator je zamenio WWPN u alias-u i izvršio `cfgsave`. Host i dalje ne vidi storage. Zašto?
2. Zašto je `defzone --allaccess` opasno podešavanje u produkciji?
3. Server se seli iz porta 4 u port 9 na istom switchu. Da li treba menjati zoning ako je korišćen WWPN zoning? A ako je korišćen port zoning?
4. Host je u zoni sa storage portom, vidi target u `nsshow`, ali nema nijedan LUN. Gde je problem?
5. Zašto se dva inicijatora ne smeju naći u istoj zoni, ako inicijatori ionako ne razmenjuju podatke međusobno?

---

**Sledeće poglavlje:** Zoniranje II — praktično kreiranje alias-a i zona, transakcioni model, `cfgsave` i `cfgenable`, izmene na živoj fabrici i najčešće greške.
