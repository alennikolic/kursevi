# Poglavlje 7: Zoniranje II — praksa

> **Cilj poglavlja:** Samostalno napraviti zoning od nule, izmeniti ga na živoj fabrici bez prekida rada, proveriti rezultat i prepoznati najčešće greške po poruci koju switch vraća.

---

## 7.1 Priprema — prikupljanje podataka

Nijedna komanda se ne kuca dok tabela nije popunjena. Ovo je korak koji se preskače i zbog kojeg nastaju greške koje se kasnije teško nalaze.

### Sa switcha

```
SAN_A_SW01:admin> nsshow
{
 Type Pid    COS     PortName                NodeName                 TTL(sec)
 N    010000;      3;21:00:f4:e9:d4:56:7a:b1;20:00:f4:e9:d4:56:7a:b1; na
    FC4s: FCP
    PortSymb: [39] "QLE2772 FW:v9.12.01 DVR:v5.2.11.0 ESXi-01"
    Port Index: 0
 N    010100;      3;21:00:f4:e9:d4:56:8c:11;20:00:f4:e9:d4:56:8c:11; na
    FC4s: FCP
    PortSymb: [39] "QLE2772 FW:v9.12.01 DVR:v5.2.11.0 ESXi-02"
    Port Index: 1
 N    010300;      3;50:06:01:60:88:60:2a:11;50:06:01:60:08:60:2a:11; na
    FC4s: FCP
    PortSymb: [28] "Storage-NodeA-Port0"
    Port Index: 3
 N    010400;      3;50:06:01:61:88:60:2a:12;50:06:01:60:08:60:2a:12; na
    FC4s: FCP
    PortSymb: [28] "Storage-NodeB-Port0"
    Port Index: 4
The Local Name Server has 4 entries }
```

Kraći pregled koji je dovoljan za popunjavanje tabele:

```
SAN_A_SW01:admin> switchshow | grep F-Port
   0   0   010000   id    N32     Online      FC  F-Port  21:00:f4:e9:d4:56:7a:b1
   1   1   010100   id    N32     Online      FC  F-Port  21:00:f4:e9:d4:56:8c:11
   3   3   010300   id    N32     Online      FC  F-Port  50:06:01:60:88:60:2a:11
   4   4   010400   id    N32     Online      FC  F-Port  50:06:01:61:88:60:2a:12
```

### Sa hosta

Na ESXi hostu, da biste bili sigurni koji WWPN pripada kom serveru:

```
[root@esxi-01:~] esxcfg-scsidevs -a
vmhba1  qlnativefc  link-up   fc.20000f4e9d4567ab1:21000f4e9d4567ab1
vmhba2  qlnativefc  link-up   fc.20000f4e9d4567ab2:21000f4e9d4567ab2
```

Na Linux hostu:

```
[root@srv01 ~]# cat /sys/class/fc_host/host*/port_name
0x21000024ff5a1b2c
0x21000024ff5a1b2d
```

### Radna tabela

| Uređaj | Port na switchu | WWPN | Alias |
|---|---|---|---|
| ESXi-01, HBA1 P0 | 0 | `21:00:f4:e9:d4:56:7a:b1` | `ESXi01_HBA1_P0` |
| ESXi-02, HBA1 P0 | 1 | `21:00:f4:e9:d4:56:8c:11` | `ESXi02_HBA1_P0` |
| Storage Node A P0 | 3 | `50:06:01:60:88:60:2a:11` | `STG_NodeA_P0` |
| Storage Node B P0 | 4 | `50:06:01:61:88:60:2a:12` | `STG_NodeB_P0` |

> Ovo je fabrika A. Drugi port svakog hosta i drugi portovi storage nodova su u fabrici B i zoniraju se posebno, na drugom switchu, sa sopstvenom tabelom.

---

## 7.2 Transakcioni model

Zoning je jedini deo FOS-a koji ne primenjuje izmene odmah. Postoje tri nivoa:

```
   Radna kopija (transaction)     ← komande koje kucate idu ovde
          │  cfgsave
          ▼
   Sačuvana baza (defined)        ← trajno zapisano, ali nije na snazi
          │  cfgenable
          ▼
   Aktivna konfiguracija (effective)  ← ono što fabric zaista sprovodi
```

Provera da li je transakcija otvorena:

```
SAN_A_SW01:admin> cfgtransshow
Current transaction token is 271987122
It is abortable
```

```
SAN_A_SW01:admin> cfgtransshow
There is no outstanding zoning transaction
```

Odustajanje od svih nesačuvanih izmena:

```
SAN_A_SW01:admin> cfgtransabort
```

**Dve stvari koje treba znati o transakcijama:**

1. **Otvorena transakcija preživljava odjavljivanje.** Ako izađete iz sesije bez `cfgsave` ili `cfgtransabort`, izmene ostaju u radnoj kopiji. Sledeći administrator ih zatiče.
2. **U fabrici može biti otvorena samo jedna transakcija.** Ako neko drugi radi izmene, vaša komanda vraća grešku. Ovo je zaštita, ne kvar.

Zato svaki rad na zoningu počinje sa `cfgtransshow` — da vidite da li je neko nešto ostavio nedovršeno.

---

## 7.3 Alias-i

```
SAN_A_SW01:admin> alicreate "ESXi01_HBA1_P0", "21:00:f4:e9:d4:56:7a:b1"
SAN_A_SW01:admin> alicreate "ESXi02_HBA1_P0", "21:00:f4:e9:d4:56:8c:11"
SAN_A_SW01:admin> alicreate "STG_NodeA_P0", "50:06:01:60:88:60:2a:11"
SAN_A_SW01:admin> alicreate "STG_NodeB_P0", "50:06:01:61:88:60:2a:12"
```

Provera:

```
SAN_A_SW01:admin> alishow
 alias: ESXi01_HBA1_P0
        21:00:f4:e9:d4:56:7a:b1
 alias: ESXi02_HBA1_P0
        21:00:f4:e9:d4:56:8c:11
 alias: STG_NodeA_P0
        50:06:01:60:88:60:2a:11
 alias: STG_NodeB_P0
        50:06:01:61:88:60:2a:12
```

Ostale operacije nad alias-om:

```
SAN_A_SW01:admin> aliadd "ESXi01_HBA1_P0", "21:00:f4:e9:d4:56:7a:c9"
SAN_A_SW01:admin> aliremove "ESXi01_HBA1_P0", "21:00:f4:e9:d4:56:7a:c9"
SAN_A_SW01:admin> alidelete "ESXi01_HBA1_P0"
```

Alias se ne može obrisati dok je član neke zone:

```
SAN_A_SW01:admin> alidelete "STG_NodeA_P0"
"STG_NodeA_P0" is in use in one or more zones
```

---

## 7.4 Zone

Po pravilu Single Initiator — Single Target, za dva hosta i dva storage porta u fabrici A dobijamo četiri zone:

```
SAN_A_SW01:admin> zonecreate "ESXi01_P0__STG_NodeA_P0", "ESXi01_HBA1_P0; STG_NodeA_P0"
SAN_A_SW01:admin> zonecreate "ESXi01_P0__STG_NodeB_P0", "ESXi01_HBA1_P0; STG_NodeB_P0"
SAN_A_SW01:admin> zonecreate "ESXi02_P0__STG_NodeA_P0", "ESXi02_HBA1_P0; STG_NodeA_P0"
SAN_A_SW01:admin> zonecreate "ESXi02_P0__STG_NodeB_P0", "ESXi02_HBA1_P0; STG_NodeB_P0"
```

Svaki host vidi oba storage kontrolera kroz ovu fabriku — to je ono što obezbeđuje da otkaz jednog kontrolera ne prekine pristup.

```
SAN_A_SW01:admin> zoneshow
 zone:  ESXi01_P0__STG_NodeA_P0
        ESXi01_HBA1_P0; STG_NodeA_P0
 zone:  ESXi01_P0__STG_NodeB_P0
        ESXi01_HBA1_P0; STG_NodeB_P0
 zone:  ESXi02_P0__STG_NodeA_P0
        ESXi02_HBA1_P0; STG_NodeA_P0
 zone:  ESXi02_P0__STG_NodeB_P0
        ESXi02_HBA1_P0; STG_NodeB_P0
```

Izmene:

```
SAN_A_SW01:admin> zoneadd "ESXi01_P0__STG_NodeA_P0", "STG_NodeA_P1"
SAN_A_SW01:admin> zoneremove "ESXi01_P0__STG_NodeA_P0", "STG_NodeA_P1"
SAN_A_SW01:admin> zonedelete "ESXi01_P0__STG_NodeB_P0"
```

---

## 7.5 Zone konfiguracija

```
SAN_A_SW01:admin> cfgcreate "PROD_FABA_CFG", "ESXi01_P0__STG_NodeA_P0; ESXi01_P0__STG_NodeB_P0"
SAN_A_SW01:admin> cfgadd "PROD_FABA_CFG", "ESXi02_P0__STG_NodeA_P0; ESXi02_P0__STG_NodeB_P0"
```

```
SAN_A_SW01:admin> cfgshow "PROD_FABA_CFG"
 cfg:   PROD_FABA_CFG
        ESXi01_P0__STG_NodeA_P0; ESXi01_P0__STG_NodeB_P0;
        ESXi02_P0__STG_NodeA_P0; ESXi02_P0__STG_NodeB_P0
```

---

## 7.6 Čuvanje i aktiviranje

### `cfgsave` — upis u trajnu bazu

```
SAN_A_SW01:admin> cfgsave
You are about to save the Defined zoning configuration. This
action will only save the changes on Defined configuration.
Any changes made on the Effective configuration will not
take effect until it is re-enabled.
Do you want to save the Defined zoning configuration only?  (yes, y, no, n): [no] y
Nothing changed: nothing to save, returning ...
Updating flash ...
```

Posle ove komande izmene su trajno zapisane i replicirane na sve switcheve u fabrici, ali **nisu na snazi**.

### `cfgenable` — aktiviranje

```
SAN_A_SW01:admin> cfgenable "PROD_FABA_CFG"
You are about to enable a new zoning configuration.
This action will replace the old zoning configuration with the
current configuration selected. If the update includes changes
to one or more traffic isolation zones, the update may result in
localized disruption to traffic on ports associated with
the traffic isolation zone changes
Do you want to enable 'PROD_FABA_CFG' configuration  (yes, y, no, n): [no] y
zone config "PROD_FABA_CFG" is in effect
Updating flash ...
```

`cfgenable` istovremeno čuva i aktivira — nije neophodno prethodno izvršiti `cfgsave` ako odmah aktivirate.

### Da li je aktiviranje prekid rada?

**Za postojeće zone koje se ne menjaju — ne.** Aktiviranje nove konfiguracije ne ruši uspostavljene sesije uređaja čije zone ostaju iste. Dodavanje novog hosta u fabriku ne dotiče postojeće hostove.

Šta ipak izaziva prekid:

- Uklanjanje zone — uređaji iz te zone trenutno gube međusobnu vidljivost
- Aktiviranje konfiguracije koja ne sadrži zonu koja je ranije postojala (ista stvar, drugim putem)
- `cfgdisable` — deaktivira sve

Zato se kod izmena uvek postavlja pitanje: **šta u novoj konfiguraciji nedostaje u odnosu na staru?** Ono što nedostaje je ono što će pasti.

### Deaktiviranje

```
SAN_A_SW01:admin> cfgdisable
You are about to disable zoning configuration. This
action will disable the old zoning configuration and
release the reserved buffer.
Do you want to disable zoning configuration?  (yes, y, no, n): [no] y
Updating flash ...
```

Ovo je komanda koja u produkciji prekida sav pristup podacima. Posle nje ponašanje fabrike određuje podrazumevana zona — pri `defzone --noaccess`, niko nikoga ne vidi.

---

## 7.7 Kompletan primer

Sve zajedno, redosledom kojim se radi:

```
SAN_A_SW01:admin> cfgtransshow
There is no outstanding zoning transaction

SAN_A_SW01:admin> alicreate "ESXi01_HBA1_P0", "21:00:f4:e9:d4:56:7a:b1"
SAN_A_SW01:admin> alicreate "ESXi02_HBA1_P0", "21:00:f4:e9:d4:56:8c:11"
SAN_A_SW01:admin> alicreate "STG_NodeA_P0", "50:06:01:60:88:60:2a:11"
SAN_A_SW01:admin> alicreate "STG_NodeB_P0", "50:06:01:61:88:60:2a:12"

SAN_A_SW01:admin> zonecreate "ESXi01_P0__STG_NodeA_P0", "ESXi01_HBA1_P0; STG_NodeA_P0"
SAN_A_SW01:admin> zonecreate "ESXi01_P0__STG_NodeB_P0", "ESXi01_HBA1_P0; STG_NodeB_P0"
SAN_A_SW01:admin> zonecreate "ESXi02_P0__STG_NodeA_P0", "ESXi02_HBA1_P0; STG_NodeA_P0"
SAN_A_SW01:admin> zonecreate "ESXi02_P0__STG_NodeB_P0", "ESXi02_HBA1_P0; STG_NodeB_P0"

SAN_A_SW01:admin> cfgcreate "PROD_FABA_CFG", "ESXi01_P0__STG_NodeA_P0; ESXi01_P0__STG_NodeB_P0; ESXi02_P0__STG_NodeA_P0; ESXi02_P0__STG_NodeB_P0"

SAN_A_SW01:admin> cfgshow
Defined configuration:
 cfg:   PROD_FABA_CFG
        ESXi01_P0__STG_NodeA_P0; ESXi01_P0__STG_NodeB_P0;
        ESXi02_P0__STG_NodeA_P0; ESXi02_P0__STG_NodeB_P0
 zone:  ESXi01_P0__STG_NodeA_P0
        ESXi01_HBA1_P0; STG_NodeA_P0
 zone:  ESXi01_P0__STG_NodeB_P0
        ESXi01_HBA1_P0; STG_NodeB_P0
 zone:  ESXi02_P0__STG_NodeA_P0
        ESXi02_HBA1_P0; STG_NodeA_P0
 zone:  ESXi02_P0__STG_NodeB_P0
        ESXi02_HBA1_P0; STG_NodeB_P0
 alias: ESXi01_HBA1_P0
        21:00:f4:e9:d4:56:7a:b1
 alias: ESXi02_HBA1_P0
        21:00:f4:e9:d4:56:8c:11
 alias: STG_NodeA_P0
        50:06:01:60:88:60:2a:11
 alias: STG_NodeB_P0
        50:06:01:61:88:60:2a:12

Effective configuration:
 no configuration in effect

SAN_A_SW01:admin> cfgenable "PROD_FABA_CFG"
Do you want to enable 'PROD_FABA_CFG' configuration  (yes, y, no, n): [no] y
zone config "PROD_FABA_CFG" is in effect
Updating flash ...
```

Pre `cfgenable`, obavezno pregledajte ceo `cfgshow` izlaz. To je poslednja tačka na kojoj greška ne košta ništa.

---

## 7.8 Provera rezultata

### Na switchu

```
SAN_A_SW01:admin> switchshow | grep -i zoning
zoning:         ON (PROD_FABA_CFG)
```

```
SAN_A_SW01:admin> cfgactvshow
Effective configuration:
 cfg:   PROD_FABA_CFG
 zone:  ESXi01_P0__STG_NodeA_P0
        21:00:f4:e9:d4:56:7a:b1
        50:06:01:60:88:60:2a:11
 zone:  ESXi01_P0__STG_NodeB_P0
        21:00:f4:e9:d4:56:7a:b1
        50:06:01:61:88:60:2a:12
```

### Validacija baze

```
SAN_A_SW01:admin> zone --validate
Defined configuration:
 cfg: PROD_FABA_CFG
  zone: ESXi01_P0__STG_NodeA_P0
        ESXi01_HBA1_P0; STG_NodeA_P0
  zone: ESXi02_P0__STG_NodeB_P0
       *ESXi02_HBA1_P0; STG_NodeB_P0

 ------------------------------------
 ~ - Invalid configuration
 * - Member does not exist
 # - Invalid RSCN member
```

Zvezdica pored člana znači da taj WWPN **trenutno nije prijavljen u fabrici**. To ne mora biti greška — server može biti ugašen. Ali ako je server upaljen, zvezdica govori da je WWPN pogrešno unet ili da uređaj nije završio prijavu.

Ovo je izuzetno korisna provera posle unosa novih zona: uobičajena greška je jedan pogrešan heksadecimalni znak u WWPN-u, a `zone --validate` je odmah otkriva.

### Na hostu

ESXi:

```
[root@esxi-01:~] esxcli storage core adapter rescan --all
[root@esxi-01:~] esxcli storage core path list | grep -c "fc."
4
```

Četiri putanje za jedan LUN kod dva porta i dva kontrolera — to je očekivan rezultat.

Linux:

```
[root@srv01 ~]# echo "- - -" > /sys/class/scsi_host/host1/scan
[root@srv01 ~]# multipath -ll
```

---

## 7.9 Rutinske izmene na živoj fabrici

### Dodavanje novog hosta

Ovo se radi u produkciji, bez prekida za postojeće hostove:

```
SAN_A_SW01:admin> cfgtransshow
There is no outstanding zoning transaction

SAN_A_SW01:admin> switchshow | grep "^   7"
   7   7   010700   id    N32     Online      FC  F-Port  21:00:f4:e9:d4:56:9d:21

SAN_A_SW01:admin> alicreate "ESXi03_HBA1_P0", "21:00:f4:e9:d4:56:9d:21"
SAN_A_SW01:admin> zonecreate "ESXi03_P0__STG_NodeA_P0", "ESXi03_HBA1_P0; STG_NodeA_P0"
SAN_A_SW01:admin> zonecreate "ESXi03_P0__STG_NodeB_P0", "ESXi03_HBA1_P0; STG_NodeB_P0"
SAN_A_SW01:admin> cfgadd "PROD_FABA_CFG", "ESXi03_P0__STG_NodeA_P0; ESXi03_P0__STG_NodeB_P0"

SAN_A_SW01:admin> zone --validate | grep "\*"
(nema izlaza — svi članovi su prijavljeni)

SAN_A_SW01:admin> cfgenable "PROD_FABA_CFG"
Do you want to enable 'PROD_FABA_CFG' configuration  (yes, y, no, n): [no] y
zone config "PROD_FABA_CFG" is in effect
```

Postojeće zone nisu dirane, pa postojeći hostovi ne primećuju ništa.

### Zamena HBA kartice

Menja se samo WWPN u alias-u:

```
SAN_A_SW01:admin> aliremove "ESXi01_HBA1_P0", "21:00:f4:e9:d4:56:7a:b1"
SAN_A_SW01:admin> aliadd "ESXi01_HBA1_P0", "21:00:f4:e9:d4:56:aa:77"
SAN_A_SW01:admin> cfgenable "PROD_FABA_CFG"
```

**`cfgenable` je ovde obavezan.** Bez njega izmena alias-a ostaje samo u sačuvanoj bazi, a efektivna konfiguracija i dalje sadrži stari WWPN — što je tačno onaj scenario iz prethodnog poglavlja.

### Uklanjanje hosta iz produkcije

```
SAN_A_SW01:admin> cfgremove "PROD_FABA_CFG", "ESXi03_P0__STG_NodeA_P0; ESXi03_P0__STG_NodeB_P0"
SAN_A_SW01:admin> cfgenable "PROD_FABA_CFG"
SAN_A_SW01:admin> zonedelete "ESXi03_P0__STG_NodeA_P0"
SAN_A_SW01:admin> zonedelete "ESXi03_P0__STG_NodeB_P0"
SAN_A_SW01:admin> alidelete "ESXi03_HBA1_P0"
SAN_A_SW01:admin> cfgsave
```

Redosled je bitan: prvo se zona vadi iz konfiguracije i konfiguracija aktivira, tek onda se briše zona i alias. Obrnut redosled ostavlja konfiguraciju koja pokazuje na nepostojeće objekte.

Pre uklanjanja, obavezno se uverite da host zaista više ne koristi storage.

---

## 7.10 Backup zoninga

Zoning baza je deo konfiguracije switcha i izvozi se zajedno sa njom:

```
SAN_A_SW01:admin> configupload
Protocol (scp, ftp, sftp, local) [ftp]: scp
Server Name or IP Address [host]: 10.10.20.30
User Name [user]: backup
Path/Filename [<home dir>/config.txt]: /backup/san/SAN_A_SW01_pre_izmene.txt
Section (all|chassis|switch [all]): all
Password:
configUpload complete: All selected config parameters are uploaded
```

**Uzmite backup pre svake netrivijalne izmene zoninga.** To je jedini brz način da se vratite na prethodno stanje ako nešto pođe naopako.

Vraćanje se radi sa `configdownload` i obrađuje se u poglavlju 9 — uz napomenu da ta komanda zahteva `switchdisable` i predstavlja prekid rada.

### Komanda koju treba znati da biste je izbegavali

```
SAN_A_SW01:admin> cfgclear
```

`cfgclear` briše **celu zoning bazu** — sve zone, sve alias-e, sve konfiguracije. Koristi se samo pri potpunoj rekonfiguraciji, sa prethodno uzetim backup-om i planiranim prekidom rada. Ne postoji „undo".

---

## 7.11 Najčešće greške i poruke

| Poruka / simptom | Uzrok | Rešenje |
|---|---|---|
| `zone not found` | Greška u imenu, ili zona nije kreirana | `zoneshow` za tačno ime |
| `"X" is in use in one or more zones` | Pokušaj brisanja alias-a koji je član zone | Prvo ukloniti iz zona |
| `Duplicate entry in the config` | Član je već u zoni | `zoneshow <zona>` za proveru |
| `A transaction is already in progress` | Drugi administrator ima otvorenu transakciju | Dogovoriti se; `cfgtransshow` |
| Izmena alias-a nije dala efekat | Nije izvršen `cfgenable` | `cfgenable <cfg>` |
| Zvezdica u `zone --validate` | WWPN nije prijavljen u fabrici | Proveriti da li je uređaj online i da li je WWPN tačan |
| Host vidi target, nema LUN-ova | Zoning je ispravan, problem je LUN masking | Storage strana |
| Host ne vidi ništa posle `cfgenable` | Zona nije uključena u aktivnu konfiguraciju | `cfgactvshow` |
| Sve je ispravno, host i dalje ne vidi | Nije urađen rescan na hostu | `esxcli storage core adapter rescan --all` |

### Dijagnostički redosled kada „host ne vidi storage"

1. Da li je uređaj u Name Serveru? → `nsshow`
2. Da li je zoning uključen i koja je konfiguracija aktivna? → `switchshow | grep zoning`
3. Da li se oba WWPN-a nalaze u istoj zoni **efektivne** konfiguracije? → `cfgactvshow`
4. Da li su WWPN-ovi tačni, bez zvezdice? → `zone --validate`
5. Da li je urađen rescan na hostu?
6. Da li je LUN dodeljen hostu na storage nizu?

Prvih pet koraka traju dva minuta i eliminišu oko 90% slučajeva.

---

## 7.12 Rezime poglavlja

- Svaki rad na zoningu počinje sa `cfgtransshow` — otvorena tuđa transakcija je česta i preživljava odjavljivanje.
- Redosled je: alias → zone → cfg → `cfgenable`.
- `cfgsave` čuva, `cfgenable` čuva i aktivira.
- Dodavanje zona ne prekida rad; uklanjanje zona prekida rad za uređaje u njima.
- Posle izmene alias-a obavezan je `cfgenable`.
- `zone --validate` otkriva pogrešno unete WWPN-ove pre nego što ih otkrije korisnik.
- Backup konfiguracije pre svake ozbiljnije izmene.
- `cfgclear` nema povratka.

---

## 7.13 Provera znanja

1. Otvorili ste sesiju, izvršili `cfgtransshow` i dobili token transakcije koju niste vi započeli. Šta radite?
2. Dodali ste novu zonu i izvršili `cfgsave`. Host ne vidi storage. Šta nedostaje?
3. Treba ukloniti server iz produkcije. Napišite redosled komandi i objasnite zašto se `cfgenable` izvršava pre `zonedelete`.
4. `zone --validate` pokazuje zvezdicu pored jednog člana, a server je upaljen i port je `Online`. Koja su dva moguća uzroka?
5. Zašto je dodavanje novog hosta u aktivnu konfiguraciju bezbedno po postojeće hostove, a uklanjanje zone nije?

---

**Sledeće poglavlje:** Proširenje fabrike — ISL veze, trunking, spajanje switcheva, fabric merge i razlozi segmentacije.
