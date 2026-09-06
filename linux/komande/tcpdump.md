# tcpdump - Analiza i snimanje mrežnog saobraćaja

## 1. Uvod i namena
`tcpdump` je najmoćniji komandno-linijski packet analyzer (sniffer) alat za presretanje, analizu i snimanje mrežnog saobraćaja u realnom vremenu. Radi direktno na mrežnom sloju (koristeći `libpcap` biblioteku) i omogućava inženjerima uvid u mrežne pakete koji prolaze kroz mrežne interfejse (mrežne kartice, virtuelne mostove, docker interfejse).

U enterprise okruženjima nezamenjiv je za troubleshooting mrežnih protokola (TCP, UDP, ICMP, BGP), analizu problema sa kašnjenjem i gubitkom paketa, inspekciju HTTP/DNS saobraćaja i forenzičku analizu bezbednosnih incidenata.

## 2. Sintaksa i opcije (Flags)
`tcpdump [opcije] [BPF FILTER]`

| Opcija | Dugi oblik | Značenje / Ponašanje |
|:---:|:---|:---|
| `-i` | --interface | Definiše mrežni interfejs na kome se hvata saobraćaj (npr. `-i eth0` ili `-i any`). |
| `-n` | - | Onemogućava DNS razrešavanje IP adresa u host imena radi ubrzanja i tačnosti. |
| `-nn` | - | Onemogućava i DNS razrešavanje IP adresa i prevođenje brojeva portova u nazive protokola. |
| `-v` / `-vv` | - | Povećava nivo detalja u prikazu (TTL, ID, dužina zaglavlja, opcije). |
| `-w` | - | Snima sirove pakete u `.pcap` fajl radi kasnije analize u Wireshark alatu. |
| `-r` | - | Čita i analizira prethodno snimljeni `.pcap` fajl. |
| `-c` | - | Zaustavlja snimanje nakon presretanja definisanog broja paketa. |
| `-A` | - | Prikazuje sadržaj paketa u ASCII formatu (korisno za čitanje npr. HTTP tekstualnih zahteva). |

## 3. Analiza izlaza (Output Breakdown)

```bash
$ sudo tcpdump -nn -i eth0 tcp port 80 -c 2
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
14:20:01.102341 IP 192.168.1.50.54210 > 10.0.1.100.80: Flags [S], seq 12904812, win 64240, options [mss 1460,sackOK], length 0
14:20:01.102650 IP 10.0.1.100.80 > 192.168.1.50.54210: Flags [S.], seq 40291011, ack 12904813, win 65535, options [mss 1460,sackOK], length 0
```

**Objašnjenje prikaza:**
- **14:20:01.102341:** Vreme hvatanja paketa sa preciznošću u mikrosekundama.
- **IP:** Protokol mrežnog sloja (IPv4).
- **192.168.1.50.54210 > 10.0.1.100.80:** Izvorna IP adresa i port (`192.168.1.50:54210`) šalje paket ka odredišnoj IP adresi i portu (`10.0.1.100:80`).
- **Flags [S]:** TCP flegovi u paketu. `[S]` = SYN (zahtev za uspostavljanje konekcije), `[S.]` = SYN-ACK, `[P.]` = PUSH-ACK (slanje podataka), `[F.]` = FIN (zatvaranje konekcije), `[R]` = RESET (nasilno prekidanje).
- **seq 12904812:** TCP Sequence broj paketa (identifikator redosleda podataka).
- **ack 12904813:** TCP Acknowledgment broj (potvrda prijema sledećeg očekivanog bajta).
- **win 64240:** Veličina TCP Window bafera (količina podataka koju prijemnik može prihvatiti).
- **length 0:** Veličina payload-a (sadržaja podataka) unutar samog mrežnog paketa u bajtovima.

## 4. Praktični primeri iz produkcije

- **Svrha:** Snimanje saobraćaja na specifičnom interfejsu za definisanu IP adresu i skladištenje u pcap fajl.
- **Komanda:** 
  ```bash
  sudo tcpdump -nn -i eth0 host 192.168.1.10 and port 443 -w /tmp/capture.pcap
  ```
- **Objašnjenje:** Koristi BPF (Berkeley Packet Filter) sintaksu za filtriranje saobraćaja koji ide od/ka IP `192.168.1.10` na portu 443, pominjući snimanje u fajl `/tmp/capture.pcap`.

- **Svrha:** Inspekcija neželjenih ICMP (Ping) paketa i mrežne nedostupnosti.
- **Komanda:** 
  ```bash
  sudo tcpdump -nn -i any icmp
  ```
- **Objašnjenje:** Sluša na svim mrežnim interfejsima (`-i any`) i filtrira isključivo ICMP protokol.

- **Svrha:** Čitanje nešifrovanog HTTP POST saobraćaja radi provere sadržaja formi ili API poziva.
- **Komanda:** 
  ```bash
  sudo tcpdump -nn -A -s 0 -i eth0 'tcp port 80 and (((ip[20:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'
  ```
- **Objašnjenje:** `-A` ispisuje payload u ASCII tekstualnom obliku, dok složeni BPF filter garantuje izdvajanje samo onih TCP paketa koji u sebi zapravo nose podatak (payload).

## 5. Pro Tips i "Gotchas" (Zamke)
- Uvek koristite `-nn` flag u produkciji! Bez ove opcije `tcpdump` će pokušavati da uradi DNS lookup za svaku IP adresu, što pri velikom saobraćaju uzrokuje ogroman kašnjenja i gubitak presretnutih paketa (dropped by kernel).
- Pazite na opterećenje diska prilikom snimanja sa opcijom `-w` na mrežnim interfejsima od 10Gbps+. pcap datoteke mogu popuniti disk za nekoliko sekundi.
- Komanda zahteva `sudo` ili `CAP_NET_RAW` / `CAP_NET_ADMIN` Linux sposobnosti.
