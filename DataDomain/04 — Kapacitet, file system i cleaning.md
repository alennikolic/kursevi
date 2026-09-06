# 04 — Kapacitet, file system i cleaning

Najčešći razlog zašto neko otvori ovaj priručnik. Kako se čita zauzeće, zašto
brisanje podataka ne oslobađa prostor odmah, i šta se radi kada se uređaj puni.

---

## 4.1 Model kapaciteta — mentalna slika

Dva broja koja se stalno mešaju:

- **Pre-comp** — logička veličina podataka. Koliko bi zauzelo bez dedupe-a.
  To je broj koji vidi backup aplikacija.
- **Post-comp** — stvarno zauzeće na disku, nakon dedupe-a i kompresije.
  **Jedini broj koji određuje da li ćete ostati bez prostora.**

Odnos je faktor redukcije, tipično 10x–50x zavisno od tipa podataka i retencije.

Prostor se oslobađa **isključivo cleaning-om (garbage collection)**. Brisanje
fajla sa NFS share-a ne oslobađa ni bajt dok ne prođe sledeći cleaning ciklus.
To je najčešći izvor nesporazuma sa backup timom.

---

## 4.2 Provera zauzeća

```
filesys show space
filesys show space tier active
filesys show space tier cloud            # samo uz Cloud Tier licencu
filesys show space tier total
filesys show file-info <filename>
```

| Red | Značenje |
|---|---|
| `/data: pre-comp` | Logička veličina svih podataka |
| `/data: post-comp` | **Stvarno zauzeće aktivnog tier-a** |
| `/ddvar` | Logovi, support bundle-ovi, upgrade paketi |
| `/ddvar/core` | Core dump-ovi |

### Pragovi su podesivi — proverite ih

```
filesys option show
```

```
filesys option set warning-space-usage <50-90>      # ⚠️
filesys option set critical-space-usage <75-98>     # ⚠️
filesys option set staging-reserve <0-90>           # ⚠️ rezerva za disk staging
```

Dell preporučuje da `critical` bude viši od `warning`. Ako niko nije dirao
podrazumevane vrednosti, saznaćete ih iz `filesys option show` — ne pretpostavljajte.

> **Šta se dešava na 100%:** file system prelazi u režim u kome novi upisi
> padaju. Backup poslovi otkazuju, replikacija ka tom uređaju staje. Oporavak
> je spor, jer i cleaning-u treba nešto slobodnog prostora da bi radio.
> Zato je granica za akciju oko 90%, a ne 98%.

---

## 4.3 Kompresija i faktor redukcije

```
filesys show compression [<filename>] [recursive] [last <n> {hours | days}]
filesys show compression tier active summary
filesys show compression daily last 30 days
filesys show compression daily-detailed last 30 days
filesys show compression <apsolutna-putanja> [recursive]
```

Po MTree-ju:

```
mtree show compression
mtree show compression /data/col1/<mtree> last 7 days
mtree show compression /data/col1/<mtree> tier active
```

Output razdvaja **global compression** (dedupe) i **local compression**
(kompresija preostalih segmenata), pa daje ukupnu redukciju.

**Kako se ovo koristi:** ako je ukupni faktor pao sa 20x na 6x, promenili su se
ulazni podaci — najčešće je backup aplikacija počela da šalje već komprimovane
ili enkriptovane podatke. To je razgovor sa backup timom, ne problem na DD-u.

### Podešavanje kompresije

```
filesys option show
filesys option set local-compression-type {none | lz | gzfast | gz}    # ⚠️
```

`lz` je brz, `gz` štedi prostor ali troši znatno više CPU-a. Primenjuje se
na nove podatke, ne retroaktivno.

> **Upozorenje iz dokumentacije:** promena tipa kompresije sa `gzfast` na `lz`
> na DD6400, DD6410, DD6900, DD9400, DD9410, DD9900, DD9910 i DD9910F
> **isključuje QAT karticu** i može povećati opterećenje CPU-a.

Per-MTree optimizacija za aplikacije:

```
mtree option show [mtree <putanja>]
mtree option set app-optimized-compression {none | global | oracle1} mtree <putanja>   # ⚠️
mtree option reset app-optimized-compression mtree <putanja>                           # ⚠️
```

Na nivou celog file systema opcija ima uži skup vrednosti:

```
filesys option set app-optimized-compression {none | oracle1}          # ⚠️
```

---

## 4.4 Cleaning / Garbage Collection

### Provera stanja

```
filesys clean status
filesys clean show config
filesys clean show schedule
filesys clean show throttle
filesys clean watch
```

Ciklus prolazi kroz faze (pre-merge, analiza, enumeracija, filtriranje,
selekcija, kopiranje, sumiranje). Na velikim, punim sistemima može trajati
i više od 24 sata.

### Pokretanje i zaustavljanje

```
filesys clean start                  # ⚠️
filesys clean stop                   # ⚠️
```

> Na sistemu sa konfigurisanom bezbednosnom politikom, **ručno pokretanje
> cleaning-a traži autorizaciju security officera.**

### Raspored

```
filesys clean show schedule
filesys clean set schedule {never | daily <vreme> | <dan(i)> <vreme> | biweekly <dan> <vreme> | monthly <dan(i)> <vreme>}   # ⚠️
filesys clean reset {schedule | throttle | all}      # ⚠️
```

Dell preporučuje jednom nedeljno.

> **Nikada ne postavljajte `never` trajno.** Uređaj bez cleaninga se puni
> linearno i završi na 100%. Ako cleaning smeta backup prozoru, pomerite ga
> ili smanjite throttle.

### Automatski raspored po popunjenosti

Umesto fiksnog rasporeda, cleaning se može vezati za procenat zauzeća:

```
filesys clean auto schedule show
filesys clean auto schedule {days <dani> estimate-percent-used <procenat> [interval-days <dani>]}   # ⚠️
filesys clean auto schedule reset       # ⚠️
```

I obrnuto — preskakanje ciklusa kada nema potrebe:

```
filesys clean skip schedule show
filesys clean skip schedule estimate-percent-used <procenat> days <dani>   # ⚠️
filesys clean skip schedule reset       # ⚠️
```

> DDOS sam preskače zakazani cleaning ako detektuje anomaliju u količini
> obrisanih podataka. To je zaštita — ako primetite preskočene cikluse,
> proverite alerte pre nego što ručno pokrenete cleaning.

### Throttle

```
filesys clean show throttle
filesys clean set throttle <procenat>    # ⚠️
```

- `100` — cleaning ima prioritet, najbrže završava, najviše smeta backup-u
- niže vrednosti — cleaning skoro ne smeta, ali može ne stići da završi

---

## 4.5 Zašto prostor nije oslobođen — kontrolna lista

| # | Uzrok | Provera |
|---|---|---|
| 1 | **Snapshot-ovi** drže stare verzije | `snapshot list mtree /data/col1/<mtree>` |
| 2 | **Retention Lock** — zaključani fajlovi | `mtree retention-lock status mtree /data/col1/<mtree>` |
| 3 | **Podaci nisu ni obrisani** — aplikacija drži retenciju | Katalog backup aplikacije |
| 4 | **Cleaning nije završio ciklus** | `filesys clean status` |
| 5 | **Obrisan MTree čeka čišćenje** | `mtree list` (status `D`) |
| 6 | **Replikacioni kontekst** drži neprebačene podatke | `replication status` |
| 7 | **Cloud Tier** recall drži kopiju | `filesys show space tier cloud` |
| 8 | **CR PIT kopije** zaključane u vault-u | `crcli policy list-copy` (**poglavlje 12**) |

Prva dva su uzrok u ogromnoj većini slučajeva.

Izveštaj gde tačno žive podaci:

```
filesys report generate file-location path {<putanja> | all} [output-file <ime>]
```

---

## 4.6 Diskovi i storage

```
disk show state
disk show hardware
disk show reliability-data
disk show performance
disk port show {stats | summary}
disk multipath status
enclosure show all
storage show all
storage show tier active
storage show summary
```

Stanja diska:

| Oznaka | Značenje |
|---|---|
| `in-use` | Normalno, deo aktivnog RAID-a |
| `spare` | Rezervni disk, spreman |
| `available` | Prepoznat, nije dodeljen |
| `failed` | Otkazao — otvara se case |
| `absent` | Nema ga u slotu |
| `foreign` | Disk iz drugog sistema |
| `reconstructing` | RAID rekonstrukcija u toku |

Lociranje:

```
disk beacon {<enclosure-id>.<disk-id> | <serialno>}
enclosure beacon <enclosure>
```

### Proširenje kapaciteta

⚠️ Po Dell proceduri za konkretan model.

```
storage show all                     # šta je već dodeljeno
disk show state                      # koji su novi diskovi available
storage add [tier {active | cache | cloud}] {enclosures <lista> | disks <lista>}   # ⚠️
storage add force disk <disk>        # ⚠️ samo po uputstvu podrške
filesys show space                   # provera nakon dodavanja
```

> Sintaksa je `enclosures <lista>` ili `disks <lista>` — ne oznaka tipa `dev1`.

Uklanjanje i migracija:

```
storage remove <...>                 # 🛑
storage migration precheck source-enclosures <lista> destination-enclosures <lista>
storage migration start source-enclosures <lista> destination-enclosures <lista>   # ⚠️
storage migration status
storage migration suspend / resume / finalize    # ⚠️
storage migration option set throttle {low | medium | high}     # ⚠️
storage migration show history
```

Nakon dodavanja prostora sistemu treba vremena da ga uključi u file system.

---

## 4.7 Cloud Tier

```
elicense show                                    # da li Cloud Tier postoji
cloud profile show
cloud unit list
filesys show space tier cloud
data-movement policy show
data-movement status
data-movement watch
```

⚠️

```
data-movement policy set age-threshold <dani> mtrees /data/col1/<mtree> \
    to-tier cloud cloud-unit <ime>
data-movement start
data-movement stop
```

> Kombinacija Cloud Tier + Retention Lock ima dodatna ograničenja zavisno od
> DDOS verzije i provajdera. Proverite Administration Guide pre nego što ih spojite.

---

## 4.8 /ddvar se popunio

Ne dira podatke, ali obara GUI, logovanje i pravljenje support bundle-ova.

```
filesys show space                       # potvrda da je /ddvar problem
support bundle list
system upgrade package list
log list
```

⚠️

```
support bundle delete {<lista> | all}
system upgrade package delete <ime>
```

Ako `/ddvar/core` nije prazan — nešto je crash-ovalo. Otvoriti case sa bundle-om.

---

## 4.9 Kratki playbook: "DD je na 92%"

1. `filesys show space` — koji tier i koji red je pun
2. `filesys option show` — koji su stvarni pragovi na ovom uređaju
3. `filesys clean status` — da li cleaning radi; ako ne, `filesys clean start` ⚠️
4. `mtree show compression` — koji MTree je najveći potrošač
5. `snapshot list mtree /data/col1/<najveći>` — stari snapshot-ovi
6. `mtree retention-lock status mtree /data/col1/<najveći>` — je li prostor zaključan
7. `replication status` — da li kontekst zaostaje i drži podatke
8. Ako je RL glavni razlog: prostor se **ne može** osloboditi. Proširenje
   kapaciteta ili revizija retencije sa vlasnikom podataka.
9. Paralelno: `filesys clean set throttle 100` ⚠️ dok se situacija ne smiri

Detaljniji scenariji → **poglavlje 10**.

---

## Reference

- DD OS 8.6 Command Reference Guide — poglavlja `filesys`, `storage`, `disk`, `enclosure`, `cloud`, `data-movement`, `compression`
- DD OS 8.6 Administration Guide — file system, kapacitet, cleaning, Cloud Tier
