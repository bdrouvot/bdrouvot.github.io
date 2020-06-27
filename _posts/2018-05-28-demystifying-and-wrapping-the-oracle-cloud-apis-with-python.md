---
layout: post
title: demystifying and wrapping the oracle cloud APIs with Python
date: 2018-05-28 05:56:10.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- cloud
tags: []
meta:
  _edit_last: '40807211'
  _oembed_1a8253b44ddc96ce14a8dcda5e5f4deb: "{{unknown}}"
  geo_public: '0'
  _oembed_f7eb3d9f9819f7bb51a9d2c9582e79d5: "{{unknown}}"
  _oembed_687d66d44ada6be94e008634e8c20490: "{{unknown}}"
  _thumbnail_id: '3361'
  timeline_notification: '1527483374'
  _publicize_job_id: '18334156709'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=6406729629943554048&type=U&a=r4Wc
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:62:"https://twitter.com/BertrandDrouvot/status/1000963948160737280";}}
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2018/05/28/demystifying-and-wrapping-the-oracle-cloud-apis-with-python/"
---

As an oracle DBA you might have to deal with [REST API](https://en.wikipedia.org/wiki/Representational_state_transfer), especially when working with the cloud. The purpose of this post is to demystify the REST API usage from a DBA point of view. Let's take an example and write a Python wrapper to automate the Instance creation in the Oracle Database Cloud Service.

The instance creation can be done manually using the Web Interface that way:

[<img src="{{ site.baseurl }}/assets/images/screen-shot-2018-05-26-at-07-36-25.png" class="aligncenter size-full wp-image-3361" width="1100" height="461" />](https://bdrouvot.wordpress.com/2018/05/28/demystifying-and-wrapping-the-oracle-cloud-apis-with-python/screen-shot-2018-05-26-at-07-36-25/)

It could also been done using the APIs. The APIs are described in this [link](https://apicatalog.oraclecloud.com/ui/views/apicollection/oracle-public/database/csdbr-13/serviceinstances). The one related to the Instance creation is the following:

[<img src="{{ site.baseurl }}/assets/images/create_instance_api.png" class="aligncenter size-full wp-image-3351" width="1100" height="686" />](https://bdrouvot.wordpress.com/2018/05/28/demystifying-and-wrapping-the-oracle-cloud-apis-with-python/create_instance_api/)

So, identityDomainId is a path parameter, the Authorization and X-ID-TENANT-NAME are header parameters and the body describes the parameters in JSON format.

To interact with the APIs through HTTP, let's use the [Python Requests module.](http://docs.python-requests.org/en/master/)

To work with JSON let's use the [JSON one](https://docs.python.org/2/library/json.html).

First, we need to create an http session. Let's import the requests module:

```
import requests  
```
and then create the http session:

```
client = requests.Session()  
```

Once done, we can add the authorization:

```
client.auth = (USERNAME, PASSWORD)  
```

and update the header with the content-type and the X-ID-TENANT-NAME:

```
client.headers.update(  
{'content-type': 'application/json'  
'X-ID-TENANT-NAME':'{0}'.format(IDENTITY\_DOMAIN\_ID)})  
```

Now, let's create the body. In this example the JSON data has been extracted from a file (*prov\_database.json*) that contains:

    $ cat prov_database.json
    {
      "description": "BDTDESC",
      "edition": "EE",
      "level": "PAAS",
      "serviceName": "WILLBEOVERWRITTEN",
      "shape": "oc3",
      "subscriptionType": "MONTHLY",
      "version": "12.1.0.2",
      "vmPublicKeyText": "ssh-rsa <YOURKEY>",
      "parameters": [
        {
          "type": "db",
          "usableStorage": "15",
          "adminPassword": "<YOURPWD>",
          "sid": "BDTSID",
          "pdbName": "MYPDB",
          "failoverDatabase": "no",
          "backupDestination": "NONE"
        }
      ]
    }

It has been extracted in the Python wrapper that way:

```
data = json.load(open('{0}/prov\_database.json'.format(dir\_path), 'r'))  
```

Now that we have defined the authorization, the header and the body, all we have to do is to post to http.

The url structure is described [here:](https://docs.oracle.com/en/cloud/paas/database-dbaas-cloud/csdbr/url-structure.html)[<img src="{{ site.baseurl }}/assets/images/screen-shot-2018-05-27-at-07-41-39.png" class="aligncenter size-full wp-image-3371" width="1100" height="495" />](https://bdrouvot.wordpress.com/2018/05/28/demystifying-and-wrapping-the-oracle-cloud-apis-with-python/screen-shot-2018-05-27-at-07-41-39/)

So that the post is launched that way with Python:

```
response = client.post("https://dbcs.emea.oraclecloud.com/paas/service/dbcs/api/v1.1/instances/{0}".format(IDENTITY\_DOMAIN\_ID), json=data)  
```

As you can see the IDENTITY\_DOMAIN\_ID is also a path parameter and the data is part of the request. That's it!

**Remarks:**

-   The username, password and identityDomainId have also been extracted from a file (*.oracleapi\_config.yml*). The file contains:

<!-- -->

    $ cat .oracleapi_config.yml
    identityDomainId: <YOURS>
    username: <YOURS>
    password: <YOURS>
    logfile: oracleapi.log

and has been extracted that way:

```
f = open('{0}/.oracleapi\_config.yml'.format(dir\_path), 'r')  
config = safe\_load(f)  
IDENTITY\_DOMAIN\_ID = config\['identityDomainId'\]  
USERNAME = config\['username'\]  
PASSWORD = config\['password'\]  
```

-   More fun:  
    You may have noticed that the http post's response provides a link to an URL you can use to check the progress of the creation

[<img src="{{ site.baseurl }}/assets/images/screen-shot-2018-05-24-at-22-22-26.png" class="aligncenter size-full wp-image-3355" width="1100" height="235" />](https://bdrouvot.wordpress.com/2018/05/28/demystifying-and-wrapping-the-oracle-cloud-apis-with-python/screen-shot-2018-05-24-at-22-22-26/)

So that we can integrate the check in our wrapper (the source code is available at the end of this post and in this [git repository](https://github.com/bdrouvot/opc_api_wrapper)).

#### Let's use our wrapper:

    $ python ./opc_api_wrapper.py
    Usage:
        opc_api_wrapper.py <service_name> create_instance

    $ python ./opc_api_wrapper.py BDTSERV create_instance
    creating opc instance BDTSERV...
    InProgress (Starting Compute resources...)
    InProgress (Starting Compute resources...)
    InProgress (Starting Compute resources...)
    InProgress (Configuring Oracle Database Server...)
    InProgress (Configuring Oracle Database Server...)
    InProgress (Configuring Oracle Database Server...)
    InProgress (Configuring Oracle Database Server...)
    InProgress (Configuring Oracle Database Server...)
    InProgress (Configuring Oracle Database Server...)
    InProgress (Configuring Oracle Database Server...)
    InProgress (Configuring Oracle Database Server...)
    InProgress (Configuring Oracle Database Server...)
    InProgress (Configuring Oracle Database Server...)
    InProgress (Configuring Oracle Database Server...)
    InProgress (Configuring Oracle Database Server...)
    Succeeded ( Service Reachabilty Check (SRC) of Oracle Database Server [BDTSERV] completed...)

    opc instance BDTSERV created:

    SSH access to VM [DB_1/vm-1] succeeded...
    Oracle Database Server Configuration completed...
    Successfully provisioned Oracle Database Server...
    Service Reachabilty Check (SRC) of Oracle Database Server [BDTSERV] completed...

During the instance creation, you could also check the progress through the web interface:  
[  
<img src="{{ site.baseurl }}/assets/images/screen-shot-2018-05-24-at-22-15-06.png" class="aligncenter size-full wp-image-3353" width="1100" height="206" />](https://bdrouvot.wordpress.com/2018/05/28/demystifying-and-wrapping-the-oracle-cloud-apis-with-python/screen-shot-2018-05-24-at-22-15-06/)

### Source code:

```
\# Author: Bertrand Drouvot  
\# Blog : http://bdrouvot.wordpress.com/  
\# opc\_api\_wrapper.py : V1.0 (2018/05)  
\# Oracle cloud API wrapper

from yaml import safe\_load  
import os  
from os import path  
import logging  
import json  
import requests  
import time  
from docopt import docopt

FORMAT = '%(asctime)s - %(name)s - %(levelname)-s %(message)s'

help = ''' Oracle Public Cloud Wrapper

Usage:  
opc\_api\_wrapper.py &lt;service\_name&gt; create\_instance

Options:  
-h Help message

Returns .....  
'''

class APIError(Exception):

def \_\_init\_\_(self, message, status\_code=None, payload=None):  
Exception.\_\_init\_\_(self)  
self.message = message  
self.status\_code = 415

def to\_dict(self):  
rv = dict()  
rv\['message'\] = self.message  
return rv

def launch\_actions(kwargs):

dir\_path = os.path.dirname(os.path.realpath(\_\_file\_\_))  
try:  
f = open('{0}/.oracleapi\_config.yml'.format(dir\_path), 'r')  
config = safe\_load(f)  
except:  
raise ValueError("This script requires a .oracleapi\_config.yml file")  
exit(-1)

if kwargs\['create\_instance'\]:  
create\_instance(kwargs\['service\_name'\],dir\_path,config)

def return\_last\_from\_list(v\_list):  
for msg in (v\_list\[0\], v\_list\[-1\]):  
pass  
return msg

def print\_all\_from\_list(v\_list):  
for mmsg in v\_list:  
print mmsg

def check\_job(config,joburl):

IDENTITY\_DOMAIN\_ID = config\['identityDomainId'\]  
USERNAME = config\['username'\]  
PASSWORD = config\['password'\]

client = requests.Session()  
client.auth = (USERNAME, PASSWORD)  
client.headers.update({'X-ID-TENANT-NAME': '{0}'.format(IDENTITY\_DOMAIN\_ID)})

response = client.get("{0}".format(joburl))  
jsontext= json.loads(response.text)  
client.close()  
return (jsontext\['job\_status'\],jsontext\['message'\])

def create\_instance(service\_name,dir\_path,config):

logfile = config\['logfile'\]  
logging.basicConfig(filename=logfile, format=FORMAT, level=logging.INFO)

IDENTITY\_DOMAIN\_ID = config\['identityDomainId'\]  
USERNAME = config\['username'\]  
PASSWORD = config\['password'\]

data = json.load(open('{0}/prov\_database.json'.format(dir\_path), 'r'))  
data\['serviceName'\] = service\_name

print "creating opc instance {0}...".format(service\_name)

client = requests.Session()  
client.auth = (USERNAME, PASSWORD)  
client.headers.update(  
{'content-type': 'application/json',  
'X-ID-TENANT-NAME':'{0}'.format(IDENTITY\_DOMAIN\_ID)})

response = client.post("https://dbcs.emea.oraclecloud.com/paas/service/dbcs/api/v1.1/instances/{0}".format(IDENTITY\_DOMAIN\_ID), json=data)  
if response.status\_code != 202:  
raise APIError(response.json()\['message'\])  
jobburl = response.headers\['Location'\]  
jobsstatus = "InProgress"  
while (jobsstatus == "InProgress"):  
time.sleep(120)  
jobsstatus,jobmessage = check\_job(config,jobburl)  
print "{0} ({1})".format(jobsstatus,return\_last\_from\_list(jobmessage))  
client.close()  
print ""  
print "opc instance {0} created:".format(service\_name)  
print ""  
print\_all\_from\_list(jobmessage)

\#  
\# Main  
\#

def main():

arguments = docopt(help)  
for key in arguments.keys():  
arguments\[key.replace('&lt;','').replace('&gt;','')\] = arguments.pop(key)

launch\_actions(arguments)

if \_\_name\_\_ == '\_\_main\_\_':  
main()  
```

### Conclusion

We have been able to wrap the instance creation APIs with Python. We have also included a way to check/follow the creation progress. Wrapping the APIs with Python gives flexibility, for example you could launch ansible playbook(s) from the wrapper once the instance creation is done.
