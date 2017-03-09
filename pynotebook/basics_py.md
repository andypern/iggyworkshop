

# Python lab



## Other reading

You may also find information on using the boto3 python library here:

* [Using Boto3 - AWS's SDK for Python](https://igneoussystemshelp.zendesk.com/knowledge/articles/222814587)

* [Boto3 - Retry operations](https://igneoussystemshelp.zendesk.com/knowledge/articles/223204708)


## Import modules
To begin with, execute the following code to import the modules needed for this exercise:

```python
import os
import getpass
import os.path
import boto3
import botocore
from boto3.s3.transfer import S3Transfer
import tempfile
import pprint

import botocore.utils as boto_utils

print "imported modules"

```



## Instantiate variables
You'll need to input a few things in order to get started.  
'bucket': This is your target bucket, similar to a filesystem directory without the hierarchy.    

**'access_key'**: The access key which identifies who you are to the server.  You can obtain this from your IT administrator.  This is similar to a ‘username’.

**'secret_key'**: A secret key which authenticates you to the server.  You would also obtain this from your IT administrator.  This is similar to a ‘password’.  As such, you should refrain from storing the secret_key directly in your scripts, especially if they are to be placed in a shared location.  

**'endpoint_url'**: This is the URL which hosts your Igneous Data Service.  If you are unsure what to put here, please consult with your IT Administrator.  Typically the endpoint URL would look something like this:  http://igneous.company.com:80 , or https://igneous.company.com:443.

**Note: if you specify a URL which contains ‘https’ , you will need to change the ‘use_ssl’ parameter to ‘True’.**




```python
access_key=''
secret_key=''
endpoint_url='http://some.host.or.ip:7070'
use_ssl = False


print "variables set"

```

## Setup connection parameters

You don't need to change anything here, but take notice of what we're doing. Basically, we're taking the access_key and secret-key , and creating a session object which we will later use to establish a connection.


```python

def boto3_session(access_key, secret_key):

    return boto3.Session(
        aws_access_key_id=access_key,
        aws_secret_access_key=secret_key)

print "session function created"

```

Once we have the session object, Here's how to create a connection constructor.  Note that we're calling the 'boto3_session' function from within the boto3_s3_client function.

```python
def boto3_s3_client(aKey, sKey, endpoint):

    # Create a session and then a client from it.
    session = boto3_session(aKey, sKey)
    client = session.client(
        's3',
        region_name='iggy-1',
        use_ssl=False,
        verify=False,
        endpoint_url=endpoint,
        config=boto3.session.Config(
            signature_version='s3',
            s3={
                'addressing_style': 'path'
            }
        ))

    # All finished here. We can start using the client as expected now
    return client

print "client function created"
```

Now you can actually call it..note that it won't 'do' anything yet.  But we'll print out the object so you can verify that it returned.

```python

client = boto3_s3_client(access_key, secret_key, endpoint_url)
print client

```



## Test connection & List buckets
Note that merely creating and executing a connection constructor isn't enough to know if its working, you actually have to *use* it.  Therefore, to test it out, lets have it list buckets


```python
def list_buckets(client):

    # See the following URL for more details about this call:
    # http://boto3.readthedocs.io/en/latest/reference/services/s3.html#S3.Client.list_buckets
    lsb_resp = client.list_buckets()

    for thisBucket in lsb_resp['Buckets']:
        yield thisBucket['Name']
print "defined list_buckets function"

```


Now call it, and see what prints out:

```python

bucket_list = list_buckets(client)

for bucketEntry in bucket_list:
    print bucketEntry

```


## List objects

Now that you know how to get a list of buckets, you can write a function to list the objects within one (or more) of them, provided your access/secret key has access.

First, lets define the function:


```python


def list_objects(client, bucket):
    """
    Return a generator that iterates through the keys contained within the
    specified bucket.
    """

    # See the following URL for more details about this call:
    # http://boto3.readthedocs.io/en/latest/reference/services/s3.html#S3.Client.list_objects
    lsb_resp = client.list_objects(
        Bucket=bucket)

    for obj in lsb_resp['Contents']:
        yield obj['Key']

print "list_objects function created"
```


Now, call it.  You will first have to define the bucket you want to list though, and uncomment out the line in the block below.  Keep in mind that you probably don't want to choose a bucket that has a lot of objects in it, since the printout will take time and screen real estate.

```python

#uncomment following line and put proper bucket in
#myBucket='mybucket'
objList = list_objects(client, myBucket)

for listKey in objList:
    print listKey

```


## Put your own key

Now, objectStores don't have a concept of a 'directory' , but you can be clever and form the names of your objectKeys to mimic this. Basically, you can prefix all of your objectKeys with something like "username/" .

Lets define a prefix now:

```python

objPrefix = getpass.getuser() + '/'

print "Your prefix is : %s" %(objPrefix)
```

Now that you have a prefix, any keys that you put that you place will start that way..

## Put an object

In order to put an object, you probably want some content.  This can either be a file (more specifically, the contents of a file), or it can simply be some data.  Later on you can upload files from your laptop, but to get started quickly you can just upload a dummy file with some text data.

First you'll set a variable to hold some text:

```python

TEST_TEXT = ('Nintendo is an awesome place!  Switch and Breath of the Wild are the best things since and before sliced bread!')

print TEST_TEXT
```

Next, lets define a function which will do the actual work of uploading:


```python
def put_object(client, bucket, objKey, TEST_TEXT):

    # See the following URL for more details about this call:
    # http://boto3.readthedocs.io/en/latest/reference/services/s3.html#S3.Client.put_object
    put_resp = client.put_object(
        Body=TEST_TEXT,
        Bucket=bucket,
        Key=objKey
    )

    return put_resp

print "put_object function defined"
```

In order to run it, we'll call it and pass it some variables & objects that we got from before.  Note that you'll want to define the object key (aka: filename) that you want to upload as, and uncomment the appropriate line.  See how we put the prefix in front of it.  

If it runs successfully, you should see a JSON printout of some metadata associated with the object & request.

```python
#uncomment following line and make up a filename
#keyName = "tempfile"
objKey = objPrefix + keyName
put_object(client,myBucket,objKey,TEST_TEXT)

```

## List objects with your prefix

Ok great, you put your very own object.  Remember how, when you listed objects before, you saw everything that was in the bucket?  Well, if you want to only see *YOUR* stuff, you can filter just for objects that start with your prefix.

Lets define a function to do that:



```python


def list_my_objects(client, bucket,objPrefix):
    """
    Return a generator that iterates through the keys contained within the
    specified bucket, which start with a specific prefix.
    """

    lsb_resp = client.list_objects(
        Bucket=bucket,
        Prefix=objPrefix) #notice this additional parameter

    if not lsb_resp.has_key('Contents'):
        return
    for obj in lsb_resp['Contents']:
        yield obj['Key']

print "list_my_objects function created"
```

Now lets give it a whirl...you should only see what you put:


```python

myObjList = list_my_objects(client, myBucket,objPrefix)

for myObjKey in myObjList:
    print myObjKey

```



## Get object

So you put an object, and  were able to list just your objects, congrats!  Next, lets define a function that will perform a 'get' on that object, and print the contents out to the screen.  

As always, define the function first:

```python
def get_object(client, bucket, objKey):

    # See the following URL for more details about this call:
    # http://boto3.readthedocs.io/en/latest/reference/services/s3.html#S3.Client.get_object
    get_resp = client.get_object(
        Bucket=bucket,
        Key=objKey)

    return get_resp['Body']

print "defined get_object function"
```



Now call it, specifying the key name (that we defined when we did the put).  

```python

objContents = get_object(client, myBucket, objKey)
print objContents.read()


```


## Head object

The `head_object` method allows you to get metadata, including extended metadata on an individual object basis.

First, define the function:

```python

def head_object(client, bucket,objKey):
    try:
        response = client.head_object(
            Bucket=bucket,
            Key=objKey)
        return response

    except botocore.exceptions.ClientError as e:
        error_code = int(e.response['Error']['Code'])
        #print error_code
        return "failed: %s" %(error_code)

print "defined head_object function"

```

Next, lets actually run it:

```python

objMeta = head_object(client, myBucket, objKey)
pprint.pprint(objMeta)

```

## Add metadata to an existing object

One of the cool things about an objectStore is that you can add custom metadata to your objects, enabling you to tag for all sorts of reasons and use cases.  In reality, adding metadata to an object is nothing more than performing a PUT on an existing object, with a special parameter.  The function is identical whether you are putting a brand new object, or adding metadata to an existing one.

Define a customized function:

```python
def put_object_with_metadata(client, bucket, objKey,metaDict):

    # See the following URL for more details about this call:
    # http://boto3.readthedocs.io/en/latest/reference/services/s3.html#S3.Client.put_object
    put_resp = client.put_object(
        Body=TEST_TEXT,
        Bucket=bucket,
        Key=objKey,
        Metadata = metaDict
    )

    # Return the object's version
    return put_resp

print "put_object_with_metadata function defined"
```

Perform the put, first defining some metadata.  Feel free to add or change as many fun tags as you like..

```python

metaDict = {
    "username" : getpass.getuser(),
    "job" : "superhero",
    "location" : "marioworld"
}

put_object_with_metadata(client,myBucket,objKey,metaDict)

```

If you got no errors, do another HEAD on the object and see what we get:

```python
objMeta = head_object(client, myBucket, objKey)
pprint.pprint(objMeta)

```
## Upload a file

Putting dummy files is fun, but real files are better.  Try uploading some files from your Desktop (I'd recommend photo's...)

First, create a function to upload a file, note that its slightly different than putting a file.

```python
def upload_file(client,bucket,objKey,fileName,metaDict):
    transfer = S3Transfer(client)
    transfer.upload_file(fileName, bucket, objKey,
    extra_args={'Metadata': metaDict})

print "defined upload_file function"
```

Now, lets identify a file to upload:

```python
'''
Whatever file you upload must live on your desktop for simplicity sake. Lets list out 10 files that live there..
'''

homedir = os.path.expanduser('~')
desktop = os.path.join(homedir,'Desktop')
print "\n".join(os.listdir(desktop)[0:10]

#uncomment the following line and put in the name of a file that exists on  your desktop
#myFile = "photo.jpg"
fileName = os.path.join(desktop,myFile)
fileTest = os.path.isfile(fileName)
print fileTest

```


If that returned `True` , you are good to proceed.  If it returned `False` , make sure `myFile` is actually a filename which lives on your desktop, and be sure to include the file extension.

Now, upload the file:

```python

objKey = objPrefix + myFile
upload_file(client,myBucket,objKey,fileName,metaDict)
uploadMeta = head_object(client,myBucket,objKey)
pprint.pprint(uploadMeta)
```
## Relist to see everyone's work ..

Lets see what your colleague's have been up to..

```python

#uncomment following line and put proper bucket in
#myBucket='mybucket'
objList = list_objects(client, myBucket)

for listKey in objList:
    print listKey

```



## Delete object

now its time to cleanup after yourself, basically you want to delete the object you just created.

First, define the function:

```python
def delete_object(client, bucket, objKey):

    # See the following URL for more details about this call:
    # http://boto3.readthedocs.io/en/latest/reference/services/s3.html#S3.Client.delete_object
    get_resp = client.delete_object(
        Bucket=bucket,
        Key=objKey)

    return get_resp
print "defined delete_object function"
```


Now, actually call it

```python

delete_response = delete_object(client, myBucket, objKey)
print delete_response
```

## string it all together

well, what if you want to run all of this, at once?

If you were able to run each step piece-meal above without any errors, you should now be able to go to the top of your browser and :

1.  Click the `Kernel` menu
2.  Click `Restart and run all`
