# Konfiguracija MPIO za Dell EMC Unity na Windows Server (FC)

Kada se LUN-ovi sa Unity array-a prezentuju Windows serveru preko Fibre Channel-a, svaki LUN se u sistemu pojavi **više puta** — jednom po putanji. To nije greška u mapiranju, nego znak da MPIO još nije konfigurisan.

> Diskove ne inicijalizovati i ne formatirati dok MPIO nije podešen i proveren.

## Preduslovi

- Windows Server (2019 ili noviji)
- LUN-ovi mapirani na host preko FC-a, preko oba SP-a
- PowerShell sa administratorskim privilegijama

Ceo postupak se ponavlja na **svakom** hostu kojem su LUN-ovi prezentovani.

## 1. Instalacija MPIO feature-a

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName MultiPathIO -All
```

Zahteva restart.

## 2. Registracija Unity uređaja

Unity se u SCSI inquiry-ju identifikuje kao `DGC` / `VRAID` (nasleđe iz VNX linije).

```powershell
New-MSDSMSupportedHW -VendorId "DGC" -ProductId "VRAID"
```

Posle ove komande je **obavezan restart** — MSDSM preuzima diskove tek pri podizanju sistema.

> `Enable-MSDSMAutomaticClaim -BusType FC` ne radi — taj parametar prihvata samo `SAS` i `iSCSI`. FC uređaji se preuzimaju isključivo preko liste podržanog hardvera.

## 3. Verifikacija

### 3.1 Da li su diskovi preuzeti

```powershell
mpclaim -s -d
```

Očekivano — svaki LUN kao MPIO disk pod Microsoft DSM-om, sa politikom `RRWS`:

```
MPIO Disk    System Disk  LB Policy    DSM Name
-------------------------------------------------------------------------------
MPIO Disk0   Disk 1       RRWS         Microsoft DSM
MPIO Disk1   Disk 2       RRWS         Microsoft DSM
```

Ako piše `No MPIO disks are present.`, vidi poglavlje *Rešavanje problema*.

### 3.2 Da li su duplikati nestali

```powershell
Get-Disk
```

Svaki LUN se sada vidi tačno jednom, kao `DGC VRAID`. Diskovi su u stanju `Offline` / `RAW` dok se ne inicijalizuju — to je očekivano.

Svi Unity diskovi dele isti `Serial Number` u ovom izlazu — to je serijski broj array-a, ne LUN-a. Pravi identifikator LUN-a je WWN iz sledećeg koraka.

### 3.3 Stanje putanja

```powershell
mpclaim -s -d 0
```

Broj iza `-d` je **MPIO disk broj** iz koraka 3.1, ne broj diska iz `Get-Disk`. Ako upišeš broj koji ne postoji, dobićeš `Could not find specified device.`

Unity je ALUA array, pa ispravan izlaz izgleda ovako: jedna putanja je `Active/Optimized` (ka SP-u koji je vlasnik LUN-a), ostale su `Active/Unoptimized`.

```
MPIO Disk0: 02 Paths, Round Robin with Subset, Implicit and Explicit
    Controlling DSM: Microsoft DSM
    SN: 6006016...
    Supported Load Balance Policies: FOO RRWS LQD WP LB

    Path ID          State              SCSI Address      Weight
    ---------------------------------------------------------------------------
    xxxxxxxxxxxxxxxx Active/Unoptimized 003|000|000|000   0
      TPG_State : Active/Unoptimized, TPG_Id: 2
    xxxxxxxxxxxxxxxx Active/Optimized   002|000|000|000   0
    * TPG_State : Active/Optimized  , TPG_Id: 1
```

Šta proveriti:

- **Broj putanja** odgovara broju FC veza ka array-u. Dva porta na HBA, ukrštena na oba SP-a, daju četiri putanje po LUN-u; dve putanje znače po jednu vezu do svakog SP-a.
- **WWN (`SN:`)** se mora poklapati između svih hostova u klasteru — tako se potvrđuje da svi vide iste LUN-ove.
- Ako su različiti LUN-ovi optimizovani preko različitih SP-ova, opterećenje je raspoređeno na oba kontrolera.

## 4. Politika balansiranja

MSDSM sam prepozna ALUA array i postavi **Round Robin with Subset**, pa u većini slučajeva nema šta da se menja. Ako treba forsirati:

```powershell
mpclaim -l -d 0 3    # 3 = Round Robin with Subset
```

`Set-MSDSMGlobalDefaultLoadBalancePolicy` prihvata samo `FOO`, `RR`, `LQD` i `None` — RRWS se postavlja po disku, kao gore.

## Rešavanje problema

### `No MPIO disks are present.`

Najčešći uzrok je da server nije restartovan posle koraka 2.

Ako jeste restartovan, problem je verovatno u hardverskom ID-u. MPIO ga poredi kao fiksni string od 24 znaka: 8 za vendora, 16 za proizvod, dopunjeno razmacima. Proveri šta sistem stvarno vidi na magistrali i uporedi sa registrovanim unosom:

```powershell
Get-MPIOAvailableHW
Get-MSDSMSupportedHW
```

### Dodavanje uređaja kroz GUI

Alternativa koja ne zavisi od ručnog kucanja ID-a, jer čita string direktno sa uređaja:

```powershell
Remove-MSDSMSupportedHW -VendorId "DGC" -ProductId "VRAID"
mpiocpl
```

U tabu **Discover Multi-Paths**, pod sekcijom *Others*, izabrati Unity uređaj → **Add** → restart.

Uređaj se u toj listi pojavljuje samo ako nije već registrovan i ako Windows u tom trenutku vidi više putanja ka njemu. Ako je lista prazna, proveri zoniranje i mapiranje LUN-ova preko oba SP-a.

### Sadržaj taba MPIO Devices

Posle uspešne registracije, u `mpiocpl` → tab **MPIO Devices** stoje dva unosa:

```
DGC     VRAID
Vendor 8Product    16
```

Drugi red nije uređaj — to je podrazumevani primer koji Windows isporučuje i pokazuje format stringa (8 + 16 znakova). Ne dirati ga.

## Dalji koraci

Tek kada je MPIO potvrđen na svim hostovima, pristupa se inicijalizaciji diskova — i to **samo sa jednog hosta**, dok na ostalima diskovi ostaju offline.
