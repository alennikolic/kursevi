# 01 — Osnove i pristup

Kako se doći do CLI-ja, kako se snaći u njemu i ko šta sme da radi.

---

## 1.1 Načini pristupa

| Način | Kada se koristi | Napomena |
|---|---|---|
| **SSH** | 95% posla | `ssh sysadmin@<ip-ili-fqdn>` |
| **Serijska konzola** | Uređaj nije na mreži, inicijalna konfiguracija, oporavak | DB9 / RJ45, `115200 8N1`, bez flow control-a |
| **DD System Manager (GUI)** | Vizuelni pregled, izveštaji, wizardi | `https://<ip>` |
| **IPMI / remote konzola** | Uređaj ne odgovara, treba power cycle | Konfiguriše se komandom `ipmi` |
| **DDMC** | Centralni nadzor više DD uređaja | Zasebna aplikacija |

Za novi uređaj: podrazumevani korisnik je `sysadmin`, a **podrazumevana lozinka je
serijski broj sistema** (PSNT sa nalepnice na šasiji). Prvi login traži promenu lozinke.

---

## 1.2 Struktura CLI-ja

DDOS CLI nije Linux shell. To je zatvoreni skup komandi u obliku:

```
<komanda> <podkomanda> [argumenti]
```

Primeri:

```
filesys show space
mtree retention-lock status mtree /data/col1/backup1
replication show config
```

Pomoć:

```
help                             # spisak svih glavnih komandi
help mtree                       # sve podkomande za mtree
help mtree retention-lock        # detaljna sintaksa
?                                # kontekstualna pomoć
```

**Tab completion radi.** Kucanje `mtree ret<TAB>` dopunjava komandu. Ovo je najbrži
način da se proveri da li podkomanda uopšte postoji u vašoj verziji DDOS-a.

Ako komanda iz ovog priručnika ne prolazi, prvo `help <komanda>` na samom uređaju —
sintaksa se menja između verzija.

### Filtriranje outputa

Novije DDOS verzije podržavaju ograničeno pipe-ovanje:

```
mtree list | grep backup
filesys show space | more
alerts show history | tail
```

Ako `|` ne radi u vašoj verziji, jedina opcija je izvlačenje outputa preko SSH-a
i obrada na strani klijenta.

### Izvršavanje bez interaktivne sesije

DDOS prihvata komandu kao SSH argument, što je osnova za svaku automatizaciju:

```bash
ssh sysadmin@dd01 "filesys show space"
ssh sysadmin@dd01 "replication status"
```

Za neinteraktivni rad postavite SSH ključ:

```
adminaccess add ssh-keys user sysadmin      # ⚠️ traži paste javnog ključa
adminaccess show ssh-keys
adminaccess del ssh-keys <index> user sysadmin
```

---

## 1.3 Role i korisnici

| Rola | Šta može |
|---|---|
| `admin` | Sve, uključujući konfiguraciju sistema |
| `limited-admin` | Kao admin, ali ne može da isključi file system niti da uradi destruktivne sistemske operacije |
| `security-officer` | Autorizuje operacije zaštićene Retention Lock Compliance režimom. Ne može da administrira sistem |
| `backup-operator` | Snapshot-ovi i osnovne backup operacije |
| `user` | Samo read-only pregled |
| `none` | Nema pristup CLI-ju; koristi se za DD Boost naloge |
| `tenant-admin` / `tenant-user` | Secure Multi-Tenancy, ograničeno na svoj tenant unit |

Komande:

```
user show list                                  # svi nalozi i njihove role
user show detailed <username>
user add <username> role <rola>                 # ⚠️
user change role <username> <nova-rola>         # ⚠️
user change password <username>                 # ⚠️
user enable <username>
user disable <username>                         # ⚠️
user del <username>                             # 🛑
```

**Preporuka:** za monitoring i skripte koristite zaseban nalog sa rolom `user`.
Read-only provere iz poglavlja 02 ne traže `admin`.

---

## 1.4 Upravljanje pristupom (adminaccess)

```
adminaccess show                                # koji su protokoli otvoreni
adminaccess enable ssh                          # ⚠️
adminaccess disable telnet                      # ⚠️
adminaccess disable ftp                         # ⚠️
adminaccess show certificate
```

**Otvrdnjavanje — minimum:**

```
adminaccess disable telnet
adminaccess disable ftp
adminaccess disable http                        # ostaviti samo https
```

Ograničavanje sa kojih IP adresa se sme prijaviti:

```
adminaccess show
adminaccess add ssh <ip-ili-mreza>              # ⚠️
adminaccess del ssh <ip-ili-mreza>              # ⚠️
```

Politike lozinki i sesija:

```
adminaccess option show
adminaccess option set login-max-attempts <n>           # ⚠️
adminaccess option set login-lockout-period <minuti>    # ⚠️
adminaccess option set session-timeout <sekunde>        # ⚠️
```

---

## 1.5 Osnovna orijentacija na nepoznatom uređaju

Kada prvi put sednete na tuđi ili nasleđen sistem, ovim redom:

```
system show version                  # verzija DDOS-a
system show modelno                  # model
system show serialno                 # serijski broj za support case
system show uptime
license show                         # koje su funkcije uopšte licencirane
elicense show
net show settings                    # mrežna konfiguracija
mtree list                           # čemu ovaj uređaj služi
replication show config              # da li je izvor ili odredište replikacije
filesys show space                   # koliko je pun
alerts show current
```

`license show` je bitan korak — bez njega ne znate da li uređaj uopšte ima
Retention Lock, Replication, Cloud Tier ili Encryption licencu, pa ćete
tražiti komande koje neće raditi.

---

## 1.6 Vreme, NTP i DNS

Pogrešno vreme lomi Retention Lock, replikaciju i sertifikate. Provera:

```
system show date
ntp show config
ntp status
net show dns
```

Izmena:

```
ntp add timeserver <ip-ili-fqdn>       # ⚠️
ntp enable                             # ⚠️
net set dns <ip1> <ip2>                # ⚠️
timezone show
timezone set <zona>                    # ⚠️ traži reboot
```

> **Upozorenje:** na sistemu sa uključenim Retention Lock Compliance režimom,
> izmena sistemskog vremena je zaštićena operacija i traži autorizaciju
> security officera. To je namerno — pomeranje sata unazad bi zaobišlo lock.

---

## 1.7 Konfiguracioni wizard

Za inicijalno postavljanje ili za ponovno prolazak kroz osnovnu konfiguraciju:

```
config setup                          # ⚠️ interaktivni wizard
```

Prolazi kroz mrežu, DNS, vreme, mejl obaveštenja, licence i osnovne protokole.
Na produkcijskom sistemu ne pokrećite ga bez razloga — lako se slučajno
prepiše postojeća konfiguracija.

Pregled bez izmene:

```
config show all
```

---

## 1.8 Šta CLI ne radi

- **Nema pristup shell-u.** `se` (Support Engineering) mod je rezervisan za Dell
  support i ne koristi se bez otvorenog case-a. Komande iz `se` moda ne pripadaju
  u operativne procedure.
- **Nema `rm`, `ls`, `cd` nad podacima.** Podacima se pristupa preko protokola
  (NFS, CIFS, DD Boost), ne preko CLI-ja. Za kopiranje unutar uređaja postoji
  `filesys fastcopy`.
- **Nema rollback konfiguracije.** Ne postoji "undo". Pre svake ⚠️ komande
  snimite trenutno stanje relevantnom `show` komandom.

---

## 1.9 Reboot i gašenje

🛑 Sve u ovoj sekciji prekida servis.

```
system reboot
system poweroff
system show uptime                    # provera nakon vraćanja
```

Pre reboot-a obavezno:

1. Proveriti da nema aktivnog backup-a: `ddboost show connections`, `nfs show active`, `cifs show active`
2. Proveriti da cleaning ne radi: `filesys clean status` (ako radi — `filesys clean stop`)
3. Proveriti replikaciju: `replication status` (kontekst će se sam oporaviti, ali dobro je znati stanje pre)
4. Ako je uključen Retention Lock Compliance — može tražiti autorizaciju security officera

---

## Reference

- DDOS Administration Guide, poglavlja o CLI-ju i upravljanju korisnicima
- DD Security Configuration Guide — za `adminaccess`, politike lozinki i otvrdnjavanje
