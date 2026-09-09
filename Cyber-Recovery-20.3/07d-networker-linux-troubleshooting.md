# DEO VII-d — DIJAGNOSTIKA I TROUBLESHOOTING

**Dell PowerProtect Cyber Recovery 20.3 — Uputstvo za instalaciju i integraciju**
Verzija dokumenta: 0.1 | **Namena: sistem administratori i System Engineer-i**
**Referentni OS:** Red Hat Enterprise Linux 9.6 | **Referentni NetWorker:** 19.11 / 19.13

> **Izvori:**
> - *Dell NetWorker 19.11 Command Reference Guide* — **[CMD]**
> - *Dell NetWorker 19.13 Administration Guide* — **[ADM]**
> - *Dell NetWorker Server Disaster Recovery and Availability Best Practices Guide 19.13* — **[DR]**
> - *Dell PowerProtect Cyber Recovery 20.3 Product Guide / Installation and Upgrade Guide* — **[CR]**

> **Namena ovog dela:** dijagnostika po simptomu i prikupljanje podataka za Dell Support. Mapa log fajlova je u `07a`, poglavlje 45 — ovde se bavimo **nivoima logovanja** i **konkretnim kvarovima**.

---

## Sadržaj

- [60. Nivoi logovanja i debug](#60-nivoi-logovanja-i-debug)
- [61. Dijagnostika po simptomu](#61-dijagnostika-po-simptomu)
- [62. Prikupljanje podataka za Dell Support](#62-prikupljanje-podataka-za-dell-support)
- [63. Brza referenca](#63-brza-referenca)

---

# 60. Nivoi logovanja i debug

## 60.1 Tri načina povećanja detaljnosti

| Način | Kada se koristi | Zahteva restart |
|---|---|---|
| **`dbgcommand`** | Proces **već radi**, hoćete više detalja bez prekida | **Ne** |
| **Troubleshoot režim** | Problem je pri **startu** procesa | Da |
| **`runtime rendered log`** | Trajno, radi čitljivosti logova | Da (za `runtime rollover by time`) |

> **Za vault:** `dbgcommand` je gotovo uvek bolji izbor. Zaustavljanje NetWorker servisa u toku oporavka može ostaviti nekonzistentno stanje.

## 60.2 `dbgcommand` — izmena nivoa bez restarta **[CMD]**

> `dbgcommand` se koristi za postavljanje debug nivoa raznih NetWorker procesa **koji rade**. Može se koristiti i za promenu nivoa verbosity i trace nivoa procesa koji radi, ili za uključivanje ispisa message ID-a poruke u log fajlu.
>
> Za **`nsrd`, `nsrmmgd` i `nsrexecd`** procese mogu se dobiti i **dodatne debug informacije** odgovarajućim opcijama komande.

### Sintaksa

```
dbgcommand -p <pid>[,<pid>...] <parametar>
dbgcommand -n <process-name> [-n <process-name>...] <parametar>
```

| Selektor | Značenje |
|---|---|
| `-p <pid>` | Process ID NetWorker procesa koji radi |
| `-n <process-name>` | Ime NetWorker procesa koji radi |

### Parametri nivoa **[CMD]**

| Parametar | Opseg | Značenje |
|---|---|---|
| `Debug=<level>` | **0–9** | Menja debug nivo procesa. **`0` isključuje debagovanje.** |
| `Vflag=<level>` | **0–9** | Menja nivo verbosity. `0` isključuje. |
| `Trace=<level>` | **0–9** | Menja trace nivo. `0` isključuje. |
| `MsgID=<toggle-flag>` | 0 ili 1 | Uključuje/isključuje ispis **message ID-a** poruke u log fajlu. `0` = isključeno, `1` = uključeno. |
| `LogDnsThreshold=<threshold>` | — | Prag za logovanje DNS operacija |

### Dijagnostičke akcije **[CMD]**

Ovo su komande koje daju trenutni snimak internog stanja — vrlo korisne u vault-u.

| Parametar | Šta ispisuje / radi |
|---|---|
| **`PrintDevInfo`** | Informacije o uređajima |
| **`PrintDeviceSelection`** | Kako je izabran uređaj |
| **`PrintNsrAuthInfo`** | Informacije o NetWorker autentikaciji |
| **`PrintDnsCache`** | **Sadržaj DNS keša** |
| **`FlushDnsCache`** | **Prazni DNS keš** |
| **`JobsdbFullPurge`** | Puno čišćenje job baze |

### Praktični primeri

```bash
# Podizanje debug nivoa za nsrd bez restarta
dbgcommand -n nsrd Debug=5

# Uključivanje message ID-a — olakšava traženje greške u dokumentaciji
dbgcommand -n nsrd MsgID=1

# Šta NetWorker misli da zna o imenima hostova
dbgcommand -n nsrd PrintDnsCache

# Pražnjenje DNS keša posle izmene /etc/hosts
dbgcommand -n nsrd FlushDnsCache

# Stanje uređaja
dbgcommand -n nsrd PrintDevInfo
dbgcommand -n nsrmmd PrintDeviceSelection

# Autentikacija
dbgcommand -n nsrexecd PrintNsrAuthInfo

# Vraćanje na normalu — OBAVEZNO posle prikupljanja
dbgcommand -n nsrd Debug=0
dbgcommand -n nsrd MsgID=0
```

> **`PrintDnsCache` i `FlushDnsCache` su izuzetno korisni u vault-u.** Kada izmenite `/etc/hosts` (vidi `07c`, odeljak 53.4), NetWorker i dalje koristi stari keš. Umesto restarta servisa — `FlushDnsCache`.

> **Ne zaboraviti da se debug vrati na `0`.** Debug nivo 9 na opterećenom serveru brzo puni `daemon.raw` i utiče na performanse.

## 60.3 Troubleshoot režim pri startu **[ADM]**

Koristi se kada proces **ne uspeva da startuje**, pa `dbgcommand` nema šta da uhvati.

> Na NetWorker serveru mogu se konfigurisati **`nsrctld`** i **`nsrexecd`** da startuju u troubleshoot režimu. `nsrctld` demon pokreće ostale demone po potrebi. **Za hvatanje troubleshoot izlaza demona koje pokreće `nsrctld`, koristiti `dbgcommand`.**

```bash
# 1. Zaustaviti NetWorker procese
nsr_shutdown

# 2a. nsrexecd — problemi sa klijentskim funkcijama
nsrexecd -D9 1>/tmp/nsrexecd_debug.log 2>&1

# 2b. nsrctld — problemi sa serverom
source /opt/nsr/admin/networkerrc
source /opt/nsr/admin/nsr_serverrc
nsrctld -D 9 1>/tmp/nsrctld_debug.log 2>&1

# 3. Po prikupljanju — zaustaviti i vratiti u normalan režim
nsr_shutdown
systemctl start networker
```

## 60.4 NMC troubleshoot režim **[ADM]**

**Kada NMC GUI radi:** Setup → System Options → **Debug Level** → broj između **1 i 20**. NMC čuva informacije u `gstd.raw`. Posle prikupljanja postaviti Debug Level na **0** i restartovati servise.

**Kada NMC GUI ne radi** — preko promenljive okruženja:

```bash
# 1. Izmeniti dozvole startnog fajla (podrazumevano read-only)
chmod u+w /etc/init.d/gst

# 2. Na POČETAK fajla dodati
vi /etc/init.d/gst
```

```sh
GST_DEBUG=10
export GST_DEBUG
```

```bash
# 3. Restartovati
/etc/init.d/gst stop
/etc/init.d/gst start
```

> **[ADM]** Vrednost je broj između **1 i 20**. Informacije se čuvaju u `gstd.raw`.

## 60.5 Trajno čitljivi logovi

Videti `07a`, odeljak 45.5 — atribut `runtime rendered log` u NSRLA bazi.

```bash
nsradmin -p nsrexec <<'EOF'
. type: NSR log; name: daemon.raw
update runtime rendered log: "/nsr/logs/daemon.log"
EOF
```

---

# 61. Dijagnostika po simptomu

## 61.1 Servis ne startuje ili operacije ne uspevaju

### Simptom **[ADM]**

```
Server not available
RPC error, no remote program registered
```

> **[ADM]** NetWorker arhitektura prati klijent/server model, gde NetWorker serveri koriste **RPC** za pružanje usluga klijentu. Te usluge žive u demon procesima. **Kada demoni startuju, registruju se kod registracione usluge koju pruža portmapper.**
>
> Ako NetWorker usluge ne rade a operacija ih zatraži, pojavljuje se navedena poruka. Ona označava da **jedna ili više NetWorker usluga ne rade na serveru.**

### Dijagnostika

```bash
# 1. Da li servis uopšte radi
systemctl status networker
/etc/init.d/networker status

# 2. Koji procesi rade
ps -ef | grep /usr/sbin/nsr

# 3. Da li portmapper radi (RPC)
systemctl status rpcbind
rpcinfo -p localhost | grep -E "390103|390113"

# 4. Šta piše u logu
nsr_render_log /nsr/logs/daemon.raw | tail -100

# 5. systemd žurnal
journalctl -u networker --since "1 hour ago"
```

> **[CMD]** RPC program broj **390103** je `nsrd`; **390113** je `nsrexecd`.

### Rešenje

```bash
systemctl start networker
/etc/init.d/networker status    # provera stabla procesa
```

Ako i dalje ne startuje — troubleshoot režim (60.3), pa SELinux provera (`07a`, poglavlje 46).

---

## 61.2 Server startuje veoma sporo

Dva potpuno različita uzroka koja se često mešaju.

### Uzrok A — velika media baza **[ADM]**

> **Provera konzistentnosti media baze**, koja se izvršava pri startu NetWorker server servisa, **može trajati značajno dugo kada je media baza veoma velika.** Dok server izvršava proveru, **konekcije klijenata ka serveru kasne.**

**Rešenje [ADM]:**

```bash
# Smanjenje veličine media management baze
nsrim -C
```

> **[ADM]** Pokrenuti kada je NetWorker server **neaktivan**. Komanda **može trajati veoma dugo** i **NetWorker server će biti nedostupan** za to vreme. **Ne mogu se izvršavati NetWorker server operacije dok se komanda ne završi.**

### Uzrok B — rezolucija imena **[DR]**

> Problemi sa rezolucijom hostname-ova mogu učiniti NetWorker server neodgovarajućim ili vrlo sporim pri pokretanju, ako:
> - NetWorker server koristi DNS, ali DNS server nije dostupan
> - NetWorker server ne može da razreši sve svoje klijentske hostove

**Dijagnostika:**

```bash
# Šta NetWorker misli da zna
dbgcommand -n nsrd PrintDnsCache

# Redosled rezolucije
grep "^hosts:" /etc/nsswitch.conf

# Da li DNS uopšte odgovara
getent hosts <neki_klijent>
dig +short <neki_klijent>
```

**Rešenje** — puna procedura u `07c`, odeljak 53.4:

```bash
# hosts pre DNS-a
sed -i 's/^hosts:.*/hosts: files/' /etc/nsswitch.conf

# Nepoznati klijenti na loopback
echo "127.0.0.1  nepoznat-klijent.example.com  nepoznat-klijent" >> /etc/hosts

# Osvežiti keš bez restarta
dbgcommand -n nsrd FlushDnsCache
```

### Kako razlikovati uzroke

| Pokazatelj | Uzrok |
|---|---|
| `mminfo -m \| wc -l` daje veliki broj volumena | A — media baza |
| `du -sh /nsr/mm` je veliko | A — media baza |
| `dbgcommand -n nsrd PrintDnsCache` pokazuje mnogo nerazrešenih unosa | B — DNS |
| DNS server u vault-u nije dostupan | B — DNS |

---

## 61.3 Uređaj je izgubio konekciju

### DCC framework **[ADM]**

> Kada uređaj izgubi konekciju sa NetWorker serverom, **Device Connectivity Check (DCC)** framework otkriva nedostupnost uređaja. Po otkriću, NetWorker server obaveštava korisnika kroz **NMC konzolu, NetWorker logove ili email.**

**Scenariji u kojima uređaji postaju nedostupni [ADM]:**

| # | Scenario |
|---|---|
| 1 | **Data Domain koji hostuje uređaje postaje nedostupan** |
| 2 | **DD administrator slučajno obriše MTree koji sadrži uređaje** |
| 3 | Mount point koji hostuje AFTD uređaje postaje offline |
| 4 | AFTD device folder je slučajno obrisan iz OS fajl sistema |

> Scenariji 1 i 2 su najrelevantniji za vault.

**Mogućnosti DCC framework-a [ADM]:**

- DCC periodično proverava konekciju uređaja
- **DCC je podrazumevano omogućen.** Vidljiv je u diagnostic režimu u NMC konzoli.
- Storage node prijavljuje DCC rezultat serveru
- Server parsira izveštaj i **premešta uređaj u `normal` ili `suspected` stanje**
- Ako se suspect status uređaja promenio, server prijavljuje događaj kroz NMC, logove ili email

> **[ADM]** Od NetWorker 19.3 DCC verifikuje postojanje i dostupnost **DD, DDCT i AFTD** uređaja. Od 19.7 uključuje i **SmartScale** uređaje.

> **[ADM] Ograničenje:** DCC **ne proverava postojanje save set-ova** koji leže na volumenu. Proverava postojanje `VolHdr` fajla.

### Dijagnostika

```bash
# Atribut suspected device (skriven)
nsradmin -s <server> <<'EOF'
option hidden
. type: NSR device
show name; suspected device; enabled; message
p
EOF

# Poruke vezane za uređaje
tail -100 /nsr/logs/media.log

# Da li je DD dostupan
ping -c3 <dd_hostname>
ssh sysadmin@<dd> "filesys status"
ssh sysadmin@<dd> "ddboost storage-unit show"
ssh sysadmin@<dd> "mtree list"
```

### Rešenje

Redosled provere: DD dostupan → MTree postoji → storage unit vidljiv → uređaj omogućen.

```bash
# Ako je MTree obrisan ili replikacija prekinuta — vidi 07b, odeljak 48.7
ssh sysadmin@<dd> "ddboost storage-unit modify <mtree> user <ddboost-user>"

# Ponovo omogućiti uređaj
nsradmin -s <server> <<'EOF'
. type: NSR device; name: <ime_uredjaja>
update enabled: Yes
EOF
```

---

## 61.4 `scanner` je označio volumen kao read-only

### Simptom **[ADM]**

> Kada koristite `scanner` program da ponovo izgradite indeks backup volumena, **`scanner` označava volumen kao read-only.**
>
> **Ovo je bezbednosna funkcija** koja sprečava NetWorker da prepiše poslednji save set na backup volumenu.

### Rešenje **[ADM]**

```bash
nsrmm -o notreadonly <volume_name>
```

> **U vault-u ovo često ne treba menjati.** Ako je volumen namenjen samo oporavku, read-only je poželjno stanje.

---

## 61.5 `scanner` traži veličinu zapisa

### Simptom **[ADM]**

> Ako koristite `scanner` sa `-s` opcijom **ali bez `-i` ili `-m`**, može se pojaviti poruka:

```
Please enter record size for this volume ('q' to quit)
```

### Rešenje **[ADM]**

> Navesti veličinu bloka **veću ili jednaku 32**.

> **[CMD]** Uz `-S` bez `-i` ili `-m`, `scanner` traži veličinu bloka volumena — **ali samo ako labela volumena nije čitljiva.**

**Praktično:** ako se ovo pojavi, prvo proverite da li je labela uopšte čitljiva:

```bash
nsrmm -p -f <device>      # verifikacija labele (demontira volumen!)
```

---

## 61.6 Client file index ne postoji

### Simptom **[ADM]**

```
scanner: File index error, file index is missing.
Please contact your system administrator to recover or recreate the index.
(severity 5, number 8)
scanner: write failed, Broken pipe
scanner: ssid 25312: scan complete
```

> **[ADM]** Pre upotrebe `scanner` programa sa `-i` opcijom, obezbediti da **client file index postoji za klijenta pridruženog svakom save set-u.**

### Rešenje **[ADM]**

```bash
# Kreirati client file index za klijenta
nsrck -L2 <ime_klijenta>

# Zatim ponoviti scanner
scanner -i <device>
```

> Ovo je klasična zamka u vault-u: sveža NetWorker instanca **nema nijedan CFI**, pa `scanner -i` pada pre nego što išta uradi.

---

## 61.7 Sumnja na oštećen client file index

### Kontekst **[ADM]**

> Svaki put kada NetWorker server startuje, startni proces koristi **`nsrck -ML1`** za proveru konzistentnosti nivoa 1 nad client file indeksima. **U nekim okolnostima ta provera ne otkriva oštećenje.**

### Rešenje **[ADM]**

```bash
# Viši nivo provere
nsrck -L5 <ime_klijenta>

# Ako ni to ne reši — vidi 61.8
```

> **[CMD] Zamka kod nivoa 7:** ako je `.rec` fajl u indeksu oštećen, a **`nsrck -L5` nije izvršen da prvo očisti oštećeni save set**, onda `nsrck -L7` **neće prepisati** oštećeni `.rec` fajl i indeks ostaje oštećen.

**Redosled pri oštećenju:**

```bash
nsrck -L5 <klijent>        # 1. očistiti oštećeno
nsrck -L7 <klijent>        # 2. tek onda vratiti sa medija
```

Puna tabela nivoa 1–7 je u `07c`, odeljak 57.3.

---

## 61.8 Istekli save set nije browsable

### Kontekst **[ADM]**

> Kada je tekući datum jednak retention datumu, NetWorker istekne save set i označava ga kao **eligible for recycling**. Kada save set ima taj status, **NetWorker uklanja informacije o njemu iz CFI-ja i ne možete izvršiti browsable oporavak podataka.**
>
> **Neke aplikacije, kao što je NetWorker Module for Databases and Applications, zahtevaju da save set bude browsable da bi se izvršio oporavak.**

> Istekle save set fajlove možete učiniti browsable za oporavak **dodavanjem informacija o save set-u nazad u client file index.**

### Postupak

**Korak 1 — utvrditi status save set-a [ADM]:**

Kroz NMC: **Administration → Media → Save Sets → All Save Sets → Query Save Set**, sa kriterijumima: Client Name, Save Set, Save Set ID, Volume, Pool, Checkpoint ID.

Iz konzole:

```bash
mminfo -a -v -q "client=<klijent>,name=<save_set>" \
  -r 'ssid(53),client,name,savetime,ssbrowse,ssretent,ssflags'
```

**Korak 2 — vratiti informacije u CFI:**

```bash
# Ako postoje index backup-i — preporučeni put
scanner -m <device>
nsrck -L7 -t "<datum>" <ime_klijenta>

# Ako index backup-i ne postoje
scanner -i -S <ssid> <device>
```

> Detalji o izboru između `-m` i `-i` su u `07b`, odeljak 51.3.

---

## 61.9 Volumen ima save set-ove nepoznate media bazi

### Simptom **[DR]**

```
nw_server nsrd media info: Volume <volume_name> has save sets unknown to media database.
Last known file number in media database is ### and last known record number is ###.
Volume <volume_name> must be scanned; consider scanning from last known file and record numbers.
```

### Kontekst

Ovo je očekivano posle `nsrdr`-a. Objašnjenje je u `07c`, odeljak 56.7.

### Rešenje

```bash
# 1. Zabeležiti file i record broj iz poruke
# 2. Skenirati od te pozicije
scanner -f <file> -r <record> -i <device>

# 3. Ukloniti zastavicu
nsrmm -o notscan <volume_name>
```

Za AFTD/DD volumene puna procedura je u `07b`, odeljak 51.8.

---

## 61.10 Oporavak indeksa na drugu lokaciju ne uspeva

### Simptom **[ADM]**

```
WARNING: The on-line index for <client_name> was NOT fully recovered.
There may have been a media error. You can retry the recover, or attempt to
recover another version of the index.
```

### Rešenje **[ADM]**

> Obezbediti da se **indeksi oporavljaju na originalnu lokaciju**, pa ih zatim premestiti u drugi direktorijum.

---

## 61.11 Preimenovan klijent ne može da oporavi stare backup-e

### Kontekst **[ADM]**

> NetWorker server održava client file index za **svakog klijenta koji je backup-ovan.** Kada promenite ime klijenta, NetWorker koristi **novi hostname da kreira novi client file index** — pa **ne možete oporaviti fajlove backup-ovane pod starim imenom klijenta.**

### Rešenje **[ADM]**

> Za oporavak podataka backup-ovanih pod starim imenom, izvršiti **directed recovery** i navesti **staro ime klijenta kao izvorni host**, a **novo ime kao odredišni host.**

> **Relevantno za vault:** ako NetWorker instanca u vault-u ima drugačiji hostname od produkcijske (što se ne preporučuje — vidi `07c`, odeljak 53.3), ovo postaje realan problem pri oporavku klijentskih podataka.

---

## 61.12 Unapproved server error

### Simptom **[ADM]**

```
<client_name>: <server_name> cannot request command execution
```

Ili pri dodavanju Windows klijenta na UNIX server:

```
<client_name>: <saveset_name> Host <server_name> cannot request command execution
<client_name>: <saveset_name> nsrexec: Host <server_name> cannot request command execution
<client_name>: <saveset_name> Permission denied
```

### Rešenje **[ADM]**

```bash
# 1. Na KLIJENTU izmeniti servers fajl — mora sadržati i kratko i dugo ime servera
vi /nsr/res/servers
```

```
mars
mars.jupiter.com
```

> **[ADM]** 2. U **Alias** atributu Client resursa navesti **i kratko i dugo ime**, i sve druge primenljive aliase klijenta.

```bash
nsradmin -s <server> <<'EOF'
. type: NSR client; name: <klijent>
update aliases: <klijent>, <klijent>.<domen>
EOF
```

> **[ADM]** Pri dodavanju Windows klijenta na UNIX server: **poruku ignorisati** i nastaviti sa dodavanjem klijenta. Da bi se izbegla, dodati hostname UNIX servera u `servers` fajl na klijentu **posle** dodavanja klijenta.

---

## 61.13 Server copy violation — server je onemogućen

### Simptom **[ADM]**

```
nsrd: registration info event: server is disabled copy violation
```

### Uzrok **[ADM]**

> Kada **Alias** atribut Client resursa za NetWorker server **ne sadrži sva imena hostova ili aliase** NetWorker servera, server može postati onemogućen.

### Rešenje **[ADM]**

> Dodati **sve aliase servera** vezane za dodatne mrežne interfejse u listu aliasa Client resursa NetWorker servera.

```bash
nsradmin -s <server> <<'EOF'
. type: NSR client; name: <server_fqdn>
p
update aliases: <shortname>, <fqdn>, <alias1>, <alias2>
EOF
```

> **[DR]** Isti korak je deo standardne DR procedure: na tabu **Globals (1 of 2)** proveriti da atribut **Aliases** sadrži ispravne hostname-ove NetWorker servera.

---

## 61.14 Promena IP adrese servera — licenca

### Kontekst **[ADM]**

> Kada se IP adresa NetWorker servera promeni, **menja se i NetWorker hostid**. Autorizacioni kod dodeljen svakoj NetWorker licenci **zavisi od hostid-a.**
>
> Kada se hostid promeni, morate kontaktirati Dell Licensing radi generisanja novih autorizacionih kodova i ažurirati svaku licencu.
>
> **Ako softver ne registrujete novim kodovima u roku od 14 dana od promene hostid-a, NetWorker se onemogućava i ne možete izvršavati nikakve operacije osim oporavka.**

> **[CMD]** Ista situacija pri oporavku bootstrap-a **na drugi host**: kontaktirati Licensing **u roku od 15 dana** radi *host transfer affidavit* postupka. Ako se kodovi ne unesu, server se onemogućava i mogu se izvršavati samo oporavci.

> **Neusklađenost izvora:** *Administration Guide* navodi **14 dana** (promena IP adrese), *Command Reference* navodi **15 dana** (oporavak na drugi host). Verovatno su u pitanju dva različita scenarija. **Za planiranje uzeti kraći rok — 14 dana.**

> **[ADM]** Ako koristite DHCP, **koristiti statičku IP adresu za NetWorker server.**

**Za vault:** instanca u vault-u je po definiciji drugi host. Ovo se gotovo uvek aktivira — planirati sa kupcem unapred (vidi `07c`, odeljak 52.4).

---

## 61.15 Bootstrap ne može da se upiše

### Ograničenje **[ADM]**

> **NetWorker piše bootstrap backup-e samo na lokalni uređaj.** Kada group backup generiše bootstrap save set, obezbediti da uređaj priključen NetWorker serveru **ima dostupan volumen za bootstrap backup.**

### Povezano ograničenje pri oporavku **[DR]**

> NetWorker **ne podržava oporavak bootstrap-a sa udaljenog uređaja.** Bootstrap sa kloniranog save set-a na udaljenom uređaju mora se prvo klonirati na uređaj **lokalan za NetWorker server.**

Procedura je u `07c`, odeljak 55.5.

---

## 61.16 Nedozvoljeni znaci u konfiguraciji

### Ograničenje **[ADM]**

> Pri davanju imena za **label template-e, direktive, grupe, politike i rasporede** ne koristiti sledeće znake:

```
/ \ * [ ] ( ) $ ! ^ ' " ? ; ` ~ < > & | { } ,
```

> **Uporediti sa PPDM ograničenjem** (Deo V, odeljak 26.5), gde je za ime VM-a u vCenter-u zabranjen sličan, ali ne identičan skup znakova. Pri definisanju konvencije imenovanja (Deo I, odeljak 3.4) uzeti **presek oba skupa.**

---

## 61.17 Nema privilegija za prikaz servera iz NMC-a

### Uzrok **[ADM]**

> NetWorker administrator neće imati privilegiju da vidi NetWorker resurse ako postoji **neslaganje u eksternim ulogama** između NetWorker user grupe i eksterne uloge prikazane u NMC-u.
>
> **Kada se NetWorker server razrešava u više od jednog hostname-a** (kada ima više aliasa), **postoji mogućnost da je server kreirao eksternu ulogu sa jednim od tih hostname-ova.**

### Rešenje **[ADM]**

> Obezbediti da se NetWorker server razrešava u **samo jedan hostname** i da je `hosts` unos za NetWorker server **isti u NMC-u i na NetWorker serveru.**

```bash
# Na oba hosta
getent hosts <networker_server>
grep <networker_server> /etc/hosts
```

---

## 61.18 Servisni režim — kontrolisano isključivanje pristupa

Nije kvar, nego alat koji vredi znati.

> **[ADM]** Za omogućavanje i onemogućavanje pristupa NetWorker serveru koriste se atributi **Accept new sessions** i **Accept new recover sessions** u NMC-u. Kada ih poništite, **server ne prihvata nove backup i recovery sesije.**
>
> Kada ograničite pristup serveru, **NetWorker prebacuje sve storage node-ove offline**, čime server efektivno prelazi u **service mode** operativno stanje. U tom stanju možete zaustaviti sve eksterne klijentske backup i recovery zahteve i sprečiti pokretanje zakazanih group backup-a.
>
> **Service mode daje period održavanja u kojem možete dijagnostikovati i rešavati probleme pre nego što vratite server u normalan rad.**

```bash
nsradmin -s <server> <<'EOF'
. type: NSR
update accept new sessions: No
update accept new recover sessions: No
EOF
```

> **Preporuka za vault:** pre bilo kakve dijagnostike na sistemu koji nešto radi — prvo service mode, pa onda dijagnostika. Sprečava da vam neki zakazani job promeni stanje pod rukama.

> Za pojedinačne uređaje isti efekat daje `enabled: Service` (vidi `07b`, odeljak 49.4).

---

## 61.19 Reset NMC administratorske lozinke

### Postupak (NetWorker 19.1 i noviji) **[ADM]**

```bash
# 1. Odrediti Base64 vrednost nove lozinke
echo -n '<nova_lozinka>' | base64

# 2. Otvoriti šablon
vi /opt/nsr/authc-server/scripts/authc-local-config.json.template
```

> **[ADM]** U šablonu:
> - Zameniti promenljivu `username` imenom administratorskog naloga
> - Zameniti promenljivu `encoded_password` Base64 kodiranom vrednošću

```bash
# 3. Preimenovati
mv /opt/nsr/authc-server/scripts/authc-local-config.json.template \
   /opt/nsr/authc-server/scripts/authc-local-config.json

# 4. Kopirati u Tomcat conf folder
#    (tačna putanja zavisi od instalacije — proveriti)

# 5. Zaustaviti pa pokrenuti servise
systemctl stop networker && systemctl stop gst
systemctl start networker && systemctl start gst
```

---

## 61.20 Cyber Recovery specifični simptomi

| Simptom | Uzrok / rešenje | Izvor |
|---|---|---|
| **NetWorker aplikacija nije na listi u CR UI-u** | Za taj host već postoji sandbox. Proveriti **Recovery → Recovery Sandboxes** i očistiti. | **[CR]** |
| **`recoverapp_<ID>` job ne uspeva** | Proveriti UID DD Boost korisnika (`crcli policy list-copy` → `user show list`), verziju NetWorker-a, broj MTree-ova | **[CR]** |
| **Automatski recovery nije podržan** | NetWorker server ima **više od jednog MTree-a** — ručna procedura, kontaktirati Dell Support | **[CR]** |
| **Oporavak se prekinuo, sistem u nekonzistentnom stanju** | Vratiti `.cr.<timestamp>` direktorijume — `07c`, odeljak 59.1 | **[CR]** |
| **Sandbox ostao montiran** | `umount /opt/dellemc/cr/mnt/cr-rec-<sandbox>_1604` | **[CR]** |
| **Lažne greške o CFI oporavku fizičkih hostova u `nsrdr.log`** | **Ignorisati** — NetWorker ne backup-uje osnovni fizički host u virtuelnom okruženju | **[DR]** |
| **`jobsdb` prazan, svi workflow-ovi `Never Run`** | **Očekivano** — Server Protection politika ne backup-uje `jobsdb` | **[DR]** |
| **NMC ne prikazuje resurse posle `nsrdr`-a** | NMC ima zasebnu bazu — pokrenuti `recoverpsm` (`07c`, odeljak 57.6) | **[DR]** |
| **Media baza prazna posle naizgled uspešnog `nsrdr`-a** | **`server state` nije bio `disaster recovery`.** `nsrdr` ne prijavljuje grešku. Ponoviti proceduru. | **[CMD]** |

---

# 62. Prikupljanje podataka za Dell Support

## 62.1 Šta Dell traži **[ADM]**

> Pre kontaktiranja tehničke podrške obezbediti:

| # | Informacija | Kako dobiti na RHEL 9.6 |
|---|---|---|
| 1 | **Verzija softvera NetWorker komponente** | `rpm -qa \| grep -i lgto` |
| 2 | **Verzija operativnog sistema** | `cat /etc/redhat-release` ; `uname -a` |
| 3 | **Hardverska konfiguracija** | `lscpu` ; `free -h` ; `lsblk` |
| 4 | **Informacije o uređajima i SCSI ID-ovima** | **`/usr/sbin/inquire`** |
| 5 | Tip konekcije autochanger-a (SCSI ili RS-232) i verzija drajvera | `lsscsi -g` |
| 6 | **Kako reprodukovati problem** | opis |
| 7 | **Tačne poruke o grešci** | iz logova |
| 8 | **Koliko puta ste videli problem** | opis |
| 9 | **Da li je operacija radila pre izmena** i koje su izmene napravljene | opis |

> **[ADM]** Za AIX, Linux i Solaris tip: `/usr/sbin/inquire`. Za HP-UX: `/etc/ioscan`.

## 62.2 Skripta za prikupljanje

```bash
#!/bin/bash
# Prikupljanje dijagnostičkih podataka za Dell Support
OUT=/tmp/nw-diag-$(hostname -s)-$(date +%F-%H%M)
mkdir -p "$OUT"

# --- Sistem ---
cat /etc/redhat-release            > "$OUT/os-release.txt"
uname -a                          >> "$OUT/os-release.txt"
lscpu                              > "$OUT/hardware.txt"
free -h                           >> "$OUT/hardware.txt"
lsblk                             >> "$OUT/hardware.txt"
df -h                              > "$OUT/df.txt"
du -sh /nsr/* 2>/dev/null          > "$OUT/nsr-usage.txt"

# --- NetWorker verzije i stanje ---
rpm -qa | grep -i lgto             > "$OUT/packages.txt"
/etc/init.d/networker status       > "$OUT/service-status.txt" 2>&1
ps -ef | grep /usr/sbin/nsr        > "$OUT/processes.txt"
systemctl status networker         > "$OUT/systemd.txt" 2>&1

# --- Konfiguracija ---
nsradmin -s "$(hostname -f)" <<'EOF' > "$OUT/nsr-resource.txt" 2>&1
. type: NSR
p
EOF

nsradmin -s "$(hostname -f)" <<'EOF' > "$OUT/devices.txt" 2>&1
option hidden
. type: NSR device
p
EOF

# --- Storage i katalog ---
nsrmm -C                           > "$OUT/nsrmm-C.txt" 2>&1
mminfo -m                          > "$OUT/volumes.txt" 2>&1
mminfo -B                          > "$OUT/bootstrap.txt" 2>&1
nsrls                              > "$OUT/index-stats.txt" 2>&1

# --- Uređaji i SCSI ---
/usr/sbin/inquire                  > "$OUT/inquire.txt" 2>&1
lsscsi -g                          > "$OUT/lsscsi.txt" 2>&1

# --- Mreža ---
hostname -f                        > "$OUT/network.txt"
grep "^hosts:" /etc/nsswitch.conf >> "$OUT/network.txt"
cat /etc/hosts                    >> "$OUT/network.txt"
nsrports                          >> "$OUT/network.txt" 2>&1
ss -tlnp                          >> "$OUT/network.txt"
firewall-cmd --list-all           >> "$OUT/network.txt" 2>&1

# --- Vreme ---
chronyc tracking                   > "$OUT/time.txt" 2>&1
timedatectl                       >> "$OUT/time.txt"

# --- SELinux ---
getenforce                         > "$OUT/selinux.txt"
ausearch -m AVC,USER_AVC -ts today >> "$OUT/selinux.txt" 2>&1

# --- Logovi ---
nsr_render_log /nsr/logs/daemon.raw > "$OUT/daemon.log" 2>&1
cp /nsr/logs/nsrdr.log             "$OUT/" 2>/dev/null
cp /nsr/logs/index.log             "$OUT/" 2>/dev/null
cp /nsr/logs/media.log             "$OUT/" 2>/dev/null
cp /nsr/logs/rap.log               "$OUT/" 2>/dev/null
cp /nsr/logs/policy_notifications.log "$OUT/" 2>/dev/null
tar czf "$OUT/policy-logs.tar.gz" /nsr/logs/policy/ 2>/dev/null
journalctl -u networker --since "24 hours ago" > "$OUT/journal.txt" 2>&1

# --- Tragovi ranijih problema ---
ls -ld /nsr /nsr/res* /nsr/mm* /nsr/index* > "$OUT/nsr-dirs.txt" 2>&1
find /nsr/cores -type f 2>/dev/null        > "$OUT/cores.txt"
find /nsr/index -name "*.sip" 2>/dev/null  > "$OUT/sip-files.txt"
ls -l /nsr/debug/ 2>/dev/null              > "$OUT/debug-dir.txt"

tar czf "${OUT}.tar.gz" -C /tmp "$(basename "$OUT")"
echo "Gotovo: ${OUT}.tar.gz"
```

## 62.3 Ako je problem vezan za Cyber Recovery

Dodatno priložiti:

```bash
# Sa Cyber Recovery management hosta
./crsetup.sh --check
```

- CR support bundle: **System Settings → Support → Support Bundles → Generate Log Bundle**
- DD kolekcije iz PowerProtect DD Management Center-a
- Izlaz `ddboost storage-unit show`, `mtree list`, `replication status` sa vault DD sistema
- Ime i status neuspelog CR job-a (`recoverapp_<ID>`)

## 62.4 Prikupljanje sa povećanim nivoom logovanja

Kada standardni logovi nisu dovoljni:

```bash
# 1. Podići debug nivo bez restarta
dbgcommand -n nsrd Debug=9
dbgcommand -n nsrd MsgID=1

# 2. Reprodukovati problem

# 3. Prikupiti log
nsr_render_log /nsr/logs/daemon.raw > /tmp/daemon-debug.log

# 4. OBAVEZNO vratiti na normalu
dbgcommand -n nsrd Debug=0
dbgcommand -n nsrd MsgID=0
```

> **Message ID je koristan** — omogućava traženje tačne greške u *NetWorker Error Message Guide*-u.

---

# 63. Brza referenca

## 63.1 Kartica komandi — jedna strana

### Servisi

```bash
systemctl start|stop|status networker      # RHEL 9.6
/etc/init.d/networker status               # stablo procesa
nsr_shutdown                               # kontrolisano gašenje
systemctl start|stop gst                   # NMC
ps -ef | grep /usr/sbin/nsr
```

### Stanje i konfiguracija

```bash
nsradmin -s <server>                       # interaktivno
nsradmin -c "type:NSR device"              # puni ekran
cd /nsr/res && nsradmin -d nsrdb           # server ne radi
nsradmin -p nsrexec                        # NSRLA baza (logovi, portovi)
nsrports                                   # opsezi portova
```

### Storage

```bash
nsrmm -C                                   # šta je montirano
nsrmm -m|-u|-j|-p -f <device>              # mount/unmount/eject/verify
nsrmm -o notscan|notreadonly <volume>
nsrmm -g <volume>                          # ukloni replica zastavicu
mminfo -m                                  # svi volumeni
```

### Katalog

```bash
mminfo -B                                  # bootstrap-ovi
mminfo -a -v                               # svi save set-ovi
mminfo -p                                  # browse/retention vremena
nsrls [<klijent>]                          # statistika indeksa
nsrinfo [-L] <klijent>                     # sadržaj indeksa
nsrim -n -v                                # pregled bez izmena
nsrim -C                                   # smanjenje media baze (DUGO!)
```

### Skeniranje

```bash
scanner -n -i <device>                     # provera BEZ izmena
scanner -B <device>                        # pronalaženje bootstrap ssid-a
scanner -m <device>                        # media baza
scanner -i <device>                        # media baza + indeksi
```

### Oporavak

```bash
nsrdr                                      # interaktivno
nsrdr -a -B <ssid> -d <device> -I
nsrck -L2 <klijent>                        # kreiranje CFI
nsrck -L5 <klijent>                        # čišćenje oštećenja
nsrck -L7 -t "<datum>" <klijent>           # vraćanje sa medija
recoverpsm -s <nw> -c <nmc> <staging>      # NMC baza
recover                                    # oporavak fajlova
```

### Dijagnostika

```bash
nsr_render_log /nsr/logs/daemon.raw | tail -100
dbgcommand -n nsrd Debug=9                 # bez restarta
dbgcommand -n nsrd PrintDnsCache
dbgcommand -n nsrd FlushDnsCache
dbgcommand -n nsrd PrintDevInfo
dbgcommand -n nsrd Debug=0                 # vratiti!
/usr/sbin/inquire                          # SCSI uređaji
nsrwatch                                   # praćenje u realnom vremenu
```

## 63.2 Redosled koraka za oporavak — jedna strana

```
┌─ PRIPREMA ─────────────────────────────────────────────┐
│ 1. Verzija i paketi                rpm -qa | grep lgto │
│ 2. /nsr simbolički link?           ls -ld /nsr         │
│ 3. FQDN = hostname                 hostname -f         │
│ 4. Rezolucija imena                hosts: files        │
│ 5. id nsrtomcat = kao produkcija                       │
│ 6. authcdb.h2.db.<ts> ne sme biti noviji od bootstrap-a│
│ 7. ★ server state: disaster recovery ★                 │
│    (bez ovoga media baza se NE uvozi, BEZ GREŠKE)      │
└────────────────────────────────────────────────────────┘
                          ↓
┌─ STORAGE ──────────────────────────────────────────────┐
│ 8.  ddboost storage-unit show          (na DD)         │
│ 9.  ddboost storage-unit modify ... user ...           │
│ 10. Kreirati NSR device (SMT ako treba)                │
│     ★ NIKAD ne označavati volumen (label) ★            │
│ 11. read only: yes  PRE montiranja                     │
│ 12. nsrmm -C                           provera         │
└────────────────────────────────────────────────────────┘
                          ↓
┌─ BOOTSTRAP ────────────────────────────────────────────┐
│ 13. policy_notifications.log     ili                   │
│     mminfo -B                    ili                   │
│     scanner -B <device>                                │
│     → ssid (4. kolona), file (5.), record (6.)         │
└────────────────────────────────────────────────────────┘
                          ↓
┌─ NSRDR ────────────────────────────────────────────────┐
│ 14. Demontirati SVE volumene                           │
│ 15. CDI ako ima trake, pa restart servisa              │
│ 16. /nsr/debug/nsrdr.conf ako ima mnogo klijenata      │
│ 17. nsrdr        (-N ako ima trake)                    │
│ 18. authc_configure.sh                                 │
└────────────────────────────────────────────────────────┘
                          ↓
┌─ POSLE ────────────────────────────────────────────────┐
│ 19. Provera resursa: Protection / Devices / Media      │
│ 20. Retention policy klijenta → Decade                 │
│ 21. Aliases: shortname + FQDN                          │
│ 22. scanner -m + nsrck -L7   (ako ima index backup-a)  │
│     scanner -i               (ako nema)                │
│ 23. nsrmm -o notscan <volume>                          │
│ 24. recoverpsm               (NMC baza)                │
│ 25. ★ server state: active ★                           │
│ 26. Licenca — 14 dana!                                 │
└────────────────────────────────────────────────────────┘
```

## 63.3 Putanje i logovi — jedna strana

### Baze

| Putanja | Sadržaj |
|---|---|
| `/nsr/res` | Resource (RAP) baza; `nsrdb` poddirektorijum; lockbox |
| `/nsr/mm` | Media baza (`mmvolume6` legacy / `mmvolrel` relacioni) |
| `/nsr/index/<klijent>/db6` | Client file indeksi |
| `/nsr/lic` | `dpa.lic`, `licspec.properties` |

### Logovi

| Putanja | Sadržaj |
|---|---|
| `/nsr/logs/daemon.raw` | **Glavni log** — `nsr_render_log` |
| `/nsr/logs/nsrdr.log` | DR čarobnjak; **prepisuje se pri svakom pokretanju** |
| `/nsr/logs/index.log` | Veličina indeksa, malo prostora |
| `/nsr/logs/media.log` | Poruke vezane za uređaje |
| `/nsr/logs/rap.log` | **Izmene konfiguracije** |
| `/nsr/logs/policy_notifications.log` | Izveštaji politika, **Server backup Action report** |
| `/nsr/logs/policy/<pol>/<wf>_<jobid>.raw` | Log workflow-a |
| `/nsr/logs/NetWorker_server_sec_audit.raw` | Bezbednosna revizija |
| `/opt/lgtonmc/management/logs/gstd.raw` | NMC |

### Debug i marker fajlovi

| Putanja | Namena |
|---|---|
| `/nsr/debug/nsrdr.conf` | `NSRDR_NUM_THREADS`, `NSRDR_SERVICES_PATH` |
| `/nsr/debug/nsr_disaster_recovery_mode` | Marker; **obrisati ako je zaostao** |
| `/nsr/debug/cdidisable` | Globalno onemogućavanje CDI |
| `/nsr/nsrrc` | Promenljive okruženja; `NSR_SERVER_STATE` |
| `/opt/nsr/admin/networkerrc` | Startno okruženje |
| `/opt/nsr/admin/nsr_serverrc` | Serversko startno okruženje |

### Tragovi ranijih oporavaka

| Obrazac | Značenje |
|---|---|
| `/nsr/res.<timestamp>` | Prethodna resource baza (ručni `nsrdr`) |
| `/nsr/res.R` | Privremeni folder tokom `nsrdr`-a |
| `/nsr/res.cr.<timestamp>` | Prethodna baza (Cyber Recovery oporavak) |
| `/nsr/mm.cr.<timestamp>` | Prethodna media baza (CR) |
| `/nsr/index.cr.<timestamp>` | Prethodni indeksi (CR) |
| `authcdb.h2.db.<timestamp>` | Zamenjena auth baza |
| `*.sip` u `db6` | **Prekinut save** |
| `tmprecov`, `recovered` u `db6` | **`nsrck` nije završen** |

### Portovi

| Port | Namena |
|---|---|
| **7937–9936** | Service ports (podrazumevani opseg) |
| **7938** | Test konekcije (`nsrports -t`) |
| **9090** | NetWorker Authentication Service |
| **9000** | NMC (GST) |
| **390103 / 390113** | RPC program brojevi (`nsrd` / `nsrexecd`) |

## 63.4 Deset stvari koje se najlakše promaše

| # | Stavka | Posledica |
|---|---|---|
| 1 | **`server state` nije `disaster recovery`** | Media baza se ne uvozi, **bez poruke o grešci** |
| 2 | Retention policy klijenta ostao na mesec dana | Save set-ovi se tiho odbacuju |
| 3 | `nsrtomcat` UID različit od produkcije | `nsrdr` ka drugom serveru ne radi |
| 4 | `/nsr` je simbolički link koji nije rekreiran | Oporavak ide na pogrešnu lokaciju |
| 5 | Volumen ponovo označen (`label`) | **Podaci trajno neoporavljivi** |
| 6 | `scanner -i` bez postojećeg CFI | `File index error, file index is missing` |
| 7 | `nsrck -L7` bez prethodnog `-L5` nad oštećenim indeksom | Oštećenje ostaje |
| 8 | Zaostao `nsr_disaster_recovery_mode` fajl | DNS keš se ne popunjava |
| 9 | `server state` ostao `disaster recovery` posle oporavka | Backup i workflow **ne rade** |
| 10 | Licenca nije prijavljena u roku od 14 dana | Server onemogućen, samo oporavci |

---

*Kraj Dela VII-d. Ovim je kompletiran Deo VII. Videti i: `07a` (anatomija), `07b` (storage i katalog), `07c` (kompletan tok oporavka).*
