# journalctl - Napredno pretraživanje i analiza systemd logova

## 1. Uvod i namena
`journalctl` je komandna linija namenjena pretraživanju, analizi i parsiranju centralizovanih binarnih logova koje prikuplja `systemd-journald` servis. Za razliku od tradicionalnog `syslog` sistema gde su logovi razbacani u tekstualnim datotekama u `/var/log/`, `journalctl` prikuplja podatke iz kernela, initrd-a, sistemskih servisa i STDOUT/STDERR izlaza svih kontejnera/aplikacija pod upravljanjem systemd-a na jednom mestu.

U produkcionim okruženjima predstavlja primarni alat za analizu padova sistema (system crashes), dijagnostiku problema sa podizanjem servisa (failed units) i bezbednosni audit korisničkih i sistemskih događaja.

## 2. Sintaksa i opcije (Flags)
`journalctl [opcije] [MEČEVI]`

| Opcija | Dugi oblik | Značenje / Ponašanje |
|:---:|:---|:---|
| `-u` | `--unit` | Prikazuje logove samo za definisanu systemd jedinicu/servis (npr. `docker.service`). |
| `-f` | `--follow` | Prati najnovije logove u realnom vremenu (slično kao `tail -f`). |
| `-n` | `--lines` | Prikazuje samo poslednjih N linija loga (podrazumevano je 10). |
| `-p` | `--priority` | Filtrira logove po nivou kritičnosti (0:emerg, 1:alert, 2:crit, 3:err, 4:warning, 5:notice, 6:info, 7:debug). |
| `-b` | `--boot` | Prikazuje logove nastale od odabranog (ili trenutnog `-b 0`) pokretanja sistema. |
| `--since` / `--until` | - | Vremensko filtriranje događaja (prihvata formata `YYYY-MM-DD HH:MM:SS` ili "1 hour ago"). |
| `-k` | `--dmesg` | Prikazuje samo poruke potekle od Linux Kernela. |
| `-o` | `--output` | Formatira prikaz logova (`json-pretty`, `short-precise`, `cat`). |

## 3. Analiza izlaza (Output Breakdown)

```bash
$ journalctl -u nginx.service -n 3 -o short-precise
2026-09-06T12:15:02.341201+02:00 web-node-01 systemd[1]: Starting A high performance web server and a reverse proxy server...
2026-09-06T12:15:02.890123+02:00 web-node-01 nginx[3102]: 2026/09/06 12:15:02 [emerg] 3102#3102: bind() to 0.0.0.0:80 failed (98: Address already in use)
2026-09-06T12:15:02.891000+02:00 web-node-01 systemd[1]: nginx.service: Main process exited, code=exited, status=1/FAILURE
```

**Objašnjenje prikaza:**
- **2026-09-06T12:15:02.341201+02:00:** Precizan ISO-8601 vremenski žig (timestamp) događaja zabeležen u mikrosekundama sa vremenskom zonom.
- **web-node-01:** Hostname servera na kome je generisan log poruke.
- **systemd[1] / nginx[3102]:** Naziv procesa/servisa koji je emitovao log poruku i njegov Process ID (PID) u zagradama (`[1]`, `[3102]`).
- **Poruka:** Stvarni tekst loga. U drugom redu vidimo nivo `[emerg]` koji ukazuje na to da Nginx ne može da startuje jer je port 80 zauzet, dok treći red prikazuje sistemsku reakciju `systemd`-a sa statusnim kodom greške.

## 4. Praktični primeri iz produkcije

- **Svrha:** Analiza kritičnih sistemskih grešaka u poslednjih 30 minuta na produkciji.
- **Komanda:** 
  ```bash
  journalctl -p err..emerg --since "30 minutes ago"
  ```
- **Objašnjenje:** Filtrira sve poruke čiji je nivo prioriteta između `err` (3) i `emerg` (0) nastale u zadatom vremenskom okviru.

- **Svrha:** Praćenje PHP-FPM servisa u realnom vremenu uz izvoz čistog teksta poruka.
- **Komanda:** 
  ```bash
  journalctl -u php8.2-fpm -f -o cat
  ```
- **Objašnjenje:** Kombinuje `live-tailing` (`-f`) sa `-o cat` opcijom koja uklanja timestamp i hostname zaglavlja kako bi izlaz bio čitljiviji.

- **Svrha:** Analiza razloga zašto se server neočekivano restartovao u prethodnom boot ciklusu.
- **Komanda:** 
  ```bash
  journalctl -b -1 -k
  ```
- **Objašnjenje:** Opcija `-b -1` prelazi na logove iz prethodnog pokretanja sistema, dok `-k` izdvaja Kernel panike ili Hardware Out-Of-Memory (OOM) killer događaje.

## 5. Pro Tips i "Gotchas" (Zamke)
- Pošto su logovi binarni, nije moguće koristiti klasičan `cat /var/log/syslog | grep`. Koristite ugrađenu pretragu `journalctl -g "fraza"` koja podržava regularne izraze (RegEx).
- Velika količina logova može popuniti particiju. Ograničite veličinu logova ili ih počistite ručno pomoću komande: `sudo journalctl --vacuum-size=2G` ili `sudo journalctl --vacuum-time=7d`.
- Da biste videli logove svih servisa i operativnog sistema, komandu morate izvršavati pod `sudo` nalogom ili biti član `systemd-journal` korisničke grupe.
