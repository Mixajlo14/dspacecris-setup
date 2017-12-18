# Zahtevi prema WoS-u

## WS Lite

Primer podataka: https://clarivate.com/products/data-integration/sample-data/ws-lite/

Autentifikacija:
```
curl -d @authlite.xml -x proxy.uns.ac.rs:8080 http://search.webofknowledge.com/esti/wokmws/ws/WOKMWSAuthenticate
```

Slanje search zahteva:
```
curl -v -d @woklite.xml -x proxy.uns.ac.rs:8080 --cookie "SID=XYZXYZXYZXYZ" http://search.webofknowledge.com/esti/wokmws/ws/WokSearchLite
```

uvek dobijemo 
```
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
	<soap:Body>
		<soap:Fault>
			<faultcode>soap:Server</faultcode>
			<faultstring>Not authorized for product: WWS</faultstring>
			<detail>
				<ns3:AuthenticationException xmlns:ns2="http://woksearchlite.v3.wokmws.thomsonreuters.com" xmlns:ns3="http://woksearch.v3.wokmws.thomsonreuters.com"/>
			</detail>
		</soap:Fault>
	</soap:Body>
</soap:Envelope>
```

## WS Premium

Primer podataka: https://clarivate.com/products/data-integration/sample-data/premium/

```
curl -d @auth.xml -x proxy.uns.ac.rs:8080 http://search.webofknowledge.com/esti/wokmws/ws/WOKMWSAuthenticate
curl -v -d @wok.xml -x proxy.uns.ac.rs:8080 --cookie "SID=XYZXYZXYZXYZ" http://search.webofknowledge.com/esti/wokmws/ws/WokSearchLite
```
