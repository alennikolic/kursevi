# Napredne Linux komande — `strace` (system call tracer)

## 1. Uvod

`strace` presreće i ispisuje **sistemske pozive** (system calls) koje proces upućuje kernelu,
kao i signale koje prima. Pošto svaka interakcija programa sa spoljnim svetom — otvaranje fajla,
čitanje mreže, alokacija memorije, pokretanje procesa — mora da prođe kroz kernel, `strace`
pokazuje šta program **stvarno** radi, bez obzira na to šta piše u dokumentaciji ili logovima.

Tipična pitanja na koja `strace` odgovara kad ništa drugo ne pomaže:

- Aplikacija javlja „Permission denied“, a ne kaže za koji fajl.
- Program se pokreće ali odmah izlazi bez ijedne poruke u logu.
- Servis se zaglavio i ne troši CPU — na čemu tačno čeka?
- Koji konfiguracioni fajl program zapravo čita, i kojim redosledom traži?
- Zašto je operacija spora kad mreža i disk izgledaju u redu?

Instalacija: `apt install strace`, `dnf install strace`.

```bash
strace -V
```

```
strace -- version 6.1
Copyright (c) 1991-2022 The strace developers <https://strace.io>.
```

> **Upozorenje o performansama:** `strace` koristi `ptrace()`, koji zaustavlja proces na svakom
> sistemskom pozivu. Praćeni proces radi **10 do 100 puta sporije**. Nikada ne ostavljajte
> `strace` da radi na produkcijskom servisu duže nego što je potrebno, i uvek ga ograničite
> filterom (`-e trace=...`). Za dugotrajno praćenje koristite `bpftrace` ili `perf trace`.

---

## 2. Preduslovi: ptrace dozvole

Na Ubuntu i mnogim drugim distribucijama Yama LSM podrazumevano zabranjuje priključivanje
na procese koji nisu direktna deca vaše ljuske:

```bash
cat /proc/sys/kernel/yama/ptrace_scope
```

```
1
```

Vrednosti:

| Vrednost | Značenje |
|---|---|
| `0` | bilo koji proces istog korisnika može da se priključi |
| `1` | samo roditelj može da prati dete (podrazumevano na Ubuntu) |
| `2` | samo procesi sa `CAP_SYS_PTRACE` (praktično samo root) |
| `3` | priključivanje potpuno zabranjeno, ne može se vratiti bez restarta |

Simptom nedostatka dozvola:

```
strace: attach: ptrace(PTRACE_SEIZE, 1330): Operation not permitted
```

Rešenja: pokrenite `strace` kao root (`sudo`), ili privremeno:

```bash
sudo sysctl -w kernel.yama.ptrace_scope=0
```

U kontejnerima je potrebno `--cap-add=SYS_PTRACE` (Docker) ili
`securityContext.capabilities.add: ["SYS_PTRACE"]` (Kubernetes).

---

## 3. Sintaksa

```
strace [OPCIJE] KOMANDA [ARGUMENTI]     # pokreni i prati novi proces
strace [OPCIJE] -p PID                  # priključi se na postojeći proces
strace [OPCIJE] -p PID1 -p PID2         # više procesa odjednom
```

Prekid praćenja: `Ctrl+C`. Kod priključivanja (`-p`) proces nastavlja normalno da radi.

---

## 4. Čitanje izlaza

```bash
strace ls /tmp
```

Skraćen izlaz:

```
execve("/usr/bin/ls", ["ls", "/tmp"], 0x7ffd4c8e2f10 /* 28 vars */) = 0
brk(NULL)                               = 0x55f8a2c4a000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=91847, ...}) = 0
mmap(NULL, 91847, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f2a1c3e0000
close(3)                                = 0
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/tmp", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
getdents64(3, 0x55f8a2c4b0a0 /* 5 entries */, 32768) = 160
getdents64(3, 0x55f8a2c4b0a0 /* 0 entries */, 32768) = 0
write(1, "snap-private-tmp\nsystemd-private"..., 48) = 48
close(1)                                = 0
exit_group(0)                           = ?
+++ exited with 0 +++
```

**Anatomija reda:**

```
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
└─ ime      └─ argumenti                                   └─ povratna vrednost
```

**Šta se konkretno vidi u primeru:**

- `execve(...)` je uvek prvi red — trenutak kada se program učitava. Prikazuje putanju, `argv` i broj promenljivih okruženja.
- `access("/etc/ld.so.preload", R_OK) = -1 ENOENT` — **neuspešan poziv**. Povratna vrednost `-1`, ime greške `ENOENT`, i tekst greške u zagradi. Ovaj konkretan neuspeh je normalan; fajl obično ne postoji.
- `= 3` kod `openat` je **broj file deskriptora**. Kasniji `fstat(3, ...)`, `mmap(..., 3, 0)` i `close(3)` odnose se na taj isti deskriptor. Ovako se prati životni ciklus fajla kroz izlaz.
- `{st_mode=S_IFREG|0644, st_size=91847, ...}` — strukture su skraćene. Tri tačke znače da je `strace` izostavio polja; `-v` prikazuje sve.
- `"snap-private-tmp\nsystemd-private"...` — stringovi su skraćeni na **32 znaka**. Tri tačke posle navodnika su znak da je odsečeno; `-s 4096` proširuje.
- `exit_group(0) = ?` — poziv od kog se ne vraća, pa nema povratne vrednosti.
- `+++ exited with 0 +++` — proces je završio sa izlaznim kodom 0.

**Posebni redovi:**

| Oblik | Značenje |
|---|---|
| `+++ exited with N +++` | proces je izašao sa kodom N |
| `+++ killed by SIGKILL +++` | proces je ubijen signalom |
| `--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=4102, ...} ---` | primljen signal, sa detaljima |
| `<unfinished ...>` | poziv je počeo ali nije završen (drugi thread je upisao red između) |
| `<... read resumed> "data", 4096) = 12` | nastavak ranije prekinutog poziva |
| `restart_syscall(<... resuming interrupted read ...>)` | poziv prekinut signalom pa nastavljen |
| `?` kao povratna vrednost | poziv se ne vraća (`exit_group`, `execve` pri zameni slike) |

---

## 5. Opcije — kompletan pregled

### 5.1 Izbor procesa

| Opcija | Duga forma | Značenje |
|---|---|---|
| `-p PID` | `--attach=PID` | priključi se na postojeći proces; može se ponoviti |
| `-f` | `--follow-forks` | prati i procese/niti nastale sa `fork()`, `vfork()`, `clone()` |
| `-ff` | | uz `-o FAJL` piše svaki proces u zaseban fajl `FAJL.PID` |
| `-b execve` | `--detach-on=execve` | odvoji se kad proces pozove `execve` (korisno za shell omotače) |
| `-u KORISNIK` | `--user=KORISNIK` | pokreni komandu kao navedeni korisnik |
| `-E VAR=VRED` | `--env=VAR=VRED` | postavi promenljivu okruženja praćenom procesu |
| `-D` / `-DD` | `--daemonize` | pokreni sam `strace` kao odvojen proces (preživi ubijanje grupe) |

Bez `-f` nećete videti ništa što radi dete procesa — najčešći razlog zašto izlaz izgleda „prazan“
kod servisa koji forkuju (Apache prefork, PostgreSQL, shell skripte).

### 5.2 Filtriranje sistemskih poziva

| Opcija | Značenje |
|---|---|
| `-e trace=open,read,write` | prati samo navedene pozive |
| `-e trace=!write` | prati sve **osim** navedenih |
| `-e trace=/regex` | prati pozive čije ime odgovara regularnom izrazu |
| `-e trace=%file` | svi pozivi koji primaju ime fajla kao argument |
| `-e trace=%desc` | operacije nad file deskriptorima |
| `-e trace=%process` | kreiranje, izvršavanje i završavanje procesa |
| `-e trace=%network` | mrežni pozivi (socket, connect, accept, send, recv...) |
| `-e trace=%signal` | rukovanje signalima |
| `-e trace=%ipc` | System V IPC |
| `-e trace=%memory` | mapiranje memorije (`mmap`, `brk`, `munmap`) |
| `-e trace=%stat` | varijante `stat` poziva |
| `-e trace=%clock` | rad sa sistemskim vremenom |
| `-e trace=%pure` | pozivi bez argumenata koji ne menjaju stanje (`getpid`, `gettid`) |
| `-P PUTANJA` | prati samo pozive koji dodiruju navedenu putanju |
| `-e signal=SIGKILL,SIGTERM` | prikaži samo navedene signale |
| `-e signal=none` | uopšte ne prikazuj signale |
| `-e status=failed` | prikaži samo neuspešne pozive |
| `-e status=successful` | prikaži samo uspešne pozive |

Skraćenice `-z` i `-Z` su ekvivalenti za `-e status=successful` odnosno `-e status=failed`.

Grupe se kombinuju zarezom: `-e trace=%file,%network`.

### 5.3 Detaljnost prikaza

| Opcija | Značenje |
|---|---|
| `-s N` | maksimalna dužina prikazanog stringa (podrazumevano **32**) |
| `-v` | ne skraćuj strukture, nizove i okruženje (isto što i `-e abbrev=none`) |
| `-x` | prikaži ne-ASCII znakove heksadecimalno |
| `-xx` | prikaži **sve** znakove heksadecimalno |
| `-y` | uz svaki file deskriptor prikaži putanju fajla |
| `-yy` | dodatno prikaži protokol i adrese za sokete |
| `-Y` | uz PID prikaži i ime procesa |
| `-e read=3,5` | prikaži pun sadržaj pročitan sa deskriptora 3 i 5 (hex + ASCII) |
| `-e write=1,2` | prikaži pun sadržaj upisan na deskriptore 1 i 2 |
| `-e abbrev=none` | ne skraćuj strukture |
| `-e verbose=none` | obrnuto — skrati sve strukture |
| `-e raw=open` | prikaži argumente navedenih poziva u sirovom, nedekodovanom obliku |
| `-i` | prikaži vrednost instrukcijskog pokazivača pri svakom pozivu |
| `-k` | prikaži stack trace za svaki poziv (traži debug simbole) |
| `-n` | prikaži i broj sistemskog poziva |
| `-q`, `-qq`, `-qqq` | sve tiši izlaz (sakrij poruke o priključivanju, izlasku, statusu) |
| `-a N` | poravnaj kolonu sa povratnim vrednostima na kolonu N (podrazumevano 40) |

### 5.4 Vreme

| Opcija | Značenje |
|---|---|
| `-t` | vreme početka poziva, sekundna preciznost (`14:22:07`) |
| `-tt` | mikrosekundna preciznost (`14:22:07.481239`) |
| `-ttt` | UNIX vreme sa mikrosekundama (`1717245727.481239`) |
| `-r` | **relativno** vreme od prethodnog poziva |
| `-T` | vreme **provedeno u samom pozivu**, ispisano na kraju reda `<0.000123>` |
| `-w` | u `-c` sažetku meri protekло (wall clock) vreme umesto CPU vremena |

`-r` odgovara na „koliko je proces čekao između poziva“, a `-T` na „koliko je trajao sam poziv“.
Za dijagnostiku sporosti gotovo uvek želite oba: `-rT`.

### 5.5 Statistika

| Opcija | Značenje |
|---|---|
| `-c` | **ne prikazuj pojedinačne pozive**, samo zbirnu tabelu na kraju |
| `-C` | prikaži i pojedinačne pozive i zbirnu tabelu |
| `-S KLJUČ` | sortiraj tabelu po: `time`, `calls`, `errors`, `name`, `nothing` |
| `-U KOLONE` | izaberi kolone tabele: `time`, `calls`, `errors`, `time-total`, `min-time`, `max-time`, `avg-time` |

### 5.6 Izlaz u fajl

| Opcija | Značenje |
|---|---|
| `-o FAJL` | upiši trag u fajl umesto na stderr |
| `-A` | dodaj na kraj postojećeg fajla umesto prepisivanja |
| `-o \|KOMANDA` | prosledi izlaz komandi kroz pipe |

Bez `-o`, `strace` piše na **stderr**, što se meša sa izlazom programa.
Za odvajanje: `strace ls 2> trace.txt` ili, čistije, `strace -o trace.txt ls`.

### 5.7 Ubrizgavanje grešaka i kašnjenja

| Opcija | Značenje |
|---|---|
| `-e fault=POZIV` | veštački izazovi neuspeh poziva (podrazumevano `ENOSYS`) |
| `-e inject=POZIV:error=KOD` | vrati konkretnu grešku |
| `-e inject=POZIV:retval=N` | vrati konkretnu uspešnu vrednost |
| `-e inject=POZIV:delay_enter=N` | odloži ulazak u poziv za N mikrosekundi |
| `-e inject=POZIV:delay_exit=N` | odloži izlazak iz poziva |
| `-e inject=POZIV:signal=SIG` | pošalji signal umesto izvršenja poziva |
| `:when=N` | primeni tek na N-ti poziv |
| `:when=N+` | primeni od N-tog poziva nadalje |
| `:when=N+M` | primeni na N-ti, pa svaki M-ti posle njega |

### 5.8 Ostalo

| Opcija | Značenje |
|---|---|
| `--seccomp-bpf` | ubrzava filtriranje prebacivanjem filtera u kernel |
| `-I N` | ponašanje pri prekidu: `1` bez prekida, `2` blokira signale dok traje poziv, `4` ne prekidaj uopšte |
| `-V` | verzija |
| `-h` | pomoć |

---

## 6. Praktični scenariji

### 6.1 Program ne nalazi konfiguracioni fajl

```bash
strace -f -e trace=%file -o /tmp/trace.txt myapp --start
grep ENOENT /tmp/trace.txt
```

```
openat(AT_FDCWD, "/etc/myapp/myapp.conf", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/local/etc/myapp.conf", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/home/marko/.myapp.conf", O_RDONLY) = -1 ENOENT (No such file or directory)
```

**Objašnjenje:** vidi se **tačan redosled** putanja koje program pretražuje. Ovo je informacija
koju dokumentacija često izostavlja. Sada znate gde da postavite fajl.

Ista tehnika za dijagnostiku grešaka dozvola:

```bash
strace -f -e trace=%file -o /tmp/t.txt myapp
grep EACCES /tmp/t.txt
```

```
openat(AT_FDCWD, "/var/lib/myapp/data.db", O_RDWR) = -1 EACCES (Permission denied)
```

Odmah znate **koji** fajl i **koji** režim pristupa je odbijen — umesto generičke poruke aplikacije.

### 6.2 Zaglavljen proces koji ne troši CPU

```bash
sudo strace -p 3391 -f -T
```

Izlaz stoji, pa se posle nekog vremena pojavi:

```
[pid  3391] futex(0x7f1a2c0e40, FUTEX_WAIT_PRIVATE, 2, NULL <unfinished ...>
```

ili:

```
[pid  3391] read(7, <unfinished ...>
```

**Objašnjenje:**

| Poziv u kom proces visi | Verovatan uzrok |
|---|---|
| `futex(..., FUTEX_WAIT, ...)` | čeka na mutex ili lock — mogući deadlock između niti |
| `read(N, ...)` na soketu | čeka odgovor sa mreže koji ne stiže |
| `read(N, ...)` na fajlu | zaglavljen I/O, često NFS koji ne odgovara |
| `flock` / `fcntl(F_SETLKW)` | čeka lock fajla koji drži drugi proces |
| `poll` / `epoll_wait` / `select` | normalno mirovanje servisa — **nije** problem |
| `wait4` / `waitpid` | čeka da dete završi |
| `connect(...)` | pokušava vezu ka nedostupnom odredištu |

Da vidite **koji fajl** je deskriptor 7, dodajte `-y`:

```bash
sudo strace -p 3391 -f -y
```

```
[pid  3391] read(7</mnt/nfs/data.bin>, <unfinished ...>
```

Ili, ako je soket, `-yy`:

```
[pid  3391] recvfrom(9<TCP:[10.0.10.15:44120->10.0.10.99:5432]>, <unfinished ...>
```

Sada je jasno: proces čeka odgovor od PostgreSQL servera na 10.0.10.99.

### 6.3 Statistika: gde se troši vreme

```bash
sudo strace -c -f -p 1502
```

Posle `Ctrl+C`:

```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 61.24    2.104332         842      2498       114 futex
 18.77    0.645108          31     20812           pread64
  9.31    0.319940          15     21329           io_submit
  4.02    0.138114          64      2158           epoll_wait
  3.88    0.133302        1189       112           fsync
  1.44    0.049507           4     12377           clock_gettime
  1.34    0.046098        2304        20         3 connect
------ ----------- ----------- --------- --------- ----------------
100.00    3.436401                 59306       117 total
```

**Objašnjenje kolona:**

| Kolona | Značenje |
|---|---|
| `% time` | procenat ukupnog vremena provedenog u ovom pozivu |
| `seconds` | ukupno vreme u ovom pozivu |
| `usecs/call` | **prosečno trajanje jednog poziva u mikrosekundama** |
| `calls` | broj poziva |
| `errors` | broj poziva koji su vratili grešku |
| `syscall` | ime sistemskog poziva |

**Kako se ovo čita:**

- `futex` sa 61% vremena i 114 grešaka ukazuje na jaku kontenciju zaključavanja između niti — aplikacija se bori sama sa sobom.
- `fsync` sa `1189 usecs/call` je spor disk ili nedostatak write cache-a; 112 poziva × ~1,2 ms.
- `clock_gettime` sa 12377 poziva je bezopasno (vDSO), ali ukazuje na aplikaciju koja preterano često čita vreme.
- `connect` sa 3 greške vredi ispitati zasebno.

Za sortiranje po broju poziva umesto po vremenu:

```bash
sudo strace -c -S calls -f -p 1502
```

> `-c` je **jedini bezbedan način** da se `strace` kratko pusti na produkciji: ne ispisuje
> pojedinačne redove, pa je nadzor manji. I dalje usporava proces — držite ga 5-10 sekundi.

### 6.4 Merenje latencije pojedinačnih poziva

```bash
sudo strace -f -T -e trace=%file -p 3391
```

```
[pid 3391] openat(AT_FDCWD, "/mnt/nfs/report.csv", O_RDONLY) = 12 <2.014882>
[pid 3391] fstat(12, {st_mode=S_IFREG|0644, st_size=8214, ...}) = 0 <0.000018>
```

`<2.014882>` znači da je `openat` trajao dve sekunde. Uz `-y` bismo videli i putanju,
a sam podatak da je u pitanju NFS mount odmah usmerava dijagnostiku na mrežu, a ne na aplikaciju.

Kombinacija `-r` (razmak između poziva) i `-T` (trajanje poziva):

```bash
sudo strace -rT -p 3391
```

```
     0.000112 epoll_wait(4, [], 128, 100) = 0 <0.100213>
     0.100389 recvfrom(9, "HTTP/1.1 200 OK\r\n"..., 4096, 0, NULL, NULL) = 1420 <0.000031>
     4.812004 sendto(9, "GET /api/v2/data"..., 312, 0, NULL, 0) = 312 <0.000044>
```

Treći red: proces je **4,8 sekundi bio neaktivan** pre nego što je poslao sledeći zahtev.
Sam `sendto` je trajao 44 mikrosekunde. Dakle usporenje nije u kernelu ni u mreži —
nego u logici aplikacije između dva poziva.

### 6.5 Praćenje mrežne komunikacije

```bash
sudo strace -f -e trace=%network -yy -p 1330
```

```
[pid 1330] accept4(6<TCP:[0.0.0.0:80]>, {sa_family=AF_INET, sin_port=htons(60112),
           sin_addr=inet_addr("10.0.10.7")}, [128 => 16], SOCK_NONBLOCK) = 14<TCP:[10.0.10.15:80->10.0.10.7:60112]>
[pid 1330] recvfrom(14<TCP:[10.0.10.15:80->10.0.10.7:60112]>, "GET / HTTP/1.1\r\nHost: web"..., 1024, 0, NULL, NULL) = 78
[pid 1330] sendto(14<TCP:[10.0.10.15:80->10.0.10.7:60112]>, "HTTP/1.1 200 OK\r\nServer:"..., 238, 0, NULL, 0) = 238
[pid 1330] close(14<TCP:[10.0.10.15:80->10.0.10.7:60112]>) = 0
```

`-yy` pretvara gole brojeve deskriptora u čitljive opise veza. Ceo životni ciklus HTTP zahteva
vidi se u četiri reda.

Za pun sadržaj razmene, bez skraćivanja:

```bash
sudo strace -f -e trace=%network -e write=14 -e read=14 -s 4096 -p 1330
```

Za nešifrovani HTTP ovo je alternativa `tcpdump`-u. Za HTTPS je **bolje** od `tcpdump`-a,
jer `strace` vidi podatke pre enkripcije.

### 6.6 Zašto se program tiho gasi

```bash
strace -f -e trace=%process,%signal ./deploy.sh
```

```
execve("./deploy.sh", ["./deploy.sh"], 0x7ffc... /* 28 vars */) = 0
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|SIGCHLD, ...) = 4501
[pid 4501] execve("/usr/bin/rsync", ["rsync", "-az", "/data/", "backup:/data/"], ...) = 0
[pid 4501] +++ killed by SIGKILL +++
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_KILLED, si_pid=4501, si_status=SIGKILL} ---
exit_group(137)                         = ?
+++ exited with 137 +++
```

**Objašnjenje:** dete (`rsync`, PID 4501) je ubijeno signalom `SIGKILL`. Izlazni kod 137
je `128 + 9`, standardna oznaka za SIGKILL. Pošto skripta nije poslala taj signal,
najverovatniji krivac je OOM killer — potvrda u `dmesg -T | grep -i oom`.

### 6.7 Praćenje samo jednog fajla

```bash
sudo strace -f -P /etc/resolv.conf -p 3391
```

`-P` filtrira po putanji i drastično smanjuje šum kad znate koji fajl vas zanima.

### 6.8 Testiranje otpornosti ubrizgavanjem grešaka

```bash
strace -e inject=openat:error=ENOSPC:when=5 ./myapp
```

Peti `openat` će vratiti „nema mesta na uređaju“ iako disk nije pun. Ovako se testira
da li aplikacija korektno rukuje greškama koje je teško izazvati u laboratoriji.

```bash
strace -e inject=connect:error=ECONNREFUSED ./myapp     # simulacija pada baze
strace -e inject=read:delay_exit=500000 ./myapp         # svako čitanje sporije za 0,5 s
strace -e inject=write:error=EIO:when=10+ ./myapp       # od 10. upisa nadalje, I/O greška
```

> Ovo je alat za test okruženje, nikada za produkciju.

### 6.9 Praćenje servisa sa mnogo procesa

```bash
sudo strace -ff -o /tmp/pg -p 1502
```

`-ff` uz `-o` pravi po jedan fajl za svaki proces: `/tmp/pg.1502`, `/tmp/pg.1503`, ...
To je jedini praktičan način da se analizira PostgreSQL, Apache prefork ili bilo šta
sa desetinama radnih procesa.

```bash
ls -S /tmp/pg.* | head -3        # najveći fajlovi = najaktivniji procesi
```

### 6.10 Praćenje od samog starta servisa

Servis koji pada pri pokretanju ne stigne da mu se priključite. Pokrenite ga pod `strace`-om:

```bash
sudo strace -f -o /tmp/svc.txt /usr/sbin/myservice --foreground
```

Ili kroz systemd, privremenom izmenom jedinice:

```bash
sudo systemctl edit myservice
```

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/strace -f -o /tmp/svc.txt /usr/sbin/myservice
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart myservice
```

Ne zaboravite da uklonite override posle dijagnostike (`systemctl revert myservice`).

---

## 7. Najčešće greške u izlazu i šta znače

| Greška | Značenje | Tipičan uzrok |
|---|---|---|
| `ENOENT` | fajl ili direktorijum ne postoji | pogrešna putanja, nedostaje konfiguracija |
| `EACCES` | pristup odbijen | dozvole, SELinux, AppArmor |
| `EPERM` | operacija nije dozvoljena | nedostaje capability, ne samo dozvole fajla |
| `EAGAIN` / `EWOULDBLOCK` | resurs privremeno nedostupan | normalno kod neblokirajućeg I/O |
| `EINTR` | poziv prekinut signalom | normalno, aplikacija treba da ponovi poziv |
| `EMFILE` | previše otvorenih fajlova (limit procesa) | curenje deskriptora, nizak `ulimit -n` |
| `ENFILE` | previše otvorenih fajlova (limit sistema) | `fs.file-max` |
| `ENOSPC` | nema mesta na uređaju | pun disk ili iscrpljeni inode-ovi |
| `ECONNREFUSED` | veza odbijena | servis na odredištu ne radi |
| `ETIMEDOUT` | isteklo vreme | firewall koji odbacuje pakete bez odgovora |
| `EADDRINUSE` | adresa već u upotrebi | port zauzet (proverite sa `ss -tulpn`) |
| `ENOMEM` | nema memorije | iscrpljena memorija ili `vm.max_map_count` |
| `ESRCH` | proces ne postoji | proces je nestao između dve provere |

`EAGAIN` i `EINTR` su u velikoj većini slučajeva **normalni** i ne treba ih tumačiti kao problem.

---

## 8. Česte greške u korišćenju

1. **Zaboravljen `-f`** — servisi koji forkuju daju prazan ili beskoristan trag.
2. **Nema `-s`** — kritični deo stringa je odsečen na 32 znaka.
3. **`strace` bez filtera na produkciji** — proces uspori toliko da počne da ispada iz klastera.
4. **Zaboravljeno preusmeravanje** — izlaz ide na stderr i meša se sa izlazom programa; koristite `-o`.
5. **Očekivanje da `strace` vidi funkcije aplikacije** — vidi **samo** sistemske pozive; za funkcije koristite `ltrace` (biblioteke) ili profajler.
6. **`strace` u kontejneru bez `SYS_PTRACE`** — poruka „Operation not permitted“.
7. **Priključivanje na proces u drugom PID namespace-u** — PID sa hosta i iz kontejnera se razlikuju; koristite `nsenter` ili PID sa hosta.
8. **Tumačenje svakog `ENOENT` kao greške** — programi rutinski proveravaju nepostojeće putanje.
9. **Zaboravljen `strace` u pozadini** — proces trajno radi 50 puta sporije. Uvek proverite: `pgrep -a strace`.

---

## 9. Podsetnik (cheat sheet)

```bash
strace -f -e trace=%file myapp                 # koji fajlovi se traže i otvaraju
strace -f -e trace=%file myapp 2>&1 | grep ENOENT   # šta nedostaje
strace -f -e trace=%network -yy -p PID         # mrežna komunikacija sa čitljivim soketima
strace -c -f -p PID                            # gde se troši vreme (bezbedno za kratak uvid)
strace -c -S calls -f -p PID                   # sortirano po broju poziva
strace -f -T -p PID                            # trajanje svakog poziva
strace -rT -p PID                              # razmaci između poziva + trajanje
strace -y -p PID                               # deskriptori sa putanjama
strace -yy -p PID                              # deskriptori sa adresama soketa
strace -f -s 4096 -o /tmp/t.txt -p PID         # pun trag u fajl, dugi stringovi
strace -ff -o /tmp/pref -p PID                 # zaseban fajl po procesu
strace -P /etc/app.conf -f -p PID              # samo pozivi nad jednim fajlom
strace -e status=failed -f -p PID              # samo neuspešni pozivi
strace -f -e trace=%process,%signal ./skripta  # zašto se proces gasi
strace -e inject=openat:error=ENOSPC ./myapp   # test rukovanja greškama
pgrep -a strace                                # provera da nije ostao da radi
```

---

## 10. Povezani alati

| Alat | Kada ga koristiti umesto `strace` |
|---|---|
| `ltrace` | praćenje poziva **bibliotečkih funkcija**, ne sistemskih |
| `perf trace` | isto što i `strace`, ali sa znatno manjim usporenjem |
| `bpftrace` / `bcc` alati | produkciono praćenje sa zanemarljivim uticajem; jedini izbor za duže sesije |
| `ftrace` / `trace-cmd` | praćenje unutar samog kernela, ne samo granice sistemskih poziva |
| `gdb -p PID` + `thread apply all bt` | kad treba stanje **aplikacijskih** funkcija, ne sistemskih poziva |
| `lsof -p PID` | trenutni snimak otvorenih fajlova, bez usporavanja procesa |
| `cat /proc/PID/stack` | u kom kernel pozivu proces trenutno spava, bez `ptrace` |
| `cat /proc/PID/wchan` | ime kernel funkcije u kojoj proces čeka |
| `tcpdump` | kad treba videti pakete na žici, uključujući ono što aplikacija ne vidi |

Za brzu proveru „gde visi proces“ bez ikakvog usporavanja, `/proc/PID/stack` i `/proc/PID/wchan`
su prvi izbor; `strace` dolazi kad treba videti **niz** poziva, a ne samo trenutno stanje.
