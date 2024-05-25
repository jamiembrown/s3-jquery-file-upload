# Working solution to upload to an AWS S3 pre-signed post URL in Javascript with jQuery
Every time I've come across this problem I've ended up wasting hours, because AWS is so finicky with how to post to a pre-signed URL.

I always end up battling with obscure errors like:

```<Code>PreconditionFailed</Code><Message>At least one of the pre-conditions you specified did not hold</Message><Condition>Bucket POST must be of the enclosure-type multipart/form-data</Condition>```

or:

```<Code>SignatureDoesNotMatch</Code><Message>The request signature we calculated does not match the signature you provided. Check your key and signing method.</Message>```

or best of all, just a blank 403 response.

Most of the resources out there are for the `get_presigned_url()` feature, which returns a URL. But I wanted to use the newer boto3 `get_presigned_post()` feature, which returns a URL and a bunch of fields you need to include with the POST request.

So here's my 2024 rundown on how I got it working, both for me next time, and for others out there in the same boat.

My Python code generates a presigned URL as follows:

```python
client: boto3.session.Session.client = boto3.client('s3', region_name=os.getenv("AWS_REGION"), aws_access_key_id=os.getenv("AWS_KEY"), aws_secret_access_key=os.getenv("AWS_SECRET"))
response = client.generate_presigned_post(os.getenv("S3_PRIVATE_BUCKET"), object_name, ExpiresIn=1800)
return response
```

That returns a `dict` from Amazon like this:

```json
{
  "url": "https://urltobucket/",
  "fields": {
    "signature": "somevalue",
    "key": "somevalue",
    ...
  }
}
```

You now need to POST to the `url`, including both the file object and the fields that Amazon returned to you.

After lots of messing around I finally got a jQuery ajax request which successfully sends the file to the server, based on this data. Here it is:

```js
// Before this I'm posting to my own API to run the Python code,
// generate the presigned key, and return it as a dict called "data".

var file = $('#upload-file')[0].files[0]; // The original file object

// Create the form data object where we're going to put everything we need
var formData = new FormData();

// Add the fields that came back from Amazon to the FormData object
for (var key in data.fields) {
    if (data.fields.hasOwnProperty(key)) {
        formData.append(key, data.fields[key]);
    }
}

// Add the file to the FormData. Important that's it's called "file",
// AND that it is added after the fields above. If you put this before
// the fields in the form data it will not work!
formData.append("file", file, file.name);

// Make the POST request. Note this is a POST not a PUT.
$.ajax({
    type: "POST",
    url: data.url,
    xhr: function () {
        myXhr = $.ajaxSettings.xhr();
        if (myXhr.upload) {
            myXhr.upload.addEventListener('progress', function(data) {
                console.log(data.loaded / data.total);
            }, false);
        }
        return myXhr;
    },
    success: function () {
        // Will return a 204 response with no data if the file uploaded fine
        console.log("Done")
    },
    error: function (error) {
        console.log(error)
    },
    async: true,
    cache: false,
    contentType: false,
    crossDomain: true,
    processData: false,
    timeout: 0,
    data: formData
});
```

While I'm here, for this all to work in the browser, you also need to set a CORS policy on the bucket, which will look a bit like this:

```
[
    {
        "AllowedHeaders": [
            "*"
        ],
        "AllowedMethods": [
            "GET",
            "PUT",
            "POST"
        ],
        "AllowedOrigins": [
            "*"
        ],
        "ExposeHeaders": []
    }
]
```

That goes in the Bucket permissions > Cross-origin resource sharing (CORS) section in the AWS S3 console.

And you'll have to make sure that that user who creates the presigned_post request (eg. the AWS API key you're using in Python) has the permissions to PutObject in that bucket.
