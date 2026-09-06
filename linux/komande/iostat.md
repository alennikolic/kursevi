# Napredne Linux komande — `iostat`

## 1. Uvod

`iostat` prikazuje opterećenje procesora i, što je važnije, detaljnu statistiku svakog
blok uređaja: broj operacija, propusnost, latenciju i dužinu reda čekanja.

Dolazi posle `vmstat`-a. Kada `vmstat` pokaže visok `wa` i procese u `b` koloni, znate
da je problem u ulazu/izlazu; `iostat` kaže **koji uređaj**, **kakvo opterećenje** i
**da li je uređaj zaista zasićen ili je jednostavno spor**.

Instalacija: paket `sysstat` (`apt install sysstat`, `dnf install sysstat`).

```bash
iostat -V
```

```
sysstat version 12.6.1
```

> **Verzija je bitna.** `sysstat` 12 je preimenovao i dodao kolone u odnosu na verziju 11.
> Ovo uputstvo prati verziju 12 i noviju; mapiranje starih imena je u odeljku 3.4.

---

## 2. Sintaksa i opcije

```
iostat [OPCIJE] [INTERVAL [BROJ]]
```

```bash
iostat -xz 1        # prošireno, bez neaktivnih uređaja, svake sekunde
iostat -dx 2 10     # samo diskovi, deset uzoraka na 2 sekunde
```

| Opcija | Značenje |
|---|---|
| `-x` | **prošireni prikaz** — bez njega nema latencije ni reda čekanja |
| `-d` | samo uređaji, bez procesora |
| `-c` | samo procesor |
| `-z` | preskoči uređaje bez aktivnosti |
| `-y` | preskoči prvi izveštaj (prosek od podizanja sistema) |
| `-k` / `-m` | kilobajti / megabajti |
| `-h` | čitljiviji format, kolone u više redova |
| `-t` | dodaj vremensku oznaku |
| `-p [UREĐAJ]` | razloži i po particijama |
| `-N` | prikaži čitljiva imena za device-mapper i LVM |
| `-s` | sažet prikaz sa manje kolona |
| `-o JSON` | izlaz u JSON formatu |

Praktično svaki koristan poziv počinje sa `-x`:

```bash
iostat -xz 1
```

Za praćenje sa vremenskim oznakama i zapisom u fajl:

```bash
iostat -xzt 1 60 | tee /tmp/iostat.log
```

---

## 3. Kolone

### 3.1 Izveštaj o procesoru

```
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           3.12    0.00    1.84   22.41    0.00   72.63
```

| Kolona | Značenje |
|---|---|
| `%user` | korisnički prostor |
| `%nice` | korisnički prostor sa sniženim prioritetom |
| `%system` | kernel |
| **`%iowait`** | procesor je bio **neaktivan jer se čekalo na ulaz/izlaz** |
| **`%steal`** | vreme koje je hipervizor oduzeo virtuelnoj mašini |
| `%idle` | neaktivno bez čekanja na ulaz/izlaz |

`%iowait` je deo neaktivnog vremena, ne opterećenje. Na sistemu koji ionako nema šta da radi
visok `%iowait` nije problem. Postaje značajan tek kada aplikacije čekaju.

### 3.2 Osnovni izveštaj o uređajima (bez `-x`)

```
Device             tps    kB_read/s    kB_wrtn/s    kB_dscd/s    kB_read    kB_wrtn    kB_dscd
sda              41.20      1204.10       842.30         0.00   14204120    9931204          0
```

| Kolona | Značenje |
|---|---|
| `tps` | operacija u sekundi (IOPS), čitanja i upisi zajedno |
| `kB_read/s`, `kB_wrtn/s` | propusnost čitanja i upisa |
| `kB_dscd/s` | propusnost `discard`/TRIM operacija |
| `kB_read`, `kB_wrtn`, `kB_dscd` | **ukupno od podizanja sistema**, ne po sekundi |

Ovaj prikaz nema latenciju i red čekanja, pa je za dijagnostiku nedovoljan.

### 3.3 Prošireni izveštaj (`-x`)

```bash
iostat -xz 1
```

```
Device      r/s   rkB/s  rrqm/s %rrqm r_await rareq-sz    w/s    wkB/s wrqm/s %wrqm w_await wareq-sz   d/s  dkB/s drqm/s %drqm d_await dareq-sz   f/s f_await aqu-sz %util
sda      182.00 2912.00   14.00  7.14    8.42    16.00  64.00  8192.00  92.00 58.97   12.80   128.00  0.00   0.00   0.00  0.00    0.00     0.00  4.00    2.10   2.34  71.20
```

Red je vrlo širok. Kolone su grupisane po vrsti operacije — `r` čitanje, `w` upis,
`d` discard/TRIM, `f` flush.

| Kolona | Značenje |
|---|---|
| **`r/s`, `w/s`** | završenih čitanja / upisa u sekundi (**IOPS**) |
| **`rkB/s`, `wkB/s`** | propusnost u KiB/s |
| `rrqm/s`, `wrqm/s` | zahteva spojenih u redu u sekundi |
| `%rrqm`, `%wrqm` | **procenat zahteva koje je kernel spojio** sa susednim |
| **`r_await`, `w_await`** | prosečno vreme obrade zahteva u **milisekundama**, uključujući čekanje u redu |
| **`rareq-sz`, `wareq-sz`** | prosečna veličina zahteva u KiB |
| `d/s`, `dkB/s`, `d_await`, `dareq-sz` | isto za discard/TRIM |
| `f/s`, `f_await` | flush operacije i njihova latencija |
| **`aqu-sz`** | prosečna dužina reda čekanja |
| **`%util`** | procenat vremena u kom je uređaj imao bar jedan zahtev u obradi |

**Pet kolona nosi gotovo sve zaključke:** `r/s`, `w/s`, `r_await`, `w_await` i `aqu-sz`.

### 3.4 Mapiranje starih imena (sysstat 11 i starije)

| Staro | Novo |
|---|---|
| `rsec/s`, `wsec/s` | `rkB/s`, `wkB/s` |
| `avgrq-sz` | `rareq-sz` / `wareq-sz` (razdvojeno po vrsti) |
| `avgqu-sz` | `aqu-sz` |
| `await` | `r_await` / `w_await` (razdvojeno) |
| `svctm` | **uklonjeno** — vrednost je bila izračunata i nepouzdana, ne koristite je |

---

## 4. Prvi izveštaj se ignoriše

Kao i kod `vmstat`-a, **prvi izveštaj je prosek od podizanja sistema**. Na serveru koji radi
mesecima on izglađuje sve i redovno navodi na pogrešan zaključak.

```bash
iostat -xzy 1        # -y preskače prvi izveštaj
iostat -xz 1 2 | tail -n +7    # jedan trenutni uzorak za skriptu
```

---

## 5. Kako se čitaju ključne kolone

### 5.1 `%util` — najčešće pogrešno tumačena kolona

`%util` meri **udeo vremena u kom je uređaj imao bar jedan zahtev u obradi**.

- Na klasičnom disku sa jednim redom (`HDD`), `%util` blizu 100% je zaista značilo zasićenje.
- Na SSD i NVMe uređajima, koji obrađuju desetine zahteva paralelno, `%util` 100% znači
  samo da uređaj **nikad nije bio dokon** — može biti na 5% svog stvarnog kapaciteta.

> Na modernim uređajima `%util` **nije** merilo zasićenja. Merilo su `await` i `aqu-sz`.

### 5.2 `await` — latencija u odnosu na očekivanu

`await` obuhvata i čekanje u redu i samu obradu na uređaju. Poredi se sa očekivanom
vrednošću za tip uređaja:

| Tip uređaja | Očekivani `await` |
|---|---|
| NVMe SSD | ispod 1 ms |
| SATA / SAS SSD | 0,3 – 2 ms |
| SAS HDD 15k o/min | 4 – 8 ms |
| SATA HDD 7,2k o/min | 8 – 20 ms |
| mrežno skladište (iSCSI, NFS, EBS) | 1 – 10 ms, zavisi od mreže |

### 5.3 `aqu-sz` razlikuje „zasićen“ od „spor“

Ovo je najvažnija razlika u celoj analizi:

| `await` | `aqu-sz` | Zaključak |
|---|---|---|
| visok | **visok** | uređaj je **zasićen** — stiže više zahteva nego što može da obradi |
| visok | **nizak** | uređaj je **spor sam po sebi** — malo zahteva, ali svaki dugo traje |
| nizak | bilo koji | uređaj nije problem |

Prvi slučaj rešava se smanjenjem opterećenja, bržim uređajem ili boljim rasporedom.
Drugi slučaj upućuje na kvar diska, problem na mreži ka skladištu ili operacije koje
po prirodi imaju veliku latenciju (`fsync`, sinhroni upisi).

Odnos je poznat kao Littleov zakon:

```
aqu-sz  ≈  (r/s + w/s) × await / 1000
```

### 5.4 `rareq-sz` otkriva prirodu opterećenja

| Prosečna veličina zahteva | Priroda opterećenja |
|---|---|
| 4 – 16 KiB | **nasumično** — baze podataka, metapodaci, mnogo malih fajlova |
| 32 – 64 KiB | mešovito |
| 128 KiB i više | **sekvencijalno** — kopiranje, bekap, striming, skeniranje tabela |

Visok `%rrqm`/`%wrqm` (procenat spojenih zahteva) je dodatna potvrda sekvencijalnog pristupa:
kernel je uspeo da spoji susedne zahteve u veće.

Nasumično opterećenje na klasičnom disku daje niske IOPS uz visok `await`; isto opterećenje
na NVMe uređaju prolazi bez problema. Ista propusnost u MB/s znači potpuno različite stvari
zavisno od veličine zahteva.

---

## 6. Obrasci — kako izgleda koji problem

U primerima ispod prikazane su samo bitne kolone; stvarni ispis je širi.

### 6.1 Zdrav sistem

```
Device      r/s   rkB/s r_await rareq-sz    w/s   wkB/s w_await wareq-sz aqu-sz %util
nvme0n1   12.40  198.40    0.08    16.00  84.20 4210.00    0.21    50.00   0.02   1.84
```

Latencije daleko ispod milisekunde, red čekanja praktično prazan. Uređaj se ne oseća.

### 6.2 Klasičan disk u zasićenju

```
Device      r/s   rkB/s r_await rareq-sz    w/s   wkB/s w_await wareq-sz aqu-sz %util
sda      184.00 2944.00   62.41    16.00  22.00  352.00   88.10    16.00  12.84  99.80
```

`await` 62 i 88 ms na disku od kog se očekuje 10 ms, `aqu-sz` 12,8 — dvanaest zahteva
stalno čeka u redu. `rareq-sz` 16 KiB znači nasumičan pristup, najgori scenario za
mehanički disk. `%util` 99,8% je ovde zaista relevantan jer je uređaj rotacioni.

Zaključak: disk ne stiže da obradi opterećenje. Rešenja su SSD, keširanje ili smanjenje
broja nasumičnih operacija (indeksi u bazi, veći bafer).

### 6.3 NVMe sa `%util` 100% koji nije problem

```
Device       r/s    rkB/s r_await rareq-sz     w/s    wkB/s w_await wareq-sz aqu-sz %util
nvme0n1  8420.00 67360.00    0.11     8.00 2104.00 33664.00    0.18    16.00   1.14  99.90
```

`%util` je 100%, ali `await` je 0,11 ms i `aqu-sz` je 1,14. Uređaj radi neprekidno,
ali bez ijednog zaostatka. Ovo je **normalno stanje pod opterećenjem**, ne usko grlo.
Da se gledao samo `%util`, zaključak bi bio pogrešan.

### 6.4 Spor uređaj, a ne zasićen

```
Device      r/s   rkB/s r_await rareq-sz    w/s   wkB/s w_await wareq-sz aqu-sz %util
sdb         2.00   32.00  412.60    16.00   1.00   16.00  680.20    16.00   0.98  84.10
```

Samo tri operacije u sekundi, a latencija je pola sekunde. `aqu-sz` je ispod jedan —
nema reda, svaki pojedinačni zahtev traje neprihvatljivo dugo.

Ovo nije preopterećenje nego kvar ili problem na putu do uređaja: disk pred otkazom,
degradiran RAID, izgubljena putanja na SAN-u, mrežni problem ka iSCSI ili NFS skladištu.

Dalje: `dmesg -T | grep -iE "I/O error|ata[0-9]|medium error"`, `smartctl -a /dev/sdb`,
`journalctl -k -p err`, `multipath -ll`.

### 6.5 Opterećenje sinhronim upisima

```
Device      r/s   rkB/s r_await rareq-sz    w/s   wkB/s w_await wareq-sz  f/s f_await aqu-sz %util
nvme0n1    4.00   64.00    0.09    16.00 412.00 3296.00    1.84     8.00 408.00   1.92   0.84  62.10
```

`wareq-sz` je samo 8 KiB, a `f/s` je gotovo jednak `w/s` — svaki upis prati flush operacija.
To je potpis baze podataka koja radi `fsync()` po transakciji ili aplikacije koja
otvara fajlove sa `O_SYNC`.

Propusnost u MB/s izgleda skromno, ali uređaj radi mnogo posla. Optimizacija ide kroz
grupisanje transakcija (`commit_delay`, `group commit`), a ne kroz brži disk.

### 6.6 Sekvencijalno čitanje

```
Device      r/s    rkB/s rrqm/s %rrqm r_await rareq-sz    w/s  wkB/s w_await aqu-sz %util
sda      412.00 210944.00 892.00 68.40    2.10   512.00   2.00  32.00    0.80   0.86  94.20
```

`rareq-sz` 512 KiB i `%rrqm` 68% pokazuju da kernel uspešno spaja susedne zahteve.
206 MB/s uz `await` od 2 ms je blizu maksimuma za rotacioni disk i **očekivano** stanje
pri bekapu, kopiranju ili punom skeniranju tabele.

`%util` 94% ovde ne znači problem — znači da posao teče punom brzinom.

### 6.7 Dupli redovi kod LVM i device-mapper-a

```
Device      r/s   rkB/s r_await    w/s   wkB/s w_await aqu-sz %util
dm-0      184.00 2944.00   62.80  22.00  352.00   88.40  12.90  99.80
sda       184.00 2944.00   62.41  22.00  352.00   88.10  12.84  99.80
```

Isto opterećenje se pojavljuje dvaput: jednom kao logički volumen (`dm-0`), jednom kao
fizički uređaj (`sda`). To nije duplo opterećenje.

Za čitljiva imena umesto `dm-N`:

```bash
iostat -xzN 1
```

```
Device            r/s   rkB/s r_await    w/s   wkB/s w_await aqu-sz %util
vg0-podaci     184.00 2944.00   62.80  22.00  352.00   88.40  12.90  99.80
sda            184.00 2944.00   62.41  22.00  352.00   88.10  12.84  99.80
```

Ako se latencija na logičkom volumenu bitno razlikuje od one na fizičkom uređaju,
uzrok je u sloju između — šifrovanje (`dm-crypt`), RAID ili tanko obezbeđivanje prostora.

---

## 7. Brza tabela zaključivanja

| Šta se vidi | Zaključak |
|---|---|
| `await` visok, `aqu-sz` visok | uređaj je zasićen |
| `await` visok, `aqu-sz` < 1 | uređaj je spor ili u kvaru |
| `await` nizak, `%util` 100% | normalno na SSD/NVMe — nije problem |
| `rareq-sz` 4–16 KiB, niske IOPS na HDD | nasumično opterećenje na pogrešnom tipu uređaja |
| `rareq-sz` > 128 KiB, visok `%rrqm` | sekvencijalno opterećenje, očekivano pri bekapu |
| `f/s` ≈ `w/s`, mali `wareq-sz` | sinhroni upisi baze; optimizovati grupisanje transakcija |
| `%iowait` visok, svi diskovi mirni | čeka se na mrežni fajl sistem — `nfsiostat`, `mount \| grep nfs` |
| dva uređaja sa istim brojevima | LVM/device-mapper sloj iznad fizičkog diska |
| `%steal` > 5% | problem nije u disku nego u hipervizoru |

---

## 8. Česte greške

1. **Tumačenje prvog izveštaja** — prosek od podizanja sistema; koristite `-y`.
2. **`%util` kao merilo zasićenja SSD-a** — netačno; gledajte `await` i `aqu-sz`.
3. **`iostat` bez `-x`** — nema latencije ni reda čekanja, pa nema ni zaključka.
4. **Korišćenje `svctm`** — uklonjeno u sysstat 12 jer je bilo nepouzdano.
5. **Brojanje `dm-0` i `sda` kao dva opterećenja** — isto opterećenje, dva sloja.
6. **Poređenje `await` bez znanja o tipu uređaja** — 10 ms je uredu za HDD, katastrofa za NVMe.
7. **Gledanje samo MB/s** — 50 MB/s u zahtevima od 4 KiB je ozbiljno opterećenje, u zahtevima od 1 MiB je trivijalno.
8. **Jedan uzorak** — bekap ili rotacija logova daju vrhunac koji nije trajno stanje; posmatrajte bar 30 sekundi.

---

## 9. Podsetnik i sledeći korak

```bash
iostat -xz 1                 # standardno praćenje
iostat -xzy 1                # bez prvog izveštaja
iostat -xzN 1                # čitljiva imena LVM volumena
iostat -dx -p sda 1          # razlaganje po particijama
iostat -xz 1 2 | tail -n +7  # jedan trenutni uzorak
lsblk -d -o NAME,ROTA,MODEL  # da li je uređaj rotacioni (ROTA=1)
```

| Ako `iostat` pokazuje | Sledeći alat |
|---|---|
| zasićen uređaj | `iotop -o`, `pidstat -d 1` — koji proces generiše opterećenje |
| spor uređaj ili kvar | `dmesg -T`, `smartctl -a`, `journalctl -k -p err`, `multipath -ll` |
| sinhrone upise | `strace -c -e trace=fsync,fdatasync -p PID` |
| latenciju bez očiglednog uzroka | `biolatency`, `biosnoop` (bcc/bpftrace) |
| mirne diskove uz visok `%iowait` | `nfsiostat`, `mountstats`, `ss -tinp` ka serveru skladišta |
| visok `%steal` | dijagnostika na strani hipervizora |
