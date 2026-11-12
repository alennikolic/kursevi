# Konfiguracija mreže na Hyper-V hostu (Windows Server 2025 + SET)

Uputstvo za podešavanje mreže na Hyper-V hostovima sa 4x 10GbE porta, u konfiguraciji sa dva odvojena SET tima (Switch Embedded Teaming).

---

## Sadržaj

- [Arhitektura](#arhitektura)
- [Preduslovi](#preduslovi)
- [Korak 1 — Instalacija rola](#korak-1--instalacija-rola)
- [Korak 2 — Identifikacija i preimenovanje NIC-eva](#korak-2--identifikacija-i-preimenovanje-nic-eva)
- [Korak 3 — Osnovne postavke na fizičkim portovima](#korak-3--osnovne-postavke-na-fizičkim-portovima)
- [Korak 4 — Kreiranje vSwitch-eva](#korak-4--kreiranje-vswitch-eva)
- [Korak 5 — Host vNIC-ovi i VLAN-ovi](#korak-5--host-vnic-ovi-i-vlan-ovi)
- [Korak 6 — IP adresacija](#korak-6--ip-adresacija)
- [Korak 7 — QoS](#korak-7--qos)
- [Korak 8 — Live Migration](#korak-8--live-migration)
- [Korak 9 — Hosts fajl](#korak-9--hosts-fajl-privremeno)
- [Korak 10 — Provera](#korak-10--provera)
- [VLAN-ovi za virtuelne mašine](#vlan-ovi-za-virtuelne-mašine)
- [Konfiguracija na switchu](#konfiguracija-na-switchu)
- [Troubleshooting](#troubleshooting)
- [Izveštaj o konfiguraciji](#izveštaj-o-konfiguraciji)

---

## Arhitektura

Četiri 10GbE porta podeljena su u dva SET tima:

```
NIC1 + NIC2  ->  vSwitch-Host  ->  vNIC Mgmt  (VLAN 110)
                                   vNIC LM    (VLAN 111, nosi i CSV)

NIC3 + NIC4  ->  vSwitch-VM    ->  virtuelne mašine (VLAN po VM adapteru)
```

### Logika podele

| Nivo | Mehanizam | Svrha |
|---|---|---|
| Logička izolacija | VLAN + zaseban subnet | Failover Clustering prepoznaje mreže po subnetu; bez toga ne može da dodeli role ni da bira mrežu za Live Migration |
| Fizička izolacija | dva odvojena SET tima | VM saobraćaj ne guši host i cluster saobraćaj |

### Adresni plan (primer)

| Mreža | VLAN | Subnet | HV01 | HV02 | Gateway |
|---|---|---|---|---|---|
| Management | 110 | 10.0.10.0/24 | 10.0.10.11 | 10.0.10.12 | 10.0.10.1 |
| Live Migration + CSV | 111 | 10.0.11.0/24 | 10.0.11.11 | 10.0.11.12 | — |
| VM produkcija | 120 | po potrebi | — | — | — |
| VM DMZ | 130 | po potrebi | — | — | — |

> **Napomena:** Live Migration mreža namerno **nema** default gateway. Jedini interfejs sa gatewayem je Management.

### Zašto CSV deli mrežu sa Live Migration

U normalnom radu CSV mreža nosi samo metapodatke — kreiranje fajlova, proširenje VHDX-a, promena veličine. Sav stvarni I/O ide direktno preko SAN putanja.

CSV mreža postaje opterećena tek u **redirected I/O** režimu, kada nod izgubi pristup LUN-u i sav I/O se tunelira kroz coordinator nod. Za klaster od dva noda sa ispravnim MPIO-om to je redak scenario, pa je zaseban vNIC za CSV nepotrebna komplikacija.

---

## Preduslovi

- Windows Server 2025 instaliran na oba hosta
- **Pristup preko out-of-band konzole (iDRAC / iLO / IPMI)** — obavezno, veza preko RDP-a puca u Koraku 4
- Trunk portovi na switchu sa propuštenim VLAN-ovima
- Hostovi u workgroup-u (domen dolazi kasnije)

> **Upozorenje:** Ne pokreći ove komande preko RDP-a. Kada fizički NIC uđe u SET tim, Windows otkači IP stek sa njega i sesija puca.

---

## Korak 1 — Instalacija rola

Provera trenutnog stanja:

```powershell
Get-WindowsFeature Hyper-V, Failover-Clustering, Multipath-IO |
    Format-Table Name, InstallState -AutoSize
```

Instalacija:

```powershell
Install-WindowsFeature -Name Hyper-V, Failover-Clustering, Multipath-IO `
    -IncludeManagementTools -Restart
```

Reboot je obavezan. Nakon podizanja sistema potvrdi da su Hyper-V cmdlet-i dostupni:

```powershell
Get-Command New-VMSwitch
```

> Ako `New-VMSwitch` ne postoji, Hyper-V modul nije učitan — rola nije instalirana ili reboot nije odrađen.

---

## Korak 2 — Identifikacija i preimenovanje NIC-eva

```powershell
Get-NetAdapter | Sort-Object Name |
    Format-Table Name, InterfaceDescription, LinkSpeed, Status, MacAddress -AutoSize
```

Zabeleži MAC adrese pre preimenovanja — koriste se za identifikaciju ako konfiguracija krene naopako.

```powershell
Rename-NetAdapter -Name "NIC in Slot 5 Port 1-1" -NewName "NIC1"
Rename-NetAdapter -Name "NIC in Slot 5 Port 2-1" -NewName "NIC2"
Rename-NetAdapter -Name "NIC in Slot 5 Port 3-1" -NewName "NIC3"
Rename-NetAdapter -Name "NIC in Slot 5 Port 4-1" -NewName "NIC4"

Get-NetAdapter | Format-Table Name, Status, LinkSpeed -AutoSize
```

**Kontrola:** sva četiri adaptera moraju biti u stanju `Up` sa brzinom `10 Gbps`. Adapter u stanju `Disconnected` znači problem sa kablom ili switch portom — reši pre nastavka.

---

## Korak 3 — Osnovne postavke na fizičkim portovima

```powershell
foreach ($n in "NIC1","NIC2","NIC3","NIC4") {
    Enable-NetAdapterRss -Name $n -ErrorAction SilentlyContinue
    Disable-NetAdapterPowerManagement -Name $n -ErrorAction SilentlyContinue
}
```

RSS raspoređuje obradu paketa po više CPU jezgara. Bez njega jedan tok ostaje na jednom jezgru i ne može da iskoristi 10 Gb link.

### Jumbo frames (opciono)

```powershell
foreach ($n in "NIC1","NIC2","NIC3","NIC4") {
    Set-NetAdapterAdvancedProperty -Name $n -DisplayName "Jumbo Packet" `
        -DisplayValue "9014 Bytes" -ErrorAction SilentlyContinue
}
```

> Jumbo frames zahtevaju MTU 9216 na switchevima. Ako switch konfiguracija nije pod tvojom kontrolom, preskoči ovaj deo — klaster radi sasvim dobro na MTU 1500.

---

## Korak 4 — Kreiranje vSwitch-eva

> **Ovde puca veza ako radiš preko RDP-a.**

```powershell
New-VMSwitch -Name "vSwitch-Host" -NetAdapterName "NIC1","NIC2" `
    -EnableEmbeddedTeaming $true -AllowManagementOS $false `
    -MinimumBandwidthMode Weight
Set-VMSwitchTeam -Name "vSwitch-Host" -LoadBalancingAlgorithm Dynamic

New-VMSwitch -Name "vSwitch-VM" -NetAdapterName "NIC3","NIC4" `
    -EnableEmbeddedTeaming $true -AllowManagementOS $false
Set-VMSwitchTeam -Name "vSwitch-VM" -LoadBalancingAlgorithm Dynamic
```

### Objašnjenje parametara

| Parametar | Razlog |
|---|---|
| `-AllowManagementOS $false` | Sprečava automatsko kreiranje vNIC-a bez VLAN taga. Takav adapter uzrokuje gubitak pristupa čim se management VLAN tagira. vNIC-ovi se dodaju ručno u Koraku 5. |
| `-MinimumBandwidthMode Weight` | **Može se zadati isključivo pri kreiranju switcha.** Bez njega QoS težine iz Koraka 7 neće raditi. |
| `-LoadBalancingAlgorithm Dynamic` | Raspoređuje odlazni saobraćaj po oba fizička porta, uključujući saobraćaj jednog vNIC-a. |

> **Imena vSwitch-eva moraju biti identična na svim hostovima u klasteru.** Live Migration poredi imena, ne konfiguraciju — različito ime znači neuspelu migraciju.

**Kontrola:**

```powershell
Get-VMSwitch | Format-Table Name, EmbeddedTeamingEnabled, AllowManagementOS, BandwidthReservationMode -AutoSize
Get-VMSwitchTeam | Format-Table Name, NetAdapterName, LoadBalancingAlgorithm -AutoSize
```

---

## Korak 5 — Host vNIC-ovi i VLAN-ovi

```powershell
Add-VMNetworkAdapter -ManagementOS -Name "Mgmt" -SwitchName "vSwitch-Host"
Add-VMNetworkAdapter -ManagementOS -Name "LM"   -SwitchName "vSwitch-Host"

Set-VMNetworkAdapterVlan -ManagementOS -VMNetworkAdapterName "Mgmt" -Access -VlanId 110
Set-VMNetworkAdapterVlan -ManagementOS -VMNetworkAdapterName "LM"   -Access -VlanId 111
```

**Kontrola:**

```powershell
Get-VMNetworkAdapterVlan -ManagementOS |
    Format-Table ParentAdapter, OperationMode, AccessVlanId -AutoSize
```

---

## Korak 6 — IP adresacija

### HV01

```powershell
New-NetIPAddress -InterfaceAlias "vEthernet (Mgmt)" `
    -IPAddress 10.0.10.11 -PrefixLength 24 -DefaultGateway 10.0.10.1

New-NetIPAddress -InterfaceAlias "vEthernet (LM)" `
    -IPAddress 10.0.11.11 -PrefixLength 24
```

### HV02

```powershell
New-NetIPAddress -InterfaceAlias "vEthernet (Mgmt)" `
    -IPAddress 10.0.10.12 -PrefixLength 24 -DefaultGateway 10.0.10.1

New-NetIPAddress -InterfaceAlias "vEthernet (LM)" `
    -IPAddress 10.0.11.12 -PrefixLength 24
```

### Zajedničke postavke na oba hosta

```powershell
# Management ima prioritet nad LM mrežom
Set-NetIPInterface -InterfaceAlias "vEthernet (Mgmt)" -AddressFamily IPv4 -InterfaceMetric 10
Set-NetIPInterface -InterfaceAlias "vEthernet (LM)"   -AddressFamily IPv4 -InterfaceMetric 20

# LM se ne registruje u DNS-u
Set-DnsClient -InterfaceAlias "vEthernet (LM)" -RegisterThisConnectionsAddress $false

# IPv6 nije potreban na LM mreži
Disable-NetAdapterBinding -Name "vEthernet (LM)" -ComponentID ms_tcpip6 -ErrorAction SilentlyContinue
```

### DNS serveri

```powershell
Set-DnsClientServerAddress -InterfaceAlias "vEthernet (Mgmt)" `
    -ServerAddresses 10.0.10.20,10.0.10.21
```

> Ako su domenski kontroleri virtuelne mašine na ovim istim hostovima, DNS serveri se postavljaju tek nakon njihove promocije. Do tada ostavi prazno.

---

## Korak 7 — QoS

```powershell
Set-VMSwitch -Name "vSwitch-Host" -DefaultFlowMinimumBandwidthWeight 10
Set-VMNetworkAdapter -ManagementOS -Name "Mgmt" -MinimumBandwidthWeight 30
Set-VMNetworkAdapter -ManagementOS -Name "LM"   -MinimumBandwidthWeight 60
```

Težine su relativne i dolaze do izražaja samo kada je link zasićen.

### Alternativa bez QoS-a

Ako je switch kreiran bez `-MinimumBandwidthMode Weight`, gornje komande vraćaju grešku `Failed while modifying virtual Ethernet switch connection settings`. Umesto rekreiranja switcha, ograniči Live Migration direktno:

```powershell
Set-SmbBandwidthLimit -Category LiveMigration -BytesPerSecond 750MB
```

Oko 6 Gb od ukupnih 20 Gb — management ostaje prohodan tokom migracije.

---

## Korak 8 — Live Migration

```powershell
Enable-VMMigration
Set-VMHost -VirtualMachineMigrationAuthenticationType Kerberos
Set-VMHost -VirtualMachineMigrationPerformanceOption SMB
Set-VMHost -MaximumVirtualMachineMigrations 2 -MaximumStorageMigrations 2

Enable-NetAdapterRss -Name "vEthernet (LM)"
```

> Kerberos autentikacija zahteva da su hostovi u domenu i da je podešena constrained delegation. Dok su hostovi u workgroup-u, koristi `CredSSP`.

Nakon kreiranja klastera, ograniči Live Migration na odgovarajuću mrežu:

```powershell
Get-ClusterResourceType "Virtual Machine" | Set-ClusterParameter -Name MigrationExcludeNetworks `
    -Value ([String]::Join(";",(Get-ClusterNetwork | Where-Object {$_.Name -notlike "*LM*"}).ID))
```

---

## Korak 9 — Hosts fajl (privremeno)

Dok hostovi nisu u domenu i nema DNS servera, dodaj statičke zapise na oba hosta:

```powershell
Add-Content -Path "$env:SystemRoot\System32\drivers\etc\hosts" -Value @"
10.0.10.11   HV01
10.0.10.12   HV02
"@
```

> Obriši ove zapise nakon podizanja DNS-a.

---

## Korak 10 — Provera

```powershell
# IP konfiguracija
Get-NetIPAddress -AddressFamily IPv4 |
    Where-Object InterfaceAlias -like "vEthernet*" |
    Format-Table InterfaceAlias, IPAddress, PrefixLength -AutoSize

# Default ruta — sme postojati samo jedna, na Mgmt interfejsu
Get-NetRoute -DestinationPrefix "0.0.0.0/0" |
    Format-Table InterfaceAlias, NextHop -AutoSize

# Stanje interfejsa
Get-NetIPInterface -AddressFamily IPv4 |
    Format-Table InterfaceAlias, ConnectionState, InterfaceMetric -AutoSize

# Konektivnost
Test-NetConnection 10.0.10.1  -InformationLevel Detailed
Test-NetConnection 10.0.10.12 -InformationLevel Detailed
Test-NetConnection 10.0.11.12 -InformationLevel Detailed
```

### Kriterijumi prolaza

| Provera | Očekivano |
|---|---|
| Ping ka gatewayu | uspešan |
| Ping između hostova po Mgmt mreži | uspešan |
| Ping između hostova po LM mreži | uspešan |
| Default ruta | samo na `vEthernet (Mgmt)` |
| `ConnectionState` oba vNIC-a | `Connected` |

U ispisu `Test-NetConnection` proveri i polja `InterfaceAlias` i `SourceAddress` — moraju odgovarati mreži koja se testira. Ako test ka LM adresi izlazi kroz `vEthernet (Mgmt)`, saobraćaj se rutira preko default gatewaya umesto on-link.

---

## VLAN-ovi za virtuelne mašine

Na `vSwitch-VM` se **ne konfiguriše nijedan VLAN**. Switch prosleđuje tagove sa trunk porta, a VLAN se zadaje po pojedinačnom VM adapteru:

```powershell
# VM u produkcionom VLAN-u
Add-VMNetworkAdapter -VMName "SRV-APP01" -SwitchName "vSwitch-VM" -Name "NET"
Set-VMNetworkAdapterVlan -VMName "SRV-APP01" -VMNetworkAdapterName "NET" -Access -VlanId 120

# VM u DMZ VLAN-u
Add-VMNetworkAdapter -VMName "SRV-WEB01" -SwitchName "vSwitch-VM" -Name "NET"
Set-VMNetworkAdapterVlan -VMName "SRV-WEB01" -VMNetworkAdapterName "NET" -Access -VlanId 130
```

VM bez zadatog VLAN-a radi u native VLAN-u trunk porta.

### Zaštite na VM adapterima

```powershell
Set-VMNetworkAdapter -VMName "SRV-WEB01" -MacAddressSpoofing Off -DhcpGuard On -RouterGuard On
```

| Opcija | Efekat |
|---|---|
| `MacAddressSpoofing Off` | VM ne može da menja izvornu MAC adresu |
| `DhcpGuard On` | Blokira DHCP odgovore iz VM-a |
| `RouterGuard On` | Blokira router advertisement poruke iz VM-a |

---

## Konfiguracija na switchu

Portovi se unakrsno raspoređuju po switchevima:

```
NIC1 -> SW1   trunk: VLAN 110 (native), 111
NIC2 -> SW2   trunk: VLAN 110 (native), 111
NIC3 -> SW1   trunk: VLAN 110, 120, 130
NIC4 -> SW2   trunk: VLAN 110, 120, 130
```

SET je switch-independent — **LACP, MLAG i VLT nisu potrebni.** Dovoljno je da oba switcha imaju isti set VLAN-ova.

### Preporuke

- **Native VLAN = management** na portovima ka `vSwitch-Host`. Host ostaje dostupan odmah nakon instalacije, pre konfigurisanja VLAN-ova.
- **`spanning-tree portfast trunk`** (ili `edge-port`, zavisno od vendora) na svim portovima ka hostovima.
- VLAN 110 mora biti propušten i na portovima ka `vSwitch-VM` ako se u njemu nalaze infrastrukturne VM-ove poput domenskih kontrolera.

### Dodavanje VLAN-a u postojeći trunk

```
interface range Gi1/0/1-2
 switchport trunk allowed vlan add 111
```

> Ključna je reč **`add`**. Bez nje se prepisuje cela lista dozvoljenih VLAN-ova i gubi se management pristup hostu.

---

## Troubleshooting

### `New-VMSwitch` nije prepoznata komanda

```
The term 'New-VMSwitch' is not recognized as the name of a cmdlet...
```

Hyper-V modul nije učitan. Instaliraj rolu i restartuj server (Korak 1).

---

### QoS komande vraćaju `Failed while modifying virtual Ethernet switch connection settings`

`MinimumBandwidthMode` se zadaje isključivo pri kreiranju switcha. Provera:

```powershell
Get-VMSwitch | Format-Table Name, BandwidthReservationMode -AutoSize
```

Rešenja:

1. Koristi `Set-SmbBandwidthLimit` umesto QoS težina (vidi Korak 7).
2. Rekreiraj switch sa `-MinimumBandwidthMode Weight`, pa ponovi Korake 5 i 6.

> Brisanje vSwitch-a uklanja i sve vNIC-ove sa njihovim IP adresama.

---

### Ping ka LM adresi drugog hosta ne prolazi, Mgmt prolazi

U ispisu `Test-NetConnection` proveri `InterfaceAlias` i `SourceAddress`:

```
ComputerName   : 10.0.11.12
InterfaceAlias : vEthernet (Mgmt)     <- trebalo bi vEthernet (LM)
SourceAddress  : 10.0.10.11           <- trebalo bi 10.0.11.11
PingSucceeded  : False
```

Saobraćaj izlazi kroz pogrešan interfejs i ide ka default gatewayu.

**Dijagnostika na oba hosta:**

```powershell
Get-NetIPInterface -AddressFamily IPv4 |
    Format-Table InterfaceAlias, ConnectionState, InterfaceMetric -AutoSize

Get-NetRoute -AddressFamily IPv4 |
    Where-Object DestinationPrefix -like "10.0.1*" |
    Format-Table DestinationPrefix, InterfaceAlias, NextHop -AutoSize

Get-VMNetworkAdapterVlan -ManagementOS |
    Format-Table ParentAdapter, OperationMode, AccessVlanId -AutoSize
```

**Mogući uzroci, po verovatnoći:**

1. **VLAN nije propušten na trunk portovima** — najčešći uzrok. Jedan VLAN prolazi, drugi ne, iako je port u trunk režimu.
2. **Drugi host nema LM vNIC ili IP adresu** — konfiguracija nije dovršena na oba hosta.
3. **LM interfejs je u stanju `Disconnected`** — retko, obično nakon neuspelog `Set-VMNetworkAdapterVlan`.

Ako su na oba hosta `ConnectionState = Connected`, ruta za LM subnet postoji na LM interfejsu, i VLAN je ispravno postavljen — problem je isključivo na switchu.

Prisilni test kroz konkretan interfejs:

```powershell
ping -S 10.0.11.11 10.0.11.12
```

---

### Gubitak pristupa nakon kreiranja vSwitch-a

Očekivano ponašanje. Fizički NIC gubi IP stek kada uđe u SET tim. Pristupi preko out-of-band konzole i nastavi od Koraka 5.

---

### Privremeni netagovani pristup hostu

Ako treba direktan pristup sa laptopa dok VLAN-ovi nisu sređeni:

```powershell
Set-VMNetworkAdapterVlan -ManagementOS -VMNetworkAdapterName "Mgmt" -Untagged
```

Vraćanje:

```powershell
Set-VMNetworkAdapterVlan -ManagementOS -VMNetworkAdapterName "Mgmt" -Access -VlanId 110
```

Paralelni privremeni adapter:

```powershell
Add-VMNetworkAdapter -ManagementOS -Name "Temp" -SwitchName "vSwitch-Host"
Set-VMNetworkAdapterVlan -ManagementOS -VMNetworkAdapterName "Temp" -Untagged
New-NetIPAddress -InterfaceAlias "vEthernet (Temp)" -IPAddress 192.168.50.11 -PrefixLength 24
Set-DnsClient -InterfaceAlias "vEthernet (Temp)" -RegisterThisConnectionsAddress $false

# brisanje nakon upotrebe
Remove-VMNetworkAdapter -ManagementOS -Name "Temp"
```

> Laptop se **ne kači direktno** u port koji je član SET tima. Dynamic load balancing može poslati odgovor kroz drugi fizički port, pa veza radi nepouzdano ili uopšte ne radi.

---

## Izveštaj o konfiguraciji

Skript prikuplja kompletno stanje mreže i upisuje ga u fajl. Pokrenuti na oba hosta.

```powershell
$out = "C:\Temp\NET-$env:COMPUTERNAME-$(Get-Date -f yyyyMMdd-HHmm).txt"
New-Item -ItemType Directory -Path C:\Temp -Force | Out-Null

Start-Transcript -Path $out

Write-Host "`n===== HOST =====" -ForegroundColor Cyan
Get-ComputerInfo -Property CsName, OsName, OsVersion, CsDomain, CsDomainRole | Format-List

Write-Host "`n===== FIZICKI NIC-evi =====" -ForegroundColor Cyan
Get-NetAdapter | Sort-Object Name |
    Format-Table Name, InterfaceDescription, Status, LinkSpeed, MacAddress -AutoSize

Write-Host "`n===== vSWITCH-evi =====" -ForegroundColor Cyan
Get-VMSwitch |
    Format-Table Name, SwitchType, EmbeddedTeamingEnabled, AllowManagementOS, BandwidthReservationMode -AutoSize

Write-Host "`n===== SET TIMOVI =====" -ForegroundColor Cyan
Get-VMSwitchTeam |
    Format-Table Name, @{N='Members';E={$_.NetAdapterName -join ', '}}, LoadBalancingAlgorithm, TeamingMode -AutoSize

Write-Host "`n===== HOST vNIC-evi =====" -ForegroundColor Cyan
Get-VMNetworkAdapter -ManagementOS |
    Format-Table Name, SwitchName, MacAddress, Status -AutoSize

Write-Host "`n===== VLAN-ovi =====" -ForegroundColor Cyan
Get-VMNetworkAdapterVlan -ManagementOS |
    Format-Table @{N='Adapter';E={$_.ParentAdapter.Name}}, OperationMode, AccessVlanId -AutoSize

Write-Host "`n===== IP KONFIGURACIJA =====" -ForegroundColor Cyan
Get-NetIPAddress -AddressFamily IPv4 |
    Where-Object { $_.InterfaceAlias -notlike "Loopback*" } |
    Sort-Object InterfaceAlias |
    Format-Table InterfaceAlias, IPAddress, PrefixLength, PrefixOrigin -AutoSize

Write-Host "`n===== INTERFEJSI =====" -ForegroundColor Cyan
Get-NetIPInterface -AddressFamily IPv4 |
    Sort-Object InterfaceMetric |
    Format-Table InterfaceAlias, ConnectionState, InterfaceMetric, Dhcp, NlMtu -AutoSize

Write-Host "`n===== RUTE =====" -ForegroundColor Cyan
Get-NetRoute -AddressFamily IPv4 |
    Where-Object { $_.DestinationPrefix -eq "0.0.0.0/0" -or $_.DestinationPrefix -like "10.0.1*" } |
    Format-Table DestinationPrefix, InterfaceAlias, NextHop, RouteMetric -AutoSize

Write-Host "`n===== DNS =====" -ForegroundColor Cyan
Get-DnsClientServerAddress -AddressFamily IPv4 |
    Where-Object { $_.InterfaceAlias -notlike "Loopback*" } |
    Format-Table InterfaceAlias, @{N='Servers';E={$_.ServerAddresses -join ', '}} -AutoSize

Write-Host "`n===== ROLE =====" -ForegroundColor Cyan
Get-WindowsFeature Hyper-V, Failover-Clustering, Multipath-IO, Data-Center-Bridging |
    Format-Table Name, InstallState -AutoSize

Write-Host "`n===== LIVE MIGRATION =====" -ForegroundColor Cyan
Get-VMHost | Format-List VirtualMachineMigrationEnabled,
    VirtualMachineMigrationAuthenticationType,
    VirtualMachineMigrationPerformanceOption,
    MaximumVirtualMachineMigrations, VirtualMachinePath, VirtualHardDiskPath

Write-Host "`n===== KONEKTIVNOST =====" -ForegroundColor Cyan
$peer = if ($env:COMPUTERNAME -eq "HV01") { @("10.0.10.12","10.0.11.12") }
        else { @("10.0.10.11","10.0.11.11") }

foreach ($t in @("10.0.10.1") + $peer) {
    $r = Test-NetConnection $t -WarningAction SilentlyContinue
    "{0,-16} Ping={1,-6} Via={2}" -f $t, $r.PingSucceeded, $r.InterfaceAlias
}

Stop-Transcript
```

---

## Sledeći koraci

Nakon završene mrežne konfiguracije:

1. Podizanje domenskih kontrolera kao neklasterovanih VM-ova na lokalnim diskovima hostova
2. Učlanjenje hostova u domen
3. Konfiguracija MPIO i povezivanje sa deljenim storage-om
4. Validacija klastera (`Test-Cluster`)
5. Kreiranje klastera, quorum, CSV
6. Cluster-Aware Updating

> **Domenski kontroleri ostaju na lokalnim diskovima hostova, nikada na CSV-u.** U suprotnom se pri hladnom startu celog okruženja stvara zavisnost u krug: klaster se ne podiže bez AD-a, a AD se ne podiže bez klastera.

---

## Licenca

MIT
