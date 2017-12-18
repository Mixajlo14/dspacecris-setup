## Instalacija DSpace-CRIS

### Kreiranje baze podataka

Potrebno je kreirati korisnika `dspace` sa lozinkom `dspace`:
```
su - postgres
createuser -U postgres -d -A -P dspace
```
A zatim kreirati i bazu podataka sa nazivom `dspace`:
```
createdb -U dspace -E UNICODE dspace
```

### Preuzimanje izvornog koda

```
mkdir /opt/crisinstallation
cd /opt/crisinstallation
git clone https://github.com/4Science/DSpace.git --branch dspace-5_x_x-cris dspace-parent/
```

### Početna konfiguracija

Potrebno je podesiti parametre u fajlu `/opt/crisinstallation/dspace-parent/build.properties`:

```
dspace.install.dir=/dspace
dspace.hostname = dspacecris.uns.ac.rs
dspace.baseUrl = https://dspacecris.uns.ac.rs
dspace.ui = jspui
dspace.url = ${dspace.baseUrl}
dspace.name = DSpace-CRIS at University of Novi Sad
mail.server = smtp.uns.ac.rs
mail.server.username=dspace
mail.server.password=*****
handle.canonical.prefix = ${dspace.url}/handle/
http.proxy.host = proxy.uns.ac.rs
http.proxy.port = 8080
authentication-oauth.application-client-name = DSpace-CRIS at University of Novi Sad
authentication-oauth.application-client-id = ****
authentication-oauth.application-client-secret = ****
authentication-oauth.application-client-redirect = ${dspace.baseUrl}/oauth-login
cris.ametrics.elsevier.scopus.enabled = true
cris.ametrics.elsevier.scopus.apikey = ****
cris.ametrics.google.scholar.enabled = true
cris.ametrics.altmetric.enabled = true
jspui.google.analytics.key	= ****
submission.lookup.scopus.apikey = ****
submission.lookup.webofknowledge.ip.authentication = true
scopus.query.param.default=affilorg("University of Novi Sad")
key.googleapi.maps = ****
cookies.policy.enabled = false
```

### Pokretanje mavena

```
mkdir /dspace
cd /opt/crisinstallation/dspace-parent/
mvn package
```

### Pokretanje anta

```
cd /opt/crisinstallation/dspace-parent/dspace/target/dspace-installer/
ant fresh_install
```

### Inicijalizacija baze
```
/dspace/bin/dspace database clean
/dspace/bin/dspace database migrate
/dspace/bin/dspace create-administrator
/dspace/bin/dspace load-cris-configuration -f /root/work/#konfiguracija.xls
/dspace/bin/dspace load-cris-configuration -f /root/work/#konfiguracija.xls
```

### Kopiranje webapp fajlova

Minimalno su potrebne `jspui` i `solr` aplikacije a po potrebi se mogu kopirati i druge.

```
systemctl stop tomcat
cd /dspace/webapps
cp -R jspui/ oai/ rdf/ rest/ solr/ sword/ swordv2/ /opt/tomcat/webapps
```

Aplikaciju `jspui` je potrebno preimenovati u `ROOT`.
```
cd /opt/tomcat/webapps
rm -rf ROOT
mv jspui/ ROOT
```

### Ispravke za UI

Iskopirati ispravke iz [jspui](jspui) foldera u `/opt/tomcat/webapps/ROOT`.

### Pokretanje i molitva

Pokrenuti Tomcat.
```
systemctl start tomcat
```

### Provera u browseru

Proveriti da li je pokrenut DSpace-CRIS: [https://dspacecris.uns.ac.rs/](https://dspacecris.uns.ac.rs/)
