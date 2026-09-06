# Napredne Linux komande — `pidstat`

## 1. Uvod

`pidstat` prikazuje iste vrste statistike kao `vmstat` i `iostat`, ali **po pojedinačnom
procesu ili niti**. Time zatvara lanac dijagnostike:

| Korak | Alat | Pitanje na koje odgovara |
|---|---|---|
| 1 | `vmstat` | koji resurs je usko grlo |
| 2 | `iostat` | koji uređaj, i da li je zasićen ili spor |
| 3 | **`pidstat`** | **koji proces to uzrokuje i zašto** |

Prednost nad `top`-om je u tome što `pidstat` daje podatke koje `top` uopšte nema:
vreme čekanja u redu za procesor, glavne greške stranica po procesu, kašnjenje na
ulazu/izlazu i razdvajanje dobrovoljnih od prinudnih promena konteksta.

Dolazi u paketu `sysstat`, kao i `iostat`.

```bash
pidstat -V
```

```
sysstat version 12.6.1
```

---

## 2. Sintaksa i opcije

```
pidstat [OPCIJE] [INTERVAL [BROJ]]
```

```bash
pidstat 1              # procesor, svake sekunde
pidstat -d 1           # ulaz/izlaz
pidstat -urd 1 5       # procesor, memorija i I/O, pet uzoraka
```

| Opcija | Šta prikazuje |
|---|---|
| `-u` | procesor (podrazumevano) |
| `-r` | memorija i greške stranica |
| `-d` | ulaz/izlaz po procesu (**traži root**) |
| `-w` | promene konteksta |
| `-v` | broj niti i otvorenih deskriptora |
| `-s` | zauzeće steka |

| Opcija | Izbor i format |
|---|---|
| `-p PID` | jedan ili više procesa; `-p ALL` za sve, `-p SELF` za sam `pidstat` |
| `-C ŠABLON` | filtriraj po imenu komande (regularni izraz) |
| `-G ŠABLON` | filtriraj po punom imenu procesa |
| `-t` | prikaži i **pojedinačne niti**, ne samo proces |
| `-l` | prikaži punu komandnu liniju |
| `-U` | prikaži korisničko ime umesto UID-a |
| `-h` | sve u jednom širokom redu, bez ponovljenih zaglavlja |
| `-e PROGRAM` | pokreni program i prati ga |
| `--human` | čitljive jedinice |

Najkorisniji oblici:

```bash
pidstat -u 1 | awk '$8 > 10'          # samo procesi iznad 10% CPU-a
sudo pidstat -d 1                      # ko generiše I/O
pidstat -w 1                           # promene konteksta
pidstat -urd -h 1                      # sve u jednoj tabeli
pidstat -t -p 1502 1                   # razlaganje po nitima jednog procesa
```

---

## 3. Kolone po režimima

Zaglavlje svakog izveštaja počinje vremenom i `UID`/`PID` kolonama, koje se ne ponavljaju
u objašnjenjima ispod.

### 3.1 Procesor (`-u`)

```
14:22:08      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
14:22:08       33      1330   62.38   12.87    0.00   18.81   75.25     0  nginx
```

| Kolona | Značenje |
|---|---|
| `%usr` | vreme u korisničkom prostoru |
| `%system` | vreme u kernelu (sistemski pozivi) |
| `%guest` | vreme utrošeno na virtuelni procesor gosta |
| **`%wait`** | **vreme provedeno u redu čekanja na procesor** |
| `%CPU` | ukupno `%usr` + `%system` |
| `CPU` | broj jezgra na kom je proces poslednji put radio |
| `Command` | ime procesa |

`%CPU` može preći 100% kod višenitnih procesa — sabira se vreme sa svih jezgara.

**`%wait` je kolona koju drugi alati nemaju.** Ona ne meri čekanje na disk (to je `iodelay`),
nego vreme u kom je proces bio **spreman za rad, ali nije dobio procesor**. Visok `%wait`
znači nadmetanje za procesor, čak i kada `%CPU` samog procesa deluje umereno.

### 3.2 Memorija (`-r`)

```
14:22:08      UID       PID  minflt/s  majflt/s     VSZ     RSS   %MEM  Command
14:22:08      110      1502    412.00     84.00 8412344 6104220  74.12  mysqld
```

| Kolona | Značenje |
|---|---|
| `minflt/s` | **male** greške stranica u sekundi — stranica je nađena u memoriji, bez pristupa disku |
| **`majflt/s`** | **glavne** greške stranica — stranicu je trebalo **pročitati sa diska** |
| `VSZ` | virtuelna veličina u KiB — rezervisan adresni prostor, ne stvarno zauzeće |
| `RSS` | stvarno zauzeta fizička memorija u KiB |
| `%MEM` | udeo u ukupnoj fizičkoj memoriji |

`minflt/s` je normalna posledica rada i može biti u hiljadama bez ikakvog problema.

**`majflt/s` trajno različit od nule je znak memorijskog pritiska** — proces pristupa
stranicama koje su izbačene na disk ili tek treba da se učitaju iz izvršne datoteke.
Ovo je pandan kolonama `si`/`so` iz `vmstat`-a, samo po procesu.

`VSZ` je gotovo uvek beskorisno velik i ne treba ga tumačiti kao potrošnju memorije.

### 3.3 Ulaz/izlaz (`-d`)

```
14:22:08      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
14:22:08      110      1502  84120.00   2104.00      0.00     412  mysqld
```

| Kolona | Značenje |
|---|---|
| `kB_rd/s` | pročitano sa blok uređaja |
| `kB_wr/s` | upisano na blok uređaj |
| `kB_ccwr/s` | **otkazani** upisi — podaci obrisani pre nego što su stigli na disk |
| **`iodelay`** | **koliko je proces bio blokiran čekajući na ulaz/izlaz**, u otkucajima sata |

`kB_ccwr/s` različit od nule znači da aplikacija piše pa briše ili skraćuje fajlove —
tipično za privremene fajlove i keš.

`iodelay` je pandan koloni `b` iz `vmstat`-a, po procesu. Proces sa visokim `iodelay`
je onaj koji čeka na disk.

> **`-d` zahteva root** i uključeno praćenje kašnjenja u kernelu.
> Od kernela 5.14 je ono **podrazumevano isključeno**, pa je `iodelay` uvek nula.
> Uključivanje:
> ```bash
> sudo sysctl -w kernel.task_delayacct=1
> ```
> Za trajno uključivanje dodajte `delayacct` u parametre pokretanja kernela.

### 3.4 Promene konteksta (`-w`)

```
14:22:08      UID       PID   cswch/s nvcswch/s  Command
14:22:08      110      1502   8412.00     41.00  mysqld
14:22:08       33      1330     102.00   1840.00  nginx
```

| Kolona | Značenje |
|---|---|
| **`cswch/s`** | **dobrovoljne** promene konteksta — proces se sam odrekao procesora jer čeka na resurs |
| **`nvcswch/s`** | **prinudne** promene konteksta — planer je oduzeo procesor jer je vremenski odsečak istekao |

Razlika je dijagnostički presudna:

- **Visok `cswch/s`** znači da proces stalno čeka — na disk, mrežu, zaključavanje ili
  drugi proces. Sam po sebi nije problem, ali ukazuje gde se troši vreme.
- **Visok `nvcswch/s`** znači da ima **više spremnih procesa nego jezgara**. Sistem je
  preopterećen procesorom, bez obzira na to šta pojedinačni proces radi.

U primeru iznad `mysqld` čeka (verovatno na disk ili zaključavanja), a `nginx` se nadmeće
za procesor sa drugim procesima.

### 3.5 Niti i deskriptori (`-v`)

```
14:22:08      UID       PID threads   fd-nr  Command
14:22:08      110      1502      42    1204  mysqld
```

| Kolona | Značenje |
|---|---|
| `threads` | broj niti procesa |
| `fd-nr` | broj otvorenih file deskriptora |

Ovo je najjednostavniji način da se otkrije curenje resursa: obe vrednosti treba da se
kreću oko stabilne vrednosti. Stalan rast znači da aplikacija ne zatvara deskriptore
ili ne spaja niti — dopuna dijagnostici iz `lsof`-a.

---

## 4. Prvi izveštaj se ignoriše

Kao kod `vmstat`-a i `iostat`-a, **prvi izveštaj obuhvata period od podizanja sistema**,
odnosno od pokretanja procesa. Na dugotrajnom serveru je izglađen i navodi na pogrešan
zaključak. Tumači se od drugog izveštaja.

Kada se navede konačan broj uzoraka, na kraju se ispisuje red `Average:` sa prosekom
celog perioda posmatranja:

```bash
pidstat -u 1 5
```

```
Average:      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
Average:       33      1330   58.12   11.04    0.00   16.22   69.16     -  nginx
```

---

## 5. Kako se čitaju ključne kolone

### 5.1 `%wait` odvaja „spor proces“ od „preopterećen sistem“

| `%CPU` | `%wait` | Zaključak |
|---|---|---|
| visok | nizak | proces stvarno radi i troši procesor |
| umeren | **visok** | proces bi radio više, ali **ne dobija procesor** — sistem je preopterećen |
| nizak | nizak | proces čeka na nešto drugo (disk, mreža, zaključavanje) |

Drugi red je najvredniji: aplikacija je spora, u `top`-u izgleda kao da ne troši mnogo,
a stvarni uzrok je nadmetanje za procesor. Bez `%wait` taj zaključak se ne vidi.

### 5.2 `majflt/s` je jedini pouzdan pokazatelj memorijskog pritiska po procesu

`RSS` i `%MEM` govore koliko proces zauzima, ali ne i da li mu memorija nedostaje.
`majflt/s` govori upravo to — koliko puta u sekundi je morao da čeka na disk zbog stranice
koje nema u memoriji.

### 5.3 `cswch/s` i `nvcswch/s` pokazuju gde se gubi vreme

Zbir obe kolone po svim procesima treba da odgovara koloni `cs` iz `vmstat`-a.
Kada `vmstat` pokaže oluju promena konteksta, `pidstat -w 1` odmah pokazuje krivca
i vrstu problema.

### 5.4 `-t` pronalazi problematičnu nit

Kod višenitnih aplikacija (Java, PostgreSQL, nginx) proces može trošiti 400% procesora,
a uzrok je jedna nit:

```bash
pidstat -t -p 1502 1
```

```
14:22:08      UID      TGID       TID    %usr %system  %wait    %CPU   CPU  Command
14:22:08      110      1502         -   38.61   12.87   4.95   51.48     -  mysqld
14:22:08      110         -      1503    0.99    0.00   0.00    0.99     1  |__mysqld
14:22:08      110         -      1518   36.63   11.88   4.95   48.51     2  |__ib_pg_flush
14:22:08      110         -      1521    0.99    0.99   0.00    1.98     0  |__mysqld
```

`TGID` je ID procesa, `TID` ID niti. Red bez `TGID` je pojedinačna nit.
Ovde je jasno da gotovo sav posao radi nit `ib_pg_flush`.

Dobijeni `TID` se dalje koristi u `strace -p TID` ili se traži u ispisu
`jstack` odnosno `gdb thread apply all bt`.

---

## 6. Obrasci

### 6.1 Jedan proces troši procesor

```
14:22:08      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
14:22:08     1001      4102   98.02    1.98    0.00    0.00   100.00    2  konverter
14:22:08       33      1330    2.97    0.99    0.00    0.00     3.96    0  nginx
```

`%usr` 98%, `%wait` nula — proces dobija procesor kad god traži i radi u korisničkom kodu.
Ovo je pitanje optimizacije same aplikacije, ne sistema.

Dalje: `perf top -p 4102`, profajler aplikacije.

### 6.2 Nadmetanje za procesor

```
14:22:08      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
14:22:08       33      1330   24.75    5.94    0.00   48.51   30.69     1  nginx
14:22:08       33      1331   22.77    6.93    0.00   51.49   29.70     3  nginx
14:22:08      110      1502   18.81    8.91    0.00   44.55   27.72     0  mysqld
```

Nijedan proces ne troši mnogo, ali svi imaju `%wait` oko 50% — polovinu vremena čekaju u redu.
Sistem ima više spremnih procesa nego jezgara. Potvrda je kolona `r` u `vmstat`-u
i visok `nvcswch/s` u `pidstat -w`.

Rešenje je manje istovremenih radnika, više jezgara ili raspoređivanje posla u vremenu.

### 6.3 Memorijski pritisak

```
14:22:08      UID       PID  minflt/s  majflt/s     VSZ     RSS   %MEM  Command
14:22:08      110      1502   4102.00    842.00 9412344 3104220  38.12  mysqld
14:22:08     1001      4102   1204.00    412.00 2412344  804220   9.87  konverter
```

`majflt/s` u stotinama znači da procesi neprekidno čitaju sa diska stranice koje bi
trebalo da su u memoriji. `RSS` je pri tome pao — kernel im oduzima memoriju.

Ovo se u `vmstat`-u vidi kao `si`/`so` aktivnost, a u `iostat`-u kao nasumično čitanje.

Dalje: `free -m` (kolona `available`), `ps -eo pid,rss,comm --sort=-rss | head`,
`journalctl -k -g "Out of memory"`.

### 6.4 Proces blokiran na disku

```
14:22:08      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
14:22:08      110      1502    412.00  84120.00      0.00    1204  mysqld
14:22:08        0      2210      0.00      4.00      0.00       0  sshd
```

`mysqld` piše 84 MB/s i ima `iodelay` 1204 — pretežno čeka na disk.
Poklapa se sa procesom u `D` stanju iz `vmstat` kolone `b`.

Dalje: `iostat -xz 1` za stanje samog uređaja, pa `strace -c -e trace=fsync,fdatasync -p 1502`
ako se sumnja na sinhrone upise.

### 6.5 Oluja promena konteksta

```
14:22:08      UID       PID   cswch/s nvcswch/s  Command
14:22:08     1001      4102  84120.00   1204.00  aplikacija
14:22:08     1001      4103  82044.00   1188.00  aplikacija
```

Preko 80 hiljada **dobrovoljnih** promena konteksta po procesu: niti se neprekidno
blokiraju i bude. Tipičan uzrok je nadmetanje oko zaključavanja ili prekomerna razmena
poruka između niti.

Potvrda se dobija u `strace -c -p 4102` — ako `futex` zauzima najveći deo vremena,
radi se o zaključavanjima.

Obrnut slučaj, visok `nvcswch/s` uz nizak `cswch/s`, znači preopterećenje procesorom
i vodi na obrazac 6.2.

### 6.6 Curenje deskriptora ili niti

```bash
pidstat -v -p 3391 60
```

```
14:00:08      UID       PID threads   fd-nr  Command
15:00:08      UID       PID threads   fd-nr  Command
15:00:08     1001      3391      42    4102  java
16:00:08     1001      3391      42    8214  java
17:00:08     1001      3391      42   12408  java
```

Broj niti je stabilan, ali broj deskriptora ravnomerno raste. Aplikacija ne zatvara
soketе ili fajlove i uskoro će dobiti `EMFILE: Too many open files`.

Dalje: `lsof -p 3391 | awk '{print $5}' | sort | uniq -c | sort -rn` za tip resursa
koji curi, i `cat /proc/3391/limits | grep "open files"` za granicu.

---

## 7. Brza tabela zaključivanja

| Šta se vidi | Zaključak |
|---|---|
| `%usr` visok, `%wait` nizak | proces stvarno radi — optimizovati aplikaciju |
| `%system` visok | vreme se troši u kernelu — sistemski pozivi, mreža, zaključavanja |
| `%wait` visok kod više procesa | preopterećenje procesorom, nedostatak jezgara |
| `majflt/s` > 0 trajno | memorijski pritisak, stranice se čitaju sa diska |
| `minflt/s` visok, `majflt/s` nula | normalno, nije problem |
| `iodelay` visok | proces čeka na disk — dalje u `iostat` |
| `kB_ccwr/s` > 0 | aplikacija piše pa briše — privremeni fajlovi |
| `cswch/s` vrlo visok | čekanje na zaključavanja ili ulaz/izlaz |
| `nvcswch/s` vrlo visok | previše spremnih procesa u odnosu na broj jezgara |
| `fd-nr` ili `threads` stalno rastu | curenje resursa u aplikaciji |
| `VSZ` ogroman, `RSS` mali | normalno, `VSZ` nije potrošnja memorije |

---

## 8. Česte greške

1. **Tumačenje prvog izveštaja** — obuhvata period od podizanja sistema.
2. **`-d` bez root prava** — izlaz je prazan ili nepotpun.
3. **`iodelay` uvek nula** — od kernela 5.14 praćenje kašnjenja je isključeno; uključite `kernel.task_delayacct=1`.
4. **`VSZ` shvaćen kao potrošnja memorije** — to je rezervisan adresni prostor.
5. **Zanemarivanje `%wait`** — bez nje se preopterećenje procesorom ne vidi kod procesa sa umerenim `%CPU`.
6. **Mešanje `cswch` i `nvcswch`** — dobrovoljne i prinudne promene konteksta ukazuju na suprotne probleme.
7. **`%CPU` preko 100% shvaćen kao greška** — višenitni procesi sabiraju vreme sa svih jezgara.
8. **Praćenje procesa bez `-t`** — kod višenitnih aplikacija se ne vidi koja nit je uzrok.
9. **Jedan uzorak** — posmatrajte bar 10 do 30 sekundi.

---

## 9. Podsetnik i sledeći korak

```bash
pidstat 1                          # procesor po procesu
pidstat -u 1 | awk '$8 > 10'       # samo iznad 10% CPU-a
pidstat -r 1                       # memorija i greške stranica
sudo pidstat -d 1                  # ulaz/izlaz po procesu
pidstat -w 1                       # promene konteksta
pidstat -v -p PID 60               # praćenje curenja niti i deskriptora
pidstat -t -p PID 1                # razlaganje po nitima
pidstat -urd -h 1                  # sve u jednoj tabeli
pidstat -C '^nginx$' -u 1          # filtriranje po imenu
pidstat -e /usr/local/bin/skripta  # prati program koji se pokreće
nproc                              # broj jezgara — referenca za %wait
```

| Ako `pidstat` pokazuje | Sledeći alat |
|---|---|
| visok `%usr` | `perf top -p PID`, profajler aplikacije |
| visok `%system` | `strace -c -p PID` — koji sistemski pozivi |
| visok `%wait` / `nvcswch` | `vmstat 1` (kolona `r`), smanjiti broj radnika |
| visok `majflt/s` | `free -m`, `smem`, `ps --sort=-rss` |
| visok `iodelay` | `iostat -xz 1`, `iotop -o`, `biosnoop` |
| visok `cswch/s` | `strace -c -p PID` i udeo `futex` poziva |
| rast `fd-nr` | `lsof -p PID`, `/proc/PID/limits` |
| problematičnu nit (`TID`) | `strace -p TID`, `jstack`, `gdb thread apply all bt` |
