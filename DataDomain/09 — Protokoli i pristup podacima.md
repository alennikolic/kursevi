# 09 — Protokoli i pristup podacima

Kako backup aplikacije i klijenti stižu do podataka: NFS, CIFS/SMB, DD Boost, VTL.

---

## 9.0 Koji protokol kada

| Protokol | Kada se koristi | Napomena |
|---|---|---|
| **DD Boost** | **Podrazumevani izbor** za podržane backup aplikacije | Dedupe na strani klijenta — manje saobraćaja, brži backup, upravljana replikacija |
| **NFS** | Linux/UNIX klijenti, aplikacije bez DD Boost podrške | Sav dedupe je na DD-u |
| **CIFS/SMB** | Windows okruženja bez DD Boost-a | Traži integraciju sa AD-om |
| **VTL** | Nasleđena okruženja, mainframe | Legacy. Ne birati za nove instalacije |

Ako backup aplikacija podržava DD Boost, koristi se DD Boost.

---

## 9.1 NFS

DD OS 8.6 ima **dva mehanizma**: noviji `nfs export` (imenovani export-i) i
stariji `nfs add` (direktno po putanji). Oba rade. **Za nove konfiguracije
koristite `nfs export`** — export dobija ime, može se menjati bez ponovnog
definisanja, i podržava referrals.

### Provera

```
nfs status
nfs export show
nfs show clients [tenant-unit <tu>]
nfs show active [tenant-unit <tu>]
nfs show stats
nfs show detailed-stats [<page-no>]
nfs show histogram
nfs show port
```

### Servis

⚠️

```
nfs enable [version <lista>]
nfs disable [version <lista>]         # 🛑 prekida pristup NFS klijentima
```

`version` omogućava selektivno uključivanje verzija protokola (npr. isključiti NFSv3
a ostaviti NFSv4).

### Preporučeni način — nfs export

⚠️

```
nfs export create [<export-name>] path <putanja> [clients <lista> [options <opcije>]]
nfs export add {<export-spec> | all} clients <lista> [options <opcije>]
nfs export modify {<export-spec> | all} clients {<lista> | all} options <opcije>
nfs export del {<export-spec> | all} clients {<lista> | all}
nfs export destroy {<export-spec> | all}
nfs export show
```

Primer — kreiranje export-a i naknadno dodavanje klijenta:

```
nfs export create backup1-export path /data/col1/backup1 \
    clients 192.168.10.0/24 options rw,no_root_squash,secure

nfs export add backup1-export clients 192.168.20.15 \
    options rw,no_root_squash,secure

nfs export show
```

Referrals (preusmeravanje klijenta na drugi server):

```
nfs export add {<export-spec> | all} referral <ime> remote-servers <adrese>   # ⚠️
nfs export del {<export-spec> | all} referrals {<lista> | all}                # ⚠️
```

### Stariji način — nfs add

⚠️ **Opcije idu u zagradama.**

```
nfs add <putanja> <client-list> [(<option-list>)]
nfs del <putanja> <client-list>
```

Primer:

```
nfs add /data/col1/backup1 192.168.10.0/24 (rw,no_root_squash,secure)
```

> Zagrade su deo sintakse. Bez njih komanda neće proći.

### Uobičajene opcije

| Opcija | Značenje |
|---|---|
| `rw` / `ro` | Čitanje-pisanje / samo čitanje |
| `no_root_squash` | `root` sa klijenta ostaje `root` — traži većina backup aplikacija |
| `secure` | Zahteva izvorni port ispod 1024 |
| `sec=sys` | Autentikacija (moguć i Kerberos: `krb5`, `krb5i`, `krb5p`) |

> **Bezbednosna napomena:** `no_root_squash` sa širokim opsegom klijenata
> (`*` ili cela mreža) znači da svako sa te mreže ima pun pristup backup podacima.
> Ograničite na konkretne IP adrese backup servera.

### Mount sa klijenta

```bash
mount -t nfs -o hard,intr,nfsvers=3 <dd-host>:/data/col1/backup1 /mnt/dd
```

### Playbook: klijent ne može da mount-uje

1. `nfs status` — servis uključen, i za koju verziju protokola
2. `nfs export show` / `nfs show clients` — postoji li export za tu putanju i taj IP
3. `net ping <klijent>` — mrežna dostupnost
4. Sa klijenta: `showmount -e <dd-host>`
5. `mtree list` — postoji li MTree i u kom je stanju (`RD` = ne može se pisati)
6. Firewall — portovi 111 i 2049 (potvrdi u Dell KB 000004184)
7. `log view` na DD-u u trenutku pokušaja mount-a

---

## 9.2 CIFS / SMB

### Provera

```
cifs status
cifs show config
cifs share show [<share>]
cifs show active
cifs show stats
cifs show detailed-stats
```

### Servis i share-ovi

⚠️

```
cifs enable
cifs disable                             # 🛑

cifs share create <share> path <putanja> \
    {max-connections <n> | clients <lista> | users <lista> | ...}
cifs share modify <share> {max-connections <n> | clients <lista> | users <lista> | ...}
cifs share show [<share>]
cifs share enable <share>
cifs share disable <share>
cifs share destroy <share>               # 🛑
```

### Autentikacija

⚠️

```
cifs set authentication active-directory <realm> {[<dc1> [<dc2> ...]] | *}
cifs set authentication workgroup <radna-grupa>
cifs set nb-hostname <nb-hostname>
```

`*` znači automatsko otkrivanje domenskih kontrolera preko DNS-a.

Preduslovi za AD: ispravan DNS, sinhronizovano vreme, nalog sa pravom
pridruživanja domenu.

```
net show dns
net lookup <dc>
system show date
ntp status
```

### Playbook: CIFS share nedostupan

1. `cifs status`, `cifs show config` — servis i tip autentikacije
2. `cifs share show` — postoji li share i da li je omogućen
3. `system show date` + `ntp status` — **odstupanje vremena je najčešći uzrok**
   kod AD autentikacije (Kerberos ne tolerira više od nekoliko minuta)
4. `net show dns`, `net lookup <dc>` — razrešavanje domenskih kontrolera
5. `cifs show active` — uspeva li ijedna sesija
6. `log view` u trenutku pokušaja pristupa
7. Ako je uređaj ispao iz domena — ponovno pridruživanje
   `cifs set authentication active-directory` ⚠️

---

## 9.3 DD Boost

DD Boost pomera deo deduplikacije na klijenta — šalju se samo segmenti koje
DD nema, što smanjuje saobraćaj i skraćuje backup prozor.

### Provera

```
ddboost status
ddboost show connections [detailed]
ddboost show stats [interval <sec>] [count <n>]
ddboost show histogram
ddboost show user-name
ddboost storage-unit show [compression] [<storage-unit>] [tenant-unit <tu>]
ddboost option show
ddboost association show [all | storage-unit <su>]
ddboost file-replication show config
```

`ddboost show connections` je prva komanda kada backup tim pita
"da li vidite naš server".

### Storage unit-ovi

Ispod je običan MTree — vidi se i u `mtree list`.

⚠️

```
ddboost storage-unit create <su> user <user-name> [tenant-unit <tu>] \
    [quota-soft-limit <n> {MiB|GiB|TiB}] [quota-hard-limit <n> {MiB|GiB|TiB}]

ddboost storage-unit modify <su> [user <user-name>] [tenant-unit {<tu> | none}]
ddboost storage-unit rename <su> <novo-ime>
ddboost storage-unit show [compression] [<su>]
ddboost storage-unit delete <su>         # 🛑
ddboost storage-unit undelete <su>
```

Kvote se mogu postaviti i naknadno kroz `quota capacity set storage-units ...`
(**poglavlje 05**).

### Korisnici

DD Boost nalog je običan DD korisnik, po pravilu sa rolom `none` — može da
koristi DD Boost, ali ne može da se prijavi na CLI.

⚠️

```
user add <ime> role none
ddboost user assign <user-name-list>
ddboost user show [<user>] [default-tenant-unit <tu>]
ddboost user option set <user> default-tenant-unit <tu>
ddboost user option reset <user> [default-tenant-unit]
ddboost user revoke token-access <user-name-list>
ddboost user unassign <user-name>
```

> DD Boost korisnik se može obrisati samo ako ne poseduje nijedan storage unit.

### Klijenti i enkripcija

⚠️

```
ddboost clients add <client-list> [encryption-strength {none | medium | high} \
    authentication-mode {...}]
ddboost clients del <client-list>
ddboost clients show
```

### Opcije

```
ddboost option show [distributed-segment-processing | virtual-synthetics | fc | global-authentication-mode | global-encryption-strength]
```

⚠️

```
ddboost option set distributed-segment-processing {enabled | disabled}
ddboost option set virtual-synthetics {enabled | disabled}
ddboost option set fc {enabled | disabled}
ddboost option set global-authentication-mode {none | two-way | two-way-password}
ddboost option reset {distributed-segment-processing | virtual-synthetics | fc | ...}
```

| Opcija | Šta radi |
|---|---|
| `distributed-segment-processing` | Dedupe na klijentu. **Suština DD Boost-a** — bez toga se gubi glavna prednost |
| `virtual-synthetics` | Sintetički puni backup bez ponovnog čitanja podataka |
| `fc` | DD Boost preko Fibre Channel-a |

### Managed File Replication i asocijacije

```
ddboost file-replication show config
ddboost file-replication show stats
ddboost file-replication show active
ddboost file-replication option show
ddboost association show [all | storage-unit <su>]
```

⚠️

```
ddboost association create <local-su> {replicate-to | replicate-from} <remote-host> <remote-su>
ddboost association destroy <local-su> {replicate-to | replicate-from} <remote-host> <remote-su>
```

> Ako aplikacija upravlja replikacijom preko MFR-a, **nemojte praviti paralelni
> MTree replikacioni kontekst za isti MTree.** Dva mehanizma nad istim podacima
> daju nepredvidive rezultate. Vidi **poglavlje 07**.

### ifgroup

```
ifgroup show config [<group>] {all | summary | interfaces | clients | replication}
ifgroup show connections
```

Detaljno → **poglavlje 08**.

### Playbook: DD Boost konekcija pada

1. `ddboost status` — servis uključen
2. `ddboost show connections detailed` — vidi li se klijent
3. `ddboost user show` — postoji li nalog i da li je dodeljen
4. `ddboost storage-unit show` — postoji li SU, je li dostigao kvotu
5. `quota capacity show all` — hard limit znači da upisi padaju
6. `filesys show space` — ima li mesta
7. `ifgroup show config all` — ako se koristi, je li interfejs dostupan
8. `ddboost clients show` — nesaglasnost u enkripciji ili autentikaciji
9. Verzija DD Boost biblioteke na klijentu naspram DDOS verzije — **E-Lab Navigator**
10. `log view` u trenutku pada

---

## 9.4 VTL

Emulacija tračne biblioteke. Za nove instalacije bira se DD Boost.

### Provera

```
vtl status
vtl show config
vtl library show list
vtl library show detailed <biblioteka>
vtl tape show <biblioteka>
vtl drive show list
vtl group show list
vtl port show
```

SCSI target sloj (Fibre Channel):

```
scsitarget show summary
scsitarget port show list
scsitarget group show list
scsitarget initiator show list
```

### Osnovne operacije

⚠️

```
vtl enable
vtl disable                              # 🛑

vtl add <vtl> [model <model>] [slots <num-slots>] [caps <num-caps>]
vtl cap add <vtl> [count <n>]
vtl cap del <vtl> [count <n>]
vtl tape add pool <pool> capacity <n> count <n>
vtl group add <grupa> initiator <initiator>
vtl config export [vtl <vtl>] output-file <ime>
vtl config import [vtl <vtl>] [check-only] [skip-initiators] [retain-serial-numbers]
```

> Komanda za kreiranje biblioteke je **`vtl add`**, ne `vtl library create`.
> Prima `slots` i `caps`; broj drajvova se ne zadaje tu.

`vtl config export` je koristan pre svake izmene — snima konfiguraciju u fajl.

### Playbook: biblioteka se ne vidi na backup serveru

1. `vtl status` — servis uključen
2. `scsitarget port show list` — je li FC port online
3. `scsitarget initiator show list` — vidi li DD inicijator (HBA backup servera)
4. `vtl group show list` — je li inicijator u odgovarajućoj grupi
5. Zoning na FC svič-u — najčešći uzrok, radi ga SAN tim
6. Drajveri i firmware HBA na backup serveru

---

## 9.5 Ko trenutno koristi uređaj

Pre reboot-a ili planiranih radova:

```
ddboost show connections detailed
nfs show active
cifs show active
vtl status
replication status
filesys clean status
```

Ovih šest komandi daju punu sliku. Ide u proceduru pre svakog 🛑 zahvata
(**poglavlje 03**).

---

## 9.6 Checklist za novi pristup podacima

- [ ] Izabran protokol prema tabeli 9.0 (DD Boost ako je podržan)
- [ ] Kreiran zaseban MTree, nije korišćen `/backup` (**poglavlje 05**)
- [ ] Postavljena kvota i uključen `quota capacity enable`
- [ ] Za NFS: korišćen `nfs export create`, ograničen na konkretne IP adrese
- [ ] Za CIFS: DNS i NTP provereni pre pridruživanja domenu
- [ ] Za DD Boost: nalog sa rolom `none`, storage unit kreiran,
      `distributed-segment-processing` uključen
- [ ] Verzija klijentske biblioteke/agenta proverena u E-Lab Navigator-u
- [ ] Testiran upis i čitanje pre puštanja u produkciju
- [ ] Ako MTree ide na replikaciju ili u CR vault — kontekst konfigurisan
      pre nego što stignu podaci (poglavlja **07** i **12**)

---

## Reference

- DD OS 8.6 Command Reference Guide — poglavlja `nfs`, `cifs`, `ddboost`, `vtl`, `scsitarget`, `ifgroup`
- DD OS 8.6 Administration Guide — protokoli i pristup podacima
- DD Boost for Partner Integration / OpenStorage Administration Guide
- DD BoostFS Configuration Guide (Linux / Windows)
