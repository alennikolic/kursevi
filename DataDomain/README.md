# PowerProtect Data Domain (DDOS) — Priručnik za System Engineere

Interni operativni priručnik za rad sa Dell PowerProtect Data Domain uređajima
preko CLI-ja. Namenjen je inženjerima koji održavaju DD kao backup storage,
kao replikacioni target i kao uređaj sa zaključavanjem backup fajlova
(DD Retention Lock), uključujući Cyber Recovery vault okruženja.

> **Ciljna verzija:** DD OS 8.6 · PowerProtect Cyber Recovery 20.3
> **Komande provereno prema:** DD OS 8.6 Command Reference Guide (rev. decembar 2025),
> DD OS 8.6 Administration Guide (rev. novembar 2025),
> Cyber Recovery 20.3 Command-Line Interface Reference Guide (rev. avgust 2026)
> **Poslednja izmena:** _(popuniti)_
> **Vlasnik dokumenta:** _(popuniti)_

Napomena: sve komande u ovom priručniku proverene su i prema DD OS 8.9 CRG-u —
nije nađena nijedna razlika u sintaksi između 8.6 i 8.9 za komande koje se ovde
koriste. Priručnik je upotrebljiv na obe grane.

---

## Kako koristiti ovaj priručnik

- Poglavlje **02** je dnevni health check — to je ono što se najčešće otvara.
- Poglavlja **04–07** su dubinska poglavlja po temama (kapacitet, MTree, lock, replikacija).
- Poglavlje **10** su gotovi playbook-ovi za konkretne incidente.
- Poglavlje **12** je Cyber Recovery vault — čitati **pre** bilo kakvog rada na vault DD-u.

### Konvencije

| Oznaka | Značenje |
|---|---|
| bez oznake | Read-only komanda, bezbedna u produkciji u bilo kom trenutku |
| ⚠️ | Menja konfiguraciju — radi se uz change/odobrenje |
| 🛑 | Destruktivno ili prekida servis (restart FS, break replikacije, brisanje) |
| ⛔ | Ne raditi — narušava dizajn okruženja |
| `<...>` | Vrednost koju popunjavaš |
| `[...]` | Opcioni argument |
| `{a \| b}` | Izbor jedne od navedenih vrednosti |

### Pravila kuće

1. **Uvek prvo `help <komanda>` na samom uređaju.** Sintaksa se menja između DDOS verzija.
   Ovaj priručnik je vodič; zvanični izvor je Command Reference Guide za tvoju verziju.
2. Proveri verziju pre nego što kopiraš komandu: `system show version`
3. Nikada ne pokrećeš 🛑 komande bez provere da li je uređaj replikacioni target
   aktivnog konteksta i da li je u toku backup prozor.
4. Izmene se rade `admin` ili `limited-admin` rolom; Retention Lock Compliance
   operacije zahtevaju rolu **`security`** (vidi poglavlje 06).

---

## Sadržaj

| # | Poglavlje | Šta pokriva |
|---|---|---|
| 01 | [Osnove i pristup](01-osnove-i-pristup.md) | SSH/serijski pristup, CLI navigacija, alijasi, help sistem, role i korisnici, `net filter` |
| 02 | [Dnevni health check](02-dnevni-health-check.md) | Cheat sheet: komande za "da li je sve u redu", redosled provere, značenje outputa |
| 03 | [Sistem, alerti, logovi, podrška](03-sistem-alerti-logovi.md) | `alerts`, `autosupport`, `support bundle`, `log`, hardver, `elicense`, upgrade, reboot |
| 04 | [Kapacitet, file system i cleaning](04-kapacitet-filesystem-cleaning.md) | `filesys show space`, kompresija, GC/cleaning, pragovi zauzeća, `storage`, `disk`, Cloud Tier |
| 05 | [MTree, kvote i snapshot](05-mtree-kvote-snapshot.md) | MTree upravljanje, per-MTree statistika, capacity i stream kvote, snapshot i rasporedi |
| 06 | [Retention Lock](06-retention-lock.md) | Governance vs Compliance, rola `security`, min/max retention, automatic lock, indefinite hold, izveštaji |
| 07 | [Replikacija](07-replikacija.md) | MTree/Collection/Managed File replikacija, kontekst, inicijalizacija, monitoring, throttle, break, resync |
| 08 | [Mreža i performanse](08-mreza-i-performanse.md) | Interfejsi, agregacija/failover, `net route`, `net filter`, opterećenje, `net iperf`, dijagnostika |
| 09 | [Protokoli i pristup podacima](09-protokoli-nfs-cifs-ddboost.md) | `nfs export`, CIFS/SMB, DD Boost storage unit-ovi, ifgroup, VTL |
| 10 | [Troubleshooting playbook-ovi](10-troubleshooting-playbooks.md) | 16 scenarija: FS pun, cleaning, replikacija, protokoli, disk, CR incidenti |
| 12 | [Cyber Recovery vault](12-cyber-recovery-vault.md) | CR 20.3: stanja vault-a, politike, CRCLI, kapacitet vault-a, oporavak, šta se NE dira na vault DD-u |

---

## Brzi start — 5 komandi koje pokrivaju 80% pitanja

```
system show version              # koja je DDOS verzija i model
alerts show current              # da li nešto trenutno gori
filesys show space               # koliko ima mesta
mtree list                       # koji MTree-jevi postoje i njihov status
replication status               # stanje svih replikacionih konteksta
```

---

## Zvanična dokumentacija (izvor istine)

Sve dole navedeno je na Dell Support portalu i **traži Dell Online Support login**.

- **Dell PowerProtect Data Domain Info Hub** — svi dokumenti po DDOS verzijama:
  <https://www.dell.com/support/kbdoc/en-us/000126375/powerprotect-and-data-domain-core-documents>
- **DDOS Software Versions and Download Links** — koja je verzija aktivna, LTS grane, EOSS datumi:
  <https://www.dell.com/support/kbdoc/en-us/000081247/dd-os-software-versions>
- **Port requirements for allowing access to a Data Domain system through a Firewall**:
  <https://www.dell.com/support/kbdoc/en-us/000004184/1245-port-requirements-for-allowing-access-to-data-domain-system-through-a-firewall>
- **Data Domain Compatibility Matrix (E-Lab Navigator)** — obavezno pre svakog upgrade-a:
  <https://elabnavigator.dell.com/eln/modernHomeDataProtection>
- **Hardware Documentation Guide** — FRU/CRU procedure po modelima:
  <https://www.dell.com/support/kbdoc/en-us/000130388/powerprotect-and-data-domain-hardware-documents>

### Dokumenti korišćeni za proveru komandi u ovom priručniku

| Dokument | Verzija | Revizija |
|---|---|---|
| DD OS Command Reference Guide | 8.6 | decembar 2025 |
| DD OS Administration Guide | 8.6 | novembar 2025 |
| Cyber Recovery CLI Reference Guide | 20.3 | avgust 2026 |
| DD OS Command Reference Guide (kontrolna provera) | 8.9 | avgust 2026 |

### Interni linkovi

_(prostor za linkove na internu dokumentaciju, evidenciju uređaja, change proceduru i
kontakte — popuniti)_
