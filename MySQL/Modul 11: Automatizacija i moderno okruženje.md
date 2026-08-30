# Modul 11: Automatizacija i moderno okruženje

*MySQL za Linux administratore*

---

## Uvod: isti principi, drugi alati

Do sada smo sve radili ručno, na jednom serveru. To je bilo namerno — da razumete šta se zaista dešava pre nego što to prepustite alatu.

Ovaj modul je o tome kako se isto radi kada imate deset servera umesto jednog, kada baza živi u kontejneru, ili kada je baza tuđa briga.

Jedna napomena unapred, jer određuje ton celog modula: **nijedan alat iz ovog modula ne menja principe iz prethodnih.** Kontejner i dalje mora da ima ispravno dimenzionisan buffer pool. Managed baza i dalje traži da testirate restore. Ansible i dalje ne zna koje privilegije aplikacija treba.

Ono što se menja je količina ručnog rada i broj mesta gde možete pogrešiti.

---

## 52. MySQL u kontejneru

### Kada ovo raditi, a kada ne

Krenimo od iskrenog razgovora, jer se ovo pitanje retko postavlja.

**Kontejner ima smisla:**

- za razvojna i test okruženja — podizanje baze za trideset sekundi, brisanje bez traga,
- kada već imate orkestraciju i sve ostalo radi tako,
- kada podižete više instanci na istoj mašini radi izolacije,
- u CI cevovodima, gde svaka izgradnja dobija čistu bazu.

**Kontejner ima manje smisla:**

- za jednu produkcijsku bazu na namenskom serveru. Dobijate malo, a dodajete sloj koji može da otkaže i koji morate razumeti.
- kada tim ne poznaje kontejnere dobro. Dijagnostika problema u tri ujutru je teža kroz dodatni sloj.
- kada je opterećenje I/O-intenzivno i neko slučajno ostavi podatke na overlay fajl sistemu umesto na volumenu.

**Za Kubernetes:** ne pišite `StatefulSet` ručno. Koristite operator (Percona Operator za MySQL, Oracleov MySQL Operator, ili Vitess za veće razmere). Baza u Kubernetesu bez operatora je posao koji ćete raditi pogrešno, ma koliko se trudili.

### Osnovno pokretanje

```bash
docker run -d \
  --name mysql-dev \
  -e MYSQL_ROOT_PASSWORD='...' \
  -e MYSQL_DATABASE=shop \
  -p 127.0.0.1:3306:3306 \
  -v mysql-data:/var/lib/mysql \
  mysql:8.4
```

Dve stvari u ovoj komandi su obavezne i obe su podsetnici iz ranijih modula.

**`-p 127.0.0.1:3306:3306`** — bez prefiksa `127.0.0.1`, Docker objavljuje port na svim interfejsima **i zaobilazi UFW** (Modul 4). Vaš `ufw status` pokazuje `deny`, a baza je otvorena ka internetu.

**`-v mysql-data:/var/lib/mysql`** — bez ovoga podaci žive unutar kontejnera i **nestaju kada ga obrišete.** Ovo je najčešći način da se izgubi baza u kontejnerskom okruženju.

### Trajanje podataka

Dve opcije:

```bash
# imenovani volumen — Docker upravlja lokacijom
-v mysql-data:/var/lib/mysql

# bind mount — vi birate putanju
-v /data/mysql:/var/lib/mysql
```

Imenovani volumen je jednostavniji i bolje se ponaša sa pravima. Bind mount daje kontrolu nad lokacijom (npr. zaseban disk, Modul 2), ali uvodi problem vlasništva:

```bash
sudo mkdir -p /data/mysql
sudo chown -R 999:999 /data/mysql
```

Korisnik `mysql` unutar zvanične slike ima UID 999. Ako se to ne poklopi, kontejner ne startuje uz `Permission denied` — isti simptom kao u Modulu 2, drugi uzrok.

### Konfiguracija

Ne pravite sopstvenu sliku samo da biste promenili `my.cnf`. Montirajte fajl:

```bash
docker run -d \
  -v ./my-custom.cnf:/etc/mysql/conf.d/99-custom.cnf:ro \
  ...
```

```ini
# my-custom.cnf
[mysqld]
innodb_buffer_pool_size = 2G
max_connections = 200
slow_query_log = ON
long_query_time = 1
log_timestamps = SYSTEM
```

### Ograničenje memorije — obavezno

```bash
docker run -d -m 4g ...
```

I sada ono najvažnije, podsetnik iz Modula 7: **MySQL unutar kontejnera vidi RAM hosta, ne ograničenje kontejnera.** Ako server ima 64 GB, a kontejner ograničenje od 4 GB, i vi podesite buffer pool "na 60% RAM-a", dobićete 38 GB — i kontejner će biti ubijen čim MySQL pokuša da ga zauzme.

Pravilo: **buffer pool se računa prema ograničenju kontejnera.** Za 4 GB ograničenja, buffer pool oko 2 GB.

Provera unutar kontejnera:

```bash
docker exec mysql-dev cat /sys/fs/cgroup/memory.max
```

### Vreme gašenja — zamka koja pravi crash recovery pri svakom restartu

`docker stop` šalje `SIGTERM` i čeka **10 sekundi**, pa šalje `SIGKILL`.

Deset sekundi je premalo za MySQL. Podsetnik iz Modula 2: uredno gašenje podrazumeva pražnjenje prljavih stranica iz buffer pool-a, što na većoj bazi traje minutima.

Posledica: svaki `docker stop` je zapravo `kill -9`, a svaki naredni start radi crash recovery.

```bash
docker run -d --stop-timeout 300 ...
```

U `docker-compose.yml`:

```yaml
stop_grace_period: 5m
```

**Ovo je jedno od najvažnijih podešavanja za MySQL u kontejneru i skoro nikad se ne pominje.**

### Lozinke

Promenljive okruženja su vidljive:

```bash
docker inspect mysql-dev | grep MYSQL_ROOT_PASSWORD
sudo cat /proc/<pid>/environ | tr '\0' '\n'
```

Bolji način je kroz fajl:

```bash
echo -n 'duga-nasumicna-lozinka' | docker secret create mysql_root_pw -
```

```yaml
services:
  mysql:
    image: mysql:8.4
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/mysql_root_pw
    secrets:
      - mysql_root_pw
```

Sve zvanične slike podržavaju `_FILE` sufiks za osetljive promenljive.

### Kompletan `docker-compose.yml`

```yaml
services:
  mysql:
    image: mysql:8.4
    container_name: mysql
    restart: unless-stopped

    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/mysql_root_pw
      MYSQL_DATABASE: shop
      MYSQL_USER: app_shop
      MYSQL_PASSWORD_FILE: /run/secrets/mysql_app_pw

    secrets:
      - mysql_root_pw
      - mysql_app_pw

    ports:
      - "127.0.0.1:3306:3306"

    volumes:
      - mysql-data:/var/lib/mysql
      - ./config/99-custom.cnf:/etc/mysql/conf.d/99-custom.cnf:ro
      - ./init:/docker-entrypoint-initdb.d:ro

    deploy:
      resources:
        limits:
          memory: 4G

    stop_grace_period: 5m

    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 120s

    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "5"

volumes:
  mysql-data:

secrets:
  mysql_root_pw:
    file: ./secrets/root_pw.txt
  mysql_app_pw:
    file: ./secrets/app_pw.txt
```

Napomena o `start_period: 120s` u healthcheck-u: ako je kraći, orkestrator će smatrati kontejner nezdravim tokom crash recovery-ja i restartovati ga — u petlji.

Napomena o `logging`: bez ograničenja veličine, Docker log fajl raste neograničeno i puni disk (Modul 10).

### Skripte za inicijalizaciju

Fajlovi u `/docker-entrypoint-initdb.d/` izvršavaju se **samo pri prvoj inicijalizaciji**, kada je `datadir` prazan. Ne izvršavaju se pri narednim startovima.

```sql
-- init/01-users.sql
CREATE USER 'report'@'%' IDENTIFIED BY 'lozinka';
GRANT SELECT ON shop.* TO 'report'@'%';
```

Korisno za razvojna okruženja; za produkciju koristite Ansible (sledeće poglavlje).

### Backup iz kontejnera

Najjednostavnije je pokrenuti backup **sa hosta**, protiv objavljenog porta:

```bash
mysqldump -h 127.0.0.1 -P 3306 \
  --defaults-file=/etc/mysql/backup.cnf \
  --single-transaction --source-data=2 --routines --events \
  --all-databases | gzip > /backup/full-$(date +%F).sql.gz
```

Time koristite ceo skript iz Modula 5, bez izmena.

Alternativa kroz `exec`:

```bash
docker exec mysql sh -c \
  'mysqldump --defaults-file=/etc/mysql/backup.cnf --single-transaction --all-databases' \
  | gzip > /backup/full-$(date +%F).sql.gz
```

Za fizički backup volumen mora biti zaustavljen ili snimljen — ista pravila kao u Modulu 5.

### Podman

Podman radi sa istim slikama, uz dve razlike koje su bitne.

**Rootless režim.** Kontejneri rade pod vašim korisnikom, bez root privilegija. Bezbednije, ali komplikuje vlasništvo nad bind mount-ovima kroz preslikavanje UID-ova:

```bash
podman unshare chown -R 999:999 /data/mysql
```

**Integracija sa systemd-om.** Ovo je prava prednost Podmana za sysadmina — kontejner postaje običan systemd servis.

Moderni način je Quadlet:

```ini
# ~/.config/containers/systemd/mysql.container
[Unit]
Description=MySQL u kontejneru
After=network-online.target

[Container]
Image=docker.io/library/mysql:8.4
ContainerName=mysql
PublishPort=127.0.0.1:3306:3306
Volume=mysql-data:/var/lib/mysql:Z
Volume=%h/mysql/99-custom.cnf:/etc/mysql/conf.d/99-custom.cnf:ro,Z
Environment=MYSQL_ROOT_PASSWORD_FILE=/run/secrets/root_pw
Secret=mysql_root_pw,type=mount,target=/run/secrets/root_pw
Memory=4g

[Service]
Restart=always
TimeoutStopSec=300

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user start mysql
```

Sada vam rade sve komande iz Modula 2 i 6: `systemctl status`, `journalctl -u`, `systemctl edit`. To je opipljiva prednost nad Dockerom za administratora koji misli u systemd terminima.

Uočite `TimeoutStopSec=300` — isto ono podešavanje o kome smo pričali, samo drugim imenom.

---

## 53. Ansible role za MySQL

### Zašto

Kada imate jedan server, ručno podešavanje je brže. Kada imate pet, Ansible se isplati iz tri razloga:

1. **Ponovljivost** — novi server izgleda tačno kao stari.
2. **Dokumentacija** — role je zapis šta je gde podešeno i zašto. Bolji od wiki stranice, jer ne može da zastari a da to ne primetite.
3. **Revizija** — Git istorija pokazuje ko je šta promenio i kada.

### Priprema

```bash
ansible-galaxy collection install community.mysql
```

Na ciljnim serverima treba `PyMySQL`:

```yaml
- name: Python biblioteka za MySQL module
  ansible.builtin.apt:
    name: python3-pymysql
    state: present
```

### Prijava bez lozinke

Najkorisniji detalj u celom poglavlju: na Ubuntuu se Ansible može prijaviti preko Unix socket-a kao root, koristeći `auth_socket` (Modul 3):

```yaml
- name: Kreiranje aplikacionog naloga
  community.mysql.mysql_user:
    login_unix_socket: /run/mysqld/mysqld.sock
    name: app_shop
    host: "10.0.1.%"
    password: "{{ mysql_app_password }}"
    priv: "shop.*:SELECT,INSERT,UPDATE,DELETE"
    state: present
```

**Nema lozinke za administratora nigde u kodu.** Ansible se povezuje kao root nad socketom, jer se izvršava kao root na tom serveru.

### Struktura role

```
roles/mysql/
├── defaults/main.yml
├── tasks/
│   ├── main.yml
│   ├── install.yml
│   ├── config.yml
│   ├── users.yml
│   └── backup.yml
├── handlers/main.yml
├── templates/
│   ├── 99-custom.cnf.j2
│   ├── backup.cnf.j2
│   ├── mysql-backup.service.j2
│   └── mysql-backup.timer.j2
└── files/
    └── mysql-backup.sh
```

### `defaults/main.yml`

```yaml
---
mysql_version: "8.4"

# Memorija — računa se iz RAM-a mašine (Modul 7)
mysql_buffer_pool_pct: 60
mysql_buffer_pool_size: "{{ (ansible_memtotal_mb * mysql_buffer_pool_pct / 100) | int }}M"

mysql_max_connections: 200
mysql_bind_address: "127.0.0.1"

# Logovi (Modul 6)
mysql_slow_query_log: "ON"
mysql_long_query_time: 1
mysql_log_timestamps: "SYSTEM"

# Binlog i replikacija (Moduli 5, 8)
mysql_server_id: 1
mysql_log_bin: true
mysql_binlog_expire_seconds: 604800
mysql_gtid_mode: "ON"

# Izdržljivost (Modul 7)
mysql_sync_binlog: 1
mysql_flush_log_at_trx_commit: 1

# Administrativni port (Modul 7)
mysql_admin_port: 33062

# Fajl deskriptori (Modul 7)
mysql_limit_nofile: 65535

mysql_backup_dir: /backup/mysql
mysql_backup_retention_days: 14
mysql_backup_hour: "02:00:00"

mysql_databases: []
mysql_users: []
```

### `tasks/main.yml`

```yaml
---
- import_tasks: install.yml
  tags: [mysql, mysql-install]

- import_tasks: config.yml
  tags: [mysql, mysql-config]

- import_tasks: users.yml
  tags: [mysql, mysql-users]

- import_tasks: backup.yml
  tags: [mysql, mysql-backup]
```

### `tasks/install.yml`

```yaml
---
- name: Instalacija paketa
  ansible.builtin.apt:
    name:
      - mysql-server
      - python3-pymysql
      - percona-toolkit
    state: present
    update_cache: true

- name: Direktorijum za systemd override
  ansible.builtin.file:
    path: /etc/systemd/system/mysql.service.d
    state: directory
    mode: "0755"

- name: Podizanje limita fajl deskriptora
  ansible.builtin.copy:
    dest: /etc/systemd/system/mysql.service.d/override.conf
    content: |
      [Service]
      LimitNOFILE={{ mysql_limit_nofile }}
      OOMScoreAdjust=-600
    mode: "0644"
  notify: reload systemd

- name: Servis je uključen i pokrenut
  ansible.builtin.systemd:
    name: mysql
    enabled: true
    state: started
```

### `templates/99-custom.cnf.j2`

```jinja
# UPRAVLJANO ANSIBLE-OM — ručne izmene će biti prepisane
# Role: mysql | Host: {{ inventory_hostname }}

[mysqld]
server_id = {{ mysql_server_id }}
bind-address = {{ mysql_bind_address }}

# Memorija: {{ mysql_buffer_pool_pct }}% od {{ ansible_memtotal_mb }} MB RAM-a
innodb_buffer_pool_size = {{ mysql_buffer_pool_size }}
max_connections = {{ mysql_max_connections }}

# Administrativni ulaz kada je limit konekcija iscrpljen
admin_address = 127.0.0.1
admin_port = {{ mysql_admin_port }}
create_admin_listener_thread = ON

# Logovi
log_timestamps = {{ mysql_log_timestamps }}
slow_query_log = {{ mysql_slow_query_log }}
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = {{ mysql_long_query_time }}
log_slow_extra = ON

{% if mysql_log_bin %}
# Binarni log
log_bin = /var/lib/mysql/binlog
binlog_format = ROW
binlog_expire_logs_seconds = {{ mysql_binlog_expire_seconds }}
gtid_mode = {{ mysql_gtid_mode }}
enforce_gtid_consistency = ON
sync_binlog = {{ mysql_sync_binlog }}
{% endif %}

innodb_flush_log_at_trx_commit = {{ mysql_flush_log_at_trx_commit }}
innodb_flush_method = O_DIRECT

# Mreža
skip_name_resolve = ON

# X Protocol nije u upotrebi
mysqlx = 0
```

Prvi red je važan. Za šest meseci neko će otvoriti ovaj fajl na serveru i tražiti gde da promeni parametar — komentar ga upućuje na pravo mesto.

### `tasks/config.yml` i pitanje restarta

```yaml
---
- name: Konfiguracija servera
  ansible.builtin.template:
    src: 99-custom.cnf.j2
    dest: /etc/mysql/mysql.conf.d/99-custom.cnf
    owner: root
    group: root
    mode: "0644"
    validate: "mysqld --validate-config --defaults-file=%s"
  notify: restart mysql

- name: Logrotate konfiguracija
  ansible.builtin.template:
    src: logrotate-mysql.j2
    dest: /etc/logrotate.d/mysql-server
    mode: "0644"
```

Opcija `validate` je važna: Ansible proverava konfiguraciju **pre** nego što je postavi. Neispravan fajl nikada ne stigne na server.

**Sada ono najvažnije u celoj roli.** Handler:

```yaml
# handlers/main.yml
---
- name: reload systemd
  ansible.builtin.systemd:
    daemon_reload: true

- name: restart mysql
  ansible.builtin.systemd:
    name: mysql
    state: restarted
  when: mysql_allow_restart | default(false) | bool
```

Uslov `when` znači da se restart **neće** desiti osim ako ga eksplicitno ne tražite:

```bash
ansible-playbook site.yml --limit db01 -e mysql_allow_restart=true
```

**Zašto ovako:** automatski restart produkcijske baze zbog izmene komentara u konfiguraciji je scenario koji ne želite. Ansible će primeniti izmenu i reći vam da je restart potreban; vi birate kada.

Ovo je razlika između role napisane za razvojno okruženje i role napisane za produkciju.

### `tasks/users.yml`

```yaml
---
- name: Baze podataka
  community.mysql.mysql_db:
    login_unix_socket: /run/mysqld/mysqld.sock
    name: "{{ item.name }}"
    encoding: "{{ item.encoding | default('utf8mb4') }}"
    collation: "{{ item.collation | default('utf8mb4_0900_ai_ci') }}"
    state: present
  loop: "{{ mysql_databases }}"

- name: Korisnički nalozi
  community.mysql.mysql_user:
    login_unix_socket: /run/mysqld/mysqld.sock
    name: "{{ item.name }}"
    host: "{{ item.host }}"
    password: "{{ item.password }}"
    priv: "{{ item.priv }}"
    state: present
  loop: "{{ mysql_users }}"
  no_log: true

- name: Backup nalog preko auth_socket, bez lozinke
  community.mysql.mysql_query:
    login_unix_socket: /run/mysqld/mysqld.sock
    query:
      - "CREATE USER IF NOT EXISTS 'backup'@'localhost' IDENTIFIED WITH auth_socket AS 'root'"
      - >
        GRANT SELECT, SHOW VIEW, TRIGGER, EVENT, LOCK TABLES,
        RELOAD, PROCESS, REPLICATION CLIENT ON *.* TO 'backup'@'localhost'
```

`no_log: true` sprečava da se lozinke pojave u izlazu Ansible-a i u CI logovima. Bez toga ih ispisujete u svaki log koji neko može pročitati.

### Podaci o hostu

```yaml
# host_vars/db01.yml
mysql_server_id: 1
mysql_bind_address: "10.0.1.10"
mysql_buffer_pool_pct: 65

mysql_databases:
  - name: shop

mysql_users:
  - name: app_shop
    host: "10.0.1.%"
    password: "{{ vault_app_shop_password }}"
    priv: "shop.*:SELECT,INSERT,UPDATE,DELETE"

  - name: migrate_shop
    host: "10.0.1.%"
    password: "{{ vault_migrate_password }}"
    priv: "shop.*:SELECT,INSERT,UPDATE,DELETE,CREATE,DROP,ALTER,INDEX"

  - name: exporter
    host: "localhost"
    password: "{{ vault_exporter_password }}"
    priv: "*.*:PROCESS,REPLICATION CLIENT/performance_schema.*:SELECT"
```

Ovo je direktno preslikavanje recepata iz Modula 3 u kod.

### Tajne kroz Ansible Vault

```bash
ansible-vault encrypt_string 'duga-nasumicna-lozinka' --name 'vault_app_shop_password'
```

```yaml
# group_vars/db/vault.yml
vault_app_shop_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  38623763...
```

Ovo sme u Git. Ključ za dešifrovanje ne sme.

### Provera pre primene

```bash
ansible-playbook site.yml --limit db01 --check --diff
```

`--check` ne menja ništa, `--diff` pokazuje šta bi se promenilo. **Na produkciji uvek prvo ovako.**

---

## 54. Lozinke u skriptama

### Šta se nikada ne radi

```bash
# NE — vidljivo u ps i u ~/.bash_history
mysql -u backup -pTajnaLozinka -e "SELECT 1;"
```

```bash
# Dokaz, sa druge sesije dok komanda traje:
ps aux | grep mysql
```

Lozinka je vidljiva **svakom korisniku na sistemu**. Ovo nije teorijski problem.

```bash
# NE — promenljiva okruženja
export MYSQL_PWD='TajnaLozinka'
```

Vidljivo kroz `/proc/<pid>/environ` i u `docker inspect`.

### Rangirano po bezbednosti

**1. `auth_socket` — najbolje**

Nema lozinke. Nema ničega što se može ukrasti.

```sql
CREATE USER 'backup'@'localhost' IDENTIFIED WITH auth_socket AS 'backup';
```

```bash
sudo -u backup mysqldump --all-databases > backup.sql
```

Radi samo lokalno i samo kroz socket. Za backup, monitoring i skripte na samom serveru — koristite ovo.

**2. Zaseban `defaults-file` sa pravima 600**

Za sve što `auth_socket` ne pokriva:

```bash
sudo tee /etc/mysql/backup.cnf > /dev/null <<'EOF'
[client]
user = backup
password = duga-nasumicna-lozinka
host = 10.0.1.10
EOF

sudo chown root:root /etc/mysql/backup.cnf
sudo chmod 600 /etc/mysql/backup.cnf
```

```bash
mysqldump --defaults-file=/etc/mysql/backup.cnf --all-databases
```

**Koristite `--defaults-file` sa punom putanjom, ne `~/.my.cnf`.** Time je jasno koji fajl se koristi, a skript radi isto bez obzira ko ga pokreće.

Napomena: `--defaults-file` mora biti **prvi argument** komande. Ako ga stavite kasnije, biće ignorisan bez upozorenja.

**3. `mysql_config_editor` — sa iskrenom ogradom**

```bash
mysql_config_editor set --login-path=backup \
  --host=10.0.1.10 --user=backup --password
```

```bash
mysql --login-path=backup
mysqldump --login-path=backup --all-databases
```

Fajl `~/.mylogin.cnf` je **obfuskovan, a ne šifrovan.** Ključ je fiksan i ugrađen u alat, pa se sadržaj može povratiti:

```bash
my_print_defaults -s backup
```

Dakle: štiti od slučajnog pogleda preko ramena i od `grep`-a kroz skripte. **Ne štiti od nekoga ko ima pristup fajlu.** Nemojte ga tretirati kao bezbednosnu meru; tretirajte ga kao urednost.

**4. systemd `LoadCredential`**

Za servise, moderna opcija:

```ini
[Service]
LoadCredential=dbpass:/etc/mysql/backup.pass
ExecStart=/usr/local/bin/backup.sh
```

Fajl je dostupan skriptu preko `$CREDENTIALS_DIRECTORY`, sa pravima ograničenim na taj servis, i nije vidljiv drugim procesima.

**5. Menadžer tajni**

HashiCorp Vault, AWS Secrets Manager, Azure Key Vault. Ima smisla kada već imate takvu infrastrukturu. Za jedan server je preterivanje.

### Rotacija bez ispada

Podsetnik iz Modula 3, jer je ovo mesto gde se primenjuje:

```sql
ALTER USER 'app'@'10.0.1.%' IDENTIFIED BY 'nova' RETAIN CURRENT PASSWORD;
-- ažurirate konfiguraciju i restartujete aplikaciju
ALTER USER 'app'@'10.0.1.%' DISCARD OLD PASSWORD;
```

### Revizija

```bash
# Lozinke u skriptama
sudo grep -rIl --exclude-dir=.git -E "\-p[A-Za-z0-9]|password\s*=" \
  /usr/local/bin /opt /etc/cron* 2>/dev/null

# Fajlovi sa lozinkama koji nisu 600
sudo find / -name "*.cnf" -perm /o+r 2>/dev/null | xargs grep -l password 2>/dev/null

# Istorija shell-a
grep -E "mysql.* -p[^ ]" ~/.bash_history
```

Ako poslednja komanda nešto vrati, očistite istoriju i promenite te lozinke.

Za Git repozitorijume postavite `pre-commit` hook ili alat poput `gitleaks`, jer lozinka jednom commit-ovana ostaje u istoriji zauvek — brisanje fajla je ne uklanja.

---

## 55. Managed baze

### Šta prestaje da bude vaš posao

Kod RDS-a, Cloud SQL-a, Azure Database for MySQL ili DigitalOcean Managed Database:

- zakrpe operativnog sistema,
- instalacija i osnovna nadogradnja MySQL-a,
- infrastruktura za backup (izrada, rotacija, skladištenje),
- postavljanje replikacije,
- automatsko preuzimanje posla pri otkazu,
- hardver, diskovi, mreža.

To je znatan deo Modula 1, 2, 5 i 8. Nije malo.

### Šta i dalje ostaje vaš posao

Ovaj spisak je razlog zbog kog ovaj kurs ima smisla i za nekoga ko koristi managed bazu.

**Provera backupa.** Provajder pravi backup. Provajder **ne proverava da li se iz njega može vratiti vaša aplikacija.** Vežba restore-a iz Modula 5 ostaje vaša obaveza, u punom obimu: vratite snapshot u zasebnu instancu, pustite aplikaciju na nju, izmerite RTO.

Ovo je najčešće zanemarena stavka kod managed baza, upravo zato što deluje kao da je rešena.

**Nalozi i privilegije.** Ceo Modul 3 važi nepromenjeno. Provajder vam daje jedan administratorski nalog; sve ostalo je na vama.

**Performanse upita i šeme.** Nijedan provajder neće dodati indeks umesto vas. Slow log postoji, `performance_schema` postoji, `sys` šema postoji. Modul 6 se primenjuje skoro u celini.

**Konfiguracija.** Postoji, samo se zove drugačije — parameter group, database flags, server parameters. `innodb_buffer_pool_size` je često unapred vezan za veličinu instance, ali `max_connections`, `long_query_time`, `binlog_expire_logs_seconds` i ostalo i dalje podešavate vi.

**Monitoring i alarmi.** Provajder prikuplja metrike. **Alarmi se ne podešavaju sami.** Spisak iz Modula 6 — dostupnost, disk, konekcije, replikacija, starost backupa — i dalje treba da neko postavi.

**Mrežna bezbednost.** VPC, security grupe, privatan pristup, TLS. Modul 4 u drugom obliku.

**Connection pooling.** Managed baza ima isti problem sa konekcijama kao i vaša. RDS Proxy ili pool u aplikaciji — vaša odluka.

**Troškovi.** Ovo postaje pravi posao. Managed baza je tipično dva do četiri puta skuplja od iste količine sirovog računarskog resursa. Neko mora da prati veličinu instance, retenciju backupa, saobraćaj i skladište.

### Šta gubite

**Nema pristupa fajl sistemu.** Nema `datadir`-a, nema `/var/log/mysql`, nema `lsof`. Ceo Modul 2 postaje neprimenljiv.

**Nema `SUPER` privilegije.** Neke operacije jednostavno nisu moguće.

**Nema XtraBackup-a.** Backup radi provajder, na svoj način. Izlaz iz platforme je logički dump — što na velikoj bazi znači dug postupak.

**Ograničen pristup binlogovima.** Neki provajderi dozvoljavaju `mysqlbinlog --read-from-remote-server`, neki ne. Proverite pre nego što se oslonite na PITR po svom scenariju.

**Termini održavanja koje ne birate.** Provajder će restartovati vašu instancu radi zakrpa. Možete birati prozor, ne i da li se dešava.

**Teža dijagnostika.** Nema `tail -f` nad error logom u realnom vremenu. Logovi se dobijaju kroz API ili konzolu, često sa kašnjenjem i ograničenom retencijom. Modul 10 se izvodi znatno teže.

**Vezanost za provajdera.** Migracija napolje je moguća, ali je posao — naročito ako ste koristili specifičnosti platforme.

### Kada birati šta

| Situacija | Preporuka |
|---|---|
| Mali tim, nema DBA, potrebna dostupnost | managed |
| Regulatorni zahtevi koje provajder pokriva | managed |
| Jedan sysadmin sa još petnaest servera | managed |
| Osetljivost na trošak pri većim razmerama | sopstveni server |
| Potrebna puna kontrola i specifična podešavanja | sopstveni server |
| Postoji ekspertiza i vreme za održavanje | sopstveni server |
| Podaci ne smeju napustiti sopstvenu infrastrukturu | sopstveni server |

Nema univerzalno tačnog odgovora. Ali postoji jedno korisno pitanje: **koliko sati mesečno trošite na održavanje baze, i koliko to košta u odnosu na razliku u ceni?** Kod jednog servera odgovor je često "sopstveni je jeftiniji". Kod potrebe za HA i dežurstvom — često nije.

### Migracija na managed platformu

Metoda iz Modula 9 radi i ovde, i bolja je od alata za migraciju koje provajderi nude:

```
1. Napravite managed instancu.
2. Napunite je početnim dumpom.
3. Podesite je kao REPLIKU vašeg postojećeg servera.
   Većina provajdera podržava replikaciju sa spoljašnjeg izvora.
4. Pustite da se sinhronizuje i radi nekoliko dana.
5. U dogovorenom terminu: zaustavite upise, sačekajte sinhronizaciju,
   prekinite replikaciju, preusmerite aplikaciju.
6. Zadržite stari server kao rezervu bar nedelju dana.
```

Zastoj se meri minutima, a imate put nazad.

Pre migracije prođite kroz kontrolnu listu iz Modula 9 — naročito naloge, host polja (koja se menjaju!), kolacije i TLS.

---

## Kontrolna lista na kraju modula

```bash
# KONTEJNERI
# 1. Podaci su na volumenu, ne u kontejneru
docker inspect mysql | grep -A5 Mounts

# 2. Port je vezan za 127.0.0.1 (Docker zaobilazi UFW)
docker port mysql

# 3. Vreme gašenja je dovoljno dugo
docker inspect mysql | grep -i stoptimeout

# 4. Buffer pool je prema ograničenju kontejnera, ne prema RAM-u hosta

# ANSIBLE
# 5. Role prolazi --check bez grešaka
ansible-playbook site.yml --check --diff

# 6. Restart baze NIJE automatski
grep -A3 "restart mysql" roles/mysql/handlers/main.yml

# 7. Lozinke su u Vault-u, no_log je postavljen

# LOZINKE
# 8. Nigde nema -p u komandnoj liniji
sudo grep -rIl -E "\-p[A-Za-z0-9]" /usr/local/bin /etc/cron* 2>/dev/null

# 9. Svi .cnf fajlovi sa lozinkama imaju prava 600
sudo find /etc/mysql -name "*.cnf" -exec ls -l {} \;

# MANAGED
# 10. Restore iz snapshot-a je testiran i RTO izmeren — i ovde
```

---

## Vežbe

**Vežba 1 — Kontejner bez volumena**
Pokrenite MySQL kontejner bez `-v`, unesite podatke, pa obrišite kontejner sa `docker rm`. Objasnite šta se desilo. Zatim ponovite sa imenovanim volumenom i potvrdite da podaci preživljavaju.

**Vežba 2 — Docker i UFW**
Uključite UFW sa `default deny incoming`. Pokrenite kontejner sa `-p 3306:3306` i sa druge mašine proverite dostupnost porta. Objasnite rezultat kroz `iptables -L DOCKER -n`, pa popravite.

**Vežba 3 — Vreme gašenja**
Napunite bazu u kontejneru sa nekoliko gigabajta podataka i intenzivnim upisom. Izvršite `docker stop` sa podrazumevanim tajmautom i pročitajte error log pri sledećem startu. Zatim postavite `--stop-timeout 300` i ponovite. Uporedite trajanje starta.

**Vežba 4 — Buffer pool u kontejneru**
Na mašini sa 16 GB RAM-a pokrenite kontejner sa `-m 2g` i postavite buffer pool na 10 GB. Posmatrajte šta se dešava. Objasnite zašto MySQL nije "video" ograničenje.

**Vežba 5 — Ansible role od nule**
Napišite minimalnu role koja instalira MySQL, postavlja konfiguraciju iz šablona sa `validate`, i kreira jedan aplikacioni nalog preko `login_unix_socket`. Pokrenite je dvaput i potvrdite da je drugi prolaz idempotentan.

**Vežba 6 — Restart koji se nije desio**
Podesite handler sa uslovom `mysql_allow_restart`. Promenite parametar koji zahteva restart i pokrenite playbook bez zastavice. Potvrdite da je fajl promenjen a servis nije restartovan, i da `SHOW VARIABLES` i dalje pokazuje staru vrednost. Objasnite zašto je ovo poželjno ponašanje na produkciji.

**Vežba 7 — Lozinka u `ps`**
Pokrenite dug `mysqldump` sa lozinkom u komandnoj liniji. Sa druge sesije, kao **neprivilegovan korisnik**, pronađite lozinku pomoću `ps aux`. Zatim ponovite sa `--defaults-file` i potvrdite da je nema.

**Vežba 8 — `mysql_config_editor` nije šifrovanje**
Postavite login path sa lozinkom, pa je povratite pomoću `my_print_defaults -s`. Zaključite čemu ovaj alat zaista služi.

**Vežba 9 — Managed baza, ono što ostaje**
Ako imate pristup nekoj managed platformi (većina ima besplatan probni period), podignite instancu i pokušajte: pročitati error log u realnom vremenu, pokrenuti XtraBackup, promeniti `innodb_buffer_pool_size`, videti `datadir`. Zabeležite šta radi, šta ne, i šta biste morali da uradite drugačije. Zatim uradite vežbu restore-a iz snapshot-a i izmerite RTO.

---

## Kraj glavnog dela kursa

Ovim je završen sadržajni deo. Ostaje još **Dodatak** sa minimumom SQL-a koji vam je zaista potreban — deset naredbi koje ćete stvarno koristiti, kako pročitati šemu baze koju niste vi pravili, i rečnik pojmova za razgovor sa developerima.

Pre nego što pređete na njega, vredi se osvrnuti na to gde ste.

Ako ste prošli kroz sve module i uradili vežbe, sada umete da:

- postavite MySQL server od nule i razumete svaki fajl koji je pri tome nastao,
- objasnite zašto server ne startuje, u pet koraka i bez guglanja,
- dodelite tačno one privilegije koje su potrebne, i obrazložite zašto ne više,
- zatvorite bazu prema mreži u šest slojeva,
- napravite backup iz koga ste **stvarno vratili podatke** i znate koliko to traje,
- vratite bazu na trenutak pre nesreće,
- pronađete upit koji obara server i predate ga sa dokazom,
- dimenzionišete memoriju umesto da prepisujete tuđe brojke,
- postavite repliku i preuzmete posao kada primarni otkaže,
- promenite šemu velike tabele bez ispada,
- oporavite se od punog diska, oštećenja i OOM killer-a,
- i, možda najvažnije, **dokažete brojkama kada problem uopšte nije u bazi.**

To nije znanje DBA. To je znanje sysadmina koji ne mora da zove DBA — a to je bio cilj.
