# 09 — Protokoli i pristup podacima

Kako backup aplikacije i klijenti stižu do podataka na DD-u. Četiri načina:
NFS, CIFS/SMB, DD Boost i VTL.

---

## 9.0 Koji protokol kada

| Protokol | Kada se koristi | Napomena |
|---|---|---|
| **DD Boost** | **Podrazumevani izbor** za podržane backup aplikacije | Dedupe na strani klijenta — manje saobraćaja, brži backup, upravljana replikacija |
| **NFS** | Linux/UNIX klijenti, aplikacije bez DD Boost podrške | Jednostavno, ali sav dedupe je na DD-u |
| **CIFS/SMB** | Windows okruženja bez DD Boost-a | Traži integraciju sa AD-om |
| **VTL** | Nasleđena okruženja, mainframe, aplikacije koje znaju samo za trake | Legacy. Ne birati za nove instalacije |

Ako backup aplikacija podržava DD Boost, koristi se DD Boost. Sve ostalo su izuzeci.

---

## 9.1 NFS

### Provera

```
nfs status                               # da li je NFS servis uključen
nfs show clients                         # svi definisani export-i i klijenti
nfs show active                          # trenutno aktivni klijenti
nfs show detailed-stats
nfs show stats
```

### Konfiguracija

⚠️

```
nfs enable
nfs disable                              # 🛑 prekida pristup NFS klijentima

nfs add <putanja> <klijenti> [options <opcije>]
nfs del <putanja> <klijenti>
nfs reset clients
```

Primer:

```
nfs add /data/col1/backup1 192.168.10.0/24 \
    options rw,no_root_squash,no_all_squash,secure
```

Uobičajene opcije:

| Opcija | Značenje |
|---|---|
| `rw` / `ro` | Čitanje-pisanje / samo čitanje |
| `no_root_squash` | `root` sa klijenta ostaje `root` — traži većina backup aplikacija |
| `secure` | Zahteva izvorni port ispod 1024 |
| `sec=sys` | Autentikacija (moguć i Kerberos: `krb5`, `krb5i`, `krb5p`) |

> **Bezbednosna napomena:** `no_root_squash` u kombinaciji sa širokim opsegom
> klijenata (`*` ili cela mreža) znači da svako sa te mreže ima pun pristup
> backup podacima. Ograničite na konkretne IP adrese backup servera.

### Mount sa klijenta

```bash
mount -t nfs -o hard,intr,nfsvers=3 <dd-host>:/data/col1/backup1 /mnt/dd
```

### Playbook: klijent ne može da mount-uje

1. `nfs status` — servis uključen?
2. `nfs show clients` — postoji li export za tu putanju i za taj IP?
3. `net ping <klijent>` — mrežna dostupnost
4. Sa klijenta: `showmount -e <dd-host>`
5. `mtree list` — postoji li MTree i u kom je stanju (`RD` = odredište replikacije,
   ne može se pisati)
6. Firewall između klijenta i DD-a — portovi 111 i 2049
7. `log view` na DD-u u trenutku pokušaja mount-a

---

## 9.2 CIFS / SMB

### Provera

```
cifs status
cifs show config
cifs share show
cifs show active                         # aktivne sesije
cifs show detailed-stats
```

### Konfiguracija

⚠️

```
cifs enable
cifs disable                             # 🛑

cifs share create <ime-share-a> path /data/col1/<mtree> clients <lista>
cifs share modify <ime-share-a> clients <lista>
cifs share show <ime-share-a>
cifs share destroy <ime-share-a>         # 🛑
cifs share enable <ime-share-a>
cifs share disable <ime-share-a>
```

### Autentikacija

```
cifs show config
cifs set authentication active-directory <realm> [<dc-lista>]     # ⚠️
cifs set authentication workgroup <radna-grupa>                   # ⚠️
```

Za Active Directory integraciju moraju biti ispunjeni preduslovi:
ispravan DNS, sinhronizovano vreme (odstupanje veće od nekoliko minuta ruši
Kerberos) i nalog sa pravom pridruživanja domenu.

```
net show dns
system show date
ntp status
```

### Playbook: CIFS share nedostupan

1. `cifs status` i `cifs show config` — servis i tip autentikacije
2. `cifs share show` — postoji li share i da li je omogućen
3. `system show date` + `ntp status` — **odstupanje vremena je najčešći uzrok**
   kod AD autentikacije
4. `net show dns`, `net lookup <dc>` — DNS razrešavanje domenskih kontrolera
5. `cifs show active` — da li ijedna sesija uspeva
6. `log view` u trenutku pokušaja pristupa
7. Ako je uređaj ispao iz domena — ponovno pridruživanje kroz
   `cifs set authentication active-directory` ⚠️

---

## 9.3 DD Boost

DD Boost pomera deo deduplikacije na klijenta. Backup aplikacija šalje samo
segmente koje DD još nema, što drastično smanjuje mrežni saobraćaj i skraćuje
backup prozor.

### Provera

```
ddboost status
ddboost show connections                 # aktivne sesije po klijentu
ddboost show stats
ddboost storage-unit show                # storage unit-ovi i zauzeće
ddboost option show
ddboost file-replication show config     # managed file replication
```

`ddboost show connections` pokazuje koji klijenti su trenutno povezani i
koliko tokova koriste. Prva komanda kada backup tim pita "da li vidite naš server".

### Storage unit-ovi

Storage unit je DD Boost pojam za prostor dodeljen jednoj backup aplikaciji.
Ispod je običan MTree — vidi se i u `mtree list`.

⚠️

```
ddboost storage-unit create <ime> user <korisnik>
ddboost storage-unit create <ime> user <korisnik> \
    quota-soft-limit <n> GiB quota-hard-limit <n> GiB
ddboost storage-unit modify <ime> ...
ddboost storage-unit show <ime>
ddboost storage-unit delete <ime>        # 🛑
ddboost storage-unit undelete <ime>
```

### Korisnici

DD Boost nalog je običan DD korisnik, po pravilu sa rolom `none` — može da
koristi DD Boost, ali ne može da se prijavi na CLI.

⚠️

```
user add <ime> role none
ddboost user assign <ime>
ddboost user show
ddboost user unassign <ime>
```

### Opcije

```
ddboost option show
ddboost option set distributed-segment-processing enabled     # ⚠️
ddboost option set virtual-synthetics enabled                 # ⚠️
```

| Opcija | Šta radi |
|---|---|
| `distributed-segment-processing` | Dedupe na klijentu. **Ovo je suština DD Boost-a** — bez toga se gubi glavna prednost |
| `virtual-synthetics` | Sintetički puni backup bez ponovnog čitanja podataka |

### Managed File Replication

Replikacija koju kontroliše backup aplikacija, a ne DD.

```
ddboost file-replication show config
ddboost file-replication show stats
ddboost file-replication show active
ddboost file-replication option show
```

Ako aplikacija upravlja replikacijom, **nemojte praviti paralelni MTree
replikacioni kontekst za isti MTree** — dva mehanizma nad istim podacima
proizvode nepredvidive rezultate. Vidi **poglavlje 07**.

### ifgroup — raspodela po interfejsima

```
ifgroup show config
ifgroup status
```

Detaljno → **poglavlje 08**.

### Playbook: DD Boost konekcija pada

1. `ddboost status` — servis uključen?
2. `ddboost show connections` — vidi li se klijent uopšte
3. `ddboost user show` — postoji li nalog i da li je dodeljen
4. `ddboost storage-unit show` — postoji li storage unit, je li dostigao kvotu
5. `quota capacity show` — hard limit dostignut znači da upisi padaju
6. `filesys show space` — ima li mesta uopšte
7. `ifgroup status` — ako se koristi, je li interfejs dostupan
8. Verzija DD Boost biblioteke na klijentu naspram DDOS verzije — **Support Matrix**
9. `log view` u trenutku pada

---

## 9.4 VTL

Emulacija tračne biblioteke. Koristi se u nasleđenim okruženjima.
Za nove instalacije bira se DD Boost.

### Provera

```
vtl status
vtl show config
vtl library show list
vtl library show detailed <biblioteka>
vtl tape show <biblioteka>
vtl group show list
vtl drive show list
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

vtl library create <ime> model <model> slots <n> drives <n>
vtl tape add pool <pool> capacity <n> count <n>
vtl group add <grupa> initiator <initiator>
```

### Playbook: biblioteka se ne vidi na backup serveru

1. `vtl status` — servis uključen
2. `scsitarget port show list` — je li FC port online, vidi li se link
3. `scsitarget initiator show list` — vidi li DD inicijator (HBA backup servera)
4. `vtl group show list` — je li inicijator dodeljen odgovarajućoj grupi
5. Zoning na FC svič-u — najčešći uzrok, radi ga SAN tim
6. Drajveri i verzija HBA firmware-a na backup serveru

---

## 9.5 Ko trenutno koristi uređaj

Jedna od najčešćih potreba pre reboot-a ili planiranih radova:

```
ddboost show connections
nfs show active
cifs show active
vtl status
replication status
filesys clean status
```

Ovih šest komandi daju punu sliku ko i šta trenutno radi na uređaju.
Ide u proceduru pre svakog 🛑 zahvata (**poglavlje 03**).

---

## 9.6 Checklist za novi pristup podacima

- [ ] Izabran protokol prema tabeli 9.0 (DD Boost ako je podržan)
- [ ] Kreiran zaseban MTree, nije korišćen `/backup` (**poglavlje 05**)
- [ ] Postavljena kvota (soft + hard)
- [ ] Za NFS: export ograničen na konkretne IP adrese, ne na celu mrežu
- [ ] Za CIFS: DNS i NTP provereni pre pokušaja pridruživanja domenu
- [ ] Za DD Boost: nalog sa rolom `none`, storage unit kreiran,
      `distributed-segment-processing` uključen
- [ ] Verzija klijentske biblioteke/agenta proverena u Support Matrix-u
- [ ] Testiran upis i čitanje pre puštanja u produkciju
- [ ] Ako MTree ide na replikaciju ili u CR vault — kontekst konfigurisan
      pre nego što stignu podaci (poglavlja **07** i **12**)

---

## Reference

- DDOS Administration Guide — poglavlja o NFS-u, CIFS-u, DD Boost-u i VTL-u
- DD Boost for Partner Integration Administration Guide
- DD Boost for OpenStorage (OST) Administration Guide
- DD BoostFS Configuration Guide (Linux / Windows)
- DDOS Command Reference Guide — sekcije `nfs`, `cifs`, `ddboost`, `vtl`, `scsitarget`, `ifgroup`
