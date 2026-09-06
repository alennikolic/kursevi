# 02 — Dnevni health check

Redosled provere kada se ujutru logujete na uređaj ili kada vas neko pita
"da li je DD u redu". Sve komande u ovom poglavlju su **read-only** —
nijedna ne menja konfiguraciju i sve se mogu pustiti u produkciji u bilo koje doba.

---

## 1. Orijentacija — gde sam

```
system show version              # DDOS verzija i build
system show detailed-version     # verzije pojedinačnih komponenti
system show modelno              # model uređaja
system show serialno             # serijski broj (za Dell support case)
system show uptime
```

Ako radite na više uređaja u seriji, prvo ovo — sprečava da se komanda odradi
na pogrešnoj kutiji.

---

## 2. Da li nešto trenutno gori

```
alerts show current                          # aktivni alerti
alerts show current-detailed                 # sa detaljima i alert ID-jevima
alerts show history last 7 days
alerts show history-detailed last 24 hours
alerts show daily
```

> Ne postoji `alerts show current-hardware`. Hardverski alerti se vide u
> `alerts show current` kao i svi ostali, filtrira se po klasi.

**Kako čitati:** svaki alert ima klasu, severity i timestamp. `CRITICAL` i
`EMERGENCY` znače da se otvara case. `WARNING` na temu kapaciteta je najčešći
i vodi u poglavlje 04.

Ko dobija obaveštenja:

```
alerts notify-list show
autosupport show all                 # da li AutoSupport ide ka Dell-u
```

---

## 3. Osnovni status sistema

```
system status                    # sažet status
system show hardware             # inventar hardvera
system show all                  # sve zajedno (dugačak output)
system show ports
```

Hardver detaljnije:

```
disk show state                  # mapa diskova: in-use / spare / failed / unknown
disk show hardware               # model, firmware, serijski broj po disku
disk show reliability-data       # SMART-like podaci
enclosure show all               # police / šasije
enclosure show fans              # ventilatori
enclosure show cpus
enclosure show memory
enclosure show misconfiguration
```

> Ne postoji `system show environment`. Temperature, ventilatori i slično
> su pod `enclosure show ...`.

**Šta tražiti:** bilo koji disk u stanju `failed`, `absent` ili `foreign`.
Jedan `spare` koji je preuzeo mesto `failed` diska znači da rekonstrukcija
ide ili je gotova.

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
| `/data: pre-comp` | Logička veličina podataka pre dedupe/kompresije | Raste najbrže, nije alarm sam po sebi |
| `/data: post-comp` | Stvarno zauzeće na disku | **Ovo je red koji se prati.** Vidi pragove ispod |
| `/ddvar` | Sistemska particija: logovi, support bundle-ovi, upgrade paketi | Ako se popuni, GUI i logovanje otkazuju |
| `/ddvar/core` | Core dump-ovi | Ako raste — nešto je crash-ovalo |

Pragovi za zauzeće su **podesivi**, ne fiksni:

```
filesys option show                          # trenutne vrednosti
```

Podrazumevano se koriste `warning-space-usage` (opseg 50–90) i
`critical-space-usage` (opseg 75–98). Proverite kako su postavljeni na vašem
uređaju pre nego što zaključite da je 85% "normalno" ili "alarmantno".
Podešavanje → **poglavlje 04**.

Dodatno:

```
filesys show space tier active
filesys show space tier cloud        # samo ako je Cloud Tier licenciran
filesys show uptime
filesys clean status                 # da li cleaning ide i dokle je stigao
filesys clean show config
filesys clean show schedule
```

---

## 5. MTree pregled

```
mtree list                                       # svi MTree-jevi, status, RL, kvote
mtree show compression                           # zauzeće po MTree-ju
mtree show compression /data/col1/<mtree> last 24 hours
mtree show performance /data/col1/<mtree>
quota capacity show all                          # kvote i popunjenost
quota capacity status                            # da li je kvota mehanizam aktivan
```

> Ne postoji `mtree show stats`. Postoje samo `mtree show compression`
> i `mtree show performance`.

`mtree list` u jednoj tabeli pokazuje da li je MTree `RW`, `RO` ili `RD`
(replication destination), i da li ima uključen Retention Lock. Najbrži način
da vidite koji su MTree-jevi zaključani, a koji su odredište replikacije.

Detaljno → **poglavlje 05**, Retention Lock → **poglavlje 06**.

---

## 6. Replikacija

```
replication status                       # sažeto stanje svih konteksta
replication status all detailed          # detaljno, po kontekstu
replication show config
replication show performance             # trenutna brzina
replication show stats
replication show detailed-stats
replication show history
```

> Ne postoji `replication show detailed-status`. Detaljan prikaz je
> `replication status [<destination> | all] detailed`, a statistika je
> `replication show detailed-stats`.

**Šta se gleda u `replication status`:**

| Kolona | Značenje |
|---|---|
| `State` | `initializing`, `normal`, `disabled`, `disconnected`, `error`, `uninitialized` |
| `Sync'ed-as-of-time` | Do kog trenutka je odredište konzistentno sa izvorom. **Ovo je stvarni RPO.** |
| `Pre-comp Remaining` | Koliko logičkih podataka još treba preneti — mera zaostajanja |

Ako `Sync'ed-as-of-time` zaostaje više od dogovorenog RPO-a → **poglavlje 07**,
playbook PB-03 u **poglavlju 10**.

Praćenje uživo:

```
replication watch <destination>
replication throttle show all            # da nije neko ostavio throttle
```

---

## 7. Snapshot-ovi

```
snapshot list mtree /data/col1/<mtree>
snapshot list all
snapshot schedule show
```

Traže se snapshot-ovi kojima je isteklo vreme a nisu obrisani, i rasporedi
koji su prestali da rade. Detaljno → **poglavlje 05**.

---

## 8. Mreža i opterećenje

```
net show settings                # IP adrese, MTU, stanje interfejsa
net show config [<ifname>]
net show hardware                # fizički portovi, brzina, link status
net show stats                   # statistike, greške, drop-ovi
net show stats interfaces
net aggregate show [detailed]
net failover show
net route show tables
net route show gateways
```

Trenutno opterećenje u realnom vremenu:

```
system show stats interval 2                     # osvežava se na 2 sekunde
system show stats view sysstat interval 2
system show performance duration 30 min          # throughput za prethodnih 30 min
```

**Šta tražiti:** rastući `errors`, `dropped` ili `collisions` na interfejsu,
i link koji je pao na nižu brzinu nego što bi trebalo. Detaljno → **poglavlje 08**.

---

## 9. Aktivne konekcije klijenata

```
ddboost status
ddboost show connections [detailed]      # aktivne DD Boost sesije
ddboost storage-unit show                # storage unit-ovi i zauzeće
nfs status
nfs show active                          # aktivni NFS klijenti
nfs export show                          # definisani export-i
cifs status
cifs show active                         # aktivne CIFS/SMB sesije
```

Detaljno → **poglavlje 09**.

---

## 10. Logovi ako nešto ne štima

```
log list                         # koji log fajlovi postoje
log view [<filename>]            # tekući messages log
log view messages.engineering
log watch [<filename>]           # prati log uživo, kao tail -f
log view access-info             # pokušaji prijave
log view audit-info              # autorizacione greške
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
replication throttle show all
net show hardware
```

Ovo je oko 30 sekundi rada i pokriva sve što se najčešće pita.
Ako je bilo koja komanda vratila nešto neočekivano, otvorite dublje poglavlje.

---

## Automatizacija

Sve gore navedeno može se pokrenuti preko SSH-a iz skripte:

```bash
for c in "filesys show space" "replication status" "alerts show current"; do
    echo "===== $c ====="
    ssh sysadmin@dd01 "$c"
done
```

Za neinteraktivno izvršavanje postavite SSH ključ (`adminaccess add ssh-keys`)
i koristite nalog sa rolom `user` — read-only provere ne traže `admin`.
