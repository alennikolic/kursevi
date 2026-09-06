# 03 — Sistem, alerti, logovi i podrška

Sve što se tiče praćenja zdravlja uređaja, komunikacije sa Dell podrškom i
nadogradnje DDOS-a.

---

## 3.1 Alerti

DD generiše alerte za hardver, kapacitet, file system, replikaciju i bezbednost.

```
alerts show current                      # aktivni alerti
alerts show current-hardware             # samo hardverski
alerts show history                      # podrazumevano poslednja 24h
alerts show history last 7 days
alerts show history class <klasa>
alerts show daily                        # dnevni pregled
```

Alert ima klasu, severity, vreme nastanka i opis. Ostaje "aktivan" dok se
uzrok ne otkloni — DD ga sam povlači kada stanje prođe.

| Severity | Reakcija |
|---|---|
| `EMERGENCY` / `CRITICAL` | Otvara se case kod Dell-a, ide se odmah |
| `WARNING` | Istraga isti dan. Najčešće kapacitet ili replikacija |
| `NOTICE` / `INFO` | Informativno |

### Ko dobija obaveštenja

```
alerts notify-list show                  # sve notifikacione grupe
alerts notify-list show <ime-grupe>
```

⚠️ Izmene:

```
alerts notify-list create <ime> class <klasa> severity <nivo>
alerts notify-list add <ime> emails <adresa>
alerts notify-list del <ime> emails <adresa>
alerts notify-list destroy <ime>
alerts notify-list test <ime>            # slanje test mejla
```

> **Provera koja se često preskoči:** `alerts notify-list test`. Uređaj koji
> generiše savršene alerte a ne može da pošalje mejl je isto što i uređaj bez
> alerta. Testirajte nakon svake izmene SMTP-a ili mrežne konfiguracije.

SMTP konfiguracija:

```
config show mailserver
config set mailserver <ime-ili-ip>       # ⚠️
```

---

## 3.2 AutoSupport i ASUP

AutoSupport je dnevni izveštaj o stanju sistema koji ide ka Dell-u i,
opciono, ka vašem timu. Na osnovu njega Dell proaktivno otvara case-ove
za hardverske kvarove.

```
autosupport show all                     # kompletna konfiguracija
autosupport show history
autosupport show schedule
autosupport show report                  # sadržaj tekućeg izveštaja
```

⚠️ Izmene:

```
autosupport add alert-summary emails <adresa>
autosupport add asup-detailed emails <adresa>
autosupport del asup-detailed emails <adresa>
autosupport set schedule daily <vreme>
autosupport send [emails <adresa>]       # ručno slanje
autosupport test email <adresa>
```

> Ako ASUP ne stiže do Dell-a, uređaj je efektivno bez proaktivne podrške.
> `autosupport show history` pokazuje da li su poslednji izveštaji otišli.

---

## 3.3 Support bundle

Kada se otvara case kod Dell-a, prvo što će tražiti je support bundle.

```
support bundle list                      # postojeći bundle-ovi
support bundle create                    # ⚠️ generisanje (traje, troši /ddvar)
support bundle create <ime>
support bundle delete <ime>              # ⚠️
support upload <ime>                     # slanje ka Dell-u
```

**Napomene:**
- Bundle se pravi u `/ddvar`. Na sistemu sa punim `/ddvar` neće uspeti —
  prvo obrisati stare (`support bundle list` → `support bundle delete`).
- Generisanje na velikom sistemu traje i može privremeno opteretiti uređaj.
- Ako `support upload` ne prolazi (nema izlaza ka internetu), bundle se
  preuzima preko GUI-ja ili SCP-a i ručno kači na case.

Provera veze ka Dell podršci:

```
support show all
support notification show
```

---

## 3.4 Logovi

```
log list                                 # koji log fajlovi postoje
log view                                 # tekući messages log
log view <ime-fajla>                     # npr. log view messages.engineering
log watch                                # praćenje uživo, kao tail -f
log watch <ime-fajla>
```

Najkorisniji logovi:

| Log | Sadržaj |
|---|---|
| `messages` | Glavni sistemski log — sve bitno prolazi ovuda |
| `messages.engineering` | Detaljniji, za dublju dijagnostiku |
| `space.log` | Istorija zauzeća i cleaning ciklusa |
| `debug/` | Podlogovi pojedinačnih servisa |

Slanje logova na eksterni syslog server:

```
log host show
log host add <ip-ili-fqdn>               # ⚠️
log host del <ip-ili-fqdn>               # ⚠️
log host enable                          # ⚠️
```

> Za CR vault okruženja i za sisteme pod revizijom, eksterni syslog nije opcija
> nego zahtev — log koji živi samo na kompromitovanom uređaju nije dokaz.

---

## 3.5 Hardverski status

```
system show hardware
system status
disk show state
disk show hardware
disk show reliability-data
enclosure show all
enclosure show topology
```

Napajanje, ventilatori, temperature:

```
system show environment                  # temperature, ventilatori
system show ports
```

Lociranje uređaja ili diska u rack-u:

```
system show serialno
disk beacon <enclosure>.<disk>
```

Detaljnije o diskovima i storage-u → **poglavlje 04**.

---

## 3.6 Licence

```
license show
elicense show
elicense update                          # ⚠️ interaktivno, unosi se licencni fajl
```

`license show` je korak koji se preskače, a objašnjava zašto pola komandi iz
priručnika "ne radi". Bez odgovarajuće licence nema Replication, Retention Lock,
Cloud Tier, Encryption ni VTL funkcionalnosti.

---

## 3.7 Nadogradnja DDOS-a

🛑 Prekida servis. Planira se kroz change proceduru.

### Priprema

```
system show version                      # trenutna verzija
system show detailed-version
license show
filesys show space                       # /ddvar mora imati mesta za paket
alerts show current                      # sistem mora biti čist pre upgrade-a
disk show state                          # nema failed diskova
replication status                       # stanje konteksta pre upgrade-a
```

**Obavezno pre nadogradnje:**
1. Proveriti **kompatibilnost** sa backup aplikacijama i sa CR verzijom
   (Support Matrix), i **putanju nadogradnje** — ne ide se uvek direktno
   sa verzije na verziju
2. Pročitati Release Notes za ciljnu verziju
3. Napraviti i sačuvati support bundle
4. Kod replikacije: odredište se po pravilu nadograđuje **pre** izvora
5. Kod CR vault-a: provera Support Matrix-a i pauziranje politika (**poglavlje 12**)

### Postupak

```
system upgrade package list              # koji su paketi na sistemu
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
```

---

## 3.8 Reboot, gašenje i oporavak

🛑

```
system reboot
system poweroff
system show uptime
```

Pre reboot-a, kontrolna lista:

1. `ddboost show connections`, `nfs show active`, `cifs show active` — nema aktivnih backup-a
2. `filesys clean status` — cleaning ne radi (ako radi: `filesys clean stop`)
3. `replication status` — zabeležiti stanje
4. Ako je CR vault: `crcli jobs list -t protection -running` (**poglavlje 12**)
5. Ako je Retention Lock Compliance: može tražiti autorizaciju security officera

File system operacije:

```
filesys status
filesys enable                           # ⚠️
filesys disable                          # 🛑 prekida sav pristup podacima
filesys restart                          # 🛑
```

> `filesys disable` na sistemu sa uključenim Retention Lock Compliance režimom
> traži autorizaciju security officera. To je namerno.

---

## 3.9 Konfiguracija — snimanje stanja

DDOS nema "backup konfiguracije" u smislu jednog fajla koji se vrati.
Ono što se radi je snimanje outputa relevantnih `show` komandi pre svake izmene.

Minimalni set koji vredi čuvati periodično:

```
config show all
net show settings
net show hardware
route show table
user show list
adminaccess show
mtree list
quota capacity show
snapshot schedule show
replication show config
filesys clean show config
alerts notify-list show
autosupport show all
license show
```

Preko SSH-a u fajl:

```bash
for cmd in "config show all" "net show settings" "mtree list" \
           "replication show config" "quota capacity show"; do
    echo "===== $cmd ====="
    ssh sysadmin@dd01 "$cmd"
done > dd01-config-$(date +%F).txt
```

> Ovo je jeftino, traje sekunde i jedini je način da posle greške znate
> kako je stvar izgledala pre nje.

---

## Reference

- DDOS Administration Guide — poglavlja o monitoringu, alertima i nadogradnji
- DDOS Command Reference Guide — sekcije `alerts`, `autosupport`, `support`, `log`, `system`
- Release Notes za ciljnu verziju pre svake nadogradnje
