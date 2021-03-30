# OPNsense templates for Zabbix

**Currently a work-in-progress.**

Even though there appears to be at least one set of [templates for *pfSense*](https://github.com/rbicelli/pfsense-zabbix-template), I found none for *OPNsense.*
So, countless house of digging through the (in my opinion, poorly documented) API, and the result is this.
Almost all checks are done using HTTP items, meaning the Zabbix server (or Zabbix proxy) is making a standard HTTP request.
A few checks are done with actual agent items, of which there's two templates: one for "passive" checks and one for "active" checks.
All the information I think you might need (or want) to change / customize have all been put into template macros.

## A Note on Zabbix HTTP Items

As mentioned, everything from the API is fetched with HTTP.
OPNsense requires authentication to access the API, which comes in two parts: the API key, and the API secret, both of which are 80 characters long.
In practice, the API key is the "username" and the API secret is the "password" as far as HTTP authentication is concerned.
Well, Zabbix does not support usernames / passwords for HTTP auth that exceed 64 characters in length. [I've filed a bug for this](https://support.zabbix.com/browse/ZBX-19181) (If you can ignore my inability to type), but what this means is that until then, you will have to manually calculate the data blob that's used for HTTP auth.
The script `basic-auth-encode.py` here can be used to automatically generate the needed string, if you give it the INI file that OPNsense generated when you generate an API key, like so:

```shell
$ ./basic-auth-encode.py apikey.txt
```

If you don't have that file, you can either make one:

```
key=<80 character API key>
secret=<80 character API secret>
```

Or you can just do the conversion yourself.
The final value you need is the base64-encoded string "`$key:$secret`" where `$key` and `$secret` are replaced by your API key and API secret.
Really, copy and paste work fine if you're in a terminal:

```shell
$ echo "an-api-key:an-api-secret" | base64
```

The result is either of those is going to be some 200 characters of data, copy this into the `{$API.AUTH}` macro on the Zabbix.

~~When~~ **If** this gets "fixed", then we can just disregard *all of this* and put the key and secret into `{$API.KEY}` and `{$API.SECRET}`, respectively.

## Active vs. Passive checks

As mentioned, one template has *active* item checks and the other has *passive* item checks.
The difference is that **passive** checks (the usual default) are where the Zabbix server (or proxy) requests a particular item from the host's agent, and it responds with a value.
The burden of retrieving the value is on the server.
An **active** check is where the *host's agent* will periodically connect to the server, and download a list of active items.
After this, every time the item is supposed to be updated, it will send the value to the server to be processed.
The burden of providing the value rests on the host, not the server.

In other words, active checks will use a bit of CPU on the host you want to monitor, and passive checks will use a bit of CPU on the Zabbix server itself.
Use whichever you prefer.
