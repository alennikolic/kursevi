# Pun vSAN datastore i pad vCentera — dijagnostika i oporavak

> **Tip dokumenta:** incident runbook
> **Scenario:** vSAN datastore je popunjen, I/O staje ili je usporen, vCenter (VCSA) ne radi
> **Verzije:** ESXi / vSAN 6.7, 7.0 (U1–U3), 8.0 (U1–U3), OSA i ESA
> **Pristup:** SSH ili DCUI na ESXi hostove, root

---

## Sadržaj

1. [Prvih 15 minuta — stabilizacija](#1-stabilizacija)
2. [Zašto pun vSAN zaključava sam sebe](#2-zasto)
3. [Dijagnostika — koliko je zaista puno](#3-dijagnostika)
4. [Šta troši prostor](#4-sta-trosi)
5. [Zaustavljanje krvarenja — rebuild i resync](#5-zaustavljanje)
6. [Oslobađanje prostora, po redosledu rizika](#6-oslobadjanje)
7. [Migracija na drugi datastore](#7-migracija)
8. [Oporavak vCentera](#8-vcenter)
9. [Posle oporavka](#9-posle)
10. [Sažeti checklist](#10-checklist)

---

<a name="1-stabilizacija"></a>
## 1. Prvih 15 minuta — stabilizacija

### 1.1 Šta NE raditi

| Ne radi | Zašto |
|---|---|
| Ne pali nove VM-ove | Svaki upaljen VM kreira swap objekat veličine RAM minus rezervacija |
| Ne pravi snapshot-ove | Delta objekat troši prostor, a konsolidacija kasnije troši još više |
| Ne restartuj hostove | VM-ovi se možda neće upaliti nazad — nema prostora za swap |
| Ne ulazi u maintenance mode sa „Full data migration" | Evakuacija podataka zahteva prostor na preostalim hostovima |
| Ne uklanjaj disk grupe / diskove iz pool-a | Kapacitet koji ti fali |
| Ne pokreći Storage vMotion na isti vSAN | Privremeno duplira objekat |
| Ne brisi disk grupu da bi je „prekreirao" | Nepovratan gubitak komponenti |
| Ne diraj `objtool delete` dok ne pročitaš 6.5 | Nepovratno brisanje objekata |

### 1.2 Šta uraditi odmah

**Korak 1 — Utvrdi da li je I/O još uvek živ.**

```
esxcli vsan debug disk summary get
df -h | grep -i vsan
```

Ako je bilo koji **pojedinačni disk** preko 95%, klaster je već u congestion stanju bez obzira što prosek izgleda bolje.

**Korak 2 — Zamrzni rebuild.** Na **svakom** hostu:

```
esxcfg-advcfg -g /VSAN/ClomRepairDelay
esxcfg-advcfg -s 1440 /VSAN/ClomRepairDelay
/etc/init.d/clomd restart
```

Podrazumevanih 60 minuta znači da će klaster, ako je neki host ili disk ispao, krenuti u pun rebuild i potrošiti ono malo prostora što ti je ostalo. 1440 = 24 sata, dovoljno za intervenciju.

Zabeleži originalnu vrednost — vraćaš je na kraju.

**Korak 3 — Ugasi sve što nije kritično.**

```
vim-cmd vmsvc/getallvms
vim-cmd vmsvc/power.getstate <vmid>
vim-cmd vmsvc/power.shutdown <vmid>
```

Ako guest ne odgovara na graceful shutdown:
```
vim-cmd vmsvc/power.off <vmid>
```

Ovo je najveći i najbrži izvor prostora. Ugašen VM oslobađa swap objekat i prestaje da upisuje.

**Korak 4 — Zapiši stanje pre nego što bilo šta menjaš.**

Sa svakog hosta, u tekstualni fajl van vSAN-a:
```
esxcli vsan cluster get
esxcli vsan debug disk summary get
esxcli vsan debug object health summary get
esxcli vsan debug limit get
esxcli vsan debug resync summary get
esxcli vsan storage list
```

Za ESA umesto poslednje:
```
esxcli vsan storagepool list
```

**Korak 5 — Proveri vreme.** Drift između hostova pravi dodatne CMMDS probleme usred incidenta.

```
date -u
esxcli system ntp get
```

> Na 6.7 nema `esxcli system ntp` — koristi `esxcli system time get` i `/etc/init.d/ntpd status`.

---

<a name="2-zasto"></a>
## 2. Zašto pun vSAN zaključava sam sebe

Razumevanje ovog dela ti štedi sate pogrešnih poteza.

### 2.1 Mehanizam

vSAN nije klasičan fajl sistem. Objekti su raspoređeni u komponente po diskovima, a svaki upis ide prvo u cache/log sloj pa se destage-uje na capacity sloj. Kada capacity disk nema mesta:

1. Destage staje
2. Log sloj se puni
3. LSOM podiže **congestion** vrednost
4. Congestion usporava dolazni I/O (namerno, da bi se sistem stabilizovao)
5. Kad congestion dostigne maksimum, I/O praktično staje
6. Guest OS dobija timeout-e, fajl sistemi prelaze u read-only

### 2.2 Zašto ne možeš samo da obrišeš nešto

Deo operacija oslobađanja prostora **sam zahteva prostor**:

| Operacija | Traži prostor? | Bezbedno pri punom DS |
|---|---|---|
| Brisanje VM-a / objekta | Ne (dealokacija metapodataka) | **Da** |
| Brisanje fajla iz namespace-a | Minimalno | Da |
| Gašenje VM-a | Ne | Da |
| Konsolidacija snapshot-a | **Da**, ponekad značajno | **Ne** |
| Storage vMotion na isti DS | Da, duplira | Ne |
| Klon / template deploy | Da | Ne |
| Rebuild / resync | Da | Ne |
| Promena politike (npr. FTT 1→2) | Da | Ne |
| Evakuacija hosta (full data migration) | Da | Ne |

Dakle: **brisanje da, konsolidacija ne.** To određuje redosled u poglavlju 6.

### 2.3 Cirkularna zavisnost sa vCenterom

Ovaj scenario ima dupli zaključ:

- VCSA je na vSAN-u → nema prostora → VCSA ne može da piše → vpxd staje
- Bez vCentera → nema SPBM-a → ne možeš da smanjiš rezervacije politikom
- Bez vCentera → nema UI-ja za brisanje snapshot-ova i migraciju
- Bez prostora → ne možeš da deploy-uješ novi VCSA na vSAN

Izlaz je uvek isti: **oslobodi prostor sa hosta preko CLI-ja, pa tek onda vraćaj vCenter.**

### 2.4 Dve različite „VCSA je pukao" situacije

Razdvoji ih odmah, tretman je različit:

**A) VCSA VM ne može da piše jer je vSAN pun.**
Simptom: VCSA je upaljen, ali servisi padaju, log-in ne radi, `/storage/*` prijavljuje I/O greške. Rešenje je oslobađanje vSAN prostora (poglavlje 6). VCSA se oporavlja sam ili uz restart servisa.

**B) VCSA-ini interni diskovi su puni.**
Simptom: vSAN ima prostora ili si ga upravo oslobodio, ali VCSA i dalje ne radi. Uzrok je unutar appliance-a — `/storage/seat`, `/storage/log`, `/storage/db`, `/storage/core`. Rešenje je u poglavlju 8.

Često su prisutne obe. Prvo A, pa B.

---

<a name="3-dijagnostika"></a>
## 3. Dijagnostika — koliko je zaista puno

### 3.1 Zbirni kapacitet

```
df -h
```

Traži red sa `vsanDatastore`. Ovo je gruba slika i **vara** — u OSA sa dedup/compression prikazani brojevi ne odgovaraju fizičkom stanju diskova.

### 3.2 Po disku — ovo je merodavno

```
esxcli vsan debug disk summary get
esxcli vsan debug disk list
```

`disk summary get` daje tabelu po uređaju sa iskorišćenošću, health i congestion vrednostima. **Ovo je najvažnija komanda u celom dokumentu.**

Klaster može biti na 72% proseka, a jedan capacity disk na 98% — i taj jedan disk obara sve objekte koji imaju komponentu na njemu.

Šta tražiš:
- Bilo koji disk preko **90%** → visok rizik
- Bilo koji disk preko **95%** → congestion je verovatno već aktivan
- Razlika između najpunijeg i najpraznijeg diska preko **20 procentnih poena** → imbalans

Detaljnije, iz CMMDS-a:
```
cmmds-tool find -t DISK_USAGE -f json
cmmds-tool find -t DISK -f json
```

### 3.3 Congestion u logovima

```
grep -i congestion /var/log/vmkernel.log | tail -50
grep -iE "LSOMLogCongestion|Congestion threshold" /var/log/vmkernel.log | tail -30
```

Tipovi congestion-a i šta znače:

| Tip | Uzrok |
|---|---|
| **Log congestion** | log sloj se ne prazni dovoljno brzo — često direktna posledica punog capacity sloja |
| **Memory congestion** | nedostatak RAM-a za vSAN strukture |
| **SSD congestion** | cache disk pun ili spor destage |
| **Slab congestion** | iscrpljeni interni bufferi |
| **Component congestion** | previše komponenti po disku |

U ovom scenariju očekuješ log i SSD congestion.

Ostale relevantne poruke:
```
grep -iE "No space left|out of space|ENOSPC|disk full" /var/log/vmkernel.log | tail -50
grep -iE "Lost access to volume|permanent error" /var/log/vmkernel.log | tail -30
```

### 3.4 Limit komponenti

```
esxcli vsan debug limit get
```

Prekoračen broj komponenti po hostu se manifestuje **kao da nema prostora**, iako kapaciteta ima. Ako si blizu limita (na modernim verzijama oko 9000 po hostu), rešenje nije brisanje podataka nego smanjenje broja komponenti — manji stripe width, konsolidacija malih objekata.

### 3.5 Zdravlje objekata

```
esxcli vsan debug object health summary get
```

Kod punog datastore-a tipično vidiš kombinaciju:
- `inaccessible` — objekti bez kvoruma
- `reducedavailabilitywithnorebuild` — degradirani, rebuild ne može da krene jer nema prostora
- `nonavailabilityrelatedincompliance` — ne poštuju politiku

**Ako imaš `reducedavailabilitywithnorebuild`, to je potvrda da je prostor uzrok**, a ne posledica.

### 3.6 Resync u toku

```
esxcli vsan debug resync summary get
esxcli vsan debug resync list
```

Aktivan resync na punom datastore-u je najgori mogući scenario — troši prostor koji ti treba. Ako vidiš resync u toku, prioritet je poglavlje 5.

### 3.7 Slack space i rezerve

vSAN nikad ne sme da radi na 100%. Potreban je prostor za rebuild, resync, snapshot-ove i interne operacije.

- **Pre 7.0 U1:** pravilo je bilo ostaviti ~25–30% slobodno. Nije bilo mehanizma koji to sprovodi.
- **7.0 U1 i noviji:** uvedeni su *Operations Reserve* i *Host Rebuild Reserve*, koji rezervišu prostor i sprečavaju da ga korisnički podaci pojedu. **Konfigurišu se isključivo iz vCentera** — bez njega ne možeš da ih uključiš ni isključiš.

Ako su rezerve bile uključene i datastore je i dalje pun, znači da su rezerve već probijene ili su bile isključene.

### 3.8 Popis objekata

```
esxcli vsan debug object list --all
esxcli vsan debug vmdk list
```

`vmdk list` ti daje mapiranje VMDK → objekat → veličina, što je osnova za odluku šta brisati i šta seliti.

Za pregled direktorijuma na datastore-u:
```
/usr/lib/vmware/osfs/bin/osfs-ls /vmfs/volumes/vsanDatastore/
ls -la /vmfs/volumes/vsanDatastore/
```

---

<a name="4-sta-trosi"></a>
## 4. Šta troši prostor

Prođi kroz listu redom — obično se nađe više uzroka odjednom.

### 4.1 Object Space Reservation (thick provisioning)

Politika sa `objectSpaceReservation: 100` rezerviše pun kapacitet diska bez obzira što guest koristi 10%.

Provera po objektu:
```
esxcli vsan debug object list -u <object-uuid>
```
Traži `proportionalCapacity` u policy bloku (100 = thick).

Podrazumevana politika hosta:
```
esxcli vsan policy getdefault
```

> Menjanje ovoga bez vCentera ima ozbiljno ograničenje — vidi 6.6.

### 4.2 Swap objekti

Svaki upaljen VM ima `.vswp` objekat veličine **konfigurisani RAM minus rezervacija memorije**. VM sa 64 GB RAM-a i bez rezervacije troši 64 GB swap prostora.

Provera da li je swap thin:
```
esxcli system settings advanced list -o /VSAN/SwapThickProvisionDisabled
```
Vrednost `1` znači da je swap thin (podrazumevano od 6.7). Ako je `0`, svaki upaljen VM ti jede pun RAM u prostoru.

**Gašenje VM-a odmah oslobađa ovaj prostor.**

Zaostali swap objekti od VM-ova koji su bili na hostu koji je pao — vidi 6.5.

### 4.3 Snapshot-ovi

Najčešći tihi krivac. Zaboravljeni snapshot od pre 8 meseci može biti veći od baznog diska.

```
vim-cmd vmsvc/get.snapshotinfo <vmid>
ls -la /vmfs/volumes/vsanDatastore/<VM>/ | grep -i delta
```

Prođi kroz sve VM-ove. Traži `-000001.vmdk`, `-000002.vmdk` itd.

> **Ne brisi ih odmah.** Konsolidacija traži prostor. Prvo oslobodi prostor drugim putem, pa tek onda konsoliduj. Vidi 6.7.

### 4.4 ISO fajlovi, template-i, zaostali folderi

```
ls -la /vmfs/volumes/vsanDatastore/
find /vmfs/volumes/vsanDatastore/ -name "*.iso" -exec ls -lh {} \;
```

ISO biblioteke na vSAN datastore-u su čest i potpuno nepotreban trošak. Ovo je najbezbednija stvar za brisanje.

Traži i foldere VM-ova koji više ne postoje u inventaru — uporedi `ls` sa `vim-cmd vmsvc/getallvms`.

### 4.5 vSAN File Services i iSCSI

Ako su uključeni, njihovi objekti se ne vide kao VM-ovi:
```
esxcli vsan debug object list --all
esxcli vsan iscsi target list
esxcli vsan iscsi homeobject get
```

File Services kreira i sopstvene agent VM-ove po hostu.

### 4.6 Dedup i compression overhead (samo OSA)

```
esxcli vsan storage list
```
Traži `Deduplication: true` / `Compression: true`.

Kod uključenog dedup-a, **pun disk je znatno opasniji**: dedup radi na nivou disk grupe, a kada nestane prostora hash tabela ne može da se ažurira. Takođe, uklanjanje jednog diska iz grupe sa dedup-om znači evakuaciju **cele grupe**.

Ako je dedup uključen i datastore je pun, budi posebno konzervativan — ne diraj disk grupe.

### 4.7 Neoslobođen prostor iz guest OS-a (UNMAP)

Guest je obrisao podatke, ali vSAN to ne zna. Prostor ostaje zauzet.

TRIM/UNMAP podrška postoji od 6.7, ali se **uključuje na nivou klastera preko vCentera/RVC-a**. Bez vCentera ne možeš je uključiti. Zabeleži kao stavku posle oporavka (poglavlje 9).

### 4.8 Zaostali („zombie") objekti

Objekti koji ne pripadaju nijednom registrovanom VM-u — ostatak neuspelih kloniranja, migracija, ili pada hosta.

Identifikuju se poređenjem:
```
esxcli vsan debug object list --all
vim-cmd vmsvc/getallvms
/usr/lib/vmware/osfs/bin/osfs-ls /vmfs/volumes/vsanDatastore/
```

Objekat čiji `Directory Name` ne odgovara nijednom registrovanom VM-u je kandidat. **Kandidat, ne dokaz** — vidi 6.5 pre brisanja.

---

<a name="5-zaustavljanje"></a>
## 5. Zaustavljanje krvarenja — rebuild i resync

Pre nego što počneš da oslobađaš prostor, moraš zaustaviti procese koji ga troše. U suprotnom oslobodiš 200 GB i za 20 minuta ih rebuild pojede.

### 5.1 Odloži rebuild

Na svakom hostu:
```
esxcfg-advcfg -g /VSAN/ClomRepairDelay
esxcfg-advcfg -s 1440 /VSAN/ClomRepairDelay
/etc/init.d/clomd restart
```

Restart `clomd`-a je obavezan da bi promena stupila na snagu. Restart clomd-a ne prekida I/O.

> Ovo **odlaže** rebuild, ne otkazuje ga. Komponente ostaju ABSENT dok ne istekne novi tajmer ili dok ručno ne pokreneš popravku.

### 5.2 Prati resync

```
esxcli vsan debug resync summary get
esxcli vsan debug resync list
```

Ako je resync već u toku, obično je bolje pustiti ga da završi **ako ima prostora** — prekinuti resync ostavlja objekte u polovičnom stanju. Ako prostora nema, resync će stati sam i objekti ostaju degradirani. To je neprijatno ali nije gubitak podataka.

Namespace `esxcli vsan resync` postoji od 7.0 i sadrži opcije za throttling. Podkomande su se menjale kroz verzije:
```
esxcli vsan resync --help
```
Proveri šta tvoja verzija nudi pre nego što primeniš.

### 5.3 Proveri šta clomd pokušava

```
grep -iE "not enough|no space|failed|cannot place" /var/log/clomd.log | tail -50
```

Poruke tipa nedostatka resursa potvrđuju da CLOM pokušava rekonfiguraciju koja ne može da prođe. Dok je tako, on periodično ponavlja pokušaje i troši ciklus.

### 5.4 Izbegni maintenance mode

Ako ti iz nekog razloga treba maintenance mode tokom incidenta, koristi isključivo **„Ensure accessibility"** ili **„No data migration"**:

```
esxcli vsan maintenancemode get
esxcli system maintenanceMode set -e true -m ensureObjectAccessibility
```

Opcije za `-m`:
- `noAction` — ne pomera ništa
- `ensureObjectAccessibility` — pomera minimum
- `evacuateAllData` — **nikad u ovom scenariju**

Simulacija pre ulaska (7.0+):
```
esxcli vsan debug evacuation precheck -e -t 1
```

---

<a name="6-oslobadjanje"></a>
## 6. Oslobađanje prostora, po redosledu rizika

Idi strogo ovim redom. Svaki korak proveri efekat pre nego što pređeš na sledeći:

```
esxcli vsan debug disk summary get
```

Cilj prvog prolaza: **spustiti najpuniji disk ispod 90%**. To vraća I/O u funkciju i daje ti prostor za manevar.

### 6.1 Nivo 1 — ISO fajlovi i nepotrebni fajlovi (bez rizika)

```
ls -la /vmfs/volumes/vsanDatastore/
find /vmfs/volumes/vsanDatastore/ -name "*.iso" -exec ls -lh {} \;
rm /vmfs/volumes/vsanDatastore/<folder>/<fajl>.iso
```

Traži i:
- `.vmss` (suspend state fajlovi)
- `.vmem`
- stari `vmware-*.log` fajlovi u VM folderima
- core dump fajlovi (`vmx-zdump.*`)

```
find /vmfs/volumes/vsanDatastore/ -name "vmware-*.log" -exec ls -lh {} \;
find /vmfs/volumes/vsanDatastore/ -name "*.vmss" -exec ls -lh {} \;
find /vmfs/volumes/vsanDatastore/ -name "vmx-zdump*" -exec ls -lh {} \;
```

Ovi fajlovi žive u namespace objektu i njihovo brisanje je bezbedno i trenutno.

### 6.2 Nivo 2 — gašenje VM-ova (bez rizika, veliki efekat)

```
vim-cmd vmsvc/getallvms
vim-cmd vmsvc/power.getstate <vmid>
vim-cmd vmsvc/power.shutdown <vmid>
```

Napravi listu prioriteta pre gašenja. Sačuvaj upaljeno samo:
- DNS
- Domain controller
- Backup server (trebaće ti)
- Kritični produkcioni servisi

Sve ostalo gasi. Svaki ugašen VM oslobađa swap objekat.

Ako `shutdown` ne prolazi zbog toga što je guest zamrznut:
```
vim-cmd vmsvc/power.off <vmid>
```

Proveri efekat posle svakih nekoliko VM-ova.

### 6.3 Nivo 3 — brisanje nepotrebnih VM-ova i template-a (nizak rizik)

Za VM koji sigurno više ne treba:
```
vim-cmd vmsvc/getallvms
vim-cmd vmsvc/unregister <vmid>
rm -rf /vmfs/volumes/vsanDatastore/<VM-folder>/
```

> Proveri dvaput da folder odgovara VM-u koji brišeš. `rm -rf` na vSAN namespace-u je nepovratan.

Brisanje objekata je dealokacija metapodataka — radi i kada je datastore pun.

### 6.4 Nivo 4 — zaostali swap objekti (srednji rizik)

Kada host padne, `.vswp` objekti VM-ova koji su na njemu radili mogu ostati zaostali.

Identifikacija:
```
esxcli vsan debug object list --all
```
Traži objekte sa `Type: vmswap` čiji VM nije trenutno upaljen.

Unakrsna provera:
```
vim-cmd vmsvc/getallvms
vim-cmd vmsvc/power.getstate <vmid>
```

**Uslov za brisanje:** VM mora biti ugašen. Swap objekat upaljenog VM-a se ne sme dirati.

Brisanje — vidi 6.5 za proceduru i upozorenja.

### 6.5 Nivo 5 — brisanje objekata preko `objtool` (visok rizik)

> **Ovo je nepovratna operacija.** Obrisan objekat se ne vraća. Radi ovo samo ako si prošao nivoe 1–4 i i dalje nemaš prostora, i samo za objekte koje si pozitivno identifikovao.

**Obavezni pre-check za svaki objekat:**

1. Utvrdi tip i pripadnost:
```
esxcli vsan debug object list -u <object-uuid>
```
Gledaj `Type`, `Directory Name`, `Path`, `Group UUID`.

2. Utvrdi da li `Directory Name` odgovara nekom registrovanom VM-u:
```
vim-cmd vmsvc/getallvms
```

3. Utvrdi da taj VM nije upaljen:
```
vim-cmd vmsvc/power.getstate <vmid>
```

4. Proveri atribute objekta:
```
/usr/lib/vmware/osfs/bin/objtool getAttr -u <object-uuid>
```

**Ako bilo koji korak ostavlja sumnju — ne brisi.** Bolje je čekati dodatni kapacitet nego obrisati produkcioni disk.

Brisanje:
```
/usr/lib/vmware/osfs/bin/objtool delete -u <object-uuid> -f -v 10
```

Tipovi objekata i njihova bezbednost za brisanje:

| Type | Bezbedno brisati |
|---|---|
| `vmswap` | Da, ako je VM ugašen |
| `vmnamespace` | Samo ako VM više ne postoji — briše ceo VM |
| `vdisk` | **Ne**, osim ako je potvrđeno siroče |
| `vmem` | Da, ako VM nije suspendovan |
| `fileSystemOverhead` | **Ne** |
| `vsanFS` / `vsanIscsi` | **Ne** bez razumevanja konfiguracije |

### 6.6 Nivo 6 — smanjenje rezervacije prostora (ograničeno bez vCentera)

Ovde nailaziš na stvarno ograničenje.

```
esxcli vsan policy getdefault
esxcli vsan policy setdefault -c vdisk -p "((\"hostFailuresToTolerate\" i1) (\"proportionalCapacity\" i0))"
```

**Šta ovo radi:** menja podrazumevanu politiku hosta za **novokreirane** objekte.

**Šta ovo NE radi:** ne menja politiku postojećih objekata. Postojeći VM sa thick rezervacijom ostaje thick.

Rekonfiguracija postojećih objekata zahteva SPBM, dakle vCenter. To znači da je smanjenje OSR-a **rešenje posle oporavka vCentera**, ne tokom incidenta.

Kategorije za `-c`: `vdisk`, `vmnamespace`, `vmswap`, `vmem`, `cluster`.

> Zabeleži originalnu politiku pre menjanja.

### 6.7 Nivo 7 — snapshot-ovi (visok rizik pri punom datastore-u)

Konsolidacija snapshot-a **troši prostor**. Ako je datastore i dalje kritično pun, konsolidacija će pasti na pola i ostaviti ti gori problem nego što si imao.

**Uslov:** kreni tek kada imaš najmanje 15–20% slobodnog prostora na najpunijem disku.

Pregled:
```
vim-cmd vmsvc/get.snapshotinfo <vmid>
```

Uklanjanje svih snapshot-ova jednog VM-a:
```
vim-cmd vmsvc/snapshot.removeall <vmid>
```

Uklanjanje pojedinačnog:
```
vim-cmd vmsvc/snapshot.remove <vmid> <snapshotId>
```

Prati napredak:
```
tail -f /var/log/vmkernel.log
tail -f /vmfs/volumes/vsanDatastore/<VM>/vmware.log
```

Ako konsolidacija padne, VM ostaje sa `needConsolidate` stanjem. Nemoj ponavljati dok ne oslobodiš više prostora.

### 6.8 Nivo 8 — dodavanje kapaciteta

Ako imaš slobodne diskove u hostovima, ovo je najčistije rešenje.

Provera podobnosti:
```
vdq -q -H
```

**OSA — dodavanje capacity diska u postojeću disk grupu:**
```
esxcli vsan storage list
esxcli vsan storage add -s <cache-disk-naa> -d <novi-capacity-disk-naa>
```

**ESA — dodavanje u storage pool:**
```
esxcli vsan storagepool list
esxcli vsan storagepool add -d <novi-disk-naa>
```

Ako je disk označen kao neupotrebljiv zbog zaostalih particija:
```
partedUtil getptbl /dev/disks/<naa>
partedUtil delete /dev/disks/<naa> 1
```

> Potvrdi da disk **nije** deo vSAN-a niti VMFS-a pre brisanja particija.

Ako u OSA koristiš dedup/compression, dodavanje diska u postojeću grupu je podržano, ali očekuj rebalance koji sam po sebi troši I/O.

**Napomena o balansu:** dodavanje diskova u samo jedan host neće ravnomerno pomoći — CLOM raspoređuje komponente prema fault domain pravilima. Dodaj kapacitet na više hostova ako možeš.

### 6.9 Nivo 9 — rebalance

Kada je problem imbalans a ne ukupan kapacitet (jedan disk 96%, ostali 60%), treba ti rebalance.

Proaktivni rebalance se u normalnim uslovima pokreće iz vCentera ili RVC-a — oba su ti nedostupna. Automatski rebalance postoji i kontroliše se advanced parametrima na hostu, ali su se imena i ponašanje menjali kroz 6.7 → 7.x → 8.x.

```
esxcli system settings advanced list | grep -i -A3 balance
```

Pogledaj šta tvoja verzija nudi pre nego što bilo šta menjaš. **Ne pokreći rebalance dok je datastore kritično pun** — rebalance pomera podatke i privremeno ih duplira.

Praktično: u ovom incidentu rebalance je posao za posle oporavka vCentera.

---

<a name="7-migracija"></a>
## 7. Migracija na drugi datastore

Ovo je pravi izlaz iz situacije, a ne samo palijativa. Cilj je skinuti dovoljno podataka sa vSAN-a da se sistem trajno stabilizuje.

### 7.1 Najbrži ventil — privremeni NFS datastore

Ako imaš bilo koji NFS share u mreži, ovo je najbrži način da dobiješ prostor.

```
esxcli storage nfs add -H <nfs-server-ip> -s /export/temp -v tempds
esxcli storage nfs list
df -h | grep tempds
```

Za NFS 4.1:
```
esxcli storage nfs41 add -H <ip> -s /export/temp -v tempds
```

Montiraj na **sve** hostove sa istim imenom volumena — inače kasnija registracija VM-ova neće raditi konzistentno.

Alternativa: iSCSI LUN ili lokalni VMFS datastore na nekom hostu.

```
esxcli storage vmfs extent list
esxcli storage filesystem list
```

### 7.2 Redosled evakuacije

Ne seli sve. Seli ono što najbrže oslobađa najviše prostora uz najmanji rizik:

1. **Template-i i ISO biblioteke** — nula rizika, često desetine GB
2. **Ugašeni VM-ovi** — hladna kopija, bez konzistencije podataka
3. **Test/dev VM-ovi**
4. **Veliki VM-ovi sa thick rezervacijom** — najveći dobitak po jedinici truda
5. **VM-ovi sa mnogo snapshot-ova** — kopiraj pa konsoliduj na novom datastore-u gde ima prostora

Ne seli VCSA na ovaj način ako još uvek postoji — vidi poglavlje 8.

### 7.3 Kopiranje VMDK-a

VM mora biti **ugašen**.

```
vim-cmd vmsvc/power.getstate <vmid>
```

Kreiraj ciljni folder:
```
mkdir /vmfs/volumes/tempds/VM01
```

Kopiraj disk sa konverzijom u thin:
```
vmkfstools -i /vmfs/volumes/vsanDatastore/VM01/VM01.vmdk -d thin /vmfs/volumes/tempds/VM01/VM01.vmdk
```

Opcije za `-d`:
- `thin` — najmanji prostor, preporučeno u ovoj situaciji
- `zeroedthick` / `eagerzeroedthick` — samo ako ti treba na ciljnom storage-u

`vmkfstools -i` **konsoliduje snapshot lanac** u jedan disk. To je često poželjno — dobijaš čist disk bez delta fajlova. Ali zahteva da izvorni lanac bude čitljiv.

Kopiraj konfiguraciju:
```
cp /vmfs/volumes/vsanDatastore/VM01/VM01.vmx /vmfs/volumes/tempds/VM01/
cp /vmfs/volumes/vsanDatastore/VM01/*.nvram /vmfs/volumes/tempds/VM01/
```

Za VM sa više diskova, ponovi `vmkfstools -i` za svaki `.vmdk` descriptor (ne za `-flat.vmdk`).

### 7.4 Registracija na novoj lokaciji

```
vim-cmd solo/registervm /vmfs/volumes/tempds/VM01/VM01.vmx
vim-cmd vmsvc/getallvms
```

Ako `.vmx` referencira stare putanje, uredi ga pre registracije:
```
cat /vmfs/volumes/tempds/VM01/VM01.vmx | grep -i fileName
vi /vmfs/volumes/tempds/VM01/VM01.vmx
```

Zameni sve `vsanDatastore` reference sa `tempds`.

Paljenje:
```
vim-cmd vmsvc/power.on <novi-vmid>
```

Pri prvom paljenju ESXi može pitati za UUID — odgovor „I moved it" se u CLI-ju rešava sa:
```
vim-cmd vmsvc/message <vmid>
vim-cmd vmsvc/message <vmid> <messageId> 2
```

### 7.5 Brisanje originala tek posle verifikacije

**Ne brisi izvorni VM dok novi ne proradi i dok ne proveriš podatke unutar guest OS-a.**

Tek onda:
```
vim-cmd vmsvc/unregister <stari-vmid>
rm -rf /vmfs/volumes/vsanDatastore/VM01/
```

### 7.6 Izvoz preko `ovftool` (alternativa)

Ako imaš radnu stanicu sa `ovftool`-om, možeš izvesti VM direktno sa hosta:

```
ovftool vi://root@esx01.lab.local/VM01 /putanja/VM01.ova
```

Sporije od `vmkfstools`, ali daje prenosiv paket i ne zahteva montiranje dodatnog datastore-a na hostove. Korisno kada seliš na potpuno drugu infrastrukturu.

---

<a name="8-vcenter"></a>
## 8. Oporavak vCentera

Radi ovo **tek kada** je najpuniji disk ispod 85% i I/O je stabilan.

### 8.1 Slučaj A — VCSA postoji, samo nije mogao da piše

Nađi VCSA:
```
vim-cmd vmsvc/getallvms | grep -i vcsa
vim-cmd vmsvc/power.getstate <vcsa-vmid>
```

Ako je ugašen, upali:
```
vim-cmd vmsvc/power.on <vcsa-vmid>
```

Sačekaj 10–15 minuta. VCSA startuje sporo, pogotovo posle nečistog gašenja.

Provera preko konzole ili SSH-a na VCSA:
```
df -h
service-control --status
```

Ako servisi nisu pokrenuti:
```
service-control --start --all
```

### 8.2 Slučaj B — VCSA-ini interni diskovi su puni

Uloguj se na VCSA (SSH ili konzola), pa:
```
df -h
du -sh /storage/*
```

Tipični krivci:

| Particija | Sadržaj | Tipično čišćenje |
|---|---|---|
| `/storage/log` | vpxd i servisni logovi | rotirani logovi, stari `.gz` |
| `/storage/seat` | statistika, događaji, taskovi, alarmi | najčešći uzrok; čisti se kroz VMware KB proceduru |
| `/storage/core` | core dump-ovi | bezbedno za brisanje |
| `/storage/db` | PostgreSQL baza | ne diraj ručno |
| `/storage/dblog` | WAL logovi | ne diraj ručno |
| `/storage/updatemgr` | patch depot | stari update-i |

Bezbedno za brisanje:
```
ls -lh /storage/core/
rm /storage/core/core.*
ls -lh /storage/log/vmware/vpxd/
find /storage/log -name "*.gz" -mtime +7
```

> **Ne brisi ništa iz `/storage/db` ili `/storage/dblog` ručno.** To korumpira bazu i pretvara oporavak u redeploy.

Za `/storage/seat` (statistika, events, tasks) postoji zvanična VMware KB procedura za skraćivanje tabela. Ne improvizuj SQL nad vCenter bazom — nađi KB za svoju verziju.

### 8.3 Proširivanje VCSA diskova

Ako čišćenje nije dovoljno, proširi virtuelni disk. **Ovo zahteva slobodan prostor na vSAN-u** — otud redosled.

1. Ugasi VCSA ili proširi disk u letu (podržano)
2. Proširi odgovarajući VMDK — bez vCentera to znači ručno menjanje veličine kroz `vmkfstools`:
```
vmkfstools -X 50G /vmfs/volumes/vsanDatastore/VCSA/VCSA_9.vmdk
```
3. Unutar VCSA pokreni autogrow:
```
/usr/lib/applmgmt/support/scripts/autogrow.sh
df -h
```

Mapiranje VMDK broja na particiju razlikuje se po verziji VCSA — proveri VMware dokumentaciju za svoju verziju pre proširivanja. Pogrešan disk znači proširen `/storage/core` umesto `/storage/seat`.

### 8.4 Slučaj C — VCSA je nepovratan, treba novi

**Ključno pravilo: novi VCSA deploy-uj na lokalni VMFS ili NFS datastore, ne na vSAN koji si upravo oporavio.**

Razlozi:
- Prekidaš cirkularnu zavisnost trajno
- Ne trošiš tek oslobođen vSAN prostor
- Sledeći put kad se vSAN napuni, vCenter i dalje radi

Preduslovi pre deploy-a:
- DNS zapis (forward i reverse) za novi VCSA — mora raditi
- Slobodna IP adresa
- NTP izvor koji nije stari VCSA
- Slobodan prostor: računaj sa ~500 GB za Small deployment
- Isti SSO domen i ime ako želiš minimalno prekonfigurisanje

Deploy ide direktno na ESXi host (Stage 1 instalatera prihvata ESXi kao cilj, ne mora vCenter).

Posle deploy-a:
1. Kreiraj Datacenter i Cluster
2. Dodaj hostove **jedan po jedan**
3. Uključi vSAN na klasteru — hostovi već imaju vSAN konfiguraciju i podatke; klaster se prepoznaje
4. **Ne dozvoli claim novih diskova** dok ne potvrdiš da su postojeće disk grupe/pool prepoznati
5. Rekreiraj SPBM politike ručno — one su nestale sa starim VCSA
6. Ponovo dodeli politike VM-ovima
7. Uključi Operations Reserve i Host Rebuild Reserve (7.0 U1+) — vidi 9.3

> Ako imaš file-based backup starog VCSA, restore je uvek bolji od novog deploy-a — čuva SPBM politike, permissions, foldere, DRS pravila i tagove.

---

<a name="9-posle"></a>
## 9. Posle oporavka

### 9.1 Vrati privremene izmene

Na svakom hostu vrati `ClomRepairDelay` na originalnu vrednost:
```
esxcfg-advcfg -s 60 /VSAN/ClomRepairDelay
/etc/init.d/clomd restart
```

Vrati podrazumevanu politiku ako si je menjao:
```
esxcli vsan policy getdefault
```

### 9.2 Pusti klaster da se popravi

```
esxcli vsan debug object health summary get
esxcli vsan debug resync summary get
esxcli vsan debug disk summary get
```

Očekuj period resync-a. Prati dok `resync summary` ne pokaže da nema preostalih bajtova.

Sve dok ima objekata u `reducedavailability*` stanju, klaster nije oporavljen.

### 9.3 Trajne mere protiv ponavljanja

| Mera | Kako |
|---|---|
| Uključi Operations Reserve i Host Rebuild Reserve | vCenter → Cluster → Configure → vSAN Services → Reservations (7.0 U1+) |
| Alarmi na kapacitet | Alarm na 70% i 80%, ne na 90% — 90% je već kasno |
| Prebaci VCSA sa vSAN-a | Lokalni VMFS ili odvojen storage |
| Ukloni thick rezervacije | SPBM politika sa OSR 0% gde god je moguće |
| Politika snapshot-ova | Maksimalno trajanje, automatsko upozorenje |
| Ukloni ISO biblioteku sa vSAN-a | Content Library na NFS-u |
| Uključi TRIM/UNMAP | Nivo klastera, vraća prostor koji je guest oslobodio |
| Redovan rebalance | Prati imbalans po disku, ne samo prosek |
| File-based backup VCSA | VAMI (port 5480), dnevno, na odvojen storage |
| Eksterni NTP | Ne oslanjaj se na VCSA kao izvor vremena |

### 9.4 Provera guest OS-a

VM-ovi koji su radili tokom punog datastore-a mogli su dobiti I/O greške. Proveri:

- Linux: `dmesg | grep -i "read-only\|I/O error"`, remontiraj rw, `fsck` gde treba
- Windows: Event Log — Disk i Ntfs greške, `chkdsk`
- Baze podataka: provera konzistentnosti pre puštanja u produkciju

Ovo se često previdi i problem se pojavi danima kasnije.

### 9.5 Rekonstrukcija onoga što je nestalo sa starim vCenterom

Ako si deploy-ovao novi VCSA bez backupa, izgubljeno je:
- SPBM politike i njihova dodela VM-ovima
- Folder struktura i permissions
- DRS pravila, resource pool-ovi
- Tagovi i kategorije
- Alarmi i njihova konfiguracija
- Istorija taskova i događaja
- Distributed switch konfiguracija (ako nije rekreirana sa hostova)

Postojeći objekti zadržavaju SPBM ID upisan u metapodatke, ali novi vCenter tu politiku ne poznaje — prikazaće ih kao non-compliant dok ne rekreiraš politiku sa istim pravilima i ponovo je dodeliš.

---

<a name="10-checklist"></a>
## 10. Sažeti checklist

**Stabilizacija**
- [ ] Ništa se ne pali, ne pravi se snapshot, ne restartuje se host
- [ ] `ClomRepairDelay` produžen na svim hostovima, `clomd` restartovan
- [ ] Nekritični VM-ovi ugašeni
- [ ] Stanje snimljeno van vSAN-a
- [ ] Vreme na hostovima provereno

**Dijagnostika**
- [ ] `esxcli vsan debug disk summary get` — najpuniji disk identifikovan
- [ ] Congestion u `vmkernel.log` potvrđen ili isključen
- [ ] `esxcli vsan debug limit get` — limit komponenti provere
- [ ] `object health summary get` — broj `inaccessible` objekata
- [ ] Resync status proveren
- [ ] Uzrok popunjenosti identifikovan (snapshot-ovi / thick / swap / ISO / zombie)

**Oslobađanje**
- [ ] Nivo 1: ISO i pomoćni fajlovi obrisani
- [ ] Nivo 2: VM-ovi ugašeni
- [ ] Nivo 3: nepotrebni VM-ovi obrisani
- [ ] Nivo 4: zaostali swap objekti obrisani
- [ ] Nivo 5: potvrđeni zombie objekti obrisani (samo ako je bilo neophodno)
- [ ] Najpuniji disk ispod 90%

**Trajno rešenje**
- [ ] Privremeni datastore montiran
- [ ] Podaci evakuisani po prioritetu
- [ ] Kapacitet dodat ako je bilo slobodnih diskova
- [ ] Najpuniji disk ispod 80%

**vCenter**
- [ ] VCSA upaljen ili novi deploy-ovan **na non-vSAN datastore**
- [ ] Hostovi vraćeni u klaster, vSAN prepoznat
- [ ] Disk grupe / storage pool potvrđeni pre bilo kakvog claim-a
- [ ] SPBM politike rekreirane i dodeljene

**Zatvaranje**
- [ ] `ClomRepairDelay` vraćen na originalnu vrednost
- [ ] Resync završen, svi objekti `healthy`
- [ ] Rezerve kapaciteta uključene
- [ ] Alarmi na 70% i 80% postavljeni
- [ ] Guest OS fajl sistemi provereni
- [ ] VCSA backup konfigurisan

---

## Predlog imena fajla

**`vsan-pun-datastore-pad-vcentera-oporavak.md`**

Alternative ako ti treba drugačija konvencija:
- `00-incident-vsan-full-vcenter-down.md` — ako ide kao nulti/hitni modul ispred kursa
- `runbook-vsan-kapacitet-incident.md` — ako gradiš biblioteku runbook-ova po tipu incidenta
- `vsan-oporavak-bez-vcentera-pun-datastore.md` — ako ti je „bez vCentera" glavni klasifikator
