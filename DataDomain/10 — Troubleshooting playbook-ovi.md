# 10 — Troubleshooting playbook-ovi

Gotovi scenariji. Svaki ima isti oblik: simptom, brza dijagnostika, najčešći
uzroci po učestalosti, akcije, i kada se eskalira.

**Pre svakog playbook-a:**

```
system show version
alerts show current
filesys status
```

**Pre otvaranja case-a kod Dell-a uvek:**

```
system show serialno
support bundle create default            # ⚠️ ili: support bundle create mini
support bundle list
```

Slanje ide kroz konfigurisanu SupportAssist vezu (`support connectivity config show`),
preko GUI-ja ili ručno na case. Vidi **poglavlje 03**.

---

## PB-01 — File system iznad praga

**Simptom:** alert o kapacitetu, backup pada zbog nedostatka prostora.

```
filesys show space
filesys option show                      # koji su stvarni pragovi na ovom uređaju
filesys clean status
mtree show compression
snapshot list mtree /data/col1/<najveci-mtree>
mtree retention-lock status mtree /data/col1/<najveci-mtree>
replication status
```

| # | Uzrok | Provera | Akcija |
|---|---|---|---|
| 1 | Cleaning nije radio ili nije završio | `filesys clean status` | `filesys clean start` ⚠️, `filesys clean set throttle 100` ⚠️ |
| 2 | Snapshot-ovi drže stare podatke | `snapshot list mtree ...` | `snapshot expire ...` ⚠️ |
| 3 | Retention Lock — zaključani podaci | `mtree retention-lock report generate retention-details mtrees all` | **Prostor se ne može osloboditi.** Proširenje ili revizija retencije |
| 4 | Indefinite retention hold aktivan | `mtree retention-lock status mtree ...` | Proveriti da li je hold još potreban (**poglavlje 06**) |
| 5 | Backup aplikacija drži retenciju | Katalog aplikacije | Razgovor sa backup timom |
| 6 | Replikacija zaostaje i drži podatke | `replication status` | PB-03 |
| 7 | Obrisan MTree čeka cleaning | `mtree list` (status `D`) | Sačekati cleaning |
| 8 | CR PIT kopije | `crcli policy list-copy` | PB-15, **poglavlje 12** |

**Eskalacija:** iznad 95% i cleaning ne oslobađa ništa — case kod Dell-a.

Detaljno → **poglavlje 04**.

---

## PB-02 — Cleaning traje predugo ili ne završava

```
filesys clean status
filesys clean show config
filesys clean show schedule
filesys clean show throttle
filesys show space
system show stats interval 2
log view space.log
```

| # | Uzrok | Akcija |
|---|---|---|
| 1 | Throttle prenizak | `filesys clean set throttle 100` ⚠️ |
| 2 | Sistem pretrpan (backup + replikacija + cleaning) | Pomeriti raspored: `filesys clean set schedule` ⚠️ |
| 3 | File system skoro pun — cleaning-u treba prostora | PB-01 |
| 4 | Vrlo veliki FS, normalno traje >24h | Sačekati, pratiti napredak |
| 5 | Cleaning je prekidan pa kreće ispočetka | Ne prekidati; ostaviti da završi ciklus |
| 6 | DDOS je preskočio ciklus zbog anomalije u obrisanim podacima | Proveriti alerte pre ručnog pokretanja |
| 7 | Ručno pokretanje traži autorizaciju security officera | Uključena bezbednosna politika — pozvati SO |

**Eskalacija:** ciklus koji ne napreduje (isti procenat više sati) — case.

---

## PB-03 — Replikacija zaostaje

**Simptom:** `Sync'ed-as-of-time` stariji od RPO-a, `Pre-comp Remaining` raste.

```
replication status all detailed
replication show performance all interval 5
replication throttle show all
replication schedule show
net show hardware
net show stats interfaces
filesys show space
```

| # | Uzrok | Provera | Akcija |
|---|---|---|---|
| 1 | **Throttle zaboravljen na niskoj vrednosti ili na `0`** | `replication throttle show all` | `replication throttle reset <dest> current` ⚠️ |
| 2 | Vremenski prozor previše uzak | `replication schedule show` | `replication schedule set ...` ⚠️ |
| 3 | Nedovoljan propusni opseg | `net iperf` | Mrežni tim; razmotriti `low-bw-optim` na sporim linkovima |
| 4 | Skok u količini podataka na izvoru | `filesys show compression last 7 days` | Sačekati da nadoknadi ili povećati kapacitet linka |
| 5 | Odredište je puno | `filesys show space` na odredištu | PB-01 na odredištu |
| 6 | Mrežne greške | `net show stats interfaces` | PB-08 |
| 7 | Fan-in: odredište opterećeno | `replication status` na odredištu | `replication modify <dest> max-repl-streams <n>` ⚠️, razmaknuti rasporede |

> ⛔ **Ne radite `replication break` da biste "restartovali" replikaciju.**
> Za pauzu postoji `replication disable` / `enable`.

Detaljno → **poglavlje 07**.

---

## PB-04 — Replikacija u stanju `disconnected` ili `error`

```
replication status all detailed
replication show config
net ping <parnjak>
net lookup <parnjak>
net hosts show
net route show tables
replication option show
log view
```

| # | Uzrok | Akcija |
|---|---|---|
| 1 | Mrežni put zatvoren | Provera listen porta (`replication option show`) i rute; mrežni tim |
| 2 | DNS ne razrešava ime parnjaka | `net lookup`; `net hosts add` ⚠️ |
| 3 | Parnjak ugašen ili u održavanju | `net ping`, kontakt sa drugom lokacijom |
| 4 | Nekompatibilne DDOS verzije nakon nadogradnje | `system show version` na obe strane; E-Lab Navigator |
| 5 | Kontekst ručno isključen | `replication show config`, `replication enable` ⚠️ |
| 6 | Problem sa autentikacijom para | `replication reauth <destination>` ⚠️ |
| 7 | **CR vault okruženje** | ⛔ Kontekst je namerno `disabled`. **Ne uključivati ručno.** PB-14, **poglavlje 12** |

**Eskalacija:** mreža potvrđeno u redu a kontekst i dalje `error` — case sa
bundle-om sa obe strane.

---

## PB-05 — Backup pada sa "no space" a `filesys show space` pokazuje slobodno

```
quota capacity show all
quota capacity status
ddboost storage-unit show
mtree list
filesys show space
```

| # | Uzrok | Akcija |
|---|---|---|
| 1 | **Dostignut hard limit kvote** | `quota capacity show all`; podići limit ⚠️ ili osloboditi prostor |
| 2 | Kvota na DD Boost storage unit-u | `ddboost storage-unit show` |
| 3 | MTree je u `RD` stanju (odredište replikacije) | `mtree list` — u njega se **ne sme** pisati |
| 4 | Retention Lock sprečava prepisivanje fajla | `mtree retention-lock status mtree ...` |
| 5 | `/ddvar` pun, ne `/data` | PB-07 |
| 6 | Stream kvota na storage unit-u | `quota streams show all` |

> Najčešći "lažni" problem sa kapacitetom. Prva komanda je uvek
> `quota capacity show all`, ne `filesys show space`.

---

## PB-06 — MTree se ne može obrisati

```
mtree list
replication show config
mtree retention-lock status mtree /data/col1/<mtree>
snapshot list mtree /data/col1/<mtree>
nfs export show
cifs share show
ddboost storage-unit show
```

| # | Uzrok | Akcija |
|---|---|---|
| 1 | MTree sadrži zaključane fajlove | Ne može se obrisati do isteka lock-a. U Governance režimu moguć `revert` ⚠️ |
| 2 | Indefinite retention hold aktivan | `mtree retention-lock indefinite-retention-hold disable` ⚠️ (uz odobrenje) |
| 3 | MTree je deo replikacionog konteksta | Prvo `replication break` 🛑 na obe strane |
| 4 | Postoji DD Boost storage unit nad njim | `ddboost storage-unit delete` 🛑 |
| 5 | Aktivni NFS export / CIFS share | `nfs export destroy` / `cifs share destroy` ⚠️ |
| 6 | **CR vault PIT kopija (`cr-copy-*`)** | ⛔ Brisati kroz `crcli policy delete-copy`, ne kroz DD. **Poglavlje 12** |

---

## PB-07 — `/ddvar` pun

**Simptom:** GUI ne radi, logovanje otkazuje, `support bundle create` pada.
Podaci nisu ugroženi.

```
filesys show space
support bundle list
system upgrade package list
log list
```

⚠️

```
support bundle delete {<lista> | all}
system upgrade package delete <ime>
```

Ako `/ddvar/core` nije prazan — nešto je crash-ovalo. Case sa bundle-om.

---

## PB-08 — Backup je spor

Puni redosled u **poglavlju 08, sekcija 8.10**. Skraćeno:

```
net show hardware                        # 1. link speed, duplex
net show stats interfaces                # 2. greške i drop-ovi
system show stats interval 2             # 3. je li DD opterećen
filesys clean status                     # 4. cleaning u backup prozoru
replication show performance all         # 5. troši li replikacija link
ddboost show connections detailed        # 6. broj tokova
system show performance custom-view streams
filesys show compression last 24 hours   # 7. pad dedupe faktora
filesys show space                       # 8. blizu popunjenosti
```

Ključna provera koja razdvaja "DD je spor" od "mreža je spora":

```
net iperf server                         # na DD-u, pa iperf klijent sa klijenta
net ping <klijent> count 5 packet-size 8972 path-mtu do
```

---

## PB-09 — Disk u stanju `failed`

```
disk show state
disk show hardware
disk show reliability-data
alerts show current-detailed
enclosure show all
```

Akcije:

1. Potvrditi da je spare preuzeo i da rekonstrukcija ide — `disk show state`
2. Zabeležiti poziciju `<enclosure>.<disk>`, model i serijski broj
3. `disk beacon <enclosure>.<disk>` — upaliti LED pre odlaska u data centar
4. Case: serijski broj sistema + support bundle
5. Ne vaditi disk pre nego što Dell potvrdi i pre nego što se rekonstrukcija završi

**Hitno:** više od jednog `failed` diska u istoj RAID grupi — case sa najvišim
prioritetom.

---

## PB-10 — NFS klijent ne može da mount-uje

Vidi **9.1**. Skraćeno:

```
nfs status
nfs export show
nfs show clients
mtree list                               # je li MTree u RD stanju
net ping <klijent>
log view
```

Najčešće: export ne pokriva IP klijenta, MTree je odredište replikacije,
firewall blokira 111/2049, ili je NFS verzija isključena (`nfs status`).

---

## PB-11 — CIFS share nedostupan

Vidi **9.2**. Skraćeno:

```
cifs status
cifs show config
system show date
ntp status
net show dns
cifs share show
cifs show active
```

**Najčešći uzrok je odstupanje vremena** — Kerberos ne tolerira više od
nekoliko minuta razlike prema domenskom kontroleru.

---

## PB-12 — DD Boost klijent se ne povezuje

Vidi **9.3**. Skraćeno:

```
ddboost status
ddboost show connections detailed
ddboost user show
ddboost storage-unit show
ddboost clients show
quota capacity show all
ifgroup show config all
```

Uz to: verzija DD Boost biblioteke naspram DDOS verzije — proveriti u
E-Lab Navigator-u nakon svake nadogradnje bilo koje strane.

---

## PB-13 — Nakon nadogradnje DDOS-a nešto ne radi

```
system show version
system upgrade history
alerts show current
elicense show
replication status
ddboost status
cifs status
nfs status
```

| # | Uzrok | Akcija |
|---|---|---|
| 1 | Nekompatibilna verzija klijentskog agenta | E-Lab Navigator; nadograditi agenta |
| 2 | Replikacija ka starijoj verziji odredišta | Odredište se nadograđuje **pre** izvora |
| 3 | Licenca nije prenesena | `elicense show`, `elicense update` ⚠️ |
| 4 | Izmenjeno podrazumevano ponašanje | Release Notes za ciljnu verziju |
| 5 | **CR vault:** CR verzija nije usklađena sa DDOS verzijom | Support Matrix; **poglavlje 12** |

---

## PB-14 — CR vault u stanju `Unlocked` bez aktivnog posla

**Simptom:** `crcli vault state` pokazuje `Unlocked`, a nema aktivnog sync posla.

```
crcli vault state
crcli jobs list
crcli alerts list
crcli policy list
# na vault DD-u:
replication status
alerts show current
```

| # | Uzrok | Akcija |
|---|---|---|
| 1 | Posao je pao a kontekst nije zatvoren | `crcli jobs list`, pregled poslednjeg posla |
| 2 | U toku inicijalna sinhronizacija nove politike | Normalno — port se zatvara po završetku |
| 3 | **Neko je ručno uključio replikacioni kontekst na DD-u** | ⛔ Vratiti kroz CR, ne kroz DD. Utvrditi ko i zašto |
| 4 | CR servis ne radi ispravno | `crcli system details`, stanje kontejnera na management hostu |

**Ako se sumnja na kompromitaciju:**

```
crcli vault secure                       # 🛑 zaustavlja sve poslove i sve naredne
```

Zatim eskalacija po internoj proceduri za bezbednosni incident.
`crcli vault release` može **samo Security Admin**. Detaljno → **poglavlje 12**.

---

## PB-15 — CR politika pada

```
crcli jobs list
crcli jobs show --jobname <ime>
crcli alerts list
crcli dd list
crcli policy show --policyname <ime>
crcli vault state
crcli system details
```

| # | Uzrok | Akcija |
|---|---|---|
| 1 | CR ne može da priča sa DD-om (izmenjena lozinka, mreža) | `crcli dd list`; `crcli dd modify` ⚠️ |
| 2 | Vault je u `Secured` stanju — svi poslovi padaju | `crcli vault state` |
| 3 | Nema prostora na vault DD-u | `filesys show space` na vault DD-u; 12.12 |
| 4 | Lock korak pada — nema RL licence na vault DD-u | `elicense show` na vault DD-u |
| 5 | Istovremene Sync ili Lock akcije za istu politiku | Sačekati završetak prve |
| 6 | CR verzija nije usklađena sa DDOS verzijom | Support Matrix |
| 7 | Ručno menjani replikacioni konteksti na DD-u | ⛔ Vidi 12.5 |
| 8 | Stare kopije nisu očišćene, baza narasla | `crcli system clean --show` (**poglavlje 12**) |

---

## PB-16 — CyberSense je prijavio nalaz

**Simptom:** `crcli policy list-copy` pokazuje `lastanalysisstatus` različit od `Good`.

> **Ovo nije tehnički problem koji se rešava komandom.** Ovo je bezbednosni
> događaj i ide po internoj proceduri za incident. Playbook služi da se
> prikupe činjenice, ne da se stvar "popravi".

```
crcli policy list-copy --policyname <ime> --copyname <kopija>
crcli policy analysis-report download --policyname <ime> ...
crcli alerts list
crcli vault state
```

Koraci:

1. **Ne brisati** kopiju koja je označena — ona je dokaz
2. Preuzeti izveštaj analize i proslediti bezbednosnom timu
3. Razmotriti `crcli vault secure` 🛑 dok se ne utvrdi obim
4. Identifikovati poslednju kopiju sa statusom `Good` — kandidat za oporavak
5. Ne pokretati oporavak ka produkciji dok produkcija nije očišćena
6. Eskalacija po internoj proceduri

> **Mora biti napisano i uvežbano pre incidenta.** Ako se procedura piše
> u trenutku kada CyberSense prijavi nalaz, kasno je.

---

## Kada se otvara case kod Dell-a

Bez odlaganja:
- Više od jednog `failed` diska u istoj RAID grupi
- File system u read-only stanju
- Cleaning koji ne napreduje na sistemu preko 95%
- Bilo koji `EMERGENCY` alert
- Sumnja na gubitak ili oštećenje podataka

Uz case uvek priložiti:

```
system show serialno
support bundle create default            # ⚠️
support bundle list
```

Za probleme sa replikacijom — bundle sa **obe** strane.
Za CR probleme — dodatno `crcli system details` i `crcli system bundle --create`
sa management hosta.

---

## Reference

- Poglavlja ovog priručnika: 04, 07, 08, 09, 12
- DD OS 8.6 Administration Guide — troubleshooting
- Dell Support Knowledge Base za konkretne poruke o grešci
