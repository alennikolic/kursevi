# Napredne Linux komande — `vmstat`

## 1. Uvod

`vmstat` u jednom redu prikazuje stanje procesora, memorije, swap-a, diska i prekida.
To ga čini prvom komandom koju treba pokrenuti na usporenom serveru: za pet sekundi
odgovara na pitanje **gde je usko grlo** — u procesoru, memoriji, disku ili u hipervizoru.

Ostali alati daju dublju sliku jednog resursa. `vmstat` daje plitku sliku svih odjednom,
i time usmerava dalju dijagnostiku.

---

## 2. Sintaksa i opcije

```
vmstat [OPCIJE] [INTERVAL [BROJ]]
```

```bash
vmstat 1          # osvežavaj svake sekunde, beskonačno
vmstat 2 10       # deset uzoraka na svake 2 sekunde
vmstat            # jedan red — prosek od podizanja sistema, retko koristan
```

| Opcija | Značenje |
|---|---|
| `-S M` | prikaži memoriju u MiB (`k`, `K`, `m`, `M`) |
| `-w` | široki ispis, kolone se ne sabijaju |
| `-t` | dodaj vremensku oznaku svakom redu |
| `-a` | umesto `buff`/`cache` prikaži `inact`/`active` memoriju |
| `-n` | ispiši zaglavlje samo jednom (za zapisivanje u fajl) |
| `-s` | tabela ukupnih vrednosti od podizanja sistema |
| `-d` | statistika po disku |
| `-p UREĐAJ` | statistika za jednu particiju |
| `-f` | broj `fork()` poziva od podizanja sistema |

Preporučeni oblik za praćenje i kasniju analizu:

```bash
vmstat -w -t -S M 1 60 | tee /tmp/vmstat.log
```

---

## 3. Kolone

```
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 1  0      0 1204512  98304 6112000   0    0     3     8  412  801  2  1 97  0  0  0
```

### 3.1 `procs` — redovi čekanja

| Kolona | Značenje |
|---|---|
| **`r`** | broj procesa koji **rade ili čekaju na procesor** |
| **`b`** | broj procesa u **neprekidivom snu** (stanje `D`), gotovo uvek zbog ulaza/izlaza |

`r` je najvažnija kolona u celom ispisu. Poredi se sa brojem logičkih jezgara
(`nproc`). `r` trajno veći od broja jezgara znači da procesi čekaju u redu —
sistem je ograničen procesorom.

`b` veće od nule trajno znači da procesi čekaju na disk ili mrežni fajl sistem.
Procesi u stanju `D` se **ne mogu ubiti** signalom dok se ulaz/izlaz ne završi.

### 3.2 `memory` — raspodela memorije (KiB)

| Kolona | Značenje |
|---|---|
| `swpd` | koliko je memorije premešteno u swap |
| `free` | potpuno neiskorišćena memorija |
| `buff` | baferi za metapodatke blok uređaja |
| `cache` | keš sadržaja fajlova |

> **Nizak `free` nije problem.** Kernel namerno koristi svu slobodnu memoriju za `cache`
> i oslobađa je čim zatreba. Pravi pokazatelj je `available` iz `free -m`, koji `vmstat`
> ne prikazuje.

> **`swpd` veće od nule takođe nije problem sam po sebi.** Znači samo da je nešto nekada
> bilo premešteno u swap i tamo ostalo, jer se ne koristi. Problem je aktivnost — kolone `si` i `so`.

### 3.3 `swap` — aktivnost swap-a (KiB/s)

| Kolona | Značenje |
|---|---|
| **`si`** | učitano iz swap-a u memoriju |
| **`so`** | premešteno iz memorije u swap |

Ovo su prave kolone za memorijski pritisak. Povremeni skok je normalan.
**Trajno `si` i `so` različiti od nule istovremeno** znače da sistem premešta iste stranice
tamo-amo (*thrashing*) i da je praktično stao.

### 3.4 `io` — blok uređaji (blokova/s, blok = 1 KiB)

| Kolona | Značenje |
|---|---|
| `bi` | pročitano sa blok uređaja |
| `bo` | upisano na blok uređaj |

Ove kolone same po sebi ne govore da li je disk preopterećen — 100 MB/s je malo za NVMe,
a mnogo za mrežni disk. Tumače se **zajedno sa `wa` i `b`**.

### 3.5 `system` — režijski troškovi

| Kolona | Značenje |
|---|---|
| `in` | prekida (interrupts) u sekundi |
| `cs` | promena konteksta (context switches) u sekundi |

Apsolutne vrednosti zavise od opterećenja; bitan je **odnos prema korisnom radu**.
Visok `cs` uz visok `sy` i nizak `us` znači da sistem troši vreme na prebacivanje
između procesa umesto na posao.

### 3.6 `cpu` — raspodela vremena procesora (%)

| Kolona | Značenje |
|---|---|
| `us` | korisnički prostor — sam rad aplikacija |
| `sy` | kernel — sistemski pozivi, mreža, fajl sistem |
| `id` | neaktivno |
| **`wa`** | neaktivno **jer se čeka na ulaz/izlaz** |
| **`st`** | *steal* — vreme koje je hipervizor oduzeo ovoj virtuelnoj mašini |
| `gu` | vreme utrošeno na goste (na hostu sa virtuelizacijom) |

`wa` je deo `id` vremena: procesor nije imao šta da radi jer su svi spremni procesi
čekali na disk. Visok `wa` na sistemu koji nema šta da radi nije problem.

`st` je vidljivo samo u virtuelnoj mašini. Vrednost trajno iznad nekoliko procenata
znači da je fizički host preprodat i da vaša mašina ne dobija procesor koji plaća.
To se **ne može rešiti iznutra**.

---

## 4. Prvi red se ignoriše

```bash
vmstat 1 5
```

```
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 1204512  98304 6112000   0    0    41   102  388  741  8  2 89  1  0   ← prosek od boot-a
 1  0      0 1204380  98304 6112000   0    0     0     4  402  812  2  1 97  0  0
 0  0      0 1204380  98304 6112000   0    0     0     0  391  788  1  1 98  0  0
```

**Prvi red je prosek od podizanja sistema**, ne trenutno stanje. Na serveru koji radi
mesecima on je beskoristan i redovno navodi na pogrešan zaključak. Tumači se tek od drugog reda.

Za skripte:

```bash
vmstat 1 2 | tail -1
```

---

## 5. Obrasci — kako izgleda koji problem

### 5.1 Zdrav sistem

```
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 1204512  98304 6112000   0    0     3     8  412  801  2  1 97  0  0
```

`r` u granicama broja jezgara, `b` nula, bez swap aktivnosti, `id` visok.
Nizak `free` uz visok `cache` je poželjno stanje, ne problem.

### 5.2 Ograničenje procesorom

```
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
12  0      0 2104512  98304 4112000   0    0     0    12 2104 4102 94  5  1  0  0
```

`r` = 12 na mašini sa 4 jezgra: prosečno tri procesa čekaju na svako jezgro.
`us` 94% znači da vreme troše same aplikacije, `wa` i `st` su nula.

Dalje: `top -o %CPU`, `pidstat -u 1`, `perf top`.

### 5.3 Ograničenje diskom

```
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  6      0  204512  12304 1112000   0    0 84120  2104 8102 12044  3  6 12 79  0
```

`b` = 6 procesa u `D` stanju, `wa` 79%, visok `bi`. Procesor je dokon jer svi čekaju na disk.

Dalje: `iostat -xz 1` (kolone `%util`, `await`, `aqu-sz`), `iotop -o`,
`ps -eo state,pid,comm | awk '$1 ~ /D/'`.

### 5.4 Memorijski pritisak i thrashing

```
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 4 11 8102400  48120   1024  102400 4102 6210 12044  8102 14022 32104 12 28  4 56  0
```

Prepoznatljiv skup znakova: `si` i `so` istovremeno visoki, `free` i `cache` pali su na
minimum, `wa` skočio jer je swap na disku, `sy` visok zbog rada kernela na oslobađanju stranica.

Sistem u ovom stanju je praktično neupotrebljiv i sledeći korak je često OOM killer.

Dalje: `free -m` (kolona `available`), `ps -eo pid,rss,comm --sort=-rss | head`,
`journalctl -k -g "Out of memory"`, `smem -tk`.

### 5.5 Preprodat hipervizor

```
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 8  0      0 1804512  98304 3112000   0    0     2     8  902 1804 41  6 12 0 41
```

`st` 41% znači da skoro polovinu vremena procesor uzima neko drugi na istom fizičkom hostu.
Aplikacija je spora, ali unutar mašine nema šta da se optimizuje.

Dalje: obratite se dobavljaču, promenite tip instance ili je preselite.

### 5.6 Oluja promena konteksta

```
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 9  0      0 3104512  98304 2112000   0    0     0     4 22104 184022 22 61 17 0  0
```

`cs` 184 hiljade u sekundi uz `sy` 61% i `us` samo 22%: sistem troši više vremena na
prebacivanje između niti nego na posao. Uzroci su obično previše niti u odnosu na broj
jezgara ili nadmetanje oko zaključavanja.

Visok `in` uz nizak `cs` upućuje na drugo — bujicu prekida sa mrežne kartice ili kontrolera.

Dalje: `pidstat -w 1` (promene konteksta po procesu), `cat /proc/interrupts`,
`strace -c -p PID` i udeo `futex` poziva.

---

## 6. Brza tabela zaključivanja

| Šta se vidi | Zaključak |
|---|---|
| `r` > broj jezgara, `us` visok | ograničenje procesorom u aplikaciji |
| `r` > broj jezgara, `sy` visok | ograničenje u kernelu — sistemski pozivi, mreža, zaključavanja |
| `b` > 0, `wa` visok | ograničenje diskom ili mrežnim fajl sistemom |
| `wa` visok, `bi`/`bo` niski | čeka se na spor uređaj sa velikom latencijom (NFS, mrežni disk) |
| `si`/`so` trajno > 0 | nedostatak memorije, thrashing |
| `swpd` > 0, `si`/`so` = 0 | bezopasno — stare neaktivne stranice u swap-u |
| `free` nizak, `cache` visok | normalno stanje |
| `st` > 5% | preprodat hipervizor |
| `cs` vrlo visok, `sy` visok | previše niti ili nadmetanje oko zaključavanja |
| `in` vrlo visok, `cs` normalan | oluja prekida — mrežna kartica, drajver |
| sve nisko, a sistem spor | uzrok nije u ovim resursima: mreža, DNS, spoljni servis, zaključavanja u aplikaciji |

---

## 7. Česte greške

1. **Tumačenje prvog reda** — to je prosek od podizanja sistema.
2. **Panika zbog niskog `free`** — kernel namerno koristi memoriju za keš.
3. **Panika zbog `swpd` > 0** — bitna je aktivnost (`si`/`so`), ne zauzeće.
4. **Zanemarivanje `st`** — u virtuelnoj mašini je često pravi uzrok sporosti.
5. **`wa` shvaćen kao opterećenje procesora** — to je neaktivno vreme provedeno u čekanju.
6. **`vmstat` bez intervala** — jedan red prosečnih vrednosti ne govori ništa o trenutnom stanju.
7. **Prekratko posmatranje** — jedan uzorak ne razlikuje trenutni skok od trajnog stanja; posmatrajte bar 30 sekundi.

---

## 8. Podsetnik i sledeći korak

```bash
vmstat 1                     # standardno praćenje
vmstat -w -t -S M 1          # široko, sa vremenom, u MiB
vmstat 1 2 | tail -1         # jedan trenutni uzorak za skriptu
vmstat -s                    # ukupne vrednosti od podizanja sistema
vmstat -d                    # statistika po disku
nproc                        # broj jezgara — referenca za kolonu r
```

| Ako `vmstat` pokazuje | Sledeći alat |
|---|---|
| ograničenje procesorom | `top`, `pidstat -u 1`, `perf top` |
| ograničenje diskom | `iostat -xz 1`, `iotop -o`, `biolatency` |
| memorijski pritisak | `free -m`, `smem`, `ps --sort=-rss`, `journalctl -k -g "Out of memory"` |
| steal vreme | dijagnostika na strani hipervizora |
| promene konteksta / prekidi | `pidstat -w 1`, `/proc/interrupts`, `strace -c` |
| ništa neuobičajeno | `ss -tinp`, `tcpdump`, profilisanje same aplikacije |
