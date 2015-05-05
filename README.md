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
```c#
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
```c#
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
```c#
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

### Example model class for a drop object
```c#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Net;
using System.Diagnostics;

using FlowModels.Net;

using Newtonsoft.Json.Linq;

namespace FlowModels
{
    /// <summary>
    /// A Flow Drop object
    /// </summary>
    public class Drop : DomainObject
    {

        public DropElement deName = null;
        public DropElement deDescription = null;
        public List<DropElement> elems;
        public long weight = -1;


        /// <summary>
        /// create an empty drop
        /// </summary>
        public Drop() : base()
        {
            this.name = this.description = null;

            this.elems = new List<DropElement>();
        }

        /// <summary>
        /// create a drop from a jobject
        /// </summary>
        /// <param name="o"></param>
        public Drop(JObject o) : base()
        {
            this.Load(o);
        }


        /// <summary>
        /// Loads this object
        /// </summary>
        /// <param name="o"></param>
        override protected void Load(JObject o)
        {
            base.Load(o);

            JToken t;
            this.weight = o.TryGetValue("weight", out t) ? (long)t : -1;

            this.name = this.description = null;

            this.elems = new List<DropElement>();

            foreach (KeyValuePair<string, JToken> kvp in (JObject)o["elems"])
            {
                try
                {
                    this.elems.Add(DropElement.Create(kvp.Key, (JObject)kvp.Value));
                }
                catch (ModelLoadException e)
                {
                    // TODO: should we do something here?
                }
            }
        }


        /// <summary>
        /// Returns the drop element by name
        /// </summary>
        /// <param name="name"></param>
        /// <returns></returns>
        public DropElement GetDropElement(string name)
        {
            foreach (DropElement d in this.elems)
            {
                if (name.Equals(d.name))
                    return d;
            }
            return null;
        }

        /// <summary>
        /// Returns the drop element value if the drop element exists and has a value; otherwise returns
        /// null.
        /// </summary>
        /// <param name="name"></param>
        /// <returns></returns>
        public object TryGetDropElementValue(string name)
        {
            DropElement d = this.GetDropElement(name);
            if (d == null)
                return null;

            return d.GetValue();
        }



        /// <summary>
        /// Try to set teh drop element value if it exists
        /// </summary>
        /// <param name="name"></param>
        /// <param name="val"></param>
        /// <returns>true if a value was set, false otherwise</returns>
        public bool TrySetDropElementValue(string name, object val)
        {
            this.UpdateTitleDesc(name, val);

            foreach (DropElement d in this.elems)
            {
                if (name.Equals(d.name))
                {
                    return d.TrySetValue(val);
                }
            }

            return false;
        }


        /// <summary>
        /// Adds a drop element to this drop if it's not there, otherwise it updates it
        /// </summary>
        /// <param name="name"></param>
        /// <param name="displayName"></param>
        /// <param name="type"></param>
        public void UpdateDropElement(string name, string displayName, string type)
        {
            if (displayName == null) displayName = name;

            foreach (DropElement de in this.elems)
            {
                if (de.name.Equals(name))
                {
                    this.elems.Remove(de);
                    break;
                }
            }
            this.elems.Add(DropElement.Create(name, displayName, type));
        }

        /// <summary>
        /// Return as string
        /// </summary>
        /// <returns></returns>
        override public string ToString()
        {
            return this.name + " / " + this.description;
        }


        /// <summary>
        /// Returns the item as a jobject
        /// </summary>
        /// <returns></returns>
        override public JObject AsJObject()
        {
            JObject jo = new JObject();
            JObject joElems = new JObject();
            
            foreach (DropElement d in elems)
                joElems.Add(d.name, d.AsJObject());

            jo["elems"] = joElems;
            return jo;
        }


        /// <summary>
        /// Returns the drop as HTML
        /// </summary>
        /// <returns></returns>
        public string AsHTML()
        {
            string str = "";

            str += Properties.Resources.Drop_HTMLHeader;
            str += "<div class=\"post-body\">";
            str += "<div class=\"post-description\">" + this.description + "</div>";

            foreach (DropElement elem in this.elems)
            {
                if (elem != this.deName && elem != this.deDescription)
                    str += elem.AsHTML();
            }
            str += "</div>";

            return str;
        }


        /// <summary>
        /// Remove this drop from the server. Requires that the drop is deleted from the object hierarchy
        /// locally by the caller.
        /// </summary>
        /// <param name="conn">The connector to communicate on</param>
        public void Invalidate(FlowRestConnector conn, string flowId)
        {
            Drop.Invalidate(conn, flowId, this.id);
        }


        /// <summary>
        /// Remove this drop from the server. Requires that the drop is deleted from the object hierarchy
        /// locally by the caller.
        /// </summary>
        /// <param name="conn">The connector to communicate on</param>
        public static void Invalidate(FlowRestConnector conn, string flowId, string dropId)
        {
            conn.DeleteJson("/drop/" + flowId + "/" + dropId + ".json");
        }


        /// <summary>
        /// Load drops for a single flow
        /// </summary>
        /// <param name="conn"></param>
        /// <param name="flowId"></param>
        /// <returns></returns>
        public static List<Drop> LoadDropsForSingleFlow(FlowRestConnector conn, string flowId)
        {

            JObject o = conn.GetJson("/drop/" + flowId + ".json",
                new KeyValuePair<string, string>("hints", "0"),
                new KeyValuePair<string, string>("refs", "0"));

            List<Drop> drops = new List<Drop>();

            if (o["body"] == null)
                return drops;

            foreach (JObject oDrop in o["body"])
            {
                drops.Add(new Drop(oDrop));
            }

            return drops;
        }

        /// <summary>
        /// Post to the server
        /// </summary>
        /// <param name="parentFlowId"></param>
        public void Post(FlowRestConnector conn, Flow parentFlow)
        {
            if (this.id != null)
            {

                // existing drop
                JObject jo = new JObject();
                JObject joElems = new JObject();

                foreach (DropElement d in elems)
                {
                    joElems.Add(
                        new JProperty(
                            d.name,
                            new JObject(
                                new JProperty("type", d.type),
                                new JProperty("value", d.ValueAsJObjectable()))));
                }

                jo["elems"] = joElems;
                jo.Add("path", parentFlow.path);

                JObject o = conn.PutJson("/drop/" + parentFlow.id + "/" + this.id + ".json", jo);
                this.Load((JObject)o["body"]);
            }
            else
            {
                // new drop
                JObject jo = this.AsJObject();
                jo.Add("path", parentFlow.path);

                JObject o = conn.PostJson("/drop.json", jo);
                this.Load((JObject)o["body"]);
            }
        }

    }
}
```

### Example DropElement model class
```c#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Text.RegularExpressions;
using System.Threading.Tasks;
using System.Diagnostics;

using Newtonsoft.Json.Linq;

namespace FlowModels
{
    /// <summary>
    /// A generic Drop element (should subclass for each type)
    /// </summary>
    abstract public class DropElement
    {
        public string name;
        public string type;
        public bool required;
        public bool inferred;
        protected string displayName;

        protected DropElement(string name, string displayName, string type, bool required, bool inferred)
        {
            this.name = name;
            this.type = type;
            this.required = required;
            this.inferred = inferred;
            this.displayName = displayName;
        }

        /// <summary>
        /// Returns the display name for the element
        /// </summary>
        /// <returns></returns>
        public string GetDisplayName()
        {
            if (this.displayName != null)
                return this.displayName;
            else
                return "";
        }

        /// <summary>
        /// Sets the value of the drop element
        /// </summary>
        /// <param name="value"></param>
        /// <returns></returns>
        abstract public bool TrySetValue(object value);


        /// <summary>
        /// Returns the value of the drop element
        /// </summary>
        /// <returns></returns>
        abstract public object GetValue();


        /// <summary>
        /// Returns the drop as a jobject
        /// </summary>
        /// <returns></returns>
        virtual public JObject AsJObject()
        {
            return new JObject(
                new JProperty("name", this.name),
                new JProperty("displayName", this.GetDisplayName()),
                new JProperty(FlowModels.Properties.Resources.Drop_TypeName, this.type),
                new JProperty("required", this.required),
                new JProperty("inferred", this.inferred),
                new JProperty("value", this.ValueAsJObjectable()));
        }


        /// <summary>
        /// Returns the value of this element as a jobjectable item - override for objects where
        /// getvalue returns a non-jobject-serializable value
        /// </summary>
        /// <returns></returns>
        virtual public object ValueAsJObjectable()
        {
            return this.GetValue();
        }


        /// <summary>
        /// Returns the drop as html
        /// </summary>
        /// <returns></returns>
        public virtual string AsHTML()
        {
            return "<div class=\"param\">" +
                "<div class=\"param-name\">" + this.GetDisplayName() + "</div>" +
                "<div class=\"param-value\">" + this.ToString() + "</div>" +
                "</div>";
        }

        /// <summary>
        /// Create a new drop element from values
        /// </summary>
        /// <param name="name"></param>
        /// <param name="displayName"></param>
        /// <param name="type"></param>
        /// <param name="value"></param>
        /// <returns></returns>
        public static DropElement Create(string name, string displayName, string type, object value = null)
        {
            DropElement d = null;

            switch (type)
            {
                case "string":
                    d = new TextDropElement(name, displayName, type);
                    break;

                case "text":
                    d = new TextDropElement(name, displayName, type);
                    break;

                case "boolean":
                    d = new BoolDropElement(name, displayName, type);
                    break;

                case "float":
                    d = new FloatDropElement(name, displayName, type);
                    break;

                case "media":
                    d = new MediaDropElement(name, displayName, type);
                    break;

                case "date":
                    d = new DateDropElement(name, displayName, type);
                    break;

                case "location":
                    d = new LocationDropElement(name, displayName, type);
                    break;

                case "url":
                    d = new UrlDropElement(name, displayName, type);
                    break;
            }
            return d;
        }

        /// <summary>
        /// Create a new drop element from a name and jobject
        /// </summary>
        /// <param name="name"></param>
        /// <param name="o"></param>
        /// <returns></returns>
        public static DropElement Create(string name, JObject o)
        {
            string type = (string)o[FlowModels.Properties.Resources.Drop_TypeName];
            string displayName = (string)o["displayName"];

            switch (type)
            {
                case "string":
                    return DropElement.Create(name, displayName, type, (string)o["value"]);

                case "text":
                    return DropElement.Create(name, displayName, type, o);
                
                case "boolean":
                    return DropElement.Create(name, displayName, type, (string)o["value"]);
                
                case "float":
                    return DropElement.Create(name, displayName, type, (string)o["value"]);

                case "media":
                    return DropElement.Create(name, displayName, type, o);

                case "location":
                    return DropElement.Create(name, displayName, type, o);

                case "date":
                    return DropElement.Create(name, displayName, type, (string)o["value"]);
                
                case "url":
                    return DropElement.Create(name, displayName, type, (string)o["value"]);
                
            }

            throw new ModelLoadException("Bad input for create drop element: " + (string)o);
        }

    }

    /// <summary>
    /// A Drop element for Flow Text and String types
    /// </summary>
    public class TextDropElement : DropElement
    {
        protected string value;

        public TextDropElement(string name, string displayName, string type, bool required = false, bool inferred = false)
            : base(name, displayName, type, required, inferred)
        {

        }


        /// <summary>
        /// Sets the value of this
        /// </summary>
        /// <param name="value">Either a jobject or a toStringable</param>
        /// <returns></returns>
        public override bool TrySetValue(object value)
        {
            if (value is JObject)
            {
                JObject jo = (JObject)value, jo2, jo3;

                JToken v1, v2, v3;

                if (jo.TryGetValue("value", out v1))
                {
                    if ((jo2 = (v1 as JObject)) != null)
                    {
                        if (jo2.TryGetValue("content", out v2))
                        {
                            if ((jo3 = (v2 as JObject)) != null)
                            {
                                if (jo3.TryGetValue("value", out v3))
                                    this.value = (string)v3;
                                else
                                    return false;
                            }
                            else
                            {
                                this.value = (string)v2;
                            }
                        }
                        else {
                            this.value = (string)v1;
                        }
                    }
                    else
                    {
                        this.value = (string)v1;
                    }
                }
                else
                {
                    return false;
                }
            }
            else
            {
                this.value = value.ToString();
            }
            return true;
        }


        /// <summary>
        /// Returns this object's value as text
        /// </summary>
        /// <returns></returns>
        public override object GetValue()
        {
            string val = this.value;

            return val;
        }


        /// <summary>
        /// Returns the value of this element as a jobjectable item - override for objects where
        /// getvalue returns a non-jobject-serializable value
        /// </summary>
        /// <returns></returns>
        override public object ValueAsJObjectable()
        {
            if ("string".Equals(this.type))
            {
                return this.value;
            }
            else if ("text".Equals(this.type))
            {
                return new JObject(new JProperty("format", "html"),
                    new JProperty("content", this.value),
                    new JProperty("safe", false));
            }

            // put this back when all types are implemented
            //throw new ArgumentException("Bad type for item " + this.type);
            return this.GetValue();
        }


        /// <summary>
        /// Returns this as a string
        /// </summary>
        /// <returns></returns>
        public override string ToString()
        {
            return this.value;
        }
    }

    /// <summary>
    /// A Drop element for Flow boolean type
    /// </summary>
    public class BoolDropElement : DropElement
    {
        protected bool value;

        public BoolDropElement(string name, string displayName, string type, bool required = false, bool inferred = false)
            : base(name, displayName, type, required, inferred)
        {


        }


        /// <summary>
        /// Tries to set the value of this
        /// </summary>
        /// <param name="value">Either a jobject or a bool</param>
        /// <returns></returns>
        public override bool TrySetValue(object value)
        {
            if (value is string)
            {
                try
                {
                    this.value = bool.Parse((string)value);
                    return true;
                }
                catch
                {

                }

            }
            else if (value is bool)
            {
                this.value = (bool)value;
                return true;
            }

            return false;
        }


        /// <summary>
        /// Returns the value
        /// </summary>
        /// <returns></returns>
        public override object GetValue()
        {
            return this.value;
        }


        /// <summary>
        /// Returns this as a string
        /// </summary>
        /// <returns></returns>
        public override string ToString()
        {
            return this.value.ToString();
        }

    }

    /// <summary>
    /// A Drop element for Flow float type
    /// </summary>
    public class FloatDropElement : DropElement
    {
        protected double value;

        public FloatDropElement(string name, string displayName, string type, bool required = false, bool inferred = false)
            : base(name, displayName, type, required, inferred)
        {

        }


        /// <summary>
        /// Try to set the value of this
        /// </summary>
        /// <param name="value">Either a string or a double</param>
        /// <returns></returns>
        public override bool TrySetValue(object value)
        {
            if (value is string)
            {
                try
                {
                    this.value = double.Parse((string)value);
                    return true;
                }
                catch
                {

                }

            }
            else if (value is double)
            {
                this.value = (double)value;
                return true;
            }

            return false;
        }


        /// <summary>
        /// Returns the value
        /// </summary>
        /// <returns></returns>
        public override object GetValue()
        {
            return this.value;
        }


        /// <summary>
        /// Returns this as a string
        /// </summary>
        /// <returns></returns>
        public override string ToString()
        {
            return this.value.ToString();
        }
    }

    /// <summary>
    /// A Drop element for flow Date type
    /// </summary>
    public class DateDropElement : DropElement
    {
        protected long value;

        public DateDropElement(string name, string displayName, string type, bool required = false, bool inferred = false)
            : base(name, displayName, type, required, inferred)
        {


        }


        /// <summary>
        /// Set the value of this
        /// </summary>
        /// <param name="value">Either a string with a number representing time since epoch, or the number 
        /// itself</param>
        /// <returns></returns>
        public override bool TrySetValue(object value)
        {
            if (value is string)
            {
                try
                {
                    this.value = long.Parse((string)value);
                    return true;
                }
                catch
                {

                }

            }
            else if (value is long || value is int)
            {
                this.value = (long)value;
                return true;
            }

            return false;
        }


        /// <summary>
        /// Return this value
        /// </summary>
        /// <returns></returns>
        public override object GetValue()
        {
            return this.value;
        }


        /// <summary>
        /// Return this as a string
        /// </summary>
        /// <returns></returns>
        public override string ToString()
        {
            return this.value.ToString();
        }
    }

    /// <summary>
    /// A Drop element for the Flow media type
    /// </summary>
    public class MediaDropElement : DropElement
    {
        protected Media media;

        public MediaDropElement(string name, string displayName, string type, bool required = false, bool inferred = false)
            : base(name, displayName, type, required, inferred)
        {
            media = new Media();
        }


        /// <summary>
        /// Try to set the value (see TrySetValue in Media)
        /// </summary>
        /// <param name="value"></param>
        /// <returns></returns>
        public override bool TrySetValue(object value)
        {
            if (value is Media)
            {
                this.media = (Media)value;
                return true;
            }
            else
            {
                return media.TrySetValue(value);
            }
        }


        /// <summary>
        /// Returns an HTML representation of this
        /// </summary>
        /// <returns></returns>
        public override string AsHTML()
        {
            return "<div class=\"param\">" +
                "<div class=\"param-name\">" + this.GetDisplayName() + "</div>" +
                "<div class=\"param-value\">" + this.media.AsHTML() + "</div>" +
                "</div>";
        }


        /// <summary>
        /// Returns this value
        /// </summary>
        /// <returns></returns>
        public override object GetValue()
        {
            return this.media;
        }


        /// <summary>
        /// Returns the value of this element as a jobjectable item - override for objects where
        /// getvalue returns a non-jobject-serializable value
        /// </summary>
        /// <returns></returns>
        override public object ValueAsJObjectable()
        {
            return this.media.AsJObject();
        }


        /// <summary>
        /// Returns this as a string
        /// </summary>
        /// <returns></returns>
        public override string ToString()
        {
            return this.media.ToString();
        }
    }

    /// <summary>
    /// A Drop element for the Flow location type
    /// </summary>
    public class LocationDropElement : DropElement
    {

        protected float lat = 0, lon = 0;

        public LocationDropElement(string name, string displayName, string type, bool required = false, bool inferred = false)
            : base(name, displayName, type, required, inferred)
        {

        }


        /// <summary>
        /// Sets the value for the location object
        /// </summary>
        /// <param name="value">Must be of type Tuple(lat,lon)</param>
        /// <returns></returns>
        public override bool TrySetValue(object value)
        {
            if (value is JObject)
            {
                JObject jo = (JObject)value, jo2, jo3;

                JToken v1, v2, v3;

                if (jo.TryGetValue("value", out v1))
                {
                    if ((jo2 = (v1 as JObject)) != null)
                    {
                        // lat
                        if (jo2.TryGetValue("lat", out v2))
                        {
                            if ((jo3 = (v2 as JObject)) != null)
                            {
                                if (jo3.TryGetValue("value", out v3))
                                    this.lat = (float)v3;
                                else
                                    return false;
                                }
                            else
                            {
                                this.lat = (float)v2;
                            }
                        }
                        else
                        {
                            return false;
                        }

                        // lon
                        if (jo2.TryGetValue("lon", out v2))
                        {
                            if ((jo3 = (v2 as JObject)) != null)
                            {
                                if (jo3.TryGetValue("value", out v3))
                                    this.lon = (float)v3;
                                else
                                    return false;
                            }
                            else
                            {
                                this.lat = (float)v2;
                            }
                        }
                        else
                        {
                            return false;
                        }

                    }
                    else
                    {
                        return false;
                    }
                }
                else
                {
                    return false;
                }
            }
            else
            {
                try
                {
                    Tuple<float, float> ll = (Tuple<float, float>)value;
                    this.lat = ll.Item1;
                    this.lon = ll.Item2;
                }
                catch
                {
                    return false;
                }
            }

            return true;
        }


        /// <summary>
        /// Returns the value of this element as a jobjectable item - override for objects where
        /// getvalue returns a non-jobject-serializable value
        /// </summary>
        /// <returns></returns>
        override public object ValueAsJObjectable()
        {
            return new JObject(
                new JProperty("lat",
                    new JObject(
                        new JProperty("type", "float"),
                        new JProperty("value", this.lat))),
                new JProperty("lon",
                    new JObject(
                        new JProperty("type", "float"),
                        new JProperty("value", this.lon))));

        }


        /// <summary>
        /// Return this as html
        /// </summary>
        /// <returns></returns>
        public override string AsHTML()
        {
            return "<div class=\"param\">" +
                "<div class=\"param-name\">" + this.GetDisplayName() + "</div>" +
                "<div class=\"param-value\">(" + this.lat + "," + this.lon + ")</div>" +
                 "</div>";
        }


        /// <summary>
        /// Return this value
        /// </summary>
        /// <returns></returns>
        public override object GetValue()
        {
            return new Tuple<float, float>(this.lat, this.lon);
        }


        /// <summary>
        /// Return this as a string
        /// </summary>
        /// <returns></returns>
        public override string ToString()
        {
            return "location";
        }
    }

    /// <summary>
    /// A Drop element for the Flow media type
    /// </summary>
    public class UrlDropElement : DropElement
    {
        protected string url;

        public UrlDropElement(string name, string displayName, string type, bool required = false, bool inferred = false)
            : base(name, displayName, type, required, inferred)
        {

        }


        /// <summary>
        /// Try to set the value of this
        /// </summary>
        /// <param name="value">a url string</param>
        /// <returns></returns>
        public override bool TrySetValue(object value)
        {
            this.url = (string)value;
            return true;
        }


        /// <summary>
        /// Return this as html
        /// </summary>
        /// <returns></returns>
        public override string AsHTML()
        {
            return "<div class=\"param\">" +
                "<div class=\"param-name\">" + this.GetDisplayName() + "</div>" +
                "<div class=\"param-value\">" + this.url + "</div>" +
                "</div>";
        }


        /// <summary>
        /// Return this URL
        /// </summary>
        /// <returns></returns>
        public override object GetValue()
        {
            return this.url;
        }


        /// <summary>
        /// Return this as a string
        /// </summary>
        /// <returns></returns>
        public override string ToString()
        {
            return this.url;
        }
    }
}
```

### Create a user (requires a superuser actor) (python)
```python

  def create_user(self, firstName, lastName, email, password, eflowId, alias, client):

    user = {
      "firstName" : firstName,
      "lastName" : lastName,
      "middleName" : "",
      "name" : alias,
      "email" : email,
      "password" : password,
      "eflowId" : eflowId
      }

    result = json.loads(client.http_post("/identity.json", json.dumps(user)))

    if not result['head']['ok'] and len(result['head']['errors']) > 0:
      raise Exception(result['head']['errors'][0][1])

    return result['body']['id']
```

