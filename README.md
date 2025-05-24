# dehydrated-hook-luadns
Bash only hook script for dehydrated for use with Luadns.com to solve the DNS-01
challenge to get trusted ssl certificates from LetsEncrypt or other providers.

# Description
Luadns.com is supported by certain other acme clients like certbot,
or acme.sh, but there did not seem to be any bash script out there
for **dehydrated**. So that was the motivation for the author to
create this script.

It has worked very well for the author to date, so hopefully it will be
of use to someone else!

It has only been tested on Debian Bookworm and Trixie at the time of writing
but should work on any distro with the below dependencies installed.

* curl   
* dig   
* grep   
* jq   
* awk   
* sed   

# Usage and Installation
To install the package provided do the below. Note this has only been tested on Debian
Bookworm. Then continue from editing the **dehydrated** config as below.
```bash
sudo apt install ./dehydrated-hook-luadns_0.8.2-1_amd64_bookworm.db
```

Otherwise to install manually, download the hook script, put it somewhere, make it
executable with **$ chmod +x**

Next install the required dependencies:
```bash
sudo apt install curl bind9-dnsutils jq
```

# Dehydrated Config Settings

Regardless of whether the sript is installed manually or via the package.
The following is required in the main **dehydrated** config to start using the script.

```bash
CHALLENGETYPE="dns-01" 
HOOK="/path/to/dehydrated-hook-luadns"   
HOOK_CHAIN="yes"
export lua_email="email@example.com"
export lua_api_key="124...abc....uidlsj"  
export automatic_nginx_reload="yes" 
```

The **CHALLENGETYPE, HOOK**, and **HOOK_CHAIN** options are all
covered in the wider dehydrated documentation and are not
considered here.

In the above **lua_email** is your login email for **Luadns.com** and **lua_api_key** is an
API generated via the Luadns.com login. Note that one needs to enable API access.

The HOOK_CHAIN option is supported for certificates with
multiple alternative names. This is useful to reduce the
number of calls to the script. It is recommeneded to enable
this option within the dehydrated config. However the script
will also work with HOOK_CHAIN="no"

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
by using the **-k** or **\--hook** options. See the dehydrated documentation.

After the above lines are deployed in the wider dehydrated
config, that should allow one to solve the DNS-01 challenge
via **Luadns.com** with **dehydrated**!

# Testing Directly
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

## Action

Supported are:

1. **deploy_challenge**    
2. **clean_challenge**   
3. **deploy_cert**    

### deploy\_challenge
For **deploy_challenge** the following arguments are expected,

**$ dehydrated-hook-luadns deploy_challenge \[FQDN\] \[TOKEN\] \[TXT-TOKEN\]**

**FQDN:** Fully Qualified Domain Name of Certificate being requested.   
**TOKEN:** This is actually for the HTTP-01 Challenge and is unused.   
**TXT-TOKEN:** Value of the TXT Record to be deployed.   

For multiple certificates with **HOOK_CHAIN="yes"**, these
arguments should be in a series like so:     

```bash
deploy_challenge fqdn1 token1 txt-token1 fqdn2 token2 txt-token2
```

### clean\_challenge
For **clean_challenge**, the same arguments as **deploy_challenge**
are expected:

**dehydrated-hook-luadns clean_challenge \[FQDN\] \[TOKEN\] \[TXT-TOKEN\]**

Likewise, for multiple certificates with **HOOK_CHAIN=\"yes\"**, these
arguments should be in a series like so:     

```bash
clean_challenge fqdn1 token1 txt-token1 fqdn2 token2 txt-token2
```

### deploy\_cert
If **deploy_cert** is used as an **ACTION**, in this case
all input arguments are ignored.

```bash
dehydrated-hook-luadns deploy_cert
```

If one is using **nginx**, this can be used to execute a reload of
nginx. This requires the parameter in the dehydrated config
**automatic_nginx_reload="yes"**

Set it to **\"no\"**, comment it out, or omit it in the dehydrated
config to disable.

