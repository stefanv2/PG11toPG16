
# PostgreSQL Upgrade van 15 → 16

## 1. Backup maken (als `postgres`)
```bash
pg_dumpall > /mnt/datadomain/pg_dump/pg001t_backup_before_upgrade_16mei2025.sql
vi /mnt/datadomain/pg_dump/pg0011t_backup_before_upgrade_16mei2025.sql
```

Controleer huidige versie:
```bash
psql --version
# psql (PostgreSQL) 15.12 (PostgresPURE-10.12 Splendid Data)
```

## 2. PostgreSQL 16 installeren (als `oracle`)
```bash
sudo dnf -qy module disable postgresql
sudo dnf -qy module disable postgresql-15
sudo dnf install postgres-16-server postgres-16-client postgres-16-contrib postgres-16-pgaudit
```

## 3. Nieuwe datadirectory voorbereiden (als `postgres`)
```bash
mkdir -p /u01/app/postgres/data_v16
/usr/pgpure/postgres/16/bin/initdb -D /u01/app/postgres/data_v16
```

## 4. PostgreSQL 15 stoppen (als `oracle`)
```bash
sudo systemctl status postgres
sudo systemctl stop postgres
```

## 5. Upgrade voorbereiden en checken (als `postgres`)
```bash
/usr/pgpure/postgres/16/bin/pg_upgrade \
  --old-datadir=/u01/app/postgres/data \
  --new-datadir=/u01/app/postgres/data_v16 \
  --old-bindir=/usr/pgpure/postgres/15/bin \
  --new-bindir=/usr/pgpure/postgres/16/bin \
  --check
```

### Aanpassingen:
- Kopieer `postgresql.conf` van de oude naar de nieuwe folder.
- Pas daarin aan: `data_directory`, `hba_file`, `ident_file`.

Voer de check opnieuw uit tot er geen errors meer zijn.

## 6. Upgrade uitvoeren (als `postgres`)
```bash
/usr/pgpure/postgres/16/bin/pg_upgrade \
  --old-datadir=/u01/app/postgres/data \
  --new-datadir=/u01/app/postgres/data_v16 \
  --old-bindir=/usr/pgpure/postgres/15/bin \
  --new-bindir=/usr/pgpure/postgres/16/bin
```

Start de nieuwe cluster:
```bash
/usr/pgpure/postgres/16/bin/pg_ctl -D /u01/app/postgres/data_v16 -l logfile start
/usr/pgpure/postgres/16/bin/pg_ctl -D /u01/app/postgres/data_v16 -l logfile status
ps -ef | grep data_v16
```

Controleer in de database:
```bash
psql --version
# PostgreSQL 16.8 (PostgresPURE-11.8 Splendid Data)
psql
select version();
```

## 7. Extensies bijwerken
```sql
\i update_extensions.sql
CREATE EXTENSION pgaudit; -- foutmelding mogelijk (shared_preload_libraries nodig)
SELECT * FROM pg_available_extension_versions WHERE name = 'pgaudit';
CREATE EXTENSION pgaudit;
\dx
```

### Verwachte output:
```
Name    | Version | Schema     | Description
--------+---------+------------+---------------------------------
pgaudit | 16.0    | public     | provides auditing functionality
plpgsql | 1.0     | pg_catalog | PL/pgSQL procedural language
```

## 8. Oude PostgreSQL 15 verwijderen (als `oracle`)
```bash
sudo dnf erase postgres-15-server postgres-15-client postgres-15-contrib postgres-15-pgaudit postgres-15-libs
```

## 9. Migratie afronden (als `postgres`)
```bash
/usr/pgpure/postgres/16/bin/pg_ctl -D /u01/app/postgres/data_v16 -l logfile stop
rm -rf data
mv data_v16 data
```

Pas in `postgresql.conf` de paden aan (`data_directory`, `hba_file`, `ident_file`):
```bash
/usr/pgpure/postgres/16/bin/pg_ctl -D /u01/app/postgres/data -l logfile start
```

Update `.bash_profile`:
```bash
vi ~/.bash_profile
export PATH=/usr/pgpure/postgres/16/bin:$PATH
```

## 10. Systemd service inrichten (als `root`)
Maak nieuwe unit:
```bash
vi /etc/systemd/system/postgres.service
```

### Inhoud:
```ini
[Unit]
Description=PostgreSQL 16 database server
Documentation=man:postgres(1)
After=network.target

[Service]
Type=forking
User=postgres
Group=postgres
ExecStart=/usr/pgpure/postgres/16/bin/pg_ctl start -D /u01/app/postgres/data -l /u01/app/postgres/data/pg_log/postgresql.log
ExecStop=/usr/pgpure/postgres/16/bin/pg_ctl stop -D /u01/app/postgres/data -m fast
ExecReload=/usr/pgpure/postgres/16/bin/pg_ctl reload -D /u01/app/postgres/data
TimeoutSec=300
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Zet rechten en activeer:
```bash
sudo chmod 755 /etc/systemd/system/postgres.service
sudo systemctl daemon-reload
sudo systemctl start postgres
sudo systemctl enable postgres
```

## 11. Logging directory (als `postgres`)
```bash
mkdir -p /u01/app/postgres/data/pg_log
```

## 12. `pg_hba.conf` aanpassen
Voeg regels toe:
```
host    all   all   *   trust   # BARMAN server
host    all   all   */32    md5     # BEHEER server
host    all   all   */32   md5     # BEHEER server
host    all   all   */32   md5     # BEHEER server
host    all   all   */32     trust   # OEM13C
```

Herlaad config:
```sql
SELECT pg_reload_conf();
```

## 13. `postgresql.conf` basisinstellingen

Voorbeeldinstellingen:
```conf
port = 5432
log_directory = 'pg_log'
log_filename = 'postgresql-%Y%m%d_%H%M%S.log'
log_truncate_on_rotation = on
log_rotation_age = 1d
log_line_prefix = '%a %u %r %d %p %t %c '
logging_collector = on
log_connections = on
log_disconnections = on
log_statement = 'ddl'
```

## 14. Barman configuratie

Aanpassen in `$PGDATA/postgresql.conf`:
```conf
listen_addresses = '*'
wal_level = archive
archive_mode = on
archive_command = 'rsync -a %p barman@1.4.1.1:/mnt/datadomain/barman/<servernaam-dbnaam>/incoming/%f'
```

## 15. Test backup met Barman

Op Barman server:
```bash
barman check sv0001-pg01t
```

### Bij foutmelding:
```bash
mv /mnt/datadomain/barman/sv0001-pg01t/identity.json /mnt/datadomain/TEMP
barman backup sv0001-pg01t
```

## 16. Stats opnieuw opbouwen (als `postgres`)
```bash
/usr/pgpure/postgres/16/bin/vacuumdb --all --analyze-in-stages
```
