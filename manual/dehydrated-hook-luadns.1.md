---
title: dehydrated-hook-luadns
section: 1
header: User Manual
footer: dehydrated-hook-luadns
---

# NAME
**dehydrated-hook-luadns**

# SYNOPSIS

Dehydrated Hook Script for Luadns.com to solve the dns-01
challenge

USAGE: **dehydrated-hook-luadns \[ACTION\]**

# DESCRIPTION

Hook script for **dehydrated** to allow one to solve the dns-01
challenge using **Luadns.com**. Luadns is widely supported by
other clients that can use the dns-01 challenge like certbot
but not by dehydrated. This was the motivation for the author
to create the script.

The DNS-01 Challenge is quite a nice way to prove ownership
of a domain to satisfy the acme protocol in order to get
trusted SSL certificates from LetsEncrypt or other providers.

It involves the user deploying a special DNS TXT Record to
prove ownership of the domain instead of opening ports to
the internet like the http-01 challenge.

The script will also check that the record has correctly propagated after
deploying it. It does this by initially querying the Luadns
nameservers **(usually ns[1-4].luadns.net)** for the official nameservers
for the domain passed in by **dehydrated**. Then it will query each
of these nameservers to ensure the record is available on all of
them before handing back control to **dehydrated**.

This script is not really intended to be used directly, but
rather by specifying as a hook script within the wider
dehydrated config - see the section on the config below.

However it's a very good idea to test it directly before using it
with dehydrated (see below).

# REQUIRED DEPENDENCIES

The Following Dependencies are required to be installed:

1. **curl**     
2. **dig**     
3. **grep**    
4. **jq**    
5. **awk**    
6. **sed**    

# REQUIRED DEHYDRATED CONFIG SETTINGS

To use this script, modify the **dehydrated** config, which
(if installed from the debian repos), is here:

* **/etc/dehydrated/config**

Or (as recommended) create a drop-in file here:

* **/etc/dehydrated/conf.d/config.sh**

Then add the following to the config:

* **CHALLENGETYPE=\"dns-01\"**   
* **HOOK=\"/usr/bin/dehydrated-hook-luadns\"**    
* **HOOK_CHAIN=\"yes\"**   
* **export lua_email="email@example.com"**   
* **export lua_api_key="124...abc....uidlsj"**    
* **export automatic_nginx_reload=\"yes\"**   

The **CHALLENGETYPE, HOOK**, and **HOOK_CHAIN** options are all
covered in the wider dehydrated documentation and are not
considered here.

**The HOOK_CHAIN option is supported for certificates with**
**multiple alternative names. This is useful to reduce the**
**number of calls to the script. It is recommeneded to enable**
**this option within the dehydrated config. However the script**
**will also work with HOOK_CHAIN="no"**

In the above config options, **lua_email** and **lua_api_key**
are the login email for your luadns.com account, and the
**lua_api_key** is an api key that you can generate via
their WebUI that has access to the DNS Zone where you want
to deploy the TXT Record. See the section below on creating
the API key.

The **automatic_nginx_reload** parameter is optional above.
It is intended to automatically reload an nginx webserver
after new certificates are issued if one is running nginx.
It can be omitted or left commented out if the user desires.

Note the script can also be passed to **dehydrated** directly
by using the **-k** or **\--hook** options. See **dehydrated(1)**

After the above lines are deployed in the wider dehydrated
config, that should allow one to solve the DNS-01 challenge
via **Luadns.com** with **dehydrated**!

# TESTING DIRECTLY

The script can be tested directly by following the procedure below.
It is recommended to ensure it works when being called on its own
before using with **dehydrated** to issue real certificates and also
to ensure the **Luadns.com** credentials are correct etc. At a minimum
the **deploy_challenge** and **clean_challenge** actions should be
tested directly.

**dehydrated** itself will also follow the below procedure.

To use the script directly, 1st execute the following commands:

**$ export lua_email="email@example.com"**   
**$ export lua_api_key="124...abc....uidlsj"**    

Where **lua_email** and **lua_api_key** are the email login and
api created by the **Luadns.com** WebUI as mentioned above. You can also 
source the **dehydrated** config directly, if the required parameters have
already been added.

**$ source /etc/dehydrated/config**

Then follow the below:

USAGE: **dehydrated-hook-luadns \[ACTION\]  \[ARGUMENTS\]**

## ACTION

Supported are:

1. **deploy_challenge**    
2. **clean_challenge**   
3. **deploy_cert**    

### deploy\_challenge
For **deploy_challenge** the following arguments are expected,

**$ dehydrated-hook-luadns deploy_challenge \[FQDN\] \[TOKEN\] \[TXT-TOKEN\]**

**FQDN:** Fully Qualified Domain Name of Certificate being requested.
**TOKEN:** This is actually for the HTTP-01 Challenge and is unused.
**TXT-TOKEN:** Value of the TXT Record to be deployed

For multiple certificates with **HOOK_CHAIN=\"yes\"**, these
arguments should be in a series like so:     

**$ deploy_challenge fqdn1 token1 txt-token1 fqdn2 token2 txt-token2**

### clean\_challenge
For **clean_challenge**, the same arguments as **deploy_challenge**
are expected:

**$ dehydrated-hook-luadns clean_challenge \[FQDN\] \[TOKEN\] \[TXT-TOKEN\]**

Likewise, for multiple certificates with **HOOK_CHAIN=\"yes\"**, these
arguments should be in a series like so:     

**$ clean_challenge fqdn1 token1 txt-token1 fqdn2 token2 txt-token2**

### deploy\_cert
If **deploy_cert** is used as an **ACTION**, in this case
all input arguments are ignored.

**$ dehydrated-hook-luadns deploy_cert**

If one is using **nginx**, this can be used to execute a reload of
nginx. This requires the parameter in the dehydrated config
**automatic_nginx_reload=\"yes\"**

Set it to **\"no\"**, comment it out, or omit it in the dehydrated
config to disable.

# DEHYDRATED OFFICAL HOOK SCRIPT DOCUMENTATION

See all supported hook script actions and their usage here:

* https://github.com/dehydrated-io/dehydrated/blob/master/docs/examples/hook.sh

# LUADNS.COM API KEY

An API key can be created via the Luadns.com WebUI relatively easily.
Log into the Luadns.com via your account and:

1. In the **General** Tab - Select **Enable API Access**   
2. In the **API Keys** Tab - Select **New API Key**   
3. Select the **Zone** (Domain) you want it to have access to   
4. Enter a Description, hit Save.   
5. Download the API Key generated and save it to the **dehydrated** config   
6. Its a good idea to secure the config with a **chmod 700**   

For security purposes one can only view the API key via the
WebUI one time so if it's lost one will require creating a
new API Key.

Note also that the default for Luadns.com is a single API key
has access to a single Zone (Domain). Global API keys with
access to all zones are not provided.

# DETAILED OPERATION

If one is interested, a detailed description of all the operations carried out
by the script are shown below for **deploy_challenge** and **clean_challenge**
actions.

## STEP 1 - INITIAL CHECK

An Initial check is carried out to see if a login email and api
key are correctly defined (or have been passed in by **dehydrated**). The
script will exit if either are undefined, as they are required to
interact with the LuaDNS REST API server.

Next, the dependencies are checked for. This should be redundant if
installed via the debian package, but is included for good practise.
A check is carried out for **awk, curl, grep, jq, and sed**. If any are
found not to be installed, the script will exit.

## STEP 2 - GET THE ZONE NAME

For a challenge **test1.test2.test3.example.org**, **dehydrated** will supply
the script with exactly that as an argument. However the root domain,
which in the above example is **example.org** is required to interact
with the Luadns API. The script extracts this from what is passed in from **dehydrated**,
and removes the trailing dot if present (this is another requirement of the LuaDNS API).

If an invalid domain name is specified, the script will exit. The check is relatively
simple and is just based on the number of dots present in what was passed in, so it is possible
to confuse it. However this is considered properly in the next step.

## STEP 3 - GET THE NS RECORDS FOR THE ZONE

Next, the script will query the main LuaDNS nameservers one after another until
valid NS (Nameserver) records are obtained. The servers are:

* ns1.luadns.net   
* ns2.luadns.net   
* ns3.luadns.net   
* ns4.luadns.net   

It would probably be possible to automatically assume that the NS records
will always point to the above, but it is possible to have Vanity Nameservers
with Luadns on their paid plans, which would mean different NS records than
the above. These NS records are queried directly later to ensure that the
record has propagated correctly.

If no valid NS records are found from the above nameservers, the script will
exit. This serves as a way of testing that the domain is valid and actually using
LuaDNS, before attempting to contact the API.

## STEP 4 - CARRY OUT THE ACTION

### deploy\_challenge

For **deploy_challenge**, at this point, the script has a valid domain to
work with, all dependencies and LuaDNS credentials. It will then move on to
do the following:

* Query the LuaDNS REST API for the Zone ID using the Zone Name from Step 2    
* Append "_acme-challenge." to what was given by dehydrated to create the record to deploy    
* Deploy the Record (Challenge)     
* Wait for Propagation to all Nameservers given by the NS Records from Step 3      

Curl is called to do this as per the documenation here:   
**https://www.luadns.com/api.html**

The script will check that a valid response is obtained for each of the above
operations by parsing the json data received, and will exit if a problem is found.

After the record has found to be successfully deployed, the script will then move
on to query all the Nameservers found from the NS records of **STEP 3** above for
what was just deployed.

Each nameserver is queried for the record using **dig**, and if the record is not present,
the script will wait a period and then try again. Each attempt the waiting time will double
before trying again. In the authors experience new records deployed via the API are very quick
to appear on **ALL** of the LuaDNS nameservers.

Once the record is found to be present on **ALL** of the Nameservers given by the NS
records of **STEP 3**, the script will move on to the next challenge to deploy or hand control
back to **dehydrated**.

### clean\_challenge

For **clean_challenge** the process is similiar:

* Append "_acme-challenge." to what was given by dehydrated     
* Query the LuaDNS REST API for the Zone ID    
* Find the Record ID for a TXT record with the required name and content     
* Delete the Record (Challenge)     

Curl is called to do this as per the documenation here:   
**https://www.luadns.com/api.html**

A check is done for the type (usually **TXT**), name and content to find the record to delete. 
All 3 need to be considered as for Wildcard certificates, two challenges with the same
name and different content are required to be deployed. The response from the api
server is tested that it is valid by parsing the json data and investigating if
the same Record ID was returned as was given.

After that, the script will move on to the next challenge to clean or hand control
back to **dehydrated**.

It is also possible at this point that there may be multiple records with the
same name, type, and content. This is probably impossible to happen with **dehydrated**
calling the script, but may be possible if the script is called directly to do
a **deploy_challenge** without being followed by a **clean_challenge** action, and doing
the same **deploy_challenge** action again.

If this happens, the script will cycle through each of the records found with the
same name, type and content and delete all of them.

This is an unusual edge case, and is probably very unlikely to happen but is accounted for
here.

# AUTHOR

Mark Finnan <mfinnan101@gmail.com>

# COPYRIGHT

Copyright (C) Mark Finnan 2025

# SEE-ALSO

dehydrated(1), curl(1), grep(1), dig(1), jq(1)

