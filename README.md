# Flow Platform 3.0 Examples

### Basic GET from Flow platform (APEX code)

```java
    static public void getJSON(String uri) 
    {
		String t = String.valueOf(System.currentTimeMillis());
        HttpRequest req = new HttpRequest();

        String s = 'x-actor:' + ACTOR +
            'x-key:' + API_KEY +
            'x-timestamp:' + t + API_SECRET;
        String sig = EncodingUtil.convertToHex(Crypto.generateDigest('SHA1', Blob.valueOf(s))); 
        
        req.setEndpoint(API_URL + uri);
        req.setMethod('GET');
        req.setHeader('X-Actor', ACTOR);
        req.setHeader('X-Key', API_KEY);
        req.setHeader('X-Timestamp', t);
        req.setHeader('X-Signature', sig);
        req.setHeader('Content-type', 'application/json');
        
        Http http = new Http();
        HttpResponse res = http.send(req);
        
        System.debug(res.getBody());
    }
```

### Basic POST to the platform

```java
    static public void postJSON(String uri, String body) 
    {
		String t = String.valueOf(System.currentTimeMillis());
        HttpRequest req = new HttpRequest();

        String s = 'x-actor:' + ACTOR +
            'x-key:' + API_KEY +
            'x-timestamp:' + t + API_SECRET;
        String sig = EncodingUtil.convertToHex(Crypto.generateDigest('SHA1', Blob.valueOf(s))); 
        
        System.debug(body);
        
        req.setEndpoint(API_URL + uri);
        req.setMethod('POST');
        req.setBody(body);
        req.setHeader('X-Actor', ACTOR);
        req.setHeader('X-Key', API_KEY);
        req.setHeader('X-Timestamp', t);
        req.setHeader('X-Signature', sig);
        req.setHeader('Content-type', 'application/json');
        
        Http http = new Http();
        HttpResponse res = http.send(req);
        
        System.debug(res.getBody());
    }
```
