# Poglavlje 3: Prvi pristup i inicijalna konfiguracija

> **Cilj poglavlja:** Dovesti switch iz fabričkog stanja do tačke u kojoj ima ispravno ime, IP adresu, tačno vreme, jedinstven Domain ID i kontrolisan pristup — pre nego što se na njega priključi ijedan uređaj.

---

## 3.1 Načini pristupa switchu

| Pristup | Kada se koristi |
|---|---|
| **Serijska konzola** | Prvo puštanje u rad, oporavak, kada mrežni pristup ne radi |
| **SSH** | Svakodnevni rad |
| **Web Tools (HTTPS)** | Grafički pregled, povremena upotreba |
| **REST API** | Automatizacija i integracija sa alatima |
| **SANnav** | Centralizovano upravljanje većim brojem switcheva |

Ovaj kurs se drži CLI pristupa. Sve što se može uraditi kroz grafiku može se uraditi i kroz komandnu liniju, dok obrnuto ne važi.

### Serijska konzola

Parametri su fiksni i isti na svim generacijama:

```
Brzina:     9600 bps
Bitovi:     8
Parnost:    None
Stop bit:   1
Flow ctrl:  None
```

Kabl je RJ-45 na DB-9 ili RJ-45 na USB, isporučuje se uz uređaj.

### Podrazumevani mrežni pristup

Novi switch iz fabrike ima statičku adresu:

```
IP:      10.77.77.77
Maska:   255.255.255.0
```

Laptop se privremeno podesi na istu podmrežu i pristup je moguć preko SSH. Ako je switch već korišćen, ova adresa ne važi i konzola je jedini siguran put.

---

## 3.2 Prvo prijavljivanje

Fabrički nalozi i lozinke:

| Nalog | Lozinka | Uloga |
|---|---|---|
| `admin` | `password` | Puna administracija |
| `user` | `password` | Samo pregled |
| `root` | — | Zaključan, koristi se samo uz podršku |
| `factory` | — | Rezervisan za proizvođača |

Pri prvom prijavljivanju sistem traži promenu lozinke:

```
login as: admin
admin@10.77.77.77's password:

Please change your passwords now.
Use Control-C to exit or press 'Enter' key to proceed.

Password was not changed. Will prompt again at next login
until password is changed.

swd77:admin>
```

Promena lozinke naknadno:

```
swd77:admin> passwd admin
Changing password for admin
Enter new password:
Re-type new password:
passwd: all authentication tokens updated successfully
Saving password to stable storage.
Password saved to stable storage successfully.
```

> Lozinke se čuvaju odvojeno od konfiguracije switcha. `configupload` ih **ne** uključuje u backup u čitljivom obliku — nemojte računati da ćete ih povratiti iz backup fajla.

---

## 3.3 Mrežna konfiguracija

### Trenutno stanje

```
swd77:admin> ipaddrshow
SWITCH
Ethernet IP Address: 10.77.77.77
Ethernet Subnetmask: 255.255.255.0
Gateway IP Address: 0.0.0.0
DHCP: Off
IPv6 Autoconfiguration Enabled: Yes
Local IPv6 Addresses:
IPv6 Gateways:
```

### Postavljanje adrese

Komanda `ipaddrset` je interaktivna. Prazan unos zadržava postojeću vrednost:

```
swd77:admin> ipaddrset
Ethernet IP Address [10.77.77.77]: 10.10.10.11
Ethernet Subnetmask [255.255.255.0]: 255.255.255.0
Gateway IP Address [0.0.0.0]: 10.10.10.1
DHCP [Off]:
IPv6 Autoconfiguration Enabled [Yes]: No
Committing configuration...Done.
```

Ako ste povezani preko SSH, sesija će pući u trenutku promene adrese. Zato se inicijalno adresiranje radi sa konzole.

### Provera

```
swd77:admin> ipaddrshow
SWITCH
Ethernet IP Address: 10.10.10.11
Ethernet Subnetmask: 255.255.255.0
Gateway IP Address: 10.10.10.1
DHCP: Off

swd77:admin> ping 10.10.10.1
PING 10.10.10.1 (10.10.10.1): 56 data bytes
64 bytes from 10.10.10.1: icmp_seq=0 ttl=64 time=0.412 ms
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.388 ms
--- 10.10.10.1 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.388/0.400/0.412/0.012 ms
```

### Brzina i duplex management porta

```
swd77:admin> ifmodeshow eth0
Link mode: negotiated 1000baseT/Full, flow control: none

swd77:admin> ifmodeset eth0
Auto-negotiate (yes, y, no, n): [yes]
Committing configuration...done.
```

Ostavite auto-negotiation osim ako mrežni tim izričito ne traži drugačije.

### DNS

```
swd77:admin> dnsconfig

Enter option
  1 Display Domain Name Service (DNS) configuration
  2 Set DNS configuration
  3 Remove DNS configuration
  4 Quit
Select an item: (1..4) [4] 2

Enter Domain Name: [] domain.local
Enter Name Server IP address in dot notation: [] 10.10.5.10
Enter Name Server IP address in dot notation: [] 10.10.5.11
DNS configuration changed successfully
```

DNS nije obavezan za rad switcha, ali je potreban ako koristite FQDN za syslog, NTP ili LDAP servere.

---

## 3.4 Identitet switcha

### Ime switcha

Ime je ono što vidite u promptu i u `fabricshow` izlazu na svim ostalim switchevima u fabrici. Fabrički podrazumevano ime je kod svih uređaja isto, pa dva nova switcha u istoj fabrici izgledaju identično — što je recept za grešku.

```
swd77:admin> switchname SAN_A_SW01
Committing configuration...
Done.
Switch name has been changed.Please re-login into the switch for the change to be applied.

swd77:admin> exit
```

Posle ponovnog prijavljivanja:

```
SAN_A_SW01:admin>
```

Dozvoljeni su slova, cifre, crtica i donja crta; ime počinje slovom, do 30 znakova. Razmaci nisu dozvoljeni.

**Preporuka za konvenciju imenovanja:** ugradite u ime lokaciju, fabriku i redni broj, na primer `BG_FABA_SW01`. Kada u tri sata ujutru gledate `fabricshow` izlaz, ime treba da vam kaže gde je uređaj fizički i u kojoj je fabrici.

### Ime šasije i fabrike

```
SAN_A_SW01:admin> chassisname SAN_A_CH01
Committing configuration...
Done.

SAN_A_SW01:admin> fabricname --set FABRIC_A
Fabric Name set to FABRIC_A

SAN_A_SW01:admin> fabricname --show
Fabric Name: FABRIC_A
```

Ime fabrike je informativno, ali pomaže kod otklanjanja zabune u okruženjima sa više fabrika.

### Domain ID

Ovo je najosetljiviji korak u inicijalnoj konfiguraciji, jer **zahteva da switch bude isključen**. Ako je switch u produkciji, ovo je planirani prekid rada.

```
SAN_A_SW01:admin> switchdisable

SAN_A_SW01:admin> configure

Configure...

  Fabric parameters (yes, y, no, n): [no] y

    Domain: (1..239) [1] 1
    WWN Based persistent PID (yes, y, no, n): [no]
    Allow XISL Use (yes, y, no, n): [no]
    R_A_TOV: (4000..120000) [10000]
    E_D_TOV: (1000..5000) [2000]
    WAN_TOV: (0..30000) [0]
    MAX_HOPS: (7..19) [7]
    Data field size: (256..2112) [2112]
    Sequence Level Switching: (0..1) [0]
    Disable Device Probing: (0..1) [0]
    Suppress Class F Traffic: (0..1) [0]
    Per-frame Route Priority: (0..1) [0]
    Long Distance Fabric: (0..1) [0]
    BB credit: (1..27) [16]
    Disable FID Check (yes, y, no, n): [no]
    Insistent Domain ID Mode (yes, y, no, n): [no]

  Virtual Channel parameters (yes, y, no, n): [no] n
  F-Port login parameters (yes, y, no, n): [no] n
  Zoning Operation parameters (yes, y, no, n): [no] n
  RSCN Transmission Mode (yes, y, no, n): [no] n
  Arbitrated Loop parameters (yes, y, no, n): [no] n
  System services (yes, y, no, n): [no] n
  Portlog events enable (yes, y, no, n): [no] n
  ssl attributes (yes, y, no, n): [no] n
  http attributes (yes, y, no, n): [no] n
  snmp attributes (yes, y, no, n): [no] n
  rpcd attributes (yes, y, no, n): [no] n
  cfgload attributes (yes, y, no, n): [no] n
  webtools attributes (yes, y, no, n): [no] n

SAN_A_SW01:admin> switchenable
```

Na drugom switchu u istoj fabrici, isti postupak sa `Domain: 2`.

Ostale parametre iz ovog dijaloga **ne dirajte** dok tačno ne znate šta rade. `E_D_TOV` i `R_A_TOV` moraju biti identični na svim switchevima u fabrici — različite vrednosti su jedan od klasičnih uzroka segmentacije fabrike.

Provera:

```
SAN_A_SW01:admin> switchshow | grep -i domain
switchDomain:   1
```

> **Iz prakse:** dva nova switcha iz kutije oba imaju Domain ID 1 i isto ime. Ako ih tako povežete ISL kablom, fabric se neće formirati — link će se segmentirati. Promenite ime i Domain ID pre nego što povežete bilo šta.

---

## 3.5 Vreme i vremenska zona

Tačno vreme nije kozmetika. Svaki log unos, svaki `porterrshow` snimak i svaki supportsave se tumače kroz vremenske oznake — a kada se problem analizira zajedno sa logovima sa hosta i storage-a, razlika u satu čini korelaciju nemogućom.

```
SAN_A_SW01:admin> date
Tue Sep 15 08:14:22 UTC 2026

SAN_A_SW01:admin> tstimezone Europe/Belgrade
System Time Zone change will take effect at next reboot

SAN_A_SW01:admin> tstimezone
Time Zone : Europe/Belgrade
```

### NTP

```
SAN_A_SW01:admin> tsclockserver 10.10.5.20
Updating Clock Server configuration...done.
Updated with the NTP servers

SAN_A_SW01:admin> tsclockserver
10.10.5.20
```

Više servera se navodi pod navodnicima, razdvojeni razmakom:

```
SAN_A_SW01:admin> tsclockserver "10.10.5.20 10.10.5.21"
```

Provera da je sinhronizacija zaista prihvaćena:

```
SAN_A_SW01:admin> date
Tue Sep 15 10:14:31 CEST 2026
```

Ako `tsclockserver` vrati `LOCL`, switch koristi sopstveni sat i NTP nije aktivan.

---

## 3.6 Korisnici i uloge

FOS ima ugrađene uloge sa unapred definisanim opsegom prava:

| Uloga | Šta može |
|---|---|
| `root` | Sve; rezervisano za podršku |
| `admin` | Puna administracija switcha |
| `switchadmin` | Administracija switcha bez upravljanja korisnicima |
| `fabricadmin` | Administracija switcha i zoninga, bez korisnika |
| `zoneadmin` | Samo zoning |
| `operator` | Rutinske operacije, bez izmena konfiguracije |
| `securityadmin` | Bezbednosne postavke i korisnici |
| `user` | Samo pregled |

### Pregled naloga

```
SAN_A_SW01:admin> userconfig --show -a
Account name: admin
Description: Administrator
Enabled: Yes
Password Last Change Date: Mon Sep 14 2026
Password Expiration Date: Not Applicable
Locked: No
Home LF Role: admin
Role-LF List: admin: 1-128
Chassis Role: admin

Account name: user
Description: User
Enabled: Yes
Locked: No
Home LF Role: user
Role-LF List: user: 1-128
```

### Kreiranje naloga

```
SAN_A_SW01:admin> userconfig --add anikolic -r fabricadmin -d "Alen Nikolic - SAN admin"
Setting initial password for anikolic
Enter new password:
Re-type new password:
Account anikolic has been successfully added.
```

Ostale operacije:

```
SAN_A_SW01:admin> userconfig --change anikolic -r admin
SAN_A_SW01:admin> userconfig --disable anikolic
SAN_A_SW01:admin> userconfig --delete anikolic
```

**Preporuka:** napravite imenovane naloge za svakog administratora i koristite njih. Zajednički `admin` nalog znači da u logu ne postoji trag ko je šta uradio.

### Politika lozinki

```
SAN_A_SW01:admin> passwdcfg --show
passwdcfg configuration parameters:
      MinLength: 8
      LowerCase: 0
      UpperCase: 0
          Digits: 0
     Punctuation: 0
          History: 1
MinPasswordAge: 0
MaxPasswordAge: 0
      Warning: 0
      LockoutThreshold: 0
      LockoutDuration: 30
      AdminLockout: 0
      RepeatCharLimit: 1
      SequenceLength: 1
      ReverseUserID: 0
      HashType: sha256

SAN_A_SW01:admin> passwdcfg --set -minlength 12 -uppercase 1 -digits 1 -punctuation 1 -history 5
```

### Centralizovana autentifikacija

U okruženjima sa AD ili RADIUS serverom, lokalni nalozi ostaju kao rezervni put:

```
SAN_A_SW01:admin> aaaconfig --show
Primary AAA Service: Local
Secondary AAA Service: None
```

Konfiguracija LDAP-a i RADIUS-a prelazi okvire ovog kursa, ali zapamtite pravilo: **uvek zadržite bar jedan funkcionalan lokalni nalog.** Ako AAA server postane nedostupan, a lokalni pristup nije ostavljen, ostajete bez pristupa switchu.

---

## 3.7 Osnovno obezbeđivanje pristupa

```
SAN_A_SW01:admin> ipfilter --show
Name: default_ipv4, Type: ipv4, State: active
Rule    Source IP    Protocol    Dest Port    Action
1       any          tcp         22           permit
2       any          tcp         23           permit
3       any          tcp         897          permit
4       any          tcp         898          permit
5       any          tcp         111          permit
6       any          tcp         80           permit
7       any          tcp         443          permit
```

Telnet (port 23) je u podrazumevanom skupu pravila. U produkciji ga treba ukloniti i ostaviti isključivo SSH. Rad sa `ipfilter` pravilima podrazumeva kreiranje kopije politike, izmenu i aktivaciju — pogrešan redosled može da vas zaključa van uređaja, pa se radi sa konzolnog pristupa.

Pregled aktivnih sesija:

```
SAN_A_SW01:admin> ssh --show
SSH sessions:
  admin    10.10.20.55    Tue Sep 15 10:12:04 2026
```

---

## 3.8 Verifikacija i prvi backup

Kada je inicijalna konfiguracija gotova, proverite celinu:

```
SAN_A_SW01:admin> switchshow
switchName:     SAN_A_SW01
switchType:     183.0
switchState:    Online
switchMode:     Native
switchRole:     Principal
switchDomain:   1
switchId:       fffc01
switchWwn:      10:00:38:ba:b0:fc:9f:b0
zoning:         OFF
switchBeacon:   OFF
Fabric Name:    FABRIC_A
```

Kontrolna lista pre nego što se priključi prvi uređaj:

- [ ] Ime switcha postavljeno po konvenciji
- [ ] IP adresa, maska i gateway podešeni, gateway odgovara na ping
- [ ] Domain ID jedinstven u fabrici
- [ ] Vremenska zona i NTP podešeni, `date` prikazuje tačno lokalno vreme
- [ ] Lozinka fabričkih naloga promenjena
- [ ] Imenovani administratorski nalozi napravljeni
- [ ] Licence proverene sa `licenseport --show`
- [ ] Firmware verzija zabeležena (`firmwareshow`)
- [ ] Konfiguracija izvezena na spoljni server

Poslednji korak, u detalje obrađen u poglavlju 9:

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

Uzmite backup **odmah po inicijalnoj konfiguraciji**, pre zoninga. To je vaša čista polazna tačka.

---

## 3.9 Rezime poglavlja

- Inicijalnu konfiguraciju radite sa serijske konzole — promena IP adrese prekida SSH sesiju.
- Fabrički switch je na `10.77.77.77`, sa nalogom `admin` / `password`.
- Promena Domain ID-a zahteva `switchdisable`, dakle prekid rada — uradite je pre nego što uređaj uđe u produkciju.
- `E_D_TOV` i `R_A_TOV` moraju biti isti na svim switchevima u fabrici.
- Tačno vreme je preduslov za svaku ozbiljnu analizu logova.
- Imenovani nalozi umesto zajedničkog `admin` naloga; uvek zadržite lokalni nalog kao rezervu.
- Backup konfiguracije uzmite pre nego što počnete sa zoningom.

---

## 3.10 Provera znanja

1. Zašto se inicijalno IP adresiranje radi sa konzole, a ne preko SSH?
2. Koje su dve stvari koje morate promeniti na dva nova switcha pre nego što ih povežete u istu fabriku?
3. Kolegi je potrebno da samostalno kreira zone, ali ne želite da može da menja mrežnu konfiguraciju switcha. Koju ulogu mu dodeljujete?
4. `tsclockserver` vraća `LOCL`. Šta to znači i zašto je problem?
5. Konfigurisali ste LDAP autentifikaciju i obrisali sve lokalne naloge osim `admin`, koji ste takođe prebacili na LDAP. Koji je rizik?

---

**Sledeće poglavlje:** FOS CLI — snalaženje u komandnoj liniji, struktura komandi, pomoć, filtriranje izlaza i komande za svakodnevni pregled stanja.
