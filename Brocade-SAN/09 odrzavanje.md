# Poglavlje 9: Održavanje

> **Cilj poglavlja:** Znati kako se konfiguracija izvozi i vraća, kako se bezbedno nadograđuje firmware, šta je `supportsave` i kako izgleda rutina periodičnih provera koja sprečava da se problemi otkriju tek kada nešto stane.

---

## 9.1 Rutina održavanja

SAN switchevi su uređaji koji godinama rade bez intervencije — i upravo zbog toga se zapuste. Sledeći raspored pokriva ono što se u praksi pokazalo kao dovoljno.

| Učestalost | Šta se radi |
|---|---|
| **Dnevno** (automatizovano) | Praćenje alarma preko syslog-a i SNMP-a |
| **Nedeljno** | `switchstatusshow`, `porterrshow`, pregled loga |
| **Mesečno** | `sfpshow -all` i poređenje optičkih vrednosti, provera backup-a, `cfgsize` |
| **Kvartalno** | Provera FOS verzije prema matrici, pregled nekorišćenih portova i zona |
| **Godišnje** | Plan nadogradnje firmware-a, provera isteka licenci i ugovora o podršci |
| **Pre svake izmene** | `configupload` |

---

## 9.2 Backup konfiguracije

### Interaktivni oblik

```
SAN_A_SW01:admin> configupload
Protocol (scp, ftp, sftp, local) [ftp]: scp
Server Name or IP Address [host]: 10.10.20.30
User Name [user]: backup
Path/Filename [<home dir>/config.txt]: /backup/san/SAN_A_SW01_2026-09-15.txt
Section (all|chassis|switch [all]): all
Password:
configUpload complete: All selected config parameters are uploaded
```

### Oblik u jednoj liniji

Pogodan za skriptovanje:

```
SAN_A_SW01:admin> configupload -all -scp 10.10.20.30,backup,/backup/san/SW01_2026-09-15.txt
Password:
configUpload complete: All selected config parameters are uploaded
```

### Sekcije

| Sekcija | Sadržaj |
|---|---|
| `chassis` | Podešavanja šasije — zajednička za ceo uređaj |
| `switch` | Podešavanja switcha, uključujući zoning |
| `all` | Oboje — koristite ovo |

### Šta backup sadrži, a šta ne

**Sadrži:** zoning bazu, podešavanja portova, mrežnu konfiguraciju, SNMP, korisničke naloge (bez lozinki u čitljivom obliku), fabric parametre.

**Ne sadrži:** firmware, lozinke u upotrebljivom obliku, licencne ključeve u svim slučajevima.

Zato uz backup vodite i zaseban zapis:

```
SAN_A_SW01:admin> licenseshow > (zabeležiti ručno)
SAN_A_SW01:admin> licenseidshow
10:00:38:ba:b0:fc:9f:b0
```

Licencne ključeve i License ID svakog uređaja čuvajte u dokumentaciji, odvojeno od switcha.

### Provera da backup nije prazan

Backup koji niko nikada nije otvorio nije backup. Na serveru gde je fajl:

```
[root@backup ~]# head -20 /backup/san/SAN_A_SW01_2026-09-15.txt
[Configuration upload Information]
Configuration Format = 2.0
Vf Id = 128
Switch Name = SAN_A_SW01
Fabric OS Version = v9.2.0c3
Time Stamp = Tue Sep 15 11:42:03 2026

[Chassis Configuration Begin]
...

[root@backup ~]# grep -c "^zone" /backup/san/SAN_A_SW01_2026-09-15.txt
14

[root@backup ~]# grep "^zone.cfg" /backup/san/SAN_A_SW01_2026-09-15.txt
zone.cfg.PROD_FABA_CFG:ESXi01_P0__STG_NodeA_P0;ESXi01_P0__STG_NodeB_P0;...
```

Zoning baza je u fajlu u čitljivom obliku — što znači i da se backup fajl može koristiti kao dokumentacija zoninga.

### Automatizacija

Backup se pokreće sa spoljnog servera preko SSH-a, uz autentifikaciju ključem:

```
#!/bin/bash
DATUM=$(date +%Y-%m-%d)
for SW in 10.10.10.11 10.10.10.12 10.10.20.11 10.10.20.12; do
  ssh -i /root/.ssh/san_key admin@$SW \
    "configupload -all -scp 10.10.20.30,backup,/backup/san/${SW}_${DATUM}.txt"
done
```

Zadržavajte bar tri meseca istorije. Greška u zoningu se ponekad primeti tek nedeljama kasnije.

---

## 9.3 Vraćanje konfiguracije

```
SAN_A_SW01:admin> switchdisable

SAN_A_SW01:admin> configdownload
Protocol (scp, ftp, sftp, local) [ftp]: scp
Server Name or IP Address [host]: 10.10.20.30
User Name [user]: backup
Path/Filename [<home dir>/config.txt]: /backup/san/SAN_A_SW01_2026-09-15.txt
Section (all|chassis|switch [all]): all

*** CAUTION ***

This command is used to download a backed-up configuration
for a specific switch. If using a file from a different
switch, this file's configuration settings will overwrite
the current switch settings.

Do you want to continue [y/n]: y
Activating configDownload: Switch is disabled
configDownload complete: All config parameters are downloaded

SAN_A_SW01:admin> switchenable
```

### Ograničenja

- **Zahteva `switchdisable`** — dakle prekid rada na tom switchu
- Fajl mora poticati sa **istog modela** uređaja
- FOS verzija u fajlu treba da odgovara verziji na uređaju
- Neka podešavanja (IP adresa, ime) zahtevaju restart da bi se primenila

### Vraćanje samo zoninga

Ako je problem isključivo u zoningu, ne mora se vraćati cela konfiguracija. Iz backup fajla se pročitaju definicije i ručno rekonstruišu, ili se koristi sekcija `switch`. Za manje izmene je često najbrže izvući relevantne linije iz fajla i ispraviti ciljanim komandama — bez `switchdisable`.

---

## 9.4 Nadogradnja firmware-a

### Pripreme

**1. Provera trenutne verzije i stanja particija**

```
SAN_A_SW01:admin> firmwareshow
Appl     Primary/Secondary Versions
------------------------------------------
FOS      v9.2.0c3
         v9.2.0c3
```

Obe particije su iste — uređaj je u stabilnom stanju i spreman za nadogradnju.

**2. Provera matrice kompatibilnosti**

Ciljna verzija mora biti podržana sa vašim storage nizom, HBA karticama i ostalim switchevima u fabrici. Merodavne su matrice proizvođača storage niza i OEM proizvođača switcha.

**3. Pravilo o preskakanju verzija**

Ne može se skočiti preko više većih verzija. Sa v8.2 na v9.2 ide se preko međukoraka. Put nadogradnje proverite u napomenama uz izdanje ciljne verzije.

**4. Backup i provera zdravlja**

```
SAN_A_SW01:admin> configupload -all -scp 10.10.20.30,backup,/backup/san/SW01_pre_fw.txt
SAN_A_SW01:admin> switchstatusshow
SAN_A_SW01:admin> errdump | tail -50
```

Ne nadograđujte uređaj koji već ima problem. Prvo rešite problem.

**5. Provera druge fabrike**

Nadogradnja se radi **na jednom switchu u jednom trenutku, u jednoj fabrici**. Pre početka potvrdite da hostovi imaju aktivne putanje kroz drugu fabriku:

```
[root@esxi-01:~] esxcli storage core path list | grep -c active
4
```

### Hot Code Load

Kod nadogradnje unutar iste veće verzije, Brocade podržava **non-disruptive** nadogradnju: FC saobraćaj se ne prekida, prekidaju se samo upravljačke sesije. Uslovi: uređaj mora biti u ispravnom stanju, obe particije iste, i nadogradnja ne sme preskakati veće verzije.

Kod prelaska na novu veću verziju, prekid je moguć. Planirajte prozor.

### Izvršavanje

```
SAN_A_SW01:admin> firmwaredownload
Server Name or IP Address: 10.10.20.30
User Name: fwuser
File Name: /firmware/v9.2.1a
Network Protocol(1-auto-select, 2-FTP, 3-SCP, 4-SFTP) [1]: 3
Password:

Server IP: 10.10.20.30, Protocol IPv4
Checking system settings for firmwaredownload...
System settings check passed.

You can run firmwaredownloadstatus to get the status of this command.

This command will cause a warm/non-disruptive boot on the switch,
but will require that existing telnet, secure telnet or SSH sessions
be restarted.

Do you want to continue [Y]: y

Firmware is being downloaded to the switch. This step may take up to 30 minutes.
```

Praćenje toka iz druge sesije:

```
SAN_A_SW01:admin> firmwaredownloadstatus
[1]: Tue Sep 15 12:02:11 2026
Firmware is being downloaded to the switch. This step may take up to 30 minutes.

[2]: Tue Sep 15 12:14:55 2026
Firmware has been downloaded to the secondary partition of the switch.

[3]: Tue Sep 15 12:16:03 2026
The firmware commit operation has started. This may take up to 10 minutes.

[4]: Tue Sep 15 12:24:47 2026
The commit operation has completed successfully.

[5]: Tue Sep 15 12:24:50 2026
Firmwaredownload command has completed successfully.
```

### Provera posle nadogradnje

```
SAN_A_SW01:admin> firmwareshow
Appl     Primary/Secondary Versions
------------------------------------------
FOS      v9.2.1a
         v9.2.1a

SAN_A_SW01:admin> switchshow | head -12
SAN_A_SW01:admin> fabricshow
SAN_A_SW01:admin> porterrshow
SAN_A_SW01:admin> cfgactvshow | head -5
```

Proverite i sa strane hosta da su sve putanje ponovo aktivne. **Tek posle toga** prelazite na drugi switch. Razmak od bar nekoliko sati, a idealno jedan radni dan, daje priliku da se problem pokaže dok druga fabrika još radi na staroj verziji.

### Vraćanje na prethodnu verziju

Dok `firmwarecommit` nije završen, moguće je vratiti se na sadržaj druge particije:

```
SAN_A_SW01:admin> firmwarerestore
```

Posle commit-a, obe particije sadrže novu verziju i povratak znači novu nadogradnju na staru verziju.

---

## 9.5 `supportsave`

Kompletan snimak stanja uređaja — logovi, konfiguracija, dijagnostika, interni podaci. Ovo je ono što podrška traži kada otvorite slučaj.

```
SAN_A_SW01:admin> supportsave
This command collects RASLOG, TRACE, supportShow, core file, FFDC data
and then transfers them to a FTP/SCP/SFTP server or a USB device.
This operation can take several minutes.

OK to proceed? (yes, y, no, n): [no] y

Host IP or Host Name: 10.10.20.30
User Name: backup
Protocol (ftp | sftp | scp): scp
Remote Directory: /support/SW01
Password:

Saving support information for chassis:SAN_A_SW01, module:RAS...
Saving support information for chassis:SAN_A_SW01, module:TRACE_OLD...
Saving support information for chassis:SAN_A_SW01, module:CORE_FFDC...
Saving support information for chassis:SAN_A_SW01, module:SSHOW_SYS...
...
SupportSave completed
```

Napomene:

- Traje nekoliko minuta i generiše desetine megabajta podataka
- Ne prekida saobraćaj, ali privremeno opterećuje procesor uređaja — izbegavajte u vršnim satima ako nije hitno
- **Uzmite ga dok problem traje.** Snimak napravljen posle restarta uređaja često više ne sadrži tragove uzroka.

Kada prijavljujete problem, uzmite `supportsave` sa **svih switcheva u pogođenoj fabrici**, ne samo sa onog na kojem ste primetili simptom.

---

## 9.6 Rutinske provere

Skup komandi koji pokriva nedeljni pregled:

```
SAN_A_SW01:admin> switchstatusshow
Switch Health Report                 Report time: 09/15/2026 13:02:11
Switch Name: SAN_A_SW01
SwitchState: HEALTHY

Contributing factors:
------------------------------------
PowerSupplies monitor           HEALTHY
Temperatures monitor            HEALTHY
Fans monitor                    HEALTHY
Marginal ports monitor          HEALTHY
Faulty ports monitor            HEALTHY
Error ports monitor             HEALTHY
```

```
SAN_A_SW01:admin> porterrshow
SAN_A_SW01:admin> errdump | tail -30
SAN_A_SW01:admin> psshow
SAN_A_SW01:admin> fanshow
SAN_A_SW01:admin> tempshow
SAN_A_SW01:admin> uptime
```

Mesečno, dodatno:

```
SAN_A_SW01:admin> sfpshow -all
SAN_A_SW01:admin> cfgsize
SAN_A_SW01:admin> licenseport --show
SAN_A_SW01:admin> firmwareshow
```

### Praćenje trenda optičkih vrednosti

Pojedinačna vrednost `Rx Power` govori malo; **trend govori sve**. Modul koji je pre šest meseci primao -2.4 dBm, a sada prima -6.8 dBm, degradira — i to će se pretvoriti u CRC greške pre nego u potpun prekid.

Zato mesečni `sfpshow -all` izlaz čuvajte u fajlu i upoređujte sa prethodnim. Ovo je jedina rutina iz celog poglavlja koja zaista predviđa kvar pre nego što se dogodi.

---

## 9.7 Zamena SFP-a ili kabla u produkciji

Postupak koji ne izaziva incident:

**1. Potvrdite da host ima drugu putanju**

```
[root@esxi-01:~] esxcli storage core path list | grep -A2 "naa.60060160" | grep State
   State: active
   State: active
```

**2. Ugasite port pre vađenja modula**

```
SAN_A_SW01:admin> portdisable 3
```

Ovo hostu daje uredan signal da putanja odlazi, umesto naglog nestanka svetla.

**3. Zamenite modul ili kabl**

**4. Uključite port i proverite**

```
SAN_A_SW01:admin> portenable 3
SAN_A_SW01:admin> sfpshow 3 | grep -E "Rx Power|Tx Power|Vendor PN"
SAN_A_SW01:admin> switchshow | grep "^   3"
SAN_A_SW01:admin> statsclear
```

**5. Posle nekoliko minuta pod opterećenjem**

```
SAN_A_SW01:admin> porterrshow | grep "^  3"
```

Nule potvrđuju da je zamena rešila problem.

---

## 9.8 Zamena celog switcha

Kada se uređaj menja zbog kvara, cilj je da ostatak fabrike ne primeti razliku.

Skica postupka:

1. Uzmite `configupload` sa starog uređaja, ako je još dostupan — inače koristite poslednji backup
2. Na novom uređaju postavite **isti FOS** kao na starom
3. Postavite osnovnu mrežnu konfiguraciju
4. Izvršite `configdownload` sa backup fajla starog uređaja
5. Proverite da su Domain ID, ime i zoning identični starom
6. Priključite kablove **istim redosledom** kao na starom uređaju
7. Proverite `fabricshow`, `switchshow`, `cfgactvshow` i stanje putanja na hostovima

Korak 6 je razlog zbog kojeg se portovi imenuju u trenutku priključivanja i zbog kojeg se vodi tabela kabliranja. Bez toga, zamena switcha postaje rekonstrukcija po sećanju.

---

## 9.9 Rezime poglavlja

- `configupload` pre svake izmene; automatizovano bar nedeljno, sa istorijom.
- Backup ne sadrži licence u svim slučajevima — vodite ih odvojeno, uz License ID uređaja.
- `configdownload` zahteva `switchdisable` i fajl sa istog modela.
- Firmware se nadograđuje na jednom switchu u jednom trenutku, uz prethodnu proveru da druga fabrika drži saobraćaj.
- Veće verzije se ne preskaču; put nadogradnje se proverava u napomenama uz izdanje.
- `supportsave` uzimajte **dok problem traje**, sa svih switcheva u pogođenoj fabrici.
- Mesečno poređenje `sfpshow` vrednosti je jedina rutina koja predviđa kvar unapred.
- Pre vađenja SFP-a ili kabla uvek prvo `portdisable`.

---

## 9.10 Provera znanja

1. Zašto se nadogradnja firmware-a ne radi istovremeno na oba switcha u fabrici, iako je proces deklarisan kao non-disruptive?
2. Backup fajl postoji, ali je sa drugog modela switcha. Možete li ga iskoristiti za `configdownload`?
3. Problem se desio juče, uređaj je u međuvremenu restartovan, a podrška traži `supportsave`. Šta je verovatan ishod?
4. `Rx Power` na portu je -6.9 dBm, link radi, nema grešaka. Da li je ovo razlog za intervenciju?
5. Koja je jedina komanda koju izvršavate pre nego što fizički izvučete SFP iz produkcijskog switcha, i zašto?

---

**Sledeće poglavlje:** Dijagnostika i troubleshooting — sistematski pristup, `porterrshow`, D_Port testovi, `fcping` i `pathinfo`, MAPS i kontrolna lista od fizičkog sloja do hosta.
