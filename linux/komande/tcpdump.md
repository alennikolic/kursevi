# Napredne Linux komande — `tcpdump`

## 1. Uvod

`tcpdump` presreće pakete na mrežnom interfejsu i prikazuje ih u čitljivom obliku ili
zapisuje u `pcap` fajl. To je poslednja instanca u mrežnoj dijagnostici: kada `ss` pokazuje
da veza ne postoji, aplikacija javlja timeout, a logovi ćute, `tcpdump` pokazuje šta se
stvarno dešava na žici.

Pitanja na koja odgovara samo `tcpdump`:

- Da li paketi uopšte stižu do servera, ili ih neko odbacuje ranije?
- Ko prekida vezu — klijent, server ili uređaj između njih?
- Zašto DNS upit ne dobija odgovor?
- Da li aplikacija zaista koristi TLS ili šalje podatke u čistom tekstu?
- Ko na mreži tvrdi da ima našu IP adresu?

Ključna prednost nad drugim alatima je **mesto presretanja**. `tcpdump` koristi `AF_PACKET`
soket, koji na dolaznom smeru vidi pakete **pre** nego što ih obradi netfilter (iptables/nftables),
a na odlaznom **posle** obrade. Praktična posledica:

> Ako paket vidite u `tcpdump`-u, a aplikacija ga ne dobija, krivac je gotovo sigurno
> firewall ili rutiranje na samom hostu. Ako paket ne vidite ni u `tcpdump`-u,
> problem je pre ovog servera — na mreži, kod klijenta ili u rutiranju.

Instalacija: `apt install tcpdump`, `dnf install tcpdump`.

```bash
tcpdump --version
```

```
tcpdump version 4.99.3
libpcap version 1.10.3 (with TPACKET_V3)
OpenSSL 3.0.11 19 Sep 2023
```

---

## 2. Dozvole

Za presretanje su potrebne `CAP_NET_RAW` i `CAP_NET_ADMIN`. Najčešće se koristi `sudo`,
ali trajno rešenje bez root prava je:

```bash
sudo setcap cap_net_raw,cap_net_admin=eip /usr/bin/tcpdump
```

Na Debianu/Ubuntu postoji i grupa `pcap`:

```bash
sudo usermod -aG pcap marko
```

Kada se `tcpdump` pokreće kao root ali dugo radi, obavezno spustite privilegije nakon
otvaranja interfejsa:

```bash
sudo tcpdump -i eth0 -Z tcpdump -w /var/cap/trace.pcap
```

`-Z KORISNIK` prebacuje proces na neprivilegovanog korisnika čim otvori soket i izlazni fajl.
Bez toga dugotrajan presretač radi kao root i predstavlja nepotrebnu izloženost —
`tcpdump` je istorijski imao ranjivosti u dekoderima protokola.

U kontejnerima je potreban `--cap-add=NET_ADMIN --cap-add=NET_RAW` ili `--net=host`.

---

## 3. Sintaksa

```
tcpdump [OPCIJE] [FILTER]
```

Filter je poslednji argument i piše se u BPF jeziku (odeljak 6). Zbog zagrada i specijalnih
znakova gotovo uvek ide pod navodnicima.

Spisak interfejsa:

```bash
tcpdump -D
```

```
1.eth0 [Up, Running, Connected]
2.lo [Up, Running, Loopback]
3.docker0 [Up, Running]
4.br-3a1b2c [Up, Running]
5.any (Pseudo-device that captures on all interfaces) [Up, Running]
```

Bez `-i` `tcpdump` bira prvi aktivan interfejs, što je retko ono što želite.

> **`-i any` ima ograničenja.** Koristi „Linux cooked“ format zaglavlja (SLL/SLL2),
> pa MAC adrese nisu vidljive, `-e` daje manje podataka, a neki filteri na nivou Etherneta
> ne rade. Za analizu na sloju 2 uvek navedite konkretan interfejs.

---

## 4. Opcije

### 4.1 Izbor izvora i količine

| Opcija | Značenje |
|---|---|
| `-i INTERFEJS` | interfejs; `any` za sve |
| `-D` | izlistaj dostupne interfejse |
| `-c N` | prekini posle N paketa |
| `-p` | **ne** uključuj promiskuitetni režim |
| `-Q in\|out\|inout` | samo dolazni, samo odlazni ili oba smera (Linux) |
| `-r FAJL` | čitaj iz `pcap` fajla umesto sa interfejsa |
| `-F FAJL` | učitaj filter iz fajla |
| `-s N` | **snaplen** — koliko bajtova po paketu se hvata (0 = ceo paket) |
| `-B N` | veličina bafera kernela u KiB |

`-c` je najvažnija zaštita od preplavljivanja terminala. Uvek počnite sa `-c 20`.

**Snaplen:** moderne verzije podrazumevaju 262144 bajta, dakle ceo paket. Za dugotrajno
presretanje na opterećenom serveru dovoljna su zaglavlja:

```bash
sudo tcpdump -i eth0 -s 96 -w /var/cap/zaglavlja.pcap
```

96 bajtova obuhvata Ethernet, IP i TCP zaglavlje sa opcijama. Fajl je nekoliko puta manji,
a analiza toka, retransmisija i RST paketa je i dalje potpuna. Za analizu sadržaja
(HTTP, DNS) potreban je pun paket.

### 4.2 Razrešavanje imena

| Opcija | Značenje |
|---|---|
| `-n` | ne razrešavaj IP adrese u imena hostova |
| `-nn` | dodatno ne razrešavaj ni brojeve portova u imena servisa |
| `-N` | ne prikazuj domen uz ime hosta |
| `-f` | ne razrešavaj adrese van lokalne mreže |

> **`-nn` je praktično obavezno.** Bez njega `tcpdump` za svaki paket radi reverse-DNS upit.
> Ti upiti su sami po sebi mrežni saobraćaj, pa ako presrećete port 53, ulazite u
> **povratnu petlju** u kojoj svaki prikazani paket generiše nove pakete.
> Uz to, na sporom ili nedostupnom DNS-u izlaz kasni sekundama po paketu.

### 4.3 Detaljnost prikaza

| Opcija | Značenje |
|---|---|
| `-v`, `-vv`, `-vvv` | sve više detalja (TTL, ID, dužina, kontrolne sume, dekodiranje protokola) |
| `-e` | prikaži zaglavlje sloja veze (MAC adrese, VLAN oznake) |
| `-q` | kratak prikaz, manje protokolskih detalja |
| `-x` | heksadecimalni ispis sadržaja (bez zaglavlja sloja veze) |
| `-xx` | heksadecimalno, uključujući zaglavlje sloja veze |
| `-X` | heksadecimalno **i** ASCII — najkorisnije za čitanje sadržaja |
| `-XX` | isto, uključujući zaglavlje sloja veze |
| `-A` | samo ASCII sadržaj |
| `-S` | apsolutni TCP sekvencioni brojevi umesto relativnih |
| `-#` | numeriši pakete |
| `-K` | ne proveravaj kontrolne sume |

`-K` rešava čestu zabunu: mrežne kartice računaju kontrolne sume same (checksum offload),
pa `tcpdump` odlazne pakete vidi **pre** nego što je suma upisana i prijavljuje
`cksum 0x0000 (incorrect)`. To nije greška u mreži.

### 4.4 Vremenske oznake

| Opcija | Format |
|---|---|
| (podrazumevano) | `14:22:07.481239` |
| `-t` | bez vremena |
| `-tt` | UNIX vreme sa mikrosekundama |
| `-ttt` | **razlika u odnosu na prethodni paket** |
| `-tttt` | pun datum i vreme |
| `-ttttt` | razlika u odnosu na prvi paket |
| `--time-stamp-precision=nano` | nanosekundna preciznost |

`-ttt` je nezamenljiv za merenje latencije: odmah se vidi da li je pauza između SYN-a i
SYN-ACK-a bila 0,2 ms ili 3 sekunde.

### 4.5 Zapisivanje u fajl

| Opcija | Značenje |
|---|---|
| `-w FAJL` | zapiši sirove pakete u `pcap` format |
| `-w -` | zapiši na standardni izlaz (za cevovod) |
| `-C N` | rotiraj kad fajl dostigne N miliona bajtova |
| `-G N` | rotiraj svakih N sekundi (ime fajla mora imati `strftime` oznake) |
| `-W N` | zadrži najviše N fajlova — **kružni bafer** |
| `-U` | ne baferiši pri pisanju u fajl (paket po paket) |
| `-Z KORISNIK` | spusti privilegije posle otvaranja interfejsa |
| `-P`, `--print` | prikazuj pakete i kada se piše u fajl |

Format `pcap` čuva pakete tačno kako su primljeni, pa se fajl kasnije može analizirati
u Wireshark-u, `tshark`-u ili ponovo u `tcpdump`-u sa drugim filterom.

### 4.6 Ostalo

| Opcija | Značenje |
|---|---|
| `-l` | linijsko baferisanje — **obavezno kad se izlaz šalje kroz cev** |
| `-L` | izlistaj tipove sloja veze za interfejs |
| `-y TIP` | izaberi tip sloja veze |
| `-d`, `-dd`, `-ddd` | ispiši kompajliran BPF program umesto presretanja |
| `-T TIP` | prinudno tumači pakete kao navedeni protokol |
| `-M TAJNA` | ključ za proveru TCP-MD5 potpisa |

---

## 5. Čitanje izlaza

### 5.1 TCP

```bash
sudo tcpdump -i eth0 -nn -c 5 'tcp port 22'
```

```
14:22:07.481239 IP 10.0.10.4.51422 > 10.0.10.15.22: Flags [S], seq 2847392019,
                win 64240, options [mss 1460,sackOK,TS val 3921042 ecr 0,nop,wscale 7], length 0
14:22:07.481402 IP 10.0.10.15.22 > 10.0.10.4.51422: Flags [S.], seq 118293042, ack 2847392020,
                win 65160, options [mss 1460,sackOK,TS val 8812004 ecr 3921042,nop,wscale 7], length 0
14:22:07.481688 IP 10.0.10.4.51422 > 10.0.10.15.22: Flags [.], ack 1, win 502,
                options [nop,nop,TS val 3921042 ecr 8812004], length 0
14:22:07.492011 IP 10.0.10.15.22 > 10.0.10.4.51422: Flags [P.], seq 1:42, ack 1, win 510,
                options [nop,nop,TS val 8812014 ecr 3921042], length 41
14:22:07.492244 IP 10.0.10.4.51422 > 10.0.10.15.22: Flags [.], ack 42, win 502, length 0
```

**Anatomija reda:**

```
14:22:07.481239 IP 10.0.10.4.51422 > 10.0.10.15.22: Flags [S], seq 2847392019, win 64240, ... length 0
└─ vreme        └─ proto └─ izvor:port  └─ odredište:port  └─ zastavice └─ sekvenca └─ prozor    └─ payload
```

**Zastavice (`Flags`):**

| Znak | Zastavica | Značenje |
|---|---|---|
| `S` | SYN | zahtev za uspostavljanje veze |
| `.` | ACK | potvrda (sama tačka = samo ACK, bez podataka) |
| `P` | PSH | proslediti podatke aplikaciji odmah |
| `F` | FIN | uredan završetak veze |
| `R` | RST | **nasilan prekid veze** |
| `U` | URG | hitni podaci |
| `W` | CWR | prozor zagušenja smanjen |
| `E` | ECE | eksplicitno obaveštenje o zagušenju |
| `[.]` bez ičega | | čist ACK |

Kombinacije: `[S.]` je SYN+ACK, `[P.]` je PSH+ACK, `[F.]` je FIN+ACK, `[R.]` je RST+ACK.

**Ostala polja:**

| Polje | Značenje |
|---|---|
| `seq 2847392019` | početni sekvencioni broj (kod SYN paketa je apsolutan) |
| `seq 1:42` | opseg bajtova u ovom segmentu, relativno u odnosu na početak veze |
| `ack 42` | očekuje se sledeći bajt broj 42 |
| `win 502` | oglašeni prijemni prozor, **pomnožen faktorom skaliranja** |
| `length 41` | broj bajtova korisnih podataka (bez zaglavlja) |
| `options [...]` | TCP opcije |

> **Relativni brojevi.** Posle prvog SYN-a `tcpdump` prikazuje sekvence relativno u odnosu
> na početak veze, što je znatno čitljivije. Za apsolutne vrednosti koristite `-S`.
> Ako presretanje počne usred već uspostavljene veze, `tcpdump` ne zna početnu vrednost
> i prikazuje apsolutne brojeve.

**TCP opcije koje se najčešće vide:**

| Opcija | Značenje |
|---|---|
| `mss 1460` | maksimalna veličina segmenta koju strana prihvata |
| `sackOK` | podržana selektivna potvrda |
| `wscale 7` | faktor skaliranja prozora (2⁷ = ×128) |
| `TS val X ecr Y` | vremenske oznake — `val` je moja, `ecr` je odjek tuđe |
| `nop` | popuna za poravnanje |

Prva tri reda gornjeg primera su **trostruko rukovanje**: SYN, SYN-ACK, ACK.
Razlika u vremenu između prva dva reda (163 mikrosekunde) je latencija ka serveru.

### 5.2 UDP i DNS

```bash
sudo tcpdump -i eth0 -nn -c 4 'udp port 53'
```

```
14:31:02.104211 IP 10.0.10.15.44120 > 10.0.10.1.53: 43120+ A? example.com. (29)
14:31:02.118904 IP 10.0.10.1.53 > 10.0.10.15.44120: 43120 1/0/0 A 93.184.216.34 (45)
14:31:05.201033 IP 10.0.10.15.51002 > 10.0.10.1.53: 51882+ AAAA? interni.local. (31)
14:31:10.201455 IP 10.0.10.15.51002 > 10.0.10.1.53: 51882+ AAAA? interni.local. (31)
```

**Objašnjenje DNS zapisa:**

| Deo | Značenje |
|---|---|
| `43120` | ID transakcije — povezuje upit i odgovor |
| `+` | zatražena rekurzija |
| `A?` | tip upita (`A`, `AAAA`, `MX`, `PTR`, `SOA`, `TXT`) |
| `example.com.` | ime koje se traži |
| `(29)` | veličina DNS poruke u bajtovima |
| `1/0/0` | u odgovoru: broj odgovora / autoritativnih / dodatnih zapisa |
| `A 93.184.216.34` | sam odgovor |

Treći i četvrti red pokazuju **ponovljeni upit sa istim ID-em posle 5 sekundi** —
odgovora nema, resolver ponavlja. To je jasan pokazatelj da DNS server ne odgovara
ili da odgovor negde nestaje.

Ostale oznake u DNS odgovorima: `NXDomain` (ime ne postoji), `ServFail` (greška servera),
`Refused` (server odbija upit), `*` (autoritativan odgovor), `|` (skraćen odgovor, sledi TCP).

### 5.3 ICMP

```
14:35:11.201 IP 10.0.10.15 > 8.8.8.8: ICMP echo request, id 4102, seq 1, length 64
14:35:11.223 IP 8.8.8.8 > 10.0.10.15: ICMP echo reply, id 4102, seq 1, length 64
14:35:14.502 IP 10.0.10.1 > 10.0.10.15: ICMP host 10.0.10.99 unreachable, length 46
14:35:20.118 IP 10.0.10.1 > 10.0.10.15: ICMP 10.0.10.15 udp port 514 unreachable, length 36
14:35:22.004 IP 10.0.10.1 > 10.0.10.15: ICMP 93.184.216.34 unreachable -
                need to frag (mtu 1400), length 556
```

Poslednji red je dragocen: ruter javlja da paket ne staje i traži fragmentaciju na MTU 1400.
Ako te poruke ne stižu do izvora (blokirane na firewall-u), nastaje **PMTU crna rupa** —
mali zahtevi prolaze, veliki odgovori nikada ne stignu, veza „visi“ bez greške.

### 5.4 ARP

```
14:40:01.100 ARP, Request who-has 10.0.10.99 tell 10.0.10.15, length 28
14:40:01.100 ARP, Reply 10.0.10.99 is-at 00:1a:2b:3c:4d:5e, length 46
14:40:05.200 ARP, Reply 10.0.10.15 is-at 00:de:ad:be:ef:00, length 46
```

Treći red je sumnjiv: neko šalje ARP odgovor za **našu** adresu sa tuđom MAC adresom.
To je ili duplirana IP adresa ili pokušaj ARP trovanja.

### 5.5 Statistika na kraju

Posle `Ctrl+C`:

```
^C
1247 packets captured
1389 packets received by filter
142 packets dropped by kernel
```

| Broj | Značenje |
|---|---|
| `packets captured` | paketi koje je `tcpdump` obradio i prikazao ili zapisao |
| `packets received by filter` | paketi koje je filter propustio ka `tcpdump`-u |
| **`packets dropped by kernel`** | **paketi koje je kernel odbacio jer bafer nije bio ispražnjen na vreme** |

Bilo koji broj veći od nule u trećem redu znači da je presretanje **nepotpuno** i da zaključci
mogu biti pogrešni. Rešenja u odeljku 8.

---

## 6. BPF filter jezik

Filteri se kompajliraju u BPF program i izvršavaju **u kernelu**. Zato je filtriranje u
`tcpdump`-u nesamerljivo efikasnije od `tcpdump ... | grep` — nefiltrirani paketi nikada
ne stižu do korisničkog prostora.

### 6.1 Struktura primitiva

Primitiv se sastoji od tri vrste kvalifikatora i vrednosti:

```
[proto] [dir] [type] vrednost
```

| Vrsta | Vrednosti |
|---|---|
| **type** | `host`, `net`, `port`, `portrange` |
| **dir** | `src`, `dst`, `src or dst` (podrazumevano), `src and dst` |
| **proto** | `ether`, `ip`, `ip6`, `arp`, `rarp`, `tcp`, `udp`, `icmp`, `icmp6` |

```bash
'host 10.0.10.15'
'src host 10.0.10.4'
'dst net 192.168.1.0/24'
'tcp port 443'
'udp dst port 53'
'tcp portrange 8000-8100'
'ether host 00:1a:2b:3c:4d:5e'
'ip6 host 2001:db8::1'
```

Izostavljanje kvalifikatora: `host` je podrazumevani tip, oba smera su podrazumevana,
a bez protokola se hvataju svi.

### 6.2 Ostali primitivi

| Primitiv | Značenje |
|---|---|
| `arp`, `rarp`, `icmp`, `tcp`, `udp` | samo protokol |
| `ip broadcast`, `ip multicast` | emitovanje |
| `ether broadcast`, `ether multicast` | na sloju 2 |
| `less N`, `greater N` | dužina paketa |
| `gateway HOST` | paketi koje host rutira, a nisu njemu namenjeni |
| `vlan [ID]` | VLAN označen saobraćaj |
| `mpls [oznaka]` | MPLS |
| `ip proto \tcp` | po broju protokola u IP zaglavlju (obrnuta kosa crta je obavezna) |
| `ip protochain \tcp` | prati lanac zaglavlja (IPv6 ekstenzije) |

### 6.3 Logički operatori

`and` / `&&`, `or` / `||`, `not` / `!`, zagrade `( )`.

```bash
sudo tcpdump -i eth0 -nn 'host 10.0.10.4 and tcp port 443'
sudo tcpdump -i eth0 -nn 'port 80 or port 443'
sudo tcpdump -i eth0 -nn 'not port 22'
sudo tcpdump -i eth0 -nn 'src net 10.0.0.0/8 and not dst port 53'
```

> **Ceo filter pod jednostrukim navodnicima.** Zagrade, `!` i `|` su specijalni znaci
> u shell-u. Bez navodnika komanda ili ne radi ili radi nešto sasvim drugo.

Klasična greška u prioritetu:

```bash
'not host 10.0.10.4 and port 80'      # (ne 10.0.10.4) I (port 80)
'not (host 10.0.10.4 and port 80)'    # ne (10.0.10.4 na portu 80)
```

### 6.4 Pristup bajtovima

Najmoćniji deo jezika: `protokol[pomeraj:veličina]`, gde je veličina 1, 2 ili 4 bajta.

```bash
'ip[8] < 5'                    # TTL manji od 5
'ip[6] & 0x40 != 0'            # postavljen Don't Fragment bit
'ip[6:2] & 0x1fff != 0'        # fragmentovan paket koji nije prvi
'tcp[13] & 4 != 0'             # postavljen RST bit
```

Za TCP zastavice postoje čitljive konstante:

```
tcp-fin  tcp-syn  tcp-rst  tcp-push  tcp-ack  tcp-urg
```

```bash
# Samo SYN, bez ACK-a — pokušaji uspostavljanja veze
'tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn'

# Svi RST paketi — ko prekida veze
'tcp[tcpflags] & tcp-rst != 0'

# SYN ili FIN — početak i kraj veza
'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0'
```

Za ICMP:

```bash
'icmp[icmptype] == icmp-echo'
'icmp[icmptype] == icmp-unreach'
'icmp[icmptype] == 3 and icmp[icmpcode] == 4'    # fragmentacija potrebna (PMTU)
```

### 6.5 Filtriranje po sadržaju

Pomeraj do početka podataka u TCP paketu je promenljiv, jer zaglavlje ima promenljivu dužinu.
Dužina se nalazi u gornja četiri bita bajta 12:

```
((tcp[12] & 0xf0) >> 2)
```

```bash
# Paketi koji uopšte nose podatke
'tcp and ((ip[2:2] - ((ip[0] & 0x0f) << 2)) - ((tcp[12] & 0xf0) >> 2)) != 0'

# HTTP zahtevi koji počinju sa "GET "
'tcp port 80 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'

# HTTP zahtevi koji počinju sa "POST"
'tcp port 80 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504f5354'

# Početak TLS rukovanja (record type 0x16 = handshake)
'tcp port 443 and tcp[((tcp[12:1] & 0xf0) >> 2)] = 0x16'
```

Heksadecimalne vrednosti su ASCII kodovi: `0x47455420` je `G`, `E`, `T`, razmak.

### 6.6 Provera kompajliranog filtera

```bash
tcpdump -d 'tcp port 443'
```

```
(000) ldh      [12]
(001) jeq      #0x86dd          jt 2	jf 8
(002) ldb      [20]
(003) jeq      #0x6             jt 4	jf 19
...
```

Korisno kada filter ne radi kako se očekuje — vidi se šta je `tcpdump` zaista razumeo.

---

## 7. Praktični scenariji

### 7.1 Da li paketi uopšte stižu

```bash
sudo tcpdump -i any -nn -c 20 'host 203.0.113.44'
```

Tri moguća ishoda i njihovo tumačenje:

| Šta se vidi | Zaključak |
|---|---|
| ništa | paketi ne stižu do servera — problem je u mreži, rutiranju ili kod klijenta |
| samo dolazni SYN, bez odgovora | paketi stižu, ali ih odbacuje firewall na hostu (`DROP`) ili ništa ne sluša na portu uz `DROP` politiku |
| SYN pa odmah `R` iz našeg pravca | na portu ništa ne sluša, kernel odbija vezu |
| SYN, SYN-ACK, ACK | veza je uspostavljena, problem je viši u steku |

Ova podela je najvrednija stvar koju `tcpdump` daje: razdvaja „mreža“ od „host“ od „aplikacija“
u jednom potezu.

### 7.2 Neuspešno rukovanje

```bash
sudo tcpdump -i eth0 -nn -ttt 'tcp port 5432 and tcp[tcpflags] & (tcp-syn|tcp-rst) != 0'
```

```
 00:00:00.000000 IP 10.0.10.15.44120 > 10.0.10.99.5432: Flags [S], seq 1042, win 64240, length 0
 00:00:01.000112 IP 10.0.10.15.44120 > 10.0.10.99.5432: Flags [S], seq 1042, win 64240, length 0
 00:00:02.000208 IP 10.0.10.15.44120 > 10.0.10.99.5432: Flags [S], seq 1042, win 64240, length 0
 00:00:04.000411 IP 10.0.10.15.44120 > 10.0.10.99.5432: Flags [S], seq 1042, win 64240, length 0
```

Ponovljeni SYN sa **istim sekvencionim brojem** u intervalima 1, 2, 4 sekunde je
eksponencijalno odustajanje kernela. Odgovora nema — paketi se tiho odbacuju
(firewall sa `DROP` politikom ili pogrešno rutiranje).

Suprotan slučaj:

```
 00:00:00.000000 IP 10.0.10.15.44122 > 10.0.10.99.5432: Flags [S], seq 2041, win 64240, length 0
 00:00:00.000203 IP 10.0.10.99.5432 > 10.0.10.15.44122: Flags [R.], seq 0, ack 2042, win 0, length 0
```

Trenutan RST znači da je paket stigao, ali na tom portu ništa ne sluša ili firewall
koristi `REJECT` umesto `DROP`. Razlika između tišine i RST-a je razlika između
`DROP` i `REJECT` pravila.

### 7.3 Ko prekida uspostavljene veze

```bash
sudo tcpdump -i eth0 -nn -tttt 'tcp[tcpflags] & tcp-rst != 0'
```

Smer izvora u RST paketu pokazuje krivca. Ako RST dolazi sa naše strane, aplikacija ili
kernel prekidaju vezu; ako dolazi spolja, prekid je kod klijenta ili na posredniku
(load balancer, firewall sa timeoutom stanja).

Česta pojava: firewall sa praćenjem stanja izbacuje neaktivne veze iz tabele posle
nekoliko minuta i zatim šalje RST na prvi sledeći paket. Simptom je aplikacija koja
radi kad je opterećena, a puca posle perioda mirovanja. Rešenje su TCP keepalive
paketi kraći od timeouta firewall-a.

### 7.4 Dijagnostika DNS-a

```bash
sudo tcpdump -i any -nn -s 0 'port 53'
```

Šta tražiti:

- upit bez odgovora → server ne odgovara ili je odgovor blokiran,
- `NXDomain` → ime stvarno ne postoji,
- `ServFail` → problem na strani DNS servera, često DNSSEC,
- `Refused` → server ne opslužuje ovog klijenta,
- upit ide ka pogrešnom serveru → pogrešan `/etc/resolv.conf` ili `systemd-resolved`,
- odgovor sa oznakom `|` pa isti upit preko TCP-a → odgovor je prevelik za UDP.

Presretanje odgovora većih od 512 bajta, koji često izazivaju probleme na starim firewall-ima:

```bash
sudo tcpdump -i any -nn 'udp port 53 and greater 512'
```

### 7.5 Duplirana IP adresa i ARP problemi

```bash
sudo tcpdump -i eth0 -nn -e 'arp'
```

```
14:40:05.200 00:de:ad:be:ef:00 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42:
             Reply 10.0.10.15 is-at 00:de:ad:be:ef:00, length 28
```

`-e` prikazuje MAC adrese, bez čega je ARP analiza besmislena. Ako za istu IP adresu
vidite odgovore sa dve različite MAC adrese, imate dupliranu adresu ili ARP trovanje.

Provera sopstvene adrese:

```bash
sudo arping -D -I eth0 10.0.10.15
```

### 7.6 MTU i fragmentacija

```bash
sudo tcpdump -i any -nn 'icmp[icmptype] == 3 and icmp[icmpcode] == 4'
```

Ako ove poruke vidite, PMTU otkrivanje radi. Ako ne vidite ništa, a veliki transferi
zastaju dok mali prolaze, poruke se filtriraju negde na putu.

Provera stvarnog MTU-a bez `tcpdump`-a:

```bash
ping -M do -s 1472 -c 3 10.0.10.99      # 1472 + 28 = 1500
```

Fragmentovani paketi u presretanju:

```bash
sudo tcpdump -i eth0 -nn 'ip[6:2] & 0x3fff != 0'
```

### 7.7 DHCP

```bash
sudo tcpdump -i eth0 -nn -v 'port 67 or port 68'
```

Vidi se ceo redosled DISCOVER → OFFER → REQUEST → ACK, uključujući ponuđenu adresu,
zakup i opcije. Neophodno kada klijent ne dobija adresu, a serverski log ćuti — često
zato što je u istom segmentu drugi, neovlašćen DHCP server.

### 7.8 Provera da li je saobraćaj šifrovan

```bash
sudo tcpdump -i eth0 -nn -A -s 0 'tcp port 8080 and greater 100'
```

Ako u ASCII prikazu vidite čitljive HTTP zaglavlja, korisnička imena ili tokene,
saobraćaj nije šifrovan. Ovo je najbrži način da se opovrgne tvrdnja „aplikacija koristi TLS“.

Za HTTP zahteve isključivo:

```bash
sudo tcpdump -i eth0 -nn -A -s 0 'tcp port 80 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'
```

### 7.9 Presretanje bez sopstvene SSH sesije

```bash
sudo tcpdump -i eth0 -nn 'not port 22'
```

Bez ovoga svaki prikazani paket generiše izlaz koji putuje kroz vašu SSH vezu,
koji `tcpdump` opet presreće — nastaje lavina. Ako koristite nestandardni port,
prilagodite izuzetak:

```bash
sudo tcpdump -i eth0 -nn "not (host $(echo $SSH_CLIENT | awk '{print $1}') and port 22)"
```

### 7.10 Dugotrajno presretanje sa rotacijom

```bash
sudo tcpdump -i eth0 -nn -s 96 -Z tcpdump \
     -w /var/cap/trace-%Y%m%d-%H%M%S.pcap \
     -G 3600 -W 24 -C 200 \
     'not port 22'
```

- `-G 3600` rotira svakih sat vremena; ime fajla **mora** sadržati `strftime` oznake.
- `-W 24` zadržava najviše 24 fajla — kružni bafer od 24 sata.
- `-C 200` dodatno rotira ako fajl pređe 200 MB.
- `-s 96` drži fajlove malim.
- `-Z tcpdump` spušta privilegije.

Ovako se hvata problem koji se javlja nepredvidivo: kada se pojavi, dokazi su već u fajlu.

Kroz systemd, da preživi odjavu:

```ini
[Unit]
Description=Trajno presretanje mrežnog saobraćaja

[Service]
ExecStart=/usr/bin/tcpdump -i eth0 -nn -s 96 -Z tcpdump \
          -w /var/cap/trace-%%Y%%m%%d-%%H%%M%%S.pcap -G 3600 -W 24 not port 22
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Znakovi procenta se u systemd jedinicama udvostručuju.

### 7.11 Naknadna analiza `pcap` fajla

Presretanje i analiza su odvojeni poslovi. Uhvatite široko, analizirajte usko:

```bash
sudo tcpdump -i eth0 -nn -s 0 -w /tmp/uhvaceno.pcap -c 100000
tcpdump -r /tmp/uhvaceno.pcap -nn 'host 10.0.10.99 and tcp port 443' | head -50
tcpdump -r /tmp/uhvaceno.pcap -nn 'tcp[tcpflags] & tcp-rst != 0' | wc -l
```

Filter pri čitanju ne mora biti isti kao pri presretanju. Zato je bolje uhvatiti nešto
šire nego propustiti pakete koji se kasnije ispostave kao ključni.

Statistika iz fajla, u kombinaciji sa `awk` iz prethodnog uputstva:

```bash
tcpdump -r /tmp/uhvaceno.pcap -nn 2>/dev/null \
| awk '{split($3, s, "."); ip = s[1]"."s[2]"."s[3]"."s[4]; c[ip]++}
       END {for (i in c) printf "%8d  %s\n", c[i], i}' | sort -rn | head
```

```
   84213  203.0.113.44
    3102  198.51.100.9
     871  10.0.10.4
```

### 7.12 Udaljeno presretanje u Wireshark

```bash
ssh -p 22 root@10.0.10.15 'tcpdump -i eth0 -nn -s 0 -U -w - not port 22' \
| wireshark -k -i -
```

- `-w -` šalje `pcap` tok na standardni izlaz.
- `-U` ne baferiše, pa paketi stižu odmah.
- `not port 22` je obavezno — bez toga se SSH tok sam presreće.

### 7.13 Presretanje u kontejneru

Kontejner ima svoj mrežni namespace, pa `tcpdump` sa hosta ne vidi njegov saobraćaj:

```bash
PID=$(docker inspect -f '{{.State.Pid}}' ime-kontejnera)
sudo nsenter -t "$PID" -n tcpdump -i eth0 -nn -c 50
```

`tcpdump` se ovako pokreće sa hosta, u mrežnom namespace-u kontejnera, pa ga nije potrebno
instalirati u samu sliku.

Za Kubernetes pod:

```bash
PID=$(crictl inspect --output go-template --template '{{.info.pid}}' KONTEJNER_ID)
sudo nsenter -t "$PID" -n tcpdump -i any -nn
```

### 7.14 Slanje izlaza kroz cev

```bash
sudo tcpdump -i eth0 -nn -l 'tcp port 80' | awk '/Flags \[R/ {print; fflush()}'
```

`-l` uključuje linijsko baferisanje. Bez njega `tcpdump` baferiše izlaz u blokovima od
nekoliko kilobajta i sledeći alat u cevovodu ništa ne vidi minutima.

---

## 8. Odbačeni paketi i performanse

Kada `packets dropped by kernel` nije nula, presretanje je nepotpuno.

| Uzrok | Rešenje |
|---|---|
| bafer kernela premali | `-B 8192` (u KiB) |
| ispis na terminal usporava obradu | `-w fajl` umesto prikaza |
| razrešavanje imena | `-nn` |
| hvata se ceo paket bez potrebe | `-s 96` |
| previše paketa prolazi filter | precizniji filter — obrada je u kernelu, pa filter štedi sve dalje |
| spor disk | pisati na brži uređaj ili u `tmpfs` |

```bash
sudo tcpdump -i eth0 -nn -s 96 -B 16384 -w /var/cap/trace.pcap 'port 443'
```

Dodatne pojave koje zbunjuju pri analizi:

- **Checksum offload** — odlazni paketi imaju „pogrešne“ sume jer ih kartica računa posle presretanja. Koristite `-K` ili zanemarite.
- **TSO/GSO/GRO** — kartica spaja segmente, pa se u presretanju vide paketi veći od MTU-a (npr. 24000 bajtova). To nije greška. Za verniju sliku privremeno isključite: `sudo ethtool -K eth0 tso off gso off gro off` (uticaj na performanse je primetan, vratite posle analize).
- **Bond i bridge interfejsi** — presretanje na `bond0` i na `eth0` daje različite rezultate; kod problema proverite oba.

---

## 9. Česte greške

1. **Bez `-nn`** — DNS upiti usporavaju izlaz i stvaraju povratnu petlju pri presretanju porta 53.
2. **Bez `-c`** — terminal se preplavi za sekundu.
3. **Presretanje sopstvene SSH sesije** — lavina saobraćaja. Uvek `not port 22`.
4. **Filter bez navodnika** — shell tumači zagrade i `!`.
5. **Pogrešan prioritet `not`** — `not host X and port Y` nije isto što i `not (host X and port Y)`.
6. **`-i any` za analizu na sloju 2** — MAC adrese nisu dostupne.
7. **Zaključivanje da mreža radi jer paketi stižu** — `tcpdump` dolazne pakete vidi pre firewall-a.
8. **Zanemarivanje `packets dropped by kernel`** — zaključci na osnovu nepotpunih podataka.
9. **Tumačenje „bad cksum“ kao problema** — posledica checksum offload-a na odlaznim paketima.
10. **Čuđenje zbog paketa većih od MTU-a** — posledica GRO/TSO.
11. **Bez `-l` u cevovodu** — sledeći alat ne dobija ništa zbog baferisanja.
12. **`-G` bez `strftime` oznaka u imenu fajla** — rotacija ne radi.
13. **Dugotrajno presretanje kao root** — koristite `-Z`.
14. **Presretanje sa hosta za saobraćaj kontejnera** — pogrešan mrežni namespace.
15. **Analiza šifrovanog saobraćaja u `tcpdump`-u** — vidi se samo TLS rukovanje i SNI; sadržaj zahteva ključeve i Wireshark.

---

## 10. Podsetnik (cheat sheet)

```bash
tcpdump -D                                              # spisak interfejsa
tcpdump -i eth0 -nn -c 20                               # prvih 20 paketa
tcpdump -i any -nn 'host 10.0.10.99'                    # sve ka/od hosta
tcpdump -i eth0 -nn 'tcp port 443'                      # po portu
tcpdump -i eth0 -nn 'not port 22'                       # bez sopstvene SSH sesije
tcpdump -i eth0 -nn -ttt 'tcp port 5432'                # razmaci između paketa
tcpdump -i eth0 -nn 'tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn'   # samo SYN
tcpdump -i eth0 -nn 'tcp[tcpflags] & tcp-rst != 0'      # ko prekida veze
tcpdump -i any -nn -s 0 'port 53'                       # DNS
tcpdump -i eth0 -nn -e 'arp'                            # ARP sa MAC adresama
tcpdump -i any -nn 'icmp'                               # ICMP
tcpdump -i any -nn 'icmp[icmptype]==3 and icmp[icmpcode]==4'  # PMTU
tcpdump -i eth0 -nn -A -s 0 'tcp port 80'               # sadržaj u ASCII
tcpdump -i eth0 -nn -X -s 0 'port 8080'                 # hex + ASCII
tcpdump -i eth0 -nn -s 0 -w /tmp/t.pcap -c 100000       # zapis u fajl
tcpdump -r /tmp/t.pcap -nn 'host 10.0.10.99'            # analiza iz fajla
tcpdump -i eth0 -s 96 -Z tcpdump -w /var/cap/t-%Y%m%d-%H%M%S.pcap -G 3600 -W 24
ssh host 'tcpdump -i eth0 -nn -s0 -U -w - not port 22' | wireshark -k -i -
nsenter -t $(docker inspect -f '{{.State.Pid}}' ime) -n tcpdump -i eth0 -nn
tcpdump -d 'tcp port 443'                               # provera filtera
```

---

## 11. Povezani alati

| Alat | Kada ga koristiti uz ili umesto `tcpdump` |
|---|---|
| `wireshark` | grafička analiza, praćenje toka, dekodiranje stotina protokola |
| `tshark` | Wireshark u terminalu; ima statistike (`-z conv,tcp`, `-z io,stat`) koje `tcpdump` nema |
| `termshark` | tekstualni interfejs nalik Wireshark-u |
| `ss -tinp` | stanje veza i TCP metrike bez presretanja i bez usporavanja |
| `nstat -az` | kernel brojači: retransmisije, odbačeni SYN paketi, greške |
| `conntrack -L` | tabela praćenja veza na firewall-u |
| `iptables -L -v -n` / `nft list ruleset` | brojači po pravilu — pokazuju šta se odbacuje |
| `mtr` | latencija i gubitak po skoku na putanji |
| `ngrep` | pretraga sadržaja paketa po šablonu, čitljivije od `-A` |
| `iftop` / `nethogs` | ko troši propusni opseg, u realnom vremenu |
| `bpftrace` / `pwru` | praćenje putanje paketa **kroz kernel**, uključujući mesto odbacivanja |

Redosled koji štedi vreme: prvo `ss` i `nstat` (bez uticaja na sistem), pa brojači na
firewall-u, i tek onda `tcpdump`. Presretanje daje najviše podataka, ali i najviše šuma.
