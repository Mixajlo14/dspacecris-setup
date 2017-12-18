## Instalacija CentOS 7 i potrebnih alata

Prvi korak predstavlja minimalna instalacija za CentOS 7. Podrazumeva se da je 
mreža konfigurisana na odgovarajući način.

### Yum proxy

Potrebno je podesiti proxy za `yum`. U fajlu `/etc/yum.conf` dodati red:

```
proxy=http://proxy.uns.ac.rs:8080
```

### Firewall konfiguracija

```
yum install firewalld
firewall-cmd --add-service=http
firewall-cmd --add-service=https
firewall-cmd --add-service=http --permanent
firewall-cmd --add-service=https --permanent
firewall-cmd --add-port=8080/tcp
firewall-cmd --add-port=8080/tcp --permanent
```

Nakon završetka instalacije možemo zatvoriti port 8080.

### Instalacija wget
Instalirati `wget` pomoću
```
yum -y install wget
```

Zatim u fajl `~/.wgetrc` dodati redove:
```
use_proxy=yes
http_proxy=proxy.uns.ac.rs:8080
https_proxy=proxy.uns.ac.rs:8080
```
[info](https://stackoverflow.com/questions/11211705/setting-proxy-in-wget)

### Instalacija curl

Instalirati `curl` pomoću
```
yum -y install curl
```

Zatim u fajl `~/.curlrc` dodati red:
```
proxy = proxy.uns.ac.rs:8080
```
[info](https://stackoverflow.com/questions/7559103/how-to-setup-curl-to-permanently-use-a-proxy)

### Instalacija git

Instalirati `git` pomoću
```
yum -y install git
```
Podesiti proxy za git komandom:
```
git config --global http.proxy http://proxy.uns.ac.rs:8080
```
[info](https://stackoverflow.com/questions/783811/getting-git-to-work-with-a-proxy-server)

### Instalacija Java

Potrebno je skinuti instalaciju JDK 8 sa zvaničnog sajta ili iskoristiti sledeći skript za skidanje instalacije direktno na server:

```
wget https://gist.githubusercontent.com/n0ts/40dd9bd45578556f93e7/raw/0e9112d60fc0c9228a30e4c92d5e845df3bc1beb/get_oracle_jdk_linux_x64.sh
chmod +x get_oracle_jdk_linux_x64.sh
./get_oracle_jdk_linux_x64.sh
```

Instalirati Javu pomoću:
```
yum localinstall jdk-8u152-linux-x64.rpm
```

Proveriti instalaciju pomoću:
```
java -version
```

### Instalacija Maven

```
cd /usr/local
wget http://www-eu.apache.org/dist/maven/maven-3/3.5.2/binaries/apache-maven-3.5.2-bin.tar.gz
tar xzf apache-maven-3.5.2-bin.tar.gz
ln -s apache-maven-3.5.2 maven
```

Dodati fajl `/etc/profile.d/maven.sh`:
```
#!/bin/bash
export M2_HOME=/usr/local/maven
export PATH=${M2_HOME}/bin:${PATH}
```

I na kraju
```
chmod +x /etc/profile.d/maven.sh
source /etc/profile.d/maven.sh
```

Proveriti instalaciju:
```
maven -version
```

[info](https://tecadmin.net/install-apache-maven-on-centos/#)

Potrebno je podesiti i proxy za Maven. U fajlu `/usr/local/maven/conf/settings.xml` treba naći `<proxies>` sekciju i izmeniti je:
```
<!-- proxies
   | This is a list of proxies which can be used on this machine to connect to the network.
   | Unless otherwise specified (by system property or command-line switch), the first proxy
   | specification in this list marked as active will be used.
   |-->
  <proxies>
    <!-- proxy
     | Specification for one proxy, to be used in connecting to the network.
     |-->
    <proxy>
      <id>optional</id>
      <active>true</active>
      <protocol>http</protocol>
      <!--
      <username>proxyuser</</username>
      <password>proxypass</password>
      -->
      <host>proxy.uns.ac.rs</host>
      <port>8080</port>
      <nonProxyHosts>localhost</nonProxyHosts>
    </proxy>
    <!-- -->
  </proxies>
```
[info](https://www.mkyong.com/maven/how-to-enable-proxy-setting-in-maven/)

### Instalacija ant
```
cd /usr/local
wget http://www-eu.apache.org/dist//ant/binaries/apache-ant-1.10.1-bin.tar.gz
tar xzf apache-ant-1.10.1-bin.tar.gz
ln -s apache-ant-1.10.1 ant
```

Dodati fajl `/etc/profile.d/ant.sh`:
```
#!/bin/bash
export ANT_OPTS="-Dhttp.proxyHost=proxy.uns.ac.rs -Dhttp.proxyPort=8080"
export ANT_HOME=/usr/local/ant
export PATH=$ANT_HOME/bin:$PATH
```

I na kraju:
```
chmod +x /etc/profile.d/ant.sh
source /etc/profile.d/ant.sh
```

Proveriti instalaciju:
```
ant -version
```

### Instalacija Apache Tomcat

```
cd /opt
wget http://www-eu.apache.org/dist/tomcat/tomcat-8/v8.5.24/bin/apache-tomcat-8.5.24.tar.gz
tar zxvf apache-tomcat-8.5.24.tar.gz
ln -s apache-tomcat-8.5.24 tomcat
rm -f apache-tomcat-8.5.24.tar.gz
```

Dodati fajl `/etc/systemd/system/tomcat.service`:
```
# Systemd unit file for tomcat
[Unit]
Description=Apache Tomcat Web Application Container
After=syslog.target network.target

[Service]
Type=forking

Environment=JAVA_HOME=/usr/java/latest
Environment=CATALINA_PID=/opt/tomcat/temp/tomcat.pid
Environment=CATALINA_HOME=/opt/tomcat
Environment=CATALINA_BASE=/opt/tomcat
Environment='CATALINA_OPTS=-Xms512M -Xmx2048M -server -XX:+UseParallelGC'
Environment='JAVA_OPTS=-Djava.awt.headless=true -Djava.security.egd=file:/dev/./urandom -Dfile.encoding=UTF-8 -Dorg.apache.el.parser.SKIP_IDENTIFIER_CHECK=true'

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/bin/kill -15 $MAINPID

User=root
Group=root

[Install]
WantedBy=multi-user.target
```

Izmeniti element `Connector` u fajlu `/opt/tomcat/conf/server.xml`:
```
<Connector port="8080" protocol="HTTP/1.1"
           maxThreads="150"
           minSpareThreads="25"
           maxSpareThreads="75"
           enableLookups="false"
           acceptCount="100"
           disableUploadTimeout="true"
           URIEncoding="UTF-8"
           connectionTimeout="20000"
           redirectPort="8443" />
```

Registrovati servis za Apache Tomcat:
```
systemctl enable tomcat
```

Pokrenuti Apache Tomcat:
```
systemctl start tomcat
```

