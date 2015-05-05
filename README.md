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

### MGET multiple items from the platform by IDs (C#)
```c#
        public JObject MgetJson(string path, JToken postData, params KeyValuePair<string, string>[] parameters)
        {
            string query = path;
            string json = "";

            if (parameters.Length > 0)
            {
                query += "?";
                foreach (KeyValuePair<string, string> parameter in parameters)
                {
                    query += parameter.Key + "=" + HttpUtility.UrlEncode(parameter.Value) + "&";
                }
            }

            string encQuery = HttpUtility.UrlEncode(query);

            HttpWebRequest req = WebRequest.Create(platformUrl) as HttpWebRequest;

	    // this adds the actor, key, accept:json, etc headers (see post example)
  	    addHeaders(req);

            req.Method = "MGET";
            req.ContentType = "application/json";

            if (postData != null)
            {
                json = postData.ToString();
                StreamWriter sw = new StreamWriter(req.GetRequestStream());
                sw.Write(json);
                sw.Flush();
                sw.Close();
            }

            HttpWebResponse res = req.GetResponse() as HttpWebResponse;
            StreamReader sr = new StreamReader(res.GetResponseStream());
            json = sr.ReadToEnd();

            JObject result = JObject.Parse(json);

            return result;
        }
  ```
  
  ### Open a Websocket session 
  ```c#
        internal JObject StartWSSession()
        {
            string json;

            WebRequest req = WebRequest.Create("ws-11.flow.net/session");
            // add headers
            req.Method = "POST";
            req.ContentType = "application/json";
  	    addHeaders(req);

	    // no post body

            HttpWebResponse res = (HttpWebResponse)req.GetResponse();

            StreamReader sr = new StreamReader(res.GetResponseStream());
            json = sr.ReadToEnd();

            JObject result = JObject.Parse(json);

            return result;
        }

        internal void Connect()
        {
            this.subscribedFlows = new List<Flow>();

	    JObject sessionInfo = StartWSSession();
            if ((bool)sessionInfo["head"]["ok"])
            {
                this.sessionId = (string)sessionInfo["body"]["id"];
                string url = FlowWSConnector.wsUrl + this.sessionId + "/ws";

                this.ws = new WebSocket(url);

                this.ws.OnOpen += ws_OnOpen;
                this.ws.OnMessage += ws_OnMessage;
                this.ws.OnClose += ws_OnClose;
                this.ws.OnError += ws_OnError;

		// need heartbeat to keep connection alive
                this.heartbeatTimer = new Timer(20000);
                this.heartbeatTimer.Elapsed += heartbeatTimer_Elapsed;
                this.heartbeatTimer.Enabled = true;

                this.ws.Connect();
            }
            else
            {
                throw new FlowConnectionException("Could not connect to the WS server.");
            }
        }
        
	internal void SendHeartbeat()
        {
            if (this.ws != null && this.ws.IsAlive)
            {
                JObject jo = new JObject(new JProperty("type", "heartbeat"));
                this.ws.Send(jo.ToString());
            }
        }
 
        void ws_OnOpen(object sender, EventArgs e)
        {
            foreach (Flow f in flowsToSubscribeWSTo)
            {
                this.Subscribe(f);
            }
        }
        
        void ws_OnMessage(object sender, MessageEventArgs e)
        {
            JObject jo = JObject.Parse(e.Data);
            string type = (string)jo["type"];

            if ("message".Equals(type))
            {
                JObject j = (JObject)jo["value"];
                string id = (string)j["id"];

                if (id[0] == 'd') // this is a drop
                    UpdateUI(j);
            }
            
        }        
```

### Subscribe a user to a Flow (this is different from a websocket subscribe)
```
        public void SubscribeTo(FlowRestConnector conn, string id)
        {
            JObject jo =
                new JObject(
                    new JProperty(
                        "subscriptions",
                        new JArray(
                            new JObject(
                                new JProperty("id", id),
                                new JProperty("isPublic", false)))));

            conn.PutJson("/identity/" + this.id + "/subscriptions.json", jo);
            this.ReloadSubscriptions(conn);
        }
```

### Load subscriptions for a user
```
            JObject o = conn.GetJson("/identity/" + this.id + "/subscriptions.json",
                new KeyValuePair<string, string>("hints", "0"),
                new KeyValuePair<string, string>("refs", "0"));

            if (o["body"] == null)
                return;

            if (o["body"]["subscriptions"] != null)
            {
                this.subscribedCollectionIds = new List<string>();
                this.subscribedFlowIds = new List<string>();

                foreach (JObject jo in o["body"]["subscriptions"])
                {
                    string id = (string)jo["id"];

                    if (id[0] == 'f') // the first char of the ID is f for flow, d for drop
                    {
                        this.subscribedFlowIds.Add(id);
                        subscribedFlowIdCount++;
                    }
                    else if (id[0] == 's')
                    {
                        this.subscribedCollectionIds.Add(id);
                        subscribedCollectionCount++;
                    }
                }
            }
        }
```

### Load my Flows
```
            JObject query
                = new JObject(
                    new JProperty("path", new JObject(
                        new JProperty("type", "expression"),
                        new JProperty("value", new JObject(
                            new JProperty("operator", "regex"),
                            new JProperty("operand", "^" + conn.GetMyIdentity().path + "/((?!:reports).)*$"))))));


            JObject o = conn.GetJson("/flow.json",
                new KeyValuePair<string, string>("hints", "0"),
                new KeyValuePair<string, string>("limit", "50"),
                new KeyValuePair<string, string>("criteria", query.ToString()));

            List<Flow> flows = new List<Flow>();

            if (o["body"] == null)
                return flows;

            foreach (JObject oFlow in o["body"])
            {
                flows.Add(new Flow(oFlow));
            }

            return flows;
```

### Post a Flow to the server
```
        override public JObject AsJObject()
        {
            JObject jo = base.AsJObject();

            jo["path"] = this.path;
            jo["dropPermissions"] = this.dropPermissions.AsJObject();

            JArray jaTemplate = new JArray();

            foreach (TemplateElement t in this.template)
                jaTemplate.Add(t.AsJObject());

            jo["template"] = jaTemplate;
            return jo;
        }

        public void Post(FlowRestConnector conn, Collection parentCollection = null)
        {
            if (this.id != null && !"".Equals(this.id))
            {
                JObject jo = this.AsJObject();

                // existing flow
                JObject o = conn.PutJson("/flow/" + this.id + ".json", jo);
                this.Load((JObject)o["body"]);
            }
            else
            {
                if (this.displayName == null)
                    this.displayName = this.name;

                this.name = Util.ToSlug(this.name);

                if (this.path == null)
                    this.path = conn.GetMyIdentity().path + "/" + Util.ToSlug(this.name);

                JObject jo = this.AsJObject();
                jo.Remove("id");

                // new flow
                JObject o = conn.PostJson("/flow.json", jo);
                this.Load((JObject)o["body"]);
            }
        }
```

### Post a drop to the server
	
