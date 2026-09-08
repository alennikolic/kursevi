# DEO VII-c — OD NOVE INSTANCE DO OPORAVKA

**Dell PowerProtect Cyber Recovery 20.3 — Uputstvo za instalaciju i integraciju**
Verzija dokumenta: 0.1 | Namena: System Engineer
**Referentni OS:** Red Hat Enterprise Linux 9.6 | **Referentni NetWorker:** 19.11 / 19.13

> **Izvori:**
> - *Dell NetWorker 19.11 Command Reference Guide* — **[CMD]**
> - *Dell NetWorker 19.13 Administration Guide* — **[ADM]**
> - *Dell NetWorker Server Disaster Recovery and Availability Best Practices Guide 19.13* — **[DR]**
> - *Dell PowerProtect Cyber Recovery 20.3 Product Guide / Installation and Upgrade Guide* — **[CR]**

> **Šta ovaj deo pokriva:** kompletan tok od potpuno nove NetWorker instance u vault-u do funkcionalnog oporavka nad repliciranim podacima, sa naglaskom na rad iz Linux konzole kada NMC nije dostupan.

---

## Sadržaj

- [52. Polazna tačka](#52-polazna-tačka)
- [53. Priprema instance](#53-priprema-instance)
- [54. Podmetanje repliciranog storage-a](#54-podmetanje-repliciranog-storage-a)
- [55. Pronalaženje bootstrap-a bez media baze](#55-pronalaženje-bootstrap-a-bez-media-baze)
- [56. `nsrdr` — puna procedura](#56-nsrdr--puna-procedura)
- [57. Posle `nsrdr`](#57-posle-nsrdr)
- [58. Oporavak podataka](#58-oporavak-podataka)
- [59. Kada oporavak ne uspe](#59-kada-oporavak-ne-uspe)
- [Dodatak: Kartica komandi](#dodatak-kartica-komandi)

---

# 52. Polazna tačka

## 52.1 Situacija

U vault-u imate:

| Element | Stanje |
|---|---|
| NetWorker instanca | Sveža, neinicijalizovana, ista verzija kao produkcijska |
| Vault DD sistem | Sadrži replicirani MTree sa produkcije |
| Cyber Recovery | Kreirao PIT kopiju, zaključao je |
| Media baza NetWorker-a | **Prazna** |
| Resource baza | **Prazna** — bez klijenata, uređaja, pool-ova |
| Client file indeksi | **Prazni** |

Sve što znate o backup-ima nalazi se **na medijumu**, ne u NetWorker-u. Zadatak je da NetWorker to pročita i rekonstruiše svoje baze.

## 52.2 Dva puta

| Put | Kada | Ko izvršava |
|---|---|---|
| **Automatizovani CR recovery** | Standardan slučaj: jedan MTree, podržana verzija | Cyber Recovery pokreće `nsrdr` umesto vas |
| **Ručni `nsrdr`** | Više MTree-ova, neuspeo automat, netipična konfiguracija | Vi, iz konzole |

> **[CR]** Cyber Recovery **ne podržava automatski** oporavak NetWorker servera sa više od jednog MTree-a. Ručni oporavak je moguć — kontaktirati Dell Support.

Ovaj deo opisuje **ručni put**. Automatizovani je u Delu IV. I u automatizovanom slučaju vredi razumeti šta se dešava ispod, jer kada job padne — dijagnostika je ista.

## 52.3 Šta morate imati pre nego što išta kucnete

| # | Stavka | Kako proveriti |
|---|---|---|
| 1 | Ista **major verzija** NetWorker-a kao na produkciji | `rpm -qa \| grep -i lgto` |
| 2 | Instalirani paketi: **client, storage node, Authentication service** **[DR]** | isto |
| 3 | NetWorker instaliran na **originalnu lokaciju** **[CMD]** | |
| 4 | Isti **FQDN** kao produkcijski server | `hostname -f` |
| 5 | Pokrenut `authc_configure.sh` **[DR]** | |
| 6 | Pristup DD sistemu u vault-u (SSH, kredencijali) | |
| 7 | DD Boost korisnik sa **istim UID-om** kao produkcijski | `user show list` na DD |
| 8 | Poznat naziv MTree-a / storage unit-a sa podacima | |
| 9 | Poznato ime foldera sa bootstrap backup-ima (ako postoji) | |
| 10 | Slobodan prostor na `/nsr` | `df -h /nsr` |

> **[CMD] IMPORTANT:** pre upotrebe `nsrdr`-a, NetWorker mora biti **potpuno instaliran i ispravno konfigurisan**. Ako hostu nedostaje bilo koji NetWorker binarni fajl, ponovo instalirajte softver iz distribucionih fajlova pre pokretanja `nsrdr`-a.

## 52.4 Napomena o licenciranju

> **[CMD]** Ako ste oporavili NetWorker server bootstrap **na drugi host** (npr. posle većeg hardverskog otkaza), NetWorker Licensing softver detektuje premeštanje. Morate kontaktirati Licensing **u roku od 15 dana** radi *host transfer affidavit* postupka i dobijanja novih autorizacionih kodova, zasnovanih na hostid-u nove mašine.
>
> Ako nove kodove ne unesete u roku od 15 dana, **NetWorker server se onemogućava i možete izvršavati samo oporavke**. Novi backup-i nisu mogući dok se server ponovo ne registruje.

U vault scenariju ovo se gotovo uvek aktivira, jer je vault instanca po definiciji drugi host. **Planirati unapred sa kupcem.**

---

# 53. Priprema instance

## 53.1 Provera verzije i paketa

```bash
# Instalirani NetWorker paketi i verzije
rpm -qa | grep -i lgto

# Detalji o glavnom paketu
rpm -qi lgtoserv 2>/dev/null | head -20
```

Uporediti sa produkcijskim serverom. **Major verzija mora biti ista.**

## 53.2 Kritično: simbolički link na `/nsr`

> **[CMD]** `/nsr` — ako je ovo bio simbolički link u trenutku kreiranja bootstrap save set-a, **morate ponovo kreirati simbolički link pre pokretanja `nsrdr` komande.**

```bash
# Provera da li je /nsr link ili direktorijum
ls -ld /nsr
```

Ista logika važi i za pojedinačne poddirektorijume:

> **[DR]** Simbolički linkovi ka bootstrap save set-ovima **ne oporavljaju se u podrazumevani direktorijum**, već u ciljni direktorijum simboličkog linka. Na primer, ako je `/nsr/res` bootstrap save set povezan sa `/bigres/res` direktorijumom, resource baza se oporavlja u `/bigres/res`. **Obezbediti dovoljno slobodnog prostora u ciljnom direktorijumu.**

## 53.3 Hostname, FQDN i aliasi

```bash
hostname
hostname -f
```

> **[DR]** NetWorker server imenovati **istim imenom** koje je korišćeno pre. Nova instalacija mora biti konfigurisana sa istim fully qualified name.
> Shortname uređaja imenovati isto kao pre.

Posle oporavka resursa proveriti atribut **Aliases** klijentskog resursa NetWorker servera — mora sadržati i shortname i FQDN. **[DR]**

## 53.4 Rezolucija imena — obavezno u vault-u

U vault-u DNS često nije dostupan. Bez ove pripreme NetWorker server startuje veoma sporo ili postaje neodgovarajući. **[DR]**

```bash
# 1. Izmeniti redosled rezolucije
vi /etc/nsswitch.conf
```

```
# Umesto:
hosts: files dns myhostname

# Postaviti:
hosts: files
```

```bash
# 2. Popuniti lokalni hosts fajl
vi /etc/hosts
```

```
# Format: FQDN prvi, pa shortname
10.10.2.5   nwserver.example.com   nwserver

# Za klijente čija je IP adresa nepoznata — loopback
127.0.0.1   nepoznat-klijent1.example.com   nepoznat-klijent1
127.0.0.1   nepoznat-klijent2.example.com   nepoznat-klijent2
```

> **[DR]** `127.0.0.1` je standardna loopback adresa. Kada se NetWorker server podigne, izvršava DNS proveru za svakog klijenta. Za klijente koji su offline ili nedostupni, server se povezuje na `127.0.0.1` što se odmah vraća na isti host. Time server postaje dostupan **brže**, umesto da čeka rezoluciju svih klijenata.

> **[DR]** Ako je client name NetWorker servera originalno bio **shortname**, onda shortname mora biti **prvi**, pa longname.

**Alternativa — marker fajl:**

> **[DR]** `nsrdr` kreira fajl `nsr_disaster_recovery_mode` u `/nsr/debug`. Ako NetWorker server pronađe taj fajl, preskače popunjavanje DNS keša i RAP consistency provere za klijente. Kada `nsrdr` završi ili izađe zbog greške, fajl briše sam.
>
> **Ako dođe do pada i `nsrdr` ne izađe uspešno, fajl obrisati ručno.** Posle završenog DR-a restartovati NetWorker server da se DNS keš ispravno popuni.

```bash
ls -l /nsr/debug/nsr_disaster_recovery_mode
rm -f /nsr/debug/nsr_disaster_recovery_mode   # samo ako je zaostao
```

## 53.5 `nsrtomcat` UID

> **[DR]** Pri pokretanju `nsrdr` ka **drugom** NetWorker serveru na Linux platformi, korisnički ID koji koristi `nsrtomcat` korisnik mora biti **isti** na originalnom i na ciljnom NetWorker serveru.

```bash
id nsrtomcat
```

Uporediti sa produkcijskim serverom. Ako se razlikuje, uskladiti pre oporavka.

## 53.6 Provera Authentication Service baze

> **[DR]** Pre DR-a proveriti da direktorijum authentication baze **ne sadrži oporavljeni fajl baze noviji od bootstrap-a** koji se oporavlja. Ime takvog fajla je oblika `authcdb.h2.db.<timestamp>`.

```bash
find /nsr -name "authcdb.h2.db*" -exec ls -l {} \;
```

## 53.7 KRITIČNO: postavljanje server state atributa

> ### Ovo je najopasniji tihi otkaz u celoj proceduri
>
> **[CMD]** *When you recover a NetWorker media database, you must first ensure the "server state" attribute in the NSR resource is set to "disaster recovery". If this is not done, the media database will not be imported **even if nsrdr does not report an error**.*
>
> Prevod: ako server state nije postavljen, **media baza se neće uvesti, a `nsrdr` neće prijaviti grešku**. Dobićete naizgled uspešan oporavak sa praznom media bazom.

### Stanja servera **[ADM]**

Od NetWorker verzije 19.8, server se može postaviti u definisano stanje. Atribut se zove **`server state`** u **NSR** resursu. Podrazumevana vrednost je `active`.

| Stanje | Opis | Bira korisnik | Dozvoljeno | Zabranjeno |
|---|---|---|---|---|
| **Initiating** | Trenutno stanje nepoznato zbog nedavnog starta demona | Ne | Ništa | Akcije zavisne od stanja |
| **Changing** | Prelaz iz jednog stanja u drugo | Ne | Ništa | Akcije zavisne od stanja |
| **Active** | Očekivano normalno radno stanje | **Da** | Sve | **Disaster recovery (oporavak media baze)** |
| **Disaster Recovery ili Cyber Recovery** | Za oporavak orijentisan ka NetWorker serveru | **Da** | • Oporavak media baze<br>• Recover | • Recover space<br>• Workflow<br>• Backup<br>• Clone<br>• Index management<br>• Media management<br>• Save set operations |

> Obratite pažnju: u `Active` stanju oporavak media baze je **zabranjen**. Zato korak nije opcion.

### Postavljanje iz konzole

```bash
# Sa pokrenutim NetWorker serverom
nsradmin -s <networker_server>
```

```
nsradmin> . type: NSR
nsradmin> p
nsradmin> update server state: disaster recovery
update? y
nsradmin> quit
```

### Postavljanje preko promenljive okruženja **[ADM]**

Ako želite da server startuje u određenom stanju:

> **[ADM]** Stanje u kojem server startuje može se zadati promenljivom okruženja **`NSR_SERVER_STATE`**. Na UNIX-u se to najbolje dodaje u **`/nsr/nsrrc`**.

```bash
# /nsr/nsrrc
NSR_SERVER_STATE="disaster recovery"
export NSR_SERVER_STATE
```

> **Ne zaboraviti da se vrati na `active`** posle završenog oporavka — u DR stanju backup, clone i workflow operacije ne rade.

## 53.8 Retention policy klijentskog resursa

> **[DR]** Podrazumevana retencija klijentskog resursa NetWorker servera je **mesec dana**. Postavljanjem na **`Decade`** omogućava se oporavak **svih** zapisa u fajlovima baze. Ako se retencija ne promeni, save set-ovi NetWorker servera sa retencijom dužom od mesec dana se **odbacuju**, jer je podrazumevana browse politika mesec dana.

Ovo se izvršava **posle** oporavka resource baze, kada klijentski resurs postoji. Iz konzole:

```bash
nsradmin -s <networker_server>
```

```
nsradmin> . type: NSR client; name: <networker_server_fqdn>
nsradmin> p
nsradmin> update retention policy: Decade
update? y
```

---

# 54. Podmetanje repliciranog storage-a

## 54.1 Šta je stvarno replicirano

MTree replikacija kopira **ceo MTree** sa produkcije u vault. Ono što stiže:

| Stiže | Ne stiže |
|---|---|
| Backup podaci (save set-ovi) | NetWorker resource baza |
| Bootstrap save set-ovi | NetWorker media baza |
| Index save set-ovi | Client file indeksi u aktivnom obliku |
| Struktura volumena na medijumu | Device resursi |

> **Ključno razumevanje:** podaci postoje, ali NetWorker o njima **ne zna ništa**. Cilj naredna dva poglavlja je da NetWorker-u napravite uređaj koji pokazuje na te podatke, a zatim da iz njih rekonstruišete baze.

## 54.2 Replicirani MTree je read-only

> **[ADM]** Nakon kreiranja replikacionog para sa izvornog DD sistema, na odredišnom DD sistemu se kreira replicirani MTree. Taj MTree je **vidljiv na odredišnom DD sistemu, ali ne i u njegovoj storage unit listi**.

> **[ADM]** Pri kreiranju replikacionog para, **ne koristiti ime ciljnog servera kao ime ciljnog MTree-a**. Koristiti drugačije ime, jer su ti MTree-ovi **read-only i ne mogu se koristiti na odredištu za kreiranje DD uređaja.**

Iz ovoga sledi praktično pravilo za vault:

> **Uređaj se ne pravi direktno nad repliciranim MTree-om.** Radi se nad **read/write kopijom**:
>
> - **Cyber Recovery sandbox** — CR kreira sandbox nad zaključanom PIT kopijom i eksportuje ga; ovo je standardan put
> - **DD fastcopy** — ručno napravljena kopija u drugi MTree

## 54.3 Kako videti storage unit-e na DD sistemu

```bash
# Prijava na vault DD sistem
ssh sysadmin@<vault_dd>

# Spisak svih storage unit-a i njihovih DD Boost korisnika
sysadmin@dd-vault# ddboost storage-unit show

# Spisak MTree-ova (uključujući replicirane, koji nisu storage unit-i)
sysadmin@dd-vault# mtree list

# Status replikacionih konteksta
sysadmin@dd-vault# replication show config
sysadmin@dd-vault# replication status

# Spisak korisnika i UID-ova
sysadmin@dd-vault# user show list

# DD Boost korisnici
sysadmin@dd-vault# ddboost user show
```

## 54.4 Kako replicirani MTree učiniti vidljivim kao storage unit

> **[ADM]** Da bi se replicirana storage unit učinila vidljivom na odredišnom DD sistemu, ažurirati DD Boost korisnika za repliciranu storage unit:

```bash
sysadmin@dd-vault# ddboost storage-unit modify <target-mtree-name> user <ddboost-user>
sysadmin@dd-vault# ddboost storage-unit show
```

> **Podsetnik iz istog izvora:** DD Boost korisnik na izvornom i ciljnom DD sistemu mora imati **isti UID**. Validacija: `user show list`.

## 54.5 Kreiranje read/write kopije preko fastcopy

Ako se ne koristi CR sandbox, kopija se pravi na DD nivou. **[CR]**

```bash
sysadmin@dd-vault# filesys fastcopy source <folder_izvornog_MTree-a> destination <folder_odredišnog_MTree-a>
```

Gde je:
- `<folder izvornog MTree-a>` — folder repliciranog MTree-a
- `<folder odredišnog MTree-a>` — folder koji backup aplikacija preporučuje ili gde želite podatke

Zatim kreirati storage unit nad odredišnim MTree-om i dodeliti mu DD Boost korisnika.

## 54.6 Kako videti šta NetWorker vidi

### Konfigurisani uređaji i montirani volumeni

> **[CMD]** `nsrmm -C` prikazuje listu NetWorker konfigurisanih uređaja i volumena koji su u njima trenutno montirani. Prikazuje **samo uređaje i volumene dodeljene serveru**, ne nužno sve stvarno priključene. `-C` je podrazumevana opcija.

```bash
nsrmm -C
```

Očekivani oblik izlaza **[DR]**:

```
32916:nsrmm: file disk <volume_name> mounted on <device_name>, write enabled
```

> `<device_name>` je uređaj koji odgovara AFTD ili DD `<volume_name>`.

### Pregled device resursa iz konzole

```bash
nsradmin -s <networker_server>
```

```
nsradmin> . type: NSR device
nsradmin> p
```

### Rad kada NetWorker server nije pokrenut

> **[CMD]** Opcija `-d resdir` koristi NetWorker resource bazu `resdir` umesto otvaranja mrežne konekcije. Bazu treba koristiti **štedljivo, i samo kada NetWorker server nije pokrenut.**

```bash
cd /nsr/res
nsradmin -d nsrdb
```

```
nsradmin> . type: NSR device
nsradmin> p
```

> **[ADM]** Podrazumevana putanja na Linux-u je `/nsr/res`.

## 54.7 Zašto se DD Boost storage unit ne vidi u `mount`

Ovo je najčešći izvor zabune kod administratora koji dolaze iz sveta fajl sistema.

| Tip uređaja | Vidi se u `mount` / `df` | Objašnjenje |
|---|---|---|
| **AFTD preko NFS-a** | **Da** | Standardan NFS mount na OS nivou |
| **AFTD lokalni** | **Da** | Lokalni fajl sistem |
| **DD Boost uređaj** | **Ne** | DD Boost je **aplikativni protokol**. NetWorker se povezuje na DD preko DD Boost biblioteke, ne montira fajl sistem. |

**Praktična posledica:** za DD Boost uređaj `df -h` i `mount` neće pokazati ništa. Provera se radi isključivo kroz `nsrmm -C`, `nsradmin` i DD stranu (`ddboost storage-unit show`).

**Provera za AFTD/NFS:**

```bash
mount | grep -i nfs
df -h
showmount -e <dd_hostname>
```

## 54.8 Kreiranje device resursa nad podacima

> ### Pravila koja se ne smeju prekršiti **[DR]**
>
> 1. **Ne označavati (label) volumen ponovo.** Ponovno označavanje volumena sa bootstrap backup-ima, ili bilo kojim drugim backup-ima, **čini podatke neoporavljivim.**
> 2. Za disk uređaje (AFTD): **ne dozvoliti čarobnjaku da označi disk volumen.** Opcija **Label and Mount** je podrazumevano izabrana u prozoru **Device Label and Mount** — **isključiti je.**
> 3. U prozoru **Select Storage Node** navesti **lokalnu putanju** do AFTD, DD ili trakastog volumena. Mora biti **ista putanja na kojoj se nalaze bootstrap podaci.**

> **[DR]** NetWorker server zahteva **lokalni device resurs** za oporavak podataka iz bootstrap backup-a. U DR situaciji resource baza je izgubljena, pa se lokalni uređaj mora ponovo kreirati.

> **[DR]** Ako je AFTD kreiran na samom serveru, podaci mogu biti izgubljeni pri padu servera. Preporučuje se **DD ili trakasti uređaj**, ili AFTD koji nije na lokalnom disku (npr. AFTD na NFS share-u).

### Format atributa **[ADM]**

| Tip uređaja | `Name` | `Device access information` |
|---|---|---|
| **AFTD lokalni** | `aftd-1` | puna putanja do direktorijuma uređaja |
| **AFTD udaljeni** | `rd=<snode_hostname>:<device_name>` | `<NFS_host>:/<path>` |
| **Data Domain** | `rd=<snode>:<device_name>` ili `<ddhost>_<dev>` | `<dd_host>:/<storage_unit>/<device_name>` |

> **[ADM]** Za non-root ili cross-platform Client Direct pristup AFTD-u, **ne navoditi automounter putanju ni montiranu putanju**. Umesto toga navesti putanju u formatu `host:/path`, čak i ako je AFTD lokalan za storage node.

### Kreiranje uređaja kroz `nsradmin`

```bash
nsradmin -s <networker_server>
```

```
nsradmin> create type: NSR device;
          name: <ime_uredjaja>;
          device access information: <putanja_ili_dd_su>;
          media type: <tip>;
          ...
create? y
```

> **Napomena o sintaksi:** `nsradmin` očekuje atribute razdvojene znakom `;`, a unos se završava praznim redom. Za tačan spisak obaveznih atributa za konkretan tip uređaja konsultovati `nsr_device(5)` man stranicu na sistemu:
>
> ```bash
> man nsr_device
> ```

### Skriptovano kreiranje iz fajla

> **[CMD]** `-i file` uzima ulazne komande iz fajla umesto sa standardnog ulaza. U tom režimu se interaktivni prompt ne prikazuje.

```bash
cat > /tmp/dev.txt <<'EOF'
option regexp
create type: NSR device;
       name: dd-recover-01;
       device access information: dd-vault.example.com:/su_recover/dd-recover-01;
EOF

nsradmin -i /tmp/dev.txt
```

> **[ADM]** Primer istog obrasca za ažuriranje lozinki nad više uređaja:
> ```
> option regexp
> . type: NSR device; device access information: <dd_ip*>
> update password : <new password>
> ```

## 54.9 Omogućavanje CDI atributa

Potrebno pre `nsrdr`-a kod trakastih uređaja. **[DR]**

> **[ADM]** CDI (Common Device Interface) omogućava NetWorker serveru da šalje komande trakastim uređajima. **Nije podržan u NDMP okruženju.**

Kroz UI: **Devices → View → Diagnostic Mode → Devices → dvoklik na uređaj → Advanced tab → Device Configuration → CDI**

| Vrednost | Ponašanje |
|---|---|
| **Not Used** | Onemogućava CDI, koristi standardne pozive tape drajvera |
| **SCSI Commands** | Šalje eksplicitne SCSI komande uređajima |

Kada je omogućen, CDI daje jasnije poruke o statusu trake, obaveštava kada je traka write-protected i omogućava Tape Alert.

**Globalno onemogućavanje bez diranja svakog uređaja [ADM]:**

```bash
mkdir -p /nsr/debug
touch /nsr/debug/cdidisable
# zatim restartovati NetWorker server
```

> Prisustvo ovog fajla onemogućava CDI za taj server i sve storage node-ove kojima on upravlja.

> **[ADM]** CDI se postavlja ili onemogućava **samo po savetu Dell Support-a.** Upotreba CDI-ja ne menja ono što se piše na traku.

Posle izmene CDI atributa: **zaustaviti pa pokrenuti NetWorker servise.** **[DR]**

---

# 55. Pronalaženje bootstrap-a bez media baze

## 55.1 Šta tražimo

`nsrdr` traži **bootstrap save set ID (ssid)**, a kod trake i **početni file i record broj**.

## 55.2 Metoda 1 — iz log fajla notifikacija

Ako je log fajl sačuvan, ovo je najbrži put. **[DR]**

```bash
grep -A 40 "Server backup Action report" /nsr/logs/policy_notifications.log | tail -60
```

Sekcija **Bootstrap backup report** ima kolone:

```
date  time  level  ssid  file  record  volume
```

Poslednji red daje najnoviji bootstrap.

> Isti fajl može biti prosleđen na drugu lokaciju, u zavisnosti od konfiguracije notifikacija politike.

## 55.3 Metoda 2 — `mminfo -B`

Radi **samo ako media baza postoji** — dakle u scenariju gde je izgubljena samo resource baza, ne i media. **[DR]**

```bash
mminfo -B
mminfo -av -B -s <server_name>
```

Primer izlaza **[CMD]**:

```
Jun 17 22:21 2012 mars's NetWorker bootstrap information Page 1
date      time     level  ssid      file record volume
6/14/12   23:46:13 full   17826163  48   0      mars.1
6/15/12   22:45:15 9      17836325  87   0      mars.2
6/16/12   22:50:34 9      17846505  134  0      mars.2 mars.3
6/17/12   22:20:25 9      17851237  52   0      mars.3
```

Čitanje izlaza **[CMD]**:

| Podatak | Vrednost u primeru | Gde je |
|---|---|---|
| ssid najnovijeg bootstrap-a | `17851237` | **četvrta kolona** |
| početni file broj | `52` | peta kolona |
| početna record lokacija | `0` | šesta kolona |
| volumen | `mars.3` | poslednja kolona |

> **[CMD]** Ako bootstrap save set obuhvata više volumena, izlaz prikazuje **više imena volumena** za taj save set. **Redosled prikaza je redosled koji `nsrdr` zahteva.** U primeru, treći save set od 6/16/12 počinje na volumenu `mars.2` i nastavlja se na `mars.3`.

## 55.4 Metoda 3 — `scanner -B`

Ovo je put kada media baza **ne postoji** — tipičan vault scenario.

> **[CMD]** `scanner` čita NetWorker medijum da bi potvrdio sadržaj volumena, izdvojio save set ili ponovo izgradio NetWorker online indekse. **Komandu može pokretati samo super-user.** Uređaj se uvek mora navesti.

```bash
# Skeniranje uređaja radi pronalaženja bootstrap-a
scanner -B <device_name>
```

> **[DR]** Primer za CloudBoost uređaj: `scanner -B rd=bu-idd-cloudboost.iddlab.local:base/bkup`

**Šta radi `-B`** **[CMD]**: kada se koristi zajedno sa `-S` opcijom, navedeni ssid se **označava kao bootstrap**.

**Bez opcija ili sa `-v`** **[CMD]**: volumen na uređaju se otvara za čitanje, skenira, i generiše se **tabela sadržaja** sa informacijama o svakom pronađenom save set-u — ime klijenta, ime save set-a, save time, nivo, veličina, broj fajlova, ssid i zastavica.

**Zastavice u izlazu** **[CMD]**:

| Zastavica | Značenje |
|---|---|
| `B` | **Begin** — početak save set-a |
| `C` | **Continue** — save set je počeo na drugom volumenu |
| `S` | **Synchronize** — mesto sa kojeg se izdvajanje može nastaviti u slučaju oštećenja medija |
| `E` | **End** — kraj save set-a; izaziva ispis reda tabele sadržaja |

> Ostale zastavice se prikazuju samo uz `-v` opciju.

> **[CMD]** Za trakaste uređaje mora se koristiti ime **„no-rewind on close"** uređaja.

> **[CMD]** Za `adv_file` tip uređaja: ako server **nije pokrenut**, umesto imena uređaja mora se navesti **putanja do volumena**.

## 55.5 Ako je bootstrap na udaljenom uređaju

> **[DR]** NetWorker **ne podržava oporavak bootstrap-a sa udaljenog uređaja.** Da biste oporavili bootstrap sa kloniranog save set-a na udaljenom uređaju, morate klonirati save set sa udaljenog uređaja na uređaj **lokalan za NetWorker server**.

Postupak **[DR]**:

```bash
# 1. Ponovo kreirati uređaj koji sadrži klonirani bootstrap save set
#    U Device Configuration Wizard-u, na stranici Pools Configuration
#    ISKLJUČITI "Configure Media Pools for devices"
#    (sprečava ponovno označavanje kloniranog uređaja i gubitak podataka)

# 2. Kreirati novi LOKALNI uređaj na NetWorker serveru
#    Preporuka: AFTD, na koji se oporavlja bootstrap

# 3a. Utvrditi SSID save set-a
scanner -B <device_name>

# 3b. Popuniti media bazu podacima o kloniranom save set-u
scanner -m -S <SSID/CloneID> <device_name>

# 4. Klonirati bootstrap na lokalni uređaj
nsrclone ...

# 5. Utvrditi SSID/CloneID na lokalnom uređaju
mminfo -B

# 6. Oporaviti bootstrap sa lokalnog uređaja
nsrdr
```

## 55.6 Kada `scanner` može promeniti stanje volumena

> **[CMD]** Pri skeniranju `adv_file` ili Data Domain volumena, `scanner` može **resetovati scan needed zastavicu** u zapisu volumena. To se dešava **samo** kada je navedena opcija `-i` ili `-m`, **nijedna** od opcija `-c`, `-n`, `-N`, `-S` nije navedena, i `scanner` je uspešno skenirao **sve** save set-ove na volumenu.

> **[CMD]** Kada save set obuhvata više volumena, skenirati volumene **redosledom kojim su pisani**. Svi delovi save set-a moraju biti skenirani da bi oporavak bio moguć.

---

# 56. `nsrdr` — puna procedura

## 56.1 Sintaksa

> **[CMD]**
> ```
> nsrdr [ -q | -v ] [ -a ] [ -K ] [ -A ] [ -B bootstrap_id ] [ -d device_name ]
>       [ -t date ] [ -N | -n ] [ -c ] [ -l alternate_path ]
>       [ -I { client_name... | -f client_list_input_file } ]
> ```

## 56.2 Šta `nsrdr` radi

> **[CMD]** `nsrdr` je CLI alat koji automatizuje većinu ručnih koraka u tradicionalnom procesu oporavka NetWorker servera. Radi u interaktivnom i neinteraktivnom režimu.

> **[CMD] NOTE:** ova komanda **prepisuje postojeće metapodatke** na NetWorker serveru.

> **[DR]** `mmrecov` je zastarela od NetWorker 9.0 i zamenjena komandom `nsrdr`. Za rollback na raniju verziju kontaktirati Dell Support.

Client file indeksi se mogu oporaviti **odvojeno** od media baze, resource fajlova i Authentication Service baze. **Podrazumevano `nsrdr` oporavlja sve CFI-jeve.** **[CMD]**

## 56.3 Referenca opcija **[CMD]**

| Opcija | Opis |
|---|---|
| `-a` | Neinteraktivni režim. **Zahteva `-B` i `-d`.** |
| `-A` | Zadržava originalni Authentication Service database fajl i **ne** zamenjuje ga oporavljenim. Bez upita. |
| `-B <bootstrap_id>` | Bootstrap SSID. Bez upita za ssid. |
| `-c` | **Samo** oporavak indeksa. U interaktivnom režimu koristiti posle bootstrap oporavka. |
| `-d <device_name>` | Uređaj koji sadrži volumen sa bootstrap backup-om. Bez upita za uređaj. |
| `-F` | Postavlja scan needed **samo** na file i advanced file type uređajima. **Ovo je podrazumevano ponašanje**; opcija postoji radi kompatibilnosti sa ranijim verzijama. |
| `-I <client_name>...` | CFI oporavak za navedene klijente, posle bootstrap oporavka. Imena razdvojena razmakom. **Mora biti poslednja opcija.** |
| `-I -f <fajl>` | Lista klijenata iz fajla, jedno ime po liniji. |
| `-K` | Zadržava originalni `/nsr/res` folder i **ne** zamenjuje resource fajlove oporavljenima. Bez upita. |
| `-l <alternate_path>` | Alternativna putanja za preimenovanje NetWorker resursa, ako preimenovanje u `/nsr` ne uspe. Mora biti folder; ako ne postoji, kreira se. |
| `-N` | Postavlja scan needed na **sve** volumene, **uključujući trakaste**. |
| `-n` | **Sprečava** postavljanje podrazumevane scan needed zastavice na file i advanced file type uređajima. **UPOZORENJE: koristiti oprezno.** |
| `-q` | Quiet — samo poruke o greškama |
| `-t <date>` | Oporavlja CFI-jeve **na određeni trenutak**. Format prema `nsr_getdate(3)`. Za relativno vreme u prošlosti koristiti reč **`ago`**. Koristi se uz `-I` u neinteraktivnom režimu. |
| `-v` | Verbose — generiše debug informacije |

> ### Neusklađenost izvora koju treba znati
>
> *DR Best Practices Guide 19.13* opisuje `-N` kao opciju koja štiti od gubitka save set-ova, a `-F` kao opciju koja postavlja scan needed samo na File/AFTD/Cloud uređajima i **zahteva `-N`**.
>
> *Command Reference Guide 19.11* je precizniji: **podrazumevano ponašanje** je da se scan needed postavlja na file i advanced file type uređajima. `-F` je samo kompatibilnost unazad sa tim podrazumevanim ponašanjem. `-N` **dodaje** trakaste volumene. `-n` isključuje podrazumevano ponašanje.
>
> **Praktično:** ne morate navoditi `-F` — to je već podrazumevano. `-N` navodite kada u okruženju ima **trake**.

**Primer sa `-t` [CMD]:**

```bash
# Oporavak client file indeksa kakvi su bili pre mesec dana
nsrdr -t "1 Month ago"
```

> **[CMD]** Oporavak indeksa na određeni trenutak **dodaje ceo sadržaj indeksa iz tog trenutka na tekući sadržaj indeksa**. Time se omogućava pregledanje save set-ova kojima je istekla browse politika a i dalje su oporavljivi. Ti save set-ovi se označavaju kao browsable i ostaju takvi onoliko dugo koliko su originalno bili.

## 56.4 Priprema neposredno pre pokretanja

```bash
# 1. Server state mora biti "disaster recovery" (poglavlje 53.7)

# 2. Odmontirati SVE volumene — traka, file type, AFTD, cloud
#    Iz NMC-a ili:
nsrmm -C                    # pregled šta je montirano
nsrmm -u -f <device_name>   # unmount po uređaju

# 3. Omogućiti CDI ako ima trake (poglavlje 54.9), pa restartovati servise

# 4. Prijaviti se kao root
whoami
```

> **[DR]** Za autochanger okruženje:
> ```bash
> nsrjb -vHE      # reset autochanger-a, izbacivanje volumena, reinicijalizacija element statusa
> ielem           # ako uređaj ne podržava -E opciju
> nsrjb -I        # inventar — utvrđuje da li su potrebni volumeni u autochanger-u
> ```
> **Nijedan od tih volumena nije u media bazi.** Sadržaj trake se ne može videti kroz NMC i ime volumena se prikazuje kao `-*`.

## 56.5 Podešavanje `nsrdr.conf` — pre pokretanja

Za veliki broj klijenata podrazumevanih 5 niti je usko grlo.

```bash
mkdir -p /nsr/debug
cat > /nsr/debug/nsrdr.conf <<'EOF'
NSRDR_NUM_THREADS = 10
EOF
```

| Parametar | Opis **[CMD]** |
|---|---|
| `NSRDR_NUM_THREADS` | Broj niti koje proces oporavka može pokrenuti za paralelne CFI oporavke |
| `NSRDR_SERVICES_PATH` | Na UNIX platformama `nsrdr` koristi podrazumevanu putanju za start servisa pri internom restartu. Ako putanja nije podrazumevana, zadati je ovim parametrom. |

> **[DR]** Podrazumevana vrednost je **5**. Vrednost mora biti veća od 1; nula ili negativna vrednost vraća podrazumevanih 5. Obavezan **razmak pre i posle** znaka `=`. Svaki parametar u zasebnoj liniji.
>
> Neki editori dodaju `.txt` na kraj imena — ukloniti ekstenziju.

## 56.6 Interaktivni tok — šta `nsrdr` pita

> **[CMD]** U interaktivnom režimu `nsrdr` može tražiti sledeće informacije:

### Upit 1 — uređaj

Ime uređaja koji sadrži bootstrap save set.

### Upit 2 — bootstrap ssid

> **[CMD]** Broj se nalazi u **četvrtoj koloni** (`ssid`) izlaza komande `mminfo -B`.

Ako ssid ne znate, ostavite prazno i pritisnite Enter — `nsrdr` nudi skeniranje uređaja. **[DR]**

> **[DR]** Opcija skeniranja bootstrap save set ID-a **nije podržana za ne-engleske locale**. U tom slučaju koristiti `scanner` komandu.

Provera locale na RHEL 9.6:

```bash
locale
echo $LANG
```

### Upit 3 — file i record lokacija (samo traka)

> **[CMD]** Početne file i record lokacije nalaze se u **petoj i šestoj koloni** `mminfo -B` izlaza. Kada ih ne znate, koristite podrazumevanu vrednost **0**.
>
> Navođenje tačnih brojeva omogućava NetWorker-u da **brže locira** bootstrap save set na traci.

### Upit 4 — montiranje volumena

Alat traži da montirate volumen koji sadrži izabrani bootstrap ssid u navedeni uređaj.

> **[CMD]** Ako bootstrap save set obuhvata više volumena, `nsrdr` će tražiti ime uređaja koji sadrži **sledeći volumen** kada operacija dođe do kraja tekućeg.

### Upit 5 — resource fajlovi

> **[CMD]** Proces pita da li da zadrži originalne resource fajlove u `/nsr/res` folderu ili da ih zameni oporavljenima. **Ako je resource konfiguracija izgubljena, izaberite `yes`.** `nsrdr` će automatski vratiti fajlove na originalnu lokaciju.

Šta se dešava u pozadini **[DR]**:

1. Oporavljena resource baza se čuva u privremeni folder **`res.R`**
2. NetWorker servisi se gase, jer `nsrdr` ne može prepisati resource bazu dok rade
3. Postojeći resource folder se zamenjuje oporavljenim; zamenjeni folder se preimenuje u **`res.<timestamp>`**

> **[CMD]** `/nsr/res` — direktorijum i njegov sadržaj čuvaju se kao deo bootstrap-a. Pri oporavku se originalni direktorijum **privremeno preimenuje** u `/nsr/res.timestamp`.

**Ako preimenovanje ne uspe [DR]:** `nsrdr` traži alternativnu putanju. Uneti `y` i navesti putanju. Ako alternativni folder ne postoji, kreira se. **Ako se navede ime fajla, upit se ponavlja.**

### Upit 6 — Authentication Service baza

> **[DR]** Na upit *„Do you want to replace the existing NetWorker Authentication Service database file, `authcdb.h2.db`, with the recovered database file?"* uneti `y`.

Zamenjeni fajl se preimenuje u `authcdb.h2.db.<timestamp>`. NetWorker servisi se restartuju posle zamene.

### Upit 7 — client file indeksi

> **[CMD]** `nsrdr` traži oporavak CFI-jeva **posle** što je operacija restartovala NetWorker servise.

| Izbor | Posledica |
|---|---|
| `y`, pa `y` za potvrdu | Oporavlja CFI za **svakog** klijenta koji je bio backup-ovan, uključujući CFI NetWorker servera |
| `n` | Operacija se završava; CFI za izabrane klijente se oporavlja naknadno sa `nsrdr -c -I` |

Po završetku CFI oporavka pojavljuje se poruka **[CMD]**:

```
completed recovery of index for client '<client_name>'
```

> **[CMD]** Indekse možete oporavljati **bilo kojim redosledom**; ne morate oporaviti indeks NetWorker servera pre indeksa klijenta.

## 56.7 Šta se dešava sa media bazom

> **[CMD]** Kada oporavljate NetWorker media bazu, `nsrdr` postavlja **scan needed** zastavicu na volumene na file type i advanced file type uređajima koji postoje u oporavljenoj media bazi.

**Zašto — objašnjenje koje treba razumeti [CMD]:**

Posle vraćanja bootstrap save set-a, operacija oporavka **zamenjuje sve podatke u media bazi**. Ako je bilo koji backup ili clone proces pisao podatke na volumene **posle** vremena kreiranja bootstrap-a, oporavljena media baza **neće sadržati informacije o tim save set-ovima**.

| Tip medija | Posledica bez zaštite |
|---|---|
| **Traka** | NetWorker ima **netačnu informaciju o poziciji trake**. Koristiće je za pisanje novih podataka i **prepisaće postojeće.** |
| **File / AFTD** | *Recover space* operacija, koju NetWorker server pokreće automatski, **obrisaće sve save set-ove** kreirane posle vremena bootstrap-a. |

Scan needed zastavica govori `nsrmmd`-u da pronađe **stvarni kraj volumena** i sprečava gubitak podataka.

> **[CMD]** Za file ili advanced file type uređaje, NetWorker server **suspenduje recover space operaciju** dok ne skenirate uređaj i zatim uklonite scan needed zastavicu.

## 56.8 Neinteraktivni režim

```bash
# Bootstrap + svi CFI-jevi
nsrdr -a -B <bootstrap_ID> -d <device> -I

# Samo izabrani klijenti
nsrdr -a -B <bootstrap_ID> -d <device> -I client1 client2 client3

# Lista iz fajla
nsrdr -f /tmp/klijenti.txt -I
```

> **[DR]** Uz `-a` opciju **morate navesti važeći bootstrap ID sa `-B`**. U suprotnom čarobnjak izlazi kao da je otkazan, **bez opisne poruke o grešci**.

> **[CMD]** Kada je navedena `-I` opcija, mora biti **poslednja u komandi**, jer se sve posle nje tumači kao imena klijenata.

## 56.9 Posle `nsrdr`-a — obavezno

```bash
# Ponovo pokrenuti konfiguracionu skriptu Authentication Service-a
/opt/nsr/authc-server/scripts/authc_configure.sh
```

> **[DR]** Ovaj korak je naveden i pre i posle oporavka.

---

# 57. Posle `nsrdr`

## 57.1 Provera resursa

> **[DR]** Otvoriti Administration prozor u NMC-u i proveriti da se svi resursi NetWorker servera prikazuju:
>
> - **Protection** — svi resursi kao pre oporavka
> - **Devices** — svi resursi kao pre oporavka
> - **Media** — svi resursi kao pre oporavka
> - **Media → Tape Volumes / Disk Volumes** — svi volumeni imaju **isti režim kao pre oporavka**; svi uređaji na koje se piše su u **appendable** režimu

Iz konzole, bez NMC-a:

```bash
# Resursi po tipu
nsradmin -s <server> <<'EOF'
. type: NSR client
p
EOF

nsradmin -s <server> <<'EOF'
. type: NSR device
p
EOF

nsradmin -s <server> <<'EOF'
. type: NSR pool
p
EOF

# Volumeni i njihov režim
mminfo -m
mminfo -a -r 'state,volume,written,%used,volretent,read,space'
```

> **[CMD]** Primeri `mminfo` upotrebe:
> ```
> mminfo -m
> mminfo -a -r 'state,volume,written,%used,volretent,read,space'
> mminfo -a -r 'volume,%used,pool,location' -q '!full'
> mminfo -av -r 'volume, name, savetime, ssflags, clflags, ssid(53)'
> ```

## 57.2 Uklanjanje scan needed zastavice

Odluka: da li sumnjate da su save set-ovi pisani **posle** poslednjeg bootstrap-a?

> **[DR]** Ručna save operacija je **jedini** način da save set bude backup-ovan bez pokretanja backup-a CFI podataka. Ako je ručni backup izvršen pre sledećeg zakazanog backup-a (koji uvek backup-uje bootstrap i CFI), poslednji CFI **neće imati zapis** o tim save set-ovima.

### Ako sumnjate — skenirati

```bash
# 1. Utvrditi ime uređaja koje odgovara volumenu
nsrmm -C
# Izlaz: 32916:nsrmm: file disk <volume_name> mounted on <device_name>, write enabled

# 2. Popuniti CFI i media bazu
scanner -i <device_name>
#    <device_name> je ime AFTD ili DD UREĐAJA, ne ime volumena
```

> **[DR]** `scanner -i` može trajati **veoma dugo**, posebno na velikom disk volumenu.

### Preporučeni put kada postoje index backup-i

> **[CMD]** Za verziju 6.0 i noviju: ako imate medijum koji sadrži **index backup-e** uz data backup-e, **preporučeni način vraćanja indeksa** je:
>
> 1. Pokrenuti `scanner -m` da se ponovo učitaju zapisi media baze za index i data backup-e
> 2. Zatim pokrenuti `nsrck -L7 -t <date> <clientname>` da se oporavi indeks klijenta u trenutku tih backup-a
>
> To vraća zapise indeksa za taj trenutak nazad u indeks.
>
> **`scanner -i` koristiti samo ako index backup-i ne postoje.**

Ovo je bitna nijansa koju ni CR ni DR vodič ne navode eksplicitno — `scanner -i` je sporiji i grublji alat.

```bash
# Preporučeni put
scanner -m <device_name>
nsrck -L7 -t "<datum>" <ime_klijenta>

# Rezervni put, kada nema index backup-a
scanner -i <device_name>
```

> **[CMD]** Za NDMP, DSA i Block based backup save set-ove, `-i` opcija **ne rekonstruiše** zapise indeksa iz volumena. Za volumene koji imaju kombinaciju DSA/Block based i regularnih save set-ova, `scanner -i` preskače DSA i Block based save set-ove uz grešku.

### Uklanjanje zastavice

**Za AFTD i DD volumene [DR]** — kroz NMC:

1. **Administration → Devices → Devices** → desni klik na uređaj → **Unmount**. Zabeležiti pridruženi volumen.
2. **Administration → Media → Disk Volumes** → desni klik na volumen → **Mark Scan Needed** → izabrati **Scan is NOT needed** → **OK**
3. **Devices → Devices** → desni klik → **Mount**

**Za trakaste volumene [DR]:**

```bash
# Zabeležiti file i record broj iz poruke koju NetWorker prikaže
scanner -f <file> -r <record> -i <device>

# Ukloniti zastavicu
nsrmm -o notscan <volume_name>
```

Poruka koju NetWorker prikaže pri pokušaju montiranja **[DR]**:

```
nw_server nsrd media info: Volume <volume_name> has save sets unknown to media database.
Last known file number in media database is ### and last known record number is ###.
Volume <volume_name> must be scanned; consider scanning from last known file and record numbers.
```

> Za AFTD uređaje, dok je zastavica postavljena, **možete i dalje pisati na disk**, ali su *recover space* operacije **suspendovane**. **[DR]**

## 57.3 `nsrck` — nivoi provere

> **[CMD]** `nsrck` proverava konzistentnost NetWorker online indeksa. Normalno ga automatski i sinhrono pokreće `nsrindexd` pri startu.

**Sintaksa [CMD]:**

```
nsrck [ -qMv ] [ -R [ -Y ] ] [ -L check-level [ -t date ] | -X [ -x percent ] | -C | -F | -m | -n | -c ]
      [ -T tempdir ] [ clientname ... ]
```

**Nivoi provere [CMD]:**

| Nivo | Šta radi |
|---|---|
| **1** | Validira header online file indeksa, spaja žurnal izmena sa postojećim header-om. Save set record fajlovi i odgovarajući key fajlovi se premeštaju u odgovarajuće poddirektorijume pod `db6`. |
| **2** | Nivo 1 + provera indeksa za nove i otkazane save operacije. Novi se dodaju, otkazani uklanjaju. |
| **3** | Nivo 2 + usklađivanje online file indeksa sa online media indeksom. Zapisi bez odgovarajućih media save set-ova se odbacuju. Brišu se prazni poddirektorijumi pod `db6`. |
| **4** | Nivo 3 + provera validnosti internih key fajlova indeksa. Nevalidni se ponovo grade. |
| **5** | Nivo 4 + verifikacija digest-a pojedinačnih save time-ova u odnosu na njihove key fajlove. |
| **6** | Nivo 5 + izdvajanje svakog zapisa iz svakog save time-a i provera da se može izdvojiti. Digest se ponovo računa i poredi sa sačuvanim; interni key fajlovi se ponovo grade. |
| **7** | **Ne radi proveru nivoa 6.** Spaja u online file indeks **podatke indeksa oporavljene sa backup medija**, ponovo gradi interne key fajlove i header indeksa. |

> ### Zamka kod nivoa 7 **[CMD]**
>
> **Nivo 7 neće prepisati postojeće fajlove u client file indeksu.** Ako podaci online client file indeksa već postoje za save set u određenom save time-u, **moraju se ukloniti pre nego što nivo 7 može da ih vrati sa backup medija.**
>
> Primer iz dokumentacije: ako je `.rec` fajl u indeksu oštećen, a `nsrck -L5` nije izvršen da prvo očisti oštećeni save set, onda `nsrck -L7` **neće prepisati** oštećeni `.rec` fajl i indeks ostaje oštećen.

```bash
# Oporavak indeksa iz backup-a
nsrck -L7 <ime_klijenta>

# Na određeni trenutak (samo uz -L7)
nsrck -L7 -t "<datum>" <ime_klijenta>

# Ručna dublja provera pri sumnji na oštećenje
nsrck -L 6 <ime_klijenta>
```

> **[CMD]** Provere na višem nivou generalno traju duže. `nsrck` je **restartabilan u bilo kom trenutku** izvršavanja, pa može preživeti pad sistema ili iscrpljenje resursa bez gubitka podataka.

> **[CMD]** Ako pri konverziji indeksa nema dovoljno slobodnog prostora na volumenu, koristiti `-T <tempdir>` da se zada drugi direktorijum kao radni prostor.

> **[CMD]** Svaki put kada NetWorker server startuje, pokreće `nsrck -L 1` kao brzu proveru za svaki konfigurisani client file index. `nsrim` automatski poziva `nsrck -L 3` posle ažuriranja browse i retention vremena.

## 57.4 Vraćanje server state atributa

Kada je oporavak media baze završen:

```bash
nsradmin -s <server>
```

```
nsradmin> . type: NSR
nsradmin> update server state: active
update? y
```

> U `disaster recovery` stanju **backup, clone, workflow, index management, media management i save set operacije ne rade**. **[ADM]** Ako ovo zaboravite, kupac će prijaviti da „backup ne radi".

Ako ste koristili `/nsr/nsrrc`, ukloniti ili izmeniti unos:

```bash
vi /nsr/nsrrc
# ukloniti NSR_SERVER_STATE ili postaviti na "active"
```

## 57.5 Šta neće raditi — očekivano ponašanje

| Ponašanje | Objašnjenje | Izvor |
|---|---|---|
| **`jobsdb` je prazan** | Server Protection politika **ne backup-uje** `jobsdb`. Posle DR-a svi statusi workflow-ova i akcija su izgubljeni, status prelazi u **`Never Run`**. | **[DR]** |
| **NMC ne prikazuje sve resurse** | NMC ima **zasebnu bazu**, izolovanu od serverdb. `nsrdr` je ne dira. | **[DR]** |
| **Lažne greške u `nsrdr.log`** | Ako je oporavljeni server štitio virtuelne cluster klijente ili NMM zaštićen virtuelni DAG Exchange server, log sadrži lažne greške o CFI oporavku osnovnih fizičkih hostova. **Ignorisati** — NetWorker ne backup-uje osnovni fizički host u virtuelnom okruženju. | **[DR]** |

Primer lažne greške **[DR]**:

```
9348:nsrck: The index recovery for 'EXCH2010-2.vll1.local' failed.
9431:nsrck: can't find index backups for 'EXCH2010-2.vll1.local' on server 'sa-wq.vll1.local'
```

## 57.6 Oporavak NMC baze

> **[DR]** Ako želite da NMC prikaže sve resurse posle `nsrdr`-a, potrebno je oporaviti i NMC bazu komandom `recoverpsm`.
>
> Primer: ako je Data Domain dodat **pre** Server DR-a, da bi bio prikazan **posle** DR-a mora se pokrenuti `recoverpsm`.

**NMC backup sadrži [DR]:** NMC database fajlove, NMC database credential fajl (`gstd_db.conf`), NMC lockbox fajlove, legacy authentication konfiguracione fajlove.

**Priprema [DR]:**

```bash
# 1. Zaustaviti NMC servise (za oporavak na originalnu lokaciju)
/etc/init.d gst stop

# 2. Opciono: utvrditi nsavetime ranijeg backup-a
mminfo -avot -q client=<NMC_Server>,level=full -r client,name,savetime,nsavetime
#    nsavetime je u poslednjoj koloni

# 3. Postaviti LD_LIBRARY_PATH ka postgres bibliotekama
export LD_LIBRARY_PATH=<NMC_Installation_dir>/postgres/lib
#    podrazumevano: /opt/lgtonmc/postgres/lib

# 4. Preći u NMC bin direktorijum
cd /opt/lgtonmc
```

**Oporavak [DR]:**

```bash
recoverpsm -s <NetWorker_server> -c <source_NMC_server> /nsr/nmc/nmcdb_stage

# ili sa AES passphrase i alternativnim direktorijumom
recoverpsm -f -s <NetWorker_server> -c <source_NMC_server> \
           -p <AES_Passphrase> <staging_dir> -d <dir_name>
```

| Opcija | Značenje **[DR]** |
|---|---|
| `-f` | Briše fajlove baze koji trenutno postoje u direktorijumu baze. **Ne koristiti** ako želite vraćanje na drugu lokaciju. |
| `-s` | Ime NetWorker servera |
| `-c` | Ime izvornog NetWorker Authentication Service-a, kada se vraća na drugi host |
| `-p` | Passphrase korišćen pri NMC backup-u. **Potreban ako je pri backup-u postavljen datazone pass phrase.** |
| `<staging_dir>` | Staging direktorijum korišćen pri backup-u baze na izvoru |
| `-d <dir_name>` | Direktorijum za premeštanje oporavljenih fajlova baze. Uz ovu opciju **fajlove morate ručno kopirati** u direktorijum baze i **zadržati isto vlasništvo i dozvole**. |

> **[DR]** Tokom oporavka Authentication Service baze **konzola nije dostupna**, pa se zahtevi za montiranje ne mogu obraditi iz nje. Pratiti daemon log fajlove; `nsr_render_log` čini `daemon.raw` čitljivijim. Koristiti `nsrwatch` za pregled poruka i `nsrjb` za odgovor na njih.

```bash
# Pokretanje NMC servisa posle oporavka
/etc/init.d gst start
```

---

# 58. Oporavak podataka

## 58.1 Priprema

> **[DR]** Koraci za oporavak aplikativnih i korisničkih podataka:

```bash
# 1. Prijaviti se kao root
whoami

# 2. Učitati i inventarisati uređaje
#    Time NetWorker server prepoznaje lokaciju svakog volumena
nsrjb -I
```

> **[DR]** Ako učitavate **klonirani volumen**, ili obrišite originalni volumen iz media baze, ili označite željeni save set kao **suspect**. Ako koristite klonirani volumen, on se koristi do kraja procesa oporavka.

## 58.2 `recover`

```bash
recover
```

Zatim u interaktivnoj sesiji:

```
recover> add <putanja>     # označavanje fajlova/direktorijuma
recover> a                 # izbor ili dodavanje SVIH fajlova
recover> recover           # pokretanje oporavka
```

> **[DR] NOTE:** *Overwriting operating system files may cause unpredictable results.*
> Prepisivanje fajlova operativnog sistema može dovesti do nepredvidivih rezultata.

> **[CMD]** Detalje daje `recover` man stranica i *NetWorker Command Reference Guide*:
> ```bash
> man recover
> recover -help
> ```

## 58.3 Pomoć na sistemu

> **[CMD]** Informacije iz Command Reference Guide-a dostupne su i iz komandne linije na svim platformama osim Windows-a:
>
> ```bash
> man <ime_komande>
> man recover
> ```
>
> **Za to mora biti instaliran opcioni paket `LGTOman`**, a putanja do man stranica mora biti u `MANPATH` promenljivoj — u suprotnom `man` treba pokretati iz instalacione lokacije man stranica.
>
> Osnovna pomoć na svim platformama:
> ```bash
> <ime_komande> -help
> recover -help
> ```

**Provera na RHEL 9.6:**

```bash
rpm -qa | grep -i lgtoman
man -w recover
```

> **Preporuka za vault:** instalirati `LGTOman` na vault instancu. U izolovanom okruženju bez pristupa internetu, `man` stranice su jedini priručnik na licu mesta.

---

# 59. Kada oporavak ne uspe

## 59.1 Vraćanje na prethodno stanje — CR oporavak

Ako se automatizovani Cyber Recovery oporavak prekine i ne završi čisto. **[CR]**

```bash
# 1. Zaustaviti NetWorker
/etc/init.d/networker stop

# 2. Resource baza
ls -ld /nsr/res.cr.*                      # pronaći najnoviji
rm -rf /nsr/res                           # ukloniti tekući
mv /nsr/res.cr.<timestamp> /nsr/res       # vratiti prethodni

# 3. Media baza
ls -ld /nsr/mm.cr.*
rm -rf /nsr/mm
mv /nsr/mm.cr.<timestamp> /nsr/mm

# 4. Index baza
ls -ld /nsr/index.cr.*
rm -rf /nsr/index
mv /nsr/index.cr.<timestamp> /nsr/index

# 5. Pokrenuti NetWorker
/etc/init.d/networker start
```

## 59.2 Vraćanje na prethodno stanje — ručni `nsrdr`

Kod ručnog `nsrdr`-a folderi imaju drugačiji sufiks. **[CMD]**

```bash
ls -ld /nsr/res.*
# /nsr/res.<timestamp>  — prethodna resource baza
# /nsr/res.R            — privremeni folder oporavljene baze
```

Isti obrazac vraćanja kao gore, sa `.<timestamp>` umesto `.cr.<timestamp>`.

## 59.3 Lokacije media baze

> **[CMD]**
>
> | Putanja | Sadržaj |
> |---|---|
> | `/nsr/mm/mmvolume6` | Media baza u **legacy** formatu. Aktivna media baza **pre** migracije ili posle neuspele migracije. |
> | `/nsr/mm/mmvolrel` | Media baza u **relacionom** formatu. Aktivna media baza **posle** migracije ili kod nove instalacije. |

```bash
ls -ld /nsr/mm/mmvolume6 /nsr/mm/mmvolrel 2>/dev/null
```

> Prisustvo `mmvolume6` uz odsustvo `mmvolrel` ukazuje da migracija nije izvršena ili nije uspela — vredi proveriti pre nego što se krene u oporavak.

## 59.4 Brisanje objekata koje je oporavak kreirao

Posle **automatizovanog CR** oporavka. **[CR]**

> **PAŽNJA:** NetWorker server može sadržati i druge objekte koji su postojali pre CR recovery job-a. **Brisati isključivo objekte koje je Cyber Recovery kreirao.**

| Tab | Šta obrisati |
|---|---|
| **Protection** | Novododate klijente, politike, grupe, ostale tipove zaštite |
| **Devices** | Novododate uređaje, DD sisteme, storage node-ove (po potrebi) |
| **Media** | Novododate disk volumene, media pool-ove, ostale tipove medija |

## 59.5 Unmount sandbox-a sa CR management hosta

```bash
umount /opt/dellemc/cr/mnt/cr-rec-<networker_sandbox>_1604
```

## 59.6 Migracija media baze kao alternativa

> **[ADM]** NetWorker 19.9 i noviji omogućava uvoz media baze i klijenata sa jednog servera i njihovu migraciju na drugi, komandama `nsrmmdbasm`, `nsrimportmmdb` i `nsrimportclient`. Komande su međusobno nezavisne.
>
> Ako uvozite i media bazu i klijente, **prvo uvezite media bazu**, jer ona sadrži client ID mapiranja koja povezuju referentne brojeve klijenata sa imenima.

```bash
# 1. Backup media baze na IZVORNOM serveru
nsrmmdbasm -s "<media_database_path_on_source_server>" > mmdb.xdr

# 2. Uvoz media baze
nsrimportmmdb -w -s "<source_server>" -d "<destination_server>" < mmdb.xdr

# 3. Uvoz klijenata
nsrimportclient -w -s "<source_server>" -d "<destination_server>" '*'
#    '*' predstavlja sve klijente
```

> **[ADM] Ograničenje:** **indeksi se trenutno ne uvoze.**

> **[DR]** `nsrmmdbasm` je **jedina** komanda koja se sme pokretati u disaster recovery stanju.

## 59.7 Kada zvati Dell Support i sa čim

| # | Priložiti |
|---|---|
| 1 | `/nsr/logs/nsrdr.log` |
| 2 | `nsr_render_log /nsr/logs/daemon.raw > /tmp/daemon.txt` |
| 3 | Izlaz `mminfo -B` (ako media baza postoji) |
| 4 | Izlaz `scanner -B <device>` |
| 5 | Izlaz `nsrmm -C` |
| 6 | Izlaz `rpm -qa \| grep -i lgto` |
| 7 | Vrednost `server state` atributa u NSR resursu |
| 8 | `ls -ld /nsr/res* /nsr/mm* /nsr/index*` |
| 9 | Tačna komanda koja je pokrenuta, sa svim opcijama |
| 10 | Verzija DDOS-a i izlaz `ddboost storage-unit show` |

---

# Dodatak: Kartica komandi

## Provera stanja

```bash
rpm -qa | grep -i lgto              # instalirane komponente
hostname -f                         # FQDN
id nsrtomcat                        # UID koji mora odgovarati produkciji
df -h /nsr                          # prostor
ls -ld /nsr                         # da li je simbolički link
ls -ld /nsr/mm/mmvolume6 /nsr/mm/mmvolrel
nsrmm -C                            # šta je montirano na čemu
nsrwatch                            # praćenje poruka servera
```

## DD strana

```bash
ddboost storage-unit show
ddboost storage-unit modify <mtree> user <ddboost-user>
mtree list
replication show config
replication status
user show list
filesys fastcopy source <src> destination <dst>
```

## Server state

```bash
nsradmin -s <server>
  . type: NSR
  p
  update server state: disaster recovery
  # posle oporavka:
  update server state: active
```

## Rad bez pokrenutog servera

```bash
cd /nsr/res && nsradmin -d nsrdb
  . type: NSR device
  p
```

## Pronalaženje bootstrap-a

```bash
grep -A 40 "Server backup Action report" /nsr/logs/policy_notifications.log | tail -60
mminfo -B                           # 4. kolona = ssid, 5. = file, 6. = record
scanner -B <device>                 # kada media baza ne postoji
```

## Oporavak

```bash
nsrdr                               # interaktivno
nsrdr -a -B <ssid> -d <device> -I   # neinteraktivno, svi CFI
nsrdr -c -I client1 client2         # samo izabrani CFI
nsrdr -t "1 Month ago" -I <klijent> # CFI na trenutak u prošlosti
/opt/nsr/authc-server/scripts/authc_configure.sh
```

## Katalog posle oporavka

```bash
scanner -m <device>                 # preporučeno kada postoje index backup-i
nsrck -L7 -t "<datum>" <klijent>    # zatim ovo
scanner -i <device>                 # rezervni put, bez index backup-a
nsrmm -o notscan <volume>           # uklanjanje scan needed sa trake
nsrck -L 6 <klijent>                # dublja provera pri sumnji na oštećenje
```

## Vraćanje unazad

```bash
/etc/init.d/networker stop
mv /nsr/res.cr.<ts> /nsr/res        # ili res.<ts> kod ručnog nsrdr
mv /nsr/mm.cr.<ts> /nsr/mm
mv /nsr/index.cr.<ts> /nsr/index
/etc/init.d/networker start
```

## Pomoć na sistemu

```bash
man recover
man nsr_device
man nsrdr
recover -help
```

---

*Kraj Dela VII-c. Videti i: `07a` (anatomija instalacije), `07b` (storage i katalog), `07d` (logovi i troubleshooting).*
