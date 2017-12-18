## Zadaci nakon instalacije

### Početna konfiguracija

```
dspace load-cris-configuration -f c:/dspace/bin/#konfiguracija.xls
```

TODO: istraziti sta je u konfiguraciji

### jspui kao root webapp

Zaustaviti Tomcat.
```
systemctl stop tomcat
```

Preimenovati `jspui` web aplikaciju tako da se ona prikazuje u root
kontekstu.
```
cd /opt/tomcat/webapps
rm -rf ROOT
mv jspui/ ROOT
```

Pokrenuti Tomcat.
```
systemctl start tomcat
```

### UI na srpskom jeziku

Dodati [Messages.properties](jspui/WEB-INF/classes/Messages.properties) u `/opt/tomcat/webapps/ROOT/WEB-INF/classes`.
```
systemctl restart tomcat
```

### Izmenjeni UI elementi

Izmene su u jspui aplikaciji.

### Backup PostgreSQL baze

```
pg_dump --clean --create --file=dspace001.sql dspace
```

### Izmena u `jspui/discovery/static-globalsearch-component-facet.jsp`

U sadasnjem resenju zamenjene su poruke u Messages.properties tako da se redosled ikona u ovom fajlu poklapa sa naslovima. 

TODO: ispravi to

### Network Collaboration

```
/dspace/bin/dspace dsrun org.dspace.app.cris.batch.ScriptIndexNetwork -a
```

### ORCID veza

[Jersey Apache Connector](https://mvnrepository.com/artifact/org.glassfish.jersey.connectors/jersey-apache-connector)
puca prilikom korišćenja [Apache Commons HttpClient](https://mvnrepository.com/artifact/commons-httpclient/commons-httpclient)
sa sledećom greškom:
```
Caused by: java.lang.NoSuchMethodError: org.apache.http.impl.client.HttpClientBuilder.setConnectionManagerShared(Z)Lorg/apache/http/
impl/client/HttpClientBuilder;
        at org.glassfish.jersey.apache.connector.ApacheConnector.<init>(ApacheConnector.java:240)
        at org.glassfish.jersey.apache.connector.ApacheConnectorProvider.getConnector(ApacheConnectorProvider.java:115)
        at org.glassfish.jersey.client.ClientConfig$State.initRuntime(ClientConfig.java:418)
        at org.glassfish.jersey.client.ClientConfig$State.access$000(ClientConfig.java:88)
        at org.glassfish.jersey.client.ClientConfig$State$3.get(ClientConfig.java:120)
        at org.glassfish.jersey.client.ClientConfig$State$3.get(ClientConfig.java:117)
        at org.glassfish.jersey.internal.util.collection.Values$LazyValueImpl.get(Values.java:340)
        at org.glassfish.jersey.client.ClientConfig.getRuntime(ClientConfig.java:726)
        at org.glassfish.jersey.client.ClientRequest.getConfiguration(ClientRequest.java:285)
        at org.glassfish.jersey.client.JerseyInvocation.validateHttpMethodAndEntity(JerseyInvocation.java:126)
        at org.glassfish.jersey.client.JerseyInvocation.<init>(JerseyInvocation.java:98)
        at org.glassfish.jersey.client.JerseyInvocation.<init>(JerseyInvocation.java:91)
        at org.glassfish.jersey.client.JerseyInvocation$Builder.method(JerseyInvocation.java:402)
        at org.glassfish.jersey.client.JerseyInvocation$Builder.get(JerseyInvocation.java:302)
        at org.dspace.authority.orcid.OrcidService.search(OrcidService.java:461)
```

Problem je što Jersey verzija 2.18 koja se koristi u projektu poziva metodu koja ne postoji verziji 3.1 Apache Commons HttpClienta.
Problem se može zaobići tako što se u `jspui/WEB-INF/lib` dodaju fajlovi:

 * jersey-common-2.16.jar
 * jersey-client-2.16.jar
 * jersey-apache-connector-2.16.jar

a originalni (verzija 2.18) uklone.

### Scopus import

#### Proxy settings

Klasa `/dspace-api/src/main/java/org/dspace/submit/lookup/ScopusService.java` ne konfiguriše proxy na ispravan
način. Umesto
```
HttpHost proxy = new HttpHost(proxyHost, Integer.parseInt(proxyPort),
        "http");
client.getParams().setParameter(ConnRoutePNames.DEFAULT_PROXY,
        proxy);
System.out.println(client.getParams()
        .getParameter(ConnRoutePNames.DEFAULT_PROXY));

```
to treba uraditi ovako:
```
import org.apache.commons.httpclient.HostConfiguration;

HostConfiguration hostCfg = client.getHostConfiguration();
hostCfg.setProxy(proxyHost, Integer.parseInt(proxyPort));
client.setHostConfiguration(hostCfg);
```

#### Scopus query

Klasa `/dspace-cris/api/src/main/java/org/dspace/app/cris/batch/ScopusFeed.java` postavlja upit tako što uvek
na kraj upita dodaje `AND ORIG-LOAD-DATE AFT endmillis AND ORIG-LOAD-DATE BEF startmillis`. Ako se prilikom
pokretanja importa ne navede opseg datuma ovde se uzima neki default. Sa tim tekstom na kraju upita dobijao
sam samo prazne odgovore od Scopusa. Kada se ukloni taj tekst, tako što se komentariše sledeće parče u metodi
`convertToImpRecordItem`:
```
/*
if (StringUtils.isNotBlank(start))
{
    query += " AND ORIG-LOAD-DATE AFT " + start;
}
if (StringUtils.isNotBlank(end))
{
    query += " AND ORIG-LOAD-DATE BEF " + end;
}
*/
```
onda dobijamo ispravne odgovore od Scopusa za sledeći import:
```
/dspace/bin/dspace dsrun org.dspace.app.cris.batch.ScopusFeed -q "AU-ID(55948342700)" -p mbranko@uns.ac.rs -c 2
```

Pored toga, u klasi `/dspace-cris/api/src/main/java/org/dspace/app/cris/batch/ScopusFeed.java` ne proverava se
da li `record.getValues("eid").get(0)` moze da vrati element, pa smo pre poziva, na vrhu petlje, dodali
```
if (record.getValues("eid").isEmpty())
    continue;
```


#### Mapiranje Scopus podataka na bazu

Kada se sada pusti gornji skript, upis podataka pukne:
```
java.sql.SQLException: Attempting to insert null value into SQL query.
        at org.dspace.storage.rdbms.DatabaseManager.loadParameters(DatabaseManager.java:1677)
        at org.dspace.storage.rdbms.DatabaseManager.updateQuery(DatabaseManager.java:513)
        at org.dspace.app.cris.batch.dao.ImpRecordDAO.write(ImpRecordDAO.java:91)
        at org.dspace.app.cris.batch.ScopusFeed.main(ScopusFeed.java:246)
```
jer pokušava da uradi ovakav insert:
```
INSERT INTO imp_record  VALUES ( 69 , null:2017-12-02.16-17-29 , 1 , 2 , y , insert , null , null )
```

Problem je u tome što se u klasi `/dspace-cris/api/src/main/java/org/dspace/app/cris/batch/bte/ImpRecordOutputGenerator.java`,
u metodi `merge`, nikad ne poziva:
```
item.setSourceRef(providerName);
```

Rešenje je da se izmeni fajl `/dspace/config/spring/api/bte.xml` tako da se izmeni sledeći red:
```
    <bean name="outputMap" class="java.util.HashMap" scope="prototype">
        <constructor-arg>
            <map key-type="java.lang.String" value-type="java.lang.String">
                <!-- <entry value="eid" key="dc.identifier.eid" />  OVO JE STARA VREDNOST -->
                <entry value="eid" key="dc.identifier.scopus" />
```

da bi u narednoj fazi mapiranja naziv identifikatora bio ispravno prepoznat zbog sledeće konfiguracije:
```
<bean name="scopusOutputGenerator" class="org.dspace.app.cris.batch.bte.ImpRecordOutputGenerator">
    <property name="outputMap" ref="outputMap"/>
    <property name="sourceIdMetadata" value="dc.identifier.scopus" />       
    <property name="providerName" value="scopus" />     
</bean>
```

Sada treba još pokrenuti:

```
/dspace/bin/dspace dsrun org.dspace.app.cris.batch.ItemImportMainOA -E mbranko@uns.ac.rs
```
