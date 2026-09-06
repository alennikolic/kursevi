# 05 — MTree, kvote i snapshot

MTree je osnovna jedinica logičke podele na Data Domain uređaju. Kvote,
snapshot-ovi, Retention Lock, replikacija i tenant izolacija — sve se radi
na nivou MTree-ja.

---

## 5.1 Šta je MTree

Zaseban logički deo file systema sa sopstvenom putanjom, statistikom i
politikama. Putanja je uvek:

```
/data/col1/<ime-mtree-a>
```

Svaki sistem ima podrazumevani MTree `/backup` (`/data/col1/backup`). Postoji
zbog kompatibilnosti sa starijim konfiguracijama i **na njemu se ne može
uključiti Retention Lock**. Za nove namene pravi se zaseban MTree.

Broj MTree-jeva je ograničen modelom (tipično 100–256 aktivnih).

**Kada praviti zaseban MTree:**
- različita retencija ili politika zaključavanja
- različit vlasnik podataka ili tenant
- odvojena kvota
- zaseban replikacioni kontekst
- odvojeno izveštavanje o zauzeću

---

## 5.2 Pregled

```
mtree list [<mtree-path>] [tenant-unit <tu>]
mtree show compression {<mtree-path> | tenant-unit <tu>} [tier {active | cloud}] [last <n> {hours | days}]
mtree show performance {<mtree-path> | tenant-unit <tu>} [interval <n> {mins | hrs}]
mtree degradedlist
```

> **`mtree show stats` ne postoji.** Postoje samo `mtree show compression`
> i `mtree show performance`.

`mtree list` daje:

| Kolona | Značenje |
|---|---|
| `Name` | Putanja MTree-ja |
| `Pre-Comp` | Logička veličina |
| `Status` | `RW`, `RO`, `RD` (replication destination), `D` (obrisan, čeka čišćenje) |
| Retention Lock | Da li je uključen i u kom režimu |

Status `RD` znači da je MTree odredište replikacije i da se u njega **ne sme
pisati direktno**.

---

## 5.3 Kreiranje i upravljanje

⚠️

```
mtree create <mtree-path> [tenant-unit <tu>] \
    [quota-soft-limit <n> {MiB|GiB|TiB|PiB}] [quota-hard-limit <n> {MiB|GiB|TiB|PiB}]

mtree rename <stara-putanja> <nova-putanja>
mtree modify <mtree-path> tenant-unit {<tu> | none}
```

Kvota se može zadati odmah pri kreiranju — preporučeno.

🛑 Brisanje:

```
mtree delete <mtree-path>
mtree undelete <mtree-path>          # dok cleaning nije prošao
```

**Pre brisanja obavezno proveriti:**

1. Nije izvor ni odredište replikacije — `replication show config`
2. Nema zaključanih fajlova — `mtree retention-lock status mtree <putanja>`
   (MTree sa zaključanim fajlovima se **ne može** obrisati)
3. Nije u upotrebi — `nfs export show`, `cifs share show`, `ddboost storage-unit show`
4. Nema snapshot-ova koje treba sačuvati
5. Nije CR PIT kopija (`cr-copy-*`) — te se brišu kroz CRCLI (**poglavlje 12**)

Obrisan MTree ostaje u `mtree list` sa statusom `D`; prostor se oslobađa tek
nakon cleaninga. Do tada je moguć `mtree undelete`.

---

## 5.4 Per-MTree opcije

```
mtree option show [mtree <mtree-path>]
mtree option set app-optimized-compression {none | global | oracle1} mtree <mtree-path>   # ⚠️
mtree option reset app-optimized-compression mtree <mtree-path>                           # ⚠️
```

`oracle1` se koristi za Oracle RMAN backup-e. Za ostalo `none` ili `global`.
Primenjuje se na nove podatke.

---

## 5.5 Kvote

Dva tipa: **capacity** (zauzeće) i **stream** (broj istovremenih tokova).

### Capacity kvote

```
quota capacity show {all | mtrees <lista> | storage-units <lista> | tenant-unit <tu>}
quota capacity status
```

⚠️

```
quota capacity enable
quota capacity disable

quota capacity set {all | mtrees <lista> | storage-units <lista>} \
    {soft-limit <n> {MiB|GiB|TiB|PiB} | hard-limit <n> {MiB|GiB|TiB|PiB} | \
     soft-limit <n> {...} hard-limit <n> {...}}

quota capacity reset {all | mtrees <lista> | storage-units <lista>} [soft-limit] [hard-limit]
```

> **`quota enable` i `quota disable` su deprecated** (kao i `quota set`,
> `quota show`, `quota reset`, `quota status`). Koristi se `quota capacity ...`.

Primeri iz dokumentacije:

```
quota capacity set mtrees /data/col1/backup1 soft-limit 10 GiB
quota capacity set mtrees /data/col1/backup1 soft-limit 100 GiB hard-limit 1 TiB
quota capacity set storage-units DDBOOST_SU soft-limit 100 GiB hard-limit 1 TiB
```

| Limit | Šta radi |
|---|---|
| **soft-limit** | Generiše alert. Upisi prolaze. |
| **hard-limit** | Upisi **padaju**. Backup pada sa greškom o nedostatku prostora. |

Lista MTree-jeva može biti razdvojena razmakom, zarezom ili dvotačkom.
Kvote se računaju na **pre-comp** veličinu.

> **Kvote se ne primenjuju dok mehanizam nije uključen** — `quota capacity enable`.
> Postavljanje limita bez toga ne radi ništa. Provera: `quota capacity status`.
>
> Postavljanje kvota ne traži isključivanje file systema i ne utiče na performanse.

**Preporuka:** soft na ~80% od hard limita. Hard bez soft-a znači da će prvi
znak problema biti pao backup.

### Stream kvote

⚠️ Rade **samo nad storage unit-ovima** (DD Boost), ne nad MTree-jevima.

```
quota streams show {all | storage-unit <su> | tenant-unit <tu>}
quota streams set storage-units <lista> [write-stream-soft-limit <n>] [read-stream-soft-limit <n>]
quota streams reset storage-units <lista> [write-stream-soft-limit] [read-stream-soft-limit]
```

Koristi se kada jedan klijent zagušuje uređaj i ostali backup-i kasne.

---

## 5.6 Snapshot-ovi

Read-only kopija stanja MTree-ja. Zauzima prostor samo za razlike, ali
**drži stare podatke i sprečava da ih cleaning oslobodi** — čest uzrok
"prostor se ne oslobađa" situacije (poglavlje 04).

Vidljivi kroz:

```
/data/col1/<mtree>/.snapshot/<ime-snapshota>/
```

### Pregled

```
snapshot list {mtree <mtree-path> | tenant-unit <tu> | all} [show-type {secured | ...}]
```

Prikazuje ime, vreme kreiranja, rok isteka i status.

### Kreiranje

⚠️

```
snapshot create <ime> mtree <mtree-path> [retention {<datum> | <period>}] [secured]
```

`secured` pravi snapshot koji se ne može obrisati pre isteka retencije —
koristan za zaštitu od brisanja, uz Retention Lock.

> **Uvek postavljajte retenciju.** Snapshot bez roka je snapshot koji će
> neko naći za dve godine dok traži gde je nestalo 12 TB.

### Isticanje i preimenovanje

⚠️

```
snapshot expire <ime> mtree <mtree-path> [retention {<datum> | <period> | forever}]
snapshot rename <staro-ime> <novo-ime> mtree <mtree-path>
```

Istekli snapshot se briše pri sledećem cleaning ciklusu, ne odmah.

### Rasporedi

```
snapshot schedule show [<ime> | mtrees <lista> | tenant-unit <tu>]
```

⚠️

```
snapshot schedule create <ime> [mtrees <lista>] [days <dani>] time <vreme>[,<vreme>...] \
    [retention <period>] [snap-name-pattern <šablon>]

snapshot schedule create <ime> [mtrees <lista>] [days <dani>] time <vreme> every <mins> \
    [retention <period>] [snap-name-pattern <šablon>]

snapshot schedule create <ime> [mtrees <lista>] [days <dani>] time <vreme>-<vreme> \
    [every {<hrs> | <mins>}] [retention <period>] [snap-name-pattern <šablon>]

snapshot schedule modify <ime> [...]
snapshot schedule add <ime> mtrees <lista>
snapshot schedule del <ime> mtrees <lista>
snapshot schedule destroy {<ime> | all}
snapshot schedule reset
```

> **Retencija u rasporedu se zadaje isključivo u danima** — `retention 14days`.
> Drugi oblici neće proći.

Primer iz dokumentacije:

```
snapshot schedule create sm1 mtrees /data/col1/m1 time 00:00-23:00 every 1mins retention 1days
```

Provera da raspored radi: `snapshot list mtree ...` i pogledajte ima li
snapshot-ova iz poslednjih dana. Raspored koji je prestao da radi ne generiše
alert sam po sebi.

---

## 5.7 Vraćanje podataka iz snapshot-a

Snapshot je read-only. Podaci se vraćaju `fastcopy` operacijom, koja kopira
reference a ne fizičke podatke — gotovo trenutno bez obzira na veličinu.

⚠️

```
filesys fastcopy [retention-lock [<new-lock-duration>]] source <izvor> destination <odrediste>
```

Primer:

```
filesys fastcopy source /data/col1/backup1/.snapshot/snap01/db \
    destination /data/col1/backup1/restore/db
```

Putanje sa razmacima se navode pod dvostrukim navodnicima ili sa `\` pre razmaka:

```
filesys fastcopy source "/data/col1/mtree name/fast copy" destination /data/col1/mtree2/dir
```

Alternativno, klijent može direktno čitati iz `.snapshot` direktorijuma preko
NFS/CIFS mount-a.

> Rola `backup-operator` **ne može** kopirati ceo MTree pomoću `fastcopy`.
>
> Nema undo. Nikada ne radite fastcopy preko produkcijskog direktorijuma
> bez prethodne provere.

---

## 5.8 MTree i replikacija

Kod MTree replikacije odredišni MTree je u `RD` stanju i read-only:

- Ne pravite ručno odredišni MTree pre kreiranja konteksta — kreira se sam
- Ne pišite u odredišni MTree ni preko NFS-a ni preko DD Boost-a
- Snapshot-ovi se repliciraju zajedno sa MTree-jem
- Za brisanje odredišnog MTree-ja prvo mora da se prekine kontekst

Detaljno → **poglavlje 07**.

---

## 5.9 Checklist za novi MTree u produkciji

- [ ] Ime po konvenciji (npr. `<aplikacija>-<sredina>`, bez razmaka i specijalnih znakova)
- [ ] Kreiran zaseban MTree, nije korišćen `/backup`
- [ ] Postavljena capacity kvota (soft + hard) i **uključen** `quota capacity enable`
- [ ] Definisan snapshot raspored sa retencijom u danima, ako je potreban
- [ ] Odlučeno da li ide Retention Lock i u kom režimu (**poglavlje 06**) — pre nego što stignu podaci
- [ ] Kreiran replikacioni kontekst ako MTree ide na DR lokaciju (**poglavlje 07**)
- [ ] Konfigurisan pristup: `nfs export` / CIFS share / DD Boost storage unit (**poglavlje 09**)
- [ ] Zabeleženo u internoj evidenciji: vlasnik podataka, retencija

---

## Reference

- DD OS 8.6 Command Reference Guide — poglavlja `mtree`, `quota`, `snapshot`, `filesys`
- DD OS 8.6 Administration Guide — MTree-jevi, kvote, snapshot-ovi
