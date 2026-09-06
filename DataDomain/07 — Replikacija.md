# 07 — Replikacija

Replikacija prenosi već dedupliciran i komprimovan podatak sa izvornog na
odredišni Data Domain uređaj. Prenosi se samo ono što odredište nema.

---

## 7.1 Tipovi

| Tip | Šta replicira | Kada |
|---|---|---|
| **MTree replication** | Jedan MTree | **Podrazumevani izbor.** Granularno, podržava Retention Lock i sve topologije |
| **Collection replication** | Ceo sistem, 1:1 | Odredište je potpuna kopija izvora, u celosti read-only |
| **Directory replication** | Jedan direktorijum | Legacy. Ne koristiti za nove instalacije |
| **Managed File Replication (MFR)** | Pojedinačne fajlove, po nalogu aplikacije | DD Boost. Kontroliše backup aplikacija |

Topologije za MTree replikaciju: one-to-one, many-to-one (fan-in),
one-to-many (fan-out), bi-directional, cascaded. Broj konteksta je ograničen modelom.

---

## 7.2 Provera stanja (read-only)

```
replication status [<destination> | tenant-unit <tu> | all] [detailed]
replication show config [<destination> | tenant-unit <tu> | all]
replication show stats [<destination> | all]
replication show detailed-stats [<destination> | all]
replication show performance {<obj-spec> | all} [interval <sec>]
replication show history {<obj-spec> | all} [duration <hr>]
replication show detailed-history {<obj-spec> | all} [duration <hr>]
replication throttle show [<destination> | default | all]
replication throttle show performance [<destination> | default | all] [interval <sec>]
replication option show
replication schedule show
replication watch <destination>
```

> **`replication show detailed-status` ne postoji.** Detaljan prikaz stanja je
> `replication status <destination> detailed` ili `replication status all detailed`.
> Detaljna statistika je `replication show detailed-stats`.

### Kako se čita `replication status`

| Kolona | Značenje | Na šta paziti |
|---|---|---|
| `State` | `initializing`, `normal`, `disabled`, `disconnected`, `error`, `uninitialized` | Sve osim `normal` i `initializing` je za istragu |
| `Sync'ed-as-of-time` | Trenutak do kog je odredište konzistentno sa izvorom | **Ovo je stvarni RPO** |
| `Pre-comp Remaining` | Koliko logičkih podataka čeka prenos | Stalan rast = replikacija ne stiže |

`Sync'ed-as-of-time` je najvažniji broj u ovom poglavlju.

---

## 7.3 Kreiranje MTree replikacije

⚠️ Kontekst se definiše **na oba uređaja istom komandom**, pa se inicijalizuje
samo na izvoru.

```
# 1. Na IZVORU i na ODREDIŠTU (identična komanda):
replication add \
    source mtree://<izvorni-host>/data/col1/<mtree> \
    destination mtree://<odredisni-host>/data/col1/<mtree> \
    [low-bw-optim {enabled | disabled}]

# 2. Samo na IZVORU:
replication initialize mtree://<odredisni-host>/data/col1/<mtree>

# 3. Praćenje:
replication status
replication watch mtree://<odredisni-host>/data/col1/<mtree>
```

**Preduslovi:**
- Replication licenca na oba uređaja (`elicense show`)
- Izvorni MTree postoji; **odredišni se ne kreira ručno** — nastaje sam
- Ime hosta razrešivo sa obe strane (`net show dns`, `net hosts show`)
- Otvoren mrežni put (vidi 7.8)
- Kompatibilne DDOS verzije — odredište ista ili novija verzija od izvora

Nakon inicijalizacije odredišni MTree je u `RD` stanju i read-only.

---

## 7.4 Upravljanje kontekstom

```
replication disable {<destination> | all}        # ⚠️ pauza, kontekst ostaje
replication enable {<destination> | all}         # ⚠️ nastavlja odakle je stao
replication break {<destination> | all}          # 🛑 raskida kontekst (na OBA uređaja)
replication resync <destination>                 # 🛑 ponovna sinhronizacija
replication recover <destination>                # 🛑 oporavak izvora sa odredišta
replication abort {recover | resync} <destination>   # ⚠️
replication reauth <destination>                 # ⚠️ ponovna autentikacija para
replication sync [and-verify] [<destination>]    # ⚠️ prisilna sinhronizacija
```

> **`replication del` ne postoji.** Raskidanje konteksta je `replication break`.

### `disable` vs `break` — razlika koja se skupo plaća

- **`disable`** — privremena pauza. Kontekst i njegovo znanje ostaju. Nakon
  `enable` nastavlja inkrementalno, prenosi samo razliku. Koristi se za
  planirane radove, migracije linkova, održavanje.

- **`break`** — trajno raskidanje. Mora se izvršiti na **oba** uređaja.
  Odredišni MTree postaje `RW`. Za ponovno uspostavljanje treba
  `replication add` + `resync` ili puna reinicijalizacija.

> ⛔ **Nikada ne radite `break` da biste "restartovali" replikaciju koja zaostaje.**
> Za to postoji `disable` / `enable`. `break` na velikom MTree-ju znači
> danima ponovne sinhronizacije preko WAN-a.

### `resync` i `recover`

- **`resync`** — kada su izvor i odredište razišli (nakon `break`, nakon DR
  testa, ili nakon pisanja u odredišni MTree). Poredi obe strane i prenosi razliku.
  Dostupno za Directory, MTree i Collection replikaciju.
- **`recover`** — oporavak izvora sa odredišta nakon gubitka izvornog sistema.
  **Ako je prethodno urađen `replication break`, odredište se ne može koristiti
  za oporavak izvora.**

Prekid oba: `replication abort {recover | resync} <destination>`.

---

## 7.5 Modifikacija konteksta

⚠️ Po pravilu traži da je kontekst prethodno `disable`-ovan.

```
replication modify <destination> {source-host | destination-host} <novo-ime>
replication modify <destination> connection-host <novo-ime> [port <port>]
replication modify <destination> low-bw-optim {enabled | disabled}
replication modify <destination> encryption {enabled | disabled}
replication modify <destination> max-repl-streams <n>
replication modify <destination> ipversion {ipv4 | ipv6}
replication modify <destination> destination-tenant-unit <tu>
replication modify <destination> crepl-gc-bw-optim {enabled | disabled}
```

| Opcija | Kada |
|---|---|
| `low-bw-optim` | Spora WAN veza. Dodatna deduplikacija na uštrb CPU-a. **Ne koristiti na brzim linkovima** |
| `encryption` | Enkripcija replikacionog saobraćaja preko nezaštićene mreže |
| `max-repl-streams` | Ograničenje broja tokova po kontekstu — korisno kod fan-in topologija |
| `crepl-gc-bw-optim` | Optimizacija propusnog opsega za GC kod Collection replikacije |

Primer:

```
replication modify mtree://ip2/data/col1/mtr1 max-repl-streams 6
```

Globalne opcije:

```
replication option show
replication option set bandwidth <rate>                          # ⚠️
replication option set delay <vrednost>                          # ⚠️
replication option set listen-port <vrednost>                    # ⚠️
replication option set default-sync-alert-threshold <vrednost>   # ⚠️
replication option reset {bandwidth | delay | listen-port | default-sync-alert-threshold}
```

> `default-sync-alert-threshold` je prag posle kojeg DD sam podiže alert
> ako `Sync'ed-as-of-time` previše zaostane. **Postavite ga na dogovoreni RPO** —
> bolje nego da zaostajanje otkrijete ručnom proverom.

---

## 7.6 Throttle — ograničavanje propusnog opsega

```
replication throttle show [<destination> | default | all]
```

⚠️

```
replication throttle add [destination <host> | default] <sched-spec> <rate>
replication throttle del [destination <host> | default] <sched-spec>
replication throttle set current [destination <host> | default] <rate>
replication throttle set override [destination <host> | default] <rate>
replication throttle reset [destination <host> | default] {current | override | schedule | all}
```

`<sched-spec>` su dani i vreme u formatu `<dan> [<dan>...] <hhmm>`.

Primer iz dokumentacije — ograniči na 5 Mbps utorkom i petkom od 22:00:

```
replication throttle add destination ddr1-ny tue fri 2200 5Mbps
replication throttle del mon 1300
```

| Vrednost rate | Značenje |
|---|---|
| `unlimited` | Bez ograničenja |
| `0` | **Potpuno zaustavlja replikaciju** u tom terminu |
| `50Mbps` | Ograničenje na navedenu brzinu |

**Razlika `current` i `override`:**
- `set current` — traje do sledeće zakazane promene ili do reboot-a.
  Ne može se postaviti dok je `override` aktivan.
- `set override` — traje dok se ne izda drugi override. Ne može se postaviti
  dok je `current` aktivan.

> ⚠️ **Zamka:** throttle `0` postavljen "privremeno" i zaboravljen je jedan od
> najčešćih uzroka zaostajanja replikacije. Kada `Sync'ed-as-of-time` zaostaje,
> `replication throttle show all` je među prve tri komande.

---

## 7.7 Vremenski prozor za replikaciju

Umesto throttle-a, replikacija se može potpuno vezati za vremenski prozor:

```
replication schedule show
replication schedule set <destination> enable <hhmm> disable <hhmm>   # ⚠️
replication schedule reset <destination>                             # ⚠️
```

Korisno kada replikacija sme da radi samo van radnog vremena.

---

## 7.8 Mreža i portovi

```
net show settings
net show hardware
net ping <parnjak>
net show stats
net route show tables
replication option show                     # listen port
```

> **Lista portova se menjala kroz verzije.** Pre nego što se javite mrežnom timu,
> potvrdite je iz Dell KB-a o firewall zahtevima:
> <https://www.dell.com/support/kbdoc/en-us/000004184/1245-port-requirements-for-allowing-access-to-data-domain-system-through-a-firewall>
> ili iz Security Configuration Guide-a za vašu verziju. Trenutni listen port
> vidi se u `replication option show`.

Odvajanje replikacionog saobraćaja na zaseban interfejs:

```
net hosts show
net hosts add <ime-parnjaka> <ip-replikacione-mreze>     # ⚠️
```

Merenje propusnosti linka:

```
net iperf server [run] [ipversion {ipv4 | ipv6}] [bind <ipaddr>] [iperf-version {v2 | v3}]
net iperf client {<ipaddr> | <hostname>} [iperf-version {v2 | v3}]
```

Detalji → **poglavlje 08**.

---

## 7.9 Replikacija i Retention Lock

- MTree replikacija **prenosi lock status fajlova**
- Odredište mora imati odgovarajuću RL licencu. **Compliance MTree se ne može
  replicirati na uređaj bez uključenog Compliance režima**
- Ako je odredište deo Cyber Recovery vault-a, važe posebna pravila —
  **poglavlje 12**

```
mtree list
mtree retention-lock status mtree /data/col1/<mtree>
elicense show
```

---

## 7.10 Replikacija i kapacitet

Zaostala replikacija drži podatke na izvoru — cleaning ih ne može osloboditi
dok nisu preneti.

```
replication status
filesys show space
filesys clean status
```

Vidi **poglavlje 04**.

---

## 7.11 Planirani radovi — procedura

```
# Pre radova, na izvoru:
replication status                                   # zabeležiti stanje
replication throttle show all                        # zabeležiti throttle
replication disable <destination>                    # ⚠️

# Nakon radova:
replication enable <destination>                     # ⚠️
replication status
replication watch <destination>
```

Ne koristiti `break`. Kontekst se ne dira.

Ako je uređaj deo CR vault-a — **ne dirati kontekste uopšte**, vidi **poglavlje 12**.

---

## 7.12 Checklist za novi kontekst

- [ ] Replication licenca na oba uređaja (`elicense show`)
- [ ] DDOS verzije kompatibilne (odredište isto ili novije), provereno u E-Lab Navigator-u
- [ ] DNS/hosts razrešavanje radi sa obe strane
- [ ] Mrežni put otvoren i izmeren (`net ping`, `net iperf`)
- [ ] Na odredištu ima dovoljno prostora za pun MTree (`filesys show space`)
- [ ] Ako izvor ima Retention Lock — odredište ima odgovarajuću licencu i režim
- [ ] Definisan throttle raspored ili vremenski prozor ako je link deljen
- [ ] `replication option set default-sync-alert-threshold` postavljen na dogovoreni RPO
- [ ] Kontekst dodat u monitoring

---

## Reference

- DD OS 8.6 Command Reference Guide — poglavlje `replication`
- DD OS 8.6 Administration Guide — replikacija
- Dell KB 000004184 — port requirements
