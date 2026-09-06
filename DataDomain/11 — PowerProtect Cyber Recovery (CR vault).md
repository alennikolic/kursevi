# 11 — PowerProtect Cyber Recovery (CR vault)

Cyber Recovery nije "još jedna replikacija na DR lokaciju". To je izolovano
okruženje sa kontrolisanim, povremenim otvaranjem veze ka produkciji, u kome se
prave nepromenljive kopije podataka koje se mogu analizirati i iz kojih se
može oporaviti nakon napada.

Pisano iz ugla **DD inženjera** — šta se vidi na DD uređajima, šta se sme a
šta ne sme dirati, i minimum CRCLI-ja za dijagnostiku.
Provereno prema **Cyber Recovery 20.3 Command-Line Interface Reference Guide**.

> ⛔ **Najvažnije pravilo poglavlja:** na vault DD uređaju **ne dirate replikacione
> kontekste ručno**. Ako vidite kontekst u stanju `disabled` — to nije kvar, to je
> dizajn. Detalji u sekciji 12.5.

---

## 12.1 Kako se razlikuje od obične replikacije

| | Klasična DR replikacija | Cyber Recovery vault |
|---|---|---|
| Veza ka produkciji | Stalno otvorena | **Zatvorena po pravilu**, otvara se samo tokom sync prozora |
| Ko kontroliše replikaciju | DD (kontekst je `enabled`) | **CR softver** — on uključuje i isključuje kontekst |
| Šta štiti | Gubitak lokacije, hardvera | Napad, ransomware, insajder, kompromitovan administrator |
| Kopije | Odredišni MTree prati izvor | **PIT kopije**, zaključane Retention Lock-om |
| Analiza sadržaja | Nema | CyberSense (opciono) |
| Upravljanje | DD CLI / DDSM | CR UI ili CRCLI |

Kod DR replikacije napadač koji kompromituje produkcioni DD može pratiti
replikaciju do DR uređaja. Kod CR vault-a veza je zatvorena veći deo vremena,
vault DD nema pristup sa produkcione mreže, a kopije su zaključane.

---

## 12.2 Komponente i pristup

| Komponenta | Uloga |
|---|---|
| **Produkcioni DD** | Izvor. MTree-jevi koji se štite |
| **Vault DD** | Odredište u izolovanoj mreži. Replicirani MTree-jevi i PIT kopije |
| **CR management host** | Server (fizički, VM ili OVA) u vault-u. Orkestrira sve |
| **CyberSense** (opciono) | Analizira PIT kopije i traži tragove napada |
| **Recovery sandbox** | Okruženje u vault-u za testni ili stvarni oporavak |

CR softver radi kao skup Docker kontejnera. Binarni fajlovi:
`/opt/dellemc/cr/bin/` (`crcli`, `cradmin`, `crsetup.sh`).

Pristup:
- **CR UI:** `https://<cr-management-host>:14777`
- **CRCLI:** SSH na management host, pa `crcli login --username <ime>`

### Uloge u CR 20.3

| Uloga | Šta može |
|---|---|
| **Security Admin** | Upravljanje korisnicima, licencama, alertima. **Jedini može `vault release`.** Ne administrira DD storage ni politike |
| **Admin** | Politike, DD storage, poslovi, oporavak, `vault secure` |
| **Vault Operator** | Uglavnom pregled; može ručno pokrenuti akcije politike, videti kopije, preuzeti izveštaje analize, `vault secure` |

> Nalog `crso` se i dalje kreira pri instalaciji i ima **Security Admin** ulogu.
> Naziv uloge u 20.3 je "Security Admin", ne "security officer" — to je bitno
> kada čitate dokumentaciju ili tražite ko sme šta.

---

## 12.3 Stanja vault-a

Prvo što se proverava kada neko pita "da li je vault u redu".

| Stanje | Značenje |
|---|---|
| **Locked** | **Normalno.** Veza ka produkciji zatvorena. Podaci ne ulaze |
| **Unlocked** | Veza otvorena jer je u toku sync operacija. Normalno tokom sync prozora |
| **Secured** | Ručno obezbeđen zbog sumnje na napad. **Svi poslovi se otkazuju i naredni padaju dok se vault ne oslobodi** |
| **Degraded** | Nešto u okruženju ne radi — proveriti alerte i događaje |

```
crcli vault state
```

Primer izlaza:

```
The current state of the vault is: Locked
The vault has been in unlocked state for: 0 seconds
```

> **Vault dugo u `Unlocked` stanju bez aktivnog sync posla je incident.**
> Otvorena veza je upravo ono što CR treba da spreči. → PB-14 u poglavlju 10.
>
> Izuzetak: neposredno nakon instalacije, dok traje prva inicijalizacija
> replikacije. Port se zatvara automatski po završetku.

---

## 12.4 Pogled sa DD strane

### Na produkcionom DD-u

```
replication show config
replication status
mtree list
```

Izgleda kao običan izvor MTree replikacije, samo je kontekst veći deo vremena
`disabled` — CR ga isključuje nakon svakog sync prozora.

### Na vault DD-u

```
mtree list
replication show config
replication status
filesys show space
mtree retention-lock status mtree /data/col1/<mtree>
elicense show
```

Videćete:

- **Replicirane MTree-jeve** u `RD` stanju
- **PIT kopije** — zasebni MTree-jevi sa imenima po šablonu
  `cr-copy-<politika>-<timestamp>`. Nastaju `fastcopy` operacijom, pa u početku
  dele segmente sa izvornim MTree-jem
- **Retention Lock uključen** na MTree-jevima sa kopijama
- **Replikacioni kontekst u `disabled` stanju** veći deo vremena

---

## 12.5 Šta se NE radi na vault DD-u

Ovo je sekcija zbog koje poglavlje postoji. Sve navedeno su stvari koje
DD inženjer instinktivno uradi, a koje razbijaju CR okruženje.

| ⛔ Ne radi se | Zašto |
|---|---|
| `replication enable <ctx>` na `disabled` kontekstu | Kontekst je namerno isključen. Ručno uključivanje otvara vezu ka produkciji van sync prozora i ruši air gap. Replikaciju uključuje isključivo CR softver |
| `replication break` / `replication resync` | Raskida vezu koju CR očekuje. Politike počinju da padaju |
| `mtree delete` nad `cr-copy-*` MTree-jevima | To su PIT kopije kojima CR upravlja. Brisanje ide kroz `crcli policy delete-copy` |
| `mtree retention-lock disable` / `revert` | Ruši imutabilnost koja je svrha vault-a |
| Otvaranje dodatnih mrežnih puteva ka vault DD-u | Vault DD sme da priča samo sa CR management hostom i, tokom sync prozora, sa produkcionim DD-om |
| Kreiranje NFS/CIFS export-a ka produkcionoj mreži | Probija izolaciju |
| Reboot vault DD-a bez provere CR poslova | Posao u toku pada; politika ostaje u nekonzistentnom stanju |
| Promena lozinke DD naloga koji CR koristi | CR gubi pristup storage-u. Menja se kroz `crcli dd modify`, ne samo na DD-u |
| Nadogradnja DDOS-a bez provere Support Matrix-a | CR politike prestaju da rade |

**Ako mislite da nešto na vault DD-u treba popraviti — prvo `crcli jobs list`
i `crcli alerts list`.** Skoro uvek je odgovor u CR-u, ne na DD-u.

---

## 12.6 CRCLI — osnove

```bash
/opt/dellemc/cr/bin/crcli login --username <username>
crcli whoami                             # ko sam trenutno
crcli version                            # verzija i build
crcli help
crcli <komanda> --help
crcli logout
```

Većina komandi podržava `--json`, korisno za skripte i monitoring.

Stanje celog okruženja:

```
crcli system details
```

Vraća verzije CR servisa, OS, Docker, bazu, verziju DDOS-a na vault DD-u i
verzije CyberSense hostova. Prva komanda za support case.

---

## 12.7 Politike i akcije

**Politika** definiše: koji MTree se štiti, sa kog DD-a, na koji vault DD,
šta se radi sa podacima i po kom rasporedu.

### Akcije

| Akcija (`--action`) | Šta radi |
|---|---|
| `sync` | Replicira MTree sa produkcije u vault |
| `copy` | Pravi PIT kopiju već repliciranog MTree-ja |
| `sync-copy` | Sync pa odmah Copy |
| `lock` | Retention Lock nad fajlovima u PIT kopiji |
| `copy-lock` | Copy pa Lock |
| `securecopy` | Sync + Copy + Lock. **Najčešće zakazivana akcija** |
| `analyze` | CyberSense analiza PIT kopije |
| `securecopyanalyze` | Sve zajedno, uključujući analizu |
| `shelteredharborcopy` | Sheltered Harbor scenario |

> Ne mogu se pokretati istovremene Sync ili Lock akcije za istu politiku.
> Copy akcije mogu paralelno.

### Komande

```
crcli policy list
crcli policy show --policyname <ime>
crcli policy list-copy --policyname <ime> [--copyname <kopija>] [--json]
```

⚠️ Pokretanje i izmene:

```
crcli policy run --policyname <ime> --action {securecopy | sync-copy | copy-lock |
    sync | copy | lock | analyze | securecopyanalyze | shelteredharborcopy}
    [--appnickname <ime>] [--copyname <ime>] [--relockduration <trajanje>]
    [--storagedatainterface <interfejs>] [--watch <sekunde>]

crcli policy add ...
crcli policy modify ...
crcli policy delete --policyname <ime>                              # 🛑
crcli policy delete-copy --policyname <ime> --copyname <kopija>     # 🛑
crcli policy analysis-report download --policyname <ime> ...
crcli policy analysis-report email --policyname <ime> ...
crcli policy list-sandboxes --policyname <ime>
```

Primer:

```
crcli policy run --policyname phm-policy --action copy-lock --watch 15
```

### Čitanje `crcli policy list-copy`

Za svaku kopiju daje ime, datum nastanka, do kada je zaključana i status
zaključavanja, a za analizirane kopije i `lastanalysisstatus`.

**Šta se gleda:**

| Nalaz | Značenje |
|---|---|
| `Lock Status` = `Unlocked` na kopiji koja je trebalo da bude zaključana | Lock korak nije odradio — proveriti RL licencu na vault DD-u |
| `lastanalysisstatus` ≠ `Good` | CyberSense je nešto našao → PB-16 |
| Nema novih kopija u očekivanom ritmu | Raspored ne radi → `crcli jobs list` |
| `expiresOn` u prošlosti a kopija postoji | Čišćenje ne radi → 12.10 |

---

## 12.8 Poslovi (jobs)

```
crcli jobs list
crcli jobs list -t protection -running       # samo aktivni zaštitni poslovi
crcli jobs show --jobname <ime>
crcli jobs watch --jobname <ime>
crcli jobs cancel --jobname <ime> [--watch <sekunde>]      # ⚠️
```

Posao prolazi kroz korake (`Step 1 of 2: sync`, `Step 2 of 2: copy`).
Kod `jobs cancel` u detaljima se vidi korak "Disabling replication context" —
to je CR koji zatvara vezu ka produkciji. Isto se dešava i pri normalnom završetku.

**Kod sporog sync-a** posao stoji na koraku `sync` sa procentom koji sporo raste.
To je replikacija na DD nivou — proverite i sa DD strane, ali **ne dirajte kontekst**:

```
replication status all detailed
replication show performance all
net show stats interfaces
```

Vidi poglavlja 07 i 08.

---

## 12.9 Alerti, događaji i storage

```
crcli alerts list
crcli alerts show --alertid <id>
crcli alerts acknowledge --alertid <id>
crcli alerts unacknowledge --alertid <id>
crcli alerts modify --alertid <id> ...        # dodavanje beleške
crcli events list
crcli events show --eventid <id>

crcli dd list                                 # DD sistemi poznati CR-u
crcli dd show --name <ime>
crcli dd modify ...                           # ⚠️ npr. izmena kredencijala
crcli dd deleteobsoletemtrees                 # ⚠️ čišćenje zaostalih MTree-jeva

crcli apps list                               # CyberSense i drugi hostovi
crcli apps show --name <ime>
crcli license show
crcli license add ...                         # ⚠️
```

Za punu sliku treba pogledati alerte na sva tri mesta:

```
crcli alerts list          # CR management host
alerts show current        # vault DD
alerts show current        # produkcioni DD
```

`crcli dd list` pokazuje da li CR uopšte može da priča sa DD uređajima.
Prva provera ako politika pada odmah po pokretanju.

---

## 12.10 Kapacitet vault-a

Vault DD se puni brže nego što ljudi očekuju, iz dva razloga:

1. **PIT kopije se prave `fastcopy` operacijom** i u početku ne troše skoro
   ništa. Ali kako se produkcioni podaci menjaju, svaka kopija drži sve više
   sopstvenih segmenata.
2. **Kopije su zaključane Retention Lock-om.** Cleaning ih **ne može**
   osloboditi dok ne istekne rok, čak i kad ih obrišete kroz CR.

Kapacitet vault DD-a je funkcija **broja kopija × dužine lock-a**, a ne
trenutne veličine produkcionih podataka.

### Čišćenje starih kopija — crcli system clean

CR ima sopstveni mehanizam čišćenja, odvojen od DD cleaning-a:

```
crcli system clean --show
```

⚠️

```
crcli system clean --days <broj-dana> --copies <broj-kopija>
crcli system clean --alerts <broj-dana>
crcli system clean --cleannow
```

Primer iz dokumentacije:

```
crcli system clean --days 10 --copies 5
```

> Ovo definiše koliko kopija i koliko dugo CR zadržava. **Ako ovo nije
> podešeno, kopije se gomilaju** i vault DD se puni, čak i kad politike rade
> savršeno. Provera `crcli system clean --show` ide u dnevnu rutinu.

### Provera

```
# Na vault DD-u
filesys show space
mtree show compression
mtree list
mtree retention-lock report generate retention-details mtrees all
filesys clean status

# U CR-u
crcli system clean --show
crcli policy list-copy --policyname <ime>
```

### Ako se vault puni

1. `crcli system clean --show` — da li je politika zadržavanja uopšte podešena
2. `crcli policy list-copy` po svim politikama — koliko kopija i do kada su zaključane
3. Obrisati kopije kojima je lock istekao — `crcli policy delete-copy` 🛑
4. `crcli dd deleteobsoletemtrees` ⚠️ — zaostali MTree-jevi
5. Cleaning na vault DD-u — `filesys clean start` ⚠️
6. Ako su kopije još zaključane — **prostor se ne može osloboditi.**
   Revizija retencije u politikama ili proširenje kapaciteta

> Nema retroaktivnog skraćivanja lock-a. Vidi poglavlje 06.

---

## 12.11 Ručno obezbeđivanje vault-a

"Crveno dugme" — kada postoji sumnja na napad na produkciju.

```
crcli vault state                        # provera trenutnog stanja
crcli vault secure                       # 🛑 Security Admin, Admin ili Vault Operator
crcli vault release                      # 🛑 SAMO Security Admin
```

> **`crcli vault lock` i `crcli vault unlock` ne postoje.** U CR 20.3 postoje
> isključivo `vault secure`, `vault release` i `vault state`. Zatvaranje veze
> u normalnom radu je automatsko — obavlja ga CR po završetku posla.

Kada je vault `Secured`:
- **svi poslovi se otkazuju**
- **svi naredni poslovi padaju** dok se vault ne oslobodi
- CR podiže alert

Obe komande traže interaktivnu potvrdu (`y/n`). `crcli vault release` može
izvršiti **samo Security Admin**, i radi se tek kada se potvrdi da pretnja
više ne postoji.

---

## 12.12 DR backup CR konfiguracije

Ako se izgubi CR management host, gubi se i definicija svih politika.
CR ima ugrađeni DR backup koji to sprečava.

```
crcli system drbackup --show
```

⚠️

```
crcli system drbackup --enable
crcli system drbackup --disable
crcli system drbackup --backupnow
crcli system drbackup --days <broj-dana>
crcli system drbackup --mgmtddnickname <dd> --mgmtddmtree /data/col1/<mtree>
crcli system drbackup --list
```

> **Ovo mora biti uključeno i provereno.** DR backup se čuva na DD-u u vault-u,
> u zasebnom MTree-ju. Bez njega oporavak CR okruženja znači ponovnu izgradnju
> svih politika ručno.
>
> Lockbox passphrase je i dalje neophodan za obnovu — **ako se izgubi, ne može
> se povratiti** i CR softver se mora reinstalirati.

---

## 12.13 Support bundle i održavanje CR-a

```
crcli system bundle --list
crcli system bundle --show --bundlename <ime>
crcli system bundle --create             # ⚠️ CR log collection + DD support bundle
crcli system bundle --delete --bundlename <ime>    # ⚠️
crcli system details
crcli system email --show
crcli system logincount --maxcount <n>   # ⚠️
```

`crcli system bundle --create` prikuplja i CR logove i DD support bundle —
to je ono što se šalje uz CR case.

Lozinke i lockbox:

```
/opt/dellemc/cr/bin/crsetup.sh --change-passwords     # ⚠️ prekida servise
```

> Menja lockbox passphrase, lozinku baze i `crso` lozinku. **Prekida rad
> Docker kontejnera** — ne raditi dok ima aktivnih poslova, inače vault može
> ostati u neobezbeđenom stanju.

---

## 12.14 Oporavak

```
crcli recovery list-sandboxes
crcli policy list-copy --policyname <ime>          # izbor kopije za oporavak
crcli policy list-sandboxes --policyname <ime>
```

Scenariji:

- **Oporavak unutar vault-a (sandbox)** — kopija se otvori u izolovanom
  okruženju u vault-u; ništa ne napušta vault
- **Alternate recovery** — podaci se repliciraju iz vault-a na alternativni
  DD sistem, odakle backup aplikacija radi standardni restore
- **Povratak u produkciju** — tek nakon što je produkcija očišćena i potvrđeno
  da je kopija zdrava (CyberSense `Good`)

CR 20.3 podržava i **recovery check** — zakazanu ili ručnu proveru da se kopija
zaista može oporaviti za NetWorker i PowerProtect Data Manager okruženja.
To je jedina prava potvrda da vault radi svoj posao.

> **Oporavak nije DD operacija.** Ne radi se `replication resync` sa vault DD-a
> ka produkciji ručno. Ručna replikacija iz vault-a može preneti kompromitovane
> podatke i otvoriti vezu koja treba da bude zatvorena.

Za konkretnu backup aplikaciju postoje zasebni Dell white paper-i sa
korak-po-korak procedurom. **Procedura oporavka mora biti napisana i testirana
pre incidenta.**

---

## 12.15 Planirani radovi na DD-u u vault-u

1. `crcli jobs list -t protection -running` — nema aktivnih poslova
2. `crcli vault state` — vault je `Locked`
3. Sačekati završetak ili prekinuti poslove — `crcli jobs cancel` ⚠️
4. Po potrebi pauzirati raspored politika **kroz CR**, ne kroz DD
5. Provera Support Matrix-a ako je u pitanju nadogradnja DDOS-a
6. Tek onda raditi na DD-u — poglavlja 03 i 04
7. Nakon radova: `crcli dd list`, `crcli vault state`, pa pustiti jednu
   politiku ručno i pratiti `crcli jobs watch`

⛔ **Nikada ne isključujte replikacione kontekste ručno "da ne smetaju".**

---

## 12.16 Dnevna provera CR vault-a

```
# Na CR management hostu
crcli vault state                           # očekivano: Locked
crcli jobs list                             # da li su noćni poslovi prošli
crcli alerts list
crcli policy list
crcli policy list-copy --policyname <ime>   # ima li nove kopije, je li zaključana
crcli system clean --show                   # radi li čišćenje starih kopija
crcli system drbackup --show                # je li DR backup aktivan

# Na vault DD-u
alerts show current
filesys show space
mtree list
filesys clean status

# Na produkcionom DD-u
alerts show current
replication status
```

**Crvene zastavice:**

| Nalaz | Značenje |
|---|---|
| `vault state` = `Unlocked` bez aktivnog posla | Veza otvorena kad ne treba — PB-14 |
| `vault state` = `Secured` a niko nije obavestio | Neko je pritisnuo crveno dugme |
| Nema nove kopije od poslednjeg termina | Politika ili raspored ne radi — PB-15 |
| Kopija `Unlocked` a politika ima Lock akciju | Lock korak pada — proveriti RL licencu na vault DD-u |
| `lastanalysisstatus` ≠ `Good` | CyberSense nalaz — PB-16 |
| `crcli system clean --show` nije podešen | Kopije se gomilaju |
| `crcli system drbackup --show` isključen | Gubitak CR hosta = gubitak svih politika |
| Vault DD iznad 85% | Vidi 12.10 — planirati odmah |

---

## 12.17 Checklist za CR okruženje

- [ ] Vault DD nedostupan sa produkcione mreže osim replikacionim putem
- [ ] Nalozi na vault DD-u zasebni; lozinke se ne dele sa produkcijom
- [ ] `crso` lozinka i **lockbox passphrase pohranjeni van vault-a i van produkcije**
      (izgubljen passphrase se ne može povratiti — traži reinstalaciju)
- [ ] Retention Lock licenca na vault DD-u i politike koriste Lock akciju
- [ ] `crcli system clean` podešen — inače se kopije gomilaju
- [ ] `crcli system drbackup` uključen i proveren
- [ ] Kapacitet vault DD-a izračunat za broj kopija × dužinu lock-a
- [ ] Rasporedi politika definisani i verifikovano da rade
- [ ] Alerti iz CR-a idu ka timu, ne samo u UI
- [ ] Eksterni syslog konfigurisan (**poglavlje 03**)
- [ ] Procedura oporavka napisana i **testirana** u sandbox-u
- [ ] Recovery check zakazan gde je podržan
- [ ] Definisano ko sme `crcli vault release` (samo Security Admin) i po kojoj proceduri
- [ ] Definisano šta se radi kada CyberSense prijavi nalaz (PB-16)

---

## Reference

Sva CR dokumentacija je na Dell Support portalu i traži Dell Online Support login.

- **Cyber Recovery 20.3 Command-Line Interface Reference Guide** — kompletan CRCLI
- **Cyber Recovery Product Guide** — koncepti, politike, stanja vault-a, oporavak
- **Cyber Recovery Installation Guide** — `crsetup.sh`, portovi, preduslovi
- **Cyber Recovery Simple Support Matrix** — kompatibilnost CR verzije sa DDOS verzijom
- White paper-i za oporavak sa konkretnim backup aplikacijama na Dell Info Hub-u

> ⚠️ **Kompatibilnost:** CR verzija i DDOS verzija na vault DD-u moraju biti
> usklađene po Support Matrix-u. Nadogradnja DDOS-a na vault DD-u bez provere
> matrice je čest uzrok da CR politike prestanu da rade.
