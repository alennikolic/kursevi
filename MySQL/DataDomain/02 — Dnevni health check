# 02 — Dnevni health check

Redosled provere kada se ujutru logujete na uređaj ili kada vas neko pita
"da li je DD u redu". Sve komande u ovom poglavlju su **read-only** —
nijedna ne menja konfiguraciju i sve se mogu pustiti u produkciji u bilo koje doba.

---

## 1. Orijentacija — gde sam

```
system show version              # DDOS verzija i build
system show detailed-version     # verzije pojedinačnih komponenti
system show serialno             # serijski broj (za Dell support case)
system show uptime               # koliko sistem radi
system show modelno              # model uređaja
```

Ako radite na više uređaja u seriji, prvo ovo — sprečava da se komanda odradi na pogrešnoj kutiji.

---

## 2. Da li nešto trenutno gori

```
alerts show current                  # aktivni alerti
alerts show current-hardware         # samo hardverski alerti
alerts show history                  # istorija (podrazumevano poslednja 24h)
alerts show history last 7 days
```

**Kako čitati:** svaki alert ima klasu, severity i timestamp. `CRITICAL` i `EMERGENCY`
znače da se otvara case. `WARNING` na temu kapaciteta je najčešći i vodi u poglavlje 04.

Ko dobija mejlove o alertima:

```
alerts notify-list show
autosupport show all                 # da li AutoSupport ide ka Dell-u
```

---

## 3. Osnovni status sistema

```
system status                    # sažet status komponenti
system show hardware             # inventar hardvera
system show all                  # sve zajedno (dugačak output)
```

Hardver detaljnije:

```
disk show state                  # mapa diskova: in-use / spare / failed / unknown
disk show hardware               # model, firmware, serijski broj po disku
disk show reliability-data       # SMART-like podaci
enclosure show all               # police / šasije
```

**Šta tražiti:** bilo koji disk u stanju `failed`, `absent` ili `foreign`.
Jedan `spare` koji je preuzeo mesto `failed` diska znači da rekonstrukcija
ide ili je gotova — proveriti `disk show state` i alerte.

---

## 4. File system i kapacitet — najvažniji deo

```
filesys status                   # da li je FS enabled i running
filesys show space               # zauzeće po sekcijama
filesys show compression         # ukupan faktor dedupe/kompresije
```

`filesys show space` daje tabelu sa redovima koje treba znati napamet:

| Red | Šta je | Na šta paziti |
|---|---|---|
| `/data: pre-comp` | Koliko su podaci veliki *pre* dedupe/kompresije — logička veličina | Raste najbrže, nije alarm sam po sebi |
| `/data: post-comp` | Stvarno zauzeće na disku | **Ovo je red koji se prati.** Preko 80% → planiranje, preko 90% → akcija |
| `/ddvar` | Sistemska particija: logovi, support bundle-ovi, upgrade paketi | Ako se popuni, GUI i logovanje počinju da otkazuju. Čisti se starim bundle-ovima |
| `/ddvar/core` | Core dump-ovi | Ako raste — nešto je crash-ovalo, gledaj logove |

> **Prag za akciju:** post-comp `/data` iznad 85% je granica gde cleaning više ne
> stiže da oslobodi dovoljno prostora između backup prozora. Iznad 95% file system
> prelazi u read-only i backup-i počinju da padaju.

Dodatno:

```
filesys show space tier active
filesys show space tier cloud        # samo ako je Cloud Tier licenciran
filesys clean status                 # da li cleaning ide i dokle je stigao
filesys clean show config            # raspored i throttle cleaninga
```

Detalji o kapacitetu, cleaningu i o tome šta raditi kad je puno → **poglavlje 04**.

---

## 5. MTree pregled

```
mtree list                                       # svi MTree-jevi, status, kvote, RL status
mtree show compression                           # zauzeće po MTree-ju
mtree show compression /data/col1/<mtree> last 24 hours
quota capacity show                              # postavljene kvote i koliko su popunjene
```

`mtree list` u jednoj tabeli pokazuje da li je MTree `RW`, `RO` ili `RD` (replication destination),
i da li ima uključen Retention Lock. To je najbrži način da vidite koji MTree-jevi
su zaključani, a koji su odredište replikacije.

Detaljno → **poglavlje 05**, Retention Lock → **poglavlje 06**.

---

## 6. Replikacija

```
replication status                       # sažeto stanje svih konteksta
replication show config                  # definicija svih konteksta
replication show detailed-status         # detaljno, po kontekstu
replication show performance             # trenutna brzina replikacije
replication show stats                   # kumulativna statistika
```

**Šta se gleda u `replication status`:**

| Kolona | Značenje |
|---|---|
| `State` | `initializing`, `normal`, `disabled`, `disconnected`, `error` |
| `Sync'ed-as-of-time` | Do kog trenutka je odredište konzistentno sa izvorom. **Ovo je stvarni RPO.** |
| `Pre-comp Remaining` | Koliko logičkih podataka još treba preneti — mera zaostajanja |

Ako `Sync'ed-as-of-time` zaostaje više od dogovorenog RPO-a, ide se u
troubleshooting replikacije (**poglavlje 07**, playbook u **poglavlju 10**).

Da se prati kontekst uživo:

```
replication watch <destination>
```

---

## 7. Snapshot-ovi

```
snapshot list mtree /data/col1/<mtree>
snapshot schedule show
```

Traže se snapshot-ovi kojima je isteklo vreme a nisu obrisani, i rasporedi
koji su prestali da rade. Detaljno → **poglavlje 05**.

---

## 8. Mreža i opterećenje

```
net show settings                # IP adrese, MTU, stanje interfejsa
net show hardware                # fizički portovi, brzina, link status
net show stats                   # kumulativne statistike, greške, drop-ovi
net aggregate show               # LACP / agregacija
net failover show
```

Trenutno opterećenje u realnom vremenu:

```
system show stats interval 2                 # osvežava se na 2 sekunde
system show performance                      # throughput po protokolima
net show stats interval 2 count 10           # mrežni promet po interfejsu
```

**Šta tražiti:** rastući `errors`, `dropped` ili `collisions` na interfejsu,
i link koji je pao na nižu brzinu nego što bi trebalo (npr. 1 Gb umesto 10 Gb).
Detaljno → **poglavlje 08**.

---

## 9. Aktivne konekcije klijenata

```
ddboost show connections         # aktivne DD Boost sesije
ddboost storage-unit show        # storage unit-ovi i njihovo zauzeće
nfs show active                  # aktivni NFS klijenti
cifs show active                 # aktivne CIFS/SMB sesije
```

Detaljno → **poglavlje 09**.

---

## 10. Logovi ako nešto ne štima

```
log list                         # koji log fajlovi postoje
log view                         # tekući messages log (interaktivno)
log view messages.engineering
log watch                        # prati log uživo, kao tail -f
```

---

## Sažetak — copy/paste blok za dnevnu proveru

```
system show version
system show uptime
alerts show current
disk show state
filesys status
filesys show space
filesys clean status
mtree list
replication status
net show hardware
```

Ovo je ~30 sekundi rada i pokriva sve što se najčešće pita.
Ako je bilo koja od ovih komandi vratila nešto neočekivano,
otvorite odgovarajuće dublje poglavlje.

---

## Predlog za automatizaciju

Sve gore navedeno može se pokrenuti preko SSH-a iz skripte, jer DDOS
prihvata komandu kao argument SSH poziva:

```bash
ssh sysadmin@<dd-host> "filesys show space" 
ssh sysadmin@<dd-host> "replication status"
```

Za neinteraktivno izvršavanje postavite SSH ključ (`adminaccess add ssh-keys`)
i koristite nalog sa `user` ili `limited-admin` rolom — read-only provere
ne traže punu `admin` rolu.
