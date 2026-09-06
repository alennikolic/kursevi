# Napredne Linux komande — `awk`

## 1. Uvod

`awk` je programski jezik za obradu tekstualnih podataka organizovanih u redove i kolone.
Za administratora je to alat koji stoji na kraju gotovo svakog cevovoda: `ss`, `lsof`, `find`,
`journalctl`, `df` i `ps` proizvode kolonske podatke, a `awk` ih filtrira, sabira i formatira.

Kada koristiti šta:

| Zadatak | Alat |
|---|---|
| pronaći redove koji sadrže tekst | `grep` |
| izdvojiti kolonu na fiksnom separatoru | `cut` |
| zameniti tekst | `sed` |
| **filtrirati po vrednosti kolone, sabirati, grupisati, formatirati** | `awk` |
| složena obrada sa strukturama podataka | Python / Perl |

Prelomna tačka je uslov nad **vrednošću polja**. Čim vam zatreba „prikaži redove gde je
peta kolona veća od 80“ ili „saberi treću kolonu po vrednosti prve“, `grep` i `cut` više
nisu dovoljni, a `awk` rešava zadatak u jednom redu.

---

## 2. Koji `awk` imate — bitna razlika

Na Linuxu postoje tri raširene implementacije i **ne podržavaju iste mogućnosti**:

```bash
awk --version 2>/dev/null | head -1 || awk -W version 2>&1 | head -1
```

| Implementacija | Gde je podrazumevana | Napomene |
|---|---|---|
| **gawk** (GNU awk) | RHEL, Fedora, Rocky, Arch | najbogatija; `gensub()`, `strftime()`, `asort()`, `IGNORECASE`, `FPAT`, `--csv` |
| **mawk** | **Debian, Ubuntu** | najbrža, ali bez `gensub`, `strftime`, `systime`, `asort`, `IGNORECASE` |
| **busybox awk** | Alpine, ugrađeni sistemi | minimalna podrška |

Provera na Debianu/Ubuntu:

```bash
readlink -f "$(command -v awk)"
```

```
/usr/bin/mawk
```

Ako skripta koristi `strftime()` ili `gensub()`, na Ubuntu će pući. Rešenja:

```bash
sudo apt install gawk
sudo update-alternatives --config awk       # postavi gawk kao podrazumevani
```

ili u skripti eksplicitno pozivajte `gawk`. U ovom uputstvu su mogućnosti specifične za GNU
označene sa **(gawk)**.

---

## 3. Model rada

`awk` čita ulaz **zapis po zapis** (podrazumevano red po red), deli svaki zapis na **polja**
(podrazumevano po belinama), pa za svaki zapis prolazi kroz listu pravila:

```
ŠABLON { AKCIJA }
```

- ako je **šablon** tačan, izvršava se **akcija**;
- šablon bez akcije znači `{ print }` — ispiši ceo red;
- akcija bez šablona se izvršava za **svaki** red.

```bash
awk '$3 > 100 { print $1, $3 }' podaci.txt
     └─ šablon  └─── akcija ───┘
```

Polja se referišu sa `$1`, `$2`, ... `$0` je ceo zapis. `$NF` je poslednje polje,
`$(NF-1)` pretposlednje.

```bash
echo "alfa beta gama delta" | awk '{print $1, $NF, NF}'
```

```
alfa delta 4
```

---

## 4. Pozivanje i opcije komandne linije

```
awk [OPCIJE] 'PROGRAM' [FAJL...]
awk [OPCIJE] -f program.awk [FAJL...]
```

| Opcija | Značenje |
|---|---|
| `-F SEP` | separator polja (isto što i `-v FS=SEP`) |
| `-v VAR=VRED` | postavi promenljivu **pre** obrade (dostupna i u `BEGIN`) |
| `-f FAJL` | učitaj program iz fajla; može se ponoviti |
| `--` | kraj opcija |
| `-E FAJL` | (gawk) učitaj program i zabrani dalje opcije — bezbednije za setuid skripte |
| `--posix` | (gawk) strogi POSIX režim, isključuje GNU proširenja |
| `--traditional` | (gawk) ponašaj se kao originalni awk |
| `--csv` | (gawk 5.3+) ispravno parsiranje CSV-a sa navodnicima i zarezima u poljima |
| `-M` | (gawk) aritmetika proizvoljne preciznosti |
| `--lint` | (gawk) upozori na sumnjive konstrukcije |
| `--profile` | (gawk) ispiši formatiran i profilisan program |

### 4.1 `-v` protiv umetanja shell promenljive

```bash
# Ispravno
T=80
df -P | awk -v prag="$T" 'NR>1 && int($5) > prag {print $6}'

# Pogrešno i opasno
df -P | awk "NR>1 && int(\$5) > $T {print \$6}"
```

Drugi oblik zahteva bekslešovanje svakog `$`, puca ako promenljiva sadrži navodnike,
i predstavlja put za ubrizgavanje koda. `-v` je uvek ispravan izbor.

Napomena: vrednosti prosleđene sa `-v` prolaze kroz obradu escape sekvenci,
pa se `\n` u vrednosti pretvara u novi red.

---

## 5. Ugrađene promenljive

| Promenljiva | Značenje |
|---|---|
| `$0` | ceo tekući zapis |
| `$1`...`$n` | pojedinačna polja |
| `NF` | **N**umber of **F**ields — broj polja u tekućem zapisu |
| `NR` | **N**umber of **R**ecord — redni broj zapisa od početka celog ulaza |
| `FNR` | redni broj zapisa unutar **tekućeg fajla** |
| `FILENAME` | ime fajla koji se trenutno obrađuje |
| `FS` | separator ulaznih polja (podrazumevano `" "`) |
| `OFS` | separator izlaznih polja (podrazumevano `" "`) |
| `RS` | separator ulaznih zapisa (podrazumevano `"\n"`) |
| `ORS` | separator izlaznih zapisa (podrazumevano `"\n"`) |
| `SUBSEP` | separator indeksa višedimenzionalnih nizova (`"\034"`) |
| `RSTART` | pozicija poklapanja posle `match()` |
| `RLENGTH` | dužina poklapanja posle `match()` |
| `CONVFMT` | format konverzije broja u string (`"%.6g"`) |
| `OFMT` | format ispisa brojeva u `print` (`"%.6g"`) |
| `ENVIRON` | niz promenljivih okruženja: `ENVIRON["HOME"]` |
| `ARGC` / `ARGV` | broj i vrednosti argumenata |
| `IGNORECASE` | (gawk) ako je različito od nule, regex ne razlikuje veličinu slova |
| `FIELDWIDTHS` | (gawk) parsiranje po fiksnim širinama umesto po separatoru |
| `FPAT` | (gawk) definiši polja **šablonom sadržaja** umesto separatorom |
| `PROCINFO` | (gawk) niz sa podacima o procesu (`PROCINFO["pid"]`) |

### 5.1 `FS` — posebno ponašanje jednog razmaka

Podrazumevana vrednost `FS=" "` **nije** doslovan razmak. Ona znači:
„deli po nizovima razmaka i tabulatora, i odbaci vodeće i prateće beline“.
Zato ovo radi bez obzira na poravnanje:

```bash
echo "   alfa    beta	gama  " | awk '{print NF, "["$1"]", "["$3"]"}'
```

```
3 [alfa] [gama]
```

Čim se `FS` postavi na bilo šta drugo, to ponašanje nestaje i prazna polja postaju značajna:

```bash
echo "a::c" | awk -F: '{print NF; print "["$2"]"}'
```

```
3
[]
```

`FS` duži od jednog znaka tumači se kao **regularni izraz**:

```bash
awk -F'[:,]' '{print $2}'          # separator je dvotačka ili zarez
awk -F' *\\| *' '{print $2}'       # uspravna crta sa opcionim razmacima
awk -F'\t' '{print $2}'            # tabulator — obavezan za TSV
```

> Znakovi `|`, `.`, `+`, `*`, `[`, `(` su specijalni u regularnom izrazu.
> Za doslovan `|` napišite `-F'[|]'` ili `-F'\\|'`.

### 5.2 `OFS` se primenjuje tek kada se zapis „dodirne“

Ovo je najčešći izvor zbunjenosti:

```bash
echo "a b c" | awk -v OFS=, '{print $0}'          # a b c   ← OFS ignorisan
echo "a b c" | awk -v OFS=, '{print $1,$2,$3}'    # a,b,c
echo "a b c" | awk -v OFS=, '{$1=$1; print $0}'   # a,b,c
```

`print $0` ispisuje **originalni** zapis. Tek kada se bilo koje polje dodeli
(uobičajen trik je `$1=$1`), `awk` ponovo sastavlja `$0` koristeći `OFS`.

### 5.3 `RS` — zapisi koji nisu redovi

```bash
# Prazan RS: zapis je pasus (blok razdvojen praznim redom)
awk 'BEGIN {RS=""; FS="\n"} {print NR": "$1}' /etc/ssh/sshd_config

# Zapis razdvojen zarezom
echo "a,b,c" | awk 'BEGIN {RS=","} {print NR, $0}'

# (gawk) RS kao regularni izraz
gawk 'BEGIN {RS="\n[0-9]+ "} {...}' log.txt
```

Kada je `RS=""`, novi red **uvek** funkcioniše i kao separator polja, uz `FS`.

---

## 6. Šabloni

| Šablon | Značenje |
|---|---|
| `BEGIN` | izvršava se pre čitanja prvog zapisa |
| `END` | izvršava se posle poslednjeg zapisa |
| `/regex/` | zapis sadrži poklapanje |
| `$3 ~ /regex/` | polje sadrži poklapanje |
| `$3 !~ /regex/` | polje ne sadrži poklapanje |
| `izraz` | bilo koji izraz koji je istinit (različit od 0 i od praznog stringa) |
| `šablon1, šablon2` | **opseg**: od reda koji odgovara prvom do reda koji odgovara drugom |
| `BEGINFILE` / `ENDFILE` | (gawk) pre/posle svakog fajla |

```bash
awk 'BEGIN {print "start"} {n++} END {print "ukupno redova:", n}' fajl.txt
awk '/ERROR/' app.log                              # kao grep
awk '$9 >= 500' access.log                          # status 500 i više
awk '$1 ~ /^10\.0\./ && $9 == 404' access.log
awk '/BEGIN RSA/,/END RSA/' kljuc.pem                # opseg
awk 'NR >= 100 && NR <= 200' veliki.log              # redovi 100-200
```

> `awk 'NR==5000000 {print; exit}'` je brži od `sed -n '5000000p'` na velikim fajlovima
> jer `exit` prekida čitanje odmah.

### 6.1 Numeričko protiv stringovnog poređenja

```bash
echo "007" | awk '$1 == 7   {print "numericki"}'      # tačno
echo "007" | awk '$1 == "7" {print "string"}'         # netačno
```

Polja su „dvostruke prirode“: ako izgledaju kao broj i porede se sa brojem, poređenje je
numeričko; ako se porede sa stringom, poređenje je stringovno. Za prinudnu konverziju:

```bash
$1 + 0 == 7        # prinudno numerički
$1 "" == "7"       # prinudno stringovno
```

Ovo je čest uzrok tihe greške kod verzija, IP adresa i vrednosti sa vodećim nulama.

---

## 7. Nizovi

Svi nizovi u `awk` su **asocijativni** — indeks je uvek string.

```bash
awk '{count[$1]++} END {for (k in count) print count[k], k}' access.log
```

| Konstrukcija | Značenje |
|---|---|
| `a[k] = v` | dodela |
| `k in a` | postoji li ključ (**ne kreira ga**) |
| `if (a[k])` | **kreira** ključ sa praznom vrednošću — koristite `in` za proveru |
| `delete a[k]` | brisanje jednog ključa |
| `delete a` | brisanje celog niza |
| `for (k in a)` | iteracija — **redosled nije definisan** |
| `length(a)` | (gawk) broj elemenata |
| `a[i,j]` | višedimenzionalno, interno spojeno preko `SUBSEP` |
| `if ((i,j) in a)` | provera višedimenzionalnog ključa |

Za sortiran izlaz sortirajte spolja ili koristite `asorti()` (gawk):

```bash
awk '{c[$1]++} END {for (k in c) printf "%8d %s\n", c[k], k}' log | sort -rn
```

```bash
gawk '{c[$1]++} END {n=asorti(c, idx); for (i=1; i<=n; i++) print idx[i], c[idx[i]]}' log
```

---

## 8. Kontrola toka

```awk
if (uslov) { ... } else if (uslov) { ... } else { ... }

for (i = 1; i <= NF; i++) { ... }
for (kljuc in niz) { ... }

while (uslov) { ... }
do { ... } while (uslov)

break        # izađi iz petlje
continue     # sledeća iteracija
next         # pređi na sledeći ZAPIS, preskoči ostala pravila
nextfile     # pređi na sledeći FAJL
exit [kod]   # prekini obradu; END blok se i dalje izvršava
```

`exit` u `END` bloku postavlja izlazni kod procesa — korisno u skriptama:

```bash
df -P | awk 'NR>1 && int($5) > 90 {found=1} END {exit !found}'
if [ $? -eq 0 ]; then echo "Postoji particija preko 90%"; fi
```

`next` je ključan za preskakanje zaglavlja i nevalidnih redova:

```bash
awk 'NR == 1 {next} NF < 5 {next} {print $3}' podaci.txt
```

---

## 9. Ugrađene funkcije

### 9.1 Stringovne

| Funkcija | Značenje |
|---|---|
| `length(s)` | dužina stringa; `length()` bez argumenta = `length($0)` |
| `substr(s, poc [, duz])` | podstring; **indeksi počinju od 1** |
| `index(s, trazeno)` | pozicija prvog pojavljivanja, 0 ako nema |
| `split(s, niz [, sep])` | podeli string u niz, vrati broj delova |
| `sub(regex, zamena [, cilj])` | zameni **prvo** poklapanje, vrati 0 ili 1 |
| `gsub(regex, zamena [, cilj])` | zameni **sva** poklapanja, vrati broj zamena |
| `match(s, regex)` | pozicija poklapanja; postavlja `RSTART` i `RLENGTH` |
| `sprintf(format, ...)` | formatiran string (ne ispisuje) |
| `tolower(s)` / `toupper(s)` | promena veličine slova |
| `gensub(regex, zam, kako [, cilj])` | (gawk) zamena koja **vraća** rezultat i podržava `\1` grupe |

Bez trećeg argumenta `sub` i `gsub` menjaju `$0` **na mestu**:

```bash
echo "a-b-c" | awk '{gsub(/-/, ":"); print}'          # a:b:c
echo "a-b-c" | awk '{n = gsub(/-/, ":"); print n}'    # 2
```

U zameni `&` označava ceo pronađeni tekst; za doslovan ampersand napišite `\\&`:

```bash
echo "greska" | awk '{sub(/greska/, "[&]"); print}'   # [greska]
```

`gensub` (gawk) je moćniji jer podržava povratne reference i ne menja original:

```bash
echo "2026-09-06" | gawk '{print gensub(/(....)-(..)-(..)/, "\\3.\\2.\\1", 1)}'
```

```
06.09.2026
```

`split` je najkorisniji za sekundarno deljenje polja:

```bash
echo "10.0.10.7:44120" | awk '{n = split($0, p, ":"); print p[1], p[2], n}'
```

```
10.0.10.7 44120 2
```

### 9.2 Numeričke

| Funkcija | Značenje |
|---|---|
| `int(x)` | odsecanje decimala prema nuli |
| `sqrt(x)`, `exp(x)`, `log(x)` | koren, eksponent, prirodni logaritam |
| `sin(x)`, `cos(x)`, `atan2(y,x)` | trigonometrija |
| `rand()` | slučajan broj u `[0,1)` |
| `srand([seed])` | inicijalizacija generatora |

`int()` je koristan trik za čitanje procenata iz izlaza drugih komandi:

```bash
df -P | awk 'NR>1 {print int($5)}'
```

`awk` konvertuje vodeći numerički deo stringa, pa `"82%"` postaje `82`.
Nema potrebe za `tr -d '%'`.

### 9.3 Vremenske (gawk)

| Funkcija | Značenje |
|---|---|
| `systime()` | trenutno UNIX vreme |
| `strftime(format [, vreme])` | formatiranje vremena |
| `mktime("YYYY MM DD HH MM SS")` | pretvaranje u UNIX vreme |

```bash
gawk 'BEGIN {print strftime("%Y-%m-%d %H:%M:%S", systime())}'
```

**Ove funkcije ne postoje u mawk** — na Debianu/Ubuntu skripta puca sa
`calling undefined function strftime`.

### 9.4 Ulaz, izlaz i sistem

| Funkcija | Značenje |
|---|---|
| `print izraz, ...` | ispis sa `OFS` između i `ORS` na kraju |
| `printf format, ...` | formatiran ispis bez automatskog novog reda |
| `getline` | pročitaj sledeći zapis (vidi 9.5) |
| `close(izraz)` | zatvori fajl ili cev |
| `system("komanda")` | pokreni komandu, vrati izlazni kod |
| `fflush([izraz])` | isprazni izlazni bafer |

### 9.5 `getline` — varijante

Ovo je najmanje intuitivan deo jezika. Šta se ažurira zavisi od oblika:

| Oblik | Čita iz | Postavlja |
|---|---|---|
| `getline` | tekućeg ulaza | `$0`, `NF`, `NR`, `FNR` |
| `getline var` | tekućeg ulaza | `var`, `NR`, `FNR` |
| `getline < "fajl"` | fajla | `$0`, `NF` |
| `getline var < "fajl"` | fajla | `var` |
| `"komanda" \| getline` | izlaza komande | `$0`, `NF`, `NR` |
| `"komanda" \| getline var` | izlaza komande | `var`, `NR` |

Povratna vrednost: `1` uspeh, `0` kraj ulaza, `-1` greška. **Uvek je proverite**,
inače petlja koja čita nepostojeći fajl radi beskonačno.

```bash
awk 'BEGIN {
  while (("uptime" | getline red) > 0) print "UPTIME:", red
  close("uptime")
}'
```

### 9.6 Preusmeravanje izlaza iz `awk`

```bash
# Podeli log po statusnom kodu u zasebne fajlove
awk '{print > ("status-" $9 ".log")}' access.log

# Dodavanje umesto prepisivanja
awk '{print >> "svi.log"}' ulaz.txt

# Slanje kroz cev
awk '{print $1 | "sort -u"}' access.log

# Na standardnu grešku
awk '{print "upozorenje" > "/dev/stderr"}'
```

> `awk` drži otvorenim svaki fajl i cev dok se ne pozove `close()`. Skripta koja otvara
> hiljade fajlova udara u limit deskriptora sa `too many open files`. Zatvarajte ih:

```bash
awk '{f = "deo-" $1 ".txt"; print > f; close(f)}' podaci.txt
```

---

## 10. `printf` — formatiranje

`print` je za brz ispis; `printf` za tabele koje neko treba da čita.

| Specifikator | Značenje |
|---|---|
| `%s` | string |
| `%d`, `%i` | ceo broj |
| `%f` | decimalni broj |
| `%e` | naučna notacija |
| `%g` | kraći od `%e` i `%f` |
| `%o`, `%x`, `%X` | oktalno, heksadecimalno |
| `%c` | znak |
| `%%` | doslovan znak procenta |

Modifikatori: `%-10s` (levo poravnato, širina 10), `%8.2f` (širina 8, dve decimale),
`%'d` (grupisanje hiljada, zavisi od locale), `%*d` (širina iz argumenta).

```bash
df -P | awk 'NR>1 {printf "%-24s %8.1f GB  %5s  %s\n", $1, $2/1048576, $5, $6}'
```

```
/dev/sda2                    98.4 GB    82%  /
/dev/sda1                     0.5 GB    31%  /boot
```

> `printf` **ne dodaje** novi red — `\n` je obavezan.
> Format string ne sme dolaziti iz nepouzdanog ulaza (isti rizik kao u C-u).

---

## 11. Praktični scenariji

### 11.1 Analiza web log fajla

Format zapisa:

```
10.0.10.7 - - [06/Sep/2026:10:12:03 +0200] "GET /index.html HTTP/1.1" 200 4523 "-" "Mozilla/5.0"
    $1                 $4                    $6      $7        $8     $9  $10
```

Najaktivnije IP adrese:

```bash
awk '{c[$1]++} END {for (ip in c) printf "%8d  %s\n", c[ip], ip}' access.log | sort -rn | head
```

```
   84213  203.0.113.44
    3102  198.51.100.9
     871  10.0.10.4
```

Raspodela statusnih kodova:

```bash
awk '{s[$9]++; t++} END {for (k in s) printf "%-5s %8d  %5.1f%%\n", k, s[k], 100*s[k]/t}' access.log | sort
```

```
200      91204   88.2%
301       4102    4.0%
404       6210    6.0%
500       1892    1.8%
```

Saobraćaj po IP adresi:

```bash
awk '{b[$1] += $10} END {for (ip in b) printf "%10.2f MB  %s\n", b[ip]/1048576, ip}' access.log | sort -rn | head
```

Najsporiji zahtevi (ako log ima `$request_time` kao poslednje polje):

```bash
awk '$NF > 1.0 {printf "%6.3f s  %-6s %s\n", $NF, $6, $7}' access.log | sort -rn | head -20
```

Greške 5xx po satu — izdvajanje sata iz `[06/Sep/2026:10:12:03`:

```bash
awk '$9 ~ /^5/ {split($4, t, ":"); h[t[2]]++} END {for (k in h) printf "%s h  %6d\n", k, h[k]}' access.log | sort
```

```
02 h      12
03 h    1841
04 h     217
```

Jedinstveni posetioci:

```bash
awk '{ip[$1]}  END {print length(ip)}' access.log            # gawk
awk '!seen[$1]++ {n++} END {print n}' access.log             # radi svuda
```

### 11.2 Nadzor popunjenosti diska

```bash
df -P | awk 'NR > 1 && int($5) >= 80 {
    printf "UPOZORENJE: %s je popunjen %s (%s)\n", $6, $5, $1
}'
```

```
UPOZORENJE: / je popunjen 82% (/dev/sda2)
UPOZORENJE: /var je popunjen 94% (/dev/sdb1)
```

> `df -P` je obavezan: bez njega `df` prelama redove kod dugih imena uređaja,
> pa se kolone pomere i `$5` više nije procenat.

Varijanta koja vraća izlazni kod za nadzorni sistem:

```bash
df -P | awk 'NR>1 && int($5) >= 90 {print; rc=1} END {exit rc}'
```

### 11.3 Analiza korisnika i naloga

Korisnici sa mogućnošću prijave:

```bash
awk -F: '$7 !~ /(nologin|false|sync)$/ {printf "%-16s uid=%-6s home=%s\n", $1, $3, $6}' /etc/passwd
```

Obični korisnici (UID 1000-59999):

```bash
awk -F: '$3 >= 1000 && $3 < 60000 {print $1}' /etc/passwd
```

Duplirani UID-ovi — ozbiljan bezbednosni propust:

```bash
awk -F: '{u[$3] = u[$3] " " $1} END {for (id in u) if (split(u[id], a, " ") > 1) print "UID " id ":" u[id]}' /etc/passwd
```

Nalozi bez lozinke:

```bash
sudo awk -F: '$2 == "" {print "PRAZNA LOZINKA: " $1}' /etc/shadow
```

### 11.4 Spajanje dva fajla — obrazac `NR == FNR`

Ovo je najvažniji napredni idiom u `awk`. `NR == FNR` je tačno **samo dok se čita prvi fajl**,
jer se tada globalni i po-fajlu brojač poklapaju.

```bash
awk -F: 'NR == FNR { ime[$3] = $1; next }
         { print (($1 in ime) ? ime[$1] : "NEPOZNAT"), $0 }' /etc/passwd uid-lista.txt
```

Praktičan primer — koliki je zbir veličina fajlova po vlasniku, sa punim imenima:

```bash
find /home -type f -printf '%U %s\n' \
| awk 'NR == FNR { FS=":"; ime[$3]=$1; next }
       { b[$1] += $2 }
       END { for (u in b) printf "%-16s %10.2f MB\n", (u in ime ? ime[u] : u), b[u]/1048576 }' \
  /etc/passwd -
```

Crtica na kraju znači „čitaj standardni ulaz kao drugi fajl“.

### 11.5 Statistika iz numeričkog niza

```bash
ping -c 100 8.8.8.8 \
| awk -F'time=' '/time=/ {
      split($2, a, " "); t = a[1]
      n++; sum += t; sumsq += t*t
      if (n == 1 || t < min) min = t
      if (t > max) max = t
  }
  END {
      if (n == 0) { print "nema odgovora"; exit 1 }
      avg = sum/n
      printf "n=%d  min=%.2f  avg=%.2f  max=%.2f  sd=%.2f ms\n", \
             n, min, avg, max, sqrt(sumsq/n - avg*avg)
  }'
```

```
n=100  min=12.41  avg=14.83  max=61.02  sd=5.17 ms
```

Isti obrazac radi za bilo koju kolonu brojeva — vreme odgovora iz logova, veličine fajlova
iz `find -printf`, trajanja poziva iz `strace -T`.

### 11.6 Uklanjanje duplikata bez sortiranja

```bash
awk '!seen[$0]++' fajl.txt
```

Ovo je najpoznatiji `awk` jednoredni program i vredi ga razumeti: `seen[$0]++` vraća
**staru** vrednost brojača. Prvi put je to `0` (netačno), pa negacija daje tačno i red se ispisuje.
Svaki sledeći put vraća broj veći od nule, negacija je netačna, red se preskače.
Za razliku od `sort -u`, **čuva originalni redosled** i ne zahteva sortiranje celog fajla.

Duplikati po jednoj koloni:

```bash
awk '!seen[$3]++' podaci.txt
```

Samo redovi koji se ponavljaju:

```bash
awk 'seen[$0]++' fajl.txt
```

### 11.7 Izdvajanje bloka teksta

```bash
awk '/^\[database\]/, /^\[/ && !/^\[database\]/' app.ini
awk '/BEGIN CERTIFICATE/,/END CERTIFICATE/' lanac.pem
```

Kontrolisanije, sa zastavicom:

```bash
awk '/^## POČETAK/ {u=1; next} /^## KRAJ/ {u=0} u' dokument.txt
```

Poslednji `u` kao šablon bez akcije znači „ako je zastavica postavljena, ispiši red“.

### 11.8 Obrada izlaza drugih komandi iz serije

```bash
# Broj TCP soketa po stanju (nastavak na ss)
ss -tan | awk 'NR>1 {s[$1]++} END {for (k in s) printf "%-12s %6d\n", k, s[k]}' | sort -k2 -rn

# Procesi po broju otvorenih deskriptora (nastavak na lsof)
sudo lsof -n 2>/dev/null | awk 'NR>1 {c[$1"("$2")"]++} END {for (p in c) print c[p], p}' | sort -rn | head

# Trajanje sistemskih poziva iz strace -T
strace -T -o /tmp/t.txt ls >/dev/null 2>&1
awk 'match($0, /<([0-9.]+)>$/, m) { s[$1] += m[1]; n[$1]++ }
     END { for (c in s) printf "%-16s %8.6f s  %5d poziva\n", c, s[c], n[c] }' /tmp/t.txt | sort -k2 -rn

# Zauzeće po direktorijumu iz find
find /var -xdev -type f -printf '%s %h\n' 2>/dev/null \
| awk '{b[$2] += $1} END {for (d in b) printf "%12.1f MB  %s\n", b[d]/1048576, d}' | sort -rn | head
```

Napomena: treći primer koristi `match()` sa trećim argumentom, što je **gawk** proširenje.
Prenosiva varijanta koristi `match()` uz `RSTART`/`RLENGTH` i `substr()`.

### 11.9 Praćenje uživo

```bash
journalctl -f -o cat | awk '/ERROR|FATAL/ {print; fflush()}'
```

`fflush()` je obavezan. Kada `awk` piše u cev ili fajl, izlaz se baferiše u blokovima od
nekoliko kilobajta, pa se u režimu praćenja poruke pojavljuju sa velikim zakašnjenjem
ili nikako. Isto važi i za komandu **pre** `awk`-a — ako i ona baferiše, pomaže `stdbuf`:

```bash
stdbuf -oL tcpdump -l -nn port 80 | awk '{print $1, $3, $5; fflush()}'
```

Brojanje događaja u prozorima od 10 sekundi:

```bash
tail -F app.log | gawk '{
    b = int(systime() / 10)
    if (b != prev && prev) { printf "%s  %d dogadjaja\n", strftime("%H:%M:%S"), n; n = 0 }
    prev = b; n++
    fflush()
}'
```

### 11.10 CSV sa zarezima unutar polja

`-F,` **ne radi ispravno** na pravom CSV-u:

```
id,naziv,cena
1,"Kabl, 2m",450
```

`-F,` bi dao četiri polja umesto tri. Rešenja:

```bash
gawk --csv '{print $2}' proizvodi.csv                     # gawk 5.3+
gawk 'BEGIN {FPAT = "([^,]*)|(\"[^\"]*\")"} {print $2}' proizvodi.csv   # stariji gawk
```

Ako nemate gawk, koristite `csvkit`, `mlr` (Miller) ili Python. Ručno parsiranje CSV-a
regularnim izrazima u `awk` je izvor tihih grešaka.

---

## 12. Programi u fajlu

Za bilo šta duže od jednog reda, izdvojite program u fajl:

```awk
#!/usr/bin/awk -f
# disk-report.awk — izveštaj o popunjenosti particija

BEGIN {
    prag = (prag == "") ? 80 : prag
    printf "%-24s %10s %6s  %s\n", "UREĐAJ", "VELIČINA", "USED", "TAČKA"
    printf "%s\n", "-------------------------------------------------------"
}

NR == 1 { next }

{
    used = int($5)
    if (used < prag) next
    printf "%-24s %9.1fG %5d%%  %s%s\n", $1, $2/1048576, used, $6,
           (used >= 90 ? "   <<< KRITIČNO" : "")
    broj++
}

END {
    printf "\nParticija iznad %d%%: %d\n", prag, broj + 0
    exit (broj > 0)
}
```

```bash
chmod +x disk-report.awk
df -P | ./disk-report.awk -v prag=70
```

```
UREĐAJ                     VELIČINA   USED  TAČKA
-------------------------------------------------------
/dev/sda2                     98.4G    82%  /
/dev/sdb1                    491.2G    94%  /var   <<< KRITIČNO

Particija iznad 70%: 2
```

`broj + 0` u `END` bloku pretvara neinicijalizovanu promenljivu u nulu — bez toga bi se
ispisao prazan string kada nema pogodaka.

---

## 13. Česte greške

1. **Dvostruki navodnici oko programa** — shell tumači `$1` kao svoj argument. Program uvek pod **jednostrukim** navodnicima.
2. **Umetanje shell promenljive u program** — koristite `-v`.
3. **Očekivanje da `OFS` deluje na `print $0`** — potreban je `$1=$1`.
4. **`-F'|'` bez zaštite** — uspravna crta je regex operator; napišite `-F'[|]'`.
5. **Poređenje sa stringom umesto sa brojem** — `$1 == "7"` je netačno za `007`.
6. **`if (a[k])` za proveru postojanja** — kreira ključ; koristite `if (k in a)`.
7. **Oslanjanje na redosled u `for (k in a)`** — redosled je nedefinisan; sortirajte spolja.
8. **`strftime` / `gensub` na Ubuntu** — mawk ih nema; instalirajte gawk.
9. **Bez `fflush()` u režimu praćenja** — izlaz kasni zbog baferisanja.
10. **Otvaranje mnogo izlaznih fajlova bez `close()`** — iscrpljuju se deskriptori.
11. **`df` bez `-P`** — prelomljeni redovi pomeraju kolone.
12. **Parsiranje CSV-a sa `-F,`** — puca na zarezima unutar navodnika.
13. **`substr` sa indeksom 0** — indeksi u `awk` počinju od **1**.
14. **`getline` bez provere povratne vrednosti** — beskonačna petlja kod greške.

---

## 14. Podsetnik (cheat sheet)

```bash
awk '{print $1, $NF}' f                       # prvo i poslednje polje
awk -F: '{print $1}' /etc/passwd               # drugi separator
awk 'NF' f                                     # izbaci prazne redove
awk '!seen[$0]++' f                            # duplikati, uz očuvanje redosleda
awk 'END {print NR}' f                         # broj redova
awk '{s += $3} END {print s}' f                # zbir kolone
awk '{s += $3} END {print s/NR}' f             # prosek
awk '$3 > 100' f                               # filtriranje po vrednosti
awk '$1 ~ /^10\./ && $9 == 404' f              # više uslova
awk 'NR >= 10 && NR <= 20' f                   # opseg redova
awk '/POCETAK/,/KRAJ/' f                       # opseg po šablonu
awk '{c[$1]++} END {for (k in c) print c[k], k}' f | sort -rn    # grupisanje
awk '{b[$1] += $2} END {for (k in b) print b[k], k}' f | sort -rn # zbir po ključu
awk 'NR==FNR {m[$1]=$2; next} {print $0, m[$1]}' mapa.txt podaci.txt  # spajanje
awk '{print > ("deo-" $1 ".txt")}' f           # deljenje na fajlove
awk '{$1=$1; print}' OFS=, f                   # promena separatora
awk 'BEGIN {while (("uptime"|getline l)>0) print l}'   # čitanje iz komande
awk -v p="$VREDNOST" '$3 > p' f                # promenljiva iz shell-a
awk '{printf "%-20s %8.2f\n", $1, $2}' f       # formatiran ispis
awk '/ERROR/ {print; fflush()}'                # praćenje uživo
awk 'NR==1000000 {print; exit}' veliki.log     # brz skok na red
```

---

## 15. Povezani alati

| Alat | Kada ga koristiti umesto `awk` |
|---|---|
| `grep` / `rg` | samo traženje redova po šablonu — znatno brže |
| `cut` | izdvajanje kolone bez ikakve logike |
| `sed` | zamena teksta u toku, bez rada sa poljima |
| `sort` / `uniq -c` | sortiranje i brojanje; često se kombinuju sa `awk` |
| `join` | spajanje sortiranih fajlova po ključu |
| `datamash` | statistike po grupama bez pisanja programa |
| `mlr` (Miller) | ispravan rad sa CSV, TSV, JSON i NDJSON |
| `jq` | JSON — `awk` za JSON nije prikladan |
| `perl -lane` | kad zatrebaju napredniji regularni izrazi i moduli |
| Python | kada program pređe 30-40 redova ili traži strukture podataka |

Praktično pravilo: ako `awk` program ne staje u jedan ekran ili zahteva ugnježdene strukture,
prešli ste granicu i vreme je za Python.
