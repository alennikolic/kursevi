# ss - Analiza mrežnih soketa i aktivnih konekcija

## 1. Uvod i namena
`ss` (socket statistics) je savremena Linux komanda za detaljnu analizu mrežnih soketa i konekcija. Deluje kao direktna, znatno brža zamena za stariji `netstat` alat jer podatke prikuplja direktno iz Kernel space-a (preko Netlink subsystema) umesto da sporo parsira datoteke iz `/proc/net/` fajlsistema.

U produkcionom okruženju koristi se za brzu mrežnu dijagnostiku, detekciju zauzetih portova od strane mikroservisa ili Docker kontejnera, kao i za praćenje potencijalnih DDoS napada identifikacijom neobično velikog broja konekcija u stanjima kao što su `SYN_RECV` ili `TIME_WAIT`.

## 2. Sintaksa i opcije (Flags)
`ss [opcije] [FILTER]`

| Opcija | Dugi oblik | Značenje / Ponašanje |
|:---:|:---|:---|
| `-t` | `--tcp` | Prikazuje samo TCP sokete. |
| `-u` | `--udp` | Prikazuje samo UDP sokete. |
| `-l` | `--listening` | Prikazuje samo listening sokete (servise koji čekaju konekcije). |
| `-a` | `--all` | Prikazuje i listening i non-listening (aktivne) sokete. |
| `-p` | `--processes` | Prikazuje PID i naziv procesa koji vlasnik soketa. |
| `-n` | `--numeric` | Prikazuje portove i IP adrese kao brojeve (ne vrši DNS/service lookup). |
| `-s` | `--summary` | Prikazuje zbirnu statistiku soketa po tipovima i stanjima. |
| `-e` | `--extended` | Prikazuje dodatne detalje o soketu (UID, inode, socket cookie). |

## 3. Analiza izlaza (Output Breakdown)

```bash
$ sudo ss -tulpn
Netid  State      Recv-Q Send-Q  Local Address:Port   Peer Address:Port  Process                                                                                                            
udp    UNCONN     0      0          0.0.0.0:68           0.0.0.0:*      users:(("dhclient",pid=842,fd=6))                                                                                  
tcp    LISTEN     0      128        0.0.0.0:22           0.0.0.0:*      users:(("sshd",pid=1045,fd=3))                                                                                     
tcp    LISTEN     0      511      127.0.0.1:8080         0.0.0.0:*      users:(("php-fpm",pid=2150,fd=7),("php-fpm",pid=2149,fd=7))                                                        
tcp    ESTAB      0      0     192.168.1.100:22      192.168.1.45:54322  users:(("sshd",pid=3102,fd=4))
```

**Objašnjenje prikaza:**
- **Netid:** Protokol koji soket koristi (`tcp`, `udp`, `raw`, `unix`).
- **State:** Trenutno stanje soketa. Za TCP: `LISTEN` (čeka konekciju), `ESTAB` (uspostavljena konekcija), `UNCONN` (UDP konekcija bez stanja), `TIME_WAIT`, `SYN_SENT`.
- **Recv-Q:** Broj bajtova u prijemnom baferu. Za `LISTEN` sokete označava trenutni broj neprirađenih konekcija u backlog redu. Ako je vrednost visoka, aplikacija ne stiže da obrađuje dolazne zahteve.
- **Send-Q:** Broj bajtova u slateljskom baferu. Za `LISTEN` sokete označava maksimalnu veličinu backlog reda. Ako je Recv-Q blizu Send-Q vrednosti, mrežni keš aplikacije je preopterećen.
- **Local Address:Port:** IP adresa i port na kome servis sluša ili sa koga je inicirana konekcija (`0.0.0.0` znači da sluša na svim mrežnim interfejsima).
- **Peer Address:Port:** Odredišna IP adresa i port udaljenog klijenta (`0.0.0.0:*` znači da nema definisanog peer-a).
- **Process:** Naziv procesa, njegov PID i File Descriptor (FD) broj koji drži soket otvorenim.

## 4. Praktični primeri iz produkcije

- **Svrha:** Pronalazak procesa koji drži otvoren specifičan port (npr. port 80 ili 443).
- **Komanda:** 
  ```bash
  sudo ss -tulpn 'sport = :80 or sport = :443'
  ```
- **Objašnjenje:** Koristi ugrađeno filtriranje `sport` (source port) kako bi prikazao samo sokete posvećene web saobraćaju uz PID i naziv aplikacije.

- **Svrha:** Prikaz svih aktivnih i uspostavljenih TCP konekcija ka spoljnim serverima bez listening soketa.
- **Komanda:** 
  ```bash
  ss -nt state established
  ```
- **Objašnjenje:** Opcija `state established` filtrira isključivo aktivno uspostavljene sesije, što eliminiše šum servisa u stanu slušanja.

- **Svrha:** Otkrivanje top 10 IP adresa sa najviše uspostavljenih konekcija ka serveru (detekcija zloupotrebe).
- **Komanda:** 
  ```bash
  ss -nt state established | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr | head -n 10
  ```
- **Objašnjenje:** Kombinuje `ss` sa `awk` i `cut` za izdvajanje samo IP adrese udaljenog računara, zatim prebrojava jedinstvene adrese i sortira ih u opadajućem redosledu.

## 5. Pro Tips i "Gotchas" (Zamke)
- Za prikaz informacija o procesima (`-p` opcija), komanda mora biti pokrenuta sa `sudo` ili kao `root` korisnik; u suprotnom, kolona Process ostaje prazna.
- Uvek koristite `-n` opciju u produkciji jer DNS razrešavanje imena hostova na serverima sa hiljadama konekcija može dovesti do privremenog zamrzavanja izvršavanja komande.
- Zamenite starije `netstat -an` skripte sa `ss -an` radi značajnih ušteda na CPU resursima tokom automatskog monitoringa.
