# Dell PowerProtect Cyber Recovery 20.3
## Uputstvo za instalaciju i integraciju

Dokumentacija za System Engineer-e i sistem administratore koji implementiraju Cyber Recovery vault i integrišu ga sa NetWorker-om i PowerProtect Data Manager-om.

**Jezik:** srpski | **Format:** Markdown | **Bez slika** — procedure kroz tekst, CLI blokove i tabele

---

## Struktura

| Fajl | Poglavlja | Sadržaj |
|---|---|---|
| `01-uvod-i-planiranje.md` | 1–3 | Pregled rešenja, arhitektura, operacije, uloge, planiranje, sizing, redosled radova |
| `02-preduslovi.md` | 4–9 | Management host, DD sistemi, mreža, backup aplikacije, Podman, priprema vault DD-a |
| `03a-instalacija-rhel.md` | 10–15 | Softverska instalacija na RHEL/SLES **+ zajednička poglavlja 14 i 15** |
| `03b-instalacija-ova.md` | 10b–13b | Deployment virtuelnog appliance-a na VMware |
| `04-integracija-networkera.md` | 16–24 | Priprema produkcije i vault-a, CR konfiguracija, UID-ovi, recovery, `nsrdr` referenca |
| `05-integracija-ppdm.md` | 25–31 | Server DR, PPDM politika, linked recovery, post-recovery koraci |
| `06-validacija-predaja-odrzavanje.md` | 32–38 | Acceptance testiranje, operativne procedure, nadogradnja, migracija, DR CR servera |
| `07a-networker-linux-anatomija.md` | 39–47 | `/nsr` struktura, procesi, portovi, logovi, provera zdravlja |
| `07b-networker-linux-storage-katalog.md` | 48–51 | Uređaji, volumeni, media baza, indeksi, `scanner` |
| `07c-networker-linux-recovery.md` | 52–59 | Od nove instance do oporavka, korak po korak |
| `07d-networker-linux-troubleshooting.md` | 60–63 | Debug nivoi, dijagnostika po simptomu, brza referenca |
| `07-NetWorker-Linux.md` | — | Radni dokument: plan i inventar komandi za Deo VII |

---

## Redosled čitanja

**Implementacija od nule:** `01` → `02` → `03a` ili `03b` → `04` i/ili `05` → `06`

**Oporavak NetWorker-a na terenu:** `07c`, uz `07a` i `07b` za kontekst

**Dijagnostika:** `07d`, poglavlje 61 (po simptomu) i 63 (brza referenca za štampu)

> Poglavlja **14** (prva prijava) i **15** (osnovna konfiguracija u CR UI) su zajednička za oba načina instalacije i nalaze se u `03a`. Ako radite OVA deployment, posle `03b` nastavite tamo.

---

## Izvori

| Dokument | Verzija |
|---|---|
| Cyber Recovery Installation and Upgrade Guide | 20.3 |
| Cyber Recovery Product Guide | 20.3 |
| NetWorker Server Disaster Recovery and Availability Best Practices Guide | 19.13 |
| NetWorker Administration Guide | 19.13 |
| NetWorker Command Reference Guide | 19.11 |
| NetWorker and VMware Integration Guide | 19.13 |
| PowerProtect Data Manager Administrator Guide | 19.22 |
| PowerProtect Data Manager Deployment Guide | 19.22 |

U Delu VII izvor svake tvrdnje je označen: **[CMD]**, **[ADM]**, **[DR]**, **[CR]**, **[PPDM]**. Oznaka **[?]** znači da tvrdnju treba potvrditi.

---

## Nedostajuća dokumentacija

Potrebna za dopunu — otvorene stavke su navedene u dodacima B pojedinih delova.

- **E-Lab Navigator** izvod i **KB 000205512** — matrica verzija (Dodatak H nije izrađen)
- **KB 198370** (root squash), **KB 205800** (port 3009)
- **DD OS Administration Guide** — replication contexts, Retention Lock, **ifGroups**
- **Cyber Recovery Security Configuration Guide** — sertifikati, FIPS, audit logging
- **NetWorker Security Configuration Guide** — puna tabela portova za firewall
- **PPDM 20.3** Administrator i Deployment Guide — ako se ide na tu verziju

---

## Otvorena pitanja

| Tema | Gde je dokumentovano |
|---|---|
| vProxy u vault-u — CR dokumentacija ćuti | `04`, Dodatak B |
| Search Engine / reporting engine — CR i PPDM se ne poklapaju | `05`, Dodatak B |
| PPDM 20.3 grana — nije potkrepljena PPDM dokumentacijom | `05`, Dodatak B |
| SELinux i firewalld politika za NetWorker | `07a`, poglavlje 46 |

---

## Pre svake implementacije

- [ ] Popunjen radni list iz `01`, Dodatak
- [ ] Popunjen radni list preduslova iz `02`, Dodatak
- [ ] Provereno u E-Lab Navigator da su verzije podržane
- [ ] Dogovoreno gde se čuva **lockbox passphrase** (ne može se povratiti)

---

*Verzija dokumentacije: 0.1*
