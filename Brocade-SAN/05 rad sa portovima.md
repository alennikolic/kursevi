# Poglavlje 5: Rad sa portovima

> **Cilj poglavlja:** Znati kako se port konfiguriše, uključuje i isključuje, kako se postavlja i proverava brzina, koje se opcije zaključavanja koriste iz bezbednosnih razloga i kako se potvrđuje da je link zaista uspostavljen do kraja.

---

## 5.1 Pregled konfiguracije porta

U prethodnom poglavlju uvedena je razlika između stanja i podešavanja. Podešavanja se gledaju sa `portcfgshow`.

### Svi portovi odjednom

```
SAN_A_SW01:admin> portcfgshow
Ports of Slot 0    0   1   2   3   4   5   6   7
-----------------+---+---+---+---+---+---+---+---
Speed             AN  AN  AN  AN  AN  AN  AN  AN
Fill Word(On Active)  0   0   0   0   0   0   0   0
AL_PA Offset 13   ..  ..  ..  ..  ..  ..  ..  ..
Trunk Port        ON  ON  ON  ON  ON  ON  ON  ON
Long Distance     ..  ..  ..  ..  ..  ..  ..  ..
VC Link Init      ..  ..  ..  ..  ..  ..  ..  ..
Locked L_Port     ..  ..  ..  ..  ..  ..  ..  ..
Locked G_Port     ..  ..  ..  ..  ..  ..  ..  ..
Disabled E_Port   ..  ..  ..  ..  ..  ..  ..  ..
Locked E_Port     ..  ..  ..  ..  ..  ..  ..  ..
ISL R_RDY Mode    ..  ..  ..  ..  ..  ..  ..  ..
RSCN Suppressed   ..  ..  ..  ..  ..  ..  ..  ..
Persistent Disable..  ..  ..  ..  ..  ON  ..  ..
NPIV capability   ON  ON  ON  ON  ON  ON  ON  ON
QOS Port          AE  AE  AE  AE  AE  AE  AE  AE
Port Auto Disable ..  ..  ..  ..  ..  ..  ..  ..
Compression       ..  ..  ..  ..  ..  ..  ..  ..
Encryption        ..  ..  ..  ..  ..  ..  ..  ..
FEC               ON  ON  ON  ON  ON  ON  ON  ON
```

Dve tačke (`..`) znače da je opcija isključena, `ON` da je uključena, `AN` je auto-negotiation za brzinu. Odmah se vidi da je port 5 persistent disabled — to je podatak koji objašnjava zašto „port ne radi iako je kabl ispravan".

### Jedan port, sve opcije

```
SAN_A_SW01:admin> portcfgshow 3
Area Number:              3
Octet Speed Combo:        1(32G,16G,8G,4G,2G)
Speed Level:              AUTO(HW)
Trunk Port                ON
Long Distance             OFF
Locked L_Port             OFF
Locked G_Port             OFF
Disabled E_Port           OFF
Locked E_Port             OFF
Persistent Disable        OFF
NPIV capability           ON
NPIV PP Limit             126
QOS E_Port                AUTO
EX Port                   OFF
Mirror Port               OFF
Credit Recovery           ON
F_Port Buffers            OFF
D-Port mode               OFF
FEC                       ON
Fill Word(Current)        0(Idle-Idle)
Compression               OFF
Encryption                OFF
Port Auto Disable         OFF
Rate Limit                OFF
```

---

## 5.2 Uključivanje i isključivanje porta

### Obično isključivanje

```
SAN_A_SW01:admin> portdisable 5

SAN_A_SW01:admin> switchshow | grep "^   5"
   5   5   010500   id    N32     Offline     FC   Disabled
```

```
SAN_A_SW01:admin> portenable 5

SAN_A_SW01:admin> switchshow | grep "^   5"
   5   5   010500   id    N32     Online      FC  F-Port  21:00:f4:e9:d4:56:9a:01
```

Ovo stanje **ne preživljava restart switcha**. Posle ponovnog pokretanja port se vraća u uključeno stanje.

### Trajno isključivanje

```
SAN_A_SW01:admin> portcfgpersistentdisable 5
SAN_A_SW01:admin> portcfgpersistentenable 5
```

Persistent disable ostaje na snazi i posle restarta. Koristi se za portove koji se ne koriste, za portove na kojima se radi, i kao mera predostrožnosti u okruženjima gde je fizički pristup switchu nekontrolisan.

Razlika je važna u dijagnostici:

```
SAN_A_SW01:admin> portshow 5 | grep -i disable
portDisableReason: Persistently disabled port
```

Kada `portenable` ne pomogne, port je skoro sigurno persistent disabled — `portenable` ne poništava persistent stanje.

### Zašto se port sam isključio

```
SAN_A_SW01:admin> portshow 7 | grep -i portDisableReason
portDisableReason: No POD License
```

Mogući razlozi koje ćete videti:

| Razlog | Značenje |
|---|---|
| `None` | Port nije disabled |
| `Persistently disabled port` | Ručno trajno isključen |
| `No POD License` | Nema slobodne licence za port |
| `Port Fault` | Hardverski kvar detektovan pri dijagnostici |
| `Security Violation` | Prekršena bezbednosna politika (npr. neovlašćen WWN) |
| `Port Auto Disabled` | Automatski isključen zbog ponovljenih grešaka na linku |

### Isključivanje celog switcha

```
SAN_A_SW01:admin> switchdisable
SAN_A_SW01:admin> switchenable
```

Ovo gasi sve portove odjednom. Potrebno je za promenu Domain ID-a i neke druge operacije. U produkciji je to potpuni prekid saobraćaja na toj fabrici — planirajte ga i proverite da druga fabrika radi pre nego što ga izvršite.

---

## 5.3 Brzina porta

### Podrazumevano: auto-negotiation

```
SAN_A_SW01:admin> portcfgspeed 3 0
```

Nula znači auto. U `switchshow` izlazu brzina se prikazuje sa prefiksom `N` kada je dogovorena automatski:

```
   3   3   010300   id    N32     Online      FC  F-Port  ...
```

### Fiksiranje brzine

```
SAN_A_SW01:admin> portcfgspeed 3 16

SAN_A_SW01:admin> switchshow | grep "^   3"
   3   3   010300   id    16G     Online      FC  F-Port  ...
```

Bez prefiksa `N` — brzina je ručno postavljena.

**Kada fiksirati brzinu?** Retko. Auto-negotiation na FC-u radi pouzdano. Fiksiranje ima smisla samo kada se auto pregovaranje ponaša nestabilno sa konkretnim uređajem, ili kada proizvođač storage niza to izričito propisuje.

**Zamka fiksiranja:** brzina mora biti fiksirana na **obe strane** ili nijednoj. Fiksirano na jednoj a auto na drugoj strani često daje link koji ne dolazi online, ili dolazi pa pada.

### Octet Speed Combo

Portovi na Brocade switchevima su grupisani po osam (okteti), i grupa deli podržani skup brzina. Postavljanje neuobičajene brzine na jednom portu može uticati na ostalih sedam u istoj grupi. U `portcfgshow` izlazu to je red `Octet Speed Combo`. U praksi ovo dolazi do izražaja samo kada mešate vrlo stare i vrlo nove uređaje na istom switchu.

---

## 5.4 Zaključavanje tipa porta

Ove opcije su bezbednosna mera i sprečavaju da se switch ponaša na neželjen način ako neko priključi pogrešan uređaj.

### Zabrana E_Porta na host portovima

```
SAN_A_SW01:admin> portcfgeport 3 0
```

Nula znači da port ne sme postati E_Port. Ako neko u taj port priključi drugi switch, link se neće uspostaviti.

Zašto je ovo važno: neplanirano povezivanje dva switcha pokreće **fabric merge**. Ako se zoning konfiguracije razlikuju, rezultat može biti segmentacija fabrike ili, gore, spajanje dve fabrike koje su namerno bile razdvojene. Zabrana E_Porta na svim portovima gde se očekuju samo serveri i storage je jeftina zaštita.

```
SAN_A_SW01:admin> portcfgeport 3 1     # ponovo dozvoli
```

### Zaključavanje na G_Port

```
SAN_A_SW01:admin> portcfggport 3 1
```

Port ostaje generički i ne prelazi u F_Port. Koristi se izuzetno retko.

### Zaključavanje na L_Port

Arbitrated Loop nije podržan iznad 8G, pa ova opcija na modernim uređajima nema praktičnu primenu. Ostavite je isključenom.

---

## 5.5 NPIV

**N_Port ID Virtualization** omogućava da se na jednom fizičkom portu prijavi više WWPN-ova. Na Brocade switchevima je podrazumevano uključen.

```
SAN_A_SW01:admin> portcfgshow 3 | grep -i npiv
NPIV capability           ON
NPIV PP Limit             126
```

Gde je potreban:

- **Access Gateway** — switch u AG režimu prosleđuje više host prijava kroz jedan uplink port
- **Virtuelne mašine sa direktnim FC pristupom** — svaka VM dobija sopstveni WWPN
- **Blade šasije** sa ugrađenim FC modulima

Ako je AG uplink priključen na port sa isključenim NPIV-om, prijaviće se samo jedan uređaj, a ostali hostovi neće biti vidljivi. Simptom je zbunjujuć: link radi, jedan host vidi storage, ostali ne.

```
SAN_A_SW01:admin> portcfgnpivport 3 1     # uključi
SAN_A_SW01:admin> portcfgnpivport 3 0     # isključi
```

---

## 5.6 Imenovanje portova

Ovo je opcija koju većina administratora zanemari, a koja se najbrže isplati.

```
SAN_A_SW01:admin> portname 3 -n ESXi-01_HBA1_P1
SAN_A_SW01:admin> portname 0 -n ESXi-01_HBA1_P0
SAN_A_SW01:admin> portname 6 -n ISL_to_SAN_A_SW02

SAN_A_SW01:admin> portname 3
ESXi-01_HBA1_P1
```

Ime se pojavljuje u `portshow` izlazu i u grafičkim alatima:

```
SAN_A_SW01:admin> portshow 3 | head -3
portIndex:  3
portName:   ESXi-01_HBA1_P1
portHealth: HEALTHY
```

Imenujte port u trenutku kada u njega priključujete kabl. Šest meseci kasnije, kada treba ugasiti port zbog zamene servera, to je razlika između sigurnog i nesigurnog poteza.

---

## 5.7 Provera da je link stvarno uspostavljen

Link prolazi kroz nekoliko faza, i „port je zelen" ne znači da je sve završeno.

### Korak 1 — fizički sloj

```
SAN_A_SW01:admin> sfpshow 3 | grep -E "Rx Power|Tx Power"
Rx Power:    -2.4    dBm (575.1uW)
Tx Power:    -1.8    dBm (660.0uW)
```

### Korak 2 — stanje porta i tip

```
SAN_A_SW01:admin> switchshow | grep "^   3"
   3   3   010300   id    N32     Online      FC  F-Port  21:00:f4:e9:d4:56:7a:b1
```

`Online` i `F-Port` sa prikazanim WWPN-om znači da je FLOGI uspešno završen.

### Korak 3 — prijave na portu

```
SAN_A_SW01:admin> portloginshow 3
Type  PID     World Wide Name         credit df_sz cos
=======================================================
  fe  010300  21:00:f4:e9:d4:56:7a:b1     16  2048   c  scr=0x00000003
  ff  010300  21:00:f4:e9:d4:56:7a:b1     12  2048   c  d_id=FFFFFC
```

Tip `ff` je fabric login (FLOGI), `fe` je port login (PLOGI). Prisustvo oba znači da je uređaj kompletno ušao u fabric.

### Korak 4 — registracija u Name Serveru

```
SAN_A_SW01:admin> nsshow | grep -A2 21:00:f4:e9:d4:56:7a:b1
 N    010300;      3;21:00:f4:e9:d4:56:7a:b1;20:00:f4:e9:d4:56:7a:b1; na
    FC4s: FCP
    PortSymb: [39] "QLE2772 FW:v9.12.01 DVR:v5.2.11.0 ESXi-01"
```

Tek posle ovog koraka uređaj je spreman za zoniranje. Ako `switchshow` pokazuje `Online`, a uređaja nema u `nsshow`, prijava nije završena — proverite podešavanja HBA kartice, posebno režim veze (mora biti **Point To Point Only**, nikako Loop Preferred).

### Korak 5 — brojači grešaka

```
SAN_A_SW01:admin> statsclear
SAN_A_SW01:admin> porterrshow | grep "^  3"
  3:  1.2m   3.4m     0      0      0      0      0      0     0     0      0      0      0      0      0
```

Nule posle resetovanja, pod opterećenjem, znače zdrav link.

---

## 5.8 Baferi i krediti

Buffer-to-Buffer krediti su mehanizam kojim FC obezbeđuje da pošiljalac nikada ne pošalje više okvira nego što primalac može da prihvati. Svaki kredit odgovara jednom baferu za jedan okvir.

```
SAN_A_SW01:admin> portbuffershow
User  Port Lx Max/Resv  Buffer   Needed    Link   Remaining
Port  Type Mode Buffers Usage    Buffers Distance Buffers
----  ---- ---- ------- ------   ------- -------- ---------
  0     F    -     8      8         -        -      3960
  1     F    -     8      8         -        -
  2     F    -     8      8         -        -
  3     F    -     8      8         -        -
  6     E    -    64     64         -        -
```

Za kratke veze unutar data centra podrazumevane vrednosti su uvek dovoljne. Krediti postaju tema kod veza na velikim udaljenostima, gde okvir dugo putuje kroz vlakno i pošiljalac ostaje bez kredita pre nego što stignu potvrde.

```
SAN_A_SW01:admin> portcfglongdistance 6 LS 1 100
```

Ovo se detaljnije obrađuje u poglavlju o proširenju fabrike. Za sada je dovoljno zapamtiti simptom: **link radi, ali propusnost je mnogo manja od očekivane, a `tim_txcrd_z` raste** — to je nedostatak kredita, ne kvar.

---

## 5.9 Vraćanje porta na podrazumevane vrednosti

Kada se na portu nakupe podešavanja iz prošlih instalacija, a ne znate koja su sve menjana:

```
SAN_A_SW01:admin> portcfgdefault 5
```

Ovo poništava sva `portcfg` podešavanja na tom portu i vraća ga u fabričko stanje. Korisno pre nego što se u port priključi novi uređaj. Pažnja: poništava i persistent disable, pa port može odmah doći online.

---

## 5.10 Tipični problemi sa portovima

| Simptom | Verovatan uzrok | Provera |
|---|---|---|
| `No_Module` | SFP nije ubačen ili nije prepoznat | `sfpshow <port>` |
| `No_Light` | Nema signala sa druge strane | Kabl, slot na serveru, druga strana ugašena |
| `No_Sync` | Signal stiže, ali nema sinhronizacije | Neusklađena brzina, prljav konektor |
| Port zaglavljen kao `G_Port` | Pregovaranje ne uspeva | Režim veze na HBA, brzina, `portcfgeport` |
| `portenable` ne pomaže | Persistent disable | `portshow <port> \| grep Disable` |
| Port disabled bez razloga | Nema POD licence | `licenseport --show` |
| Online ali nema u `nsshow` | FLOGI nije završen | HBA u Loop režimu, driver, firmware HBA |
| Link pada i diže se | Loš kabl, SFP ili konektor | `porterrshow`, `sfpshow`, D_Port test |

---

## 5.11 Rezime poglavlja

- `portdisable` je privremeno, `portcfgpersistentdisable` preživljava restart — i `portenable` ga ne poništava.
- `portDisableReason` iz `portshow` izlaza odmah kaže zašto je port ugašen.
- Auto-negotiation je podrazumevano i skoro uvek ispravno rešenje; ako fiksirate brzinu, fiksirajte je na obe strane.
- Zabrana E_Porta (`portcfgeport <port> 0`) na host portovima sprečava neplanirano spajanje fabrika.
- NPIV mora biti uključen na portovima ka Access Gateway uređajima i blade šasijama.
- Imenujte portove pri priključivanju kabla, ne kasnije.
- Link je potpuno uspostavljen tek kada se uređaj pojavi u `nsshow`, a ne kada port postane `Online`.

---

## 5.12 Provera znanja

1. Port 12 je `Offline` i `Disabled`. `portenable 12` prolazi bez greške, ali port ostaje ugašen. Šta je najverovatnije i kako to proveravate?
2. Novi server je priključen, `switchshow` pokazuje `Online` i `F-Port`, ali u `nsshow` ga nema. Gde tražite uzrok?
3. Zašto biste na portovima ka serverima isključili mogućnost da postanu E_Port?
4. Kolega je fiksirao brzinu na 32G na switch portu, dok je na storage portu ostavljeno auto. Link je nestabilan. Objasnite zašto.
5. Hostovi iza blade šasije se ne vide u fabrici, iako je uplink port online i jedan WWPN je prijavljen. Koju opciju proveravate prvo?

---

**Sledeće poglavlje:** Zoniranje I — koncepti, alias, zone i zone konfiguracije, WWPN naspram port zoninga i pravilo single-initiator/single-target.
