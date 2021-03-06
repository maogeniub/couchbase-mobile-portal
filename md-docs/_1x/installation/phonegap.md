---
id: phonegap
title: PhoneGap
permalink: installation/phonegap/index.html
---

## Add Couchbase Lite to your app

### PhoneGap plugin registry

1. Navigate to the root of an existing PhoneGap project.
2. Run the following command to install the plugin.

	```bash
	phonegap local plugin add https://github.com/couchbaselabs/Couchbase-Lite-PhoneGap-Plugin.git
	```
	
3. You can access the Couchbase Lite Listener URL in the `receivedEvent` lifecycle method with the following.

	```javascript
	if (window.cblite) {
		window.cblite.getURL(function (err, url) {
			if (err) {
				console.log("error launching Couchbase Lite: " + err)
			} else {
				console.log("Couchbase Lite running at " + url);
			}
		});
	} else {
		console.log("error, Couchbase Lite plugin not found.");
	}
	```

4. Build and run.

## Getting Started

### Creating a new project

Create a new PhoneGap project using the PhoneGap CLI. Then, install the Couchbase Lite plugin.

```bash
npm install -g phonegap
phonegap create PhoneGap-Example
cd PhoneGap-Example
phonegap local plugin add https://github.com/couchbaselabs/Couchbase-Lite-PhoneGap-Plugin.git
```

Add the platforms where you plan to have the application running.

```bash
phonegap platform add android
phonegap platform add ios
```

> Note: On iOS, you must have the **ios-sim** module installed globally to start the emulator from the command line `npm install -g ios-sim && phonegap run ios`.

### Adding Couchbase Lite

The Couchbase Lite Listener exposes the same functionality as the native SDKs through a common RESTful API. You can perform the same operations on the database by sending HTTP requests to it. In this example, you'll use the Swagger JS client with the Couchbase Lite REST API spec to perform various operations.

- [Download the Swagger JS client](http://couchbase-docs.s3.amazonaws.com/assets/swagger-js/2.x/swagger-client.min.js) to a new file under **www/js/swagger-client.min.js**.
- [Download the Couchbase Lite REST API spec]({{ site.swagger_url }}) to a new file **www/js/couchbase-lite.js**. Your IDE might show an error because you've copied a JSON object into a JavaScript file but don't worry, prepend the content with the following to set the spec on the `window` object as you will need a reference to it later to initialize the Swagger client.

	```javascript
	window.spec = {
		"swagger": "2.0",
		"info": {
			"title": "Couchbase Lite",
			...
		}
	}
	```
  Here, you're embedding the API spec as part of the application so that the JS library can also work offline.
- Create an new file under **www/js/data-manager.js**. You will shortly add the code to create the database and insert a document.
  
Reference the 3 files you created above in **www/index.html** before `app.initialize()` is executed.

```html
<script type="text/javascript" src="js/swagger-client.min.js"></script>
<script type="text/javascript" src="js/couchbase-lite.js"></script>
<script type="text/javascript" src="js/data-manager.js"></script>
<script type="text/javascript">
    app.initialize();
</script>
```

Then, open **www/js/data-manager.js** and add the following.

```javascript
var DB_NAME = 'todo';

function initRESTClient(url) {
  var client = new SwaggerClient({
    spec: window.spec,
    usePromise: true,
  })
    .then(function (client) {
      client.setHost(url);
      if (device.platform == 'android') {
        var encodedCredentials = "Basic " + window.btoa(url.split('/')[1].split('@')[0]);
        client.clientAuthorizations.add("auth", new SwaggerClient.ApiKeyAuthorization('Authorization', encodedCredentials, 'header'));
      }
      client.server.get_all_dbs()
        .then(function (res) {
          var dbs = res.obj;
          if (dbs.indexOf(DB_NAME) == -1) {
            return client.database.put_db({db: DB_NAME});
          }
          return client.database.get_db({db: DB_NAME});
        })
        .then(function (res) {
          return client.document.post({db: DB_NAME, body: {title: 'Couchbase Mobile', sdk: 'PhoneGap'}});
        })
        .then(function (res) {
          console.log('Document ID :: ' + res.obj.id);
        })
        .catch(function (err) {
          console.log(err);
        });
    });

}
```

This code initializes the REST API client with the url passed as a parameter. The promise chaining then creates the database and inserts a document. To find out what the available methods are, use the `help()` method to print the list of parameters for each endpoint (for example, `client.help()`, `client.database.help()` etc.)

In the next section, you will write some code to detect if the app is running on a desktop browser (i.e during development) or on the device/emulator.

To call this method, open **www/js/index.js** and append the following in the `onDeviceReady` method.

```javascript
if (window.cblite && window.cordova.platformId == 'browser') { // desktop browser, development only.
	initRESTClient('localhost:59840');
} else if (window.cblite) {
	window.cblite.getURL(function (err, url) {
		if (err) {
			console.log("error launching Couchbase Lite: " + err)
		} else {
			console.log("Couchbase Lite running at " + url);
			initRESTClient(url.split('/')[2]);
		}
	});
} else {
	console.log("error, Couchbase Lite plugin not found.");
}
```

The code above initializes the application by calling the `initRESTClient` with the URL to access the Couchbase Lite Listener.

### Development environment

While the application will usually run on a device, it is useful to know how to run it in a desktop browser. This is useful during development because you won't have to re-deploy the application to a device or emulator on every change.

Download [LiteServ for macOS](https://cl.ly/1d1k261T3d2s/LiteServ.zip), unzip the folder and start it from the command line with the CORS flag.

```bash
cd LiteServ
./LiteServ.app/Contents/Tools/LiteServ --cors
```

Run the following command to serve the PhoneGap application from a static web server.

```bash
phonegap serve --browser
```

Open [localhost:3000](http://localhost:3000/) in a web browser and notice that the document is successfully persisted to the database.

![](../img/web-browser.png)

### Deploying

Build and run for Android.

```bash
phonegap run android
```

Open [chrome://inspect](chrome://inspect) in Chrome and select the Android device where the app is running. Notice the document ID and property are printed to the console. The document was successfully persisted to the database.

![](../img/phonegap-console-android.png)

The application won't run as is on iOS just yet. You will need to modify the `Content-Security-Policy` header. Open **www/index.html** and replace the `<meta http-equiv="Content-Security-Policy" ...` header with the following.

```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self' gap://ready file://* *; style-src 'self' 'unsafe-inline'; script-src 'self' 'unsafe-inline' 'unsafe-eval'">
```
 
Build and run for iOS.

```bash
phonegap run ios
```

Open Safari and select the **Develop\Simulator\index.html** menu to open the console. It will be empty because the code already ran but you can enter `window.location.reload()` to re-run it. Notice the document ID and property are printed to the console. The document was successfully persisted to the database.

![](../img/phonegap-console-ios.png)

{% include next.html %}