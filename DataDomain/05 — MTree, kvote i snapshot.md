# 05 — MTree, kvote i snapshot

MTree je osnovna jedinica logičke podele na Data Domain uređaju. Sve što se
kasnije radi — kvote, snapshot-ovi, Retention Lock, replikacija, tenant izolacija —
radi se na nivou MTree-ja.

---

## 5.1 Šta je MTree

MTree je zaseban logički deo file systema sa sopstvenom putanjom, statistikom
i politikama. Putanja je uvek oblika:

```
/data/col1/<ime-mtree-a>
```

Svaki sistem ima podrazumevani MTree `/backup` (odnosno `/data/col1/backup`).
On postoji zbog kompatibilnosti sa starijim konfiguracijama i **na njemu se
ne može uključiti Retention Lock**. Za sve nove namene pravi se zaseban MTree.

Broj MTree-jeva po sistemu je ograničen modelom (tipično 100–256 aktivnih).
Nije beskonačno — nemojte praviti MTree po klijentu ako imate stotine klijenata.

**Kada praviti zaseban MTree:**
- različita retencija ili različita politika zaključavanja
- različit vlasnik podataka ili tenant
- odvojena kvota
- zaseban replikacioni kontekst
- odvojeno izveštavanje o zauzeću

---

## 5.2 Pregled MTree-jeva

```
mtree list                                       # svi MTree-jevi
mtree list /data/col1/<mtree>                    # jedan MTree
mtree show compression                           # zauzeće po MTree-ju
mtree show compression /data/col1/<mtree> last 7 days
mtree show performance
mtree show stats
```

`mtree list` je najkorisnija komanda u ovom poglavlju. U jednoj tabeli daje:

| Kolona | Značenje |
|---|---|
| `Name` | Putanja MTree-ja |
| `Pre-Comp` | Logička veličina |
| `Status` | `RW` (čitanje/pisanje), `RO` (read-only), `RD` (replication destination), `D` (obrisan, čeka čišćenje) |
| Retention Lock | Da li je uključen i u kom režimu |

Status `RD` znači da je MTree odredište replikacije i da se u njega **ne sme
pisati direktno** — pisanje ide samo kroz replikaciju sa izvora.

---

## 5.3 Kreiranje i upravljanje

⚠️

```
mtree create /data/col1/<ime>
mtree rename /data/col1/<staro> /data/col1/<novo>
mtree modify /data/col1/<mtree> tenant-unit <tenant>
```

🛑 Brisanje:

```
mtree delete /data/col1/<mtree>
mtree undelete /data/col1/<mtree>          # dok cleaning nije prošao
```

**Pre brisanja MTree-ja obavezno proveriti:**

1. Da nije odredište ili izvor aktivnog replikacionog konteksta — `replication show config`
2. Da nema zaključanih fajlova — `mtree retention-lock status mtree /data/col1/<mtree>`
   (MTree sa zaključanim fajlovima se **ne može** obrisati)
3. Da nije mount-ovan kod klijenata — `nfs show active`, `cifs show active`, `ddboost show connections`
4. Da nema aktivnih snapshot-ova koji se čuvaju iz nekog razloga

Obrisan MTree ostaje u `mtree list` sa statusom `D` i njegov prostor se
oslobađa tek nakon cleaninga. Do tada je moguć `mtree undelete`.

---

## 5.4 Per-MTree opcije

```
mtree option show /data/col1/<mtree>
mtree option set app-optimized-compression {none | global | oracle1} \
    mtree /data/col1/<mtree>                                    # ⚠️
```

`oracle1` se koristi za Oracle RMAN backup-e i daje značajno bolji dedupe
na tom tipu podataka. Za sve ostalo `none` ili `global`.

Izmena se primenjuje na **nove podatke**, ne retroaktivno.

---

## 5.5 Kvote

Kvote sprečavaju da jedan MTree pojede ceo uređaj. Postoje dva tipa:

- **Capacity quota** — ograničenje zauzeća (pre-comp)
- **Stream quota** — ograničenje broja istovremenih tokova (write/read/replikacija)

### Capacity kvote

```
quota capacity show                                              # sve postavljene kvote
quota capacity show mtrees /data/col1/<mtree>
quota enable                                                     # ⚠️ globalno uključivanje
quota disable                                                    # ⚠️

quota capacity set mtrees /data/col1/<mtree> \
    soft-limit 8 TiB hard-limit 10 TiB                           # ⚠️

quota capacity reset mtrees /data/col1/<mtree>                   # ⚠️
```

| Limit | Šta radi |
|---|---|
| **soft-limit** | Generiše alert. Upisi i dalje prolaze. |
| **hard-limit** | Upisi **padaju** kada se dostigne. Backup pada sa greškom "no space". |

Jedinice: `MiB`, `GiB`, `TiB`. Kvote se računaju na **pre-comp** veličinu.

> **Praktično:** postavite soft na ~80% od hard limita. Hard limit bez soft
> limita znači da će prvi znak problema biti pao backup.

### Stream kvote

```
quota streams show
quota streams set mtree /data/col1/<mtree> \
    write-stream-soft-limit <n> read-stream-soft-limit <n>       # ⚠️
quota streams reset mtree /data/col1/<mtree>                     # ⚠️
```

Koristi se kada jedan klijent zagušuje uređaj i ostali backup-i počnu da kasne.

---

## 5.6 Snapshot-ovi

Snapshot je read-only kopija stanja MTree-ja u trenutku kreiranja. Zauzima
prostor samo za razlike, ali **drži stare podatke i sprečava da ih cleaning
oslobodi** — što je čest uzrok "prostor se ne oslobađa" situacije (poglavlje 04).

Snapshot-ovi su vidljivi kroz skriveni direktorijum:

```
/data/col1/<mtree>/.snapshot/<ime-snapshota>/
```

### Pregled

```
snapshot list mtree /data/col1/<mtree>
snapshot list mtree /data/col1/<mtree> detailed
```

Prikazuje ime, vreme kreiranja, rok isteka i status (`active` / `expired`).

### Ručno kreiranje

⚠️

```
snapshot create <ime> mtree /data/col1/<mtree>
snapshot create <ime> mtree /data/col1/<mtree> retention 30days
```

Ako se `retention` ne navede, snapshot ostaje dok se ručno ne obriše.
**Uvek postavljajte retenciju** — snapshot bez roka je snapshot koji će
neko naći za dve godine dok traži gde je nestalo 12 TB.

### Isticanje i brisanje

⚠️

```
snapshot expire <ime> mtree /data/col1/<mtree>                   # istekne odmah
snapshot expire <ime> mtree /data/col1/<mtree> retention 7days   # produži rok
snapshot rename <staro> <novo> mtree /data/col1/<mtree>
```

Istekli snapshot se briše pri sledećem cleaning ciklusu, ne odmah.

### Rasporedi

```
snapshot schedule show
snapshot schedule show <ime-rasporeda>

snapshot schedule create <ime-rasporeda> \
    mtrees /data/col1/<mtree> \
    days mon,tue,wed,thu,fri \
    time 22:00 \
    retention 14days                                             # ⚠️

snapshot schedule modify <ime-rasporeda> retention 30days        # ⚠️
snapshot schedule destroy <ime-rasporeda>                        # ⚠️
```

Provera da raspored zaista radi: `snapshot list mtree ...` i pogledajte
da li postoje snapshot-ovi iz poslednjih nekoliko dana. Raspored koji je
prestao da radi ne generiše alert po sebi.

---

## 5.7 Vraćanje podataka iz snapshot-a

Snapshot je read-only. Podaci se vraćaju kopiranjem — na DD-u se to radi
komandom `fastcopy`, koja ne kopira fizički podatke već samo reference,
pa je gotovo trenutna bez obzira na veličinu.

⚠️

```
filesys fastcopy source /data/col1/<mtree>/.snapshot/<snap>/<putanja> \
    destination /data/col1/<mtree>/<putanja-za-vracanje>
```

Alternativno, klijent može direktno pročitati iz `.snapshot` direktorijuma
preko NFS/CIFS mount-a, ako je pristup dozvoljen.

> **Napomena:** `fastcopy` na odredište koje već postoji traži `force`.
> Nikada ne radite fastcopy preko produkcijskog direktorijuma bez prethodne
> provere — nema undo.

---

## 5.8 MTree i replikacija

Kod MTree replikacije (najčešći tip) odredišni MTree je u `RD` stanju i
read-only. Bitno:

- Ne pravite ručno odredišni MTree pre kreiranja konteksta — kreira se sam.
- Ne pišite u odredišni MTree, ni preko NFS-a ni preko DD Boost-a.
- Snapshot-ovi se repliciraju zajedno sa MTree-jem.
- Za brisanje odredišnog MTree-ja prvo mora da se prekine kontekst.

Detaljno → **poglavlje 07**.

---

## 5.9 Checklist za novi MTree u produkciji

- [ ] Ime po konvenciji (npr. `<aplikacija>-<sredina>`, bez razmaka i specijalnih znakova)
- [ ] Kreiran zaseban MTree, nije korišćen `/backup`
- [ ] Postavljena capacity kvota (soft + hard)
- [ ] Definisan snapshot raspored sa retencijom, ako je potreban
- [ ] Odlučeno da li ide Retention Lock i u kom režimu (**poglavlje 06**) — pre nego što stignu podaci
- [ ] Kreiran replikacioni kontekst ako MTree ide na DR lokaciju (**poglavlje 07**)
- [ ] Konfigurisan pristup: NFS export / CIFS share / DD Boost storage unit (**poglavlje 09**)
- [ ] Zabeleženo u internoj evidenciji: ko je vlasnik podataka, koja je retencija

---

## Reference

- DDOS Administration Guide — poglavlja o MTree-jevima, kvotama i snapshot-ovima
- DDOS Command Reference Guide — sekcije `mtree`, `quota`, `snapshot`, `filesys fastcopy`
