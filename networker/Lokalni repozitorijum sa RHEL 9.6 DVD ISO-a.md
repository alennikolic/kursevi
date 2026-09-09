# Lokalni repozitorijum sa RHEL 9.6 DVD ISO-a (bez interneta)

**Namena:** procedura za instalaciju OS paketa na RHEL 9.6 serveru koji nema pristup internetu ni Satellite/CDN-u.
Koristi se `rhel-9.6-x86_64-dvd.iso`, koji sadrži oba repozitorijuma: **BaseOS** i **AppStream**.

---

## 1. Kopiranje ISO fajla na server

ISO prebaciti na server (npr. preko `scp`, USB-a ili ISO-a montiranog u hipervizoru).

```bash
[root@nwsrv ~]# mkdir -p /var/iso
[root@nwsrv ~]# scp rhel-9.6-x86_64-dvd.iso root@10.10.20.15:/var/iso/
rhel-9.6-x86_64-dvd.iso                       100%  9216MB   112.4MB/s   01:22
```

Provera:

```bash
[root@nwsrv ~]# ls -lh /var/iso/
total 9.1G
-rw-r--r--. 1 root root 9.1G Sep  9 09:14 rhel-9.6-x86_64-dvd.iso
```

> ISO staviti na particiju koja ima bar ~10 GB slobodno. Ne stavljati ga u `/tmp` (briše se pri restartu).

---

## 2. Kreiranje tačke montiranja i montiranje

```bash
[root@nwsrv ~]# mkdir -p /mnt/rhel9

[root@nwsrv ~]# mount -o loop,ro /var/iso/rhel-9.6-x86_64-dvd.iso /mnt/rhel9
mount: /mnt/rhel9: WARNING: source write-protected, mounted read-only.
```

Provera sadržaja — moraju postojati direktorijumi `BaseOS` i `AppStream`:

```bash
[root@nwsrv ~]# ls /mnt/rhel9
AppStream  BaseOS  EFI  extra_files.json  images  isolinux  media.repo  RPM-GPG-KEY-redhat-release
```

```bash
[root@nwsrv ~]# ls /mnt/rhel9/BaseOS
Packages  repodata
[root@nwsrv ~]# ls /mnt/rhel9/AppStream
Packages  repodata
```

Direktorijum `repodata` mora postojati — on je metapodatak repozitorijuma.

---

## 3. Trajno montiranje (preživi restart)

Da se ISO montira automatski posle reboot-a, dodati liniju u `/etc/fstab`:

```bash
[root@nwsrv ~]# cat >> /etc/fstab <<'EOF'
/var/iso/rhel-9.6-x86_64-dvd.iso  /mnt/rhel9  iso9660  loop,ro,nofail  0 0
EOF
```

Test da unos nije pogrešan (ovo je obavezan korak — greška u fstab-u može sprečiti podizanje sistema):

```bash
[root@nwsrv ~]# umount /mnt/rhel9
[root@nwsrv ~]# mount -a
[root@nwsrv ~]# df -h /mnt/rhel9
Filesystem      Size  Used Avail Use% Mounted on
/dev/loop0      9.1G  9.1G     0 100% /mnt/rhel9
```

> `nofail` znači da se sistem podiže i ako ISO iz nekog razloga nedostaje.

---

## 4. Kreiranje repo fajla

```bash
[root@nwsrv ~]# cat > /etc/yum.repos.d/rhel9-local.repo <<'EOF'
[rhel9-baseos-local]
name=RHEL 9.6 BaseOS - Local ISO
baseurl=file:///mnt/rhel9/BaseOS
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[rhel9-appstream-local]
name=RHEL 9.6 AppStream - Local ISO
baseurl=file:///mnt/rhel9/AppStream
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF
```

Obratiti pažnju: `file://` + apsolutna putanja `/mnt/rhel9/...` daje **tri kose crte** — `file:///mnt/rhel9/BaseOS`.

Provera fajla:

```bash
[root@nwsrv ~]# cat /etc/yum.repos.d/rhel9-local.repo
[rhel9-baseos-local]
name=RHEL 9.6 BaseOS - Local ISO
baseurl=file:///mnt/rhel9/BaseOS
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
...
```

---

## 5. Import GPG ključa

```bash
[root@nwsrv ~]# rpm --import /mnt/rhel9/RPM-GPG-KEY-redhat-release
```

Bez izlaza = uspešno.

> Ako ne želite proveru potpisa, umesto ovoga stavite `gpgcheck=0` u repo fajlu. Preporuka je da ostane `gpgcheck=1`.

---

## 6. Isključivanje online repozitorijuma (ako postoje)

Ako je server ranije bio registrovan na CDN, `dnf` će pokušavati da ga kontaktira i praviti timeout-e.

```bash
[root@nwsrv ~]# ls /etc/yum.repos.d/
redhat.repo  rhel9-local.repo
```

```bash
[root@nwsrv ~]# subscription-manager config --rhsm.manage_repos=0
[root@nwsrv ~]# mv /etc/yum.repos.d/redhat.repo /root/redhat.repo.bak
```

---

## 7. Provera repozitorijuma

```bash
[root@nwsrv ~]# dnf clean all
36 files removed

[root@nwsrv ~]# dnf repolist
repo id                    repo name
rhel9-appstream-local      RHEL 9.6 AppStream - Local ISO
rhel9-baseos-local         RHEL 9.6 BaseOS - Local ISO
```

Detaljan prikaz sa brojem paketa:

```bash
[root@nwsrv ~]# dnf repolist -v | egrep 'Repo-id|Repo-pkgs'
Repo-id            : rhel9-appstream-local
Repo-pkgs          : 5 421
Repo-id            : rhel9-baseos-local
Repo-pkgs          : 1 748
```

Ako je `Repo-pkgs` nula, `baseurl` je pogrešan ili ISO nije montiran.

---

## 8. Test instalacije paketa

```bash
[root@nwsrv ~]# dnf install -y ksh libnsl libnsl.i686 glibc.i686
RHEL 9.6 AppStream - Local ISO          12 MB/s | 8.4 MB     00:00
RHEL 9.6 BaseOS - Local ISO             15 MB/s | 3.1 MB     00:00
Dependencies resolved.
=========================================================================
 Package        Arch    Version              Repository             Size
=========================================================================
Installing:
 glibc          i686    2.34-125.el9_6       rhel9-baseos-local    4.2 M
 ksh            x86_64  1.0.0~beta.1-3.el9   rhel9-appstream-local 924 k
 libnsl         x86_64  2.34-125.el9_6       rhel9-baseos-local     58 k
 libnsl         i686    2.34-125.el9_6       rhel9-baseos-local     61 k

Transaction Summary
=========================================================================
Install  4 Packages

Complete!
```

Provera:

```bash
[root@nwsrv ~]# rpm -q ksh libnsl glibc.i686
ksh-1.0.0~beta.1-3.el9.x86_64
libnsl-2.34-125.el9_6.x86_64
glibc-2.34-125.el9_6.i686
```

---

## 9. Česti problemi

| Poruka / simptom | Uzrok i rešenje |
|---|---|
| `Failed to download metadata for repo 'rhel9-baseos-local'` | ISO nije montiran (`mount -a`) ili je pogrešan `baseurl` |
| `Cannot open: file:///mnt/rhel9/BaseOS/repodata/repomd.xml` | Pogrešan broj kosih crta u `file://` ili nedostaje `repodata` direktorijum |
| `GPG check FAILED` | Nije uvezen ključ — `rpm --import /mnt/rhel9/RPM-GPG-KEY-redhat-release` |
| `dnf` dugo visi pa javlja timeout | Aktivan `redhat.repo` prema CDN-u — isključiti kao u koraku 6 |
| Posle reboot-a `dnf` ne radi | Nije dodat unos u `/etc/fstab` (korak 3) |
| `No match for argument: <paket>` | Paket nije na DVD ISO-u; treba dodatni repo ili ručni RPM |

---

## 10. Uklanjanje lokalnog repoa (kad server dobije internet)

```bash
[root@nwsrv ~]# rm -f /etc/yum.repos.d/rhel9-local.repo
[root@nwsrv ~]# umount /mnt/rhel9
[root@nwsrv ~]# sed -i '\|rhel-9.6-x86_64-dvd.iso|d' /etc/fstab
[root@nwsrv ~]# subscription-manager config --rhsm.manage_repos=1
[root@nwsrv ~]# dnf clean all
```

---

## Skraćena verzija (copy-paste)

```bash
mkdir -p /var/iso /mnt/rhel9
# ISO već kopiran u /var/iso/

mount -o loop,ro /var/iso/rhel-9.6-x86_64-dvd.iso /mnt/rhel9

echo '/var/iso/rhel-9.6-x86_64-dvd.iso /mnt/rhel9 iso9660 loop,ro,nofail 0 0' >> /etc/fstab

cat > /etc/yum.repos.d/rhel9-local.repo <<'EOF'
[rhel9-baseos-local]
name=RHEL 9.6 BaseOS - Local ISO
baseurl=file:///mnt/rhel9/BaseOS
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[rhel9-appstream-local]
name=RHEL 9.6 AppStream - Local ISO
baseurl=file:///mnt/rhel9/AppStream
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF

rpm --import /mnt/rhel9/RPM-GPG-KEY-redhat-release
dnf clean all
dnf repolist
```

---

### Napomena o izlazima komandi

Prikazani izlazi su reprezentativni primeri — verzije paketa, veličine i broj paketa u repozitorijumu razlikuju se u zavisnosti od tačnog izdanja ISO-a.
