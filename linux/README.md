# Instructions — Napredne Linux komande (GitHub projekat)

Ovaj dokument definiše kako se piše, imenuje i organizuje svaki fajl u repozitorijumu.
Cilj je da svih 60+ komandi izgleda kao da ih je pisala jedna osoba, istog dana.

---

## 1. Cilj projekta

Repozitorijum sa uputstvima za napredne Linux komande, gde:

- svaka komanda ima **svoj zaseban `.md` fajl**,
- svaki fajl prati **isti template** (sekcije istim redosledom),
- svaki primer ima **komandu, stvarni izlaz i objašnjenje tog izlaza**,
- čitalac posle jednog fajla ume da upotrebi komandu na svom sistemu, bez guglanja.

Ciljna publika: neko ko zna osnove terminala (`cd`, `ls`, `cat`) ali nije koristio
`awk`, `ss`, `strace`, `rsync` i slično.

Jezik: srpski (latinica), tehnički termini u originalu (`pipe`, `stdout`, `exit code`).

---

## 2. Struktura repozitorijuma

```
linux/komande/
    ├── tekst-i-podaci/        # awk, sed, cut, sort, uniq, tr, jq, xargs...
    ├── pretraga/              # find, grep, rg, fd, locate
    ├── procesi/               # ps, lsof, strace, kill, nice, nohup
    ├── sistem-i-performanse/  # vmstat, iostat, sar, journalctl, dmesg, perf
    ├── fajl-sistem/           # du, df, rsync, dd, lsblk, stat, tee
    ├── mreza/                 # ss, ip, tcpdump, dig, curl, nc, ssh, mtr
    ├── korisnici-i-dozvole/   # chmod, chown, umask, setfacl, chattr, sudoers
    ├── arhiviranje/           # tar, split, gzip/zstd, cpio
    └── automatizacija/        # cron, systemd timers, watch, parallel, tmux, make
```

**Pravila imenovanja**

- Ime fajla = tačno ime komande, mala slova: `awk.md`, `ss.md`, `systemd-timers.md`.
- Bez verzija, bez prefiksa, bez razmaka. Nikad `01-awk.md` ili `AWK.md`.
- Folderi: mala slova, crtica umesto razmaka, bez dijakritika (`mreza`, ne `mreža`).
- Slike (ako ih ima): `komande/<kategorija>/slike/<komanda>-01.png`.

---

## 3. Obavezni template svakog fajla

Svaki `.md` fajl ima ove sekcije, ovim redosledom. Sekcija koja nema sadržaj se
**briše**, ne ostavlja se prazna (izuzetak: sekcije 1–5 i 10 su uvek obavezne).

```markdown
---
komanda: awk
kategorija: tekst-i-podaci
tezina: srednje            # osnovno | srednje | napredno
paket: gawk                # paket koji se instalira
testirano-na: "Ubuntu 24.04, GNU Awk 5.2.1"
tagovi: [tekst, kolone, izvestaji]
---

# awk — obrada teksta po kolonama

## Šta radi
2–4 rečenice. Šta komanda radi i po čemu je drugačija od alternativa.

## Kada se koristi
3–5 konkretnih scenarija u bulletima ("kad treba da sabereš kolonu iz log fajla").

## Instalacija i provera verzije
Komanda za instalaciju (apt / dnf / pacman) + `awk --version`.

## Sintaksa
Osnovni oblik poziva + objašnjenje svakog dela.

## Najvažnije opcije
| Opcija | Značenje | Primer |
|---|---|---|
| `-F` | razdvajač polja | `awk -F: '{print $1}'` |

## Primeri
(pravila u sekciji 4 ovog dokumenta — najmanje 3, idealno 5)

## Kombinovanje sa drugim komandama
1–3 pipeline primera (`ps aux | awk ...`), sa objašnjenjem toka podataka.

## Česte greške i zamke
Tabela ili lista: greška → zašto se dešava → rešenje.

## Upozorenja
Samo za destruktivne komande (`dd`, `rm`, `chmod -R`, `iptables`). Vidi sekciju 6.

## Povezane komande
Linkovi ka drugim fajlovima u repou: [`sed`](../tekst-i-podaci/sed.md)

## Izvori
`man awk`, zvanična dokumentacija, link na GNU manual.
```

Template se drži u `template/KOMANDA-TEMPLATE.md` i kopira za svaku novu komandu.

---

## 4. Pravila za primere (najvažniji deo)

Svaki primer ima **četiri obavezna dela**, uvek istim redosledom:

### Primer N: kratak naslov šta se postiže

**Cilj:** jedna rečenica — šta hoćemo da dobijemo.

**Komanda:**

````markdown
```bash
awk -F: '$3 >= 1000 {print $1, $3}' /etc/passwd
```
````

**Izlaz:**

````markdown
```text
milan 1000
ana 1001
backup 1002
```
````

**Objašnjenje:** razbij komandu na delove i objasni **i izlaz**:

| Deo | Značenje |
|---|---|
| `-F:` | polja se razdvajaju dvotačkom, jer je to format `/etc/passwd` |
| `$3 >= 1000` | uslov: uzmi samo redove gde je treće polje (UID) veće ili jednako 1000 |
| `{print $1, $3}` | ispiši prvo polje (korisničko ime) i treće (UID) |

Zatim 1–3 rečenice o samom izlazu: šta znači svaka kolona, zašto ih ima toliko,
zašto se sistemski korisnici ne vide (UID < 1000).

### Tvrda pravila za primere

1. **Najmanje 3 primera po komandi**, poređana od najjednostavnijeg ka najsloženijem.
2. Izlaz mora biti **stvarno pokrenut na sistemu**, ne izmišljen. Ako je izlaz dugačak,
   skrati ga i označi to sa `[...]` u sredini.
3. Komanda i izlaz idu u **odvojene code blokove**. Nikad zajedno.
4. U bloku sa komandom **nema `$` prompta** — da čitalac može da kopira jednim potezom.
   Ako je nužno naglasiti root, koristi `sudo` u samoj komandi.
5. Blok sa komandom ima jezik `bash`, blok sa izlazom ima jezik `text`.
6. Ako izlaz ima kolone (`ps`, `df`, `ss`, `ls -l`), **obavezna je tabela koja objašnjava
   svaku kolonu**. To je suština projekta.
7. Bez pravih ličnih podataka. Koristi:
   - korisnici: `milan`, `ana`, `korisnik`
   - hostovi: `example.com`, `server01`
   - IP adrese: `192.0.2.10`, `198.51.100.5`, `203.0.113.7` (rezervisani TEST-NET opsezi)
   - putanje: `/home/korisnik/projekat`, `/var/log/app.log`
8. Ako izlaz zavisi od distribucije ili verzije, napiši to ispod primera jednom rečenicom.

---

## 5. Stil pisanja

- Direktno i kratko. Bez uvoda tipa "u ovom uputstvu ćemo naučiti".
- Obraćanje u drugom licu: "pokreni", "proveri", "dobićeš".
- Rečenice kratke. Jedna misao po rečenici.
- Imena komandi, opcija, fajlova i putanja **uvek** u inline kodu: `` `ls -l` ``, `` `/etc/fstab` ``.
- Bez emodžija u tekstu. Dozvoljeni samo u tabeli statusa u `README.md`.
- Naslovi: `#` samo jednom (ime komande), sekcije `##`, primeri `###`.
- Maksimalna dužina reda u izvoru: 100 karaktera (lakši `git diff`).
- Bez copy-paste teksta sa `man` stranica ili tuđih sajtova — sve prepričano svojim rečima.

---

## 6. Destruktivne komande

Za sve što može da uništi podatke ili zaključa sistem (`dd`, `rm -rf`, `mkfs`, `chmod -R`,
`chown -R`, `iptables`, `truncate`, `> fajl`) obavezno:

- sekcija `## Upozorenja` sa jasnim opisom šta se gubi i da li je povratno,
- primeri prvo u **bezbednoj varijanti**: `rsync --dry-run`, `find ... -print` pre `-delete`,
- u primeru se nikad ne cilja stvarni sistemski uređaj — koristi `/dev/sdX` kao placeholder
  i napiši eksplicitno da to nije komanda za kopiranje bez izmene,
- blockquote na vrhu sekcije:

```markdown
> **Pažnja:** ova komanda briše podatke bez potvrde i bez mogućnosti povratka.
> Proveri ciljni uređaj sa `lsblk` pre pokretanja.
```

---

## 7. README.md

Sadrži:

1. Kratak opis projekta (3–4 rečenice) i za koga je.
2. Kako se koristi repo (pročitaj fajl komande koja te zanima).
3. **Tabelu svih komandi**, grupisanu po kategorijama:

   | Komanda | Kategorija | Težina | Opis | Status |
   |---|---|---|---|---|
   | [`awk`](komande/tekst-i-podaci/awk.md) | tekst | srednje | obrada po kolonama | ✅ |
   | [`strace`](komande/procesi/strace.md) | procesi | napredno | praćenje syscall-ova | 🚧 |

   Status: ✅ gotovo · 🚧 u izradi · 📝 planirano

4. Link na `CONTRIBUTING.md` i licencu.

README tabela se ažurira **u istom commitu** u kom se dodaje novi fajl komande.

---

## 8. Git konvencije

- Grana po komandi: `komanda/awk`, `komanda/tcpdump`.
- Commit poruke (Conventional Commits):
  - `docs(awk): dodato uputstvo sa 5 primera`
  - `docs(ss): ispravljeno objašnjenje kolone State`
  - `chore(readme): azurirana tabela komandi`
  - `fix(dd): dodato upozorenje o brisanju diska`
- Jedan commit = jedna logička promena. Ne mešati novu komandu i izmenu README stila.
- Pull request opis: koja komanda, koliko primera, na čemu je testirano.

---

## 9. Checklist pre merge-a (definicija "gotovo")

- [ ] Fajl je na ispravnoj putanji i ispravno imenovan
- [ ] YAML front matter popunjen (uključujući `testirano-na`)
- [ ] Sve obavezne sekcije prisutne, istim redosledom
- [ ] Najmanje 3 primera, svaki sa ciljem, komandom, izlazom i objašnjenjem
- [ ] Svaki izlaz sa kolonama ima tabelu objašnjenja kolona
- [ ] Svi izlazi stvarno pokrenuti, bez ličnih podataka
- [ ] Destruktivne komande imaju `## Upozorenja`
- [ ] Interni linkovi rade (relativne putanje)
- [ ] `README.md` tabela dopunjena i status promenjen u ✅
- [ ] Markdown prolazi lint (bez tab karaktera, bez trailing whitespace)

---

## 10. Redosled rada (predlog)

Prvo napravi kostur: `README.md`, `CONTRIBUTING.md`, `template/KOMANDA-TEMPLATE.md`,
prazne foldere sa `.gitkeep`. Zatim napiši **jedan fajl do kraja** (predlog: `awk.md` ili
`find.md`) i tretiraj ga kao referentni uzorak — sve ostale komande se pišu poređenjem
sa njim. Tek onda ubrzavaj tempo.

Prioritet po korisnosti: `find` → `awk` → `sed` → `xargs` → `grep` → `rsync` → `ss` →
`lsof` → `journalctl` → `tar` → `du`/`df` → `systemd timers` → `tcpdump` → `strace`.
