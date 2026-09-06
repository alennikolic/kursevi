# 06 — DD Retention Lock (zaključavanje backup fajlova)

Retention Lock (RL) sprečava brisanje i izmenu fajlova na MTree-ju dok ne istekne
zadati rok. Radi se na nivou pojedinačnog fajla, a uključuje se na nivou MTree-ja.

---

## 6.1 Dva režima — razlika koju morate znati

| | **Governance** | **Compliance** |
|---|---|---|
| Namena | Interna politika, zaštita od greške i od običnog ransomware brisanja | Regulatorni zahtev (SEC 17a-4, GDPR retencija, revizija) |
| Ko može da ga isključi | `admin` / `limited-admin` | Niko — ni Dell support. Traži `security-officer` odobrenje i nema revert-a nad zaključanim fajlovima |
| Može li se lock skratiti | Da, `mtree retention-lock revert` | **Ne** |
| Traži security officer nalog | Ne | **Da**, obavezno |
| Traži poseban system-level enable | Ne | Da |
| Tipičan max rok | do ~5 godina (default) | do ~70 godina |
| Licenca | Zasebna RL Governance licenca | Zasebna RL Compliance licenca |

> **Praktično pravilo:** ako niste sigurni koji vam treba — treba vam Governance.
> Compliance je jednosmerna vrata; pogrešno postavljen rok od 10 godina na
> pogrešnom MTree-ju znači da taj prostor ne možete osloboditi 10 godina.

---

## 6.2 Provera trenutnog stanja (read-only)

```
license show                                          # da li RL licenca uopšte postoji
elicense show
mtree list                                            # kolona sa RL statusom po MTree-ju
mtree retention-lock status mtree /data/col1/<mtree>  # detaljan status jednog MTree-ja
```

Za Compliance režim na nivou sistema:

```
system retention-lock compliance status
authorization show
user show list                                        # da li postoji security-officer nalog
```

`mtree retention-lock status` vraća:
- da li je RL `enabled` ili `disabled`
- režim (`governance` / `compliance`)
- `min-retention-period` i `max-retention-period`
- da li je konfigurisan automatic retention lock

---

## 6.3 Uključivanje Governance režima

⚠️ Menja konfiguraciju MTree-ja.

```
# 1. Licenca (ako već nije dodata)
elicense update                       # interaktivno, unosi se licencni fajl

# 2. Uključivanje na MTree-ju
mtree retention-lock enable mode governance mtree /data/col1/<mtree>

# 3. Postavljanje granica roka
mtree retention-lock set min-retention-period 24hours mtree /data/col1/<mtree>
mtree retention-lock set max-retention-period 1825days mtree /data/col1/<mtree>

# 4. Provera
mtree retention-lock status mtree /data/col1/<mtree>
```

Jedinice za period: `minutes`, `hours`, `days`, `months`, `years`
(npr. `12hours`, `90days`, `7years`).

**Bitna ograničenja:**
- RL se **ne može** uključiti na `/backup` MTree-ju.
- `min-retention-period` ne može biti kraći od 12 sati.
- Rok se ne može skratiti nakon što je fajl zaključan — samo produžiti.

---

## 6.4 Uključivanje Compliance režima

🛑 Nepovratno. Radi se samo uz formalno odobrenje i uz definisanog security officera.

```
# 1. Kreiranje security officer naloga (radi se kao admin)
user add <so-username> role security-officer

# 2. Uključivanje security officer autorizacije
authorization policy set security-officer enabled

# 3. Uključivanje Compliance na nivou sistema
#    Traži potvrdu security officera i restart file systema
system retention-lock compliance enable

# 4. Uključivanje na MTree-ju
mtree retention-lock enable mode compliance mtree /data/col1/<mtree>

# 5. Rokovi
mtree retention-lock set min-retention-period 24hours mtree /data/col1/<mtree>
mtree retention-lock set max-retention-period 3650days mtree /data/col1/<mtree>
```

Od trenutka kada je Compliance uključen na sistemu, veliki broj administrativnih
operacija (brisanje MTree-ja, isključivanje file systema, izmena sistemskog vremena,
neki upgrade koraci) traži dodatnu potvrdu security officera. To je namerno.

---

## 6.5 Kako se fajl zapravo zaključava

Retention Lock se **ne postavlja komandom na DD-u za pojedinačni fajl.** Klijent
(backup aplikacija ili skripta) zaključava fajl tako što mu postavi `atime`
u budućnost. DD tumači budući `atime` kao "zaključaj do tog datuma".

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
NetBackup) ovo radi automatski ako je RL integracija uključena u samoj aplikaciji.

---

## 6.6 Automatic Retention Lock

Za MTree-jeve gde klijent ne postavlja `atime` sam — tipično obični NFS/CIFS share,
ili odredište replikacije u Cyber Recovery scenariju — koristi se automatski lock.
DD sam zaključava svaki novi fajl nakon isteka "delay" perioda.

⚠️

```
mtree retention-lock set automatic-retention-period 30days mtree /data/col1/<mtree>
mtree retention-lock set automatic-lock-delay 120minutes mtree /data/col1/<mtree>
mtree retention-lock status mtree /data/col1/<mtree>
```

- `automatic-lock-delay` — koliko dugo fajl ostaje izmenjiv nakon poslednjeg
  upisa pre nego što se zaključa. Mora biti duže od trajanja najdužeg backup posla,
  inače će se fajl zaključati dok se još upisuje.
- `automatic-retention-period` — na koliko dugo se zaključava.

> **Najčešća greška:** prekratak `automatic-lock-delay`. Ako backup traje 4 sata,
> a delay je 120 minuta, fajl se zaključa usred posla i backup pada.

---

## 6.7 Otključavanje i revert (samo Governance)

⚠️ Radi se samo uz odobrenje. Loguje se i vidi u auditu.

```
# Vrati zaključavanje pojedinačnog fajla
mtree retention-lock revert /data/col1/<mtree>/<putanja-do-fajla>

# Isključi RL na celom MTree-ju (samo Governance)
mtree retention-lock disable mtree /data/col1/<mtree>
```

`mtree retention-lock disable` **ne otključava postojeće zaključane fajlove** —
samo sprečava zaključavanje novih. Postojeći fajlovi ostaju zaključani do isteka
roka ili dok se pojedinačno ne urade `revert`.

U **Compliance** režimu `revert` ne postoji. MTree se ne može obrisati dok
u njemu ima zaključanih fajlova.

---

## 6.8 Retention Lock i replikacija

Ovo je deo koji se najčešće pogrešno postavi.

- MTree replikacija **prenosi lock status fajlova** na odredište.
- Odredišni uređaj mora imati **istu ili kompatibilnu RL licencu i režim**.
  Compliance MTree se ne može replicirati na uređaj koji nema Compliance uključen.
- Kod Collection replikacije ceo sistem se preslikava, uključujući RL konfiguraciju.
- Retention Lock **ne zamenjuje replikaciju**. Zaključan fajl na jednom uređaju
  i dalje nestaje ako uređaj izgori. RL štiti od logičkog brisanja, replikacija
  od fizičkog gubitka.

Provera na odredištu:

```
mtree list
mtree retention-lock status mtree /data/col1/<mtree>
replication show config
```

Detalji o replikaciji → **poglavlje 07**.

---

## 6.9 Retention Lock i kapacitet

Zaključani podaci se **ne mogu očistiti cleaning-om (GC)** dok im ne istekne rok.
To je najčešći uzrok situacije "cleaning je odradio, a prostor se nije oslobodio".

```
mtree show compression /data/col1/<mtree>       # koliko MTree stvarno zauzima
filesys show space
filesys clean status
```

Ako je MTree sa RL-om glavni potrošač prostora, jedina opcija je čekanje isteka
roka ili proširenje kapaciteta. Planirajte kapacitet za **ceo period retencije**,
ne za trenutno zauzeće. Vidi **poglavlje 04**.

---

## 6.10 Checklist pre nego što uključite RL na produkciji

- [ ] Potvrđeno da je izabran ispravan režim (Governance vs Compliance) i to zapisano
- [ ] RL licenca postoji i na izvoru i na svim replikacionim odredištima
- [ ] `min` i `max` retention period usaglašeni sa politikom retencije backup aplikacije
- [ ] Ako se koristi automatic lock: `automatic-lock-delay` duži od najdužeg backup posla
- [ ] Za Compliance: security officer nalog kreiran, lozinka pohranjena van DD uređaja
- [ ] Izračunat kapacitet za pun period retencije, ne za trenutno zauzeće
- [ ] Testirano na test MTree-ju sa kratkim rokom (npr. 12h) pre produkcije
- [ ] Dokumentovano ko sme da radi `revert` i po kojoj proceduri

---

## Reference

- DDOS 8.9 Administration Guide, poglavlje o DD Retention Lock
- DDOS 8.9 Command Reference Guide, sekcije `mtree` i `system retention-lock`
- <https://www.dell.com/support/kbdoc/en-us/000126375/powerprotect-and-data-domain-core-documents>
