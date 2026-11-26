# Dodatak 8a: Fencing u produkciji

> **Cilj:** Kada završiš ovaj tekst, razumećeš kako Proxmox VE radi fencing, koja je razlika između softverskog (`softdog`) i hardverskog watchdog-a i kako da podesiš **hardverski watchdog preko IPMI-ja** (iDRAC, iLO, XClarity...). Naučićeš i kako se fencing **testira** i koje greške u produkciji dovode do neželjenih restarta čvorova.
>
> Ovaj tekst je nastavak [Poglavlja 8: Klaster i visoka dostupnost](08-klaster-i-ha.md). Pročitaj prvo odeljke o kvorumu i HA.

---

## 8a.1 Zašto je fencing neophodan

HA sme da pokrene gosta na drugom čvoru **samo ako je sigurno** da gost više ne radi na starom čvoru. U suprotnom bi ista VM radila **dva puta istovremeno**, nad istim diskom na deljenom storage-u (Ceph, SAN, NFS), pa bi fajl sistem u VM-u bio **uništen**.

Problem je što ostali čvorovi ne mogu da razlikuju dve situacije:

| Situacija | Šta stvarno važi | Šta ostali čvorovi vide |
|---|---|---|
| Čvor je **ugašen** (nestanak struje, kvar) | VM ne radi | Čvor se ne javlja |
| Čvor je **izgubio mrežu** za corosync | VM **i dalje radi** i piše na storage | Čvor se ne javlja |

**Fencing** ("ograđivanje") je mehanizam koji **garantuje** da čvor koji se ne javlja zaista više ne radi, pre nego što HA pokrene njegove goste negde drugde.

---

## 8a.2 Kako Proxmox radi fencing: self-fencing sa watchdog-om

Mnogi HA sistemi (npr. Pacemaker) koriste **spoljni fencing** (STONITH): preživeli čvorovi preko IPMI-ja ili pametne utičnice **sami ugase** problematični čvor.

Proxmox VE koristi drugačiji pristup, **self-fencing**: čvor koji izgubi kvorum **sam sebe restartuje** pomoću **watchdog-a**.

### Šta je watchdog?

Watchdog je **tajmer** koji restartuje računar ako ga niko ne "resetuje" u zadatom vremenu. Softver na čvoru ga stalno resetuje dok je sve u redu. Ako softver to prestane da radi (namerno ili zato što je sistem zaglavljen), watchdog istekne i **restartuje mašinu**.

### Komponente

| Komponenta | Uloga |
|---|---|
| **watchdog-mux** | Servis koji otvara watchdog uređaj (`/dev/watchdog`) i prosleđuje ga HA servisima |
| **pve-ha-lrm** | Lokalni HA menadžer. Dok čvor ima kvorum i HA lock, redovno resetuje watchdog |
| **pve-ha-crm** | HA menadžer klastera. Odlučuje kada i gde se gosti oporavljaju |
| **watchdog** | `softdog` (softverski, podrazumevano) ili hardverski (IPMI, iTCO...) |

### Tok događaja

```
t = 0 s      pve1 gubi corosync vezu (kvorum)
             ├─ pve1: LRM više ne može da obnovi svoj lock → prestaje da resetuje watchdog
             └─ pve2, pve3 (imaju kvorum): CRM primećuje da je pve1 nestao

t ≈ 60 s     watchdog na pve1 ističe → pve1 se RESTARTUJE (fence)

t ≈ 120 s    lock čvora pve1 ističe na strani klastera
             → CRM sada SIGURNO zna da je pve1 fencovan
             → HA gosti sa pve1 se pokreću na pve2/pve3

t ≈ 2–3 min  gosti rade na novim čvorovima
```

Ključno: preživeli čvorovi **čekaju** dovoljno dugo da watchdog na izgubljenom čvoru sigurno istekne, pa tek onda pokreću njegove goste. Zato oporavak traje minute, a ne sekunde.

### Kada je watchdog aktivan?

Watchdog je "naoružan" **samo na čvorovima koji imaju aktivne HA resurse**. Ako na čvoru nema HA gostiju, LRM posle nekog vremena prelazi u stanje `idle` i zatvara watchdog. Takav čvor se **ne restartuje** pri gubitku kvoruma.

Stanje proveravaš ovako:

```bash
ha-manager status
```

```
quorum OK
master pve2 (active, Thu Nov 26 12:10:03 2026)
lrm pve1 (active, Thu Nov 26 12:10:05 2026)
lrm pve2 (active, Thu Nov 26 12:10:04 2026)
lrm pve3 (idle, Thu Nov 26 12:10:06 2026)
service vm:100 (pve1, started)
```

Ovde `pve1` i `pve2` imaju `lrm ... active` (watchdog je naoružan), a `pve3` je `idle`.

---

## 8a.3 Softdog i hardverski watchdog

### Softdog (podrazumevano)

`softdog` je **softverski** watchdog u Linux kernelu. Radi na svakom hardveru, bez podešavanja.

**Ograničenje:** pošto radi **unutar kernela**, ako se kernel potpuno zaglavi (npr. *hard lockup* zbog drajvera ili hardvera), softdog možda **neće uspeti** da restartuje mašinu. Takav čvor ostaje "visiti". U većini slučajeva je i tada bezbedno, jer zaglavljen kernel ne izvršava ni VM-ove, ali to **nije garantovano** (npr. kada je zaglavljen samo deo sistema).

### Hardverski watchdog

Hardverski watchdog je **poseban čip** na matičnoj ploči ili u kontroleru za daljinsko upravljanje (BMC: iDRAC, iLO, XClarity, IPMI). Radi **nezavisno od operativnog sistema**: ako ga OS ne resetuje, BMC fizički restartuje ili isključi server, šta god da se dešava sa kernelom.

| | **softdog** | **Hardverski (IPMI / iTCO / hpwdt)** |
|---|---|---|
| Podešavanje | Nije potrebno | Potrebno, zavisi od hardvera |
| Radi pri potpuno zaglavljenom kernelu | **Ne mora** | **Da** |
| Nezavisan od OS-a | Ne | Da |
| Rizik od pogrešnog podešavanja | Nema | Postoji (konflikt sa BIOS/BMC podešavanjima) |
| Preporuka | Laboratorija, manja okruženja | **Produkcija na serverskom hardveru** |

> **Preporuka:** Na serverskom hardveru sa BMC-om podesi **hardverski watchdog**, **ali** tek pošto ga testiraš na jednom čvoru (odeljak 8a.6). Pogrešno podešen hardverski watchdog je gori od ispravnog softdog-a.

### Najčešći hardverski watchdog moduli

| Modul | Hardver |
|---|---|
| `ipmi_watchdog` | Većina servera sa IPMI BMC-om (Dell iDRAC, Lenovo XClarity, Supermicro, Fujitsu iRMC...) |
| `hpwdt` | HPE ProLiant (iLO) |
| `iTCO_wdt` | Intel čipsetovi (TCO watchdog na matičnoj ploči) |
| `sp5100_tco` | AMD čipsetovi |

---

## 8a.4 Podešavanje hardverskog watchdog-a (IPMI)

> **Pre početka:** Radi ovo **čvor po čvor**, u terminu za održavanje. Prvo isprazni čvor (HA režim održavanja, Poglavlje 8), jer je za promenu watchdog modula potreban **restart**.

### Korak 1: Proveri šta trenutno koristiš

```bash
# Koji watchdog moduli su učitani
lsmod | grep -E 'softdog|ipmi_watchdog|hpwdt|iTCO_wdt|sp5100_tco'

# Informacije o watchdog uređaju (alat iz paketa util-linux)
wdctl

# Status servisa
systemctl status watchdog-mux
```

### Korak 2: Proveri da li BMC ima watchdog

```bash
apt install ipmitool
modprobe ipmi_devintf
modprobe ipmi_si
ipmitool mc watchdog get
```

Ako dobiješ informacije o watchdog tajmeru, BMC ga podržava.

### Korak 3: Isključi konfliktne funkcije u BIOS-u ili BMC-u

Neki proizvođači imaju **sopstveni** "OS watchdog" ili "automatic system recovery". Ako ostane uključen pored Proxmox watchdog-a, može da restartuje server kada ne treba. Primeri (nazivi zavise od generacije):

- **Dell iDRAC**: *Automated System Recovery Agent* / *OS Watchdog Timer* u BIOS-u,
- **HPE iLO**: *Automatic Server Recovery (ASR)*,
- **Supermicro**: *Watchdog Function* u BIOS-u.

> Pravilo: **watchdog-om treba da upravlja samo jedan softver**, a to je Proxmox (`watchdog-mux`). Proveri dokumentaciju proizvođača za svoj model.

### Korak 4: Podesi Proxmox da koristi hardverski modul

Proxmox bira watchdog modul preko fajla `/etc/default/pve-ha-manager`:

```bash
nano /etc/default/pve-ha-manager
```

```bash
# Izaberi hardverski watchdog modul umesto softdog-a
WATCHDOG_MODULE=ipmi_watchdog
```

### Korak 5: Opcije za `ipmi_watchdog`

Napravi fajl sa opcijama modula:

```bash
nano /etc/modprobe.d/ipmi_watchdog.conf
```

```
options ipmi_watchdog action=power_cycle panic_wdt_timeout=10
```

- `action=power_cycle`: kada watchdog istekne, BMC **isključi i ponovo uključi** server (najpouzdanije, jer je gubitak kvoruma često posledica problema sa hardverom),
- `panic_wdt_timeout=10`: posle kernel panike server se restartuje posle 10 sekundi.

Ažuriraj initramfs, da bi se opcije primenile i pri pokretanju:

```bash
update-initramfs -u -k all
```

### Korak 6: Restart i provera

```bash
reboot
```

Posle restarta:

```bash
lsmod | grep -E 'softdog|ipmi_watchdog'   # treba da vidiš ipmi_watchdog, a ne softdog
wdctl                                      # identitet uređaja treba da pokazuje IPMI
systemctl status watchdog-mux
journalctl -b -u watchdog-mux
ipmitool mc watchdog get                   # da li tajmer radi (kada je HA aktivan)
```

> **Napomena:** Proxmox kernel može da drži neke hardverske watchdog module na *blacklist*-i, da se ne bi učitali sami i pravili konflikt. Podešavanje `WATCHDOG_MODULE` u `/etc/default/pve-ha-manager` je ispravan način da Proxmox učita izabrani modul. Ne briši blacklist ručno.

### Korak 7: Ponovi na ostalim čvorovima

Kada je prvi čvor testiran (odeljak 8a.6), ponovi postupak na ostalim čvorovima, **jedan po jedan**. Svi čvorovi u klasteru treba da imaju **isto** podešavanje.

---

## 8a.5 Mreža i kvorum: najčešći uzrok neželjenog fencing-a

U praksi je **većina** fencing restarta **neželjena**: nije otkazao server, nego je corosync izgubio vezu. Zato je mreža za corosync deo podešavanja fencing-a.

### Opasan scenario: ceo klaster gubi kvorum

Ako svi čvorovi koriste **jedan** svič za corosync i taj svič otkaže:

```
svič za corosync ✗
    → nijedan čvor ne vidi ostale
    → NIJEDAN čvor nema kvorum
    → SVI čvorovi sa aktivnim HA resursima se restartuju (fence)
    → svi HA gosti staju
```

Kvar jednog sviča tako postaje **prekid rada celog klastera**. HA je u ovom slučaju napravio veću štetu nego korist.

### Kako to sprečiti

| Mera | Zašto |
|---|---|
| **Dva corosync linka** na **dva različita sviča** (link 0, link 1) | Otkaz jednog sviča ne ruši kvorum |
| Corosync na **posebnim karticama**, bez storage i backup saobraćaja | Zagušenje ne izaziva gubitak veze |
| **Ne koristi bond** za corosync, nego više corosync linkova | Corosync (knet) sam bira ispravan link i brže reaguje |
| Nadzor corosync linkova (`corosync-cfgtool -s`) | Otkaz jednog linka vidiš **pre** nego što otkaže i drugi |
| Mreža za upravljanje kao **dodatni** link (npr. link 2), sa nižim prioritetom | Treća putanja za slučaj nužde |

Provera linkova:

```bash
corosync-cfgtool -s
```

```
LINK ID 0 udp
        addr    = 10.0.1.11
        status:
                nodeid:   2:     connected
                nodeid:   3:     connected
LINK ID 1 udp
        addr    = 10.0.2.11
        status:
                nodeid:   2:     connected
                nodeid:   3:     connected
```

Dodavanje linka postojećem klasteru radi se izmenom `/etc/pve/corosync.conf`. Pre izmene obavezno pročitaj odeljak dokumentacije o izmeni corosync konfiguracije: menja se kopija, povećava se `config_version`, pa se fajl zamenjuje, jer greška može da sruši ceo klaster.

---

## 8a.6 Testiranje fencing-a

> **Fencing koji nisi testirao ne znaš da li radi.** Testiraj ga **pre puštanja u produkciju** i posle svake veće promene (novi hardver, novi watchdog modul, promene u mreži).

### Priprema

- test radi u **terminu za održavanje**, uz obavešten tim,
- napravi **test VM** na deljenom storage-u i dodaj je u HA (`ha-manager add vm:999 --state started`),
- obezbedi **pristup konzoli** čvora preko BMC-a, da vidiš šta se dešava,
- na drugom čvoru prati `watch ha-manager status` i `journalctl -f -u pve-ha-crm`.

### Test 1: Gubitak corosync mreže (najvažniji)

Na čvoru sa test VM-om **isključi sve corosync linkove**: fizički izvuci kablove ili isključi portove na sviču.

Očekivano:

1. posle ~**60 s** čvor se **restartuje** (na konzoli vidiš restart ili power cycle),
2. posle ~**2–3 min** test VM radi na drugom čvoru,
3. kada vratiš kablove, čvor se pridružuje klasteru, a VM ostaje na novom čvoru (ili se vraća, prema HA pravilima).

### Test 2: Zaglavljen kernel (test hardverskog watchdog-a)

Ovaj test proverava da li watchdog radi i kada OS ne radi:

```bash
# PAŽNJA: odmah ruši kernel na ovom čvoru!
echo c > /proc/sysrq-trigger
```

Očekivano: server se restartuje posle `panic_wdt_timeout` sekundi ili posle isteka watchdog-a, a VM se oporavlja na drugom čvoru.

### Test 3: Nestanak struje

Isključi čvor preko BMC-a (**Power Off**, ne *Graceful Shutdown*). Proveri da se VM oporavlja na drugom čvoru.

### Test 4: Otkaz jednog corosync linka

Izvuci kabl **samo jednog** linka. Očekivano: **ništa se ne dešava** sa čvorom i gostima, a `corosync-cfgtool -s` pokazuje jedan link kao nepovezan. Ako se čvor ipak restartuje, redundansa ne radi.

### Zapiši rezultate

| Test | Očekivano | Izmereno vreme | Rezultat |
|---|---|---|---|
| Gubitak corosync mreže | Fence za ~60 s, VM radi za ~2–3 min | | |
| Kernel crash | Restart, VM se oporavlja | | |
| Nestanak struje | VM se oporavlja | | |
| Otkaz jednog linka | Bez restarta | | |

---

## 8a.7 Održavanje bez neželjenog fencing-a

| Radnja | Rizik | Kako bezbedno |
|---|---|---|
| Ažuriranje i restart čvora | Mali | **HA režim održavanja** (od 9.3 u GUI-ju) |
| Radovi na corosync mreži ili svičevima | **Fencing više čvorova** | **HA disarm** za klaster (od 9.2), ili radovi na jednom linku po jednom |
| Ručno zaustavljanje `corosync` ili `pve-cluster` na čvoru sa HA gostima | Čvor gubi kvorum i fencuje se | Prvo režim održavanja, pa tek onda radovi |
| Ručno zaustavljanje `watchdog-mux` dok je HA aktivan | **Trenutni restart** | Nikad ručno, samo preko režima održavanja ili disarm-a |
| Restart svih čvorova odjednom | Gubitak kvoruma i prekid svih servisa | Čvor po čvor |

> **Važno:** Posle HA disarm-a **ne zaboravi** da ponovo uključiš HA (arm). Pre upotrebe pročitaj u dokumentaciji za 9.3 kako se tačno ponašaju gosti i watchdog dok je HA isključen.

---

## 8a.8 Česte greške u produkciji

| Greška | Posledica | Kako izbeći |
|---|---|---|
| Jedan corosync svič | Otkaz sviča restartuje **sve** HA čvorove | Dva linka, dva sviča |
| Corosync deli mrežu sa Ceph-om i backup-om | Zagušenje izaziva lažne gubitke kvoruma i restarte | Posebne kartice za corosync |
| Hardverski watchdog + uključen ASR/OS watchdog u BIOS-u | Nasumični restarti | Isključi watchdog funkcije proizvođača |
| Hardverski watchdog podešen bez testa | Fencing ne radi, ili radi kada ne treba | Testovi iz odeljka 8a.6 |
| Različita podešavanja watchdog-a na čvorovima | Nepredvidivo ponašanje | Isto podešavanje (Ansible) |
| Ručno zaustavljanje cluster servisa | Neočekivan restart | Režim održavanja |
| HA za sve goste "za svaki slučaj" | Više čvorova sa naoružanim watchdog-om, veći efekat lažnih alarma | HA samo za goste kojima stvarno treba |
| Klaster od 2 čvora bez QDevice-a sa HA | Gubitak jednog čvora = gubitak kvoruma = fencing i drugog | 3 glasa (QDevice) |
| Fencing nikad testiran | Iznenađenje pri prvom pravom kvaru | Test pre produkcije i posle promena |

---

## 8a.9 Kontrolna lista: fencing za produkciju

- [ ] Klaster ima najmanje **3 glasa** (3 čvora ili 2 + QDevice)
- [ ] **Dva corosync linka** na **različitim svičevima**, na posebnim karticama
- [ ] Nadzor corosync linkova (alarm kada jedan link padne)
- [ ] Na serverskom hardveru: **hardverski watchdog** podešen (`WATCHDOG_MODULE`) i testiran
- [ ] Watchdog funkcije proizvođača (ASR, OS Watchdog) **isključene**
- [ ] Isto podešavanje na **svim** čvorovima
- [ ] HA samo za goste kojima je potreban
- [ ] Testovi iz odeljka 8a.6 urađeni i dokumentovani, sa izmerenim vremenima
- [ ] Postupci održavanja (režim održavanja, disarm/arm) dokumentovani za tim
- [ ] Notifikacije za HA događaje (fencing) idu dežurnom (Poglavlje 10)

---

## 8a.10 Pitanja za proveru znanja

1. Zašto preživeli čvorovi ne smeju odmah da pokrenu goste čvora koji se ne javlja?
2. Koja je razlika između spoljnog fencing-a (STONITH) i self-fencing-a koji koristi Proxmox?
3. Šta se dešava sa watchdog-om na čvoru koji nema HA gostiju?
4. Kada softdog možda neće uspeti da restartuje čvor?
5. U kom fajlu biraš watchdog modul za Proxmox HA?
6. Zašto treba isključiti ASR ili OS watchdog funkcije u BIOS-u?
7. Opiši scenario u kojem kvar jednog sviča restartuje sve čvorove klastera i kako ga sprečiti.
8. Koja četiri testa fencing-a treba uraditi pre produkcije?

---

## Izvori

- [Proxmox VE Administration Guide (9.3.1): High Availability, Fencing](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)
- [Proxmox VE Administration Guide (9.3.1): Cluster Manager, Corosync Redundancy](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)
- [Proxmox VE 9.2 najava na forumu](https://forum.proxmox.com/threads/proxmox-virtual-environment-9-2-available.183741/latest)

---

**Nazad na:** [Poglavlje 8: Klaster i visoka dostupnost (HA)](08-klaster-i-ha.md)
**Sledeće poglavlje:** [Poglavlje 9: Ceph i replikacija](09-ceph-i-replikacija.md)
