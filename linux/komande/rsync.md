# rsync - Sinhronizacija i bezbedan prenos fajlova

## 1. Uvod i namena
`rsync` (Remote Sync) je izuzetno brza i fleksibilna CLI alatka namenjena sinhronizaciji fajlova i direktorijuma lokalno ili između udaljenih servera. Njena ključna prednost leži u *delta-transfer* algoritmu koji prenosi samo razlike (izmene) u fajlovima umesto ponovnog kopiranja celokupnog sadržaja, smanjujući mrežni saobraćaj i vreme izvršavanja.

U enterprise i disaster recovery okruženjima primarno se koristi za kreiranje inkrementalnih bekapa, migraciju podatak između storage sistema i sinhronizaciju produkcionih asseta bez prekida u radu servisa.

## 2. Sintaksa i opcije (Flags)
`rsync [opcije] Izvor Odredište`

| Opcija | Dugi oblik | Značenje / Ponašanje |
|:---:|:---|:---|
| `-a` | `--archive` | Arhivski režim (rekurzivno, čuva permisije, vlasništvo, timestamps i symlink-ove). |
| `-v` | `--verbose` | Prikazuje detaljnije informacije tokom procesa sinhronizacije. |
| `-z` | `--compress` | Kompresuje podatke u letu tokom prenosa radi uštede protoka. |
| `-P` | `--partial --progress` | Prikazuje progres prenosa pojedinačnih fajlova i omogućava nastavak prekinutog prenosa. |
| `-e` | `--rsh` | Definiše protokoli za udaljenu konekciju (npr. `-e ssh` ili sa specifičnim portom). |
| `--delete` | - | Briše fajlove na odredištu koji više ne postoje na izvoru (stvara identičan mirror). |
| `--dry-run` | `-n` | Simulira sinhronizaciju bez stvarnih izmena na disku (test režim). |
| `--exclude` | - | Isključuje fajlove ili foldere iz sinhronizacije na osnovu definisanog šablona. |

## 3. Analiza izlaza (Output Breakdown)

```bash
$ rsync -avzP --delete /var/www/html/ admin@10.0.1.50:/backup/www/html/
building file list ... done
created directory /backup/www/html
./
index.php
        12,450 100%   11.87MB/s    0:00:00 (xfr#1, to-chk=2/4)
app/config.php
         2,105 100%    2.01MB/s    0:00:00 (xfr#2, to-chk=1/4)
deleting old_file.tmp

sent 4,512 bytes  received 110 bytes  3,081.33 bytes/sec
total size is 14,555  speedup is 3.15
```

**Objašnjenje prikaza:**
- **building file list ... done:** Faza u kojoj `rsync` skenira izvornu i odredišnu strukturu radi poređenja veličina i datuma poslednje izmene.
- **index.php / app/config.php:** Lista fajlova koji se sinhronizuju jer su novi ili izmenjeni.
- **12,450 100% 11.87MB/s 0:00:00:** Veličina pojedinačnog fajla u bajtovima, procenat završenosti prenosa, trenutna brzina prenosa i preostalo vreme za taj fajl.
- **(xfr#1, to-chk=2/4):** Ukazuje da je ovo 1. preneti fajl, dok je preostalo 2 fajla za proveru od ukupno 4.
- **deleting old_file.tmp:** Označava da je navedeni fajl obrisan sa odredišta jer je ubačena opcija `--delete`, a fajl ne postoji na izvoru.
- **sent/received:** Ukupan obim saobraćaja poslat ka i primljen od udaljenog računara.
- **speedup is 3.15:** Odnos ukupne veličine podataka i stvarno prenetih bajtova (pokazuje efikasnost delta algoritma i kompresije).

## 4. Praktični primeri iz produkcije

- **Svrha:** Testiranje sinhronizacije (Dry-Run) pre stvarne migracije kako bi se izbeglo slučajno brisanje podataka.
- **Komanda:** 
  ```bash
  rsync -avzP --dry-run --delete /data/storage/ admin@192.168.1.200:/mnt/backup/
  ```
- **Objašnjenje:** Pokreće kompletan algoritam i ispisuje šta BI bilo preneto ili obrisano, bez pravljenja bilo kakvih fizičkih izmena na ciljnom serveru.

- **Svrha:** Bezbedna sinhronizacija preko nestandardnog SSH porta (npr. port 2222) uz ograničenje protoka diska/mreže.
- **Komanda:** 
  ```bash
  rsync -avzP --bwlimit=10000 -e 'ssh -p 2222' /var/backups/ user@backup.local:/remote/backups/
  ```
- **Objašnjenje:** Sređuje prenos preko redefinisanog SSH porta koristeći opciju `-e`, dok `--bwlimit=10000` ograničava brzinu na 10 MB/s kako se ne bi zauzela kompletna mrežna magistrala.

- **Svrha:** Bekap aplikacije uz ignorisanje keš foldera i privremenih log fajlova.
- **Komanda:** 
  ```bash
  rsync -avz --exclude='*.tmp' --exclude='cache/' --exclude='.git/' /app/src/ /backup/src/
  ```
- **Objašnjenje:** Omogućava izostavljanje neželjenih struktura koje nepotrebno uvećavaju bekap arhivu.

## 5. Pro Tips i "Gotchas" (Zamke)
- **Kosa crta (Slash Gotcha):** Obratite pažnju na završnu kosu crtu! `/src/` sinhronizuje **sadržaj** tog direktorijuma unutar odredišta. `/src` (bez kose crte) kreira **sam direktorijum `src`** unutar odredišnog direktorijuma.
- Budite izuzetno oprezni sa opcijom `--delete`. Ako pogrešno navedete izvornu putanju (npr. prazan folder), možete obrisati celokupno bekap odredište. Uvek prvo testirajte sa `--dry-run`.
- Prilikom prenosa milion malih fajlova, `rsync` može potrošiti veliku količinu RAM memorije za kreiranje "file list"-a pre samog prenosa.
