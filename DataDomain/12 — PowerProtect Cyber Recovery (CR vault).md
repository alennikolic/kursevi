# 12 — PowerProtect Cyber Recovery (CR vault)

Cyber Recovery nije "još jedna replikacija na DR lokaciju". To je izolovano
okruženje sa kontrolisanim, povremenim otvaranjem veze ka produkciji, u kome se
prave nepromenljive kopije podataka koje se mogu analizirati i iz kojih se
može oporaviti nakon napada.

Ovo poglavlje je pisano iz ugla **DD inženjera** — šta se vidi na DD uređajima,
šta se sme a šta ne sme dirati, i minimum CRCLI-ja za dijagnostiku.

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
| Kopije | Odredišni MTree prati izvor | **PIT kopije** (point-in-time), zaključane Retention Lock-om |
| Analiza sadržaja | Nema | CyberSense (opciono) — detekcija tragova napada |
| Upravljanje | DD CLI / DDSM | CR UI ili CRCLI; DD je ispod, ali se ne administrira direktno |

Ključna razlika: kod DR replikacije napadač koji kompromituje produkcioni DD može
pratiti replikaciju do DR uređaja. Kod CR vault-a veza je zatvorena veći deo
vremena, vault DD nema pristup sa produkcione mreže, a kopije su zaključane.

---

## 12.2 Komponente

| Komponenta | Uloga |
|---|---|
| **Produkcioni DD** | Izvor. Standardni DD sa MTree-jevima koji se štite |
| **Vault DD** | Odredište. Nalazi se u izolovanoj mreži. Na njemu žive replicirani MTree-jevi i PIT kopije |
| **CR management host** | Server (fizički, VM ili OVA appliance) u vault-u na kome radi CR softver. Orkestrira sve |
| **CyberSense** (opciono) | Zaseban host koji analizira PIT kopije i traži tragove napada |
| **Recovery host / sandbox** | Okruženje u vault-u za testni ili stvarni oporavak |

CR softver radi kao skup Docker kontejnera na management hostu.
Binarni fajlovi su u `/opt/dellemc/cr/bin/` (`crcli`, `cradmin`, `crsetup.sh`).

Pristup:
- **CR UI:** `https://<cr-management-host>:14777`
- **CRCLI:** SSH na management host, pa `crcli`

Uloge: **crso** (Cyber Recovery Security Officer — kreira se pri instalaciji,
jedini može određene operacije) i **admin** korisnici. Postoji i `dashboard`
uloga samo za pregled.

---

## 12.3 Stanja vault-a

Ovo je prvo što se proverava kada neko pita "da li je vault u redu".

| Stanje | Značenje |
|---|---|
| **Locked** | **Normalno stanje.** Veza ka produkciji je zatvorena. Podaci ne ulaze |
| **Unlocked** | Veza je otvorena jer je u toku sync operacija. Normalno tokom sync prozora |
| **Secured** | Vault je ručno zaključan zbog sumnje na napad. **Sve Sync operacije staju i nove se ne mogu pokrenuti.** Ne-Sync politike (Copy, Lock, Analyze) i dalje rade |
| **Degraded** | Nešto u okruženju ne radi kako treba — proveriti alerte i događaje |

```
crcli vault state
```

Primer izlaza:

```
The current state of the vault is: Locked
The vault has been in unlocked state for: 0 seconds
```

> **Ako je vault dugo u `Unlocked` stanju bez aktivnog sync posla — to je incident.**
> Otvorena veza je upravo ono što CR treba da spreči. Proverite `crcli jobs list`
> i alerte.
>
> Izuzetak: neposredno nakon instalacije i inicijalne konfiguracije vault može
> biti otključan dok traje prva inicijalizacija replikacije. Port se zatvara
> automatski kada se inicijalizacija završi.

---

## 12.4 Pogled sa DD strane

### Na produkcionom DD-u

Izgleda kao običan izvor MTree replikacije:

```
replication show config
replication status
mtree list
```

Razlika je što ćete videti kontekst koji je veći deo vremena `disabled`.
To je CR softver koji ga isključuje nakon svakog sync prozora.

### Na vault DD-u

```
mtree list
replication show config
replication status
filesys show space
mtree retention-lock status mtree /data/col1/<mtree>
```

Šta ćete videti:

- **Replicirani MTree-jevi** u `RD` stanju — odredišta replikacije sa produkcije
- **PIT kopije** — zasebni MTree-jevi koje pravi CR softver, sa imenima
  po šablonu `cr-copy-<politika>-<timestamp>`. Nastaju `fastcopy` operacijom,
  pa ne troše prostor kao pune kopije — dele segmente sa izvornim MTree-jem
- **Retention Lock uključen** na MTree-jevima sa kopijama
- **Replikacioni kontekst u `disabled` stanju** veći deo vremena

---

## 12.5 Šta se NE radi na vault DD-u

Ovo je sekcija zbog koje ovo poglavlje postoji. Sve navedeno su stvari koje
DD inženjer instinktivno uradi, a koje razbijaju CR okruženje.

| ⛔ Ne radi se | Zašto |
|---|---|
| `replication enable <ctx>` na kontekstu koji je `disabled` | Kontekst je namerno isključen. Ručno uključivanje otvara vezu ka produkciji van sync prozora i ruši air gap. Replikaciju uključuje isključivo CR softver |
| `replication break` / `replication del` | Raskida vezu koju CR softver očekuje. CR politike počinju da padaju |
| `mtree delete` nad `cr-copy-*` MTree-jevima | To su PIT kopije kojima CR upravlja. Brisanje se radi kroz `crcli policy delete-copy`, ne kroz DD |
| `mtree retention-lock disable` / `revert` | Ruši imutabilnost koja je čitava svrha vault-a. U Compliance režimu ni ne može |
| Otvaranje dodatnih mrežnih puteva ka vault DD-u | Vault DD sme da priča samo sa CR management hostom i, tokom sync prozora, sa produkcionim DD-om |
| Kreiranje NFS/CIFS export-a ka produkcionoj mreži | Isto — probija izolaciju |
| Reboot vault DD-a bez provere CR poslova | Sync ili lock posao u toku pada; politika ostaje u nekonzistentnom stanju |
| Promena lozinke DD naloga koji koristi CR softver | CR gubi pristup storage-u. Lozinka se menja kroz CR (`crcli dd modify`), ne samo na DD-u |

**Ako mislite da nešto na vault DD-u treba popraviti — prvo `crcli jobs list`
i `crcli alerts list`.** Skoro uvek je odgovor u CR-u, ne na DD-u.

---

## 12.6 CRCLI — osnove

SSH na CR management host, pa:

```bash
/opt/dellemc/cr/bin/crcli login          # prijava (crso ili admin nalog)
crcli --help                             # pomoć
crcli <komanda> --help                   # pomoć za komandu
crcli logout
```

Većina komandi podržava `--json`, što je korisno za skripte i monitoring.

> Sintaksa i skup komandi se razlikuju između CR verzija (19.10 → 19.20+).
> Uvek proverite `crcli <komanda> --help` na svom sistemu pre nego što
> kopirate komandu iz ovog priručnika.

Provera verzije i stanja celog okruženja:

```
crcli system details
```

Vraća verzije CR servisa, OS, Docker, bazu, verziju DDOS-a na vault DD-u
i verzije CyberSense hostova. To je prva komanda za support case.

---

## 12.7 Politike i akcije

**Politika** je definicija: koji MTree se štiti, sa kog DD-a, na koji vault DD,
šta se radi sa podacima i po kom rasporedu.

### Akcije

| Akcija | Šta radi |
|---|---|
| **Sync** | Replicira MTree sa produkcije u vault. Otvara vezu, prenese, zatvori |
| **Copy** | Pravi PIT kopiju već repliciranog MTree-ja u vault-u |
| **Sync Copy** | Sync pa odmah Copy — jedna operacija |
| **Copy Lock** | Retention Lock nad svim fajlovima u PIT kopiji |
| **Secure Copy** | Sync + Copy + Lock. **Ovo je akcija koja se najčešće zakazuje** |
| **Analyze** | CyberSense analiza PIT kopije |
| **Secure Copy Analyze** | Sve zajedno, uključujući analizu |

> Ne mogu se pokretati istovremene Sync ili Lock akcije za istu politiku.
> Copy akcije mogu paralelno.

### Komande

```
crcli policy list                                    # sve politike
crcli policy show --policyname <ime>                 # detalji politike
crcli policy run --policyname <ime> ...              # ⚠️ ručno pokretanje
crcli policy list-copy --policyname <ime>            # sve PIT kopije politike
crcli policy list-copy --policyname <ime> --copyname <kopija>
crcli policy delete-copy --policyname <ime> --copyname <kopija>    # 🛑
crcli policy add ...                                 # ⚠️
crcli policy modify ...                              # ⚠️
crcli policy delete --policyname <ime>               # 🛑
```

`crcli policy list-copy` je najkorisnija read-only komanda u ovom poglavlju.
Za svaku kopiju daje ime, datum nastanka, do kada je zaključana i status
zaključavanja, a za analizirane kopije i rezultat CyberSense analize
(`lastanalysisstatus`).

**Šta se tu gleda:**
- `Lock Status` = `Unlocked` na kopiji koja je trebalo da bude zaključana → politika nije odradila Lock korak
- `lastanalysisstatus` različit od `Good` → CyberSense je našao nešto; ide se u proceduru za sumnju na kompromitovane podatke
- Nema novih kopija u očekivanom ritmu → raspored ne radi, proveriti `crcli jobs list`

Izveštaji analize:

```
crcli policy analysis-report download --policyname <ime> ...
crcli policy analysis-report email --policyname <ime> ...
```

---

## 12.8 Poslovi (jobs)

```
crcli jobs list                                      # svi poslovi
crcli jobs list -t protection -running               # samo aktivni zaštitni poslovi
crcli jobs show --jobname <ime>
crcli jobs watch --jobname <ime>                     # praćenje uživo
crcli jobs cancel --jobname <ime> [--watch 5]        # ⚠️
```

Posao prolazi kroz korake (`Step 1 of 2: sync`, `Step 2 of 2: copy`...).
Kod `jobs cancel` u detaljima se vidi i korak "Disabling replication context" —
to je CR koji zatvara vezu ka produkciji. Isto se dešava i pri normalnom
završetku posla.

**Prilikom dijagnostike sporog sync-a:** posao će stajati na koraku `sync`
sa procentom koji sporo raste. To je replikacija na DD nivou — proverite
i sa DD strane:

```
replication status
replication show performance
net show stats
```

Ali **ne dirajte kontekst** — samo gledajte. Vidi poglavlje 07 i 08.

---

## 12.9 Alerti i događaji

```
crcli alerts list
crcli alerts show --alertid <id>
crcli alerts acknowledge --alertid <id>
crcli alerts unacknowledge --alertid <id>
crcli events list
crcli events show --eventid <id>
```

CR alerti su odvojeni od DD alerta. Za punu sliku treba pogledati oba:

```
crcli alerts list          # na CR management hostu
alerts show current        # na vault DD-u
alerts show current        # na produkcionom DD-u
```

---

## 12.10 Storage i aplikacije registrovane u CR-u

```
crcli dd list                            # DD sistemi poznati CR-u
crcli dd show --name <ime>
crcli dd modify ...                      # ⚠️ npr. izmena kredencijala
crcli apps list                          # CyberSense i drugi hostovi
crcli apps show --name <ime>
crcli license show
```

`crcli dd list` pokazuje da li CR uopšte može da priča sa DD uređajima.
Ako politika pada odmah po pokretanju, ovo je prva provera.

---

## 12.11 Ručno obezbeđivanje vault-a (Secure / Release)

Ovo je "crveno dugme" — koristi se kada postoji sumnja na napad na produkciju
i želite da se ništa više ne replicira u vault.

```
crcli vault state                        # provera trenutnog stanja
crcli vault secure                       # 🛑 zaustavlja sve Sync operacije
crcli vault release                      # 🛑 vraća u normalno stanje (samo crso)
```

Kada je vault `Secured`:
- sve Sync operacije se **odmah zaustavljaju**
- nove Sync operacije se **ne mogu pokrenuti**
- CR podiže alert da je vault obezbeđen
- ne-Sync politike (Copy, Lock, Analyze) i dalje rade — možete analizirati
  i oporavljati iz postojećih kopija

`crcli vault release` može izvršiti **samo security officer (crso)** i traži
interaktivnu potvrdu. Radi se tek kada se potvrdi da pretnja više ne postoji.

Ručno zatvaranje i otvaranje veze (bez ulaska u `Secured` stanje) rade
security officer ili admin:

```
crcli vault lock
crcli vault unlock
```

> Proverite tačan skup `vault` podkomandi na svojoj verziji sa
> `crcli vault --help`. Skup se menjao kroz CR verzije.

---

## 12.12 Kapacitet vault-a

Vault DD se puni brže nego što ljudi očekuju, iz dva razloga:

1. **PIT kopije se prave `fastcopy` operacijom** i u početku ne troše skoro
   ništa — dele segmente sa izvornim MTree-jem. Ali kako se produkcioni podaci
   menjaju, svaka kopija sve više drži sopstvene, jedinstvene segmente.
2. **Kopije su zaključane Retention Lock-om.** Cleaning ih **ne može** osloboditi
   dok ne istekne rok, čak i kad ih obrišete kroz CR.

To znači da je kapacitet vault DD-a funkcija **broja kopija × retencije lock-a**,
a ne trenutne veličine produkcionih podataka.

Provera:

```
# Na vault DD-u
filesys show space
mtree show compression
mtree list
filesys clean status

# U CR-u
crcli policy list-copy --policyname <ime>
```

Ako se vault puni:

1. `crcli policy list-copy` po svim politikama — koliko kopija postoji i do kada su zaključane
2. Obrisati kopije kojima je lock istekao — `crcli policy delete-copy` 🛑
3. Pokrenuti cleaning na vault DD-u — `filesys clean start` ⚠️
4. Ako su kopije još zaključane — **prostor se ne može osloboditi**.
   Ide se na reviziju retencije u politikama ili na proširenje kapaciteta.

> Revidiranje broja kopija i dužine lock-a radi se **u politici, unapred**.
> Nema retroaktivnog skraćivanja lock-a. Vidi poglavlje 06.

---

## 12.13 Oporavak

```
crcli recovery list-sandboxes
crcli policy list-copy --policyname <ime>          # izbor kopije za oporavak
```

Osnovni scenariji:

- **Oporavak unutar vault-a (sandbox)** — kopija se otvori u izolovanom
  okruženju u vault-u i tamo se aplikacija podigne i proveri. Ništa ne napušta vault
- **Alternate recovery** — podaci se repliciraju iz vault-a na alternativni
  DD sistem, odakle backup aplikacija radi standardni restore
- **Povratak u produkciju** — tek nakon što je produkcija očišćena i potvrđeno
  je da je kopija zdrava (CyberSense `Good`)

> **Oporavak nije DD operacija.** Ne radi se `replication resync` sa vault DD-a
> ka produkciji ručno. Ide se kroz CR politiku ili kroz proceduru backup aplikacije.
> Ručna replikacija iz vault-a nazad može preneti kompromitovane podatke i
> otvoriti vezu koja treba da bude zatvorena.

Za konkretnu backup aplikaciju (NetWorker, NetBackup, PPDM, Avamar, Veeam)
postoje zasebni Dell white paper-i sa korak-po-korak procedurom oporavka
iz vault-a. **Procedura oporavka mora biti napisana i testirana pre incidenta**,
ne improvizovana tokom njega.

---

## 12.14 Planirani radovi na DD-u u vault-u

Kada treba uraditi upgrade, reboot ili hardversku intervenciju na vault DD-u:

1. `crcli jobs list -t protection -running` — proveriti da nema aktivnih poslova
2. `crcli vault state` — potvrditi da je vault `Locked` (veza zatvorena)
3. Sačekati završetak ili prekinuti tekuće poslove — `crcli jobs cancel` ⚠️
4. Po potrebi pauzirati raspored politika kroz CR (ne kroz DD)
5. Tek onda raditi na DD-u — poglavlja 03 i 04
6. Nakon radova: `crcli dd list`, `crcli vault state`, pa pustiti jednu
   politiku ručno i pratiti `crcli jobs watch`

**Nikada ne isključujte replikacione kontekste ručno "da ne smetaju" tokom radova.**
CR će ih naći u neočekivanom stanju i politike će padati.

---

## 12.15 Dnevna provera CR vault-a

```
# Na CR management hostu
crcli vault state                        # očekivano: Locked
crcli jobs list                          # da li su noćni poslovi prošli
crcli alerts list                        # ima li novih alerta
crcli policy list                        # sve politike aktivne
crcli policy list-copy --policyname <ime>   # ima li nove kopije, je li zaključana

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
| `vault state` = `Unlocked` bez aktivnog posla | Veza otvorena kad ne treba — istražiti odmah |
| `vault state` = `Secured` a niko nije obavestio | Neko je pritisnuo crveno dugme |
| Nema nove kopije od poslednjeg zakazanog termina | Politika ili raspored ne radi |
| Kopija `Unlocked` a politika ima Lock akciju | Lock korak pada — proveriti RL licencu na vault DD-u |
| `lastanalysisstatus` ≠ `Good` | CyberSense je nešto našao — procedura za sumnju na kompromitaciju |
| Vault DD iznad 85% | Vidi 12.12 — planirati odmah, jer se prostor ne oslobađa lako |

---

## 12.16 Checklist za CR okruženje

- [ ] Vault DD nije dostupan sa produkcione mreže ni na koji način osim replikacionim putem
- [ ] Nalozi na vault DD-u su zasebni; lozinke se ne dele sa produkcijom
- [ ] `crso` lozinka i lockbox passphrase pohranjeni van vault-a i van produkcije
      (izgubljen lockbox passphrase se **ne može** povratiti — traži reinstalaciju)
- [ ] Retention Lock licenca postoji na vault DD-u i politike koriste Lock akciju
- [ ] Kapacitet vault DD-a izračunat za broj kopija × dužinu lock-a
- [ ] Rasporedi politika definisani i verifikovano da rade
- [ ] Alerti iz CR-a idu ka timu, ne samo u UI
- [ ] Procedura oporavka napisana i **testirana** u sandbox-u
- [ ] Definisano ko sme da uradi `crcli vault release` i po kojoj proceduri
- [ ] Definisano šta se radi kada CyberSense prijavi nalaz

---

## Reference

Sva CR dokumentacija je na Dell Support portalu i traži Dell Online Support login.
Verzije se označavaju kao 19.x (npr. 19.14, 19.16, 19.20).

- **Cyber Recovery Product Guide** — koncepti, politike, stanja vault-a, oporavak
- **Cyber Recovery Command-Line Interface Reference Guide** — kompletan CRCLI
- **Cyber Recovery Installation Guide** — `crsetup.sh`, portovi, preduslovi
- **Cyber Recovery Simple Support Matrix** — kompatibilnost CR verzije sa DDOS verzijom
- White paper-i za oporavak sa konkretnim backup aplikacijama na
  Dell Technologies Info Hub-u

> ⚠️ **Kompatibilnost:** CR verzija i DDOS verzija na vault DD-u moraju biti
> usklađene po Support Matrix-u. Nadogradnja DDOS-a na vault DD-u bez provere
> matrice je čest uzrok da CR politike prestanu da rade.
