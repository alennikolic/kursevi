# Proxmox VE 9.3: kurs na srpskom jeziku

Praktičan kurs za **Proxmox Virtual Environment 9.3**, sa fokusom na **produkciona okruženja**. Svako poglavlje ima objašnjenja, primere komandi, deo o najboljim praksama i čestim greškama u produkciji, vežbu za laboratoriju i pitanja za proveru znanja.

Kurs prati zvaničnu dokumentaciju **Proxmox VE 9.3.1** (novembar 2026).

## Sadržaj

| # | Poglavlje | Opis |
|---|---|---|
| 1 | [Uvod u Proxmox VE 9.3](01-uvod-u-proxmox-ve.md) | Šta je Proxmox, KVM i LXC, arhitektura, novine u 9.3, laboratorija za vežbu |
| 2 | [Instalacija i prvo podešavanje](02-instalacija-i-prvo-podesavanje.md) | Hardver, instaler (bond/VLAN), repozitorijumi, ažuriranje, razlike u produkciji |
| 3 | [Administracija hosta i web interfejs](03-administracija-hosta-i-web-interfejs.md) | GUI, CLI alati, logovi, ACME sertifikati, zaštita hosta, ažuriranje i nadzor |
| 4 | [Mrežna podešavanja i SDN](04-mreza-i-sdn.md) | Bridge, bond, VLAN, razdvajanje mreža u produkciji, Software-Defined Network |
| 5 | [Skladišta podataka (storage)](05-skladista-podataka.md) | LVM-thin, ZFS, NFS, SAN sa multipath-om, snapshotovi na deljenom LVM-u |
| 6 | [Virtuelne mašine (QEMU/KVM)](06-virtuelne-masine.md) | VM po najboljim praksama, CPU tip, cloud-init, migracija, uvoz sa VMware-a |
| 7 | [Kontejneri (LXC)](07-kontejneri-lxc.md) | Šabloni, neprivilegovani kontejneri, mount point-ovi, Docker i LXC |
| 8 | [Klaster i visoka dostupnost (HA)](08-klaster-i-ha.md) | Kvorum, corosync, QDevice, HA i fencing, HA pravila, režim održavanja |
| 9 | [Ceph i replikacija](09-ceph-i-replikacija.md) | Hiperkonvergentni Ceph, kapacitet, održavanje, ZFS replikacija i RPO |
| 10 | [Bezbednost, backup i notifikacije](10-bezbednost-backup-notifikacije.md) | Dozvole, API tokeni, firewall, Proxmox Backup Server, notifikacije, završna kontrolna lista |

## Za koga je kurs

Za administratore sistema koji znaju osnove Linuxa i mreža, a žele da nauče Proxmox VE i da ga koriste u produkciji, uključujući one koji prelaze sa VMware-a.

## Šta ti treba za vežbe

- jedan računar sa podrškom za virtuelizaciju (Intel VT-x / AMD-V), 16 GB RAM-a ili više i SSD,
- za poglavlja 8 i 9: tri Proxmox čvora, fizička ili ugnježdena (nested) u VM-ovima.

## Napomena

Kurs je nezvaničan i nije povezan sa kompanijom Proxmox Server Solutions GmbH. Pre primene u produkciji uvek proveri detalje u [zvaničnoj dokumentaciji](https://pve.proxmox.com/pve-docs/) za svoju verziju.
