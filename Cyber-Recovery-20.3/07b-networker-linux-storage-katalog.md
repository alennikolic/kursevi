# DEO VII-b — STORAGE, UREĐAJI I KATALOG IZ KONZOLE

**Dell PowerProtect Cyber Recovery 20.3 — Uputstvo za instalaciju i integraciju**
Verzija dokumenta: 0.1 | **Namena: sistem administratori i System Engineer-i**
**Referentni OS:** Red Hat Enterprise Linux 9.6 | **Referentni NetWorker:** 19.11 / 19.13

> **Izvori:**
> - *Dell NetWorker 19.11 Command Reference Guide* — **[CMD]**
> - *Dell NetWorker 19.13 Administration Guide* — **[ADM]**
> - *Dell NetWorker Server Disaster Recovery and Availability Best Practices Guide 19.13* — **[DR]**
> - *Dell PowerProtect Cyber Recovery 20.3 Product Guide / Installation and Upgrade Guide* — **[CR]**

> **Namena ovog dela:** objasniti kako NetWorker vidi storage i kako se katalog ispituje iz komandne linije. Ovo je most između `07a` (šta je gde na sistemu) i `07c` (kako se izvodi oporavak).

---

## Sadržaj

- [48. Kako NetWorker vidi storage](#48-kako-networker-vidi-storage)
- [49. Pregled uređaja i volumena iz konzole](#49-pregled-uređaja-i-volumena-iz-konzole)
- [50. Katalog: media baza i client file indeksi](#50-katalog-media-baza-i-client-file-indeksi)
- [51. Skeniranje medija i rekonstrukcija kataloga](#51-skeniranje-medija-i-rekonstrukcija-kataloga)
- [Dodatak: Kartica komandi za storage i katalog](#dodatak-kartica-komandi-za-storage-i-katalog)

---

# 48. Kako NetWorker vidi storage

## 48.1 Lanac apstrakcija

Ovo je model koji treba imati u glavi pre bilo koje komande:

```
DD sistem
   └── MTree / storage unit                 ← postoji na DD-u
          └── NSR device (resurs)           ← NetWorker konfiguracija
                 └── volume (labela)        ← medij, ima ime i režim
                        └── save set        ← podaci, ima ssid
                               └── fajlovi  ← zapisi u client file indeksu
```

Svaki nivo ima svoju komandu:

| Nivo | Komanda za pregled |
|---|---|
| Storage unit na DD-u | `ddboost storage-unit show` (na DD sistemu) |
| NSR device | `nsradmin`, `nsrmm -C` |
| Volume | `mminfo -m`, `nsrmm -C` |
| Save set | `mminfo` |
| Fajlovi | `nsrinfo`, `nsrls` |

> **Ključna razlika koju administratori često promaše:** NetWorker uređaj **nije** fajl sistem. To je resurs u RAP bazi koji opisuje **kako se dolazi do** medija. Postojanje uređaja ne znači da su podaci tu, niti odsustvo uređaja znači da podataka nema.

## 48.2 Tri porodice medija **[CMD]**

Atribut **`media family`** opisuje klasu storage medija, izvedenu iz `media type`. Dozvoljene vrednosti su samo tri:

| `media family` | Značenje |
|---|---|
| **`tape`** | Trakasti storage uređaj |
| **`disk`** | Disk storage uređaj |
| **`logical`** | Koristi se pri interakciji sa eksternim media management servisom |

Atribut **`media type`** označava konkretan tip medija koji uređaj koristi. Vrednosti zavise od operativnog sistema i platforme.

Primeri iz dokumentacije **[CMD]**:

| `media type` | Opis | Podrazumevani kapacitet |
|---|---|---|
| `adv_file` | Advanced file type device; podržan standardni UNIX fajl sistem | — |
| `4mm` | 4mm digital audio tape | 1 GB |
| `8mm` | 8mm video tape | 2 GB |
| `8mm 5GB` | 8mm video tape | 5 GB |
| `dlt` | Digital linear tape cartridge | 10 GB |

> **[CMD]** Za sveobuhvatnu listu tipova medija podržanih na vašoj platformi videti online *NetWorker Hardware Compatibility Guide*.

## 48.3 `device access information` — format po tipu **[CMD]**

Ovo je atribut koji određuje **kako se dolazi do podataka**. Format se razlikuje po tipu uređaja.

> Atribut `device access information` određuje pristupnu putanju za **advanced file** ili **Data Domain** uređaj. String mora imati jedan od sledećih formata: `hostname:device_path` ili `device_path`, u zavisnosti od tipa uređaja.

| Tip uređaja | Format | Objašnjenje **[CMD]** |
|---|---|---|
| **Data Domain** | `<dd_hostname>:<device_path>` | `hostname` je **ime hosta Data Domain servera**; `device_path` je putanja **relativna u odnosu na mount point koji DD server eksportuje**. **DD device putanja treba da počinje kratkim imenom hosta NetWorker servera.** |
| **NFS-based advanced file** | `<nfs_hostname>:<device_path>` | `hostname` je ime hosta NFS servera; `device_path` je **puna eksportovana putanja** do željenog direktorijuma |
| **Ostali advanced file** | `<device_path>` | **Apsolutna** putanja dostupna klijentu. Preporučuje se UNC ili automounter putanja. Prva vrednost u listi mora biti apsolutna putanja do uređaja na storage node-u na kojem je uređaj definisan. |

> **[CMD]** **Razmaci u imenu hosta nisu podržani.**

Primer iz dokumentacije **[CMD]**:

```
device access information: "<host1>:/<host2>/ddDev1";
```

gde je `host1` DD sistem, a `host2` kratko ime NetWorker servera.

Praktičan primer za vault:

```
name: dd-recover-01;
device access information: "dd-vault.example.com:/nwvault/dd-recover-01";
```

## 48.4 Ime uređaja **[CMD]**

> Atribut `name` je ime uređaja koje server koristi za snimanje i vraćanje podataka. Ime uređaja je obično **putanja** trakastog ili file type uređaja, na primer `/dev/nrmt8`. Ime može imati prefiks `rd=`, ime hosta uređaja i dvotačku.
>
> **Za `adv_file` ili Data Domain uređaj ime može biti bilo koji string.** Za te uređaje se putanja zadaje atributom `device access information`.
>
> **Ime se ne može promeniti nakon što je uređaj kreiran** — po potrebi obrisati uređaj i kreirati novi.

| Situacija | Format `name` |
|---|---|
| Lokalni uređaj | `<ime>` ili putanja, npr. `/dev/nrmt8` |
| Udaljeni storage node **[ADM]** | `rd=<remote_snode_hostname>:<device_name>`, npr. `rd=snode-1:aftd-1` |

## 48.5 Zašto se DD Boost uređaj ne vidi u `mount`

Ovo je najčešća zabuna kod administratora koji dolaze iz sveta fajl sistema.

| Tip uređaja | Vidi se u `mount` / `df` | Zašto |
|---|---|---|
| **AFTD lokalni** | **Da** | Običan lokalni direktorijum na fajl sistemu |
| **AFTD preko NFS-a** | **Da** | Standardan NFS mount na OS nivou |
| **DD Boost uređaj** | **Ne** | DD Boost je **aplikativni protokol**. NetWorker se povezuje na DD kroz DD Boost biblioteku i **ne montira fajl sistem.** |
| **Traka** | **Ne** | Blok uređaj, `/dev/nst*` |

**Šta to znači u praksi:**

```bash
# Ovo NEĆE pokazati DD Boost uređaj
df -h
mount

# Ovo hoće
nsrmm -C
nsradmin -c "type:NSR device"
```

**Provera koja ima smisla po tipu:**

```bash
# AFTD / NFS
mount | grep -i nfs
df -h
showmount -e <dd_hostname>

# Traka
lsscsi -g
ls -l /dev/nst*
mt -f /dev/nst0 status

# DD Boost — samo kroz NetWorker i DD stranu
nsrmm -C
ssh sysadmin@<dd> "ddboost storage-unit show"
```

## 48.6 Secure multi tenancy — bitno za vault **[CMD]**

> Atribut **`secure multi tenancy`** (read/write, **hidden**) označava da li se koristi secure multi tenancy pri kreiranju Data Domain uređaja. **Primenljiv je samo pri kreiranju uređaja.**
>
> Kada je postavljen, **prednji deo `device access information` atributa sme imati vrednosti različite od imena servera.** Ako storage unit povezana sa tim delom putanje ne postoji, **kreira se automatski**. Ako već postoji, prosleđeni kredencijali moraju pripadati **istom korisniku koji je kreirao tu storage unit.**

**Zašto je ovo važno u vault-u:**

Podrazumevano DD device putanja mora početi kratkim imenom hosta NetWorker servera. U vault-u često radite nad **repliciranim MTree-om** čije ime ne odgovara imenu vault NetWorker servera. SMT opcija je mehanizam koji to dozvoljava.

> **[ADM]** Za operacije nad repliciranim volumenima: *kreirati DD Device koristeći **secure multi-tenancy (SMT)** opciju za replicirane volumene na ciljnom NetWorker serveru. Izabrati ime ciljnog MTree-a koje je korišćeno pri kreiranju Data Domain mtree replikacije.*

Pošto je atribut **hidden**, u `nsradmin`-u ga treba prikazati:

```bash
nsradmin -s <server>
```

```
nsradmin> option hidden
nsradmin> . type: NSR device
nsradmin> p
```

## 48.7 Replicirani MTree i replica zastavica

### MTree replikacija se ne konfiguriše iz NetWorker-a

> **[ADM]** NetWorker **ne pruža nikakav mehanizam za podešavanje Data Domain mtree replikacije.** Za to videti *Dell NetWorker Data Domain Boost Integration Guide*.
>
> NetWorker mtree replikacija je kompatibilna sa **DDOS 6.2 i novijim**.

### Preduslov: isti UID **[ADM]**

> Obezbediti da DD Boost korisnik koji se koristi na izvornom i ciljnom DD sistemu ima **isti UID**. Validacija:
> ```bash
> sysadmin@dd# user show list
> ```

### Kreiranje replikacionog para **[ADM]**

Iz DD CLI-ja:

```bash
sysadmin@dd-prod# replication add \
  source mtree://<source-DD-host-name>/data/col1/<source-mtree-name> \
  destination mtree://<destination-DD-host-name>/data/col1/<destination-mtree-name>
```

Iz DD UI-ja: **Replication → Automatic → Create Pair**, dodati ciljni DD preko **Add System**, uneti izvornu i odredišnu MTree putanju.

> **[ADM]** Pri kreiranju replikacionog para **ne koristiti ime ciljnog servera kao ime ciljnog MTree-a.** Koristiti drugačije ime, jer su ti MTree-ovi **read-only** i ne mogu se koristiti na odredištu za kreiranje DD uređaja.

### Replicirani MTree nije automatski storage unit **[ADM]**

> Nakon kreiranja replikacionog para sa izvornog DD sistema, na ciljnom DD sistemu se kreira replicirani MTree. Taj MTree je **vidljiv na ciljnom DD sistemu, ali ne i u njegovoj storage unit listi.**
>
> Da bi replicirana storage unit postala vidljiva na ciljnom DD sistemu, ažurirati DD Boost korisnika:

```bash
sysadmin@dd-vault# ddboost storage-unit modify <target-mtree-name> user <ddboost-user>
sysadmin@dd-vault# ddboost storage-unit show
```

### Replica zastavica na volumenu **[CMD]**

Kada je replikacija prekinuta, volumeni na ciljnoj strani nose **replica** zastavicu koja ograničava operacije nad njima.

> Opcija **`nsrmm -g`** koristi se **samo za replica volumen nakon što je DD replikacija prekinuta** u mtree replication funkcionalnosti. Uklanja **replica zastavicu sa zapisa volumena i sa svakog save set-a na tom volumenu.** Nakon toga se volumen i njegovi save set-ovi mogu normalno obrađivati komandom `nsrmm`.

```bash
nsrmm -g <volume>
```

> **[CMD]** Operacija označavanja (`nsrmm -l`) **ne uspeva ako se pokuša nad repliciranim volumenom.** To je zaštita — vidi 49.9.

## 48.8 Šta je stvarno replicirano u vault

| Stiže MTree replikacijom | Ne stiže |
|---|---|
| Backup podaci (save set-ovi) | NetWorker resource baza (`/nsr/res`) |
| Bootstrap save set-ovi | NetWorker media baza (`/nsr/mm`) |
| Index save set-ovi (kao save set-ovi) | Aktivni client file indeksi (`/nsr/index`) |
| Labele volumena na medijumu | NSR device resursi |

> **Ključno razumevanje:** podaci postoje na DD sistemu u vault-u, ali NetWorker o njima **ne zna ništa**. Cilj poglavlja 51 i celog `07c` je da NetWorker-u napravite uređaj koji pokazuje na te podatke i da iz njih rekonstruišete katalog.

---

# 49. Pregled uređaja i volumena iz konzole

## 49.1 Šta je montirano na čemu — `nsrmm -C`

> **[CMD]** `-C` prikazuje listu **NetWorker konfigurisanih uređaja i volumena koji su u njima trenutno montirani.** Lista prikazuje **samo uređaje i volumene dodeljene serveru**, ne nužno sve stvarno priključene. `-C` je **podrazumevana opcija.**

```bash
nsrmm
nsrmm -C
nsrmm -C -v          # verbose
```

Očekivani oblik izlaza **[DR]**:

```
32916:nsrmm: file disk <volume_name> mounted on <device_name>, write enabled
```

> **[CMD]** Opcija `-p` verifikuje labelu volumena (vidi 49.8).

**Ako je izlaz prazan:** nema konfigurisanih uređaja, ili server ne odgovara. Proveriti da li servis radi (`07a`, poglavlje 42).

## 49.2 Uređaji kroz `nsradmin`

> **[CMD]** Za uređivanje NSR device resursa:
> ```bash
> nsradmin -c "type:NSR device"
> ```
> **Obavezno uključiti navodnike i razmak između `NSR` i `device`.**

Opcija `-c` koristi UNIX `curses(3)` biblioteku i daje **puni ekranski prikaz** — isto kao `visual` komanda unutar `nsradmin`-a. **[CMD]**

Neinteraktivno:

```bash
nsradmin -s <server> <<'EOF'
. type: NSR device
p
EOF
```

Sa skrivenim atributima:

```bash
nsradmin -s <server> <<'EOF'
option hidden
. type: NSR device
p
EOF
```

Kada server **ne radi** **[CMD]**:

```bash
cd /nsr/res
nsradmin -d nsrdb
```

```
nsradmin> . type: NSR device
nsradmin> p
```

> **[CMD]** Opciju `-d resdir` koristiti **štedljivo, i samo kada NetWorker server nije pokrenut.**

## 49.3 Ključni atributi NSR device resursa **[CMD]**

| Atribut | Pristup | Značenje |
|---|---|---|
| `name` | read-only, static | Ime uređaja. **Ne može se menjati posle kreiranja.** |
| `device access information` | read-only | Pristupna putanja za advanced file ili DD uređaj |
| `media type` | read-only, static | Tip medija |
| `media family` | read-only, static, hidden | `tape`, `disk` ili `logical` |
| `volume name` | read-only, dynamic, hidden | **Ime volumena kada je montiran; prazno kada nije** |
| `enabled` | read/write | `Yes`, `No` ili `Service` — vidi 49.4 |
| `read only` | read/write | `yes` ili `no` — vidi 49.5 |
| `enable fibre channel` | read/write | FC pristup za DD uređaje |
| `fibre channel hostname` | read/write | Hostname koji DD koristi za FC operacije; **case-sensitive**, mora odgovarati Server Name u DD Enterprise Manager-u (Data Management → DD Boost → Fibre Channel) |
| `secure multi tenancy` | read/write, hidden | Samo pri kreiranju uređaja — vidi 48.6 |
| `suspected device` | read-only, hidden | Da li je uređaj izgubio konekciju na storage node-u. Početna vrednost `No`. |
| `message` | read-only, dynamic, hidden | Poslednja poruka servera vezana za uređaj |
| `comment` | read/write | Proizvoljna napomena administratora |
| `description` | read/write | Kratak opis radi lakše identifikacije |

## 49.4 Atribut `enabled` — tri stanja **[CMD]**

Ovo je često previđen atribut sa tri vrednosti, ne dve.

| Vrednost | Ponašanje |
|---|---|
| **`Yes`** | Uređaj je potpuno operativan i može se koristiti za sve operacije. **Podrazumevano.** |
| **`No`** | Uređaj je onemogućen i ne može se koristiti. **Ne može se postaviti na `No` ako je volumen montiran**, jer bi montirani volumen postao nedostupan NetWorker-u dok se ne vrati na `Yes`. |
| **`Service`** | Uređaj se **ne može montirati za save ili recover operacije.** Stanje se koristi za **rezervisanje uređaja za održavanje.** Uređaj se može koristiti za administrativne svrhe — verifikaciju volumena, označavanje ili inventar — **ako se uređaj izabere opcijom `-f`.** |

> **`Service` režim je korisna alatka u vault-u:** dozvoljava rad sa uređajem u dijagnostičke svrhe bez rizika da neki job počne da piše na njega.

```bash
nsradmin -s <server> <<'EOF'
. type: NSR device; name: <ime_uredjaja>
update enabled: Service
EOF
```

## 49.5 Atribut `read only` **[CMD]**

> Označava da li je uređaj rezervisan za **read-only operacije**, kao što su recover ili retrieve. Vrednost može biti `yes` ili `no`. Ako je `yes`, **dozvoljene su samo read operacije.**
>
> **Vrednost se ne može promeniti dok je volumen montiran.**

> **Preporuka za vault:** uređaj kreiran nad repliciranim podacima radi oporavka postaviti na `read only: yes` **pre** montiranja. To je najjeftinija zaštita od slučajnog prepisivanja.

## 49.6 Pregled volumena — `mminfo -m` **[CMD]**

> `-m` prikazuje ime svakog volumena u media bazi, broj upisanih bajtova, procenat iskorišćenog prostora (ili reč `full`), vreme retencije (isteka), broj pročitanih bajtova, broj izvršenih read-label operacija nad volumenom (**ne broj eksplicitnih montiranja**) i kapacitet volumena.
>
> **Za disk tipove uređaja, `written` prikazuje količinu podataka na disku i treba da odgovara zauzeću diska na putanji uređaja.**

```bash
mminfo -m
mminfo -m -v          # + volid, broj sledećeg fajla za pisanje, tip medija
mminfo -m -V          # + kolona zastavica
mminfo -m <volume1> <volume2>
mminfo -m -t 'last week'
```

### Zastavice u prvoj koloni **[CMD]**

| Zastavica | Značenje |
|---|---|
| **`E`** | **Eligible for recycling** — volumen je recyclable (vidi `nsrim`) |
| **`M`** | Volumen je označen kao **manually-recyclable** |
| **`X`** | Volumen je **i** manually-recyclable **i** eligible for recycling |
| **`A`** | **Archive** ili **migration** volumen |
| (prazno) | Volumen nije archive/migration i nije recyclable |

### Dodatne zastavice uz `-V` **[CMD]**

| Zastavica | Značenje |
|---|---|
| **`d`** | Volumen se trenutno piše (**dirty**) |
| **`r`** | Volumen je označen kao **read-only** |
| (prazno) | Nijedno od navedenog |

**Ekvivalentan custom izveštaj [CMD]:**

```bash
mminfo -a -r 'state,volume,written,%used,volretent,read,space' \
       -r 'mounts(5),space(2),capacity'
```

## 49.7 Korisni upiti nad volumenima **[CMD]**

```bash
# Svi volumeni koji nisu puni, sa procentom, pool-om i lokacijom
mminfo -a -r 'volume,%used,pool,location' -q '!full'

# Volumeni između 20% i 80% iskorišćenosti
mminfo -a -q '%used>20,%used<80' -r 'volume,%used,pool'

# Izveštaj sa barkodom umesto labele
mminfo -a -r 'state,barcode,written,%used,read,space' \
       -r 'mounts(5),space(2),capacity'

# Da li svi završeni save set-ovi na volumenu imaju bar jedan uspešan klon
mminfo -q 'volume=<ime_volumena>,validcopies>1'
```

## 49.8 Montiranje, demontiranje i verifikacija **[CMD]**

| Komanda | Radnja |
|---|---|
| `nsrmm -m [-f <device>] [<volume>]` | **Montira** volumen u uređaj. Montiranje se izvodi nakon što je volumen smešten u uređaj i **označen**. Mogu se montirati **samo označeni volumeni.** |
| `nsrmm -m -r ...` | Montira volumen kao **read-only**. Volumeni označeni kao `full` i oni u read-only režimu se automatski montiraju read-only. |
| `nsrmm -u [-f <device> \| <volume>...]` | **Demontira** volumen |
| `nsrmm -j ...` | **Izbacuje (eject)** volumen iz uređaja. Slično unmount-u, ali se volumen i fizički izbacuje ako je moguće. **Nije podržano na nekim tipovima uređaja, disk uređajima i trakama.** |
| `nsrmm -p [-f <device>]` | **Verifikuje i ispisuje labelu volumena.** Za potvrdu da spoljna labela odgovara internoj. **Verifikacija labele demontira montirane volumene.** |
| `nsrmm -H -f <device>` | **Softverski reset uređaja.** Tekuće operacije se prekidaju, što ponekad može dovesti do gubitka podataka. **Resetuje interno NetWorker stanje uređaja, ne fizički uređaj.** |

> **[CMD]** Kada je konfigurisan više od jednog uređaja, `nsrmm` podrazumevano bira **prvi**. Opcija `-f <device>` to nadjačava.

```bash
# Pregled pa demontiranje konkretnog uređaja
nsrmm -C
nsrmm -u -f <ime_uredjaja>

# Verifikacija labele
nsrmm -p -f <ime_uredjaja>
```

## 49.9 Režimi volumena — `nsrmm -o` **[CMD]**

> `-o mode` postavlja režim **volumena, save set-a ili save set clone instance.** Režim može biti jedan od:

```
[not]recyclable   [not]readonly   [not]scan   [not]full
[not]offsite      [not]manual     [not]suspect
```

```bash
# Uklanjanje scan needed zastavice sa volumena
nsrmm -o notscan <volume_name>

# Postavljanje volumena u read-only
nsrmm -o readonly <volume_name>

# Označavanje save set-a kao suspect
nsrmm -o suspect -S <ssid>
```

> **[CMD]** Postavljanje recyclable save set clone instance na `not recyclable` primorava i pridruženi save set i volumen da postanu `not recyclable`.

## 49.10 Označavanje volumena — pravila koja se ne smeju prekršiti

| Komanda | Šta radi **[CMD]** | Rizik |
|---|---|---|
| `nsrmm -l` | **Označava (inicijalizuje)** volumen da bi ga NetWorker koristio i prepoznavao. Označavanje se izvodi **nakon** što je volumen fizički učitan u uređaj. **Operacija ne uspeva ako se pokuša nad repliciranim volumenom.** | **Uništava postojeće podatke** |
| `nsrmm -R` | **Ponovo označava** volumen. Prepisuje labelu i **čisti NetWorker indekse od svih korisničkih fajlova prethodno snimljenih na volumen.** Deo informacija o upotrebi volumena se zadržava. | **Uništava postojeće podatke** |
| `nsrmm -B` | Verifikuje da volumen **nema** čitljivu NetWorker labelu. Ako labela postoji i čitljiva je, **operacija označavanja se otkazuje uz poruku o grešci.** Volumen se sme označiti samo ako već nema čitljivu labelu. | Zaštitna opcija |
| `nsrmm -E` | **Briše medij u uređaju, uključujući labelu i sve NetWorker strukture direktorijuma.** Implementirano za Data Domain i `adv_file` uređaje. | **Uništava sve** |

> ### CAUTION
>
> **[DR]** *Do not relabel the volume when you create the device. Relabeling a volume with bootstrap backups, or any other backups, renders the data unrecoverable.*
>
> Ne označavati volumen ponovo pri kreiranju uređaja. Ponovno označavanje volumena sa bootstrap backup-ima, ili bilo kojim drugim backup-ima, **čini podatke neoporavljivim.**
>
> **[DR]** Za disk uređaje: **ne dozvoliti čarobnjaku da označi disk volumen.** Opcija **Label and Mount** je podrazumevano izabrana u prozoru **Device Label and Mount** — **isključiti je.**

> **Dobra vest:** `nsrmm -l` **ne uspeva nad repliciranim volumenom** **[CMD]**. To je ugrađena zaštita u vault scenariju — ali ne oslanjati se na nju kao jedinu.

**Bezbedna praksa u vault-u:**

```bash
# 1. Uređaj kreirati kao read-only PRE montiranja
nsradmin -s <server> <<'EOF'
. type: NSR device; name: <ime_uredjaja>
update read only: yes
EOF

# 2. Verifikovati labelu bez pisanja
nsrmm -p -f <ime_uredjaja>

# 3. Tek onda montirati
nsrmm -m -f <ime_uredjaja>
```

## 49.11 Brisanje save set-a iz baza **[CMD]**

```bash
nsrmm -d -S <ssid>[/<cloneid>]
nsrmm -d -i <ssid_file>
nsrmm -d -V <volid>
```

> `-d` **briše client file indekse i zapise media baze** iz NetWorker baza. Radnja **ne uništava volumen**, nego uklanja sve reference koje NetWorker koristi za volumen i korisničke fajlove na njemu.
>
> **Preporučuje se upotreba dugog formata ssid** radi jedinstvene identifikacije backup instanci pri agregiranju save set-ova između data zone-a:
> ```bash
> mminfo -r "ssid(53)"
> ```

> **[CMD] Ograničenja vezana za Retention Lock:**
> - Ako se brisanje pokuša nad kopijom save set-a koja ima **Data Domain Retention Lock**, pokušaj **ne uspeva**.
> - Ako se brisanje pokuša nad save set-om **bez navođenja clone id-a**, operacija ne uspeva ako **bilo koja** kopija tog save set-a ima Retention Lock.
> - Operacija **ne uspeva ako se pokuša nad repliciranim save set-om.**

> **Za Cyber Recovery vault ovo je očekivano i poželjno ponašanje** — zaključane kopije se ne mogu obrisati ni greškom ni namerno.

## 49.12 Provera na DD strani

```bash
ssh sysadmin@<vault_dd>

# Storage unit-i i njihovi DD Boost korisnici
sysadmin@dd-vault# ddboost storage-unit show

# MTree-ovi, uključujući replicirane koji nisu storage unit-i
sysadmin@dd-vault# mtree list

# Replikacija
sysadmin@dd-vault# replication show config
sysadmin@dd-vault# replication status

# Korisnici i UID-ovi
sysadmin@dd-vault# user show list
sysadmin@dd-vault# ddboost user show

# Zauzeće
sysadmin@dd-vault# filesys show space
```

**Uparivanje DD i NetWorker pogleda:**

| DD strana | NetWorker strana |
|---|---|
| `ddboost storage-unit show` → ime storage unit-a | prednji deo `device access information` |
| `mtree list` → putanja MTree-a | fizička lokacija podataka |
| `user show list` → UID | mora odgovarati produkcijskom DD Boost UID-u |

---

# 50. Katalog: media baza i client file indeksi

## 50.1 Podela odgovornosti

| | Media baza | Client file index |
|---|---|---|
| **Lokacija** | `/nsr/mm` | `/nsr/index/<klijent>` |
| **Odgovara na pitanje** | „Gde se nalazi save set?" | „Koji fajlovi su u save set-u?" |
| **Granularnost** | Save set | Pojedinačan fajl |
| **Komanda za upit** | `mminfo` | `nsrinfo`, `nsrls` |
| **Politika koja upravlja** | **Retention policy** | **Browse policy** |
| **Neophodna za oporavak?** | **Da** | **Ne uvek** — vidi 50.6 |
| **U bootstrap-u** | Da | Zaseban save set |

## 50.2 Browse policy i retention policy **[CMD]**

Ove dve politike deluju nezavisno i njihova razlika je izvor mnogih nesporazuma.

> **[CMD]** Zapisi koji su u online file indeksu duže od perioda zadatog **browse politikom** klijenta se **uklanjaju**. Save set-ovi koji postoje duže od perioda zadatog **retention politikom** klijenta se **označavaju kao recyclable** u media indeksu.
>
> Kada su svi save set-ovi na volumenu označeni kao recyclable, **volumen se smatra recyclable**. Recyclable volumene NetWorker može izabrati (i jukebox automatski ponovo označiti) kada je potreban writable volumen za nove backup-e. **Kada se recyclable volumen ponovo iskoristi, stari podaci se brišu i više nisu oporavljivi.**

```
                    browse policy istekla        retention policy istekla
                            │                              │
   backup ──────────────────┼──────────────────────────────┼──────────────►
            browsable       │      recoverable             │   recyclable
      (recover ga vidi)     │  (scanner ga može vratiti)   │  (može biti prepisan)
```

> **Praktična posledica u DR-u [DR]:** podrazumevana retencija klijentskog resursa NetWorker servera je **mesec dana**. Ako se ne postavi na **`Decade`**, save set-ovi servera sa retencijom dužom od mesec dana se **odbacuju** pri oporavku (vidi `07c`, odeljak 53.8).

## 50.3 Statusi save set-a **[CMD]**

`nsrim` dodeljuje jedan od četiri statusa. Ovo su vrednosti koje ćete videti u izlazu `nsrim -v`:

| Status | Značenje |
|---|---|
| **`browse`** | Zapisi fajlova su **browsable** — save set fajlovi i dalje postoje u online indeksu. Lako se vraćaju NetWorker recover mehanizmima. |
| **`recover`** | Starost save set-a **ne prelazi** retention politiku, ali su njegovi zapisi **očišćeni iz online indeksa**. Save set se može oporaviti sa backup medija komandom `recover`. `scanner` se takođe može koristiti, ali **korisnici treba prvo da probaju `recover`.** |
| **`recycle`** | Save set je stariji od pridružene retention politike i **može biti prepisan (obrisan)** kada se medij reciklira. Do reciklaže je i dalje oporavljiv sa medija. **Recyclable save set-ovi disk familije se uklanjaju sa volumena i iz media baze — podaci više nisu oporavljivi.** |
| **`delete`** | Save set će biti obrisan iz media baze. **`nsrim` briše samo recyclable save set-ove koji imaju nula fajlova.** |

### Modifikatori statusa **[CMD]**

| Modifikator | Značenje |
|---|---|
| `(archive)` | Save set **nikada ne ističe** i izuzet je od bilo kakve promene statusa |
| `(migration)` | Kreirala ga je aplikacija za migraciju fajlova; nikada ne ističe, izuzet od promene statusa |
| `(scanned in)` | Save set je vraćen komandom `scanner` i **izuzet je od promene statusa** |
| `(aborted)` | Save set upitne veličine koji zauzima prostor na backup mediju |

### Prelazi statusa **[CMD]**

Kada `nsrim` promeni status, ispisuje simbol prelaza `->` i novi status:

```
17221062 3/05/92 f 23115 files 158 MB recycle
17212499 3/19/92 f   625 files  26 MB recover(aborted)->recycle
17224025 5/23/92 i     0 files   0 KB recover->recycle->delete
```

## 50.4 Zastavice u `mminfo -v` izlazu **[CMD]**

Ovo je referenca koju vredi imati pri ruci — četiri pozicije zastavica.

### Prva zastavica — koji deo save set-a je na volumenu

| Zastavica | Značenje |
|---|---|
| **`c`** | Save set je **u celini** na volumenu (**complete**) |
| **`h`** | Save set obuhvata više volumena; **glava (head)** je na ovom volumenu |
| **`m`** | Save set obuhvata više volumena; **srednji (middle)** deo je na ovom volumenu. Može biti više srednjih delova. |
| **`t`** | **Rep (tail)** save set-a koji obuhvata više volumena je na ovom volumenu |

### Druga zastavica — status save set-a

| Zastavica | Značenje |
|---|---|
| **`b`** | Save set je u online indeksu i **browsable** komandom `recover` |
| **`r`** | Save set **nije** u online indeksu i **oporavljiv je komandom `scanner`** |
| **`E`** | Save set je označen kao **eligible for recycling** i može biti prepisan u bilo kom trenutku |
| **`a`** | Save je **prekinut (aborted)** pre završetka. Prekinute save set-ove `nsrck` uklanja iz online file indeksa. |
| **`i`** | Save je **još u toku (in progress)** |

### Treća zastavica (opciona) — tip save set-a

| Zastavica | Značenje |
|---|---|
| **`N`** | NDMP save set |
| **`R`** | Raw partition backup (NetWorker moduli — Oracle, Sybase i drugi). **Ne znači da save set sadrži fajlove koji koriste `rawasm` direktivu.** |
| **`P`** | Snapshot save set |
| **`k`** | Checkpoint-enabled save set |
| **`ak`** | Prvi i svi međupartial save set-ovi |
| **`bk`** | Kompletan ili finalni partial save set |

### Četvrta zastavica (opciona)

| Zastavica | Značenje |
|---|---|
| **`s`** | NDMP save set je backup-ovan preko `nsrdsa_save` na NetWorker storage node |

## 50.5 `mminfo` — praktični recepti

### Podrazumevano ponašanje **[CMD]**

> Bez ikakvih opcija, `mminfo` prikazuje informacije o save set-ovima koji su se **ispravno završili od ponoći prethodnog dana** i **još su u online file indeksu (browsable)**.
>
> Za svaki save set se ispisuje: ime volumena, ime klijenta, datum kreiranja, veličina snimljena na tom volumenu, nivo save set-a i ime save set-a.

> **Nivo** prikazuje `full`, `incr`, `migration` ili 1–9. **Nivo se čuva samo za zakazane save operacije i migraciju fajlova** — save set-ovi nastali eksplicitnim pokretanjem `save` komande (**ad hoc** save) nemaju pridružen nivo.

### Recepti

```bash
# Sve, uključujući prekinute, purged, nepotpune i recoverable save set-ove
mminfo -a -v

# Bootstrap-ovi iz poslednjih pet nedelja
mminfo -B
mminfo -N bootstrap -t '5 weeks ago' -avot \
  -r 'savetime(24),space(2),ssid' \
  -r 'mediafile(6),mediarec(6),space(2),volume' \
  -r 'barcode,location,device,access_info'

# Browse i retention vremena po save set-u
mminfo -p

# Save set-ovi jednog klijenta u prethodnoj nedelji
mminfo -N /usr -c venus
mminfo -q 'name=/usr,client=venus'

# Save set-ovi jednog klijenta na jednom volumenu
mminfo -N /usr -c venus mars.001
mminfo -q 'name=/usr,client=venus,volume=mars.001'

# Save set-ovi sa više od jedne kopije, sortirano po vremenu i klijentu
mminfo -otc -v -q 'copies>1'

# Save set ID u dugom formatu (53 znaka) + zastavice
mminfo -av -r 'volume, name, savetime, ssflags, clflags, ssid(53)'

# Više klijenata odjednom (lista sa zarezom)
mminfo -av -q 'client=pegasus,client=avalon'
```

### Sintaksa custom upita i izveštaja **[CMD]**

```
mminfo [-aIkvV] [-o order] [-s server] [-x exportspec] [report] [query] [filter] [volname...]
  <report>: [-m | -p | -k | -B | -C | -O | -S | -L | -X | -r reportspec]
  <query>:  [-c client] [-l] [-N name] [-t time] [-q queryspec]
  <filter>: [-A attributes]
```

**Pravila upita [CMD]:**

| Pravilo | Primer |
|---|---|
| Numerički opsezi se zadaju kao **dva odvojena ograničenja** | `%used>20,%used<80` |
| String atributi su **liste** — svaka vrednost je zasebno ograničenje jednakosti | `client=pegasus,client=avalon` |
| Isti atribut više puta daje **ili** logiku | `'group=Default, group=Test'` |
| Vremena su u `nsr_getdate(3)` formatu | `-t 'four months ago'` |
| **Vremena sa zarezima moraju biti u navodnicima** | |
| `forever` kao datum isteka znači da save set ili volumen **nikada ne ističe** | |
| `undef` se prikazuje kada je vreme nedefinisano | |

**Format `reportspec` [CMD]:**

```
name [ (width) ] [ , name [ (width) ] ... ]
```

> **[CMD]** Više `-r` opcija može se navesti. Redosled kolona ide sleva nadesno i odgovara redosledu imena atributa. Prelom reda unutar logičkog reda se postiže atributom `newline`. **Ako vrednost ne stane u traženu širinu, naredne vrednosti se pomeraju udesno** (vrednosti se skraćuju na 256 znakova).

> **[CMD]** Neka zaglavlja kolona imaju kratku i dugu verziju teksta. **Kada je kolona dovoljno široka, prikazuje se dugo zaglavlje**; inače kratko. U ne-US locale-ima zaglavlje se možda neće poravnati sa podacima ako je podrazumevana širina neodgovarajuća — **zadati širinu uz ime atributa.**

```bash
# Primer poravnanja u ne-US locale
mminfo -avot -q group=Default \
  -r "savetime(17), ssbrowse(17), ssretent(17), ssid, client, name"

# Široka kolona za pun datum
mminfo -avot -r "volume, client, savetime(40), sumsize, level, ssid, name, sumflags"
```

> **[CMD]** Ako je kolona vrlo uska (manje od 17 znakova), prikazuje se **samo datum**. Kolone široke 22 znaka obično ispisuju pun datum.

**Skalni faktori [CMD]:** `KB`, `MB`, `GB`, `TB`, `PB`, `EB` mogu se dodati na `size` i `kbsize` atribute. Podrazumevana skala u upitima je **bajt**; za `kbsize` atribute — **kilobajt**.

## 50.6 Kada je client file index neophodan

> **[DR]** Client file index **nije uvek potreban za oporavak podataka**, ali se preporučuje da se indeksi backup-uju i da budu dostupni pri disaster recovery-ju. **Dostupnost CFI-ja bitno utiče na potpuno vraćanje backup i recovery servisa** posle DR-a i određuje vreme potrebno da se NetWorker server vrati u potpuno funkcionalno stanje.

| Scenario | Potreban CFI? |
|---|---|
| Vraćanje celog save set-a | **Ne** — dovoljna je media baza |
| Pregledanje (browse) i izbor pojedinačnih fajlova | **Da** |
| Oporavak baza podataka kroz module | **Da** **[CR]** |
| `scanner` vraćanje sa medija | Ne — `scanner -i` ga gradi |

## 50.7 `nsrls` — statistika indeksa **[CMD]**

> `nsrls` bez opcija ispisuje **broj zapisa u online indeksu** i **iskorišćenost online indeksa** u odnosu na broj kilobajta dodeljenih njegovim fajlovima. Administratori mogu ovom komandom utvrditi **koliko je fajlova klijent snimio.**
>
> **Prazna lista argumenata ispisuje statistiku za sve poznate klijente.**

```bash
nsrls                     # svi klijenti
nsrls <ime_klijenta>
```

Primer izlaza **[CMD]**:

```
% nsrls jupiter
/space2/nsr/index/jupiter: 292170 records requiring 50 MB
/space2/nsr/index/jupiter is currently 100% utilized
```

**Dijagnostika [CMD]:** `... is not a registered client` — navedeni klijent nije važeći NetWorker klijent.

## 50.8 `nsrinfo` — sadržaj indeksa **[CMD]**

> `nsrinfo` generiše izveštaje o sadržaju client file indeksa. Uz obavezno ime klijenta i bez opcija, ispisuje izveštaj o **svim fajlovima i objektima, jedan po redu**, u backup name space-u tog klijenta.

```
nsrinfo [-vV] [-s server | -L] [-n namespace] [-N filename] [-t time] [-T]
        [-X application] [-x exportspec] client
```

| Opcija | Značenje |
|---|---|
| `-v` | Verbose — uz ime fajla ispisuje **tip fajla, interni identifikator indeksa, veličinu** (ako je UNIX fajl) i **savetime**. Može se kombinovati sa `-V`. |
| `-V` | Alternativni verbose — uz ime fajla ispisuje **offset unutar save set-a, veličinu unutar save set-a, application name space, save time** i, ako je dostupno, **vremena fajla (mtime, atime, ctime)** i **dozvole i vlasništvo** UNIX klijenta |
| `-s <server>` | Ime NetWorker sistema nad kojim se radi upit. Podrazumevano lokalni. |
| **`-L`** | **Otvara file indeks direktno, bez servera.** Koristi se za debagovanje ili **za upit nad indeksom dok NetWorker ne radi.** |
| `-n <namespace>` | Name space nad kojim se radi upit. Podrazumevano `backup`. **Polje je case sensitive.** |
| `-N <filename>` | Tačno ime fajla. Ispisuju se samo zapisi koji se tačno poklapaju. |
| `-t <time>` | Ograničava upit na **jedno tačno save vreme**. Format `nsr_getdate(3)`. |
| `-T` | Prikazuje stvarna imena backup-ovanih fajlova, **bez continuation direktorijuma** |
| `-X <application>` | Ograničava na određenu XBSA aplikaciju. Vrednosti: `All`, `Informix`, `None`. Nije case sensitive. |
| `-x <exportspec>` | Programski čitljiv izlaz: `m` daje **XML**, `c<separator>` daje vrednosti razdvojene znakom. Npr. `nsrinfo -xc,` daje CSV. |

**Podržani name space-ovi [CMD]:**

```
backup (podrazumevano), migrated, archive, nsr (interno), bbb (Block based backup),
db2, informix, iq (SAP IQ), msexch, mssql, mysql, notes (Lotus Notes),
oracle, saphana, sybase, all
```

**Primer iz dokumentacije — izveštaj o fajlovima iz najnovijeg backup-a `/usr` za klijenta `mars` [CMD]:**

```bash
# 1. Utvrditi save time najnovijeg save set-a
mminfo -r nsavetime -v -N /usr -c mars -ot | tail -1
# izlaz: 809753754

# 2. Ispisati sadržaj indeksa za taj save time
nsrinfo -t 809753754 mars
```

**Korisno u vault-u:**

```bash
# Upit nad indeksom dok NetWorker NE radi
nsrinfo -L <ime_klijenta>

# CSV izlaz za dalju obradu
nsrinfo -xc, <ime_klijenta> > /tmp/index.csv

# Provera da li konkretan fajl postoji u indeksu
nsrinfo -N /etc/passwd <ime_klijenta>
```

> **[CMD]** File indeks može čuvati zapise za sve tipove klijenata. Svaki zapis uključuje tip zapisa. **Generalno, samo klijent koji je kreirao zapis može ga dekodirati.** `nsrinfo` prepoznaje mnoge tipove, ali **potpuno dekodira samo jedan** — UNIX verzija dekodira UNIX tipove, NT verzija NT tipove. Za ostale prepoznate tipove neke informacije mogu biti nepotpune.

## 50.9 `nsrim` — šta se dešava automatski **[CMD]**

```
nsrim [-c client] [-N saveset] [-V volume] [-lnqvKMXC]
```

> `nsrim` upravlja NetWorker online file i media indeksima. **Normalno ga pokreće „Server Protection" / „Server Backup" workflow, kroz „Expiration" akciju.**
>
> Expiration akcija istekne save set-ove u media bazi na osnovu njihovog retention vremena. Kada je retention vreme dostignuto, NetWorker koristi `nsrim` proces da istekne save set.

**Šta `nsrim` radi kada save set istekne [CMD]:**

| Korak | Radnja |
|---|---|
| 1 | Uklanja informacije o save set-u iz **client file indeksa** |
| 2 | Ako podaci save set-a leže na **AFTD/DD**: uklanja informacije iz **media baze** i **uklanja same podatke sa AFTD/DD** |
| 3 | Ako podaci leže na **trakastom uređaju**: označava save set kao **recyclable** u media bazi. Kada svi save set-ovi na volumenu isteknu, **volumen postaje podoban za ponovnu upotrebu.** |

> **[CMD]** Expiration akcija se automatski kreira u **Server maintenance** workflow-u „Server Protection" politike. Podržava samo **Execute** i **Skip** rasporedne aktivnosti.

> **[CMD]** `nsrim` automatski poziva **`nsrck -L 3`** nakon ažuriranja browse i retention vremena u media bazi, radi uklanjanja client file indeksa koji su prekoračili retention politiku.

**Tipovi save set-ova u `nsrim` izlazu [CMD]:**

| Tip | Opis |
|---|---|
| **Normal** | Svi save set-ovi backup-ovani automatski, povezani sa rasporedom, retention politikom ili protection grupom |
| **Ad hocs** | Save set-ovi koje je pokrenuo korisnik; označeni dodavanjem `ad hocs` u red zaglavlja |
| **Archives** | Save set-ovi koji **nikada ne ističu automatski**; označeni dodavanjem `archives` |
| **Migrations** | Save set-ovi koji nikada ne ističu automatski, kreirani aplikacijom za migraciju fajlova |

**Primer izlaza [CMD]:**

```
mars:/usr, retention policy: Year, browse policy: Month, ad hocs
8481 browsable files of 16481 total, 89 MB recoverable of 179 MB total
```

> **[CMD]** Zaglavlje navodi tip save set-a, ime klijenta, ime save set-a i primenljive browse i retention politike. Podnožje navodi **četiri statistike**: ukupan broj browsable fajlova koji ostaju u online indeksu, ukupan broj fajlova pridruženih save set-u, i **količinu oporavljivih podataka od ukupne količine** pridružene save set-u.

```bash
# Pregled bez izmena (dry run) — samo prikaz
nsrim -n

# Verbose: ssid, datum kreiranja, nivo, broj fajlova, veličina, status
nsrim -v

# Za jednog klijenta
nsrim -c <ime_klijenta> -v
```

> **Oprez:** `nsrim` bez `-n` **menja stanje kataloga**. U vault-u, pre nego što ga pokrenete ručno, budite sigurni zašto to radite.

---

# 51. Skeniranje medija i rekonstrukcija kataloga

## 51.1 Šta `scanner` radi **[CMD]**

> `scanner` čita NetWorker medij — backup trake ili diskove — da bi:
> - **potvrdio sadržaj volumena**
> - **izdvojio save set** sa volumena
> - **ponovo izgradio NetWorker online indekse**

> **Kako je instaliran, komandu može pokretati samo super-user.** Režimi komande se mogu izmeniti tako da je pokreću i obični korisnici uz zadržavanje root privilegija — videti `nsr(1m)`.

### Zahtevi za uređaj **[CMD]**

| Situacija | Šta navesti |
|---|---|
| Uređaj se **uvek mora navesti** | Obično jedno od imena uređaja koje koristi NetWorker server |
| **Trakasti drajv** | Mora biti ime **„no-rewind on close"** uređaja (`/dev/nst0`, ne `/dev/st0`) |
| **`adv_file` uređaj, server sa read-only mirror-om** | Kada se navede read-only ime uređaja, koristi se read-write ime |
| **`adv_file` uređaj, server bez read-only mirror-a, server RADI** | Koristi se **primarna putanja** — prvi unos u `device access information` atributu |
| **`adv_file` uređaj, server NE RADI** | **Mora se navesti putanja do volumena umesto imena uređaja** |

## 51.2 Tabela sadržaja i zastavice **[CMD]**

> Kada se `scanner` pozove bez opcija ili sa `-v`, volumen na navedenom uređaju se otvara za čitanje, skenira, i generiše se **tabela sadržaja**.

Za svaki pronađeni save set ispisuje se jedan red sa: **ime klijenta, ime save set-a, save time, nivo, veličina, broj fajlova, ssid i zastavica.**

> Tabela sadržaja se zasniva na **sinhronizacionim („note") delovima** ispresecanim sa stvarnim podacima save set-a. Postoje četiri tipa:

| Zastavica | Tip | Značenje |
|---|---|---|
| **`B`** | **Begin** | Označava početak save set-a. Kada se početni deo upiše, **veličina save set-a i broj fajlova još nisu poznati.** |
| **`C`** | **Continue** | Save set je počeo na **drugom volumenu** |
| **`S`** | **Synchronize** | Mesta u save set-u odakle se **izdvajanje može nastaviti** u slučaju ranijeg oštećenja medija (granica klijentskog fajla) |
| **`E`** | **End** | Označava kraj save set-a i **izaziva ispis reda tabele sadržaja** |

> **[CMD]** Ostale note-ove (`B`, `C`, `S`) prikazuje samo opcija `-v`.

## 51.3 `-m` vs. `-i` — koji kada

Ovo je najvažnija odluka pri radu sa `scanner`-om.

| Opcija | Šta gradi **[CMD]** |
|---|---|
| **`-m`** | Ponovo gradi **samo media indekse** za pročitane volumene. Ako se navede jedan save set sa `-S`, u media indeks se kopiraju samo zapisi za taj save set. **Podaci save set-a se pišu na standardni izlaz** — preusmeriti ih. Media baza **ne zadržava „scanned-in" status**; nema više zastavice koja to pokazuje u `ssflags` polju. Save set dobija **novu browse i retention politiku** koja počinje da teče od trenutka skeniranja. |
| **`-i`** | Ponovo gradi **i media i online file indekse** sa pročitanih volumena. Takođe **ažurira zapis save set-a u media bazi tako da bude browsable.** Ako se navede jedan save set sa `-S`, u online file indeks se kopiraju samo zapisi za taj save set. |

### Preporučeni redosled **[CMD]**

> Za verziju 6.0 i noviju: ako imate medij koji sadrži **index backup-e** uz data backup-e, **preporučeni način vraćanja indeksa je**:
>
> 1. Pokrenuti **`scanner -m`** da se ponovo učitaju zapisi media baze za **index i data backup-e**
> 2. Zatim pokrenuti **`nsrck -L7 -t <date> <clientname>`** da se oporavi indeks klijenta u trenutku tih backup-a
>
> To vraća zapise indeksa za taj trenutak nazad u indeks.
>
> **Međutim, ako imate medije za koje ne postoje index backup-i, tada morate koristiti `-i` opciju** da rekonstruišete zapise indeksa.

```bash
# ── Preporučeni put (postoje index backup-i) ──
scanner -m <device>
nsrck -L7 -t "<datum>" <ime_klijenta>

# ── Rezervni put (nema index backup-a) ──
scanner -i <device>
```

> **[DR]** `scanner -i` može trajati **veoma dugo**, posebno na velikom disk volumenu. Za volumene za koje ne sumnjate da imaju save set-ove backup-ovane posle poslednjeg bootstrap-a, korak se može preskočiti.

## 51.4 Referenca opcija **[CMD]**

```
scanner [options] { -B | -E } { -S ssid | -I ssid_file } [-im] [-z]
        [-e retention range] { -Y start time | -T end time } device

scanner [options] -i [-S ssid] [-I ssid_file] [-c client] [-N name]
        [-y retention time] [-e retention range] { -Y | -T } device

scanner [options] -m [-S ssid] [-I ssid_file] [-y retention time]
        [-e retention range] { -Y | -T } device

options: [-npqvkF] [-b pool] [-f file] [-r record] [-s server] [-t type]
         [-V volume name] [-Z datazone-id]
```

| Opcija | Namena |
|---|---|
| `-B` | Uz `-S`: navedeni ssid se **označava kao bootstrap** |
| `-E` | Uz `-S`: ssid se označava kao bootstrap za **server stariji od NetWorker 2015** |
| `-b <pool>` | Kojem pool-u volumen treba da pripada. Primenjuje se **samo na verzije koje ne čuvaju pool informaciju na mediju.** Za volumene gde je pool informacija na mediju, **medij se mora ponovo označiti** (uz uništavanje podataka) da bi se dodelio drugom pool-u. |
| `-c <client>` | Obrađuje samo save set-ove sa navedenog klijenta. Može se koristiti više puta i uz `-N`, **ali samo uz `-i` ili `-x`.** |
| `-N <name>` | Obrađuje samo save set-ove sa navedenim imenom (**samo doslovan string**) |
| `-e <retention range>` | Datumski opseg za proveru isteka save set-ova u media bazi (`nsr_getdate(3)` format). Generiše listu skeniranih save set-ova koji ističu u tom periodu. |
| **`-F`** | **Prisiljava ponovnu izgradnju zapisa save set-a** bez obzira na to da li postoje u media bazi. Save set se skenira i **njegovi metapodaci se ažuriraju, uključujući novoizračunato retention vreme.** Bez ove opcije `scanner` **neće** skenirati i ponovo graditi postojeći zapis save set-a. |
| `-f <file number>` | Počinje skeniranje od zadatog broja media fajla. **Nije korisno na medijima kao što su optički diskovi i file device tipovi.** |
| `-r <record>` | Počinje skeniranje od zadatog broja media zapisa |
| `-I <ssid_file>` | Dodaje save set-ove u listu za obradu, jedan ssid po redu |
| `-n` | **Proverava sav medij bez ponovne izgradnje** media ili index baza. **Uz `-i` daje najpotpuniju proveru medija, bez ikakve izmene baza.** |
| `-p` | Ispisuje informacije **i save set note-ove** dok se obrađuju |
| `-q` | Prikazuje samo greške i važne poruke |
| `-s <server>` | Kontrolni server pri upotrebi `scanner`-a na storage node-u |
| `-t <type>` | Tip medija, npr. `optical` ili `8mm 5GB`. Normalno se dobija od servera, ali samo ako se koristi poznat uređaj. |
| `-y <retention time>` | Retention vreme za završene clone instance save set-ova na volumenima koji se skeniraju. **Važi samo uz `-i` ili `-m`.** Uz `-F` menja retention i za save set-ove koji već postoje u media bazi. |
| `-Y <start time>` / `-T <end time>` | Ograničava vremenski prozor skeniranja |
| `-z` | **End-silently** — `scanner` neće tražiti sledeći volumen kada save set prelazi na drugi volumen; završava po čitanju prvog volumena. |
| `-S <ssid>` | Izdvaja navedeni save set. Može se koristiti više puta. **Bez `-i` ili `-m`, `scanner` traži veličinu bloka volumena — ali samo ako labela volumena nije čitljiva.** Uz `-B` ili `-E`, ssid se tretira kao bootstrap; **dozvoljen je samo jedan ssid u tom slučaju.** |
| `-V <volume name>` | Ime volumena pri skeniranju cloud volumena. **Cloud uređaji obično sadrže više volumena**, pa `scanner` zahteva ime. |
| `-Z <datazone-id>` | Datazone ID NetWorker servera ako je u drugoj data zone od cloud uređaja |

## 51.5 Ograničenja skeniranja **[CMD]**

| Ograničenje | Detalj |
|---|---|
| **NDMP, DSA i Block based backup save set-ovi** | Opcija `-i` **ne rekonstruiše zapise indeksa** iz volumena. Ako imate index backup-e, koristiti `scanner` i `nsrck` kao u 51.3. |
| **Mešoviti volumeni** | Za volumene koji sadrže kombinaciju DSA / Block based i regularnih save set-ova, **`scanner -i` preskače DSA i Block based save set-ove uz grešku.** |
| **Preusmeravanje izlaza** | Uz `-m` i `-S` (bez `-i`/`-m`) podaci se pišu na **standardni izlaz** — preusmeriti ih da se izbegne neželjeno ponašanje konzole |
| **Pipe u recover program** | **Prosleđivanje NDMP ili DSA save set stream-ova u bilo koji recover program, kao što je `uasm`, nije podržano.** |

## 51.6 Save set koji obuhvata više volumena **[CMD]**

> Kada save set obuhvata više volumena, skenirati volumene **redosledom kojim su pisani**. **Svi delovi save set-a moraju biti skenirani da bi oporavak bio moguć.**

Redosled se čita iz `mminfo -B` izlaza — **redosled prikaza je redosled koji se zahteva** (vidi `07c`, odeljak 55.3).

## 51.7 Automatsko resetovanje scan needed zastavice **[CMD]**

> Pri skeniranju `adv_file` ili Data Domain volumena, `scanner` **može resetovati scan needed zastavicu** u zapisu volumena.
>
> To se dešava **samo** kada su ispunjeni **svi** uslovi:
> 1. Navedena je opcija `-i` **ili** `-m`
> 2. **Nijedna** od opcija `-c`, `-n`, `-N`, `-S` **nije** navedena
> 3. `scanner` je **uspešno skenirao sve save set-ove** na volumenu

> **Praktična posledica:** ako skenirate ciljano (`-S`, `-c`, `-N`), zastavicu morate ukloniti **ručno**:
> ```bash
> nsrmm -o notscan <volume_name>
> ```

## 51.8 Ručno uklanjanje scan needed zastavice

**Za AFTD i DD volumene [DR]** — kroz NMC:

1. **Administration → Devices → Devices** → desni klik na uređaj → **Unmount**. Zabeležiti pridruženi volumen.
2. **Administration → Media → Disk Volumes** → desni klik na volumen → **Mark Scan Needed** → izabrati **Scan is NOT needed** → **OK**
3. **Devices → Devices** → desni klik → **Mount**

**Iz konzole:**

```bash
nsrmm -C                          # utvrditi uređaj i volumen
nsrmm -u -f <device>              # demontirati
nsrmm -o notscan <volume>         # ukloniti zastavicu
nsrmm -m -f <device>              # montirati nazad
```

**Za trakaste volumene [DR]:**

```bash
# Zabeležiti file i record broj iz poruke koju NetWorker prikaže
scanner -f <file> -r <record> -i <device>
nsrmm -o notscan <volume_name>
```

Poruka koju NetWorker prikaže **[DR]**:

```
nw_server nsrd media info: Volume <volume_name> has save sets unknown to media database.
Last known file number in media database is ### and last known record number is ###.
Volume <volume_name> must be scanned; consider scanning from last known file and record numbers.
```

## 51.9 Provera bez izmena — najbezbednija komanda

Kada ne znate šta je na volumenu, a ne želite ništa da promenite:

```bash
scanner -n -i <device>
```

> **[CMD]** `-n` proverava sav medij **bez ponovne izgradnje** media ili index baza. Uz `-i` opciju daje **najpotpuniju dostupnu proveru medija, bez ikakve izmene baza.**

To je prva komanda koju treba pokrenuti nad nepoznatim volumenom u vault-u.

---

# Dodatak: Kartica komandi za storage i katalog

## Uređaji

```bash
nsrmm -C                                  # šta je montirano na čemu
nsrmm -C -v                               # verbose
nsradmin -c "type:NSR device"             # puni ekranski prikaz uređaja
cd /nsr/res && nsradmin -d nsrdb          # kada server ne radi
nsrmm -p -f <device>                      # verifikacija labele (demontira!)
nsrmm -m -f <device>                      # montiranje
nsrmm -m -r -f <device>                   # montiranje read-only
nsrmm -u -f <device>                      # demontiranje
nsrmm -H -f <device>                      # softverski reset uređaja
```

## Volumeni

```bash
mminfo -m                                 # svi volumeni
mminfo -m -v                              # + volid, tip medija
mminfo -m -V                              # + zastavice d/r
mminfo -a -r 'volume,%used,pool,location' -q '!full'
nsrmm -o notscan <volume>                 # uklanjanje scan needed
nsrmm -o readonly <volume>                # read-only režim
nsrmm -g <volume>                         # uklanjanje replica zastavice
```

## Save set-ovi

```bash
mminfo                                    # browsable od ponoći prethodnog dana
mminfo -a -v                              # sve, uključujući aborted i recoverable
mminfo -B                                 # bootstrap-ovi
mminfo -p                                 # browse i retention vremena
mminfo -q 'name=/usr,client=venus'
mminfo -otc -v -q 'copies>1'
mminfo -av -r 'volume,name,savetime,ssflags,clflags,ssid(53)'
nsrmm -d -S <ssid>/<cloneid>              # brisanje save set-a iz baza
```

## Indeksi

```bash
nsrls                                     # statistika za sve klijente
nsrls <klijent>
nsrinfo <klijent>                         # sadržaj indeksa
nsrinfo -v <klijent>                      # + tip, veličina, savetime
nsrinfo -L <klijent>                      # dok NetWorker NE radi
nsrinfo -t <savetime> <klijent>           # jedan tačan save time
nsrinfo -xc, <klijent> > /tmp/index.csv   # CSV izlaz
nsrck -L 6 <klijent>                      # dublja provera
nsrim -n -v                               # pregled bez izmena
```

## Skeniranje

```bash
scanner -n -i <device>                    # provera BEZ izmena — počni odavde
scanner -B <device>                       # pronalaženje bootstrap ssid-a
scanner -m <device>                       # samo media baza
scanner -i <device>                       # media baza + indeksi
scanner -m -S <ssid>/<cloneid> <device>   # jedan save set u media bazu
scanner -f <file> -r <record> -i <device> # traka od zadate pozicije
scanner -i -V <volume> -Z <dzid> <device> # cloud volumen
```

## DD strana

```bash
ddboost storage-unit show
ddboost storage-unit modify <mtree> user <ddboost-user>
ddboost user show
mtree list
replication show config
replication status
user show list
filesys show space
```

## Provera po tipu uređaja

```bash
# AFTD / NFS
mount | grep -i nfs ; df -h ; showmount -e <dd>

# Traka
lsscsi -g ; ls -l /dev/nst* ; mt -f /dev/nst0 status

# DD Boost — NE vidi se u mount
nsrmm -C ; ssh sysadmin@<dd> "ddboost storage-unit show"
```

---

*Kraj Dela VII-b. Videti i: `07a` (anatomija instalacije), `07c` (kompletan tok oporavka), `07d` (logovi i dijagnostika po simptomu).*
