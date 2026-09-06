# 03 — Sistem, alerti, logovi i podrška

Praćenje zdravlja uređaja, komunikacija sa Dell podrškom i nadogradnja DDOS-a.

---

## 3.1 Alerti

```
alerts show current [local] [tenant-unit <tu>]
alerts show current-detailed [alert-id <lista>]
alerts show all [local]
alerts show daily [local]
alerts show history [last <n> {hours | days | weeks}]
alerts show history-detailed [last <n> {hours | days | weeks}]
alerts clear alert-id <lista>                # ⚠️
```

| Severity | Reakcija |
|---|---|
| `EMERGENCY` / `CRITICAL` | Otvara se case kod Dell-a, ide se odmah |
| `WARNING` | Istraga isti dan. Najčešće kapacitet ili replikacija |
| `NOTICE` / `INFO` | Informativno |

### Notifikacione grupe

```
alerts notify-list show [group <ime> | email <adresa> | tenant-unit <tu>]
```

⚠️ Izmene:

```
alerts notify-list create <ime-grupe> {class <lista> [severity <nivo>] | tenant-unit <tu>}
alerts notify-list add <ime-grupe> {[class <lista> [severity <nivo>]] [emails <adrese>]}
alerts notify-list del <ime-grupe> {[class <lista>] [emails <adrese>]}
alerts notify-list destroy <ime-grupe>
alerts notify-list reset
alerts notify-list test {group <ime> | email <adresa>}
```

> **Provera koja se često preskoči:** `alerts notify-list test`. Uređaj koji
> generiše savršene alerte a ne može da pošalje mejl je isto što i uređaj bez
> alerta. Testirajte nakon svake izmene SMTP-a ili mrežne konfiguracije.

Automatsko otvaranje servisnih zahteva:

```
alerts config show auto-service-request
alerts config enable auto-service-request                    # ⚠️
alerts config disable auto-service-request <trajanje> hr     # ⚠️
```

SMTP:

```
config show mailserver
config set mailserver <ime-ili-ip>       # ⚠️
```

---

## 3.2 AutoSupport i ASUP

Dnevni izveštaj o stanju sistema koji ide ka Dell-u. Na osnovu njega Dell
proaktivno otvara case-ove za hardverske kvarove.

```
autosupport show all
autosupport show history
autosupport show schedule
autosupport show report
```

⚠️ Izmene:

```
autosupport add {alert-summary | asup-detailed} emails <adresa>
autosupport del {alert-summary | asup-detailed} emails <adresa>
autosupport set schedule daily <vreme>
autosupport send [emails <adresa>]
autosupport test email <adresa>
```

> Ako ASUP ne stiže do Dell-a, uređaj je efektivno bez proaktivne podrške.
> `autosupport show history` pokazuje da li su poslednji izveštaji otišli.

---

## 3.3 Support bundle i veza sa Dell podrškom

`support bundle create` **traži tip bundle-a** — ne postoji kao gola komanda.

```
support bundle list
support bundle create default [with-files <lista>] [start-date yyyy-mm-dd [number-of-days <n>]]   # ⚠️
support bundle create mini [with-files <lista>]      # ⚠️ manji, brži
support bundle create {files-only <lista> | traces-only}    # ⚠️
support bundle delete {<lista> | all}                # ⚠️
```

Veza ka Dell podršci (SupportAssist / ConnectEMC):

```
support connectivity config show
support connectivity contacts show
support connectivity config modify mode {direct | gateway {ipaddress <ips>}}   # ⚠️
support connectivity register access-key <key> pin <pin> mode {direct | ...}   # ⚠️
support connectivity remote-access {enable [ipaddress <vrednost>] | disable}   # ⚠️
support connectivity hardreset                                                # ⚠️
support connectemc ...
```

> Ne postoji `support upload`. Bundle se šalje kroz konfigurisanu
> SupportAssist/ConnectEMC vezu, preuzima se preko GUI-ja, ili se kopira
> sa uređaja i ručno kači na case.

**Napomene:**
- Bundle se pravi u `/ddvar`. Na punom `/ddvar` neće uspeti — prvo obrisati stare.
- Generisanje na velikom sistemu traje i može privremeno opteretiti uređaj.
- Za probleme sa replikacijom pravi se bundle sa **obe** strane.

---

## 3.4 Logovi

```
log list                                 # koji log fajlovi postoje
log view [<filename>]                    # tekući messages log
log view messages.engineering
log watch [<filename>]                   # praćenje uživo
log view access-info [authentication-failures {all | known-users | unknown-users}]
log view audit-info [authorization-errors | all-errors] [user <ime>]
```

Najkorisniji logovi:

| Log | Sadržaj |
|---|---|
| `messages` | Glavni sistemski log |
| `messages.engineering` | Detaljniji, za dublju dijagnostiku |
| `space.log` | Istorija zauzeća i cleaning ciklusa |

Eksterni syslog:

```
log host show
log host add <ip-ili-fqdn>               # ⚠️
log host del <ip-ili-fqdn>               # ⚠️
log host enable                          # ⚠️
log host disable                         # ⚠️
log host reset                           # ⚠️
```

Zaštićeni syslog (TLS):

```
log secure-syslog host show
log secure-syslog host add <host>        # ⚠️
log secure-syslog host enable            # ⚠️
log secure-syslog server-port reset      # ⚠️
```

> Za CR vault okruženja i sisteme pod revizijom, eksterni syslog nije preporuka
> nego zahtev — log koji živi samo na kompromitovanom uređaju nije dokaz.

---

## 3.5 Hardverski status

```
system show hardware
system status
system show ports
disk show state
disk show hardware
disk show reliability-data
disk show performance
disk port show {stats | summary}
disk multipath status
enclosure show all [<enclosure>]
enclosure show chassis
enclosure show controllers <enclosure>
enclosure show firmware
enclosure show fans [<enclosure>]
enclosure show cpus [<enclosure>]
enclosure show memory [<enclosure>]
enclosure show io-cards [<enclosure>]
enclosure show misconfiguration
```

Lociranje u rack-u:

```
disk beacon {<enclosure-id>.<disk-id> | <serialno>}
enclosure beacon <enclosure>
```

Ručno označavanje diska kao neispravnog (🛑, samo po uputstvu Dell podrške):

```
disk fail <enclosure-id>.<disk-id>
```

---

## 3.6 Licence

```
elicense show [licenses | locking-id | software-id | scheme | all]
```

⚠️ Izmene:

```
elicense update [check-only] [<license-file>]
elicense download [check-only]
elicense reset [restore-evaluation]
elicense register [lac]                  # SupportAssist dinamičko licenciranje
elicense deregister
elicense expand [lac]
```

> **`license show` ne postoji.** Komanda je `elicense show`. Ovo je prvi korak
> pri preuzimanju nepoznatog uređaja — bez odgovarajuće licence nema Replication,
> Retention Lock, Cloud Tier, Encryption ni VTL funkcionalnosti.

---

## 3.7 Nadogradnja DDOS-a

🛑 Prekida servis. Planira se kroz change proceduru.

### Priprema

```
system show version
system show detailed-version
elicense show
filesys show space                       # /ddvar mora imati mesta za paket
alerts show current                      # sistem mora biti čist
disk show state                          # nema failed diskova
replication status
```

**Obavezno pre nadogradnje:**

1. Proveriti kompatibilnost u **E-Lab Navigator** (Data Domain Compatibility Matrix) —
   sa backup aplikacijama, DD Boost agentima i CR verzijom
2. Proveriti **putanju nadogradnje** — ne ide se uvek direktno sa verzije na verziju
3. Pročitati Release Notes za ciljnu verziju
4. Napraviti i sačuvati support bundle
5. Kod replikacije: odredište se po pravilu nadograđuje **pre** izvora
6. Kod CR vault-a: provera Support Matrix-a i pauziranje politika (**poglavlje 12**)

### Postupak

```
system upgrade package list
system upgrade package delete <ime>      # ⚠️ čišćenje starih
# preuzimanje paketa preko GUI-ja ili scp u /ddvar/releases
system upgrade start <ime-paketa>        # 🛑
system upgrade status
system upgrade history
```

Nakon nadogradnje:

```
system show version
alerts show current
filesys status
replication status
mtree list
elicense show
ddboost status
```

---

## 3.8 Reboot, gašenje i file system

🛑

```
system reboot
system poweroff
system show uptime
```

Kontrolna lista pre reboot-a:

1. `ddboost show connections`, `nfs show active`, `cifs show active` — nema aktivnih backup-a
2. `filesys clean status` — cleaning ne radi (ako radi: `filesys clean stop` ⚠️)
3. `replication status` — zabeležiti stanje
4. Ako je CR vault: `crcli jobs list`, `crcli vault state` (**poglavlje 12**)
5. Ako je Retention Lock Compliance: može tražiti autorizaciju security officera

File system:

```
filesys status
filesys enable                           # ⚠️
filesys disable                          # 🛑 prekida sav pristup podacima
filesys restart                          # 🛑
```

> `filesys disable` na sistemu sa Retention Lock Compliance režimom traži
> autorizaciju security officera.

---

## 3.9 Snimanje konfiguracije

DDOS nema "backup konfiguracije" u smislu jednog fajla koji se vrati.
Ono što se radi je snimanje outputa relevantnih `show` komandi pre svake izmene.

Minimalni set:

```
config show all
net show settings
net show config
net show hardware
net route show tables
net filter show
user show list
adminaccess show
adminaccess option show
mtree list
quota capacity show all
snapshot schedule show
replication show config
replication throttle show all
filesys clean show config
filesys option show
alerts notify-list show
autosupport show all
elicense show
```

Preko SSH-a u fajl:

```bash
CMDS=("config show all" "net show settings" "mtree list" \
      "replication show config" "quota capacity show all" "elicense show")
for c in "${CMDS[@]}"; do
    echo "===== $c ====="
    ssh sysadmin@dd01 "$c"
done > dd01-config-$(date +%F).txt
```

> Traje sekunde i jedini je način da posle greške znate kako je stvar
> izgledala pre nje.

---

## Reference

- DD OS 8.6 Command Reference Guide — poglavlja `alerts`, `autosupport`, `support`, `log`, `system`, `elicense`, `disk`, `enclosure`
- DD OS 8.6 Administration Guide — monitoring i nadogradnja
- Release Notes za ciljnu verziju pre svake nadogradnje
- E-Lab Navigator: <https://elabnavigator.dell.com/eln/modernHomeDataProtection>
