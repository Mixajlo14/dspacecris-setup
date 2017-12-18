## Instalacija Apache

### Instalacija i konfigurisanje Apache servera

```
yum install httpd mod_ssl
```

U fajlu `/etc/httpd/conf/httpd.conf` editovati red:
```
ServerName dspacecris:80
```
i dodati red na kraj:
```
IncludeOptional vhosts.d/*.conf
```

Kreirati konfiguracioni fajl virtuelni host `/etc/httpd/vhosts.d/dspacecris.conf`:
```
<VirtualHost *:80>
  ServerName dspacecris.uns.ac.rs
  ServerAlias www.dspacecris.uns.ac.rs
  ErrorLog /var/log/httpd/dspace_errors.log
  LogLevel warn
  CustomLog /var/log/httpd/dspace_access.log combined
</VirtualHost>
```


### Instalacija i konfigurisanje certbot

```
yum install certbot python2-certbot-apache
```

Proveriti da li Apache radi
```
systemctl start httpd
```

Certbot, ako ide kroz proxy, uzima u obzir standardne promenljive:
```
export http_proxy=http://proxy.uns.ac.rs:8080
export https_proxy=http://proxy.uns.ac.rs:8080
```

Pokrenuti certbot:
```
certbot --apache
```

Na pitanje da li da se generiše još jedan fajl za SSL konfiguraciju 
i da se sav HTTP saobraćaj preusmeri na HTTPS odgovoriti potvrdno.

Sada konfiguracija za HTTP virtuelni host izgleda ovako:
```
<VirtualHost *:80>
  ServerName dspacecris.uns.ac.rs
  ServerAlias www.dspacecris.uns.ac.rs
  ErrorLog /var/log/httpd/dspace_errors.log
  LogLevel warn
  CustomLog /var/log/httpd/dspace_access.log combined

  RewriteEngine on
  RewriteCond %{SERVER_NAME} =dspacecris.uns.ac.rs [OR]
  RewriteCond %{SERVER_NAME} =www.dspacecris.uns.ac.rs
  RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>
```

A konfiguracija za HTTPS host treba da izgleda ovako (dodata proxy konfiguracija na kraju):
```
<VirtualHost *:443>
  ServerName dspacecris.uns.ac.rs
  ServerAlias www.dspacecris.uns.ac.rs
  ErrorLog /var/log/httpd/dspace_errors.log
  LogLevel warn
  CustomLog /var/log/httpd/dspace_access.log combined

  Include /etc/letsencrypt/options-ssl-apache.conf
  SSLCertificateFile /etc/letsencrypt/live/dspacecris.uns.ac.rs/cert.pem
  SSLCertificateKeyFile /etc/letsencrypt/live/dspacecris.uns.ac.rs/privkey.pem
  Include /etc/letsencrypt/options-ssl-apache.conf
  SSLCertificateChainFile /etc/letsencrypt/live/dspacecris.uns.ac.rs/chain.pem

  ProxyRequests off
  ProxyPreserveHost on
  ProxyPass / http://localhost:8080/
  ProxyPassReverse / http://localhost:8080/  
</VirtualHost>
```

Omogućiti Apache serveru da otvara izlazne konekcije prema Tomcatu:
```
/usr/sbin/setsebool httpd_can_network_connect 1
```
A onda i trajno to uključiti:
```
/usr/sbin/setsebool -P httpd_can_network_connect 1
```

Na kraju restart:
```
systemctl restart httpd
```

Isprobati automatsko obnavljanje sertifikata:
```
certbot renew --dry-run
```

Ako je sve u redu, podesiti pokretanje certbota
```
crontab -e
```

Dodati liniju
```
35 2 * * * http_proxy=http://proxy.uns.ac.rs:8080 https_proxy=http://proxy.uns.ac.rs:8080 /usr/bin/certbot renew >> /var/log/le-renew.log
```
