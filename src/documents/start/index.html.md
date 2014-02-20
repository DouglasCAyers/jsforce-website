---
layout: default
category: start
title: Getting Started
subtitle: Start app development using JSforce
sidebar: menu
---

## Overview

JSforce (f.k.a. Node-Salesforce) is a JavaScript Library of Salesforce API, which works both on web browser and Node.js.

It capsulates the access to various APIs provided by Salesforce in asynchronous JavaScript function calls.

Unlike other Salesforce API libraries, it is intended to give integrated interface both server-side and client-side apps, 
so you don't have to rewrite similar logics with different libraries only for running in different environment.

Additionally, it has useful command line interface (CLI) which gives interactive console (REPL), 
so you can learn the usage without hassle.

### Supported APIs

- REST API ([SOQL](/document/#query), [SOSL](/document/#search), [CRUD](/document/#crud), [describe](/document/#describe), etc.)
- [Apex REST](/document/#apex-rest)
- [Analytics API](/document/#analytics-api)
- [Bulk API](/document/#bulk-api)
- [Chatter API](/document/#chatter-api)
- [Metadata API](/document/#metadata-api)
- [Streaming API](/document/#streaming-api)
- [Tooling API](/document/#tooling-api)

## Setup

### Node.js

If you want to use JSforce as an API library in your Node.js project, install it via npm.

``` shell
$ npm install jsforce
```

When you installed JSforce, you can import using `require('jsforce')` in your Node.js application code.

``` javascript
var jsforce = require('jsforce');
var conn = jsforce.Connection();
```

Details for establishing API connection are written in [API Reference](/api/#connection).


### Web Browser

If you want to use JSforce in web browser JavaScript, 
[download JSforce JS library](/download/) and put it on any path in your website, and include in `<script>` tag:

```html
<script src="/path/to/jsforce.js"></script>
```

When the script is loaded, `jsforce` object will be defined in global root.

If your app is located outside of Salesforce domain (that is, non-Visualforce app), 
you need to register your app as "Connected App" in Salesforce.

So first you have to register your app as an OAuth2 client to get client ID.
You must input callback URL (used as OAuth2 redirect URI) to get registered,
and it must be in the same origin of your web app.

Then add JSforce initialization code in your html to tell the OAuth2 application information obtained in previous step.

```html
<script src="/path/to/jsforce.js"></script>
<script>
jsforce.browser.init({
  clientId: '[your OAuth2 client ID]',
  redirectUri: '[your OAuth2 redirect URI]' 
});

// ...

</script>
```

NOTE: You may need [jsforce-ajax-proxy](https://github.com/jsforce/jsforce-ajax-proxy) 
if the app resides outside of Salesforce (that is, non-Visualforce environment).

```javascript
jsforce.browser.init({
  clientId: '<your OAuth2 client ID>',
  redirectUri: '<your OAuth2 redirect URI>',
  proxyUrl: 'https://your-jsforce-proxy-server.herokuapp.com/proxy/'
});
```

To enforce users to login, you can call `jsforce.browser.login()` to start OAuth2 authorization flow by popping up window.
Afterward you can get authorized API connection by listening `connect` event on `jsforce.browser` object.

```javascript
jsforce.browser.on('connect', function(conn) {
  conn.query('SELECT Id, Name FROM Account', function(err, res) {
    if (err) { return handleError(err); }
    handleResult(res);
  });
});
```


### Visualforce

If you are writing HTML5 apps on Visualforce, API access token is automatically obtained as session ID.
You can just pass it to Connection constructor to initialize.
Also you can use static resources to upload JSforce JavaScript file to include.

```html
<apex:page docType="html-5.0" showHeader="false">
  <apex:includeScript value="{!URLFOR($Resources.JSforce)}" />
  <script>
var conn = new jsforce.Connection({ accessToken: '{!$API.Session_Id}' });
conn.query('SELECT Id, Name FROM Account', function(err, res) {
  if (err) { return handleError(err); }
  handleResult(res);
});

// ...

  </script>

</apex:page>
```

### Salesforce Canvas

You can use JSforce in Salesforce Canvas app.
In order to create authorized API connection, pass signed request JSON value to Connection constructor

Note that the signed request value must be validated in server-side before passing to JSforce.
Also you have to load canvas JS SDK officially provided from Salesforce.

```html
<!DOCTYPE html>
<html>
<head>
  <meta id="sf-canvas-signed-request" content="<%= verifiedSignedRequestJSON %>" />
  <script src="https://login.salesforce.com/canvas/sdk/js/29.0/canvas-all.js"></script>
  <script src="/js/jsforce.js"></script>
  <script>
var sr = document.getElementById('sf-canvas-signed-request').content;
var conn = new jsforce.Connection({ signedRequest: sr });
conn.query('SELECT Id, Name FROM Account', function(err, res) {
  if (err) { return handleError(err); }
  handleResult(res);
});

//...

  </script>
</head>
<body></body>
</html>
```


## Basic Usage

When you established the connection, you can call APIs defined in connection object.

By JavaScript's nature, all API calls are done asynchronously.

### Using Callback

As broadly used in other JavaScript APIs, you can pass a callback function to wait and get the async response.

The callback function has the same interface as normal Node.js callback, accepting two arguments - error and successful result.

```javascript
conn.query("SELECT Id, Name FROM Account LIMIT 10", function(err, res) {
  if (err) { return handleError(err); }
  handleResult(res);
});
```

### Using Promise

JSforce supports [Promise/A+](http://promises-aplus.github.io/promises-spec/) spec,
and almost all API calls return Promise object.
So you can chain the promise result by `then()` to handle the async response.

```javascript
conn.query("SELECT Id, Name FROM Account LIMIT 1")
  .then(function(res) {
    // receive resolved result from the promise, 
    // then return another promise for continuing API execution.
    return conn.sobject('Account').create({ Name: 'Another Account' });
  })
  .then(function(ret) { 
    // handle final result of API execution
    // ...
  }, function(err) {
    // catch any errors in execution
    // ...
  });
```

## Command Line Interface

There is a Command Line Interface (CLI) which works on tty.
To use JSforce CLI, install npm package globally.

```
$ npm install jsforce -g
```

After the install, `jsforce` command will be available in your path.

```
$ jsforce --help

  Usage: jsforce [options]

  Options:

    -h, --help                     output usage information
    -u, --username [username]      Salesforce username
    -p, --password [password]      Salesforce password (and security token, if available)
    -c, --connection [connection]  Connection name stored in connection registry
    -e, --evalScript [evalScript]  Script to evaluate
    --coffee                       Using CoffeeScript
```

When you just type `jsforce`, it enters into REPL mode.
In the REPL environment, default connection object in the context is exposed,
so you don't need to instantiate connection object.

```
$ jsforce
> login('user@example.org', 'password123');
{ id: '00550000000vwsFAAQ',
  organizationId: '00D500000006xKGEAY',
  url: 'https://login.salesforce.com/id/00D500000006xKGEAY/00550000000vwsFAAQ' }
> 
```

To login to Salesforce, type `.connect` with username.

```
> .connect user@example.org
Password: ********
Logged in as : username@example.org
> 
```

The established session information is stored in `~/.jsforce/config.json` file, 
so you can connect without password while the session is valid.

```
$ jsforce -c user@example.org
Logged in as : user@example.org
> 
```

By passing a script in `-e` option, it automatically evaluate the script and returns result in JSON.

```
$ jsforce -c user@example.org -e "query('SELECT Id, Name FROM Account LIMIT 1')"
{"totalSize":1,"done":true,"records":[{"attributes":{"type":"Account","url":"/services/data/v29.0/sobjects/Account/0015000000KBQ5GAAX"},"Id":"0015000000KBQ5GAAX","Name":"United Oil"}]}
```

In order to authorize the connection via OAuth2 authorization flow, type `.authorize` in REPL mode.
It will popup browser and start flow for API access authorization.

```
> .authorize
```

Note that you need to register your OAuth2 client information before start OAuth2 authorization.
The `.register` command will navigate the registration.
In order to accomplish authorization flow successfully, the redirect URL of registering client must be `http://localhost:<port>`.

```shell
> .register
Input client ID (consumer key) : <your client id>
Input client secret (consumer secret) : <your client secret>
Input redirect URI : <your client redirect uri>
Input login URL (default is https://login.salesforce.com) : 
Client registered successfully.
```

