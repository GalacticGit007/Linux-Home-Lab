# AWS S3 Setup - Bucket versioning, Static website hosting & Permissions

Today I worked with Amazon S3 learned how to create and manage S3 bucket, enable bucket versioning, uploading files via CLI, static website hosting,presigned url, permisions and bucket policies.

## Creating an S3 Bucket

I created an S3 Bucket using AWS Management Console.

```my-new-s3-bucket-761595200834-ap-south-1-an```

The bucket was created in the ap-south-1 region.

I have enable bucket versioning during the setup. Versioning allows S3 to maintain multiple versions of the same object instead of permanently replacing the previous version.

In my case, If ```index.html``` is modified several times, previous versions can still be recovered.

## Uploading an object using AWS CLI

``` bash
aws s3 cp index.html s3://my-new-s3-bucket-761595200834-ap-south-1-an/
```

This uploads ```index.html``` from the current directory to the S3 bucket.

## Presigned URL

I learned that an S3 object can remain private while still being temporarily shared using a presigned URL.

Using the command in CLI I can create a presigned url for my webpage for 1800 seconds.(temporary access)

``` bash
aws s3 presign s3://my-new-s3-bucket-761595200834-ap-south-1-an/index.html --expires-in 1800
```

The url expires after 1800 seconds.

## S3 static web hosting

I also created a simple static website using S3.
The ```index.html``` contained HTML that displayed the image stored in S3.

``` html

<!DOCTYPE html>
<html>
    <head>
         <title>My S3 Website</title> 
    </head> 
    <body> 
        <h1>My S3 Static Website</h1> 
        <img src="ar1.jpeg" alt="S3 Image" width="500"> 
    </body> 
</html>

```

I enable static website hosting from the web console.

```Index document : index.html```

S3 then provided a website endpoint that could be opened in a browser.

## Things I learned

#### S3 upload returned ```AccessDenied```
Initially, an upload failed with:

```AccessDenied when calling the PutObject operation```

The issue turned out to be an incorrect S3 path rather than Block Public Access.

The correct format is:

``` bash
aws s3 cp FILE s3://BUCKET-NAME/
```

#### Presigned URL &ne; Public Object

A presigned URL does not make the object public. Instead ```Private S3 Object``` &rarr; ```Presigned URL``` &rarr; ```Temporary Access``` &rarr; ```Expires```

This is useful when I want to share a private object temporarily.

#### Understanding S3 Permissions

* Turning off Block Public Access does not automatically make the bucket public. If there is no permission allowing anonymous access, the bucket remains private.
* A bucket policy is a resource-based policy attached to an S3 bucket.

For example, a policy can allow public read access:
``` json

{ 
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject", 
            "Resource": [ 
                "arn:aws:s3:::my-bucket/index.html",
                "arn:aws:s3:::my-bucket/ar1.jpeg" 
            ] 
        }
    ] 
}

```

Here, ``` "Principal": "*"``` means anyone can make the request. ``` "Action": "s3:GetObject" ``` allows reading objects. The ```Resource``` specifies exactly which objects are accessible, which allows ``` index.html``` and ``` ar1.jpeg``` public and other files in the bucket private.

#### Three configurations I learned

``` bash

Block Public Access → ON 
Bucket Policy → None

RESULT:
        PRIVATE

Block Public Access → OFF 
Bucket Policy → None

RESULT:
        STILL PRIVATE

Block Public Access → OFF 
Bucket Policy → Allows Principal: *

RESULT:
        Public according to the policy

```

Lesson learned ;)