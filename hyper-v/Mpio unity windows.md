# Konfiguracija MPIO za Dell EMC Unity na Windows Server (FC)

Kada se LUN-ovi sa Unity array-a prezentuju Windows serveru preko Fibre Channel-a, svaki LUN se u sistemu pojavi **više puta** — jednom po putanji. To nije greška u mapiranju, nego znak da MPIO još nije konfigurisan.

> Diskove ne inicijalizovati i ne formatirati dok MPIO nije podešen i proveren.

## Preduslovi

- Windows Server (2019 ili noviji)
- LUN-ovi mapirani na host preko FC-a
- Administratorska konzola (PowerShell)

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

### Napomene

`Enable-MSDSMAutomaticClaim -BusType FC` **ne radi** — taj parametar prihvata samo `SAS` i `iSCSI`. FC uređaji se preuzimaju isključivo preko liste podržanog hardvera.

MPIO poredi hardverski ID kao fiksni string od 24 znaka: 8 za vendora, 16 za proizvod, dopunjeno razmacima. Ako se posle restarta diskovi ne preuzmu, unos verovatno nije ispravno dopunjen. Proveri šta sistem stvarno vidi na magistrali:

```powershell
Get-MPIOAvailableHW
```

Sigurnija varijanta je dodavanje kroz GUI, jer on čita ID direktno sa uređaja:

```powershell
Remove-MSDSMSupportedHW -VendorId "DGC" -ProductId "VRAID"
mpiocpl
```

U tabu *Discover Multi-Paths*, pod **Others**, izabrati Unity uređaj → **Add** → restart.

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

Ako piše `No MPIO disks are present.` — server nije restartovan ili registracija uređaja nije uspela (vidi napomene iznad).

### 3.2 Da li su duplikati nestali

```powershell
Get-Disk
```

Svaki LUN se sada vidi tačno jednom. Diskovi su u stanju `Offline` / `RAW` dok se ne inicijalizuju — to je očekivano.

### 3.3 Stanje putanja

```powershell
mpclaim -s -d 1
```

Broj iza `-d` je MPIO disk broj iz koraka 3.1.

Unity je ALUA array, pa ispravan izlaz izgleda ovako: jedna putanja je `Active/Optimized` (ka SP-u koji je vlasnik LUN-a), ostale su `Active/Unoptimized`.

```
MPIO Disk1: 02 Paths, Round Robin with Subset, Implicit and Explicit
    Controlling DSM: Microsoft DSM
    ...
    Path ID          State              SCSI Address      Weight
    ---------------------------------------------------------------------------
    xxxxxxxxxxxxxxxx Active/Unoptimized 003|000|000|001   0
      TPG_State : Active/Unoptimized, TPG_Id: 2
    xxxxxxxxxxxxxxxx Active/Optimized   002|000|000|001   0
    * TPG_State : Active/Optimized  , TPG_Id: 1
```

Broj putanja odgovara broju FC veza ka array-u. Dva porta na HBA, ukrštena na oba SP-a, daju četiri putanje po LUN-u.

## 4. Politika balansiranja

MSDSM sam prepozna ALUA array i postavi **Round Robin with Subset**, pa u većini slučajeva nema šta da se menja. Ako treba forsirati:

```powershell
mpclaim -l -d 1 3    # 3 = Round Robin with Subset
```

`Set-MSDSMGlobalDefaultLoadBalancePolicy` prihvata samo `FOO`, `RR`, `LQD` i `None` — RRWS se postavlja po disku, kao gore.

## 5. Ostali čvorovi

Ceo postupak ponoviti na svakom hostu kojem su LUN-ovi prezentovani, pre bilo kakvog rada sa diskovima.
