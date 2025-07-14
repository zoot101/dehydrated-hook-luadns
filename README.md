# Dehydrated-Hook-Luadns
dehydrated hook script for Luadns.com to solve the DNS-01 challenge.

https://github.com/dehydrated-io/dehydrated

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

The script takes in a domain that a certificate is being requested for from **dehydrated**,
appends "_acme-challenge." to the start, deploys it to the Luadns REST API, checks for propagation,
and then hands control back to **dehydrated**.

For more details a detailed desription is provided in the manual entry that is
included as part of the debian package installation.

# Usage and Installation
To install the package provided do the below, the dependencies will be installed
automatically.

Note this has only been tested on Debian Bookworm and Trixie as of the time of writing, but it should work on other Debian based
distros also. Then continue from editing the **dehydrated** config as below.
```bash
sudo apt install ./dehydrated-hook-luadns_0.8.4-1_amd64.deb
```

Otherwise to install manually, download the hook script, put it somewhere, make it
executable with:
```bash
chmod +x dehydrated-hook-luadns
```

Next install the required dependencies:
```bash
# On Debian based distros
sudo apt install curl bind9-dnsutils jq

# On Fedora
sudo dnf install curl bind-utils jq
```

# Dehydrated Config Settings

Regardless of whether the sript is installed manually or via the package.
The following is required in the main **dehydrated** config to start using the script.

```bash
CHALLENGETYPE="dns-01"

# If installing via the debian package
HOOK="/usr/bin/dehydrated-hook-luadns"

# For Manual Installation
HOOK="/path/to/dehydrated-hook-luadns"   

# Enable Hook Chain (if desired)
HOOK_CHAIN="yes"

# Dehydrated Hook (Luadns) Requirements
export lua_email="email@example.com"
export lua_api_key="124...abc....uidlsj"  
export automatic_nginx_reload="yes" 
```

The **CHALLENGETYPE, HOOK**, and **HOOK_CHAIN** options are all
covered in the wider dehydrated documentation and are not
considered here.
https://github.com/dehydrated-io/dehydrated

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

```bash
export lua_email="email@example.com"
export lua_api_key="124...abc....uidlsj"
```

Where **lua_email** and **lua_api_key** are the email login and
api created by the **Luadns.com** WebUI as mentioned above. You can also 
source the **dehydrated** config directly, if the required parameters have
already been added.

```bash
source /etc/dehydrated/config
```

Then follow the below:

USAGE: **dehydrated-hook-luadns \[ACTION\]  \[ARGUMENTS\]**

## Action

Supported are:

1. **deploy_challenge**    
2. **clean_challenge**   
3. **deploy_cert**    

### deploy\_challenge
For **deploy_challenge** the following arguments are expected,

```bash
dehydrated-hook-luadns deploy_challenge [FQDN] [TOKEN] [TXT-TOKEN]
```

1. **FQDN:** Fully Qualified Domain Name of Certificate being requested.   
2. **TOKEN:** This is actually for the HTTP-01 Challenge and is unused here.   
3. **TXT-TOKEN:** Value of the TXT Record to be deployed.   

For multiple certificates with **HOOK_CHAIN="yes"**, these
arguments should be in a series like so:     

```bash
dehydrated-hook-luadns deploy_challenge fqdn1 token1 txt-token1 fqdn2 token2 txt-token2
```

### clean\_challenge
For **clean_challenge**, the same arguments as **deploy_challenge**
are expected:

```bash
dehydrated-hook-luadns clean_challenge [FQDN] [TOKEN] [TXT-TOKEN]
```

Likewise, for multiple certificates with **HOOK_CHAIN=\"yes\"**, these
arguments should be in a series like so:     

```bash
dehydrated-hook-luadns clean_challenge fqdn1 token1 txt-token1 fqdn2 token2 txt-token2
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

# Sample Outputs

Shown below are some sample outputs that will be seen when being used by dehydrated.

```bash
root : server @ ~ # dehydrated-hook-luadns deploy_challenge test1.example.com file1 token-value
 + Hook: ############################
 + Hook: + deploy_challenge: 1 of 1
 + Hook: ############################
 + Hook: Name:_acme-challenge.test1.example.com, Type:TXT, Value:token-value
 + Hook: Zone Name - example.com
 + Hook: Querying Luadns Nameservers for example.com NS Records...
 + Hook: --> Querying Nameserver 1/4: ns1.luadns.net
 + Hook: ----> Success: Obtained 4 NS records for example.com
 + Hook: Deploying record via https://api.luadns.com/v1...
 + Hook: --> Getting Zone ID...
 + Hook: --> Zone ID - 9090
 + Hook: --> Record successfully deployed.
 + Hook: Querying all 4 example.com Nameservers for Record...
 + Hook: --> Nameserver 1/4: ns1.luadns.net.
 + Hook: ----> Waiting 2 seconds
 + Hook: ----> Querying for record - Try: 1/5
 + Hook: ----> Record live
 + Hook: --> Nameserver 2/4: ns2.luadns.net.
 + Hook: ----> Waiting 2 seconds
 + Hook: ----> Querying for record - Try: 1/5
 + Hook: ----> Record live
 + Hook: --> Nameserver 3/4: ns3.luadns.net.
 + Hook: ----> Waiting 2 seconds
 + Hook: ----> Querying for record - Try: 1/5
 + Hook: ----> Record live
 + Hook: --> Nameserver 4/4: ns4.luadns.net.
 + Hook: ----> Waiting 2 seconds
 + Hook: ----> Querying for record - Try: 1/5
 + Hook: ----> Record live
 + Hook: --> Record successfully deployed and live
 + Hook: ############################
 + Hook: + SUCCESS: All Challenges successfully deployed and now live
 + Hook: ############################
```

```bash
root : server @ ~ # dehydrated-hook-luadns clean_challenge test1.example.com file1 token-value
 + Hook: ############################
 + Hook: + clean_challenge: 1 of 1
 + Hook: ############################
 + Hook: Name:_acme-challenge.test1.example.com, Type:TXT, Value:token-value
 + Hook: Zone Name - example.com
 + Hook: Deleting record via https://api.luadns.com/v1...
 + Hook: --> Getting Zone ID...
 + Hook: --> Zone ID - 9090
 + Hook: --> Getting Record ID...
 + Hook: --> Record ID - 12345
 + Hook: --> Record: 12345 Successfully Deleted
 + Hook: ############################
 + Hook: + SUCCESS: All Challenges cleaned
 + Hook: ############################
```

# Credits

I did come across a script written back in 2017 which is still hosted here,
written by Greg Brackley.
https://plone.lucidsolutions.co.nz/web/pki/letsencrypt/letsencrypt-with-dehydrated-using-dns-01-on-centos-v7

This script still worked but had issues especially with wildcard certificates and didn't do any check
for propagation, but it did serve as a starting point. Thanks to him for his work on
that script.

And also many thanks to lukas2511 for **dehydrated** itself!


