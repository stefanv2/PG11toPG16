# PostgreSQL 11 naar 16 Upgrade README

Dit document beschrijft de uitgevoerde stappen om een PostgreSQL 11.22 database (PostgresPURE) succesvol te restoren en upgraden naar PostgreSQL 16 op een nieuwe omgeving. Het doel was om een productiesysteem (inclusief tablespaces) veilig over te zetten, te testen en klaar te maken voor productiegebruik.
---

<p align="center">
<img src="/v11tov16.png" alt="BTOP" width="400" height="600"/>  
</p>

---


---

## 🧱 Basisinformatie

- **Bronversie**: PostgreSQL 11.22 (PostgresPURE-6.23 Splendid Data)
- **Doelversie**: PostgreSQL 16.10 (PostgresPURE-11.10)
- **Platform**: Oracle Linux 8
- **Backup tool**: Barman
- **Restoremethode**: Remote `barman recover` via SSH met tablespaces
- **Upgrade methode**: `pg_upgrade` met `--link` optie

---

## 🔁 Stap 1 - Restore PostgreSQL 11 backup via Barman

1. **Remote restore gestart met aangepaste tablespace directories**:
   ```bash
   barman recover \
     --remote-ssh-command "ssh postgres@<target-host>" \
     --tablespace test1:/u01/app/postgres/data_pg11_test_test1 \
     --tablespace test2:/u01/app/postgres/data_pg11_test_test2 \
     --tablespace test3:/u01/app/postgres/data_pg11_test_resr3 \
     <server-name> \
     <backup-id> \
     /u01/app/postgres/data_pg11_test
   ```

2. **Controleren of files correct op locatie staan**
   - Timestamps van folders kunnen oud zijn: `ls -l` toont de `mtime` van bestanden binnen de tablespace folders.

3. **Start PostgreSQL 11 instance**:
   ```bash
   /usr/pgpure/postgres/11/bin/pg_ctl -D /u01/app/postgres/data_pg11_test -l logfile -o "-p 5544" start
   ```

4. **Verifiëren dat databases en tablespaces correct zijn geladen**:
   ```sql
   SELECT spcname, pg_tablespace_location(oid) FROM pg_tablespace;
   \l
   ```

---

## ⬆️ Stap 2 - Voorbereiden upgrade naar PostgreSQL 16

1. **Init PostgreSQL 16 datadir**:
   ```bash
   /usr/pgpure/postgres/16/bin/initdb -D /u01/app/postgres/data_pg16_test
   ```

2. **Controleer compatibiliteit met pg_upgrade**:
   ```bash
   /usr/pgpure/postgres/16/bin/pg_upgrade \
     --old-datadir=/u01/app/postgres/data_pg11_test \
     --new-datadir=/u01/app/postgres/data_pg16_test \
     --old-bindir=/usr/pgpure/postgres/11/bin \
     --new-bindir=/usr/pgpure/postgres/16/bin \
     --link \
     --check
   ```

3. **Fout opgelost**: ontbrekende `libssl.so.10` voor PostgreSQL 11 bin
   - Workaround:
     ```bash
     export LD_LIBRARY_PATH=/usr/lib64/compat:/usr/pgpure/postgres/11/lib
     ```

---

## 🚀 Stap 3 - Uitvoeren van de upgrade

1. **Voer upgrade uit**:
   ```bash
   /usr/pgpure/postgres/16/bin/pg_upgrade \
     --old-datadir=/u01/app/postgres/data_pg11_test \
     --new-datadir=/u01/app/postgres/data_pg16_test \
     --old-bindir=/usr/pgpure/postgres/11/bin \
     --new-bindir=/usr/pgpure/postgres/16/bin \
     --link
   ```

2. **Let op**: `--link` betekent dat oude bestanden hardlinked zijn. Niet beide clusters tegelijk draaien!

3. **Controleer post-upgrade output**:
   - Script `delete_old_cluster.sh` wordt gegenereerd
   - `update_extensions.sql` aanwezig voor het bijwerken van extensies
   - Aanbevolen:
     ```bash
     vacuumdb --all --analyze-in-stages
     ```

4. **Start PostgreSQL 16 instance**:
   ```bash
   /usr/pgpure/postgres/16/bin/pg_ctl -D /u01/app/postgres/data_pg16_test -l /u01/app/postgres/data_pg16_test/logfile start
   ```
   > Let op dat je poort aanpast indien nodig (bijv. `-o "-p 5544"`)

5. **Verifiëren databases en tablespaces**:
   ```sql
   SELECT spcname, pg_tablespace_location(oid) FROM pg_tablespace;
   \l
   ```

---

## 🧹 Optioneel: Opschonen oude cluster

- Na succesvolle test: oude 11-directory verwijderen
- **Controleer goed** dat de 16-instance draait voordat je onderstaande uitvoert:

```bash
rm -rf /u01/app/postgres/data_pg11_test
```

**Let op**: `delete_old_cluster.sh` verwijst mogelijk naar verkeerde pad. Handmatig checken vóór gebruik.

---

## ✅ Resultaat

- Database draait nu onder PostgreSQL 16
- Alle tablespaces correct overgenomen
- Databases en rechten zijn intact
- Klaar voor analyse, testing of promotie naar productie

---

Laatste controle gedaan met:
```sql
SELECT version();
SELECT spcname, pg_tablespace_location(oid) FROM pg_tablespace;
\l
```

---

🧠 *Document aangemaakt op basis van shell-logs en stappen doorlopen op 17–18 september 2025.*


