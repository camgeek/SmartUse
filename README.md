# Welcome to SmartUse!
----


## Get started

### Obtain an Access Token

```
curl -X POST -k -H 'Content-Type: application/x-www-form-urlencoded' -i 'https://developer.api.autodesk.com/authentication/v1/authenticate' --data 'client_id=95XcJdJzsD9i7IodzhL5C9TkYLhklKyn&client_secret=rDpDG3sgs3FGz1Ej&grant_type=client_credentials&scope=bucket:create bucket:read data:read data:write' 
``` 

The token labeled <span style="color:blue">*access_token*</span>, obtained by this command, will be used in the next steps.

## Manage buckets

### Create a bucket

```
curl -v 'https://developer.api.autodesk.com/oss/v2/buckets' -X 'POST' -H 'Content-Type: application/json' -H 'Authorization: Bearer ${access_token}' -d '{"bucketKey":"${bucket_name}","policyKey":"transient"}'
```

Here, we’re creating a <span style="color:green">transient</span> bucket, meaning that anything uploaded to it will be deleted after 24 hours.

### Check bucket creation

```
curl -v 'https://developer.api.autodesk.com/oss/v2/buckets/${bucket_name}/details' -X 'GET' -H 'Authorization: Bearer ${access_token}'
```

If successful, the response body should look something like this:

```
{
  "bucketKey": "${bucket_name}",
  "bucketOwner": "obQDn8P0GanGFQha4ngKKVWcxwyvFAGE",
  "createdDate": 1401735235495,
  "permissions": [
    {
      "authId": "obQDn8P0GanGFQha4ngKKVWcxwyvFAGE",
      "access": "full"
    }
  ],
  "policyKey": "transient"
}
```


## Manage files

### Upload a file

```
curl -v 'https://developer.api.autodesk.com/oss/v2/buckets/${bucket_name}/objects/${filename}.3ds' -X 'PUT' -H 'Authorization: Bearer ${access_token}' -H 'Content-Type: application/octet-stream' -H 'Content-Length: ${file_length}' -T '${filename}.3ds'
```
If successful, the response body should look something like this:

```
{
  "bucketKey" : "${bucket_name}",
  "objectId" : "urn:adsk.objects:os.object:${bucket_name}/${filename}.3ds",
  "objectKey" : "${filename}.3ds",
  "sha1" : "e84021849a9f5d1842bf792bbcbc6445c280e15b",
  "size" : 308331,
  "content-type": "application/octet-stream",
  "location": "https://developer.api.autodesk.com/oss/v2/buckets/${bucket_name}/objects/${filename}.3ds"
}
```

### Convert the source URN into a Base64-Encoded URN

You need to encode, with [this online tool](http://www.freeformatter.com/base64-encoder.html), the source URN that you retrieved when calling the previous command.


### Translate the Source File into SVF Format

```
curl -X 'POST' -H 'Authorization: Bearer ${acess_token}' -H 'Content-Type: application/json' -v 'https://developer.api.autodesk.com/modelderivative/v2/designdata/job' -d '{"input": {"urn": "${base64_encoded_urn}"},"output": {"formats": [{"type": "svf","views": ["2d","3d"]}]}}'
```

Note that this endpoint is asynchronous and initiates a job that runs in the background, rather than halting execution of your program.

### Verify the Job is Complete

```
curl -X 'GET' -H 'Authorization: Bearer ${acess_token}' -v 'https://developer.api.autodesk.com/modelderivative/v2/designdata/${base64_encoded_urn}/manifest'
```

The status parameter returns a string with one of the following values:

* pending: The job has been received and is pending for processing.
* inprogress: The job has started processing, and is running.
* success: The job has finished successfully.
* failed: The translation has failed.
* timeout: The translation has timed out and no output is generated.

## HTML


```
<head>
    <meta name="viewport" content="width=device-width, minimum-scale=1.0, initial-scale=1, user-scalable=no" />
    <meta charset="utf-8">

    <!-- The Viewer CSS -->
    <link rel="stylesheet" href="https://developer.api.autodesk.com/modelderivative/v2/viewers/style.min.css" type="text/css">

    <!-- Developer CSS -->
    <style>
        body {
            margin: 0;
        }
        #MyViewerDiv {
            width: 100%;
            height: 100%;
            margin: 0;
            background-color: #F0F8FF;
        }
    </style>
</head>
<body>

    <!-- The Viewer will be instantiated here -->
    <div id="MyViewerDiv"></div>

    <!-- The Viewer JS -->
    <script src="https://developer.api.autodesk.com/modelderivative/v2/viewers/three.min.js"></script>
    <script src="https://developer.api.autodesk.com/modelderivative/v2/viewers/viewer3D.min.js"></script>

    <!-- Developer JS -->
    <script>
        var viewer;
        var options = {
        	env: 'AutodeskProduction',
        	accessToken: '${acess_token}'
		    /*
		    With token expiration
		    getAccessToken: function(onGetAccessToken) {
		        //
		        // TODO: Replace static access token string below with call to fetch new token from your backend
		        // Both values are provided by Forge's Authentication (OAuth) API.
		        //
		        // Example Forge's Authentication (OAuth) API return value:
		        // {
		        //    "access_token": "${acess_token}",
		        //    "token_type": "Bearer",
		        //    "expires_in": ${expire_time}
		        // }
		        //
		        var accessToken = '${acess_token}';
		        var expireTimeSeconds = ${expire_time};
		        onGetAccessToken(accessToken, expireTimeSeconds);
		     }
		     */
        };
        var documentId = 'urn:${base64_encoded_urn}';
        Autodesk.Viewing.Initializer(options, function onInitialized(){
            Autodesk.Viewing.Document.load(documentId, onDocumentLoadSuccess, onDocumentLoadFailure);
        });

        /**
        * Autodesk.Viewing.Document.load() success callback.
        * Proceeds with model initialization.
        */
        function onDocumentLoadSuccess(doc) {

            // A document contains references to 3D and 2D viewables.
            var viewables = Autodesk.Viewing.Document.getSubItemsWithProperties(doc.getRootItem(), {'type':'geometry'}, true);
            if (viewables.length === 0) {
                console.error('Document contains no viewables.');
                return;
            }

            // Choose any of the avialble viewables
            var initialViewable = viewables[0];
            var svfUrl = doc.getViewablePath(initialViewable);
            var modelOptions = {
                sharedPropertyDbPath: doc.getPropertyDbPath()
            };

            var viewerDiv = document.getElementById('MyViewerDiv');
            viewer = new Autodesk.Viewing.Private.GuiViewer3D(viewerDiv);
            viewer.start(svfUrl, modelOptions, onLoadModelSuccess, onLoadModelError);
        }

        /**
         * Autodesk.Viewing.Document.load() failuire callback.
         */
        function onDocumentLoadFailure(viewerErrorCode) {
            console.error('onDocumentLoadFailure() - errorCode:' + viewerErrorCode);
        }

        /**
         * viewer.loadModel() success callback.
         * Invoked after the model's SVF has been initially loaded.
         * It may trigger before any geometry has been downloaded and displayed on-screen.
         */
        function onLoadModelSuccess(model) {
            console.log('onLoadModelSuccess()!');
            console.log('Validate model loaded: ' + (viewer.model === model));
            console.log(model);
        }

        /**
         * viewer.loadModel() failure callback.
         * Invoked when there's an error fetching the SVF file.
         */
        function onLoadModelError(viewerErrorCode) {
            console.error('onLoadModelError() - errorCode:' + viewerErrorCode);
        }

    </script>
</body>
```



