# 01 — Osnove i pristup

Kako se doći do CLI-ja, kako se snaći u njemu i ko šta sme da radi.
Provereno prema DD OS 8.6 Command Reference Guide.

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
replication status
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

Ako komanda iz ovog priručnika ne prolazi, prvo `help <komanda>` na samom uređaju.

### Podrazumevani alijasi

DDOS dolazi sa skupom alijasa koji podsećaju na UNIX komande. Korisni su i
dobro je znati na šta se mapiraju, jer u dokumentaciji ćete naći punu komandu.

| Alijas | Puna komanda |
|---|---|
| `df` | `filesys show space` |
| `sysstat` | `system show stats` |
| `iostat` | `system show stats view iostat interval` |
| `netstat` | `net show stats` |
| `nfsstat` | `nfs show stats` |
| `uptime` | `system show uptime` |
| `uname` | `system show version` |
| `hostname` | `net show hostname` |
| `date` | `system show date` |
| `who` | `user show active` |
| `ping` | `net ping` |
| `traceroute` | `net route trace` |
| `ifconfig` | `net config` |
| `passwd` | `user change password` |
| `reboot` | `system reboot` |
| `poweroff` | `system poweroff` |

Sopstveni alijasi:

```
alias show
alias add <ime> "<komanda>"      # ⚠️ vidljiv samo korisniku koji ga je napravio
alias del <ime>                  # ⚠️
alias reset                      # ⚠️ briše korisničke, vraća podrazumevane
```

### Izvršavanje bez interaktivne sesije

DDOS prihvata komandu kao SSH argument, što je osnova za svaku automatizaciju:

```bash
ssh sysadmin@dd01 "filesys show space"
ssh sysadmin@dd01 "replication status"
```

Za neinteraktivni rad postavite SSH ključ:

```
adminaccess add ssh-keys [user <username>]      # ⚠️ traži paste ključa, pa Enter i Ctrl-D
adminaccess show ssh-keys [user <username>]
adminaccess del ssh-keys <index> [user <username>]     # ⚠️
```

> `adminaccess add` prima **isključivo** `ssh-keys`. Za ograničavanje pristupa
> po IP adresi koristi se `net filter` — vidi 1.5.
>
> Na HA sistemima ključu treba 30 sekundi do minut da se propagira na standby čvor.

---

## 1.3 Role i korisnici

| Rola | Šta može |
|---|---|
| `admin` | Sve, uključujući konfiguraciju sistema |
| `limited-admin` | Kao admin, ali ne može da isključi file system niti da radi destruktivne sistemske operacije |
| **`security`** | Security officer. Autorizuje operacije zaštićene Retention Lock Compliance režimom. Ne administrira sistem |
| `backup-operator` | Snapshot-ovi i osnovne backup operacije |
| `user` | Read-only pregled |
| `none` | Nema pristup CLI-ju; koristi se za DD Boost naloge |
| `mfa-troubleshooting-user` | Pomoćna rola za probleme sa višefaktorskom autentikacijom |
| `system-user` | Sistemska rola |
| `tenant-admin` / `tenant-user` | Secure Multi-Tenancy, ograničeno na svoj tenant unit |

> **Rola se zove `security`, ne `security-officer`.** Ovo je čest izvor greške,
> jer se u dokumentaciji i u razgovoru koristi izraz "security officer".
> `authorization policy set security-officer` jeste ispravan naziv **politike**,
> ali rola korisnika je `security`.

Komande:

```
user show list                                  # svi nalozi i njihove role
user show detailed [<username>]
user show active                                # ko je trenutno prijavljen
user add <username> [uid <uid>] [role <rola>]   # ⚠️
user change role <username> <nova-rola>         # ⚠️
user change password [<username>]               # ⚠️
user enable <username>
user disable <username>                         # ⚠️
user del <username>                             # 🛑
```

Ograničenja koja treba znati:

- `user change role` prima `{admin | limited-admin | user | backup-operator | none | system-user}` —
  **rola `security` se ne može dodeliti izmenom postojećeg naloga.**
- Prvog korisnika sa rolom `security` kreira `admin`. Posle toga **samo
  korisnici sa rolom `security` mogu dodavati ili brisati druge `security` naloge.**
- Nakon kreiranja `security` naloga mora se uključiti autorizacija:
  `authorization policy set security-officer enabled` ⚠️
- Imena `root` i `admin` su rezervisana.

**Preporuka:** za monitoring i skripte koristite zaseban nalog sa rolom `user`.
Read-only provere iz poglavlja 02 ne traže `admin`.

---

## 1.4 Upravljanje pristupom (adminaccess)

```
adminaccess show                                # koji su protokoli otvoreni
adminaccess option show
```

⚠️ Izmene:

```
adminaccess enable {ssh | http | https | ftp | ftps | telnet | web}
adminaccess disable {ssh | http | https | ftp | ftps | telnet | web}
adminaccess reset <opcija>
```

**Otvrdnjavanje — minimum:**

```
adminaccess disable telnet       # ⚠️
adminaccess disable ftp          # ⚠️
adminaccess disable http         # ⚠️ ostaviti samo https
adminaccess show
```

Politike sesija i prijave ⚠️:

```
adminaccess option show
adminaccess option set login-max-attempts <n>
adminaccess option set login-unlock-timeout <sekunde>
adminaccess option set login-max-active <n>
adminaccess option set cipher-list <lista>
adminaccess option set tls-version <verzija>
adminaccess option reset {login-max-attempts | login-unlock-timeout | login-max-active | cipher-list | tls-version | password-auth | password-hash | strict-login-auth | cert-login-port-status}
```

---

## 1.5 Ograničavanje pristupa po IP adresi — net filter

Ovo je mehanizam za "sme da se konektuje samo sa ovih adresa". **Nije `adminaccess`.**

```
net filter show                                 # trenutna pravila
net filter config show
```

⚠️ Izmene:

```
net filter add [seq-id <n>] operation {allow | block} \
    protocol tcp [ports <port>] [except-clients <ip1>,<ip2>]

net filter del <seq-id>
net filter reset
```

Primer iz Dell dokumentacije — dozvoli upravljanje preko porta 3009 samo
navedenim klijentima:

```
net filter add operation block protocol tcp ports 3009 except-clients <ip1>,<ip2>
```

Dodatno:

```
net filter auto-list add ports {all}            # ⚠️ dozvoli samo portove sa listen threadom
net filter config set admin-interface <ifname> [client <host>] [ports <port>]   # ⚠️
net filter clear stats
```

> `net alias` adrese se **ne mogu** dodati u `net filter` niti u iptables pravila.

---

## 1.6 Osnovna orijentacija na nepoznatom uređaju

Kada prvi put sednete na tuđi ili nasleđen sistem, ovim redom:

```
system show version                  # verzija DDOS-a
system show detailed-version         # verzije pojedinačnih komponenti
system show modelno                  # model
system show serialno [detailed]      # serijski broj za support case
system show uptime
elicense show                        # koje su funkcije licencirane
net show settings                    # mrežna konfiguracija
net show config [<ifname>]
mtree list                           # čemu ovaj uređaj služi
replication show config              # da li je izvor ili odredište replikacije
filesys show space                   # koliko je pun
alerts show current
```

> **`license show` ne postoji.** Komanda je `elicense show [licenses | locking-id |
> software-id | scheme | all]`. Bez tog koraka ne znate da li uređaj uopšte ima
> Retention Lock, Replication, Cloud Tier ili Encryption licencu, pa ćete tražiti
> komande koje neće raditi.

---

## 1.7 Vreme, NTP i DNS

Pogrešno vreme lomi Retention Lock, CIFS/Kerberos autentikaciju i sertifikate.

```
system show date
ntp status
ntp show config
net show dns
timezone show
```

⚠️ Izmene:

```
ntp add timeserver <ip-ili-fqdn>
ntp del timeserver <ip-ili-fqdn>
ntp enable
ntp disable
ntp reset
net set dns <ip1> <ip2>
timezone set <zona>                  # ⚠️ traži reboot
```

Za zaštićenu NTP komunikaciju:

```
ntp secure add authentication-key <key-ID> SHA1     # ⚠️
ntp secure add timeserver <server> <key-ID>         # ⚠️
ntp secure add trusted-key <key-ID>                 # ⚠️
```

> **Upozorenje:** na sistemu sa uključenim Retention Lock Compliance režimom,
> izmena sistemskog vremena je zaštićena operacija i traži autorizaciju
> security officera. To je namerno — pomeranje sata unazad bi zaobišlo lock.
> Vidi i `system show clock-violation-action`, `system show date-change-frequency`
> i `system show date-change-limit`.

---

## 1.8 Konfiguracioni wizard

```
config show all                       # pregled bez izmene
config setup                          # ⚠️ interaktivni wizard
```

Wizard prolazi kroz mrežu, DNS, vreme, mejl obaveštenja, licence i osnovne
protokole. Na produkcijskom sistemu ne pokrećite ga bez razloga — lako se
slučajno prepiše postojeća konfiguracija.

---

## 1.9 Šta CLI ne radi

- **Nema pristup shell-u.** `se` (Support Engineering) mod je rezervisan za Dell
  support i ne koristi se bez otvorenog case-a.
- **Nema `rm`, `ls`, `cd` nad podacima.** Podacima se pristupa preko protokola
  (NFS, CIFS, DD Boost). Za kopiranje unutar uređaja postoji `filesys fastcopy`.
- **Nema rollback konfiguracije.** Ne postoji "undo". Pre svake ⚠️ komande
  snimite trenutno stanje relevantnom `show` komandom (vidi 3.9).

---

## 1.10 Reboot i gašenje

🛑 Sve u ovoj sekciji prekida servis.

```
system reboot
system poweroff
system show uptime                    # provera nakon vraćanja
```

Pre reboot-a obavezno:

1. Nema aktivnog backup-a: `ddboost show connections`, `nfs show active`, `cifs show active`
2. Cleaning ne radi: `filesys clean status` (ako radi — `filesys clean stop` ⚠️)
3. Stanje replikacije zabeleženo: `replication status`
4. Ako je CR vault: `crcli jobs list` i `crcli vault state` (**poglavlje 12**)
5. Ako je uključen Retention Lock Compliance — može tražiti autorizaciju security officera

---

## Reference

- DD OS 8.6 Command Reference Guide — poglavlja `adminaccess`, `alias`, `user`, `net`, `ntp`, `config`
- DD OS 8.6 Administration Guide — upravljanje korisnicima i pristupom
- DD Security Configuration Guide — otvrdnjavanje, politike lozinki, sertifikati
