# 04 — Kapacitet, file system i cleaning

Najčešći razlog zašto neko otvori ovaj priručnik. Poglavlje objašnjava kako se
čita zauzeće, zašto brisanje podataka ne oslobađa prostor odmah, i šta se radi
kada se uređaj puni.

---

## 4.1 Model kapaciteta — mentalna slika

Data Domain ima dva broja koja se stalno mešaju:

- **Pre-comp** — logička veličina podataka. Koliko bi to zauzelo da nema dedupe-a.
  Ovo je broj koji vidi backup aplikacija.
- **Post-comp** — stvarno zauzeće na disku, nakon dedupe-a i kompresije.
  **Ovo je jedini broj koji određuje da li ćete ostati bez prostora.**

Odnos ta dva broja je faktor redukcije (`Reduction Factor`), tipično 10x–50x
zavisno od tipa podataka i retencije.

Prostor se oslobađa **isključivo cleaning-om (garbage collection)**. Brisanje
fajla sa NFS share-a ne oslobađa ni bajt dok ne prođe sledeći cleaning ciklus.
To je najčešći izvor nesporazuma sa backup timom.

---

## 4.2 Provera zauzeća

```
filesys show space
filesys show space tier active
filesys show space tier cloud            # samo uz Cloud Tier licencu
```

Tabela koju vraća `filesys show space`:

| Red | Značenje | Prag za akciju |
|---|---|---|
| `/data: pre-comp` | Logička veličina svih podataka | Nema praga — samo pokazatelj rasta |
| `/data: post-comp` | **Stvarno zauzeće aktivnog tier-a** | 80% planiranje, 90% akcija, 95% alarm |
| `/ddvar` | Sistemska particija: logovi, support bundle-ovi, upgrade paketi | 80% — čistiti stare bundle-ove |
| `/ddvar/core` | Core dump-ovi | Bilo koji rast je znak da je nešto crash-ovalo |
| `/data: post-comp (cloud tier)` | Zauzeće cloud tier-a | Prema politici |

> **Šta se dešava na 100%:** file system prelazi u režim u kome novi upisi
> padaju. Backup poslovi počinju da otkazuju, a replikacija ka tom uređaju staje.
> Oporavak sa punog file systema je spor, jer i cleaning-u treba nešto slobodnog
> prostora da bi radio. Zato je 90% granica za akciju, a ne 98%.

---

## 4.3 Kompresija i faktor redukcije

```
filesys show compression                               # ukupno, poslednjih 7 dana
filesys show compression last 24 hours
filesys show compression last 7 days
filesys show compression daily
filesys show compression daily-detailed last 30 days
```

Output razdvaja:
- **Global compression** — dedupe, uklanjanje ponovljenih segmenata između backup-a
- **Local compression** — kompresija (`lz`, `gzfast`, `gz`) preostalih segmenata
- **Total reduction** — proizvod ta dva

Po MTree-ju:

```
mtree show compression
mtree show compression /data/col1/<mtree> last 7 days
```

**Kako se ovo koristi:** ako je ukupni faktor iznenada pao sa 20x na 6x,
nešto se promenilo u ulaznim podacima — najčešće je backup aplikacija počela
da šalje već komprimovane ili enkriptovane podatke, ili je uključena enkripcija
na strani klijenta. To je razgovor sa backup timom, ne problem na DD-u.

Podešavanje algoritma lokalne kompresije:

```
filesys option show
filesys option set local-compression-type {lz | gzfast | gz}     # ⚠️
```

`lz` je podrazumevan i najbrži. `gz` štedi prostor ali troši znatno više CPU-a i
usporava backup. Menjati samo uz merenje i uz svest da se primenjuje na nove podatke.

Per-MTree optimizacija za specifične aplikacije:

```
mtree option show /data/col1/<mtree>
mtree option set app-optimized-compression {none | global | oracle1} \
    mtree /data/col1/<mtree>                                     # ⚠️
```

---

## 4.4 Cleaning / Garbage Collection

Cleaning je proces koji fizički oslobađa prostor zauzet segmentima na koje
više nijedan fajl ne pokazuje.

### Provera stanja

```
filesys clean status                 # da li radi i u kojoj je fazi
filesys clean show config            # raspored i throttle
filesys clean watch                  # praćenje uživo
```

`filesys clean status` prikazuje fazu ciklusa. Faze idu redom (pre-merge,
analiza, enumeracija, filtriranje, selekcija, kopiranje, sumiranje) i normalno
je da najduže traju enumeracija i kopiranje. Na velikim, punim sistemima ciklus
može trajati i više od 24 sata.

### Pokretanje i zaustavljanje

```
filesys clean start                  # ⚠️ pokreće ciklus odmah
filesys clean stop                   # ⚠️ prekida tekući ciklus
```

Prekinuti ciklus nije izgubljen posao u potpunosti, ali sledeći ciklus kreće
ispočetka od enumeracije. Ne prekidajte cleaning rutinski.

### Raspored

Podrazumevano cleaning radi jednom nedeljno. Provera i izmena:

```
filesys clean show config
filesys clean set schedule <dan> <vreme>              # ⚠️
filesys clean set schedule never                      # ⚠️ 🛑 ne raditi
filesys clean reset schedule                          # vraća na podrazumevano
```

> **Nikada ne isključujte raspored cleaninga trajno.** Uređaj bez cleaninga se
> puni linearno i završi na 100%. Ako cleaning smeta backup prozoru, pomerite
> ga ili smanjite throttle — ne isključujte ga.

### Throttle

Cleaning troši CPU i I/O i može usporiti backup koji radi u isto vreme.

```
filesys clean show config
filesys clean set throttle <procenat>                 # ⚠️ podrazumevano 50
filesys clean reset throttle
```

- `100` — cleaning ima prioritet, najbrže završava, najviše smeta backup-u
- `50` — podrazumevano, razuman kompromis
- niže vrednosti — cleaning skoro ne smeta, ali može ne stići da završi između ciklusa

---

## 4.5 Zašto prostor nije oslobođen — kontrolna lista

Kada je cleaning završio a `post-comp` se nije smanjio, po redu proveriti:

| # | Uzrok | Provera |
|---|---|---|
| 1 | **Snapshot-ovi** drže stare verzije podataka | `snapshot list mtree /data/col1/<mtree>` |
| 2 | **Retention Lock** — zaključani fajlovi se ne mogu očistiti | `mtree retention-lock status mtree /data/col1/<mtree>` |
| 3 | **Podaci nisu ni obrisani** — backup aplikacija još drži retenciju | Katalog backup aplikacije, ne DD |
| 4 | **Cleaning nije završio ceo ciklus** | `filesys clean status` |
| 5 | **Fajlovi u obrisanom MTree-ju** koji čeka na čišćenje | `mtree list` (status `deleted`) |
| 6 | **Replikacioni kontekst** drži podatke koji još nisu prebačeni | `replication status` |
| 7 | **Cloud Tier** — podaci su migrirani ali recall drži kopiju | `filesys show space tier cloud` |

Prva dva su uzrok u ogromnoj većini slučajeva.

---

## 4.6 Diskovi i storage

```
disk show state                      # mapa svih diskova po policama
disk show hardware                   # model, firmware, kapacitet, serijski
disk show reliability-data           # SMART podaci, greške
disk show performance
enclosure show all                   # police
storage show all
storage show tier active
storage show summary
```

Stanja diska u `disk show state`:

| Oznaka | Značenje |
|---|---|
| `in-use` | Normalno, deo aktivnog RAID-a |
| `spare` | Rezervni disk, spreman |
| `available` | Prepoznat, nije dodeljen |
| `failed` | Otkazao — otvara se case kod Dell-a |
| `absent` | Nema ga u slotu |
| `foreign` | Disk iz drugog sistema |
| `reconstructing` | RAID rekonstrukcija u toku |

Lociranje fizičkog diska u rack-u:

```
disk beacon <enclosure>.<disk>       # pali LED na disku
```

### Proširenje kapaciteta

⚠️ Radi se po Dell proceduri za konkretan model.

```
storage show all                     # šta je već dodeljeno
disk show state                      # koji su novi diskovi vidljivi kao available
storage add tier active dev<n>       # ⚠️ dodavanje u aktivni tier
filesys show space                   # provera nakon dodavanja
```

Nakon dodavanja prostora sistemu treba vremena da ga uključi u file system.
Ne očekujte da se `filesys show space` promeni istog trenutka.

---

## 4.7 Cloud Tier

Samo ako je licenciran. Premešta stare podatke sa aktivnog tier-a u objektni storage.

```
license show                                     # da li Cloud Tier postoji
cloud profile show                               # konfigurisani cloud provajderi
cloud unit list                                  # cloud jedinice
filesys show space tier cloud
data-movement policy show                        # pravila premeštanja
data-movement status
data-movement watch
data-movement start                              # ⚠️
data-movement stop                               # ⚠️
```

Politika se postavlja po MTree-ju:

```
data-movement policy set age-threshold <dani> mtrees /data/col1/<mtree> \
    to-tier cloud cloud-unit <ime>               # ⚠️
```

> **Napomena o Retention Lock-u:** kombinacija Cloud Tier + Retention Lock ima
> dodatna ograničenja u zavisnosti od DDOS verzije i provajdera. Proverite
> Administration Guide za vašu verziju pre nego što ih spojite.

---

## 4.8 /ddvar se popunio

Ne dira podatke, ali obara GUI, logovanje i mogućnost slanja support bundle-ova.
Najčešći krivci su stari support bundle-ovi i preuzeti upgrade paketi.

```
filesys show space                   # potvrda da je /ddvar problem
support bundle list
support bundle delete <ime>          # ⚠️
system upgrade package list
system upgrade package delete <ime>  # ⚠️
log list                             # rotirani logovi
```

---

## 4.9 Prag alerti za kapacitet

Da alert stigne pre nego što bude kasno:

```
alerts notify-list show
alerts show current
```

Podrazumevani pragovi za prostor su na 80/90/95/100%. Ako tim ne dobija mejlove,
problem je u `alerts notify-list`, ne u pragovima.

---

## 4.10 Kratki playbook: "DD je na 92%"

1. `filesys show space` — potvrditi koji tier i koji red je pun
2. `filesys clean status` — da li cleaning radi; ako ne, `filesys clean start` ⚠️
3. `mtree show compression` — koji MTree je najveći potrošač
4. `snapshot list mtree /data/col1/<najveći>` — ima li starih snapshot-ova za brisanje
5. `mtree retention-lock status mtree /data/col1/<najveći>` — je li prostor zaključan
6. `replication status` — da li neki kontekst zaostaje i drži podatke
7. Ako je RL glavni razlog: prostor se **ne može** osloboditi. Ide se na proširenje
   kapaciteta ili na razgovor o retenciji sa vlasnikom podataka.
8. Paralelno: `filesys clean set throttle 100` ⚠️ dok se situacija ne smiri

Detaljniji scenariji → **poglavlje 10**.

---

## Reference

- DDOS Administration Guide — poglavlja o file systemu, kapacitetu i cleaning-u
- DDOS Command Reference Guide — sekcije `filesys`, `storage`, `disk`, `data-movement`
