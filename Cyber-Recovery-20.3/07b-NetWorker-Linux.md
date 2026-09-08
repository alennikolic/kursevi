# DEO VII — NETWORKER NA RHEL 9.6: PLAN I INVENTAR KOMANDI

**Dell PowerProtect Cyber Recovery 20.3 — Uputstvo za instalaciju i integraciju**
Radni dokument za dogovor o obimu | v0.1

> **Namena Dela VII:** operativna referenca za System Engineer-a koji na Linux konzoli u vault-u radi oporavak NetWorker servera. Fokus je na tome **šta se kuca**, **gde su logovi**, **gde se šta montira** i **kako se od potpuno nove instance dođe do funkcionalnog oporavka nad repliciranim storage-om.
>
> **Referentni OS:** Red Hat Enterprise Linux 9.6
> **Referentna verzija NetWorker-a:** 19.13 i novije

---

# 1. Oznake pouzdanosti izvora

Da bi bilo jasno šta je potvrđeno, a šta zahteva verifikaciju:

| Oznaka | Značenje |
|---|---|
| **[DR]** | Potvrđeno u *NetWorker Server Disaster Recovery and Availability Best Practices Guide 19.13* |
| **[CR]** | Potvrđeno u *Cyber Recovery 20.3 Product Guide* ili *Installation and Upgrade Guide* |
| **[PPDM]** | Potvrđeno u *PowerProtect Data Manager 19.22* vodičima |
| **[?]** | **Opšte znanje o proizvodu — sintaksu i ponašanje treba potvrditi** uz *NetWorker Command Reference Guide* i *NetWorker Administration Guide* za konkretnu verziju |

> **Iskreno o rizicima:** komande označene **[?]** su one koje SE u praksi najviše koristi na konzoli (`nsradmin`, `nsrinfo`, `nsrls`, `nsrim`), ali ih priložena dokumentacija ne pokriva. Ako ih pišem bez izvora, rizikujem da prenesem zastarelu ili netačnu sintaksu — što je u DR situaciji gore nego da komanda uopšte nije navedena.
>
> **Preporuka:** pribaviti *NetWorker Command Reference Guide 19.13+* i *NetWorker Administration Guide 19.13+* pre pisanja sadržaja. Bez njih ću **[?]** komande obraditi konzervativno: navesti čemu služe i uputiti na `man` stranicu, umesto da izmišljam prekidače.

---

# 2. Inventar komandi

## 2.1 Upravljanje servisima i procesima

| Komanda | Namena | Izvor |
|---|---|---|
| `/etc/init.d/networker start\|stop` | Pokretanje i zaustavljanje NetWorker-a | **[CR]** |
| `systemctl start\|stop\|status networker` | systemd ekvivalent na RHEL 9.x | **[?]** |
| `nsr_shutdown` | Kontrolisano gašenje NetWorker demona | **[?]** |
| `nsrwatch` | Praćenje poruka i statusa servera u realnom vremenu | **[DR]** |
| `/etc/init.d gst start\|stop` | NMC (GST) servis | **[DR]** |
| `net stop\|start gstd` | NMC servis na Windows-u (za poređenje) | **[DR]** |
| `ps -ef \| grep nsr` | Pregled aktivnih NetWorker procesa | opšte |
| `nsrports` | Prikaz i izmena opsega portova | **[?]** |

## 2.2 Demoni koje treba prepoznati u `ps` izlazu

| Demon | Uloga | Izvor |
|---|---|---|
| `nsrd` | Glavni NetWorker servis | **[?]** |
| `nsrexecd` | Izvršavanje udaljenih zahteva (klijentska strana) | **[?]** |
| `nsrmmd` | Media multiplexor — rad sa uređajima | **[?]** |
| `nsrmmdbd` | Media baza | **[?]** |
| `nsrindexd` | Client file indeksi | **[?]** |
| `nsrjobd` | Job baza (`jobsdb`) | **[?]** |
| `nsrlogd` | Logovanje | **[?]** |
| `nsrsnmd` | Storage node media servis | **[?]** |
| `gstd` | NMC (Management Console) demon | **[DR]** |
| `nsrtomcat` | Tomcat instanca; **UID ovog korisnika mora biti isti na izvornom i ciljnom NetWorker serveru pri `nsrdr` ka drugom serveru** | **[DR]** |
| `authc` (Authentication Service) | Autentikacija; baza `authcdb.h2.db` | **[DR]** |

## 2.3 Konfiguracija i RAP resursi

| Komanda | Namena | Izvor |
|---|---|---|
| `nsradmin` | Interaktivni i skriptabilni editor RAP resursa (klijenti, uređaji, pool-ovi, NSR resurs) — **glavni alat kad NMC nije dostupan** | **[?]** |
| `nsraddadmin -u "user=..., host=..."` | Dodavanje naloga u Administrators listu servera | **[DR]** |
| `authc_configure.sh` | Konfiguracija Authentication Service-a; putanja `/opt/nsr/authc-server/scripts/` | **[DR]** |
| `nsrcap` | Unos licence / enabler koda | **[?]** |
| `nsrlogin` / `nsrlogout` | Autentikacioni tokeni | **[?]** |

## 2.4 Uređaji, volumeni i storage

| Komanda | Namena | Izvor |
|---|---|---|
| `nsrmm -C` | **Prikaz koji volumen je montiran na kom uređaju** i u kom režimu | **[DR]** |
| `nsrmm -o notscan <volume>` | Uklanjanje Scan Needed zastavice sa volumena | **[DR]** |
| `nsrjb -vHE` | Reset autochanger-a, izbacivanje volumena, reinicijalizacija element statusa | **[DR]** |
| `nsrjb -I` | Inventar autochanger-a | **[DR]** |
| `ielem` | Inicijalizacija element statusa ako uređaj ne podržava `-E` | **[DR]** |
| `inquire` | Otkrivanje SCSI uređaja | **[DR]** |
| `sjirdtag <devname>` | Provera kontrolnog porta jukebox-a | **[DR]** |
| `mount` / `df -h` | Provera montiranih fajl sistema na OS nivou | opšte |
| `showmount -e <DD>` | Pregled NFS eksporta sa DD sistema | opšte |

> **DD Boost storage unit-i se ne vide kroz `mount`.** Ovo je čest izvor zabune i biće posebno obrađeno — DD Boost uređaj je aplikativna konstrukcija, ne fajl sistem mount.

## 2.5 Katalog — media baza i client file indeksi

| Komanda | Namena | Izvor |
|---|---|---|
| `mminfo -B` | **Najnovije bootstrap informacije** | **[DR]** |
| `mminfo -av -B -s <server>` | Bootstrap info kada media baza postoji | **[DR]** |
| `mminfo -avot -q client=<c>,level=full -r client,name,savetime,nsavetime` | Upit nad save set-ovima sa izborom kolona | **[DR]** |
| `nsrck -L7` | **Ponovna izgradnja client file indeksa** iz index backup-a | **[DR]**, **[CR]** |
| `nsrinfo` | Pregled sadržaja client file indeksa | **[?]** |
| `nsrls` | Veličina i statistika indeksa | **[?]** |
| `nsrim` | Održavanje indeksa i media baze | **[?]** |
| `nsrmmdbasm` | Rad sa media bazom; **jedina koja se sme pokretati u disaster recovery stanju** | **[DR]** |

## 2.6 Skeniranje medija

| Komanda | Namena | Izvor |
|---|---|---|
| `scanner -B <device>` | **Pronalaženje bootstrap SSID-a na uređaju** | **[DR]** |
| `scanner -i <device>` | Popunjavanje CFI i media baze informacijama o save set-ovima | **[DR]**, **[CR]** |
| `scanner -m -S <SSID/CloneID> <device>` | Popunjavanje media baze podacima o kloniranom save set-u | **[DR]** |
| `scanner -s <server> -m <device>` | Popunjavanje media baze ciljnog servera | **[DR]** |
| `scanner -f <file> -r <record> -i <device>` | Skeniranje trake od zadatog file i record broja | **[DR]** |
| `scanner -i -V <volume> -Z <datazone_ID> <device>` | Cloud volumen | **[DR]** |

> **`scanner -i` može trajati veoma dugo**, posebno na velikom disk volumenu. **[DR]**

## 2.7 Disaster recovery

| Komanda | Namena | Izvor |
|---|---|---|
| `nsrdr` | Interaktivni DR čarobnjak — media baza, resource fajlovi, CFI | **[DR]** |
| `nsrdr -N` | Zaštita od prepisivanja ručnih backup-a posle poslednjeg bootstrap-a | **[DR]** |
| `nsrdr -N -F` | Scan Needed samo za File type, AFTD i Cloud uređaje | **[DR]** |
| `nsrdr -a -B <ID> -d <device> -I` | Neinteraktivni režim | **[DR]** |
| `nsrdr -c -I <klijenti>` | Samo CFI za izabrane klijente | **[DR]** |
| `nsrdr -c -t <date/time> -I <klijenti>` | CFI od zadatog datuma | **[DR]** |
| `nsrdr -f <fajl> -I` | Lista klijenata iz ASCII fajla | **[DR]** |
| `nsrdr -K` | Koristi originalne resource fajlove umesto oporavljenih | **[DR]** |
| `nsrdr -l <putanja>` | Alternativna putanja za preimenovanje resursa | **[DR]** |
| `recoverpsm -s <NW_server> -c <NMC_server> <staging_dir>` | **Oporavak NMC baze — `nsrdr` je ne dira** | **[DR]** |
| `recover` | Interaktivni oporavak fajlova | **[DR]** |
| `nsrclone` | Kloniranje save set-a (npr. sa udaljenog na lokalni uređaj) | **[DR]** |

## 2.8 Politike i job-ovi

| Komanda | Namena | Izvor |
|---|---|---|
| `nsrpolicy start -p <policy> -w <workflow>` | Ručno pokretanje workflow-a | **[DR]** |
| `nsrpolicy monitor -p <policy> -w <workflow>` | Praćenje statusa | **[DR]** |
| `jobquery` | Upiti nad job bazom | **[?]** |
| `save` | Ručni backup | **[?]** |

## 2.9 Logovi i dijagnostika

| Komanda / putanja | Namena | Izvor |
|---|---|---|
| `nsr_render_log` | **Čitanje `daemon.raw` u ljudski čitljivom obliku** | **[DR]** |
| `/nsr/logs/nsrdr.log` | Log `nsrdr` procedure | **[DR]** |
| `/nsr/logs/policy_notifications.log` | Izveštaji o izvršavanju politika, uključujući *Server backup Action report* | **[DR]** |
| `/nsr/logs/daemon.raw` | Glavni log demona | **[?]** |
| `/nsr/debug/nsrdr.conf` | Parametri podešavanja `nsrdr`-a | **[DR]** |
| `/nsr/debug/nsr_disaster_recovery_mode` | Marker fajl; preskače popunjavanje DNS keša | **[DR]** |
| `nsr_getdate` | Format datuma prihvatljiv za `-t` opciju | **[DR]** |

## 2.10 OS nivo — RHEL 9.6

| Komanda | Namena |
|---|---|
| `systemctl` / `journalctl -u <unit>` | Servisi i systemd logovi |
| `firewall-cmd --list-all` | Firewall pravila |
| `getenforce` / `setenforce` / `ls -Z` / `chcon` | SELinux |
| `id nsrtomcat` | **Provera UID-a — kritično za `nsrdr` ka drugom serveru** |
| `ss -tlnp` | Otvoreni portovi i procesi |
| `/etc/nsswitch.conf` | Redosled rezolucije imena (`hosts: files`) |
| `/etc/hosts` | Lokalna rezolucija u vault-u |
| `chronyc tracking` / `chronyc sources` | NTP sinhronizacija |
| `df -h /nsr` | **Prostor za resource, media i index baze** |
| `du -sh /nsr/*` | Šta zauzima prostor |

---

# 3. Struktura direktorijuma koju treba dokumentovati

| Putanja | Sadržaj | Izvor |
|---|---|---|
| `/nsr` | Koren NetWorker podataka | **[DR]** |
| `/nsr/res` | **Resource baza** — klijenti, uređaji, pool-ovi, politike | **[DR]** |
| `/nsr/res/nsrdb` | Konfiguracioni fajlovi | **[DR]** |
| `/nsr/res/lockbox` (approx.) | **Lockbox** — Oracle lozinke, DD Boost lozinka, enkriptovano | **[DR]** |
| `/nsr/mm` | **Media baza** | **[CR]** |
| `/nsr/index` | **Client file indeksi**, po jedan direktorijum po klijentu | **[DR]**, **[CR]** |
| `/nsr/logs` | Logovi | **[DR]** |
| `/nsr/debug` | Debug fajlovi i `nsrdr.conf` | **[DR]** |
| `/nsr/lic` | **Licencni fajl `dpa.lic`** | **[DR]** — videti napomenu ispod |
| `/opt/nsr/authc-server/scripts/` | Skripte Authentication Service-a | **[DR]** |
| `/nsr/res.R` | Privremeni folder oporavljene resource baze tokom `nsrdr` | **[DR]** |
| `/nsr/res.<timestamp>` | Preimenovana prethodna resource baza | **[DR]** |
| `/nsr/res.cr.<timestamp>` | Prethodna resource baza pri **Cyber Recovery** oporavku | **[CR]** |
| `/nsr/mm.cr.<timestamp>` | Prethodna media baza pri CR oporavku | **[CR]** |
| `/nsr/index.cr.<timestamp>` | Prethodni indeksi pri CR oporavku | **[CR]** |
| `/opt/dellemc/cr/mnt/cr-rec-<sandbox>_1604` | **Mount tačka CR sandbox-a na CR management hostu** | **[CR]** |

> **Napomena o `/nsr/lic`:** *DR Best Practices Guide* na jednom mestu piše `/nrs/lic`. To je gotovo sigurno štamparska greška za `/nsr/lic`. U dokumentu ću navesti ispravnu putanju uz napomenu — proveriti na sistemu.

---

# 4. Plan naslova

Predlažem **četiri fajla**. Podela prati redosled kojim SE stvarno radi: prvo razume sistem, pa vidi storage, pa oporavlja, pa troubleshoot-uje.

---

## Fajl 1 — `07a-NetWorker-Linux-Anatomija.md`

### DEO VII-a — Anatomija NetWorker instalacije na RHEL 9.6

**39. Šta je NetWorker iz ugla Linux administratora**
- 39.1 Mentalni model: RAP baza, media baza, indeksi, job baza
- 39.2 Šta je stanje, a šta konfiguracija
- 39.3 Zašto je bootstrap jedini konzistentan snimak

**40. Struktura fajl sistema**
- 40.1 Mapa `/nsr` sa objašnjenjem svakog direktorijuma
- 40.2 Šta ide u bootstrap, a šta ne
- 40.3 Simbolički linkovi i posledice pri oporavku
- 40.4 Preporuke za razdvajanje na zasebne LUN-ove
- 40.5 Praćenje zauzeća prostora

**41. Paketi i instalacija**
- 41.1 Koji paketi su obavezni za vault instancu
- 41.2 Provera instaliranih paketa i verzija
- 41.3 Nadogradnja preko `rpm -U`
- 41.4 Šta `authc_configure.sh` radi i kada se pokreće

**42. Servisi i demoni**
- 42.1 Mapa demona i njihovih uloga
- 42.2 Kako izgleda zdrav `ps -ef | grep nsr`
- 42.3 Startni redosled i zavisnosti
- 42.4 systemd na RHEL 9.6 vs. `/etc/init.d/networker`
- 42.5 Kontrolisano gašenje
- 42.6 Disaster recovery stanje — šta se zaustavlja, a šta radi

**43. Mreža i portovi**
- 43.1 Portovi koje NetWorker koristi
- 43.2 `firewalld` konfiguracija na RHEL 9.6
- 43.3 Rezolucija imena u vault-u — `nsswitch.conf` i `/etc/hosts`
- 43.4 Zašto server sporo startuje bez DNS-a i kako to zaobići
- 43.5 `nsr_disaster_recovery_mode` marker fajl

**44. SELinux i NetWorker na RHEL 9.6**
- 44.1 Očekivano ponašanje
- 44.2 Dijagnostika SELinux odbijanja
- 44.3 Šta raditi kad servis ne startuje

**45. Provera zdravlja instance u 60 sekundi**
- 45.1 Redosled od šest komandi
- 45.2 Šta znači svaki izlaz
- 45.3 Kada stati i ne dirati dalje

---

## Fajl 2 — `07b-NetWorker-Linux-Storage-Katalog.md`

### DEO VII-b — Storage, uređaji i katalog iz konzole

**46. Kako NetWorker vidi storage**
- 46.1 Tri sveta: AFTD, DD Boost uređaj, traka
- 46.2 **Zašto se DD Boost storage unit ne vidi u `mount`**
- 46.3 Odnos: DD storage unit → NetWorker device → volume → save set
- 46.4 Šta je zaista replicirano u vault, a šta NetWorker mora sam da rekonstruiše

**47. Pregled uređaja i volumena iz konzole**
- 47.1 `nsrmm -C` — šta je montirano na čemu
- 47.2 Pregled uređaja kroz `nsradmin`
- 47.3 Režimi volumena i šta znače
- 47.4 Provera na DD strani — poređenje sa NetWorker pogledom
- 47.5 Provera NFS eksporta i mount tačaka na OS nivou

**48. Katalog: media baza vs. client file index**
- 48.1 Šta je u media bazi
- 48.2 Šta je u client file indeksu
- 48.3 Kada je CFI neophodan, a kada nije
- 48.4 Odnos browse policy i retention policy
- 48.5 Zašto retencija klijenta NetWorker servera mora biti `Decade`

**49. Ispitivanje kataloga**
- 49.1 `mminfo` — recepti za najčešće upite
- 49.2 Pronalaženje bootstrap-a: tri metode
- 49.3 Čitanje *Server backup Action report* sekcije
- 49.4 Pregled sadržaja indeksa

**50. Skeniranje medija**
- 50.1 Kada je `scanner` neophodan
- 50.2 `scanner -B` — pronalaženje bootstrap SSID-a
- 50.3 `scanner -i` — popunjavanje kataloga
- 50.4 Trajanje i kako proceniti
- 50.5 Scan Needed zastavica — postavljanje i uklanjanje

**51. Kreiranje device resursa nad repliciranim podacima**
- 51.1 Pravila koja se ne smeju prekršiti
- 51.2 **Zašto se volumen nikada ne sme ponovo označiti (label)**
- 51.3 Isključivanje Label and Mount opcije
- 51.4 Putanja mora odgovarati lokaciji bootstrap podataka
- 51.5 Kreiranje uređaja kroz `nsradmin` bez NMC-a

---

## Fajl 3 — `07c-NetWorker-Linux-Recovery.md`

### DEO VII-c — Od nove instance do oporavka: kompletan tok

**52. Polazna tačka**
- 52.1 Šta imamo: sveža instanca + replicirani MTree u vault-u
- 52.2 Šta nam treba pre nego što išta kucnemo — checklist
- 52.3 Dva puta: automatizovani CR recovery vs. ručni `nsrdr`
- 52.4 Kada se ide ručnim putem (više MTree-ova, neuspeo automat)

**53. Priprema instance**
- 53.1 Verifikacija verzije i paketa
- 53.2 Hostname, FQDN, aliasi
- 53.3 Rezolucija imena
- 53.4 `nsrtomcat` UID
- 53.5 Retention policy klijenta na `Decade`
- 53.6 Provera `authcdb.h2.db` — ne sme biti noviji od bootstrap-a

**54. Podmetanje repliciranog storage-a**
- 54.1 Provera da su podaci stigli u vault MTree
- 54.2 Kreiranje DD Boost korisnika sa ispravnim UID-om
- 54.3 Kreiranje device resursa
- 54.4 Verifikacija da NetWorker vidi volumen
- 54.5 Šta ako volumen nije u media bazi

**55. Pronalaženje bootstrap-a bez media baze**
- 55.1 Tri metode, po redosledu pokušaja
- 55.2 `scanner -B` na uređaju
- 55.3 Interpretacija SSID/CloneID izlaza
- 55.4 Ako je bootstrap na udaljenom uređaju — kloniranje na lokalni

**56. `nsrdr` — puna procedura**
- 56.1 Odluke pre pokretanja: `-N`, `-N -F`, CFI obim
- 56.2 Unmount svih volumena
- 56.3 Omogućavanje CDI atributa
- 56.4 Prolazak kroz interaktivne upite, korak po korak
- 56.5 Šta se dešava sa `res`, `mm`, `index` u pozadini
- 56.6 Zamena Authentication Service baze
- 56.7 Oporavak CFI — svi ili izabrani klijenti
- 56.8 `authc_configure.sh` posle oporavka
- 56.9 Podešavanje `nsrdr.conf` za veliki broj klijenata

**57. Posle `nsrdr`**
- 57.1 Verifikacija resursa: Protection, Devices, Media
- 57.2 Režimi volumena
- 57.3 Uklanjanje Scan Needed zastavice
- 57.4 `nsrck -L7` — kada i zašto
- 57.5 `jobsdb` je prazan — očekivano
- 57.6 NMC baza — `recoverpsm`
- 57.7 Lažne greške u `nsrdr.log` koje se ignorišu

**58. Oporavak podataka**
- 58.1 Učitavanje i inventarisanje uređaja
- 58.2 Rad sa kloniranim volumenima
- 58.3 `recover` — interaktivni tok
- 58.4 Upozorenje o prepisivanju sistemskih fajlova

**59. Kada oporavak ne uspe**
- 59.1 Vraćanje na prethodno stanje — `.cr.<timestamp>` direktorijumi
- 59.2 Ručno čišćenje posle prekinutog CR oporavka
- 59.3 Brisanje objekata koje je oporavak kreirao
- 59.4 Unmount sandbox-a sa CR management hosta
- 59.5 Kada zvati Dell Support i sa čim

---

## Fajl 4 — `07d-NetWorker-Linux-Logovi-Troubleshooting.md`

### DEO VII-d — Logovi i dijagnostika

**60. Mapa logova**
- 60.1 Tabela: fajl → šta sadrži → kada se gleda
- 60.2 `daemon.raw` i `nsr_render_log`
- 60.3 `nsrdr.log`
- 60.4 `policy_notifications.log`
- 60.5 systemd žurnal na RHEL 9.6
- 60.6 Rotacija i zauzeće

**61. Dijagnostika po simptomu**
- 61.1 Servis ne startuje
- 61.2 Server startuje veoma sporo
- 61.3 Uređaj se ne montira
- 61.4 Volumen nije u media bazi
- 61.5 `nsrdr` ne pronalazi bootstrap
- 61.6 CFI oporavak ne uspeva
- 61.7 NMC ne prikazuje resurse posle oporavka
- 61.8 Automatizovani CR recovery ne uspeva
- 61.9 Problemi sa dozvolama i vlasništvom nad `/nsr`

**62. Prikupljanje podataka za Dell Support**
- 62.1 Šta priložiti uz slučaj
- 62.2 Povećanje nivoa logovanja
- 62.3 Reprodukcija problema

**63. Brza referenca**
- 63.1 Kartica komandi za štampu — jedna strana
- 63.2 Redosled koraka za oporavak — jedna strana
- 63.3 Putanje i logovi — jedna strana

---

# 5. Pitanja pre pisanja

| # | Pitanje | Zašto je bitno |
|---|---|---|
| 1 | Možete li pribaviti **NetWorker Command Reference Guide** i **NetWorker Administration Guide** za 19.13+? | Bez njih **[?]** komande obrađujem konzervativno — opis namene bez tačne sintakse prekidača. To je posebno osetljivo za `nsradmin`, koji je centralni alat kada NMC nije dostupan. |
| 2 | Da li je u vault-u **NMC** u obimu? | Određuje da li poglavlje 57.6 (`recoverpsm`) ide detaljno ili se samo pominje |
| 3 | Koristi li se u vault-u **traka** ili samo DD? | Poglavlja o `nsrjb`, `ielem`, `inquire`, `sjirdtag` bi otpala ako je samo DD |
| 4 | Da li je NetWorker instanca u vault-u **fizička ili VM**? | Utiče na deo o disk layout-u i tačkama vraćanja |
| 5 | Verzija NetWorker-a kod kupca | 19.13 je referentna iz vodiča; ako je kod kupca novija, neke stvari mogu odstupati |
| 6 | Da li želite **jedan veliki fajl** umesto četiri? | Predlažem četiri zbog obima; četvrti (brza referenca) je zamišljen kao materijal za štampu |

---

# 6. Predlog redosleda izrade

1. **`07b`** (Storage i katalog) i **`07c`** (Recovery) su srž — predlažem da počnemo od njih
2. **`07a`** (Anatomija) daje kontekst, ali je manje hitan
3. **`07d`** (Logovi i troubleshooting) na kraju, jer se oslanja na sve prethodno

> Ako se slažete, krećem od **`07c`** — kompletan tok od nove instance do oporavka — pošto ste ga označili kao suštinu.
