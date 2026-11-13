# Instalacija domenskih kontrolera na Hyper-V klasteru

Uputstvo za podizanje dva domenska kontrolera (Windows Server 2025) kao virtuelnih mašina na dva Hyper-V hosta.

## Arhitektura

| Komponenta | Opis |
|---|---|
| HOST01, HOST02 | Fizički Hyper-V hostovi |
| DC01 | VM na HOST01, lokalni disk hosta |
| DC02 | VM na HOST02, lokalni disk hosta |

Domenski kontroleri se **ne** smeštaju na deljeni klasterski storage (CSV). Razlog: klaster za svoj rad traži autentikaciju prema domenu, a ako su DC-ovi na CSV-u, pri hladnom startu nastaje međuzavisnost iz koje se teško izlazi. Lokalni diskovi hostova rešavaju taj problem.

Primeri adresa u uputstvu:

```
Domen:      example.local  (NetBIOS: EXAMPLE)
Mreža:      10.0.0.0/24, gateway 10.0.0.1
DC01:       10.0.0.20
DC02:       10.0.0.21
```

---

## 1. Priprema virtuelnih mašina

Na svakom hostu kreirati po jednu VM:

| Parametar | Vrednost |
|---|---|
| Generacija | Gen 2 |
| RAM | 4096 MB, fiksno (bez Dynamic Memory) |
| vCPU | 2–4 |
| Disk | 80–100 GB, lokalni disk hosta |
| Mrežni adapter | vSwitch sa VLAN-om management mreže |

```powershell
New-VM -Name DC01 -Generation 2 -MemoryStartupBytes 4GB `
  -NewVHDPath "D:\VMs\DC01\DC01.vhdx" -NewVHDSizeBytes 100GB `
  -SwitchName "vSwitch-VM" -Path "D:\VMs"

Set-VM -Name DC01 -ProcessorCount 4 -StaticMemory
Set-VMNetworkAdapterVlan -VMName DC01 -Access -VlanId 10
```

### Hyper-V podešavanja specifična za DC

Ova tri podešavanja su obavezna:

```powershell
# Automatski start posle reboota hosta
Set-VM -Name DC01 -AutomaticStartAction Start -AutomaticStopAction ShutDown

# Automatic checkpoints - nikada na DC-u
Set-VM -Name DC01 -AutomaticCheckpointsEnabled $false

# Time Synchronization integracija - isključiti
Disable-VMIntegrationService -VMName DC01 -Name "Time Synchronization"
```

**Zašto checkpointi:** vraćanje DC-a na snapshot izaziva USN rollback i trajno kvari replikaciju. AD ima sopstvene mehanizme za backup i restore.

**Zašto Time Sync:** ako je uključen, VM uzima vreme od hosta i time zaobilazi domensku vremensku hijerarhiju. Na PDC emulatoru to dovodi do toga da se DC prijavljuje kao nesinhronizovan i odbija da servira vreme ostalima.

---

## 2. Priprema operativnog sistema

Instalirati Windows Server 2025 (Desktop Experience ili Core), pa na svakoj VM:

```powershell
# Naziv
Rename-Computer -NewName DC01 -Restart
```

Posle reboota, statička IP adresa:

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" `
  -IPAddress 10.0.0.20 -PrefixLength 24 -DefaultGateway 10.0.0.1

# Privremeno: DC01 pokazuje na sebe do promocije
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
```

Isto na DC02, sa adresom `10.0.0.21` i DNS-om `10.0.0.20` (prvi DC).

Provere pre nastavka:

```powershell
Get-NetIPConfiguration -InterfaceAlias "Ethernet"
Test-NetConnection 10.0.0.1 -InformationLevel Detailed
Get-TimeZone
Get-Date
```

Vremenska zona i sat moraju biti tačni — prvi DC postaje vremenska referenca za ceo domen.

---

## 3. Instalacija prvog domenskog kontrolera (DC01)

### Instalacija role

```powershell
Install-WindowsFeature AD-Domain-Services, RSAT-DNS-Server -IncludeManagementTools
```

### Promocija u novu šumu

```powershell
Install-ADDSForest `
  -DomainName "example.local" `
  -DomainNetbiosName "EXAMPLE" `
  -ForestMode WinThreshold `
  -DomainMode WinThreshold `
  -InstallDns `
  -DatabasePath "C:\Windows\NTDS" `
  -LogPath "C:\Windows\NTDS" `
  -SysvolPath "C:\Windows\SYSVOL" `
  -Force
```

Komanda traži DSRM lozinku. Čuvati je u password manageru — potrebna je za Directory Services Restore Mode.

DNS rola se instalira automatski uz `-InstallDns`, ne treba je posebno dodavati.

Server se restartuje sam po završetku.

### Konfiguracija posle promocije

```powershell
# Reverse lookup zona
Add-DnsServerPrimaryZone -NetworkId "10.0.0.0/24" -ReplicationScope Forest

# Forwarderi
Add-DnsServerForwarder -IPAddress 8.8.8.8, 1.1.1.1

# Provera
Get-DnsServerZone
```

### Vremenska sinhronizacija na PDC-u

Prvi DC drži PDC Emulator ulogu i služi kao izvor vremena za ceo domen.

Ako postoji eksterni ili interni NTP izvor:

```powershell
w32tm /config /manualpeerlist:"pool.ntp.org,0x8" /syncfromflags:manual /reliable:yes /update
Restart-Service w32time
w32tm /resync
w32tm /query /status
```

Ako NTP izvor nije dostupan (izolovana mreža), DC se proglašava autoritativnim na sopstvenom satu:

```powershell
w32tm /config /syncfromflags:NO /reliable:yes /update
Restart-Service w32time
```

Pre toga obavezno proveriti da je sat tačan.

Očekivani rezultat u oba slučaja:

```
Leap Indicator: 0(no warning)
```

Ako stoji `Leap Indicator: 3(not synchronized)`, DC neće servirati vreme ostalima i drugi DC se neće moći sinhronizovati.

Ako se izvor i dalje prikazuje kao `VM IC Time Synchronization Provider`, Hyper-V provajder je ostao aktivan u gostu:

```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\W32Time\TimeProviders\VMICTimeProvider" `
  -Name Enabled -Value 0 -Type DWord
Restart-Service w32time
```

---

## 4. Instalacija drugog domenskog kontrolera (DC02)

### Preduslovi

```powershell
# DNS mora pokazivati na DC01
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.0.0.20

# Provera da se domen vidi
nltest /dsgetdc:example.local
Resolve-DnsName example.local
```

Ako `nltest` ne uspe, DNS nije ispravan i promocija će pasti na autentikaciji.

### Instalacija i promocija

```powershell
Install-WindowsFeature AD-Domain-Services, RSAT-DNS-Server -IncludeManagementTools

Install-ADDSDomainController `
  -DomainName "example.local" `
  -InstallDns `
  -Credential (Get-Credential EXAMPLE\Administrator) `
  -SiteName "Default-First-Site-Name" `
  -DatabasePath "C:\Windows\NTDS" `
  -LogPath "C:\Windows\NTDS" `
  -SysvolPath "C:\Windows\SYSVOL" `
  -Force
```

Kredencijali moraju biti domenski (Domain Admin), ne lokalni administrator.

### Vreme na DC02

```powershell
w32tm /config /syncfromflags:domhier /update
Restart-Service w32time
w32tm /resync /rediscover
w32tm /query /source
```

Očekivani izlaz: `DC01.example.local`.

---

## 5. Ukrštanje DNS podešavanja

Posle promocije oba DC-a, DNS se ukršta:

```powershell
# Na DC01
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.0.0.21, 127.0.0.1

# Na DC02
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.0.0.20, 127.0.0.1
```

Primarni DNS je partnerski DC, sekundarni je loopback. Redosled je bitan: DC koji prvo pita samog sebe može ostati zarobljen u sopstvenom kešu i ne videti promene od partnera („DNS island"). Loopback služi kao rezerva kad je partner nedostupan.

Koristiti `127.0.0.1`, ne sopstvenu statičku adresu.

---

## 6. Firewall

Pravila za AD, DNS i Kerberos se uključuju automatski uz instalaciju role. Dodatno se uključuju pravila za daljinsku administraciju i dijagnostiku — bez njih `dcdiag` ne može da pročita event logove sa drugog DC-a.

Na oba DC-a:

```powershell
Enable-NetFirewallRule -DisplayGroup "Remote Event Log Management"
Enable-NetFirewallRule -DisplayGroup "Windows Management Instrumentation (WMI)"
Enable-NetFirewallRule -DisplayGroup "Remote Service Management"
Enable-NetFirewallRule -DisplayGroup "File and Printer Sharing"
Enable-NetFirewallRule -DisplayGroup "Remote Volume Management"
Enable-NetFirewallRule -DisplayGroup "Performance Logs and Alerts"
```

Ograničiti ih na domenski profil:

```powershell
Get-NetFirewallRule -DisplayGroup "Remote Event Log Management", `
  "Windows Management Instrumentation (WMI)", "Remote Service Management", `
  "File and Printer Sharing", "Remote Volume Management", `
  "Performance Logs and Alerts" |
  Where-Object Enabled -eq True | Set-NetFirewallRule -Profile Domain
```

### Profil mrežne veze

Proveriti kategoriju mreže:

```powershell
Get-NetConnectionProfile
```

Mora biti `DomainAuthenticated`. Ako stoji `Public`, firewall primenjuje javni profil i blokira RPC — posledica su greške `0x6ba "The RPC server is unavailable"` i `1908 "Could not find the domain controller"` pri replikaciji.

Uzrok je obično to što se NLA servis podigao pre NTDS-a. Rešenje:

```powershell
Restart-NetAdapter -Name "Ethernet"
Start-Sleep 20
Get-NetConnectionProfile
```

Ako ne pomogne, dodati zavisnost i restartovati server:

```powershell
sc.exe config nlasvc depend= NSI/RpcSs/TcpIp/Dhcp/Eventlog/NTDS
Restart-Computer
```

---

## 7. Provera

### Na DC01

```powershell
dcdiag /e /q
repadmin /replsummary
netdom query fsmo
Get-ADDomainController -Filter * | ft Name,IPv4Address,Site,IsGlobalCatalog
w32tm /query /status
Get-NetConnectionProfile
```

Očekivano:

- `dcdiag /e /q` bez izlaza
- `repadmin /replsummary` sa 0 failures u oba smera
- svih pet FSMO uloga na DC01
- `Leap Indicator: 0`
- `NetworkCategory: DomainAuthenticated`

### Na DC02

```powershell
Get-DnsClientServerAddress -InterfaceAlias "Ethernet" -AddressFamily IPv4
w32tm /query /source
Get-NetConnectionProfile
net share
Get-Service NTDS,Netlogon,DNS,DFSR,KDC | ft Name,Status
```

Očekivano: `SYSVOL` i `NETLOGON` podeljeni, svi servisi `Running`, izvor vremena DC01.

### SRV zapisi

```powershell
nslookup -type=SRV _ldap._tcp.dc._msdcs.example.local
```

Moraju se videti oba DC-a — klijenti i hostovi preko ovih zapisa pronalaze domen.

### Forsiranje replikacije

```powershell
# Povlačenje ka DC02
repadmin /syncall DC02 /Ade

# Guranje sa DC02
repadmin /syncall DC02 /AdeP
```

Zastavica `/P` menja smer: bez nje DC navedeni kao meta **povlači** izmene, sa njom ih **gura**. Ako greška stoji samo u jednom smeru, treba pustiti odgovarajuću varijantu.

---

## 8. Ubacivanje hostova u domen

Tek kada su obe provere čiste:

```powershell
# HOST01 - primarni DNS je DC na drugom hostu
Set-DnsClientServerAddress -InterfaceAlias "vEthernet (Mgmt)" `
  -ServerAddresses 10.0.0.21, 10.0.0.20
Add-Computer -DomainName example.local -Credential EXAMPLE\Administrator -Restart
```

```powershell
# HOST02
Set-DnsClientServerAddress -InterfaceAlias "vEthernet (Mgmt)" `
  -ServerAddresses 10.0.0.20, 10.0.0.21
Add-Computer -DomainName example.local -Credential EXAMPLE\Administrator -Restart
```

DNS se i ovde ukršta: host pokazuje prvo na DC koji se nalazi na **drugom** hostu. Kada se host restartuje, DC koji je na njemu tek počinje da se podiže, pa host dobija odgovor od onog koji već radi.

Hostove restartovati jedan po jedan i sačekati da se DC na tom hostu podigne pre nego što se dira drugi.

Provera:

```powershell
Test-ComputerSecureChannel -Verbose
Get-ComputerInfo | ft CsDomain,CsDomainRole
```

---

## Najčešći problemi

| Simptom | Uzrok | Rešenje |
|---|---|---|
| `Leap Indicator: 3`, izvor `Local CMOS Clock` | PDC nije sinhronizovan, odbija da servira vreme | Zadati NTP izvor ili `/reliable:yes` na PDC-u |
| Izvor vremena `VM IC Time Synchronization Provider` | Hyper-V Time Sync integracija aktivna | Isključiti integraciju i `VMICTimeProvider` u registru |
| `0x6ba "The RPC server is unavailable"` | Mrežni profil `Public` | Restart NLA ili adaptera, zavisnost na NTDS |
| `1908 "Could not find the domain controller"` | Isto — firewall blokira Kerberos | Isto |
| `The user name or password is incorrect` pri promociji DC02 | Lokalni umesto domenskih kredencijala, ili neispravan DNS | `EXAMPLE\Administrator` i DNS ka prvom DC-u |
| `w32tm /register` javlja `0x80070430` | Servis markiran za brisanje | Zatvoriti services.msc, sačekati ili restartovati |
| Nema `SYSVOL` i `NETLOGON` share-a na DC02 | Početna replikacija SYSVOL-a nije završena | Sačekati 10–15 min, pa `net share` ponovo |

---

## Kontrolna lista

- [ ] VM-ovi na lokalnim diskovima hostova, ne na CSV-u
- [ ] Automatic checkpoints isključeni
- [ ] Time Synchronization integracija isključena
- [ ] Automatic start action podešen na `Start`
- [ ] Statičke IP adrese i tačna vremenska zona
- [ ] Prva šuma podignuta, DSRM lozinka sačuvana
- [ ] Reverse zona i forwarderi podešeni
- [ ] PDC ima validan izvor vremena (`Leap Indicator: 0`)
- [ ] Drugi DC promovisan, vuče vreme sa prvog
- [ ] DNS ukršten na oba DC-a
- [ ] Firewall pravila uključena i ograničena na domenski profil
- [ ] Mrežni profil `DomainAuthenticated` na oba DC-a
- [ ] `dcdiag /e /q` čist, replikacija 0 failures
- [ ] SRV zapisi vraćaju oba DC-a
- [ ] Hostovi u domenu, secure channel ispravan
