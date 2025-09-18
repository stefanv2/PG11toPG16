
# PostgreSQL 11 (PostgresPURE) software overzetten van OL7.9 naar OL8/9

We gebruiken deze methode om de PostgreSQL 11.22 software van een oudere OL7.9 server over te zetten naar een nieuwe OL8/OL9 omgeving, zodat we daar een Barman backup kunnen restoren en vervolgens kunnen upgraden naar PostgreSQL 16 via `pg_upgrade`.

---

## 📦 Op de oude OL7 server (bron)

### 1. Maak een tarball van de 11-software

```bash
sudo tar --xattrs --acls --numeric-owner -C /usr/pgpure/postgres -cpf /tmp/pgpure11.tar 11
```

### 2. (Optioneel) inhoud checken

```bash
sudo tar -tvf /tmp/pgpure11.tar | head
```

### 3. (Optioneel) checksum maken

```bash
sudo sha256sum /tmp/pgpure11.tar > /tmp/pgpure11.tar.sha256
```

### 4. Bestand beschikbaar maken voor overdracht

```bash
sudo chown oracle:oracle /tmp/pgpure11.tar /tmp/pgpure11.tar.sha256
```

### 5. Grootte en inhoud checken

```bash
du -sh /usr/pgpure/postgres/11
ls -lh /tmp/pgpure11.tar
tar -tvf /tmp/pgpure11.tar | egrep 'bin/postgres$|bin/pg_dump$|bin/pg_restore$|lib/libpq|share/extension' | sed -n '1,20p'
```

---

## 📥 Op de nieuwe OL8/OL9 server (doel)

Plaats de `pgpure11.tar` in `/tmp` en voer uit:

```bash
mkdir -p /usr/pgpure/postgres
cd /usr/pgpure/postgres
tar -xpf /tmp/pgpure11.tar
```

---

## 🧱 Compatibele libraries overzetten van OL7 → OL8/9

### 1. Maak op de OL7 server een bundel van de compat-libs

```bash
mkdir -p /opt/compat/pg11-libs
cd /usr/lib64/compat
tar -czf /opt/compat/pg11-libs/pg11-compat-libs.tar.gz ./*
```

### 2. Kopieer deze naar de OL8/9 server

```bash
scp /opt/compat/pg11-libs/pg11-compat-libs.tar.gz root@ol8-server:/tmp
```

### 3. Pak deze uit op de OL8/9 omgeving

```bash
mkdir -p /usr/lib64/compat
tar -xzf /tmp/pg11-compat-libs.tar.gz -C /usr/lib64/compat
```

**OF (met `--strip-components=1` als je 1 extra mapniveau hebt):**

```bash
tar -xzf /tmp/pg11-libs.tar.gz -C /usr/lib64/compat --strip-components=1
```

### 4. Exporteer je `LD_LIBRARY_PATH`

```bash
export LD_LIBRARY_PATH=/usr/lib64/compat:/usr/pgpure/postgres/11/lib:$LD_LIBRARY_PATH
```

---

## 🧪 Test of alles werkt

```bash
/usr/pgpure/postgres/11/bin/postgres --version
/usr/pgpure/postgres/11/bin/psql --version
```

### Gebruik `ldd` om afhankelijkheden te checken

```bash
ldd /usr/pgpure/postgres/11/bin/psql
ldd /usr/pgpure/postgres/11/bin/postgres
ldd /usr/pgpure/postgres/11/bin/pg_dump
ldd /usr/pgpure/postgres/11/bin/pg_restore
```

---

## 🧹 Eventueel: Verwijder system-libs uit compat pad

```bash
rm -f /usr/lib64/compat/libc.so.6       /usr/lib64/compat/libpthread.so.0       /usr/lib64/compat/libm.so.6       /usr/lib64/compat/libdl.so.2       /usr/lib64/compat/libstdc++.so.6       /usr/lib64/compat/libcrypt.so.1       /usr/lib64/compat/librt.so.1       /usr/lib64/compat/libgcc_s.so.1
```

---

## 💾 Extra: Maak een kopie van de compat libs op OL8/9

```bash
mkdir -p /opt/compat/pg11-libs
cd /usr/lib64/compat
tar -czf /opt/compat/pg11-libs/pg11-compat-libs.tar.gz ./*
chmod 600 /opt/compat/pg11-libs/pg11-compat-libs.tar.gz
chown root:root /opt/compat/pg11-libs/pg11-compat-libs.tar.gz
```

---

Laat me weten als dit in je volledige `README.md` moet worden geïntegreerd met de rest van de stappen zoals `barman recover`, `pg_upgrade`, `delete_old_cluster.sh`, enz.
