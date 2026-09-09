# DEO VII-a — ANATOMIJA NETWORKER INSTALACIJE NA RHEL 9.6

**Dell PowerProtect Cyber Recovery 20.3 — Uputstvo za instalaciju i integraciju**
Verzija dokumenta: 0.1 | **Namena: sistem administratori i System Engineer-i**
**Referentni OS:** Red Hat Enterprise Linux 9.6 | **Referentni NetWorker:** 19.11 / 19.13

> **Izvori:**
> - *Dell NetWorker 19.11 Command Reference Guide* — **[CMD]**
> - *Dell NetWorker 19.13 Administration Guide* — **[ADM]**
> - *Dell NetWorker Server Disaster Recovery and Availability Best Practices Guide 19.13* — **[DR]**
> - *Dell PowerProtect Cyber Recovery 20.3 Product Guide / Installation and Upgrade Guide* — **[CR]**
> - **[?]** — opšte znanje, potvrditi na sistemu

> **Namena ovog dela:** dati sistem administratoru mentalni model NetWorker-a iz ugla Linux-a — šta je gde na disku, koji procesi rade i zašto, kako se servis pokreće i gasi, gde su logovi. Bez ovoga je poglavlje o oporavku (`07c`) niz komandi bez konteksta.

---

## Sadržaj

- [39. NetWorker iz ugla Linux administratora](#39-networker-iz-ugla-linux-administratora)
- [40. Struktura fajl sistema](#40-struktura-fajl-sistema)
- [41. Paketi i instalacija](#41-paketi-i-instalacija)
- [42. Servisi i procesi](#42-servisi-i-procesi)
- [43. Mreža i portovi](#43-mreža-i-portovi)
- [44. Promenljive okruženja](#44-promenljive-okruženja)
- [45. Logovi](#45-logovi)
- [46. SELinux i firewalld na RHEL 9.6](#46-selinux-i-firewalld-na-rhel-96)
- [47. Provera zdravlja instance u 60 sekundi](#47-provera-zdravlja-instance-u-60-sekundi)

---

# 39. NetWorker iz ugla Linux administratora

## 39.1 Šta je NetWorker zapravo

Za sistem administratora navikaog na servise koji imaju konfiguraciju u `/etc` i podatke u `/var`, NetWorker je neobičan: **konfiguracija i stanje žive u istom stablu**, `/nsr`, i menjaju se konstantno dok sistem radi.

NetWorker se sastoji iz **četiri baze**, koje treba razlikovati jer se pri oporavku ponašaju različito:

| Baza | Lokacija | Sadržaj | U bootstrap-u |
|---|---|---|---|
| **Resource (RAP) baza** | `/nsr/res` | Konfiguracija: klijenti, uređaji, pool-ovi, politike, sam server | **Da** |
| **Media baza** | `/nsr/mm` | Gde se koji save set fizički nalazi, na kom volumenu | **Da** |
| **Client file indeksi (CFI)** | `/nsr/index` | Koji fajlovi su u kom save set-u, po klijentu | **Zaseban save set** |
| **Job baza (`jobsdb`)** | interno | Istorija izvršavanja workflow-ova i akcija | **Ne** |

> **Praktična posledica:** posle disaster recovery-ja `jobsdb` je prazan i svi workflow-ovi imaju status `Never Run`. To nije greška — Server Protection politika ga jednostavno ne backup-uje. **[DR]**

## 39.2 Zašto je bootstrap jedini konzistentan snimak

Klasičan refleks administratora je: „snimiću `/nsr` rsync-om ili snapshot-om diska". To ne radi.

> **[DR]** Jedan od izazova pri zaštiti produkcionog backup servera je **mogućnost da se sistem uhvati u konzistentnom stanju**. Dok se izvršavaju backup i recovery operacije, stanje servera i konfiguracioni fajlovi su u **stalnom procesu izmene**. Repliciranje konfiguracionih fajlova je moguće, ali operacija može dati **crash-consistent** stanje. **Bootstrap backup je jedini metod koji obezbeđuje da se podaci mogu oporaviti.**

> **[DR]** Replikacija, mirroring i snapshot operacije zahtevaju presretanje i hvatanje svih traženih read, write i change I/O operacija. Write I/O zahteva dodatnu obradu, ne samo za ažuriranje diska nego i za potvrdu da su ažuriranja uspela.

Šta NetWorker radi na disku dok radi **[DR]**:

- Log fajlovi se ažuriraju događajima i greškama
- Client file indeksi se ažuriraju novim backup-ima ili uklanjanjem onih kojima je istekla politika
- Media baza se ažurira lokacijom i stanjem svakog korišćenog volumena
- Save set informacije se kreiraju, brišu i menjaju
- Opšta konfiguracija se ažurira tekućim stanjem servera sa svim storage node-ovima, uređajima i klijentima

> **[DR]** Ove aktivnosti zahtevaju **veliki broj I/O operacija** na disku servera. Svaki uticaj na brzinu i pouzdanost I/O operacija utiče na performanse i pouzdanost NetWorker servera i na disaster recovery.

## 39.3 Šta je bootstrap konkretno **[DR]**

Bootstrap save set sadrži pet komponenti koje se nalaze na NetWorker serveru:

| Komponenta | Lokacija |
|---|---|
| Media database | `/nsr/mm` |
| Resource files | `/nsr/res` |
| License server files (`dpa.lic`, `licspec.properties`) | `/nsr/lic` |
| NetWorker Authentication Service database (`authcdb.h2.db`) | direktorijum auth servisa |
| **Lockboxes** | `/nsr/res` (lockbox folder) |

> **[DR]** Lockbox folder u resource direktorijumu čuva poverljive informacije — na primer Oracle klijentske lozinke i **DD Boost lozinku** — u **enkriptovanom formatu**. NetWorker te informacije koristi za backup i recovery operacije.

Ovo je bitno razumeti: **DD Boost lozinka nije u konfiguracionom fajlu koji možete pročitati**. Nalazi se u lockbox-u i dolazi nazad samo kroz oporavak bootstrap-a.

## 39.4 Preporuke za disk layout **[DR]**

> Da bi se poboljšale brzina, pouzdanost, skalabilnost i performanse backup servera:
>
> - **Držati ključne konfiguracione informacije i podatke indeksa na odvojenim LUN-ovima**, radi eliminisanja problema sa oštećenjem OS-a i poboljšanja opštih performansi sistema
> - **Hostovati LUN-ove na RAID-zaštićenim ili eksternim storage sistemima**
> - Obezbediti odgovarajuću količinu prostora
> - Obezbediti da je storage zaštićen i da radi na optimalnom nivou
> - Razmotriti napredne tehnologije zaštite kao što su replikacija ili snapshot-ovi

**Praktično na RHEL 9.6:**

```bash
# Predlog razdvajanja
/nsr/index    → zaseban LVM volumen (raste sa brojem backup-ovanih fajlova)
/nsr/mm       → zaseban volumen
/nsr/res      → mali, ali kritičan
/nsr/logs     → zaseban volumen (sprečava da logovi popune ostalo)
```

> **[CMD]** Nema praktičnih ograničenja maksimalne veličine online indeksa, **osim da mora u celini stati u jedan fajl sistem.**

---

# 40. Struktura fajl sistema

## 40.1 Kanonska mapa `/nsr` **[CMD]**

> NetWorker server fajl sistem ima direktorijum `/nsr` koji sadrži log fajlove, online indekse i konfiguracione informacije. **Ovaj direktorijum se može kreirati u bilo kom fajl sistemu, sa `/nsr` postavljenim kao simbolički link na stvarni direktorijum** — to se određuje pri instalaciji.

| Putanja | Sadržaj **[CMD]** |
|---|---|
| **`/nsr/logs`** | Poruke logovanja servera. Fajlovi su u **UTF8** formatu (uglavnom ASCII, podskup UTF8). |
| **`/nsr/res`** | Konfiguracioni fajlovi raznih komponenti NetWorker servera. Server čuva konfiguracione fajlove u **`/nsr/res/nsrdb`**. |
| **`/nsr/mm`** | Media indeks. Informacije o sadržaju se ispisuju komandom `nsrls`. Za pregled i izmenu koristiti `nsrmm` i `mminfo`. |
| **`/nsr/index`** | Poddirektorijumi sa imenima koja odgovaraju NetWorker klijentima koji su snimali fajlove. |
| **`/nsr/cores`** | Direktorijumi koji odgovaraju NetWorker demonima i pojedinim izvršnim fajlovima. Svaki može sadržati **core fajlove** procesa koji su abnormalno prekinuti. |
| **`/nsr/drivers`** | Može sadržati drajvere uređaja za upotrebu sa NetWorker-om. |
| **`/nsr/tmp`** | Privremeni fajlovi koje koristi NetWorker sistem. |
| **`/nsr/debug`** | Debug fajlovi i konfiguracije alata (`nsrdr.conf`, `cdidisable`, `nsr_disaster_recovery_mode`) |
| **`/nsr/lic`** | Licencni fajlovi (`dpa.lic`, `licspec.properties`) **[DR]** |
| **`/nsr/nsrrc`** | Bourne shell skripta sa promenljivama okruženja, učitava se pre starta procesa **[ADM]** |

> **Napomena o `/nsr/lic`:** *DR Best Practices Guide* na jednom mestu piše `/nrs/lic`. To je gotovo sigurno štamparska greška. Proveriti na sistemu:
> ```bash
> ls -l /nsr/lic/
> ```

## 40.2 Provera da li je `/nsr` simbolički link

```bash
ls -ld /nsr
```

> **[CMD]** `/nsr` — ako je ovo bio simbolički link u trenutku kreiranja bootstrap save set-a, **morate ponovo kreirati simbolički link pre pokretanja `nsrdr` komande.**

> **[DR]** Simbolički linkovi ka bootstrap save set-ovima **ne oporavljaju se u podrazumevani direktorijum**, već u ciljni direktorijum simboličkog linka. Ako je `/nsr/res` povezan sa `/bigres/res`, resource baza se oporavlja u `/bigres/res`. **Obezbediti dovoljno slobodnog prostora u ciljnom direktorijumu.**

## 40.3 Unutrašnjost `/nsr/index` — `db6` **[CMD]**

Svaki klijentski index direktorijum sadrži fajlove koji omogućavaju NetWorker serveru da obezbedi online bazu snimljenih fajlova klijenta. **Najvažniji element je `db6` direktorijum**, koji sadrži NetWorker save zapise i pristupne indekse tim zapisima.

> **Sizing:** administratori treba da planiraju oko **200 bajtova po instanci snimljenog fajla** koja se smešta u indeks.

Format `db6` direktorijuma je podložan promenama i dostupan je **samo kroz RPC interfejs ka `nsrindexd`**. Komanda `nsrls` daje korisne statistike iz tog direktorijuma.

### Fajlovi u `db6` **[CMD]**

> **Ovi fajlovi su za internu upotrebu servera i ne smeju se menjati.** Navedeni su samo radi prepoznavanja pri dijagnostici.

| Fajl | Sadržaj |
|---|---|
| `<savetime>.rec` | Zapisi indeksa za svaki fajl snimljen u tom savetime; `<savetime>` je heksadecimalna reprezentacija vremena |
| `<savetime>.k0` | Ključevi nad `.rec` fajlom **po imenu fajla** |
| `<savetime>.k1` | Ključevi nad `.rec` fajlom **po inode-u**. Mogu biti nulte dužine ako je CFI za Windows klijenta. |
| `<savetime>.sip` | **Save-in-process** fajl — postoji samo dok save traje. Po završetku se preimenuje u `<savetime>.rec`. |
| `v6hdr` | Sažetak svih `.rec` fajlova koji postoje u `db6` direktorijumu klijenta |
| `v6journal` | Ažuriranja `v6hdr` fajla koja čekaju spajanje. Svaka index operacija uključuje i ove i one iz `v6hdr`. |
| `v6ck.lck` | Lock koji `nsrck` koristi da obezbedi da samo jedan `nsrck` radi nad indeksom klijenta u datom trenutku |
| `v6hdr.lck` | Zaključava `v6hdr` za čitanje i `v6journal` za čitanje i pisanje |
| `v6tmp.ptr` | Pokazuje na radni direktorijum u kojem se odvija konverzija i oporavak |
| `tmprecov` | Radni direktorijum za konverziju i oporavak |
| `recovered` | Međurezultati konvertovanog ili oporavljenog indeksa. Rezultati su kompletni i biće integrisani u file indeks kada se pokrene `nsrck` nad tim klijentom. |

> **Dijagnostička vrednost:** prisustvo `.sip` fajla znači da je save prekinut u toku. Prisustvo `tmprecov` ili `recovered` znači da je `nsrck` bio u toku i nije završen.

```bash
# Pregled indeksa jednog klijenta
ls -la /nsr/index/<ime_klijenta>/db6/ | head -30

# Da li ima zaostalih save-in-process fajlova
find /nsr/index -name "*.sip" -exec ls -l {} \;

# Statistika indeksa
nsrls <ime_klijenta>
```

## 40.4 Prenosivost `db6` fajlova **[CMD]**

> Podaci u `db6` fajlovima čuvaju se u **platformski nezavisnom redosledu**, pa se ti fajlovi mogu migrirati sa jednog NetWorker servera na drugi.
>
> **Premeštanje media baze** sa jednog NetWorker servera na drugi **nesrodne arhitekture trenutno nije podržano.**

## 40.5 Media baza — legacy vs. relacioni format **[CMD]**

| Putanja | Značenje |
|---|---|
| `/nsr/mm/mmvolume6` | Media baza u **legacy** formatu. Aktivna baza **pre** migracije ili posle **neuspele** migracije. |
| `/nsr/mm/mmvolrel` | Media baza u **relacionom** formatu. Aktivna baza **posle** migracije ili kod **nove instalacije**. |

```bash
ls -ld /nsr/mm/mmvolume6 /nsr/mm/mmvolrel 2>/dev/null
```

> **Dijagnostika:** prisustvo `mmvolume6` uz odsustvo `mmvolrel` na sistemu koji bi trebalo da je migriran ukazuje da migracija nije izvršena ili nije uspela.

## 40.6 Gde su izvršni fajlovi **[CMD]**

| Putanja | Sadržaj |
|---|---|
| `/usr/sbin` | Glavni NetWorker izvršni fajlovi na Linux-u (`nsrd`, `nsrexecd`, `nsrctld`...) |
| `/usr/bin` | Deo korisničkih komandi |
| `/usr/lib/nsr` | Pomoćni izvršni fajlovi |
| `/opt/nsr` | Instalaciono stablo (uključujući `authc-server`, `admin`) |
| `/opt/nsr/admin/networkerrc` | Startna skripta okruženja **[ADM]** |
| `/opt/nsr/admin/nsr_serverrc` | Startna skripta serverskog okruženja **[ADM]** |
| `/opt/nsr/authc-server/scripts/authc_configure.sh` | Konfiguracija Authentication Service-a **[DR]** |
| `/opt/lgtonmc` | NMC instalacija **[ADM]** |

```bash
ls /usr/sbin/nsr*
ls /opt/nsr/
which recover nsradmin mminfo nsrdr
```

## 40.7 Praćenje zauzeća

```bash
# Ukupno po komponenti
du -sh /nsr/index /nsr/mm /nsr/res /nsr/logs /nsr/tmp /nsr/cores 2>/dev/null

# Najveći indeksi po klijentu
du -sh /nsr/index/* 2>/dev/null | sort -h | tail -20

# Slobodan prostor
df -h /nsr

# Da li ima core fajlova
find /nsr/cores -type f -name "core*" -exec ls -lh {} \; 2>/dev/null
```

> **[ADM]** Log fajl `/nsr/logs/index.log` sadrži **upozorenja o veličini client file indeksa i malom slobodnom prostoru** na fajl sistemu koji sadrži index fajlove. Podrazumevano *Index size* notifikacija na serveru šalje informacije u taj fajl.

```bash
tail -50 /nsr/logs/index.log
```

---

# 41. Paketi i instalacija

## 41.1 Koji paketi su potrebni

> **[DR]** Pri ponovnoj instalaciji NetWorker Server softvera: *ensure that you install the NetWorker **client**, **storage node**, and **Authentication service** packages.*

| Paket **[?]** | Uloga |
|---|---|
| `lgtoclnt` | NetWorker client — obavezan, osnova svega |
| `lgtonode` | Storage node — rad sa uređajima |
| `lgtoserv` | NetWorker server |
| `lgtoauthc` | Authentication Service |
| `lgtoxtdclnt` | Extended client — dodatne komande **[ADM]** |
| `lgtonmc` | NMC (Management Console) — opciono |
| `lgtoman` | **Man stranice** — opciono, ali preporučeno **[CMD]** |

> **[?]** Tačna imena paketa i njihove međuzavisnosti nisu dokumentovani u priloženim vodičima — potvrditi uz *NetWorker Installation Guide* za konkretnu verziju. `lgtoclnt` i `lgtoxtdclnt` su potvrđeni u **[ADM]**, `lgtoman` u **[CMD]**.

```bash
# Šta je instalirano
rpm -qa | grep -i lgto

# Detalji paketa
rpm -qi lgtoserv

# Koji paket vlasnik komande
rpm -qf $(which nsrdr)

# Fajlovi koje je paket instalirao
rpm -ql lgtoserv | head -40
```

## 41.2 Man stranice — instalirati u vault-u

> **[CMD]** Informacije iz *Command Reference Guide*-a dostupne su i iz komandne linije na svim platformama osim Windows-a:
> ```bash
> man <ime_komande>
> ```
> **Za to mora biti instaliran opcioni paket `LGTOman`**, a putanja do man stranica mora biti u `MANPATH` promenljivoj — u suprotnom `man` treba pokretati iz instalacione lokacije man stranica.

> **Preporuka za vault:** instalirati `lgtoman`. U izolovanom okruženju bez pristupa internetu i Dell portalu, man stranice su **jedini priručnik na licu mesta**.

```bash
rpm -qa | grep -i lgtoman
man -w recover
echo $MANPATH
```

Osnovna pomoć radi i bez paketa **[CMD]**:

```bash
recover -help
nsrdr -help
```

> **[CMD]** Osnovna pomoć je ograničena na listu argumenata, opcija i parametara komande.

## 41.3 Nadogradnja

> **[CR]** Za Linux: preuzeti željenu `tar.gz` verziju NetWorker softvera, raspakovati je, pa pokrenuti `rpm -U` za nadogradnju NetWorker binarnih fajlova.

```bash
tar -xzvf <networker_paket>.tar.gz
cd <direktorijum>
rpm -U lgto*.rpm
```

> **[CR] Uticaj na Cyber Recovery:** nadogradnja je disruptivna za NetWorker server — servisi se zaustavljaju i pokreću. Cyber Recovery softver nastavlja da radi uz ograničenu smetnju, ali **automatizovani NetWorker recovery proces ne radi tokom nadogradnje.**

## 41.4 Šta radi `authc_configure.sh` **[DR]**

Skripta konfiguriše NetWorker Authentication Service posle instalacije ili posle oporavka.

```bash
/opt/nsr/authc-server/scripts/authc_configure.sh
```

Pokreće se:
1. Posle ponovne instalacije NetWorker Server softvera
2. **Posle završenog `nsrdr` oporavka**

## 41.5 Konfiguracija na istu lokaciju

> **[CMD] IMPORTANT:** pre upotrebe `nsrdr`-a, NetWorker mora biti **potpuno instaliran i ispravno konfigurisan** na hostu. Ako hostu nedostaje bilo koji NetWorker binarni fajl, ponovo instalirajte softver iz distribucionih fajlova. Koristiti **isto izdanje** NetWorker softvera i instalirati NetWorker na **originalnu lokaciju**.

> **[DR]** Na Linux-u nije potrebno ponovo učitavati license enabler-e ako NetWorker konfiguracioni fajlovi postoje. Podrazumevano se nalaze u **`/nsr/res/nsrdb`**.

---

# 42. Servisi i procesi

## 42.1 Hijerarhija — `nsrctld` je vrh

> **[ADM]** `nsrctld` je **glavni proces NetWorker servera**. To je proces najvišeg nivoa koji **prati, zaustavlja i pokreće sve NetWorker server procese.**

Za sistem administratora ovo je ključno: ne gasite pojedinačne `nsr*` procese. Gasite `nsrctld`, odnosno servis.

## 42.2 Procesi NetWorker servera **[ADM]**

| Proces | Uloga |
|---|---|
| **`nsrctld`** | Proces najvišeg nivoa; prati, zaustavlja i pokreće sve ostale NetWorker server procese |
| **`nsrd`** | NetWorker save i recovery demon. Glavni servis koji kontroliše ostale servise na serveru, klijentima i storage node-ovima. Prati aktivne save i recover sesije. Kao odgovor na recover sesiju spawn-uje agent proces **`ansrd`**. |
| **`nsrmmdbd`** | Servis media management baze. Pruža usluge upravljanja media bazom lokalnim `nsrd` i `nsrmmd` servisima i beleži zapise u media bazu. |
| **`nsrjobd`** | Prati NetWorker aktivnost tokom backup ili recovery operacije. |
| **`nsrindexd`** | Indeksni servis za čitanje, pisanje i uklanjanje zapisa indeksa. `nsrd` pokreće **jedan** `nsrindexd` proces; taj proces spawn-uje **dodatni helper `nsrindexd` za svaku index sesiju**. Kada se read/write operacija završi, helper se gasi. |
| **`nsrmmgd`** | Upravlja operacijama tape library-ja. RPC servis koji upravlja svim jukebox operacijama u ime `nsrd`. `nsrd` pokreće **samo jednu instancu** po potrebi. |
| **`nsrlogd`** | Podržava NetWorker audit log servis, podrazumevano konfigurisan da radi na serveru. |
| **`nsrcpd`** | Startuje automatski kada korisnik pristupi Hosts Task prozoru. Omogućava distribuciju i nadogradnju NetWorker i modul softvera iz centralizovanog repozitorijuma. |
| **`nsrdispd`** | Obrađuje RPC pozive za `nsrd` proces, od udaljenih procesa trećih strana. |
| **`nsrdisp_nwbg`** | Pokreće ga `nsrdispd`; obrađuje NMC zahteve za informacije iz RAP i media baza. |
| **`nsrlmc`** | Podržava licencne zahteve. Za tradicionalni licencni model traži licencu od `lgtolmd`; za CLP/ELMS model traži capacity i update licence od ELMS servera. |
| **`nsrvmwsd`** | Web servis za upravljanje VMware VM backup-ima. |
| **`tomcat`** | Tomcat web server instanca za NetWorker Authentication Service. |
| **`nsrexecd`** | Autentifikuje i obrađuje zahteve za udaljeno izvršavanje; pokreće `save` i `savefs` programe na klijentu. |

## 42.3 Procesi storage node-a **[ADM]**

| Proces | Uloga |
|---|---|
| **`nsrmmd`** | Obezbeđuje podršku uređajima, generiše mount zahteve, multipleksira save set podatke tokom backup-a više klijenata i demultipleksira recover podatke. Piše podatke koje šalje `save` na medij. Prosleđuje storage informacije `nsrmmdbd` procesu na serveru. |
| **`nsrsnmd`** | RPC servis koji upravlja svim operacijama nad uređajima koje `nsrmmd` obavlja u ime `nsrd`. Automatski ga pokreće `nsrd` po potrebi. **Samo jedan `nsrsnmd` radi na svakom storage node-u koji ima konfigurisane i omogućene uređaje.** |
| **`nsrlcpd`** | Uniforman library interfejs ka `nsrmmgd`. Upravlja medijima, slotovima, drajvovima i portovima library podsistema. **Jedan `nsrlcpd` se pokreće za svaki konfigurisani tape library.** |
| **`nsrexecd`** | Isto kao na serveru |

## 42.4 Procesi klijenta i NMC-a **[ADM]**

**Klijent:** `nsrexecd` — autentifikuje i upravlja zahtevima za udaljeno izvršavanje sa NetWorker servera i pokreće `save` i `savefs` procese na klijentu.

**NMC server:**

| Proces | Uloga |
|---|---|
| `gstd` | Generic Services Toolkit (GST); kontroliše ostale servise NMC servera |
| `httpd` | Pokreće NMC Console GUI kroz web pretraživač |
| `postgres` | Baza koja upravlja informacijama o NMC upravljanju, npr. Console izveštajima |
| `gstsnmptrapd` | Prati SNMP Trap-ove na upravljanom Data Domain sistemu. **Pokreće se samo ako je SNMP Trap monitoring konfigurisan.** |
| `nsrexecd` | Isto kao drugde |

## 42.5 Kako izgleda zdrav server

> **[ADM]** Za NetWorker server startuje se `nsrctld` demon, koji pokreće ostale procese koje server zahteva.

```bash
/etc/init.d/networker status
```

Očekivano stablo **[ADM]**:

```
+--o nsrctld (29021)
   +--o epmd (29029)
   +--o rabbitmq-server (29034)
   +--o beam (29038)
   +--o inet_gethost (29144)
   +--o inet_gethost (29145)
   +--o jsvc (29108)
   +--o jsvc (29114)
   +--o nsrd (29123)
   +--o java (29135)
   +--o nsrmmdbd (29828)
   +--o nsrindexd (29842)
   +--o nsrdispd (29853)
   +--o nsrjobd (29860)
   +--o nsrvmwsd (29968)
   +--o eventservice.ru (29154)
   +--o jsvc (29158)
   +--o jsvc (29159)
   +--o java (29838)
   +--o node-linux-x64- (29885)
   +--o nsrexecd (29004)
   +--o nsrlogd (29899)
   +--o nsrsnmd (30038)
```

> **Za administratora vredi primetiti:** u stablu su i `rabbitmq-server`, `beam` (Erlang VM), `jsvc` i `java` procesi. NetWorker 19.x nije samo skup C demona — ima Erlang message broker i više JVM instanci. To utiče na potrošnju memorije i vreme pokretanja.

**Provera procesa:**

```bash
# Svi NetWorker procesi
ps -ef | grep /usr/sbin/nsr

# Provera klijentskog procesa
ps -ef | grep /usr/sbin/nsrexecd

# NMC procesi
ps -ef | grep lgtonmc
```

Očekivani izlaz za NMC **[ADM]**:

```
nsrnmc 7190 1 0 Nov23 ? 00:00:06 /opt/lgtonmc/bin/gstd
nsrnmc 7196 1 0 Nov23 ? 00:00:00 /opt/lgtonmc/apache/bin/httpd -f /opt/lgtonmc/apache/conf/httpd.conf
nsrnmc 7212 1 0 Nov23 ? 00:00:00 /opt/lgtonmc/postgres/bin/postgres -D /nsr/nmc/nmcdb/pgdata
```

## 42.6 Pokretanje i zaustavljanje **[ADM]**

RHEL 9.6 koristi systemd. Dokumentacija navodi obe varijante jer podržava i starije distribucije.

### Zaustavljanje

```bash
# systemd (RHEL 9.6)
systemctl stop networker

# sysVinit
/etc/init.d/networker stop

# Potvrda da procesi ne rade
ps -ef | grep /usr/sbin/nsr
```

### Pokretanje

```bash
# systemd (RHEL 9.6)
systemctl start networker

# sysVinit
/etc/init.d/networker start

# Potvrda
/etc/init.d/networker status
```

> **[ADM]** Tabela startnih komandi po OS-u:
>
> | OS | Komanda |
> |---|---|
> | Solaris, Linux | `/etc/init.d/networker start` — za systemd: `systemctl start networker` |
> | HP-UX | `/sbin/init.d/networker start` |
> | AIX | `/etc/rc.nsr` |

### Kontrolisano gašenje — `nsr_shutdown`

> **[CMD]** `nsr_shutdown` je shell skripta koja se koristi za **bezbedno gašenje lokalnog NetWorker servera**. Može je pokretati **samo super-user**.

```bash
nsr_shutdown
```

Koristi se pre pokretanja demona u troubleshoot režimu. **[ADM]**

### NMC servisi

```bash
# systemd
systemctl stop gst
systemctl start gst

# sysVinit
/etc/init.d/gst stop
/etc/init.d/gst start

# Potvrda
ps -ef | grep lgtonmc
```

> **[ADM]** Pri pokretanju NMC-a, **prvo proveriti da `nsrexecd` radi.** Ako ne radi, pokrenuti NetWorker pre NMC-a.

## 42.7 Redosled pokretanja

Iz strukture procesa sledi praktičan redosled:

```
1. nsrexecd (klijentski sloj)          ← mora raditi prvi
2. nsrctld → nsrd → ostali server procesi
3. gstd (NMC)                          ← tek kada NetWorker radi
```

> **[ADM]** Pre povezivanja na Console prozor: proveriti da su NetWorker datazone ispravno konfigurisani i da traženi demoni rade na NetWorker serveru **i** na NMC serveru.

## 42.8 Troubleshoot režim **[ADM]**

Kada standardni logovi nisu dovoljni:

```bash
# 1. Zaustaviti NetWorker procese
nsr_shutdown

# 2a. nsrexecd u troubleshoot režimu (problemi sa klijentskim funkcijama)
nsrexecd -D9 1>/tmp/nsrexecd_debug.log 2>&1

# 2b. nsrctld u troubleshoot režimu (problemi sa serverom)
source /opt/nsr/admin/networkerrc
source /opt/nsr/admin/nsr_serverrc
nsrctld -D 9 1>/tmp/nsrctld_debug.log 2>&1

# 3. Po prikupljanju informacija — zaustaviti pa pokrenuti normalno
nsr_shutdown
systemctl start networker
```

> **[ADM]** `nsrctld` je glavni proces za NetWorker server; `nsrexecd` je glavni proces za NetWorker klijentske funkcije. Za probleme sa serverom pokrenuti `nsrctld` u troubleshoot režimu; za probleme sa klijentskim funkcijama — `nsrexecd`.

---

# 43. Mreža i portovi

## 43.1 Opsezi portova **[CMD]**

NetWorker ne koristi jedan fiksni port, nego **opsege**.

| Tip porta | Podrazumevani opseg | Napomena |
|---|---|---|
| **Service ports** | **7937–9936** | Portovi na kojima NetWorker servisi slušaju |
| **Connection ports** | **0–0** | `0-0` se tretira kao ekvivalent `0-65535` |
| **NetWorker Authentication Service** | **9090** | **[ADM]** |
| **Test konekcije** | **7938** | `nsrports -t <name>` radi DNS lookup i pokušava konekciju na 7938 |

> **[CMD]** Opseg portova može biti jedan ceo broj ili dva cela broja razdvojena crticom. Svaki ceo broj mora biti između 0 i 65535. Opsezi se čuvaju u `nsrexecd`-u, u **NSR system port ranges** resursu.

## 43.2 `nsrports` **[CMD]**

```bash
# Prikaz konfigurisanih opsega za lokalni sistem
nsrports

# Prikaz za drugi sistem
nsrports -s <server>

# Stanje svih NetWorker socket konekcija
nsrports -a

# Bez razrešavanja imena — brže i jasnije
nsrports -a -n

# Ograničiti na IPv4 ili IPv6
nsrports -a -f inet
nsrports -a -f inet6

# Postavljanje service port opsega
nsrports -S 7937-9936

# Test konekcije ka hostu (DNS lookup + konekcija na port 7938)
nsrports -t <hostname>
```

Alternativa preko `nsradmin` **[CMD]**:

```bash
nsradmin -s <server> -p nsrexec
```

```
nsradmin> . type: NSR system port ranges
nsradmin> p
```

## 43.3 Provera na OS nivou (RHEL 9.6)

```bash
# Šta sluša i koji proces
ss -tlnp | grep -E "79[0-9][0-9]|8[0-9]{3}|9[0-9]{3}"

# Konkretno auth servis
ss -tlnp | grep 9090

# Sve konekcije NetWorker procesa
ss -tanp | grep nsr
```

> **Puna tabela portova nije u priloženim vodičima.** **[ADM]** upućuje na *NetWorker Security Configuration Guide* za zahteve servisnih portova pri konfiguraciji firewall-a. **Za produkcionu firewall konfiguraciju pribaviti taj dokument.**

## 43.4 Imenovanje i autentikacija **[CMD]**

Ovo objašnjava zašto NetWorker toliko zavisi od ispravne rezolucije imena.

**Kako klijent određuje svoje ime:**

1. Klijentsko UNIX sistemsko ime se dobija pozivom `gethostname(3)`
2. To ime je parametar za `getaddrinfo(3)`
3. Klijent proglašava svojim imenom **official (primary)** ime koje vrati `getaddrinfo`
4. To ime se prosleđuje serveru pri uspostavljanju konekcije

**Kako server autentifikuje klijenta:**

1. Udaljena adresa konekcije se mapira u kanonsko ime preko `getnameinfo(3)`
2. Deklarisano ime klijenta se koristi kao parametar za `getaddrinfo` radi dobijanja kanonskog imena
3. **Klijent je uspešno autentifikovan samo ako se imena iz obe funkcije poklapaju**

> **Praktična posledica za vault:** ako DNS ne radi ili daje nekonzistentne odgovore, autentikacija pada. Zato je `/etc/hosts` pristup opisan u `07c` (odeljak 53.4) praktično obavezan.

**Pravila za bezbedno i efikasno imenovanje [CMD]:**

1. NetWorker klijenti i serveri treba da pristupaju **konzistentnim bazama imena hostova**. NIS i DNS su naming podsistemi koji pomažu konzistentnosti.
2. Svi `hosts` unosi za jednu mašinu treba da imaju **bar jedan zajednički alias**.
3. Pri kreiranju novog klijenta koristiti ime ili alias koji se mapira nazad na **isto official ime** koje klijentska mašina proizvodi obrnutim mapiranjem svog UNIX sistemskog imena.

## 43.5 Kako klijent pronalazi server **[CMD]**

Redosled traženja NetWorker servera:

| Korak | Metod |
|---|---|
| 1 | Eksplicitno zadat server |
| 2–4 | Provera liste mašina; svaka se ispituje da li je NetWorker server; koristi se **prva** koja jeste |
| 5 | Izdaje se **broadcast** zahtev; koristi se prvi server koji odgovori |
| 6 | Ako server i dalje nije pronađen, koristi se **lokalna mašina** |

> **[CMD]** Administrativne komande koriste **samo korak 1.**

---

# 44. Promenljive okruženja

## 44.1 `/nsr/nsrrc` **[ADM]**

> Na UNIX i Linux sistemima NetWorker **učitava (`source`) `/nsr/nsrrc` fajl pre pokretanja NetWorker procesa.**

```bash
# Ako fajl ne postoji, kreirati ga kao Bourne shell skriptu
vi /nsr/nsrrc
```

Format **[ADM]**:

```sh
ENV_VAR_NAME = value
export ENV_VAR_NAME
```

> **Zaustaviti pa pokrenuti NetWorker procese** da bi promenljive stupile na snagu.

## 44.2 `NSR_SERVER_STATE` **[ADM]**

Najvažnija promenljiva u DR kontekstu.

> Stanje u kojem server startuje može se zadati promenljivom okruženja `NSR_SERVER_STATE`. Na UNIX-u se to **najbolje dodaje u `/nsr/nsrrc`**.

```sh
# /nsr/nsrrc
NSR_SERVER_STATE="disaster recovery"
export NSR_SERVER_STATE
```

> **Ne zaboraviti da se ukloni posle oporavka.** U `disaster recovery` stanju backup, clone, workflow, index management, media management i save set operacije **ne rade**.

Puna tabela stanja servera je u `07c`, odeljak 53.7.

## 44.3 Startne skripte **[ADM]**

| Fajl | Uloga |
|---|---|
| `/opt/nsr/admin/networkerrc` | Opšte okruženje NetWorker-a |
| `/opt/nsr/admin/nsr_serverrc` | Serversko okruženje |

Koriste se pri ručnom pokretanju `nsrctld` u troubleshoot režimu.

---

# 45. Logovi

## 45.1 Format `.raw` i `nsr_render_log`

NetWorker piše glavne logove u **`.raw`** formatu — nisu čitljivi običnim `cat`-om.

> **[CMD]** `nsr_render_log` kreira **ljudski čitljivu verziju** NetWorker logova.

```bash
# Renderovanje glavnog loga
nsr_render_log /nsr/logs/daemon.raw > /tmp/daemon.log
less /tmp/daemon.log

# Praćenje u realnom vremenu (kombinacija sa tail)
nsr_render_log /nsr/logs/daemon.raw | tail -100
```

> **[ADM]** Nerenderovani log fajl ima ekstenziju `.raw`. Renderovani ima `.log`. **Nerenderovani fajlovi sadrže internacionalizovane poruke koje se mogu renderovati na lokalni jezik.** Sadržaj renderovanih fajlova je lokalizovan na jezik jedne zemlje.

## 45.2 Mapa log fajlova NetWorker servera **[ADM]**

| Komponenta | Putanja (Linux) | Sadržaj |
|---|---|---|
| **NetWorker demoni** | `/nsr/logs/daemon.raw` | **Glavni NetWorker log.** Koristiti `nsr_render_log`. |
| **`nsrdr` DR čarobnjak** | `/nsr/logs/nsrdr.log` | Detaljne informacije o internim operacijama `nsrdr` programa. **NetWorker prepisuje ovaj fajl pri svakom pokretanju `nsrdr`-a.** |
| **Index log** | `/nsr/logs/index.log` | Upozorenja o veličini client file indeksa i malom slobodnom prostoru na fajl sistemu sa index fajlovima |
| **Media management** | `/nsr/logs/media.log` | Poruke vezane za uređaje. Podrazumevano device notifikacije šalju poruke ovde, na serveru i na svakom storage node-u. |
| **RAP log** | `/nsr/logs/rap.log` | **Beleži izmene konfiguracije** u resource bazi servera |
| **Policies** | `/nsr/logs/policy.log` | Informacije o završetku VMware Protection politika |
| **Policy notifications** | `/nsr/logs/policy_notifications.log` | Izveštaji o izvršavanju politika, uključujući *Server backup Action report* **[DR]** |
| **Security audit** | `/nsr/logs/NetWorker_server_sec_audit.raw` | Poruke vezane za bezbednosnu reviziju |
| **Snapshot management** | `/nsr/logs/nwsnap.raw` | Kreiranje, montiranje, brisanje i rollover snapshot-ova. Koristiti `nsr_render_log`. |
| **Package Manager** | `/nsr/logs/nsrcpd.raw` | Informacije vezane za Package Manager i `nsrpush`. Koristiti `nsr_render_log`. |
| **Recovery Wizard** | `/nsr/logs/recover/recover_<config_name>_YYYYMMDDHHMMSS` | Pomoć pri dijagnostici neuspelih oporavaka. **Jedan fajl po recover job-u.** |
| **Migration** | `/nsr/logs/migration` | Migracija atributa iz 8.2.x i ranijih resursa pri nadogradnji |
| **Hypervisor** | `/nsr/logs/Hypervisor/hyperv-flr-ui/hyperv-flr-ui.log` | Status Hyper-V FLR interfejsa |
| **VMware politike** | `/nsr/logs/Policy/<VMware_protection_policy_name>` | Status akcija VMware Protection politika; **zaseban fajl po akciji** |

**NMC server [ADM]:**

| Komponenta | Putanja |
|---|---|
| NMC server | `/opt/lgtonmc/management/logs/gstd.raw` |
| NMC konverzija baze | `/opt/lgtonmc/logs/gstdbupgrade.log` |
| NMC web server (Apache) | `/opt/lgtonmc/management/logs/web_output` |
| NMC baza (PostgreSQL) | `/opt/lgtonmc/management/nmcdb/pgdata/db_output` |

**Klijent [ADM]:** `/nsr/logs/daemon.raw`

## 45.3 Logovi politika — hijerarhija **[ADM]**

Ovo je struktura koju administratori često ne poznaju, a najkorisnija je pri dijagnostici neuspelih backup-a.

```
/nsr/logs/policy/<policy_name>/
    <workflow_name>_<jobid>.raw              ← log workflow-a
    <workflow_name>/
        <action_name>_<job_id>.raw           ← log akcije
        <action_name>_<job_id>_logs/
            <job_id>.log                     ← logovi child akcija
```

**Primer [ADM]:**

```
/nsr/logs/policy/server protection/workflow_server backup_0010072.raw
/nsr/logs/policy/server protection/server backup/Backup_1408063.raw
/nsr/logs/policy/server protection/server backup/Clone_1408080.raw
/nsr/logs/policy/server protection/server backup/Clone more_1408200.raw
```

> **[ADM]** `job_id` je vrednost koja jedinstveno identifikuje zapis workflow job-a u `jobdb`. **Koristiti `job id` za upite nad `jobdb` komandom `jobquery`.**

> **[ADM]** Neke akcije kreiraju **child akcije** — npr. backup akcija kreira `save` i `savefs` job. Svaka child akcija ima jedinstven zapis job-a i sopstveni log fajl.

```bash
# Pronalaženje logova poslednjeg izvršavanja Server Protection politike
ls -lt "/nsr/logs/policy/server protection/" | head

# Renderovanje
nsr_render_log "/nsr/logs/policy/server protection/server backup/Backup_1408063.raw"
```

## 45.4 Rotacija `.raw` logova **[ADM]**

NetWorker sam upravlja veličinom i rotacijom `.raw` fajlova.

**Automatski, bez konfiguracije:**

| Fajl | Prag | Broj verzija |
|---|---|---|
| `nwsnap.raw` | 100 MB — proces proverava veličinu pre pisanja | 10 |
| `nsrcpd.raw` | 2 MB — provera pri startu demona | 10 |

**Konfigurabilno** za `daemon.raw`, `gstd.raw`, `networkr.raw` i `Networker_server_sec_audit.raw`:

| Atribut | Podrazumevano | Opseg / format |
|---|---|---|
| **Maximum size MB** | **500 MB** | 10–4000 MB. Od NetWorker 19.9, pri nadogradnji, vrednost manja od 10 se tretira kao 10. |
| **Maximum versions** | **10** | Kada broj kopiranih logova dostigne maksimum, najstariji se briše pri kreiranju nove kopije |
| **Runtime rollover by size** | **Disabled** | Kada je postavljeno, pokreće automatsku **satnu proveru** veličine loga |
| **Runtime rollover by time** | **nedefinisano** | `HH:MM`, dan u nedelji (Sunday–Saturday), ili N-ti dan svakog meseca (1–31). Rollover po danu u nedelji/mesecu dešava se u **prvom satu** odgovarajućeg dana. **Posle postavljanja restartovati NetWorker servise.** |

**Ponašanje mehanizma skraćivanja [ADM]:**

> Kada je konfigurisan runtime rollover po vremenu ili veličini:
> - NetWorker kopira sadržaj postojećeg log fajla u novi fajl sa konvencijom **`daemon<date>_<time>.raw`**
> - NetWorker skraćuje postojeći `daemon.raw` na 0 MB
>
> **Kada se ovaj mehanizam pokrene na opterećenom serveru, proces može potrajati.**

**Izmena kroz `nsradmin` [ADM]:**

```bash
nsradmin -p nsrexec
```

```
nsradmin> . type: NSR log
nsradmin> print

nsradmin> . type: NSR log; name: daemon.raw
nsradmin> print
nsradmin> update maximum size MB: 1000
update? y
```

Primer izlaza **[ADM]**:

```
type: NSR log;
administrator: Administrators, "group=Administrators,host=...";
owner: NetWorker;
maximum size MB: 500;
maximum versions: 10;
runtime rendered log: ;
runtime rollover by size: Disabled;
runtime rollover by time: ;
name: daemon.raw;
log path: /nsr/logs/daemon.raw;
```

## 45.5 Renderovanje u realnom vremenu **[ADM]**

Umesto ručnog pokretanja `nsr_render_log`, NetWorker može istovremeno pisati i renderovani log.

> Za pregled log fajla u tekst editoru **bez prethodnog renderovanja**, postaviti atribut **`runtime rendered log`** u NSRLA bazi.

```bash
nsradmin -p nsrexec
```

```
nsradmin> . type: NSR log; name: daemon.raw
nsradmin> update runtime rendered log: "/nsr/logs/daemon.log"
update? y
nsradmin> print
```

> **[ADM]** Runtime rendered log fajlovi sadrže: **Message ID**, **datum i vreme poruke**, **renderovanu poruku**.

> **[ADM]** Kada konfigurišete runtime rendered log, NetWorker **istovremeno skraćuje i renderovani i pridruženi `.raw` fajl.**

> **Preporuka za vault:** uključiti `runtime rendered log` za `daemon.raw`. U DR situaciji ne želite da vam jedan dodatni korak stoji između vas i loga.

## 45.6 Syslog integracija **[ADM]**

> `nsrdr` i drugi programi pišu u OS log fajl definisan sistemskom syslog konfiguracijom, kroz **`local0.notice`** i **`local0.alert`** facility-je.
>
> **NetWorker ne menja `syslog.conf` da bi konfigurisao `local0.notice` i `local0.alert`.** Vendorska dokumentacija opisuje kako se to konfiguriše.

Na RHEL 9.6 sa rsyslog-om:

```bash
# /etc/rsyslog.d/networker.conf
local0.notice   /var/log/networker.log
local0.alert    /var/log/networker-alert.log
```

```bash
systemctl restart rsyslog
```

> Ovo je korisno za prosleđivanje NetWorker događaja ka SIEM sistemu iz vault-a.

## 45.7 Šta prikupiti pri prijavi problema

```bash
# Renderovani glavni log
nsr_render_log /nsr/logs/daemon.raw > /tmp/daemon_$(date +%F).log

# DR log
cp /nsr/logs/nsrdr.log /tmp/

# Logovi politike
tar czf /tmp/policy_logs.tar.gz /nsr/logs/policy/

# Verzije i stanje
rpm -qa | grep -i lgto > /tmp/pkgs.txt
/etc/init.d/networker status > /tmp/status.txt
nsrports > /tmp/ports.txt
```

---

# 46. SELinux i firewalld na RHEL 9.6

> **Napomena:** priloženi NetWorker vodiči **ne pokrivaju SELinux ni firewalld konfiguraciju za NetWorker**. Ovaj odeljak daje dijagnostički pristup, ne gotovu politiku. Za zvanične preporuke pribaviti *NetWorker Security Configuration Guide* i *NetWorker Installation Guide*. **[?]**

## 46.1 Provera SELinux stanja

```bash
getenforce
sestatus
```

## 46.2 Dijagnostika SELinux odbijanja

Kada servis ne startuje ili uređaj ne radi, a nema očiglednog razloga:

```bash
# Skorašnja odbijanja
ausearch -m AVC,USER_AVC -ts recent

# Čitljiv rezime sa predlozima
sealert -a /var/log/audit/audit.log 2>/dev/null | head -60

# Konteksti NetWorker binarnih fajlova
ls -Z /usr/sbin/nsr*

# Konteksti /nsr stabla
ls -Zd /nsr /nsr/res /nsr/mm /nsr/index /nsr/logs
```

**Privremeni test — da li je SELinux uzrok:**

```bash
setenforce 0
systemctl restart networker
# ako sada radi, uzrok je SELinux politika

# vratiti nazad
setenforce 1
```

> **Ne ostavljati SELinux u Permissive režimu kao trajno rešenje.** To je dijagnostički korak, ne konfiguracija.

## 46.3 Kada je `/nsr` na nestandardnoj lokaciji

Ako je `/nsr` simbolički link na drugi fajl sistem, SELinux kontekst ciljnog direktorijuma može biti pogrešan:

```bash
# Prikaz stvarne lokacije
ls -ld /nsr
readlink -f /nsr

# Poređenje konteksta
ls -Zd /nsr/
ls -Zd $(readlink -f /nsr)

# Vraćanje podrazumevanih konteksta prema fcontext pravilima
restorecon -Rv $(readlink -f /nsr)
```

## 46.4 firewalld

```bash
# Trenutna pravila
firewall-cmd --list-all

# Otvaranje service port opsega
firewall-cmd --permanent --add-port=7937-9936/tcp

# Authentication Service
firewall-cmd --permanent --add-port=9090/tcp

# NMC (ako je u obimu)
firewall-cmd --permanent --add-port=9000/tcp

firewall-cmd --reload
firewall-cmd --list-ports
```

> **Portovi 7937–9936 i 9090 su potvrđeni** u **[CMD]** i **[ADM]**. Port 9000 za NMC je izveden iz `http://<gst_server_name>:9000` URL-a u **[DR]**. **Puna lista portova za produkcionu firewall politiku zahteva *NetWorker Security Configuration Guide*.**

## 46.5 Vremenska sinhronizacija

```bash
chronyc tracking
chronyc sources
timedatectl
```

> Iz **[CR]**: preporučuje se NTP sinhronizacija **svih komponenti** u vault-u. Neusklađeno vreme između NetWorker servera, DD sistema i Cyber Recovery hosta uzrokuje probleme koje je teško dijagnostikovati.

---

# 47. Provera zdravlja instance u 60 sekundi

Redosled komandi kada dolazite na nepoznat sistem.

## 47.1 Šest komandi

```bash
# 1. Šta je instalirano i koja verzija
rpm -qa | grep -i lgto

# 2. Da li procesi rade i u kakvoj hijerarhiji
/etc/init.d/networker status

# 3. Ima li prostora
df -h /nsr

# 4. Šta je montirano na kom uređaju
nsrmm -C

# 5. Šta piše u glavnom logu
nsr_render_log /nsr/logs/daemon.raw | tail -50

# 6. U kom je stanju server
nsradmin -s $(hostname -f) <<'EOF'
. type: NSR
p
EOF
```

## 47.2 Kako čitati rezultat

| Komanda | Šta znači dobar rezultat | Alarm |
|---|---|---|
| `rpm -qa \| grep lgto` | Prisutni `lgtoclnt`, `lgtonode`, `lgtoserv`, `lgtoauthc` | Nedostaje neki paket → `nsrdr` neće raditi |
| `networker status` | Stablo počinje sa `nsrctld`, sadrži `nsrd`, `nsrmmdbd`, `nsrindexd`, `nsrjobd` | Nema `nsrctld` → servis ne radi. Nema `nsrd` → server sloj nije podignut. |
| `df -h /nsr` | Dovoljno prostora | > 85% → indeksi i baze mogu stati |
| `nsrmm -C` | Lista uređaja sa montiranim volumenima | Prazan izlaz → nema konfigurisanih uređaja ili server ne odgovara |
| `daemon.raw` | Nema ponavljajućih grešaka | Ponavljajuće poruke o mount, autentikaciji ili rezoluciji imena |
| `. type: NSR` | `server state: active` | `disaster recovery` → backup i workflow **neće raditi** |

## 47.3 Proširena provera

```bash
# Rezolucija imena — najčešći uzrok problema u vault-u
hostname -f
grep "^hosts:" /etc/nsswitch.conf
getent hosts $(hostname -f)

# UID koji mora odgovarati produkciji pri nsrdr ka drugom serveru
id nsrtomcat

# Da li je /nsr simbolički link
ls -ld /nsr

# Format media baze
ls -ld /nsr/mm/mmvolume6 /nsr/mm/mmvolrel 2>/dev/null

# Zaostali marker fajlovi
ls -l /nsr/debug/nsr_disaster_recovery_mode /nsr/debug/cdidisable 2>/dev/null

# Core fajlovi — znak ranijih padova
find /nsr/cores -type f 2>/dev/null | head

# Prekinuti save-ovi
find /nsr/index -name "*.sip" 2>/dev/null | head

# Upozorenja o veličini indeksa
tail -20 /nsr/logs/index.log 2>/dev/null

# Portovi
nsrports
ss -tlnp | grep 9090

# Vreme
chronyc tracking | head -5
```

## 47.4 Kada stati i ne dirati dalje

Situacije u kojima dalja intervencija bez pripreme može naneti štetu:

| Situacija | Zašto stati |
|---|---|
| **Postoje `.cr.<timestamp>` ili `.<timestamp>` kopije `/nsr/res`, `/nsr/mm`, `/nsr/index`** | Neko je već pokretao oporavak. Utvrditi šta je urađeno pre nego što se pokrene novi. |
| **`server state` je `disaster recovery`** | Oporavak je možda u toku ili nije očišćen. Ne pokretati backup-e. |
| **Volumeni imaju `scan needed` zastavicu** | Media baza ne odgovara sadržaju medija. Pisanje može prepisati podatke. |
| **`/nsr` je pun ili blizu punog** | Operacije mogu pasti na pola i ostaviti nekonzistentno stanje. |
| **Zaostao `nsr_disaster_recovery_mode` fajl** | `nsrdr` je pao. DNS keš se ne popunjava; ponašanje servera je izmenjeno. |
| **Ne znate lockbox / DD Boost kredencijale** | Bez njih se uređaj ne može montirati ni posle uspešnog oporavka resursa. |

> **Pravilo:** pre bilo kakve intervencije na sistemu za koji ne znate istoriju — napraviti kopiju `/nsr/res` i zabeležiti izlaz šest komandi iz odeljka 47.1. To je pet minuta rada koje mogu spasiti dan.

---

*Kraj Dela VII-a. Videti i: `07b` (storage i katalog), `07c` (kompletan tok oporavka), `07d` (logovi i dijagnostika po simptomu).*
