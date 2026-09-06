# lsof - Lista otvorenih fajlova i resursa

## 1. Uvod i namena
`lsof` (List Open Files) je sistemski dijagnostički alat koji ispisuje sve otvorene fajlove u Linux operativnom sistemu i procese koji ih koriste. Pošto se u Unix arhitekturi "sve posmatra kao fajl" (uključujući regularne datoteke, direktorijume, mrežne sokete, PIPE cevovode, uređaje i deljenu memoriju), `lsof` predstavlja osnovni alat za dubinski sistem inspekciju.

U produkciji se koristi pri troubleshooting-u problema kada disk ne može da se demontira (`device is busy`), kod identifikovanja aplikacija koje prouzrokuju "Too many open files" greške ili tokom bezbednosne analize i detekcije sumnjivih konekcija na kompromitovanim serverima.

## 2. Sintaksa i opcije (Flags)
`lsof [opcije] [fajl|direktorijum]`

| Opcija | Dugi oblik | Značenje / Ponašanje |
|:---:|:---|:---|
| `-i` | - | Prikazuje mrežne konekcije (IPv4/IPv6, portove, protokole). |
| `-p` | - | Filtrira prikaz samo za zadati proces ID (PID). |
| `-u` | - | Prikazuje fajlove koje je otvorio navedeni korisnik. |
| `-c` | - | Filtrira izlaz po imenu izvršnog procesa/komande. |
| `+D` | - | Rekurzivno pretražuje sve otvorene fajlove unutar definisanog direktorijuma. |
| `-t` | - | Vraća isključivo PID-ove bez zaglavlja (pogodno za skripte i `kill` naredbe). |
| `+L1` | - | Prikazuje otvorene fajlove koji su obrisani sa fajlsistema ali još drže prostor na disku. |

## 3. Analiza izlaza (Output Breakdown)

```bash
$ sudo lsof -i :80
COMMAND   PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nginx    1120     root    6u  IPv4  23412      0t0  TCP *:http (LISTEN)
nginx    1121 www-data    6u  IPv4  23412      0t0  TCP *:http (LISTEN)
nginx    1121 www-data    7u  IPv4  45102      0t0  TCP 192.168.1.10:http->10.0.0.5:52134 (ESTABLISHED)
```

**Objašnjenje prikaza:**
- **COMMAND:** Naziv procesa koji drži fajl otvorenim (npr. `nginx`).
- **PID:** Process ID broj zadatog procesa.
- **USER:** Korisnički nalog u čijem vlasništvu se izvršava proces.
- **FD (File Descriptor):** Broj i režim otvaranja fajla. Primeri: `cwd` (current working directory), `txt` (program text/code), `mem` (memory mapped file), `6u` (deskriptor broj 6 otvoren u read-write `u` režimu; `r` za read, `w` za write).
- **TYPE:** Tip fajla. Npr. `REG` (regularni fajl), `DIR` (direktorijum), `CHR` (character device), `FIFO` (pipe), `IPv4`/`IPv6` (mrežni soket).
- **DEVICE:** Identifikacioni broj uređaja/diska na kome se fajl nalazi.
- **SIZE/OFF:** Veličina fajla u bajtovima ili pomak (offset) unutar fajla.
- **NODE:** Inode broj fajla na fajlsistemu.
- **NAME:** Puna putanja do fajla, opis interfejsa ili mrežna konekcija (`LokalnaIP:Port->UdaljenaIP:Port`).

## 4. Praktični primeri iz produkcije

- **Svrha:** Pronalaženje procesa koji sprečava unmount particije (`device is busy`).
- **Komanda:** 
  ```bash
  sudo lsof +D /mnt/data/
  ```
- **Objašnjenje:** Pregledava ceo mount point `/mnt/data/` i ispisuje sve PID-ove i aplikacije koji drže otvorene fajlove na toj particiji.

- **Svrha:** Pronalaženje obrisanih fajlova koje proces i dalje drži u memoriji i time zauzima disk prostor.
- **Komanda:** 
  ```bash
  sudo lsof +L1
  ```
- **Objašnjenje:** Prikazuje fajlove čiji je unutrašnji brojač linkova pao na 0 (obrisani sa fajlsistema), ali process i dalje ima otvoren FD. Ovo je čest uzrok zašto `df -h` pokazuje popunjen disk iako `du` ne prikazuje te fajlove.

- **Svrha:** Ubijanje svih procesa koji otvaraju konekcije na specifičnom portu (npr. port 8080).
- **Komanda:** 
  ```bash
  sudo kill -9 $(sudo lsof -t -i :8080)
  ```
- **Objašnjenje:** Opcija `-t` vraća samo PID brojeve koji se potom direktno prosleđuju `kill` komandi radi oslobađanja porta.

## 5. Pro Tips i "Gotchas" (Zamke)
- `lsof` bez opcija generiše ogromnu količinu podataka jer ispisuje hiljade otvorenih resursa svih procesa na sistemu, što može stvoriti značajan I/O i CPU overhead. Uvek koristite precizne filtere (`-p`, `-i`, `-u`).
- Za inspekciju procesa drugih korisnika neophodno je pokretanje komande sa `sudo` privilegijama.
- Ako pretraga mrežnih resursa uspori rad komande, isključite razrešavanje naziva portova i IP adresa korišćenjem `-i -n -P` flagova.
