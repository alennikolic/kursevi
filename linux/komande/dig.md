# Napredne Linux komande — `dig`

## 1. Uvod

`dig` šalje DNS upit i prikazuje **ceo odgovor onakav kakav je stigao sa žice** — zaglavlje,
zastavice, sve sekcije i podatke o samom prenosu. Zbog toga je, za razliku od `nslookup`-a
i `host`-a, jedini alat kojim se DNS problem može zaista dijagnostikovati, a ne samo primetiti.

Ključne stvari koje se vide samo u `dig`-u:

- da li je odgovor stigao od autoritativnog servera ili iz keša,
- da li ime **ne postoji** ili postoji ali nema traženi tip zapisa,
- zašto server vraća `SERVFAIL`,
- gde je lanac delegiranja prekinut,
- koliko je odgovora još u kešu (preostali TTL).

> **Važno ograničenje:** `dig` **ne koristi** `/etc/hosts`, `/etc/nsswitch.conf`, ni
> `systemd-resolved` stub. Šalje upit direktno na server iz `/etc/resolv.conf` ili na onaj
> koji navedete. Aplikacija zato može razrešavati ime drugačije nego `dig`.
> Za ono što vidi aplikacija koristite `getent hosts IME` ili `resolvectl query IME`.

Paket: `dnsutils` (Debian/Ubuntu), `bind-utils` (RHEL/Rocky/Fedora).

---

## 2. Sintaksa i opcije

```
dig [@SERVER] [IME] [TIP] [+OPCIJE]
```

```bash
dig example.com                    # A zapis preko podrazumevanog resolvera
dig @8.8.8.8 example.com MX        # MX zapis sa određenog servera
dig -x 93.184.216.34               # obrnuti upit (PTR)
```

| Opcija | Značenje |
|---|---|
| `@SERVER` | pošalji upit ovom serveru umesto onom iz `/etc/resolv.conf` |
| `+short` | samo podaci odgovora, bez ičega drugog |
| `+noall +answer` | prikaži isključivo ANSWER sekciju |
| `+trace` | **prati delegiranje od korena naniže** |
| `+norecurse` (`+nord`) | ne traži rekurziju — pita se samo šta server ima |
| `+dnssec` | zatraži DNSSEC zapise (`RRSIG`) |
| `+cd` | isključi DNSSEC proveru na resolveru |
| `+tcp` | koristi TCP umesto UDP |
| `+multiline` | razlomi duge zapise u više redova, čitljivije |
| `+stats` / `+nostats` | prikaži ili sakrij podnožje sa statistikom |
| `+ttlunits` | prikaži TTL u jedinicama vremena umesto u sekundama |
| `+time=N` | timeout po pokušaju (podrazumevano 5 s) |
| `+tries=N` | broj pokušaja (podrazumevano 3) |
| `+nssearch` | pitaj **sve** autoritativne servere zone za SOA |
| `+subnet=MREŽA` | pošalji EDNS Client Subnet — testiranje geolociranih odgovora |
| `+qr` | prikaži i odlazni upit, ne samo odgovor |
| `-p PORT` | nestandardan port |
| `-4` / `-6` | prinudno IPv4 ili IPv6 |
| `-f FAJL` | upiti iz fajla, jedan po redu |

Tipovi zapisa koji se najčešće traže: `A`, `AAAA`, `CNAME`, `MX`, `NS`, `TXT`, `SOA`, `PTR`,
`SRV`, `CAA`, `DS`, `DNSKEY`.

> `ANY` je danas praktično beskoristan — većina servera na njega odgovara skraćeno (RFC 8482).
> Pitajte tipove pojedinačno.

---

## 3. Anatomija ispisa

```bash
dig @10.0.10.1 example.com A
```

```
; <<>> DiG 9.18.24-1-Debian <<>> @10.0.10.1 example.com A
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 43120
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;example.com.			IN	A

;; ANSWER SECTION:
example.com.		2841	IN	A	93.184.216.34

;; Query time: 4 msec
;; SERVER: 10.0.10.1#53(10.0.10.1) (UDP)
;; WHEN: Sun Sep 06 14:22:07 CEST 2026
;; MSG SIZE  rcvd: 56
```

### 3.1 Zaglavlje

```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 43120
```

| Polje | Značenje |
|---|---|
| `opcode` | vrsta operacije: `QUERY`, `NOTIFY`, `UPDATE` |
| **`status`** | **kod odgovora (RCODE) — prvo što treba pogledati** |
| `id` | ID transakcije, povezuje upit i odgovor |

**Kodovi odgovora:**

| `status` | Značenje | Šta dalje |
|---|---|---|
| `NOERROR` | upit uspešan (ali proverite broj u `ANSWER`) | — |
| **`NXDOMAIN`** | **ime ne postoji** | greška u imenu ili zapis nije kreiran |
| **`SERVFAIL`** | server nije uspeo da odgovori | DNSSEC, nedostupan gornji server, greška u zoni |
| **`REFUSED`** | server odbija da odgovori | niste ovlašćeni za rekurziju ili server nije autoritativan |
| `NOTIMP` | server ne podržava ovu vrstu upita | — |
| `FORMERR` | server nije razumeo upit | često loš EDNS; probajte `+bufsize=512` ili `+notcp` |
| `NOTAUTH` | server nije autoritativan za zonu | pogrešan server za dinamičko ažuriranje |

### 3.2 Zastavice

```
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
```

| Zastavica | Značenje |
|---|---|
| `qr` | ovo je odgovor, a ne upit (uvek prisutno) |
| **`aa`** | **autoritativan odgovor** — stigao od servera koji drži zonu, ne iz keša |
| **`tc`** | **odgovor je skraćen** jer nije stao u UDP paket; klijent ponavlja preko TCP-a |
| `rd` | klijent je zatražio rekurziju |
| **`ra`** | **server nudi rekurziju** — ako nedostaje, server je samo autoritativan |
| **`ad`** | **podaci su DNSSEC provereni** od strane resolvera |
| `cd` | provera DNSSEC-a je bila isključena na zahtev klijenta |

Odsustvo `ra` uz `REFUSED` je najčešći razlog zašto „DNS ne radi“ posle promene servera:
pitate autoritativni server za ime koje nije u njegovoj zoni.

**Brojači** govore koliko zapisa ima u kojoj sekciji. `ANSWER: 0` uz `NOERROR` je poseban
i vrlo važan slučaj — vidi obrazac 5.3.

### 3.3 Sekcije

| Sekcija | Sadržaj |
|---|---|
| `QUESTION` | šta je pitano — potvrda da je upit poslat kako ste mislili |
| **`ANSWER`** | traženi zapisi |
| `AUTHORITY` | autoritativni serveri zone, ili `SOA` kod negativnog odgovora |
| `ADDITIONAL` | dopunski zapisi, najčešće A/AAAA za servere iz AUTHORITY sekcije |
| `OPT PSEUDOSECTION` | EDNS podaci: podržana veličina UDP paketa, kolačići, NSID |

Format svakog reda je uvek isti:

```
example.com.		2841	IN	A	93.184.216.34
└─ ime           └─ TTL  └─ klasa └─ tip └─ podaci
```

Tačka na kraju imena znači da je ime potpuno kvalifikovano (FQDN).

### 3.4 Podnožje

```
;; Query time: 4 msec
;; SERVER: 10.0.10.1#53(10.0.10.1) (UDP)
;; WHEN: Sun Sep 06 14:22:07 CEST 2026
;; MSG SIZE  rcvd: 56
```

| Red | Značenje |
|---|---|
| `Query time` | trajanje upita; vrlo kratko vreme znači odgovor iz keša |
| **`SERVER`** | **ko je stvarno odgovorio**, na kom portu i kojim protokolom |
| `WHEN` | vreme upita |
| `MSG SIZE rcvd` | veličina odgovora u bajtovima; preko 512 može praviti probleme na starim uređajima |

Red `SERVER` je kontrolna tačka: potvrđuje da upit nije otišao na server koji ste mislili
da ste zaobišli. Na sistemima sa `systemd-resolved` ovde često stoji `127.0.0.53`.

---

## 4. TTL otkriva keširanje

Ponovite isti upit nekoliko puta:

```bash
dig @10.0.10.1 example.com +noall +answer
sleep 5
dig @10.0.10.1 example.com +noall +answer
```

```
example.com.		2841	IN	A	93.184.216.34
example.com.		2836	IN	A	93.184.216.34
```

**TTL koji opada znači da odgovor dolazi iz keša.** Broj pokazuje koliko sekundi je još
ostalo do isteka.

**TTL koji je uvek isti pun broj** znači da odgovor dolazi svež od autoritativnog servera
(uz njega ide i zastavica `aa`).

Ovo je najbrži način da se utvrdi zašto promena zapisa „još nije vidljiva“: ako TTL opada
od 3600, čeka se do sat vremena.

---

## 5. Obrasci

### 5.1 Uspešan odgovor iz keša

```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 43120
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; ANSWER SECTION:
example.com.		2841	IN	A	93.184.216.34

;; Query time: 4 msec
```

`NOERROR`, jedan zapis u ANSWER, bez `aa`, TTL nije pun broj, vreme 4 ms — keširan odgovor.

### 5.2 Ime ne postoji (`NXDOMAIN`)

```
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 51882
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; QUESTION SECTION:
;nepostoji.example.com.		IN	A

;; AUTHORITY SECTION:
example.com.	3600 IN	SOA	ns.icann.org. noc.dns.icann.org. 2026090601 7200 3600 1209600 3600
```

`NXDOMAIN` znači da ime **stvarno ne postoji** u zoni. `SOA` zapis u AUTHORITY sekciji
je dokaz koji zona to tvrdi.

Poslednji broj u SOA zapisu (`3600`) je **TTL negativnog keširanja**: koliko dugo će
resolveri pamtiti da ime ne postoji. Ako ste upravo kreirali zapis, moraćete da sačekate
toliko sekundi.

Polja SOA zapisa redom: primarni server, adresa administratora, serijski broj, `refresh`,
`retry`, `expire`, `minimum` (negativni TTL).

### 5.3 Ime postoji, ali nema traženi tip (`NODATA`)

```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12044
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; QUESTION SECTION:
;example.com.			IN	AAAA

;; AUTHORITY SECTION:
example.com.	3600 IN	SOA	ns.icann.org. noc.dns.icann.org. 2026090601 7200 3600 1209600 3600
```

**Ovo je `NOERROR`, ali `ANSWER: 0`.** Ime postoji, samo nema `AAAA` zapis.

Razlika u odnosu na `NXDOMAIN` je suštinska i redovno se meša:

| Odgovor | Značenje |
|---|---|
| `NXDOMAIN` | ime **ne postoji** — proverite da li je zapis uopšte kreiran |
| `NOERROR` + `ANSWER: 0` | ime postoji, **nema tog tipa zapisa** — pitali ste pogrešan tip |

`+short` u oba slučaja daje **prazan izlaz**, pa se razlika bez punog ispisa ne vidi.
To je glavni razlog da se `+short` ne koristi za dijagnostiku.

### 5.4 `SERVFAIL` — najčešće DNSSEC

```
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 8102
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1
```

Test koji odmah razdvaja uzroke:

```bash
dig @10.0.10.1 problem.rs +cd
```

| Rezultat sa `+cd` | Zaključak |
|---|---|
| `NOERROR` sa zapisima | **DNSSEC provera ne uspeva** — istekli potpisi, neusklađen `DS` zapis kod registra |
| i dalje `SERVFAIL` | resolver ne može da dođe do autoritativnog servera |

`+cd` isključuje DNSSEC proveru na resolveru. Ako tek tada odgovor prolazi, problem je u
potpisima, a ne u dostupnosti.

Provera direktno kod autoritativnog servera zaobilazi resolver:

```bash
dig +short NS problem.rs
dig @ns1.problem.rs problem.rs SOA
```

Ako autoritativni server odgovara, a resolver vraća `SERVFAIL`, uzrok je između njih.

### 5.5 `REFUSED`

```
;; ->>HEADER<<- opcode: QUERY, status: REFUSED, id: 4102
;; flags: qr rd; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 0
```

Obratite pažnju: **nema zastavice `ra`**. Server ne nudi rekurziju ovom klijentu.

Dva uobičajena uzroka:

- pitate autoritativni server za zonu koju on ne drži — takav server po pravilu ne radi rekurziju,
- resolver ima listu dozvoljenih klijenata i vaša adresa nije na njoj (`allow-recursion` u BIND-u).

### 5.6 Skraćen odgovor i prelazak na TCP

```
;; Truncated, retrying in TCP mode.
...
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 14, AUTHORITY: 0, ADDITIONAL: 1
;; SERVER: 10.0.10.1#53(10.0.10.1) (TCP)
;; MSG SIZE  rcvd: 1842
```

`dig` je sam ponovio upit preko TCP-a. Odgovor je stigao — ali **aplikacije ne moraju biti
te sreće** ako firewall propušta samo UDP port 53.

Provera da li TCP prolazi:

```bash
dig @10.0.10.1 velika-zona.rs TXT +tcp
```

Ako UDP daje `tc`, a TCP ne odgovara, imate blokiran DNS preko TCP-a. To se manifestuje
kao nasumični prekidi razrešavanja za velike odgovore (DNSSEC, mnogo `TXT` ili `MX` zapisa).

### 5.7 Lanac `CNAME`

```
;; ANSWER SECTION:
www.primer.rs.		300	IN	CNAME	primer.rs.
primer.rs.		300	IN	A	203.0.113.10
```

Resolver je pratio `CNAME` i vratio oba zapisa. Ovo je normalno.

Problematičan slučaj je `CNAME` koji vodi u prazno:

```
;; ANSWER SECTION:
www.primer.rs.		300	IN	CNAME	stari-server.provajder.net.

;; AUTHORITY SECTION:
provajder.net.	900 IN SOA ...
```

Ima `CNAME`, nema `A` zapisa na kraju lanca — cilj ne postoji. Aplikacija dobija grešku
razrešavanja iako `www.primer.rs` „postoji“.

Takođe: `CNAME` **ne sme** stajati na vrhu zone (`primer.rs` bez `www`) niti uz druge
zapise istog imena. Ako `dig primer.rs CNAME` vrati zapis, a `dig primer.rs MX` takođe,
zona je neispravno konfigurisana.

### 5.8 `+trace` — gde je prekinuto delegiranje

```bash
dig +trace www.primer.rs
```

```
.			518400	IN	NS	a.root-servers.net.
;; Received 811 bytes from 10.0.10.1#53(10.0.10.1) in 12 ms

rs.			172800	IN	NS	a.nic.rs.
rs.			172800	IN	NS	b.nic.rs.
;; Received 623 bytes from 198.41.0.4#53(a.root-servers.net) in 24 ms

primer.rs.		3600	IN	NS	ns1.primer.rs.
primer.rs.		3600	IN	NS	ns2.primer.rs.
;; Received 142 bytes from 194.14.24.1#53(a.nic.rs) in 18 ms

www.primer.rs.		300	IN	A	203.0.113.10
;; Received 68 bytes from 203.0.113.53#53(ns1.primer.rs) in 8 ms
```

`+trace` počinje od korenskih servera i spušta se korak po korak, **zaobilazeći keš resolvera**.
Svaki blok pokazuje koji server je odgovorio i koliko je trajalo.

Kako se čita kad nešto ne valja:

| Gde se trag prekida | Uzrok |
|---|---|
| na nivou TLD-a (`rs.`) | domen nije registrovan ili je suspendovan |
| posle TLD-a, bez `NS` zapisa | delegiranje nije postavljeno kod registra |
| `NS` zapisi postoje, ali poslednji korak ne odgovara | autoritativni serveri su nedostupni ili ne drže zonu |
| trag prolazi, a običan upit ne uspeva | problem je u **vašem resolveru**, ne u zoni |

Poslednji red tabele je razlog zbog kog `+trace` prvo treba pokrenuti: odmah razdvaja
„problem u zoni“ od „problem kod nas“.

### 5.9 Neusklađeni autoritativni serveri

```bash
for ns in $(dig +short NS primer.rs); do
    printf '%-24s ' "$ns"
    dig +short @"$ns" www.primer.rs A
done
```

```
ns1.primer.rs.           203.0.113.10
ns2.primer.rs.           203.0.113.10
ns3.rezervni.net.        198.51.100.44
```

Treći server vraća staru adresu — replikacija zone ne radi. Deo korisnika dobija
pogrešan odgovor, nasumično, što je jedan od najtežih simptoma za dijagnostiku.

Brza provera serijskih brojeva svih servera zone:

```bash
dig +nssearch primer.rs
```

```
SOA ns1.primer.rs. admin.primer.rs. 2026090612 7200 3600 1209600 3600 from server ns1.primer.rs in 8 ms.
SOA ns1.primer.rs. admin.primer.rs. 2026090612 7200 3600 1209600 3600 from server ns2.primer.rs in 12 ms.
SOA ns1.primer.rs. admin.primer.rs. 2026083101 7200 3600 1209600 3600 from server ns3.rezervni.net in 41 ms.
```

Serijski broj `2026083101` na trećem serveru zaostaje — potvrda da prenos zone ne uspeva.

### 5.10 Provera pošte i SPF zapisa

```bash
dig +short primer.rs MX
dig +short primer.rs TXT
dig +short _dmarc.primer.rs TXT
dig +short podrazumevani._domainkey.primer.rs TXT
```

```
10 mail.primer.rs.
"v=spf1 mx include:_spf.provajder.com ~all"
"v=DMARC1; p=quarantine; rua=mailto:dmarc@primer.rs"
```

Kod dugih `TXT` zapisa koristite `+multiline` da bi se video pun sadržaj bez prelamanja
na proizvoljnim mestima.

### 5.11 Obrnuti upit

```bash
dig -x 203.0.113.10 +short
```

```
mail.primer.rs.
```

Bez `+short`:

```
;; QUESTION SECTION:
;10.113.0.203.in-addr.arpa.	IN	PTR
```

`dig -x` sam sastavlja ime u `in-addr.arpa` prostoru (obrnut redosled okteta).
Za poštanske servere je odsustvo `PTR` zapisa ili neslaganje sa `A` zapisom čest razlog
odbijanja pošte.

---

## 6. Brza tabela zaključivanja

| Šta se vidi | Zaključak |
|---|---|
| `NOERROR`, `ANSWER` > 0 | sve u redu |
| `NOERROR`, `ANSWER: 0`, `SOA` u AUTHORITY | ime postoji, nema tog tipa zapisa |
| `NXDOMAIN` | ime ne postoji |
| `SERVFAIL`, a `+cd` radi | DNSSEC provera ne uspeva |
| `SERVFAIL` i sa `+cd` | resolver ne dolazi do autoritativnog servera |
| `REFUSED`, nema `ra` | server ne radi rekurziju za vas |
| `tc` zastavica | odgovor prevelik za UDP — proverite da li TCP 53 prolazi |
| `aa` zastavica | odgovor od autoritativnog servera, ne iz keša |
| `ad` zastavica | odgovor je DNSSEC proveren |
| TTL opada pri ponavljanju | odgovor iz keša; sačekajte istek ili očistite keš |
| `+trace` prolazi, običan upit ne | problem je u vašem resolveru |
| različiti odgovori sa različitih `NS` | replikacija zone ne radi |
| `Query time` ispod 5 ms | keširano |
| `Query time` preko 200 ms | daleki ili preopterećen server |

---

## 7. Česte greške

1. **Korišćenje `+short` za dijagnostiku** — ne razlikuje `NXDOMAIN` od `NODATA`, oba daju prazan izlaz.
2. **Pretpostavka da `dig` pokazuje šta vidi aplikacija** — `dig` zaobilazi `/etc/hosts` i `nsswitch`. Koristite `getent hosts`.
3. **Zanemarivanje reda `SERVER`** — upit je otišao na drugi server nego što se mislilo.
4. **Nepitanje autoritativnog servera** — dok se ne pita direktno, ne zna se da li je problem u zoni ili u kešu.
5. **Zaboravljanje negativnog TTL-a** — posle kreiranja zapisa `NXDOMAIN` ostaje keširan do isteka `minimum` polja iz SOA.
6. **Oslanjanje na `ANY`** — savremeni serveri na njega odgovaraju skraćeno.
7. **Tumačenje odsustva `aa`** — odgovor bez te zastavice nije pogrešan, samo dolazi iz keša.
8. **Ignorisanje `tc` zastavice** — tiho blokiran TCP 53 daje nasumične ispade.
9. **Upit bez završne tačke u zoni sa `search` domenima** — `dig server` može postati `server.lokalni.domen`; proverite QUESTION sekciju.
10. **Zaključivanje o propagaciji sa jednog servera** — pitajte sve `NS` zapise zone.

---

## 8. Podsetnik i sledeći korak

```bash
dig example.com                          # osnovni upit
dig @8.8.8.8 example.com                 # preko određenog servera
dig example.com MX +short                # kratko, za skripte
dig example.com +noall +answer           # samo ANSWER sekcija
dig +trace www.primer.rs                 # gde je prekinuto delegiranje
dig @resolver primer.rs +cd              # test DNSSEC problema
dig primer.rs SOA                        # serijski broj i negativni TTL
dig +nssearch primer.rs                  # usklađenost svih NS servera
dig -x 203.0.113.10                      # obrnuti upit
dig primer.rs TXT +multiline             # dugi TXT zapisi
dig @ns1.primer.rs primer.rs AXFR        # prenos zone (ako je dozvoljen)
dig example.com +tcp                     # prinudno preko TCP-a
dig example.com +norecurse               # šta server ima u kešu, bez rekurzije
```

| Ako `dig` pokazuje | Sledeći korak |
|---|---|
| razliku u odnosu na aplikaciju | `getent hosts IME`, `resolvectl query IME`, `cat /etc/nsswitch.conf` |
| `SERVFAIL` uz DNSSEC | `delv IME`, `dnsviz`, provera `DS` zapisa kod registra |
| keširan pogrešan odgovor | `resolvectl flush-caches`, `rndc flush`, `systemctl restart unbound` |
| `tc` uz blokiran TCP | `tcpdump -i any -nn 'port 53'`, pravila na firewall-u |
| spor odgovor | `dig +stats` i poređenje više resolvera, `mtr` do servera |
| problem u zoni | `named-checkzone`, `named-checkconf`, logovi autoritativnog servera |
| nejasnu putanju upita | `tcpdump -i any -nn -s 0 'port 53'` |
