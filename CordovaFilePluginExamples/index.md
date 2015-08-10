# Cordova File Plugin Examples

I found myself in need of writing potentially large amounts of data to the file system in a Cordova app I'm working on. Naturally, I reached for the [Cordova File Plugin](https://github.com/apache/cordova-plugin-file), as a handful of cursory Google searches revealed it to be the *de facto* plugin for this task. However, its documentation, while talking about the plugin's quirks, is lacking in examples, and the main [blog post it points to](http://www.html5rocks.com/en/tutorials/file/filesystem/) is old, and not all of it is relevant the plugin. In this post, I aim to detail what I have learned, to make it easier for other people to get started with this useful plugin.

## The API Has It

A great API is simple, and a joy to work with. Here is an example:

```javascript
localStorage.setItem('myKey', JSON.stringify({ my: data }, null, '\t'));

var data = localStorage.getItem('myKey');
if (data) {
	data = JSON.parse(data);
}
```

Now, admittedly, it's a bit of a shame that the object to JSON conversion isn't done automatically, but all in all, the localStorage API is succint and developer friendly. With the Cordova File Plugin, things get more complicated.

## HTML5 File API Et Al

The File Plugin implements the [HTML5 File API](http://www.w3.org/TR/FileAPI/), plus several others, including one that is since defunct. The result is a very versatile plugin, with low-level access to much functionality. The flip-side is that basic read/write operations (arguably the lion's share of all use cases), are not obvious, nor their usage consistent.

As I mentioned above, the [HTML5 Rocks FileSystem](http://www.html5rocks.com/en/tutorials/file/filesystem/) article is linked to for usage examples. That article concerns itself with how one would use the API in a browser. What it doesn't say on that article, or on the plugin documentation, is that the plugin takes care of some of the boilerplate you have to content with, when trying to use the File API in a browser.

## Case In Point

In a browser, the File API requires you to firstly:

1. request a filesystem,
2. specify whether it is to be temporary, or persistent,
3. specify its size,
4. specify success and error handlers for this request.

```javascript
function onInitFs(fs) {
	console.log('Opened file system: ' + fs.name);
}

window.requestFileSystem(window.PERSISTENT, 50*1024*1024 /*50MB*/, onInitFs, errorHandler);
```

Furthermore, if you requested a persistent filesystem, you also need to request storage quota from the user:

```javascript
window.webkitStorageInfo.requestQuota(PERSISTENT, 1024*1024, function(grantedBytes) {
	window.requestFileSystem(PERSISTENT, grantedBytes, onInitFs, errorHandler);
}, function (e) {
	console.log('Error', e);
});
```

## The Important Part

Here is what neither the HTML5 Rocks article (sensibly, since its concern is the browser API), nor the [plugin documentation](https://github.com/apache/cordova-plugin-file#cordova-plugin-file), nor some of the [very few example usage articles](http://stackoverflow.com/questions/22336352/how-do-you-store-a-file-locally-using-apache-cordova-3-4-0), say:

> You don't need to do either of those two steps, when using the Cordova File Plugin.

Only this [collection](http://www.raymondcamden.com/2014/07/15/Cordova-Sample-Reading-a-text-file) of [example use cases](http://www.raymondcamden.com/2014/11/05/Cordova-Example-Writing-to-a-file) hints at their superflousness.

With the Cordova File Plugin, there are two essential pieces of information:

1. Like all Cordova plugins, you have to wait for the `deviceready` event before you try anything,
2. Then, `window.resolveLocalFileSystemURL(<path>, <successHandler>, <errorHandler>)` is your friend.

`window.resolveLocalFileSystemURL()` returns a `FileEntry` or `DirectoryEntry` instance (depending on whether you gave a file or a directory as path as its first parameter), which you can then work with.

## Reading the Contents of a File

Here is the first of the two main use cases (the other one being writing a file). For this example we assume a plain text file with some JSON in it. It's pretty much the same for any other plain text file, whereas it gets a little more complicated should you wish to read binary data.

```javascript
document.addEventListener('deviceready', onDeviceReady, false);
function onDeviceReady() {
	var fileName = 'data.json',
		pathToFile = cordova.file.dataDirectory + fileName;
	window.resolveLocalFileSystemURL(pathToFile, function (fileEntry) {
		fileEntry.file(function (file) {
			var reader = new FileReader();

			reader.onloadend = function (e) {
				return JSON.parse(this.result);
			};

			reader.readAsText(file);
		}, errorHandler.bind(null, fileName));
	}, errorHandler.bind(null, fileName));
}
```

Let's have a look at a few of the things going on in this snippet.

Firstly, we wait for the `deviceready` event. In you're app, you likely have done this already way before you start any reading or writing of files, but it's important to remember.

Secondly, we call `window.resolveLocalFileSystemURL()` with the magic `cordova.file.dataDirectory` value that the File Plugin exposes for us. This is one of several values for the various paths your app can access. They are [well documented](https://github.com/apache/cordova-plugin-file#where-to-store-files) on the Plugin page. In this case, it's a private data directory within your app's filesystem that won't sync with iCloud. (If you want this iOS only functionality, use `cordova.file.syncedDataDirectory`.)

Then come a bunch of complicated, confusingly labelled functions, instances, and event handlers. Essentially though, we're calling `.readAsText()` on our file pointer, after having set up an event handler for the `loadend` event, which will be triggered when `.readAsText()` reaches the end of the file. Its parameter is the event object, and the `this` context contains our data in its `result`property`. You can also find it at `e.target.result`.

Makes you a little jealous of localStorage's `getItem()` method, doesn't it? :)

### What's With That Error Handler?

You might have noticed that `errorHandler.bind(null, fileName)` parameter on two of the function calls. Here is the function, much the same as in the HTML5 Rocks article, with one improvement.

```javascript
var errorHandler = function (fileName, e) {
	var msg = '';

	switch (e.code) {
		case FileError.QUOTA_EXCEEDED_ERR:
			msg = 'QUOTA_EXCEEDED_ERR';
			break;
		case FileError.NOT_FOUND_ERR:
			msg = 'NOT_FOUND_ERR';
			break;
		case FileError.SECURITY_ERR:
			msg = 'SECURITY_ERR';
			break;
		case FileError.INVALID_MODIFICATION_ERR:
			msg = 'INVALID_MODIFICATION_ERR';
			break;
		case FileError.INVALID_STATE_ERR:
			msg = 'INVALID_STATE_ERR';
			break;
		default:
			msg = 'Unknown Error';
			break;
	};

	console.log('Error (' + fileName + '): ' + msg);
}
```

By itself, any error handler you specify is passed an error object as parameter, which holds little more than an error code. We're calling `.bind(null, fileName)` on the function when we specify it as the error handler, so that its first parameter becomes the filename instead. (The `null` value is so that errorHandler retains its own context.)

## Writing a File

Here then is the other obvious use case: writing some data to a file. In this example, we'll write some JSON. Here is the code:

```javascript
document.addEventListener('deviceready', onDeviceReady, false);
function onDeviceReady() {
	function writeToFile(fileName, data) {
		data = JSON.stringify(data, null, '\t');
		window.resolveLocalFileSystemURL(cordova.file.dataDirectory, function (directoryEntry) {
			directoryEntry.getFile(fileName, { create: true }, function (fileEntry) {
				fileEntry.createWriter(function (fileWriter) {
					fileWriter.onwriteend = function (e) {
						console.log('Write of file ' + fileName + '.json completed.');
					};

					fileWriter.onerror = function (e) {
						console.log('Write failed: ' + e.toString());
					};

					var blob = new Blob([data], { type: 'text/plain' });
					fileWriter.write(blob);
				}, errorHandler.bind(null, fileName));
			}, errorHandler.bind(null, fileName));
		}, errorHandler.bind(null, fileName));
	}

	writeToFile('example.json', { foo: 'bar'});
}
```

A lot going on there, let's break it down.

For convenience, we're defining `writeToFile()` as a re-usable function. We're opening a connection to the dataDirector, which returns a `directoryEntry` instance, with a `getFile()` method. We're calling this method with a filename, and - importantly - a configuration object that








