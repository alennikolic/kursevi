# Napredne Linux komande: `ss` (socket statistics)

## 1. Uvod

`ss` je alat za ispitivanje mrežnih soketa na Linux sistemu. Zamena je za zastareli `netstat`
iz paketa `net-tools`. Razlika nije samo kozmetička:

| Osobina | `netstat` | `ss` |
|---|---|---|
| Izvor podataka | parsira `/proc/net/tcp`, `/proc/net/udp` (tekst) | `netlink` socket `NETLINK_SOCK_DIAG` (binarno, kernel API) |
| Brzina na 100k+ soketa | sekunde do minuta | milisekunde |
| Filtriranje | nema, mora `grep` | ugrađen jezik za filtere (state, dport, dst...) |
| TCP interni podaci (cwnd, rtt, retrans) | ne | da (`-i`) |
| Status paketa | deprecated, često nije instaliran | podrazumevano prisutan (`iproute2`) |

Paket: `iproute2` (Debian/Ubuntu: `apt install iproute2`, RHEL/Rocky: `dnf install iproute`).

Provera verzije:

```bash
ss -V
```

```
ss utility, iproute2-6.1.0
```

> Napomena: neki flagovi (`-T`, `--xdp`, `--cgroup`) postoje samo u novijim verzijama.
> Ako flag ne radi, prvo proverite verziju.

---

## 2. Sintaksa

```
ss [OPCIJE] [FILTER]
```

- **OPCIJE** — određuju *koje* sokete gledamo i *koliko detalja* prikazujemo.
- **FILTER** — izraz koji sužava rezultat (stanje soketa, adrese, portovi).

Bez ijednog argumenta:

```bash
ss
```

`ss` bez argumenata prikazuje **sve uspostavljene (established) soketе svih protokola**, uključujući
UNIX domain sokete — što je obično stotine linija i retko korisno. Uvek koristite opcije.

---

## 3. Opcije — kompletan pregled

### 3.1 Izbor protokola

| Opcija | Duga forma | Značenje |
|---|---|---|
| `-t` | `--tcp` | samo TCP soketi |
| `-u` | `--udp` | samo UDP soketi |
| `-w` | `--raw` | RAW soketi (npr. `ping` koristi ICMP raw) |
| `-x` | `--unix` | UNIX domain soketi (`/run/*.sock`) |
| `-S` | `--sctp` | SCTP soketi |
| `-M` | `--mptcp` | MPTCP soketi |
| `-4` | `--ipv4` | ograniči na IPv4 |
| `-6` | `--ipv6` | ograniči na IPv6 |
| `-A` | `--query=LISTA` | eksplicitno navođenje familija: `-A inet,unix` |
| `-f` | `--family=IME` | `inet`, `inet6`, `link`, `unix`, `netlink`, `vsock`, `xdp` |

Protokolske opcije se **kombinuju kao OR**: `-tu` znači „TCP ili UDP“.

### 3.2 Izbor stanja soketa

| Opcija | Duga forma | Značenje |
|---|---|---|
| `-l` | `--listening` | samo soketi koji slušaju (LISTEN / UNCONN) |
| `-a` | `--all` | i oni koji slušaju i oni koji ne slušaju |
| (ništa) | | podrazumevano: samo *ne*-listening, tj. uspostavljene veze |

`-l` i `-a` se međusobno isključuju. `-a` je nadskup oba.

### 3.3 Prikaz i formatiranje

| Opcija | Duga forma | Značenje |
|---|---|---|
| `-n` | `--numeric` | **ne** razrešava portove u imena (`22` umesto `ssh`) i ne radi DNS |
| `-r` | `--resolve` | nasilno razrešava IP adrese u host imena (sporo — DNS upiti!) |
| `-H` | `--no-header` | izostavi red sa zaglavljem (za skripte) |
| `-O` | `--oneline` | svaki soket u tačno jednom redu (bez preloma) |
| `-Z` | `--context` | prikaži SELinux kontekst procesa |
| `-z` | `--contexts` | prikaži SELinux kontekst procesa i soketa |

`-n` je gotovo uvek poželjan: bez njega `ss` čita `/etc/services` i po potrebi pravi
reverse-DNS upite, što na serveru bez DNS-a izaziva višesekundno „visenje“ komande.

### 3.4 Dodatne informacije

| Opcija | Duga forma | Šta dodaje |
|---|---|---|
| `-p` | `--processes` | PID i ime procesa koji drži soket (za tuđe procese traži root) |
| `-e` | `--extended` | UID vlasnika, inode soketa, `sk` kuka, cgroup |
| `-o` | `--options` | stanje TCP tajmera (retransmisija, keepalive, timewait) |
| `-m` | `--memory` | zauzeće memorije po soketu (`skmem:`) |
| `-i` | `--info` | interne TCP metrike: cwnd, rtt, retrans, algoritam zagušenja |
| `-b` | `--bpf` | prikaži priključene BPF filtere (za packet sokete) |
| `-s` | `--summary` | zbirna statistika po protokolima, bez liste soketa |
| `-T` | `--threads` | prikaži i niti procesa, ne samo proces |

### 3.5 Akcije i napredno

| Opcija | Duga forma | Značenje |
|---|---|---|
| `-K` | `--kill` | **prekida** sokete koji odgovaraju filteru |
| `-N IME` | `--net=IME` | izvrši u drugom mrežnom namespace-u (`/var/run/netns/IME`) |
| `-D FAJL` | `--diag=FAJL` | sirovi dump podataka u fajl |
| `-F FAJL` | `--filter=FAJL` | pročitaj filter izraz iz fajla |
| `-E` | `--events` | prati događaje (otvaranje/zatvaranje soketa) u realnom vremenu |
| `--tos` | | prikaži TOS/DSCP polje |
| `--cgroup` | | prikaži cgroup putanju soketa |

---

## 4. Osnovni primeri sa objašnjenjem izlaza

### 4.1 Svi servisi koji slušaju na mreži

Ovo je **najkorišćenija kombinacija** i vredi je zapamtiti napamet:

```bash
sudo ss -tulpn
```

Razlaganje: `-t` TCP, `-u` UDP, `-l` samo listening, `-p` prikaži proces, `-n` bez razrešavanja imena.

**Izlaz:**

```
Netid State  Recv-Q Send-Q  Local Address:Port   Peer Address:Port  Process
udp   UNCONN 0      0       127.0.0.53%lo:53         0.0.0.0:*      users:(("systemd-resolve",pid=712,fd=12))
udp   UNCONN 0      0             0.0.0.0:68         0.0.0.0:*      users:(("dhclient",pid=888,fd=6))
tcp   LISTEN 0      4096    127.0.0.53%lo:53         0.0.0.0:*      users:(("systemd-resolve",pid=712,fd=13))
tcp   LISTEN 0      128           0.0.0.0:22         0.0.0.0:*      users:(("sshd",pid=1044,fd=3))
tcp   LISTEN 0      511           0.0.0.0:80         0.0.0.0:*      users:(("nginx",pid=1330,fd=6),("nginx",pid=1329,fd=6))
tcp   LISTEN 0      70          127.0.0.1:33060      0.0.0.0:*      users:(("mysqld",pid=1502,fd=21))
tcp   LISTEN 0      128              [::]:22            [::]:*      users:(("sshd",pid=1044,fd=4))
```

**Objašnjenje kolona:**

- **Netid** — tip soketa: `tcp`, `udp`, `raw`, `u_str` (UNIX stream), `u_dgr` (UNIX datagram), `nl` (netlink).
- **State** — stanje soketa. Kod UDP-a je uvek `UNCONN` (UDP nema stanja); kod TCP-a `LISTEN`.
- **Recv-Q** — kod `LISTEN` soketa: **trenutni broj završenih veza koje čekaju da ih aplikacija prihvati** (`accept()`). Kod uspostavljenih veza: broj bajtova primljenih u kernelu koje aplikacija još nije pročitala.
- **Send-Q** — kod `LISTEN` soketa: **maksimalna dužina accept reda (backlog)**. Kod uspostavljenih veza: broj bajtova poslatih koje druga strana još nije potvrdila (ACK).
- **Local Address:Port** — lokalna adresa i port.
- **Peer Address:Port** — udaljena strana; kod listening soketa je `0.0.0.0:*` (IPv4) ili `[::]:*` (IPv6), što znači „bilo ko“.
- **Process** — `users:(("ime",pid=PID,fd=BROJ))`.

**Šta konkretno čitamo iz gornjeg izlaza:**

- `0.0.0.0:22` — SSH sluša na **svim** IPv4 interfejsima, dakle dostupan je spolja.
- `127.0.0.1:33060` — MySQL X protokol sluša **samo na loopback-u**; spolja je nedostupan. Ovo je bezbedna konfiguracija.
- `127.0.0.53%lo:53` — `%lo` je *scope identifikator*, znači da je adresa vezana za interfejs `lo`. To je `systemd-resolved` stub resolver.
- `nginx` ima **dva reda u jednom soketu** (`pid=1330` i `pid=1329`) — master i worker proces dele isti listening file descriptor nasleđen preko `fork()`.
- `Send-Q 4096` kod porta 53 i `128` kod porta 22 su različiti backlog-ovi: nginx/`systemd-resolved` traže veći red od `sshd`.

> **Bez `sudo`** kolona `Process` ostaje prazna za sve procese koji nisu vaši. To nije greška, nego
> ograničenje kernela — čitanje tuđih file deskriptora zahteva `CAP_NET_ADMIN`.

### 4.2 Aktivne TCP veze

```bash
ss -tn
```

**Izlaz:**

```
State   Recv-Q  Send-Q      Local Address:Port      Peer Address:Port  Process
ESTAB   0       0          10.0.10.15:22          10.0.10.4:51422
ESTAB   0       36         10.0.10.15:22          10.0.10.4:51610
ESTAB   0       0          10.0.10.15:443        93.184.216.34:44120
CLOSE-WAIT 1    0          10.0.10.15:8080        10.0.10.99:39002
```

**Objašnjenje:**

- Prvi red: mirna SSH sesija, ništa ne čeka ni u jednom redu.
- Drugi red: `Send-Q 36` — 36 bajtova je poslato ka klijentu ali još nema ACK. Normalno za sesiju u kojoj se upravo nešto ispisuje.
- Četvrti red: `CLOSE-WAIT` znači da je **udaljena strana zatvorila vezu, a naša aplikacija nije**. Jedan ili dva takva soketa su normalni; stotine znače bug u aplikaciji koja ne zove `close()` na deskriptoru — klasično curenje file deskriptora.

### 4.3 Sve TCP veze sa procesima

```bash
sudo ss -tanp
```

`-a` dodaje i listening sokete uz uspostavljene, pa dobijate kompletnu TCP sliku sistema u jednoj komandi.

---

## 5. TCP stanja — šta koje znači

`ss` prikazuje stanja iz TCP state mašine. Za administratora su bitna ova:

| Stanje | Značenje | Kada je problem |
|---|---|---|
| `LISTEN` | soket čeka dolazne veze | — |
| `SYN-SENT` | poslali smo SYN, čekamo odgovor | mnogo ovakvih = odredište ne odgovara (firewall, pad servisa) |
| `SYN-RECV` | primili SYN, poslali SYN-ACK, čekamo ACK | mnogo ovakvih = mogući SYN flood |
| `ESTAB` | veza uspostavljena i aktivna | — |
| `FIN-WAIT-1` | poslali smo FIN, čekamo ACK | gomilanje = druga strana nestala |
| `FIN-WAIT-2` | naš FIN je potvrđen, čekamo FIN od druge strane | gomilanje = aplikacija na drugoj strani ne zatvara |
| `TIME-WAIT` | mi smo zatvorili prvi; čekamo 2×MSL (obično 60s) | desetine hiljada = puno kratkih odlaznih veza |
| `CLOSE-WAIT` | **druga strana** zatvorila, naša aplikacija nije | gomilanje = bug u našoj aplikaciji |
| `LAST-ACK` | poslali FIN nakon CLOSE-WAIT, čekamo ACK | prolazno |
| `CLOSING` | oba kraja poslala FIN istovremeno | retko |
| `UNCONN` | soket bez veze (svi UDP, RAW) | — |

**Ključna razlika za dijagnostiku:** mnogo `TIME-WAIT` je *naš* problem samo ako trošimo efemerne portove;
mnogo `CLOSE-WAIT` je **uvek** bug u aplikaciji koja drži deskriptore otvorenim.

---

## 6. Jezik filtera

Filteri se pišu **posle** opcija. Postoje tri grupe.

### 6.1 Filter po stanju

```bash
ss -tn state established
ss -tn state time-wait
ss -tn state connected      # sva stanja osim LISTEN i CLOSED
ss -tn state synchronized   # sva "established-like" osim SYN-SENT
ss -tn state bucket         # samo minisocket stanja: time-wait i syn-recv
ss -tn state big            # sve što nije bucket
ss -tn exclude established  # sve OSIM established
```

Ključne reči: `established`, `syn-sent`, `syn-recv`, `fin-wait-1`, `fin-wait-2`, `time-wait`,
`closed`, `close-wait`, `last-ack`, `listening`, `closing`, `all`, `connected`, `synchronized`,
`bucket`, `big`.

`exclude` (ili `excl`) je negacija.

### 6.2 Filter po adresi i portu

| Ključna reč | Značenje |
|---|---|
| `dst ADRESA` | odredišna (udaljena) adresa |
| `src ADRESA` | izvorna (lokalna) adresa |
| `dport PORT` | odredišni port |
| `sport PORT` | izvorni port |
| `autobound` | soket kome je kernel automatski dodelio port |

Portovi se pišu sa **dvotačkom**: `:22`, `:https`. Podržani operatori:
`=` (ili `eq`), `!=` (`ne`), `<` (`lt`), `>` (`gt`), `<=` (`le`), `>=` (`ge`).

Primeri:

```bash
ss -tn dst 10.0.0.0/8                     # veze ka celoj mreži
ss -tn dport = :443                       # veze ka HTTPS-u
ss -tn '( dport = :80 or dport = :443 )'  # web saobraćaj
ss -tn 'sport > :1024'                    # samo efemerni izvorni portovi
ss -tn dst 93.184.216.34:443              # tačno određen par
ss -tn dst [2001:db8::1]:443              # IPv6 — adresa u uglastim zagradama
```

> **Bitno:** operatore `>` i `<` obavezno stavite pod navodnike, inače ih shell tumači
> kao preusmeravanje izlaza i napraviće vam fajl imena `:1024`.

### 6.3 Logički operatori

`and` / `&&` / `&`, `or` / `||` / `|`, `not` / `!`, zagrade `( )`.

Zagrade i uzvičnik su specijalni znaci u shell-u — ceo izraz stavite u jednostruke navodnike:

```bash
ss -tn state established '( dport = :22 or sport = :22 )'
ss -tn '! ( dst 127.0.0.0/8 or dst ::1 )'
```

### 6.4 Filter za UNIX sokete

```bash
ss -x src /run/systemd/journal/stdout
ss -xa src '/run/php/*'          # džoker znak je dozvoljen
```

---

## 7. Napredni prikazi

### 7.1 Zbirna statistika: `-s`

```bash
ss -s
```

**Izlaz:**

```
Total: 1023
TCP:   14 (estab 5, closed 4, orphaned 0, timewait 3)

Transport Total     IP        IPv6
RAW       1         0         1
UDP       6         4         2
TCP       10        7         3
INET      17        11        6
FRAG      0         0         0
```

**Objašnjenje:**

- `Total: 1023` — ukupan broj **svih** soketa u sistemu, uključujući UNIX domain sokete. Na desktopu je ovo normalno visok broj.
- `TCP: 14` — ukupno TCP soketa, od čega:
  - `estab 5` — aktivne veze;
  - `closed 4` — soketi u zatvaranju;
  - `orphaned 0` — soketi bez pridruženog file deskriptora (aplikacija je otišla, kernel još drži vezu). **Ako ovaj broj raste, curi memorija kernela**; limit je `net.ipv4.tcp_max_orphans`;
  - `timewait 3` — soketi u TIME-WAIT stanju.
- Tabela na dnu razdvaja isti broj po IP verziji. `INET` je zbir RAW+UDP+TCP.

Ovo je prva komanda koju treba pokrenuti kad server „ne prima više veze“ — odmah se vidi
da li ste udarili u limit efemernih portova ili orphan soketa.

### 7.2 TCP interne metrike: `-i`

```bash
ss -tin dst 93.184.216.34
```

**Izlaz:**

```
State  Recv-Q Send-Q   Local Address:Port    Peer Address:Port
ESTAB  0      0          10.0.10.15:44120   93.184.216.34:443
	 cubic wscale:7,7 rto:236 rtt:34.5/2.25 ato:40 mss:1448 pmtu:1500
	 rcvmss:1448 advmss:1448 cwnd:10 bytes_sent:4218 bytes_acked:4219
	 bytes_received:52341 segs_out:38 segs_in:44 data_segs_out:9 data_segs_in:41
	 send 3.36Mbps lastsnd:1204 lastrcv:1180 lastack:1180
	 pacing_rate 6.72Mbps delivery_rate 1.15Mbps delivered:10 busy:112ms
	 rcv_rtt:35 rcv_space:14480 minrtt:33.8
```

**Objašnjenje ključnih polja:**

| Polje | Značenje |
|---|---|
| `cubic` | algoritam kontrole zagušenja (`cubic`, `bbr`, `reno`...) |
| `wscale:7,7` | window scale faktor: pošiljalac,primalac (2⁷ = ×128) |
| `rto:236` | retransmission timeout u ms — koliko se čeka pre retransmisije |
| `rtt:34.5/2.25` | izmereni RTT / srednje odstupanje, u ms |
| `mss:1448` | maximum segment size — najveći TCP payload |
| `pmtu:1500` | otkriveni Path MTU |
| `cwnd:10` | congestion window u segmentima. Nizak cwnd + visok rtt = spor transfer |
| `ssthresh` | prag ispod kog se radi slow start (pojavljuje se posle prvog gubitka) |
| `bytes_sent` / `bytes_acked` | poslato vs. potvrđeno. Velika razlika = zaglavljena veza |
| `retrans:0/3` | trenutne/ukupne retransmisije. **Bilo šta veće od nule ukazuje na gubitak paketa** |
| `lost`, `sacked`, `unacked` | segmenti izgubljeni / selektivno potvrđeni / nepotvrđeni |
| `send 3.36Mbps` | izračunata brzina slanja = cwnd × mss / rtt |
| `pacing_rate` | brzina kojom kernel tempira slanje |
| `delivery_rate` | stvarno izmerena propusnost |
| `lastsnd` / `lastrcv` / `lastack` | ms od poslednjeg slanja / prijema / ACK-a. Veliki brojevi = mrtva veza |
| `busy:112ms` | vreme u kom je veza imala podatke za slanje |
| `rwnd_limited` | vreme izgubljeno jer je **primalac** oglasio mali prozor |
| `sndbuf_limited` | vreme izgubljeno jer je naš send buffer bio pun — povećajte `net.ipv4.tcp_wmem` |
| `minrtt` | najmanji viđeni RTT — dobra procena stvarne latencije bez bafera |

**Praktično čitanje:** ako korisnici prijavljuju sporo preuzimanje, uporedite `minrtt` i `rtt`.
Ako je `rtt` mnogo veći od `minrtt`, imate bufferbloat. Ako `retrans` raste, imate gubitak paketa
na putanji, a ne problem sa aplikacijom.

### 7.3 Memorija soketa: `-m`

```bash
ss -tm state established
```

**Izlaz:**

```
State  Recv-Q Send-Q  Local Address:Port  Peer Address:Port
ESTAB  0      0         10.0.10.15:22       10.0.10.4:51422
	 skmem:(r0,rb131072,t0,tb2626560,f4096,w0,o0,bl0,d0)
```

**Objašnjenje `skmem` polja (sve u bajtovima):**

| Oznaka | Značenje |
|---|---|
| `r` | `rmem_alloc` — memorija trenutno zauzeta primljenim podacima |
| `rb` | `rcv_buf` — veličina prijemnog bafera (limit) |
| `t` | `wmem_alloc` — memorija zauzeta podacima koji su „u letu“ |
| `tb` | `snd_buf` — veličina slanjskog bafera (limit) |
| `f` | `fwd_alloc` — alocirano od strane kernela ali još neiskorišćeno |
| `w` | `wmem_queued` — podaci u redu za slanje, još neposlati |
| `o` | `opt_mem` — memorija za TCP opcije |
| `bl` | `back_log` — podaci u backlog redu soketa |
| `d` | `sock_drop` — **broj odbačenih paketa na ovom soketu** |

`d` veće od nule je crvena zastavica: kernel je odbacivao pakete jer je bafer bio pun.

### 7.4 Tajmeri: `-o`

```bash
ss -tno state established
```

**Izlaz:**

```
State  Recv-Q Send-Q  Local Address:Port   Peer Address:Port  
ESTAB  0      0         10.0.10.15:22        10.0.10.4:51422   timer:(keepalive,118min,0)
ESTAB  0      1424      10.0.10.15:443       10.0.10.7:60112   timer:(on,1.984ms,3)
```

Format je `timer:(IME, PREOSTALO_VREME, BROJ_RETRANSMISIJA)`.

- `keepalive` — čeka se sledeći keepalive probe. `118min` do sledećeg.
- `on` — **retransmission timer je aktivan**, tj. čeka se ACK. Treći element `3` znači da je
  segment već tri puta retransmitovan. Ovo je siguran znak problema na mreži ka tom klijentu.
- `timewait` — soket u TIME-WAIT, broji do isteka.
- `persist` — druga strana je oglasila prozor 0; čekamo da se otvori (spor ili preopterećen klijent).
- `keepalive` sa malim vremenom i brojačem > 0 — veza verovatno umire.

### 7.5 Prošireni podaci: `-e`

```bash
sudo ss -tenp state established
```

**Izlaz:**

```
State  Recv-Q Send-Q  Local Address:Port  Peer Address:Port  Process
ESTAB  0      0        10.0.10.15:22       10.0.10.4:51422    users:(("sshd",pid=2210,fd=4))
	 uid:1000 ino:38104 sk:2f cgroup:/user.slice/user-1000.slice/session-3.scope <->
```

- `uid:1000` — vlasnik soketa (korisnik).
- `ino:38104` — inode broj soketa; koristi se za povezivanje sa `/proc/PID/fd/`.
- `sk:2f` — interna kernel „cookie“ vrednost soketa.
- `cgroup:` — cgroup putanja, korisna za mapiranje soketa na systemd jedinicu ili kontejner.
- `<->` — indikator smera / dvosmerne veze.

---

## 8. Praktični scenariji za administratore

### 8.1 „Port je zauzet, ko ga drži?“

```bash
sudo ss -tulpn '( sport = :8080 )'
```

```
Netid State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
tcp   LISTEN 0      511          0.0.0.0:8080       0.0.0.0:*   users:(("java",pid=3391,fd=48))
```

PID 3391 drži port. Dalje: `ps -p 3391 -o pid,user,cmd`.

### 8.2 Da li je accept red pun? (odbijene veze pod opterećenjem)

```bash
ss -ltn
```

```
State  Recv-Q Send-Q Local Address:Port Peer Address:Port
LISTEN 129    128          0.0.0.0:22        0.0.0.0:*
LISTEN 0      511          0.0.0.0:80        0.0.0.0:*
```

`Recv-Q 129` > `Send-Q 128` na portu 22 znači da je **accept red prepunjen**: veze koje stižu
se odbacuju. Rešenje je povećanje backlog-a u aplikaciji i kernela:

```bash
sysctl -w net.core.somaxconn=1024
```

Ovo je jedna od najkorisnijih dijagnostika koju `netstat` uopšte ne može da uradi.

### 8.3 Koliko veza po klijentskoj IP adresi (detekcija zloupotrebe)

```bash
ss -tn state established | awk 'NR>1 {print $4}' | cut -d: -f1 | sort | uniq -c | sort -rn | head
```

```
    412 10.0.10.99
     37 10.0.10.4
      6 10.0.10.7
```

Jedan klijent sa 412 veza je kandidat za rate-limit ili blokadu.

### 8.4 Brojanje soketa po stanju

```bash
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn
```

```
  18432 TIME-WAIT
    340 ESTAB
    112 CLOSE-WAIT
     14 LISTEN
```

18432 TIME-WAIT znači da server pravi ogroman broj kratkotrajnih **odlaznih** veza —
razmislite o connection pooling-u. 112 CLOSE-WAIT je bug u aplikaciji.

### 8.5 Praćenje jednog klijenta u realnom vremenu

```bash
watch -n1 "ss -tinp dst 10.0.10.99"
```

Osvežava se svake sekunde; posmatrajte kretanje `cwnd`, `retrans` i `rtt`.

### 8.6 Nasilno prekidanje veza

```bash
sudo ss -K dst 10.0.10.99
sudo ss -K state close-wait
sudo ss -K dport = :6379
```

`-K` zahteva root i kernel opciju `CONFIG_INET_DIAG_DESTROY=y` (uključena u svim modernim distribucijama).
Ako opcija nedostaje, dobijate:

```
Failed to send destroy request: Operation not supported
```

> **Oprez:** `-K` bez filtera prekida **sve** sokete koje zahvati podrazumevani izbor.
> Uvek prvo pokrenite istu komandu **bez** `-K` da vidite šta bi bilo pogođeno.

### 8.7 Rad u mrežnim namespace-ovima (kontejneri, VRF)

```bash
ip netns list
sudo ss -N moj-namespace -tulpn
```

Bez `-N` gledate samo host namespace i nećete videti sokete kontejnera.

### 8.8 Koji servis koristi određeni UNIX soket

```bash
sudo ss -xlp src /run/docker.sock
```

```
Netid State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
u_str LISTEN 0      4096   /run/docker.sock 28714        * 0     users:(("dockerd",pid=1102,fd=9))
```

Broj posle putanje (`28714`) je **inode** soketa, ne port.

### 8.9 Skriptovanje

```bash
# Broj established veza ka portu 443 — čist broj, bez zaglavlja
ss -Htn state established dport = :443 | wc -l
```

`-H` uklanja zaglavlje pa `wc -l` daje tačan broj bez oduzimanja jedinice. Idealno za Zabbix/Nagios/Prometheus textfile collector.

---

## 9. Mapa `netstat` → `ss`

| Stara komanda | Nova komanda |
|---|---|
| `netstat -tulpn` | `ss -tulpn` |
| `netstat -an` | `ss -an` |
| `netstat -tp` | `ss -tp` |
| `netstat -s` | `ss -s` (ograničeno) ili `nstat` / `ip -s link` |
| `netstat -r` | `ip route` |
| `netstat -i` | `ip -s link` |
| `netstat -g` | `ip maddr` |

Za detaljnu statistiku protokola (`netstat -s`) prava zamena nije `ss` nego **`nstat -az`**.

---

## 10. Česte greške

1. **Izostavljanje `-n`** — komanda „visi“ nekoliko sekundi zbog reverse-DNS upita.
2. **Zaboravljen `sudo`** — kolona `Process` je prazna, pa se pogrešno zaključi da soket nema vlasnika.
3. **Nenavodnjeni filter** — `ss -tn sport > :1024` kreira fajl `:1024` umesto filtriranja.
4. **Mešanje `-l` i `-a`** — `-a` poništava efekat `-l`; ako želite samo listening, koristite samo `-l`.
5. **Pogrešno čitanje `Recv-Q`/`Send-Q`** — kod LISTEN soketa te kolone znače nešto potpuno drugo (backlog) nego kod ESTAB (bajtovi).
6. **Traženje kontejnerskih soketa sa hosta** bez `-N` ili `nsenter`.
7. **`ss -p` bez `-t`/`-u`** — dobijate i UNIX sokete, izlaz postaje neupotrebljivo dugačak.

---

## 11. Podsetnik (cheat sheet)

```bash
ss -tulpn                        # svi servisi koji slušaju + proces        [zapamtiti]
ss -tanp                         # sve TCP veze + procesi
ss -s                            # zbirna statistika
ss -ltn                          # listening TCP + provera backlog-a
ss -tn state established         # samo aktivne veze
ss -tn state time-wait | wc -l   # koliko TIME-WAIT soketa
ss -tin dst 1.2.3.4              # TCP metrike ka jednom odredištu
ss -tno                          # veze + tajmeri
ss -tm                           # memorija po soketu
ss -tn '( dport = :80 or dport = :443 )'   # web saobraćaj
ss -xlp                          # UNIX soketi koji slušaju + procesi
ss -N imenspejs -tulpn           # unutar mrežnog namespace-a
sudo ss -K dst 10.0.0.5          # prekini veze ka toj adresi
```

---

## 12. Povezani alati

| Alat | Kada ga koristiti umesto `ss` |
|---|---|
| `nstat -az` | detaljni brojači TCP/IP protokola (retransmisije, ECN, listen drops) |
| `lsof -i` | kad vam treba veza soket ↔ file deskriptor ↔ putanja izvršne datoteke |
| `ip -s link` | statistika po interfejsu (greške, odbačeni okviri) |
| `tcpdump` / `tshark` | kad je potrebno videti sam sadržaj paketa |
| `conntrack -L` | tabela praćenja veza kod NAT/firewall-a |
| `bpftrace` / `ss -E` | praćenje događaja otvaranja/zatvaranja soketa u realnom vremenu |

---

**Sledeća komanda u seriji:** `lsof` — mapiranje otvorenih fajlova, soketa i deskriptora na procese.
