# Poglavlje 10: Dijagnostika i troubleshooting

> **Cilj poglavlja:** Povezati sve alate iz prethodnih poglavlja u jedan sistematičan postupak, savladati napredne dijagnostičke komande i imati gotove recepte za scenarije koji se u praksi najčešće javljaju.

---

## 10.1 Metodologija

FC problemi se rešavaju po slojevima, odozdo nagore. Preskakanje slojeva je najčešći razlog zbog kojeg se dijagnostika pretvori u zamenu delova nasumice.

```
   5. Aplikacija / OS          ← LUN, multipath, datastore
   4. Zoning                   ← ko koga sme da vidi
   3. Fabric                   ← Name Server, rutiranje, segmentacija
   2. Link                     ← FLOGI, brzina, tip porta
   1. Fizički sloj             ← SFP, kabl, snaga signala
```

Četiri pravila koja se isplate:

**1. Uvek počnite od fizičkog sloja.** Tri komande (`switchshow`, `sfpshow`, `porterrshow`) eliminišu većinu uzroka za dva minuta. Preskakanje ovog koraka znači da ćete kasnije satima tražiti grešku u zoningu koja ne postoji.

**2. Menjajte jednu stvar u jednom trenutku.** Ako zamenite kabl i SFP istovremeno i problem nestane, ne znate šta je bio uzrok — a isti kvar će se vratiti.

**3. Sistematski eliminišite hipoteze.** Zapišite šta ste proverili i kako ste isključili svaku mogućnost. Kod problema koji traju danima, to je razlika između napretka i vrtenja u krug.

**4. Radite na jednoj fabrici.** Pre svake intervencije potvrdite da druga fabrika drži saobraćaj. Ako obe imaju problem, prvo stabilizujte jednu.

---

## 10.2 Prvih pet minuta

Kada stigne prijava, ovaj skup komandi daje sliku stanja:

```
SAN_A_SW01:admin> switchshow
SAN_A_SW01:admin> switchstatusshow
SAN_A_SW01:admin> fabricshow
SAN_A_SW01:admin> porterrshow
SAN_A_SW01:admin> errdump | tail -40
SAN_A_SW01:admin> cfgactvshow | head -5
```

Šta tražite u svakom izlazu:

| Komanda | Signal za uzbunu |
|---|---|
| `switchshow` | Portovi u `No_Light`, `No_Sync`, `Disabled`, `Segmented` |
| `switchstatusshow` | Bilo šta osim `HEALTHY` |
| `fabricshow` | Manji broj switcheva nego što treba |
| `porterrshow` | Brojači koji nisu nula na portovima u upotrebi |
| `errdump` | Poruke nivoa ERROR i WARNING iz vremena incidenta |
| `cfgactvshow` | Neočekivano ime konfiguracije ili prazan izlaz |

---

## 10.3 Fizički sloj

### Merenje snage na oba kraja

```
SAN_A_SW01:admin> sfpshow 3 | grep -E "Rx Power|Tx Power|Wavelength|Vendor PN"
Vendor PN:   57-1000335-01
Wavelength:  850  (units nm)
Rx Power:    -2.4    dBm (575.1uW)
Tx Power:    -1.8    dBm (660.0uW)
```

Tumačenje kombinacija:

| Tx lokalno | Rx lokalno | Zaključak |
|---|---|---|
| Normalan | Normalan | Fizički sloj je u redu — tražite više |
| Normalan | -40 dBm | Ne stiže svetlo: kabl, druga strana, pogrešan slot |
| Normalan | -10 do -13 dBm | Preveliki gubitak: prljav konektor, predugačak kabl |
| Nema podataka | Nema podataka | Nema SFP-a ili nije prepoznat |
| Vrlo nizak Tx | — | Kvar lasera — zameniti modul |

**Obe strane emituju, nijedna ne prima** je poseban obrazac: to je zamenjen polaritet (Tx/Rx) negde u putanji, ili su kablovi priključeni u pogrešne portove na jednoj strani.

### Brojači grešaka — merenje umesto istorije

```
SAN_A_SW01:admin> statsclear
SAN_A_SW01:admin> slotstatsclear
```

Sačekajte pod opterećenjem, pa:

```
SAN_A_SW01:admin> porterrshow
          frames      enc    crc    crc    too   too    bad   enc   disc   link   loss   loss
       tx     rx      in    err    g_eof  shrt   long   eof   out   c3     fail   sync   sig
  3:  412m   88m     31     28     28      0      0      0    12     0      0      0      0
```

Brojači koji rastu posle resetovanja znače aktivan problem. Obratite pažnju na odnos `crc err` i `crc g_eof`: kada su oba uvećana, greška je nastala na **ovom** linku.

### Redosled intervencije na fizičkom sloju

1. Očistite oba konektora namenskim alatom i ponovo utaknite
2. Ponovo izmerite i resetujte brojače
3. Zamenite patch kabl
4. Zamenite SFP na strani koja lošije prima
5. Tek onda posumnjajte na port na switchu

Preko 70% problema se reši u prva dva koraka.

---

## 10.4 D_Port — dijagnostički port

Ovo je najmoćniji alat za fizički sloj koji Brocade nudi. Port se privremeno prebacuje u dijagnostički režim, ne prenosi podatke, i izvodi seriju testova: električnu petlju, optičku petlju i test saobraćaja. Rezultat daje merenja koja se drugačije ne mogu dobiti — dužinu kabla, kašnjenje i gubitak snage po smeru.

### Pokretanje

```
SAN_A_SW01:admin> portdisable 3
SAN_A_SW01:admin> portcfgdport --enable 3
SAN_A_SW01:admin> portenable 3
```

Test se pokreće automatski. Praćenje:

```
SAN_A_SW01:admin> portdporttest --show 3

D-Port Information:
===================
Port:               3
Remote WWNN:        10:00:38:ba:b0:fc:aa:20
Remote port:        3
Mode:               Automatic
No. of test frames: 1 Million
Test frame size:    1024 Bytes
FEC (enabled/option/active): Yes/No/No
Start time:         Tue Sep 15 13:22:04 2026
End time:           Tue Sep 15 13:24:41 2026
Status:             PASSED
   ========================================================================
   Test                    Start time   Result      Comments
   ========================================================================
   Electrical loopback     13:22:06     PASSED      ----------
   Optical loopback        13:22:31     PASSED      ----------
   Link traffic test       13:22:58     PASSED      ----------
   ========================================================================
   Roundtrip link latency:    167 nano-seconds
   Estimated cable distance:  3 meters
   Buffers required:          1 (for 1024 byte frames at 32Gbps)
   Egress power loss:         0.4 dBm (Tx: 0.7 dBm, Rx: 0.3 dBm)
   Ingress power loss:        0.5 dBm (Tx: 0.8 dBm, Rx: 0.3 dBm)
```

### Vraćanje porta u normalan režim

```
SAN_A_SW01:admin> portdisable 3
SAN_A_SW01:admin> portcfgdport --disable 3
SAN_A_SW01:admin> portenable 3
```

### Tumačenje neuspelog testa

Najkorisniji podatak je ono što D_Port **ne uspe** da utvrdi.

```
D-Port Information:
===================
Port:               3
Remote WWNN:        00:00:00:00:00:00:00:00
Remote port:        --
Mode:               Automatic
Start time:         Tue Sep 15 13:40:12 2026
Status:             FAILED
   ========================================================================
   Test                    Start time   Result      Comments
   ========================================================================
   Electrical loopback     13:40:14     PASSED      ----------
   Optical loopback        13:40:38     FAILED      Remote port not responding
   ========================================================================
```

Čitanje ovog izlaza:

- **Electrical loopback PASSED** — SFP na switchu je ispravan i emituje kako treba
- **Remote WWNN sav u nulama** — switch ne dobija nikakav odgovor sa druge strane
- Zaključak: problem je u smeru **od switcha ka uređaju** — kabl, konektor, ili uređaj na drugom kraju ne prima

Ovo je ključna vrednost D_Porta: **izoluje smer u kojem link ne radi.** Bez njega, „link ne radi" je jedna informacija; sa njim su to dve — koji smer radi, a koji ne.

> **Iz prakse:** kada `sfpshow` pokazuje da switch normalno emituje, D_Port potvrdi da je lokalni SFP ispravan, a remote WWNN ostaje u nulama — a HBA na serveru istovremeno prijavljuje da nema signala, onda se svetlo nikada ne sreću. Pre zamene kablova proverite u koji slot na serveru je kabl zaista utaknut. FC i Ethernet SFP konektori su identični, a slotovi su često jedan pored drugog.

D_Port test se može izvesti i između switcha i HBA kartice, ako HBA to podržava — moderne Emulex i QLogic kartice podržavaju.

---

## 10.5 `fcping` i `pathinfo`

### `fcping` — da li uređaj odgovara

```
SAN_A_SW01:admin> fcping --number 5 21:00:f4:e9:d4:56:7a:b1
Pinging 21:00:f4:e9:d4:56:7a:b1 [0x010000] with 12 bytes of data:
received reply from 21:00:f4:e9:d4:56:7a:b1: 12 bytes time:812 usec
received reply from 21:00:f4:e9:d4:56:7a:b1: 12 bytes time:798 usec
received reply from 21:00:f4:e9:d4:56:7a:b1: 12 bytes time:805 usec
received reply from 21:00:f4:e9:d4:56:7a:b1: 12 bytes time:801 usec
received reply from 21:00:f4:e9:d4:56:7a:b1: 12 bytes time:809 usec
5 frames sent, 5 frames received, 0 frames rejected, 0 frames timeout
Round-trip min/avg/max = 798/805/812 usec
```

Provera da li dva uređaja mogu da komuniciraju — dakle provera zoninga na nivou fabrike:

```
SAN_A_SW01:admin> fcping 21:00:f4:e9:d4:56:7a:b1 50:06:01:60:88:60:2a:11
Source: 21:00:f4:e9:d4:56:7a:b1
Destination: 50:06:01:60:88:60:2a:11
Zone Check: Zoned
```

`Zone Check: Not Zoned` znači da uređaji nisu u istoj aktivnoj zoni — što je odgovor na pitanje „da li je problem u zoningu" u jednoj komandi.

### `pathinfo` — putanja kroz fabriku

```
SAN_A_SW01:admin> pathinfo 2
Target port is Embedded Port

  Hop  In Port  Domain ID (Name)            Out Port  BW  Cost
  ------------------------------------------------------------
    0       --  1   (SAN_A_SW01)                   6  32G    500
    1        6  2   (SAN_A_SW02)                  --   --     --
```

Prikazuje kroz koje switcheve i portove ide saobraćaj do odredišnog domena. Korisno kod dijagnostike u fabrikama sa više switcheva — pokazuje da li saobraćaj zaista ide putanjom koju očekujete.

---

## 10.6 `portlogdump` — šta se dešavalo na portu

Kada se uređaj prijavljuje pa nestaje, `switchshow` prikazuje trenutak, a ne istoriju. `portlogdump` prikazuje istoriju događaja na nivou pojedinačnih FC operacija.

```
SAN_A_SW01:admin> portlogdumpport 3
time         task    event   port cmd  args
------------------------------------------------------------
13:44:02.101 SPEED   speed      3 N    00000000,00000000
13:44:02.310 PORT    scn        3 1    00000000,00000000
13:44:02.512 ctp     Rx3        3      22000000,00000000,00000000  FLOGI
13:44:02.514 ctp     Tx3        3      23000000,ffffffff,00000000  ACC
13:44:02.688 ctp     Rx3        3      04000000,fffffc00,00000000  PLOGI
13:44:02.690 ctp     Tx3        3      02000000,fffffc00,00000000  ACC
13:44:08.221 PORT    scn        3 4    00000000,00000000
13:44:08.230 PORT    offline    3      00000000,00000000
13:44:11.410 PORT    online     3      00000000,00000000
13:44:11.615 ctp     Rx3        3      22000000,00000000,00000000  FLOGI
```

Iz ovog izlaza se vidi obrazac: uređaj se uspešno prijavio, šest sekundi kasnije port je otišao offline, pa se posle tri sekunde ponovo prijavio. To je **flapping** — i to je potpuno drugačiji problem od „uređaj se nikada nije prijavio".

Uključivanje detaljnijeg beleženja i čišćenje:

```
SAN_A_SW01:admin> portlogenable
SAN_A_SW01:admin> portlogclear
```

---

## 10.7 MAPS — proaktivni nadzor

**Monitoring and Alerting Policy Suite** prati stotine parametara i javlja kada neki pređe prag. Zamenjuje stariji Fabric Watch.

```
SAN_A_SW01:admin> mapsdb --show

1 Dashboard Information:
=========================
DB start time:              Mon Sep 14 00:00:02 2026
Active policy:              dflt_conservative_policy
Configured Notifications:   RASLOG,SNMP
Fenced Ports:               None
Decommissioned Ports:       None
Quarantined Ports:          None

2 Switch Health Report:
=========================
Current Switch Policy Status: HEALTHY

3 Summary Report:
=========================
Category                  |Today              |Last 7 days        |
-------------------------------------------------------------------
Port Health               |In operating range |Out of operating range|
Fru Health                |In operating range |In operating range |
Fabric State Changes      |In operating range |In operating range |
Switch Resource           |In operating range |In operating range |
Traffic Performance       |In operating range |In operating range |

4 Rules Affecting Health:
=========================
Category    |RepeatCount|Rule Name          |Execution Time     |Object |Value|
-------------------------------------------------------------------------------
Port Health |3          |defALL_OTHER_F_PORTCRC_ERR|09/12/26 02:14:33|F-Port 3|28 |
```

Sekcija 4 je najvrednija: pokazuje koje je pravilo prekršeno, kada, na kom objektu i sa kojom vrednošću. U ovom primeru — CRC greške na portu 3, pre tri dana. Podatak koji bez MAPS-a ne biste imali, jer su brojači kumulativni i ne nose vremensku oznaku.

Politike:

```
SAN_A_SW01:admin> mapspolicy --show -summary
Policy Name                          Number of Rules
-------------------------------------------------------
dflt_conservative_policy             276
dflt_moderate_policy                 276
dflt_aggressive_policy               276
dflt_base_policy                     6
```

```
SAN_A_SW01:admin> mapspolicy --enable dflt_moderate_policy
Policy dflt_moderate_policy activated successfully
```

`conservative` javlja samo ozbiljne događaje, `aggressive` reaguje na svaku anomaliju. Za produkciju je `moderate` obično dobra sredina.

Obaveštavanje:

```
SAN_A_SW01:admin> mapsconfig --actions raslog,snmp,email
SAN_A_SW01:admin> mapsconfig --emailcfg -address san-tim@firma.rs
```

---

## 10.8 Recepti za česte scenarije

### A. Port ostaje `No_Light`

```
SAN_A_SW01:admin> switchshow | grep "^   4"
   4   4   010400   id    --      No_Light    FC
```

| Korak | Komanda / radnja | Šta tražite |
|---|---|---|
| 1 | `sfpshow 4` | Da li lokalni modul emituje (Tx normalan) |
| 2 | Provera druge strane | Da li je uređaj upaljen i HBA aktivna |
| 3 | **Provera slota na serveru** | Da li je kabl u FC portu, a ne u Ethernet SFP28 portu |
| 4 | `portcfgshow 4` | Nije persistent disabled, E_Port dozvoljen ako treba |
| 5 | Zamena patch kabla | — |
| 6 | D_Port test | Koji smer linka ne radi |

### B. CRC greške

```
SAN_A_SW01:admin> porterrshow | grep "^  7"
  7:  2.1g   4.3g    112    108    108     0      0      0    44     0      2      3      1
```

1. `statsclear`, sačekati, ponovo `porterrshow` — da li rastu
2. `sfpshow 7` — Rx snaga ispod -9 dBm objašnjava greške
3. Očistiti konektore na oba kraja, ponovo izmeriti
4. Zameniti patch kabl
5. D_Port test za merenje gubitka po smeru
6. Zameniti SFP na strani koja lošije prima

### C. Host ne vidi storage

Redosled iz poglavlja 7, proširen:

```
SAN_A_SW01:admin> switchshow | grep <WWPN hosta>     # da li je port online i F-Port
SAN_A_SW01:admin> nsshow | grep <WWPN hosta>         # da li je prijavljen u fabric
SAN_A_SW01:admin> switchshow | grep -i zoning        # da li je zoning aktivan
SAN_A_SW01:admin> cfgactvshow                        # da li su oba WWPN-a u istoj zoni
SAN_A_SW01:admin> zone --validate | grep "\*"        # da li je WWPN tačno unet
SAN_A_SW01:admin> fcping <WWPN hosta> <WWPN targeta> # Zone Check: Zoned?
```

Ako je sve uredno na switchu, problem je van njega: rescan na hostu, LUN masking na storage nizu, ili host grupa na nizu.

### D. Putanja pada i diže se

```
SAN_A_SW01:admin> portlogdumpport 5 | tail -40
SAN_A_SW01:admin> porterrshow | grep "^  5"
```

Tražite obrazac: `link fail` i `loss sync` koji rastu uz ponovljene FLOGI operacije. Uzroci po učestalosti: loš kabl ili konektor, SFP pred otkazom, problem sa napajanjem ili firmware-om HBA kartice.

Zaštita dok se problem rešava:

```
SAN_A_SW01:admin> portcfgpersistentdisable 5
```

Port koji stalno pada i diže se generiše RSCN lavine i šteti celoj fabrici više nego što koristi. Bolje je ugasiti ga i osloniti se na drugu fabriku dok se ne reši.

### E. Pad performansi bez grešaka

```
SAN_A_SW01:admin> portstatsshow 6 | grep -E "tim_txcrd_z|er_"
er_enc_in                        0
er_crc                           0
tim_txcrd_z                 284417
```

Nula grešaka, a `tim_txcrd_z` raste — nema kvara, ima zagušenja. Dva moguća uzroka:

- **Nedovoljno ISL propusnosti** → dodati ISL, razmotriti trunking
- **Spor uređaj (slow drain device)** → jedan uređaj ne vraća kredite dovoljno brzo i zadržava saobraćaj celoj fabrici

```
SAN_A_SW01:admin> portbuffershow
SAN_A_SW01:admin> mapsdb --show | grep -i latency
```

Spor uređaj je posebno opasan jer simptom oseća ceo SAN, a uzrok je jedan port. MAPS ima pravila koja ga prepoznaju i mogu ga automatski izolovati.

### F. Fabrika segmentirana

```
SAN_A_SW01:admin> switchshow | grep -i segment
SAN_A_SW01:admin> errdump | grep -i segment
SAN_A_SW01:admin> portshow 6 | grep -i segment
```

Razlog i rešenje po tabeli iz poglavlja 8. Posle usklađivanja: `portdisable` pa `portenable`.

### G. Uređaj se prijavi pa nestane iz `nsshow`

```
SAN_A_SW01:admin> portlogdumpport 3 | grep -E "FLOGI|LOGO|offline"
```

Ako se vide ponovljeni FLOGI i LOGO — uređaj se sam odjavljuje. To je skoro uvek strana hosta: drajver, firmware HBA kartice, ili pogrešan režim veze (Loop umesto Point-to-Point).

### H. Sve putanje pale posle izmene zoninga

```
SAN_A_SW01:admin> cfgactvshow
SAN_A_SW01:admin> cfgtransshow
```

Najverovatniji uzroci: aktivirana je pogrešna konfiguracija, ili je nova konfiguracija izostavila zone koje su postojale. Poređenje sa backup fajlom daje odgovor odmah:

```
[root@backup ~]# grep "^zone.cfg" /backup/san/SAN_A_SW01_2026-09-15.txt
```

Ako je greška očigledna, vratite prethodnu konfiguraciju:

```
SAN_A_SW01:admin> cfgenable "PROD_FABA_CFG_STARA"
```

---

## 10.9 Kada zvati podršku

Pozovite podršku kada:

- Postoji sumnja na hardverski kvar switcha (POST greške, kvar porta na više SFP-ova)
- Problem se ponavlja bez objašnjivog uzroka posle zamene kablova i modula
- Fabrika se ponaša nepredvidivo posle nadogradnje firmware-a
- Log sadrži poruke koje niste u stanju da protumačite iz dokumentacije

Šta pripremiti pre poziva:

1. `supportsave` sa **svih switcheva u pogođenoj fabrici**, uzet **dok problem traje**
2. Tačno vreme početka problema
3. Šta je menjano neposredno pre — konfiguracija, firmware, kablovi, novi uređaji
4. Spisak već isključenih hipoteza i način na koji su isključene
5. Serijski broj uređaja (`chassisshow`)

Tačka 4 je ono što najviše ubrzava rešavanje. Bez nje ćete proći kroz istu listu osnovnih provera koju ste već završili.

---

## 10.10 Rezime poglavlja

- Dijagnostika ide odozdo nagore: fizički sloj, link, fabric, zoning, host.
- Tri komande za prvi minut: `switchshow`, `sfpshow`, `porterrshow`.
- Brojači su kumulativni — `statsclear` pa ponovno merenje je jedini pouzdan način.
- D_Port izoluje **smer** u kojem link ne radi; remote WWNN u nulama je jasan pokazatelj.
- `fcping` sa dva WWPN-a odgovara na pitanje o zoningu u jednoj komandi.
- `portlogdump` razlikuje „nikada se nije prijavio" od „prijavljuje se i pada".
- Greške bez rasta `tim_txcrd_z` su kvar; rast `tim_txcrd_z` bez grešaka je zagušenje.
- Port koji stalno pada i diže se treba ugasiti dok se problem ne reši.
- `supportsave` se uzima dok problem traje, sa svih switcheva u fabrici.

---

## 10.11 Provera znanja

1. `sfpshow` pokazuje normalan Tx i -40 dBm Rx na obe strane linka. Šta je uzrok i zašto je karakterističan?
2. D_Port test prolazi električnu petlju, a pada na optičkoj, uz remote WWNN u nulama. Šta ovo isključuje kao uzrok, a šta ostavlja otvorenim?
3. Na ISL portu nema nijedne greške, ali `tim_txcrd_z` naglo raste. Navedite dva moguća uzroka i kako ih razlikujete.
4. Host se pojavljuje u `nsshow` pa nestaje, u ciklusu od nekoliko sekundi. Kojom komandom potvrđujete obrazac i gde je najverovatniji uzrok?
5. Posle `cfgenable` svi hostovi su izgubili pristup. Koja su vaša prva dva koraka i koji vam dokument najbrže daje odgovor?

---

## Zaključak kursa

Kroz deset poglavlja prošli smo put od osnova Fibre Channel tehnologije do samostalnog rešavanja problema u produkcijskoj fabrici. Ono što ostaje kao trajna osnova:

**Četiri komande pokrivaju većinu svakodnevnog rada** — `switchshow`, `nsshow`, `porterrshow`, `cfgactvshow`.

**Dva sloja kontrole pristupa** — zoning na switchu i LUN masking na storage nizu. Oba moraju biti ispravna.

**Dve nezavisne fabrike** koje se nikada ne spajaju, i pravilo da se intervencija radi na jednoj dok druga drži saobraćaj.

**Dokumentacija nije administrativni teret** — imenovani portovi, tabela kabliranja, dosledna imena zona i redovni backup su ono što razliku između petnaestominutnog i petosatnog rešavanja problema.

**Fizički sloj prvo.** Uvek.
