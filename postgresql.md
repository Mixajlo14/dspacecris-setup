## Instalacija PostgreSQL 9.6

### Instalacija EPEL repozitorijuma

```
yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
```

[info](https://fedoraproject.org/wiki/EPEL)

### Instalacija PostgreSQL

Instalirati PostgreSQL yum repozitorijum:
```
yum install https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-7-x86_64/pgdg-centos96-9.6-3.noarch.rpm
```

Zatim instalirati PostgreSQL 9.6:
```
yum install postgresql96-server postgresql96-contrib
```

Inicijalizovati bazu:
```
/usr/pgsql-9.6/bin/postgresql96-setup initdb
```

Pokrenuti server:
```
systemctl enable postgresql-9.6
systemctl start postgresql-9.6
```

Promeniti lozinku za `postgres` korisnika u bazi:
```
su - postgres
psql -d template1 -c "ALTER USER postgres WITH PASSWORD 'newpassword';"
```

[info](https://www.linode.com/docs/databases/postgresql/how-to-install-postgresql-relational-databases-on-centos-7)

Podesiti fajl `/var/lib/pgsql/9.6/data/pg_hba.conf` tako da u njemu dva
reda za autentifikaciju glase:
```
local  all   all                             md5
host   all   all     127.0.0.1/32            md5
```
čime se uključuje autentifikacija pomoću lozinke. Restartovati server:
```
systemctl restart postresql-9.6
```

[info](https://stackoverflow.com/questions/18664074/getting-error-peer-authentication-failed-for-user-postgres-when-trying-to-ge)

