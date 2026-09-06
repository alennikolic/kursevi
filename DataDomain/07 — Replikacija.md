# 07 — Replikacija

Replikacija prenosi već dedupliciran i komprimovan podatak sa izvornog na
odredišni Data Domain uređaj. Prenosi se samo ono što odredište još nema,
zbog čega je mrežni saobraćaj drastično manji od logičke količine podataka.

---

## 7.1 Tipovi replikacije

| Tip | Šta replicira | Kada se koristi |
|---|---|---|
| **MTree replication** | Jedan MTree | **Podrazumevani izbor.** Granularno, fleksibilno, podržava Retention Lock i sve topologije |
| **Collection replication** | Ceo sistem, 1:1 | Kada je odredište potpuna kopija izvora. Odredište je u celosti read-only i ne može primati ništa drugo |
| **Directory replication** | Jedan direktorijum | Legacy. Ne koristiti za nove instalacije |
| **Managed File Replication (MFR)** | Pojedinačne fajlove, po nalogu backup aplikacije | DD Boost. Kontroliše backup aplikacija (NetWorker, Avamar, NetBackup, PPDM), ne DD |

> **Pravilo:** ako nemate izričit razlog za nešto drugo, koristite MTree replikaciju.

### Topologije (MTree replikacija)

- **One-to-one** — klasičan produkcija → DR
- **Many-to-one (fan-in)** — više filijala na jedan centralni uređaj
- **One-to-many (fan-out)** — isti MTree na dve DR lokacije
- **Bi-directional** — svaki uređaj je izvor za jedan MTree i odredište za drugi
- **Cascaded** — A → B → C

Broj konteksta po sistemu je ograničen modelom.

---

## 7.2 Provera stanja (read-only)

```
replication status                       # sažeto, svi konteksti
replication show config                  # definicija svih konteksta
replication show detailed-status
replication show performance             # trenutna brzina
replication show stats                   # kumulativna statistika
replication show history
replication watch <destination>          # praćenje uživo
```

### Kako se čita `replication status`

| Kolona | Značenje | Na šta paziti |
|---|---|---|
| `State` | `initializing`, `normal`, `disabled`, `disconnected`, `error` | Sve osim `normal` i `initializing` je za istragu |
| `Sync'ed-as-of-time` | Trenutak do kog je odredište konzistentno sa izvorom | **Ovo je vaš stvarni RPO.** Ako je stariji od dogovorenog — incident |
| `Pre-comp Remaining` | Koliko logičkih podataka još čeka prenos | Mera zaostajanja. Stalno raste = replikacija ne stiže |

`Sync'ed-as-of-time` je najvažniji broj u ovom poglavlju. Sve ostalo je dijagnostika.

### Stanja i šta znače

| State | Značenje |
|---|---|
| `initializing` | Prvi puni prenos u toku. Normalno, može trajati danima |
| `normal` | Radi kako treba |
| `disabled` | Neko ju je ručno isključio |
| `disconnected` | Nema mrežne komunikacije sa parnjakom |
| `error` | Greška u kontekstu — pogledati `replication show detailed-status` i logove |
| `uninitialized` | Kontekst kreiran ali `replication initialize` nije pokrenut |

---

## 7.3 Kreiranje MTree replikacije

⚠️ Kontekst se definiše **na oba uređaja istom komandom**, pa se inicijalizuje
samo na izvoru.

```
# 1. Na IZVORU i na ODREDIŠTU (identična komanda):
replication add \
    source mtree://<izvorni-host>/data/col1/<mtree> \
    destination mtree://<odredisni-host>/data/col1/<mtree>

# 2. Samo na IZVORU:
replication initialize mtree://<odredisni-host>/data/col1/<mtree>

# 3. Praćenje inicijalizacije:
replication status
replication watch mtree://<odredisni-host>/data/col1/<mtree>
```

**Preduslovi:**
- Replication licenca na oba uređaja (`license show`)
- Izvorni MTree postoji; **odredišni MTree se ne kreira ručno** — nastaje sam
- Ime hosta mora biti razrešivo sa obe strane (`net show dns`, `net hosts show`)
- Otvoren mrežni put između uređaja (vidi 7.7)
- Kompatibilne DDOS verzije — odredište po pravilu ista ili novija verzija od izvora

Nakon uspešne inicijalizacije odredišni MTree je u `RD` stanju i read-only.

---

## 7.4 Upravljanje kontekstom

⚠️ / 🛑

```
replication disable <destination>            # ⚠️ pauzira, kontekst ostaje
replication enable <destination>             # ⚠️ nastavlja odakle je stao

replication modify <destination> <opcija> <vrednost>       # ⚠️

replication break <destination>              # 🛑 raskida kontekst (na OBA uređaja)
replication resync <destination>             # 🛑 ponovna sinhronizacija

replication del <destination>                # 🛑 brisanje definicije konteksta
```

### `disable` vs `break` — razlika koja se skupo plaća

- **`disable`** — privremena pauza. Kontekst i sve njegovo znanje ostaju.
  Nakon `enable` nastavlja se inkrementalno, prenosi se samo razlika.
  Ovo koristite za planirane radove, migracije linkova, održavanje.

- **`break`** — trajno raskidanje. Mora se izvršiti na **oba** uređaja.
  Odredišni MTree postaje `RW`. Za ponovno uspostavljanje treba
  `replication add` + `resync` ili puna reinicijalizacija.

> **Nikada ne radite `break` da biste "restartovali" replikaciju koja zaostaje.**
> Za to postoji `disable` / `enable`. `break` na velikom MTree-ju znači
> danima ponovne sinhronizacije preko WAN-a.

### `resync`

Koristi se kada su izvor i odredište razišli (npr. nakon `break`, nakon
disaster recovery testa, ili nakon što je u odredišni MTree neko pisao).
Poredi obe strane i prenosi razliku.

```
replication resync mtree://<odredisni-host>/data/col1/<mtree>     # 🛑
replication status
```

Sporije od inkrementalne replikacije, brže od pune inicijalizacije.

---

## 7.5 Throttle — ograničavanje propusnog opsega

Replikacija će bez ograničenja pojesti ceo link. Na deljenom WAN-u to je problem.

```
replication throttle show                              # trenutna podešavanja
replication throttle set <dan> <vreme> <brzina>        # ⚠️
replication throttle add <dan> <vreme> <brzina>        # ⚠️
replication throttle del <dan> <vreme>                 # ⚠️
replication throttle reset current                     # ⚠️
```

Brzina se zadaje u `Kibps`, `Mibps`, `Gibps` ili kao ključna reč:

| Vrednost | Značenje |
|---|---|
| `unlimited` | Bez ograničenja |
| `0` | **Potpuno zaustavlja replikaciju** u tom terminu |
| `50 Mibps` | Ograničenje na navedenu brzinu |

Tipičan raspored: ograničeno tokom radnog vremena, `unlimited` noću.

```
replication throttle add mon-fri 08:00 50 Mibps        # ⚠️
replication throttle add mon-fri 18:00 unlimited       # ⚠️
replication throttle show
```

> **Zamka:** throttle `0` postavljen "privremeno" i zaboravljen je jedan od
> najčešćih uzroka zaostajanja replikacije. Kada `Sync'ed-as-of-time` zaostaje,
> `replication throttle show` je među prve tri komande koje treba pustiti.

---

## 7.6 Opcije konteksta

```
replication option show
replication show config
```

Korisne opcije (⚠️ postavljaju se preko `replication modify`):

| Opcija | Kada |
|---|---|
| `low-bw-optim` | Spora WAN veza (ispod ~6 Mbps). Dodatna deduplikacija na uštrb CPU-a. Ne koristiti na brzim linkovima |
| `encryption` | Enkripcija replikacionog saobraćaja preko nezaštićene mreže |
| `destination-tenant-unit` | Secure Multi-Tenancy scenariji |

```
replication modify mtree://<host>/data/col1/<mtree> low-bw-optim enabled    # ⚠️
replication modify mtree://<host>/data/col1/<mtree> encryption enabled      # ⚠️
```

Izmena opcija po pravilu traži da je kontekst prethodno `disable`-ovan.

---

## 7.7 Mreža i portovi

```
net show settings
net show hardware
net ping <parnjak>
net show stats
replication option show                     # listen port
```

Replikacioni saobraćaj podrazumevano ide na **TCP 2051**. U zavisnosti od
konfiguracije i verzije mogu biti potrebni i dodatni portovi (upravljanje,
DD Boost, DNS, NTP).

> Pre nego što se javite mrežnom timu, potvrdite listu portova za vašu DDOS
> verziju iz **Security Configuration Guide**-a. Lista se menjala kroz verzije.

Odvajanje replikacionog saobraćaja na zaseban interfejs radi se preko
`net hosts` mapiranja imena parnjaka na IP adresu replikacione mreže:

```
net hosts show
net hosts add <ime-parnjaka> <ip-replikacione-mreze>     # ⚠️
```

Merenje stvarne propusnosti linka između uređaja:

```
net iperf server                            # na jednom uređaju
net iperf client <ip-drugog>                # na drugom
```

Detalji o mreži → **poglavlje 08**.

---

## 7.8 Replikacija i Retention Lock

- MTree replikacija **prenosi lock status fajlova** na odredište.
- Odredište mora imati odgovarajuću RL licencu. **Compliance MTree se ne može
  replicirati na uređaj bez uključenog Compliance režima.**
- Ako je odredište deo Cyber Recovery vault-a, kombinacija replikacije,
  lock-a i sync prozora ima svoja pravila — to nije standardna replikacija.
- Retention Lock **ne zamenjuje replikaciju**, i obrnuto. Lock štiti od
  logičkog brisanja, replikacija od gubitka lokacije.

Provera na odredištu:

```
mtree list
mtree retention-lock status mtree /data/col1/<mtree>
```

Detaljno → **poglavlje 06**.

---

## 7.9 Replikacija i kapacitet

Zaostala replikacija drži podatke na izvoru — cleaning ih ne može osloboditi
dok nisu preneti. Ako izvor raste, a `Pre-comp Remaining` stalno raste,
kapacitet i replikacija su isti problem.

```
replication status
filesys show space
filesys clean status
```

Vidi **poglavlje 04**.

---

## 7.10 Planirani radovi — procedura

Kada se radi održavanje na odredištu ili na linku:

```
# 1. Pre radova, na izvoru:
replication status                                   # zabeležiti stanje
replication disable <destination>                    # ⚠️

# 2. Nakon radova:
replication enable <destination>                     # ⚠️
replication status                                   # prati Sync'ed-as-of-time
replication watch <destination>
```

Ne koristiti `break` ni `del`. Kontekst se ne dira.

---

## 7.11 Checklist za novi replikacioni kontekst

- [ ] Replication licenca na oba uređaja (`license show`)
- [ ] DDOS verzije kompatibilne (odredište isto ili novije)
- [ ] DNS/hosts razrešavanje radi sa obe strane
- [ ] Mrežni put otvoren i izmeren (`net ping`, `net iperf`)
- [ ] Na odredištu ima dovoljno prostora za pun MTree (`filesys show space`)
- [ ] Ako izvor ima Retention Lock — odredište ima odgovarajuću licencu i režim
- [ ] Definisan throttle raspored ako je link deljen
- [ ] Dogovoren RPO i definisan alert kada `Sync'ed-as-of-time` pređe taj prag
- [ ] Kontekst dodat u monitoring

---

## Reference

- DDOS Administration Guide — poglavlje o replikaciji
- DDOS Command Reference Guide — sekcija `replication`
- DD Security Configuration Guide — lista mrežnih portova za vašu verziju
