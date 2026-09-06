# 06 — DD Retention Lock (zaključavanje backup fajlova)

Retention Lock (RL) sprečava brisanje i izmenu fajlova na MTree-ju dok ne
istekne zadati rok. Radi se na nivou pojedinačnog fajla, a uključuje se na
nivou MTree-ja.

---

## 6.1 Dva režima

| | **Governance** | **Compliance** |
|---|---|---|
| Namena | Interna politika, zaštita od greške i od ransomware brisanja | Regulatorni zahtev (SEC 17a-4, revizija) |
| Ko može da ga isključi | `admin` / `limited-admin` | Niko — ni Dell support |
| Može li se lock skratiti | Da, `mtree retention-lock revert` | **Ne** |
| Traži nalog sa rolom `security` | Ne (osim ako je uključen `security-auth`) | **Da**, obavezno |
| Traži system-level enable | Ne | Da |
| Licenca | RL Governance | RL Compliance |

> **Praktično pravilo:** ako niste sigurni koji vam treba — treba vam Governance.
> Compliance je jednosmerna vrata; pogrešno postavljen rok od 10 godina na
> pogrešnom MTree-ju znači da taj prostor ne možete osloboditi 10 godina.

---

## 6.2 Provera trenutnog stanja (read-only)

```
elicense show                                         # da li RL licenca postoji
mtree list                                            # RL status po MTree-ju
mtree retention-lock status mtree /data/col1/<mtree>
mtree retention-lock show {min-retention-period | max-retention-period | automatic-retention-period | automatic-lock-delay} mtree /data/col1/<mtree>
```

Na nivou sistema:

```
system retention-lock compliance status
system retention-lock governance security-auth status
authorization policy show
authorization show history last 7 days
user show list                                        # postoji li nalog sa rolom `security`
```

> Komanda je `elicense show`, **ne** `license show`.

---

## 6.3 Uključivanje Governance režima

⚠️

```
# 1. Licenca (ako nije dodata)
elicense update [<license-file>]

# 2. Uključivanje na MTree-ju
mtree retention-lock enable mode governance mtree /data/col1/<mtree>

# 3. Granice roka
mtree retention-lock set min-retention-period 24hours mtree /data/col1/<mtree>
mtree retention-lock set max-retention-period 1825days mtree /data/col1/<mtree>

# 4. Provera
mtree retention-lock status mtree /data/col1/<mtree>
```

Jedinice: `minutes`, `hours`, `days`, `months`, `years` (npr. `12hours`, `90days`, `7years`).

**Ograničenja:**
- RL se **ne može** uključiti na `/backup` MTree-ju
- `min-retention-period` ne može biti kraći od 12 sati
- Rok se ne može skratiti nakon zaključavanja — samo produžiti

Reset postavki:

```
mtree retention-lock reset {min-retention-period | max-retention-period | automatic-retention-period | automatic-lock-delay} mtree /data/col1/<mtree>   # ⚠️
```

### Zaštita revert operacije u Governance režimu

Podrazumevano `revert` traži samo `sysadmin` lozinku. Može se pooštriti:

```
system retention-lock governance security-auth status
system retention-lock governance security-auth enable      # ⚠️
system retention-lock governance security-auth disable      # ⚠️
```

Kada je uključeno, `mtree retention-lock revert` traži i autorizaciju
security officera pored `sysadmin` lozinke. **Preporučeno na svakom
produkcijskom sistemu sa Governance režimom** — inače je jedan kompromitovan
admin nalog dovoljan da otključa sve.

---

## 6.4 Uključivanje Compliance režima

🛑 Nepovratno. Samo uz formalno odobrenje i definisanog security officera.

```
# 1. Nalog sa rolom `security` (radi se kao admin, samo prvi put)
user add <so-username> role security

# 2. Uključivanje autorizacije
authorization policy set security-officer enabled

# 3. Konfiguracija i uključivanje Compliance na nivou sistema
system retention-lock compliance configure
system retention-lock compliance enable
system retention-lock compliance status

# 4. Uključivanje na MTree-ju
mtree retention-lock enable mode compliance mtree /data/col1/<mtree>

# 5. Rokovi
mtree retention-lock set min-retention-period 24hours mtree /data/col1/<mtree>
mtree retention-lock set max-retention-period 3650days mtree /data/col1/<mtree>
```

> **Rola se zove `security`, ne `security-officer`.** `authorization policy set
> security-officer` je ispravan naziv politike, ali `user add ... role security`
> je ispravan naziv role. Ovo je najčešća greška pri postavljanju.
>
> Prvi `security` nalog kreira `admin`. Nakon toga **samo `security` korisnici**
> mogu dodavati ili brisati druge `security` naloge. Rola `security` se ne može
> dodeliti postojećem nalogu preko `user change role`.

Od trenutka kada je Compliance uključen, mnoge administrativne operacije
(brisanje MTree-ja, `filesys disable`, izmena sistemskog vremena, ručni cleaning,
neki upgrade koraci) traže dodatnu potvrdu security officera. To je namerno.

---

## 6.5 Kako se fajl zaključava

Retention Lock se **ne postavlja komandom na DD-u za pojedinačni fajl.**
Klijent zaključava fajl tako što mu postavi `atime` u budućnost. DD tumači
budući `atime` kao "zaključaj do tog datuma".

Sa Linux klijenta preko NFS mount-a:

```bash
# Zaključaj fajl do 1. januara 2030.
touch -a -t 203001010000 /mnt/dd/backup1/arhiva.bkp

# Provera - atime u budućnosti = fajl je zaključan
stat /mnt/dd/backup1/arhiva.bkp
```

Nakon zaključavanja:
- fajl se ne može obrisati ni izmeniti do isteka roka
- rok se može **produžiti** ponovnim postavljanjem daljeg `atime`
- rok se **ne može skratiti** (osim `revert`-om u Governance režimu)

Većina backup aplikacija (NetWorker, Avamar, Veeam, PowerProtect Data Manager,
NetBackup) ovo radi automatski ako je RL integracija uključena u aplikaciji.

---

## 6.6 Automatic Retention Lock

Za MTree-jeve gde klijent ne postavlja `atime` sam — obični NFS/CIFS share,
ili odredište replikacije u Cyber Recovery scenariju.

⚠️

```
mtree retention-lock set automatic-retention-period 30days mtree /data/col1/<mtree>
mtree retention-lock set automatic-lock-delay 120minutes mtree /data/col1/<mtree>
mtree retention-lock show automatic-retention-period mtree /data/col1/<mtree>
mtree retention-lock status mtree /data/col1/<mtree>
```

- `automatic-lock-delay` — koliko dugo fajl ostaje izmenjiv nakon poslednjeg
  upisa pre zaključavanja. **Mora biti duže od najdužeg backup posla.**
- `automatic-retention-period` — na koliko dugo se zaključava

> **Najčešća greška:** prekratak `automatic-lock-delay`. Ako backup traje 4 sata,
> a delay je 120 minuta, fajl se zaključa usred posla i backup pada.

---

## 6.7 Indefinite Retention Hold (legal hold)

Zadržava fajlove neograničeno, bez obzira na rok retencije. Koristi se za
sudske postupke i revizije.

⚠️

```
mtree retention-lock indefinite-retention-hold enable mtree /data/col1/<mtree>
mtree retention-lock indefinite-retention-hold disable mtree /data/col1/<mtree>
mtree retention-lock status mtree /data/col1/<mtree>
```

> Dok je hold aktivan, fajlovi se **ne mogu obrisati ni po isteku retencije**.
> Prostor ostaje zauzet dok se hold ne skine. Ovo treba biti dokumentovano —
> hold koji je neko uključio pre dve godine i zaboravio je čest uzrok
> neobjašnjivog zauzeća.

---

## 6.8 Izveštaj o zaključanim fajlovima

```
mtree retention-lock report generate retention-details mtrees {<lista> | all} [type {arl | ...}]
```

Daje pregled šta je zaključano, do kada i po kom režimu. Koristi se za:
- reviziju i dokazivanje usaglašenosti
- planiranje kapaciteta (koliko prostora je zaključano i do kada)
- istragu kada se prostor ne oslobađa

---

## 6.9 Otključavanje i revert (samo Governance)

⚠️ Loguje se i vidi u `authorization show history`.

```
mtree retention-lock revert <putanja-do-fajla>
mtree retention-lock disable mtree /data/col1/<mtree>
```

`mtree retention-lock disable` **ne otključava postojeće zaključane fajlove** —
samo sprečava zaključavanje novih. Postojeći ostaju zaključani do isteka roka
ili dok se pojedinačno ne urade `revert`.

Ako je uključen `system retention-lock governance security-auth`, `revert`
traži i autorizaciju security officera.

U **Compliance** režimu `revert` ne postoji. MTree se ne može obrisati dok
u njemu ima zaključanih fajlova.

---

## 6.10 Retention Lock i replikacija

- MTree replikacija **prenosi lock status fajlova** na odredište
- Odredište mora imati **istu ili kompatibilnu RL licencu i režim**.
  Compliance MTree se ne može replicirati na uređaj bez Compliance režima
- Kod Collection replikacije ceo sistem se preslikava, uključujući RL konfiguraciju
- Retention Lock **ne zamenjuje replikaciju**. RL štiti od logičkog brisanja,
  replikacija od fizičkog gubitka lokacije

Provera na odredištu:

```
mtree list
mtree retention-lock status mtree /data/col1/<mtree>
elicense show
replication show config
```

Detalji → **poglavlje 07**, CR vault → **poglavlje 12**.

---

## 6.11 Retention Lock i kapacitet

Zaključani podaci se **ne mogu očistiti cleaning-om** dok im ne istekne rok.
Najčešći uzrok situacije "cleaning je odradio, a prostor se nije oslobodio".

```
mtree show compression /data/col1/<mtree>
mtree retention-lock report generate retention-details mtrees all
filesys show space
filesys clean status
```

Planirajte kapacitet za **ceo period retencije**, ne za trenutno zauzeće.
Vidi **poglavlje 04**.

---

## 6.12 Checklist pre uključivanja RL na produkciji

- [ ] Potvrđen režim (Governance vs Compliance) i to zapisano
- [ ] RL licenca postoji na izvoru i na svim replikacionim odredištima (`elicense show`)
- [ ] `min` i `max` retention usaglašeni sa politikom retencije backup aplikacije
- [ ] Za Governance: razmotren `system retention-lock governance security-auth enable`
- [ ] Ako se koristi automatic lock: `automatic-lock-delay` duži od najdužeg backup posla
- [ ] Za Compliance: nalog sa rolom `security` kreiran, lozinka pohranjena van DD uređaja
- [ ] Izračunat kapacitet za pun period retencije
- [ ] Testirano na test MTree-ju sa kratkim rokom (12h) pre produkcije
- [ ] Dokumentovano ko sme da radi `revert` i po kojoj proceduri
- [ ] Dokumentovano ko sme da uključi `indefinite-retention-hold` i kako se prati

---

## Reference

- DD OS 8.6 Command Reference Guide — poglavlja `mtree` (sekcija `retention-lock`), `system`, `authorization`, `user`
- DD OS 8.6 Administration Guide — DD Retention Lock
