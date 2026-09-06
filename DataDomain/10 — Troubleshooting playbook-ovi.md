# 10 — Troubleshooting playbook-ovi

Gotovi scenariji. Svaki ima isti oblik: simptom, brza dijagnostika, najčešći
uzroci po učestalosti, akcije, i kada se eskalira ka Dell podršci.

**Pre svakog playbook-a:**

```
system show version
alerts show current
filesys status
```

**Pre otvaranja case-a kod Dell-a uvek:**

```
system show serialno
support bundle create                    # ⚠️
support upload <ime>
```

---

## PB-01 — File system iznad 90%

**Simptom:** alert o kapacitetu, backup pada sa greškom o nedostatku prostora,
ili preventivna provera pokazuje visok `post-comp`.

```
filesys show space
filesys clean status
mtree show compression
snapshot list mtree /data/col1/<najveci-mtree>
mtree retention-lock status mtree /data/col1/<najveci-mtree>
replication status
```

| # | Uzrok | Provera | Akcija |
|---|---|---|---|
| 1 | Cleaning nije radio ili nije završio | `filesys clean status` | `filesys clean start` ⚠️, `filesys clean set throttle 100` ⚠️ |
| 2 | Snapshot-ovi drže stare podatke | `snapshot list mtree ...` | Isteći nepotrebne: `snapshot expire ...` ⚠️ |
| 3 | Retention Lock — zaključani podaci | `mtree retention-lock status ...` | **Prostor se ne može osloboditi.** Proširenje ili revizija retencije |
| 4 | Backup aplikacija drži retenciju | Katalog aplikacije | Razgovor sa backup timom |
| 5 | Replikacija zaostaje i drži podatke | `replication status` | PB-03 |
| 6 | Obrisan MTree čeka cleaning | `mtree list` (status `D`) | Sačekati cleaning |
| 7 | Stvarni rast podataka | `filesys show compression daily last 30 days` | Planiranje kapaciteta |

**Eskalacija:** ako je iznad 95% i cleaning ne oslobađa ništa — case kod Dell-a.
File system preko 100% prelazi u read-only i oporavak je spor.

Detaljno → **poglavlje 04**.

---

## PB-02 — Cleaning traje predugo ili ne završava

**Simptom:** `filesys clean status` danima pokazuje isti ciklus, prostor se ne oslobađa.

```
filesys clean status
filesys clean show config
filesys show space
system show stats interval 2
log view space.log
```

| # | Uzrok | Akcija |
|---|---|---|
| 1 | Throttle prenizak | `filesys clean set throttle 100` ⚠️ |
| 2 | Sistem je pretrpan (backup + replikacija + cleaning istovremeno) | Pomeriti raspored: `filesys clean set schedule` ⚠️ |
| 3 | File system skoro pun — cleaning-u treba slobodnog prostora | PB-01, osloboditi šta se može |
| 4 | Vrlo veliki file system, normalno traje >24h | Sačekati, pratiti napredak |
| 5 | Cleaning je prekidan pa svaki put kreće ispočetka | Ne prekidati; ostaviti da završi jedan pun ciklus |

**Eskalacija:** ciklus koji ne napreduje (isti procenat više sati) — case.

---

## PB-03 — Replikacija zaostaje

**Simptom:** `Sync'ed-as-of-time` stariji od dogovorenog RPO-a, `Pre-comp Remaining` raste.

```
replication status
replication show detailed-status
replication show performance
replication throttle show
net show hardware
net show stats
filesys show space
```

| # | Uzrok | Provera | Akcija |
|---|---|---|---|
| 1 | **Throttle zaboravljen na niskoj vrednosti ili na `0`** | `replication throttle show` | `replication throttle reset current` ⚠️ |
| 2 | Nedovoljan propusni opseg linka | `net iperf` | Razgovor sa mrežnim timom; razmotriti `low-bw-optim` |
| 3 | Skok u količini podataka na izvoru | `filesys show compression last 7 days` | Očekivati da nadoknadi; ako ne — kapacitet linka |
| 4 | Odredište je puno | `filesys show space` na odredištu | PB-01 na odredištu |
| 5 | Mrežne greške | `net show stats` | Fizički sloj, PB-08 |
| 6 | Odredište opterećeno drugim kontekstima (fan-in) | `replication status` na odredištu | Razmaknuti rasporede, throttle po kontekstu |

> ⛔ **Ne radite `replication break` da biste "restartovali" replikaciju.**
> Za pauzu postoji `replication disable` / `enable`. `break` znači dane
> ponovne sinhronizacije.

Detaljno → **poglavlje 07**.

---

## PB-04 — Replikacija u stanju `disconnected` ili `error`

**Simptom:** kontekst nije u `normal` stanju.

```
replication status
replication show detailed-status
net ping <parnjak>
net lookup <parnjak>
net hosts show
log view
```

| # | Uzrok | Akcija |
|---|---|---|
| 1 | Mrežni put zatvoren (firewall, ruta) | Provera porta 2051 i rute; mrežni tim |
| 2 | DNS ne razrešava ime parnjaka | `net lookup`; dodati `net hosts add` ⚠️ |
| 3 | Parnjak ugašen ili u održavanju | `net ping`, kontakt sa drugom lokacijom |
| 4 | Nekompatibilne DDOS verzije nakon nadogradnje | `system show version` na obe strane; Support Matrix |
| 5 | Kontekst ručno isključen | `replication show config`, `replication enable` ⚠️ |
| 6 | **CR vault okruženje** | ⛔ Kontekst je namerno `disabled`. **Ne uključivati ručno.** Vidi **poglavlje 12** |

**Eskalacija:** ako je mreža potvrđeno u redu a kontekst i dalje u `error` —
case sa support bundle-om sa obe strane.

---

## PB-05 — Backup pada sa "no space" a `filesys show space` pokazuje slobodno

**Simptom:** uređaj ima mesta, ali konkretan backup pada.

```
quota capacity show
ddboost storage-unit show
mtree list
filesys show space
```

| # | Uzrok | Akcija |
|---|---|---|
| 1 | **Dostignut hard limit kvote na MTree-ju** | `quota capacity show`; podići limit ⚠️ ili osloboditi prostor |
| 2 | Kvota na DD Boost storage unit-u | `ddboost storage-unit show` |
| 3 | MTree je u `RD` stanju (odredište replikacije) | `mtree list` — u njega se **ne sme** pisati |
| 4 | Retention Lock sprečava prepisivanje postojećeg fajla | `mtree retention-lock status ...` |
| 5 | `/ddvar` pun (ne `/data`) | PB-07 |

> Ovo je najčešći "lažni" problem sa kapacitetom. Prva komanda je uvek
> `quota capacity show`, ne `filesys show space`.

---

## PB-06 — MTree se ne može obrisati

```
mtree list
replication show config
mtree retention-lock status mtree /data/col1/<mtree>
snapshot list mtree /data/col1/<mtree>
nfs show clients
cifs share show
ddboost storage-unit show
```

| # | Uzrok | Akcija |
|---|---|---|
| 1 | MTree sadrži zaključane fajlove | Ne može se obrisati do isteka lock-a. U Governance režimu moguć `revert` ⚠️ |
| 2 | MTree je deo replikacionog konteksta | Prvo `replication break` 🛑 na obe strane |
| 3 | Postoji DD Boost storage unit nad njim | `ddboost storage-unit delete` 🛑 |
| 4 | Aktivni NFS export / CIFS share | Ukloniti pristup |
| 5 | **CR vault PIT kopija (`cr-copy-*`)** | ⛔ Brisati kroz `crcli policy delete-copy`, ne kroz DD. **Poglavlje 12** |

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

Akcije ⚠️:

```
support bundle delete <ime>              # stari bundle-ovi
system upgrade package delete <ime>      # preuzeti upgrade paketi
```

Ako i dalje raste, a `/ddvar/core` nije prazan — nešto je crash-ovalo.
Otvoriti case i priložiti bundle.

---

## PB-08 — Backup je spor

Puni redosled eliminacije je u **poglavlju 08, sekcija 8.9**. Skraćeno:

```
net show hardware                        # 1. fizički sloj: speed, duplex, link
net show stats                           # 2. greške i drop-ovi
system show stats interval 2             # 3. je li DD uopšte opterećen
filesys clean status                     # 4. radi li cleaning u backup prozoru
replication show performance             # 5. troši li replikacija link
ddboost show connections                 # 6. broj tokova
filesys show compression last 24 hours   # 7. pad dedupe faktora
filesys show space                       # 8. blizu popunjenosti
```

Ključna provera koja razdvaja "DD je spor" od "mreža je spora":

```
net iperf server                         # na DD-u, pa iperf klijent sa klijenta
```

---

## PB-09 — Disk u stanju `failed`

```
disk show state
disk show hardware
disk show reliability-data
alerts show current-hardware
enclosure show all
```

Akcije:

1. Potvrditi da je spare preuzeo mesto i da rekonstrukcija ide — `disk show state`
2. Zabeležiti poziciju: `<enclosure>.<disk>`, model i serijski broj
3. `disk beacon <enclosure>.<disk>` — upaliti LED pre odlaska u data centar
4. Otvoriti case: serijski broj sistema + support bundle
5. Ne vaditi disk pre nego što Dell potvrdi i pre nego što se rekonstrukcija završi

**Hitno:** više od jednog `failed` diska u istoj RAID grupi — case sa najvišim
prioritetom, ne čekati.

---

## PB-10 — NFS klijent ne može da mount-uje

Vidi **poglavlje 09, sekcija 9.1**. Skraćeno:

```
nfs status
nfs show clients
mtree list                               # je li MTree u RD stanju
net ping <klijent>
log view
```

Najčešće: export ne pokriva IP klijenta, MTree je odredište replikacije,
ili firewall blokira portove 111 / 2049.

---

## PB-11 — CIFS share nedostupan

Vidi **poglavlje 09, sekcija 9.2**. Skraćeno:

```
cifs status
cifs show config
system show date
ntp status
net show dns
cifs share show
```

**Najčešći uzrok je odstupanje vremena** — Kerberos ne tolerira više od
nekoliko minuta razlike prema domenskom kontroleru.

---

## PB-12 — DD Boost klijent se ne povezuje

Vidi **poglavlje 09, sekcija 9.3**. Skraćeno:

```
ddboost status
ddboost show connections
ddboost user show
ddboost storage-unit show
quota capacity show
ifgroup status
```

Uz to: verzija DD Boost biblioteke na klijentu naspram DDOS verzije —
proveriti u Support Matrix-u nakon svake nadogradnje bilo koje strane.

---

## PB-13 — Nakon nadogradnje DDOS-a nešto ne radi

```
system show version
system upgrade history
alerts show current
license show
replication status
ddboost status
cifs status
nfs status
```

| # | Uzrok | Akcija |
|---|---|---|
| 1 | Nekompatibilna verzija klijentskog agenta | Support Matrix; nadograditi agenta |
| 2 | Replikacija ka starijoj verziji odredišta | Odredište se nadograđuje **pre** izvora |
| 3 | Licenca nije prenesena | `license show`, `elicense update` ⚠️ |
| 4 | Izmenjeno podrazumevano ponašanje u novoj verziji | Release Notes za ciljnu verziju |
| 5 | **CR vault:** CR verzija nije usklađena sa DDOS verzijom | Support Matrix; **poglavlje 12** |

---

## PB-14 — CR vault u stanju `Unlocked` bez aktivnog posla

**Simptom:** `crcli vault state` pokazuje `Unlocked`, a nema aktivnog sync posla.
Veza ka produkciji je otvorena kada ne bi trebalo da bude.

```
crcli vault state
crcli jobs list -t protection -running
crcli alerts list
crcli policy list
# na vault DD-u:
replication status
alerts show current
```

| # | Uzrok | Akcija |
|---|---|---|
| 1 | Posao je pao a kontekst nije zatvoren | `crcli jobs list`, pregled poslednjeg posla |
| 2 | U toku je inicijalna sinhronizacija nove politike | Normalno — port se zatvara po završetku |
| 3 | **Neko je ručno uključio replikacioni kontekst na DD-u** | ⛔ Vratiti kroz CR, ne kroz DD. Utvrditi ko i zašto |
| 4 | CR servis ne radi ispravno | `crcli system details`, stanje kontejnera na management hostu |

**Ako se sumnja na kompromitaciju:**

```
crcli vault secure                       # 🛑 zaustavlja sve sync operacije
```

Zatim eskalacija po internoj proceduri za bezbednosni incident.
`crcli vault release` može samo security officer i tek kada se potvrdi
da pretnja ne postoji. Detaljno → **poglavlje 12**.

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
| 2 | Vault je u `Secured` stanju — Sync akcije su blokirane | `crcli vault state`; namerno ili ne |
| 3 | Nema prostora na vault DD-u | `filesys show space` na vault DD-u; poglavlje 12.12 |
| 4 | Lock korak pada — nema RL licence na vault DD-u | `license show` na vault DD-u |
| 5 | Istovremene Sync ili Lock akcije za istu politiku | Sačekati završetak prve |
| 6 | CR verzija nije usklađena sa DDOS verzijom nakon nadogradnje | Support Matrix |
| 7 | Ručno menjani replikacioni konteksti na DD-u | ⛔ Vidi 12.5 |

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
4. Identifikovati poslednju kopiju sa statusom `Good` — to je kandidat za oporavak
5. Ne pokretati oporavak ka produkciji dok produkcija nije očišćena
6. Eskalacija po internoj proceduri

> **Ovo mora biti napisano i uvežbano pre incidenta.** Ako se procedura piše
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
support bundle create                    # ⚠️
support upload <ime>
```

Za probleme sa replikacijom — bundle sa **obe** strane.
Za CR probleme — dodatno `crcli system details` sa management hosta.

---

## Reference

- Odgovarajuća poglavlja ovog priručnika (04, 07, 08, 09, 12)
- DDOS Administration Guide — poglavlja o troubleshooting-u
- Dell Support Knowledge Base za konkretne poruke o grešci
