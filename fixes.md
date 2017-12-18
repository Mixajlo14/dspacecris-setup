# DSpace-CRIS 5.8.1 Fixes for Scopus Import

## Configuration

Our configuration in the build.properties file is as follows:
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
jspui.google.analytics.key  = ****
submission.lookup.scopus.apikey = ****
submission.lookup.webofknowledge.ip.authentication = true
scopus.query.param.default=affilorg("University of Novi Sad")
key.googleapi.maps = ****
cookies.policy.enabled = false
```

## Connecting to ORCID

[Jersey Apache Connector](https://mvnrepository.com/artifact/org.glassfish.jersey.connectors/jersey-apache-connector)
throws an exception while using [Apache Commons HttpClient](https://mvnrepository.com/artifact/commons-httpclient/commons-httpclient):
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

The problem is that Jersey 2.18 that is used in the project calls the method that does not exist in Apache Commons HttpClient version 3.1. A way
to fix this is to replace Jersey 2.18 with version 2.16 which includes the following files:

 * jersey-common-2.16.jar
 * jersey-client-2.16.jar
 * jersey-apache-connector-2.16.jar

## Proxy Setup

Using a proxy is not configured properly in 
`/dspace-api/src/main/java/org/dspace/submit/lookup/ScopusService.java`. 
Instead of doing it this way:
```
HttpHost proxy = new HttpHost(proxyHost, Integer.parseInt(proxyPort),
        "http");
client.getParams().setParameter(ConnRoutePNames.DEFAULT_PROXY,
        proxy);
System.out.println(client.getParams()
        .getParameter(ConnRoutePNames.DEFAULT_PROXY));

```
the proper way to configure proxy for Apache Commons HttpClient is like this:
```
import org.apache.commons.httpclient.HostConfiguration;
...
HostConfiguration hostCfg = client.getHostConfiguration();
hostCfg.setProxy(proxyHost, Integer.parseInt(proxyPort));
client.setHostConfiguration(hostCfg);
```

## Querying Scopus

The class `/dspace-cris/api/src/main/java/org/dspace/app/cris/batch/ScopusFeed.java` 
constructs a Scopus query by always appending 
`AND ORIG-LOAD-DATE AFT endmillis AND ORIG-LOAD-DATE BEF startmillis` to the end of the 
query in the method `convertToImpRecordItem`. I always get an empty result set 
regardless of the dates used, so I commented this part out. We can always partition 
the result set by year.
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
With this commented out, the following query runs correctly:
```
/dspace/bin/dspace dsrun org.dspace.app.cris.batch.ScopusFeed -q "AU-ID(55948342700)" -p mbranko@uns.ac.rs -c 1
```

## Checking the existence of Scopus ID

The class `/dspace-cris/api/src/main/java/org/dspace/app/cris/batch/ScopusFeed.java` does not check whether
`record.getValues("eid")` returns a non-empty list, so the code may raise an exception. I have added the following 
check at the top of the loop in the method `convertToImpRecordItem`:
```
if (record.getValues("eid").isEmpty())
    continue;
```

## Mapping of Scopus metadata

Running the Scopus import utility will raise the following exception:
```
java.sql.SQLException: Attempting to insert null value into SQL query.
        at org.dspace.storage.rdbms.DatabaseManager.loadParameters(DatabaseManager.java:1677)
        at org.dspace.storage.rdbms.DatabaseManager.updateQuery(DatabaseManager.java:513)
        at org.dspace.app.cris.batch.dao.ImpRecordDAO.write(ImpRecordDAO.java:91)
        at org.dspace.app.cris.batch.ScopusFeed.main(ScopusFeed.java:246)
```
because it tries to insert the following:
```
INSERT INTO imp_record  VALUES ( 69 , null:2017-12-02.16-17-29 , 1 , 2 , y , insert , null , null )
```

The problem is that `item.setSourceRef(providerName)` is never called in the method `merge` of 
`/dspace-cris/api/src/main/java/org/dspace/app/cris/batch/bte/ImpRecordOutputGenerator.java`. The fix is to
change the file `/dspace/config/spring/api/bte.xml` as follows:
```
    <bean name="outputMap" class="java.util.HashMap" scope="prototype">
        <constructor-arg>
            <map key-type="java.lang.String" value-type="java.lang.String">
                <!-- <entry value="eid" key="dc.identifier.eid" />  THIS IS THE OLD VALUE -->
                <entry value="eid" key="dc.identifier.scopus" />
```
so that it matches the `sourceIdMetadata` property in the following bean:
```
<bean name="scopusOutputGenerator" class="org.dspace.app.cris.batch.bte.ImpRecordOutputGenerator">
    <property name="outputMap" ref="outputMap"/>
    <property name="sourceIdMetadata" value="dc.identifier.scopus" />       
    <property name="providerName" value="scopus" />     
</bean>
```

## Fetching Scopus Metrics

The class `/dspace-cris-ametrics/metrics/retrieve-api/src/main/java/org/dspace/app/cris/metrics/scopus/services/ScopusService.java`
should set proxy if needed. Right after the line:
```
method = new HttpGet(uriBuilder.build());
```

A check for proxy should be added:
```
if (org.apache.commons.lang.StringUtils.isNotBlank(proxyHost)
        && org.apache.commons.lang.StringUtils.isNotBlank(proxyPort)) {
    HttpHost proxy = new HttpHost(proxyHost, Integer.parseInt(proxyPort), "http");
    RequestConfig config = RequestConfig.custom().setProxy(proxy).build();
    method.setConfig(config);
}
```