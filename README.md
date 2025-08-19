# Dehydrated-Hook-Luadns

dehydrated hook script for Luadns.com to solve the DNS-01 challenge.

* [https://github.com/dehydrated-io/dehydrated](https://github.com/dehydrated-io/dehydrated)

# Description

[Luadns.com](https://luadns.com) is supported by certain other acme clients like certbot,
or acme.sh, but there did not seem to be any hook script out there
for **dehydrated**. 

As **Luadns.com** is a very good and unique DNS provider, that was the motivation for
the author to create this hook script.

It has worked very well for the author to date, so hopefully it will be of use to someone else!

It has been tested on Debian Bookworm and Trixie, and also [IPFire](https://ipfire.org) at the time of writing
but should work on any distro with the below dependencies installed.

* curl   
* dig   
* grep   
* jq   
* awk   
* sed   

The script takes in a domain that a certificate is being requested for from **dehydrated**,
appends "\_acme-challenge." to the start, deploys it to the Luadns REST API, checks for propagation,
and then hands control back to **dehydrated**.

For more details a detailed desription is provided in the manual entry that is
included as part of the debian package installation.

# Contents

- [Usage and Installation](#usage-and-installation)
  - [Package Installation](#package-installation)
  - [Manual Installation](#manual-installation)
- [Dehydrated Config Settings](#dehydrated-config-settings)
- [Testing Directly](#testing-directly)
  - [Supported Actions](#supported-actions)
    - [deploy\_challenge](#deploy_challenge)
    - [clean\_challenge](#clean_challenge)
    - [deploy\_cert](#deploy_cert)
- [Sample Outputs](#sample-outputs)
- [Removing the Dependency on Dehydrated from the Package](#removing-the-dependency-on-dehydrated-from-the-package)
- [Credits](#credits)

# Usage and Installation

If running Debian or one of its derivaties its recommended to install the package so all of
the dependencies will be installed manually.

## Package Installation

To install the package download the latest package from the relases page [HERE](https://github.com/zoot101/dehydrated-hook-luadns/releases)

Note this has only been tested on Debian Bookworm and Trixie as of the time of writing, but it should work on other Debian based
distros also, since the dependencies are very simple.

Then, continue from the [Dehydrated Config Section](#dehydrated-config-settings) section below.

```bash
sudo apt install ./dehydrated-hook-luadns_0.8.8-1_amd64.deb
```

## Manual Installation

Otherwise to install manually, download the latest source code archive from the releases page [HERE](https://github.com/zoot101/dehydrated-hook-luadns/releases) and extract it

```bash
unzip dehydrated-hook-luadns-0.8.8.zip      # For the Zip File
tar xvf dehydrated-hook-luadns-0.8.8.tar.gz # For the Tar File

cd dehydrated-hook-luadns

# Install the script by placing it in /usr/bin, alternatively
# one can place it anywhere and just point dehydrated to it
chmod +x dehydrated-hook-luadns
sudo cp dehydrated-hook-luadns /usr/bin

# Install the manual entry (optional)
sudo cp manual/dehydrated-hook-luadns.1.gz /usr/share/man/man1
```

Next install the required dependencies:

```bash
# On Debian based distros
sudo apt install curl bind9-dnsutils jq

# On Fedora
sudo dnf install curl bind-utils jq
```

Note if using [IPFire](https://ipfire.org), the above dependencies are included by
default, so no other action should be required other than downloading the script.

# Dehydrated Config Settings

Regardless of whether the sript is installed manually or via the package.
The following is required in the main **dehydrated** config to start using the script.

```bash
# Select the DNS-01 challenge
CHALLENGETYPE="dns-01"

# Point Dehydrated to the Hook Script, change the
# path if you placed it elsewhere 
HOOK="/usr/bin/dehydrated-hook-luadns"

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

The **HOOK_CHAIN** option is supported for certificates with
multiple alternative names. This is useful to reduce the
number of calls to the script. It is recommeneded to enable
this option within the dehydrated config. However the script
will also work with **HOOK_CHAIN="no"**

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

## Supported Actions

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
 + Hook: # Dehydrated-Hook-Luadns Version: 0.8.8"
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
 + Hook: ----> Querying for record - Try: 1/11
 + Hook: ----> Record live
 + Hook: --> Nameserver 2/4: ns2.luadns.net.
 + Hook: ----> Waiting 2 seconds
 + Hook: ----> Querying for record - Try: 1/11
 + Hook: ----> Record live
 + Hook: --> Nameserver 3/4: ns3.luadns.net.
 + Hook: ----> Waiting 2 seconds
 + Hook: ----> Querying for record - Try: 1/11
 + Hook: ----> Record live
 + Hook: --> Nameserver 4/4: ns4.luadns.net.
 + Hook: ----> Waiting 2 seconds
 + Hook: ----> Querying for record - Try: 1/11
 + Hook: ----> Record live
 + Hook: --> Record successfully deployed and live
 + Hook: ############################
 + Hook: + SUCCESS: All Challenges successfully deployed and now live
 + Hook: ############################
```

```bash
root : server @ ~ # dehydrated-hook-luadns clean_challenge test1.example.com file1 token-value
 + Hook: ############################
 + Hook: # Dehydrated-Hook-Luadns Version: 0.8.8"
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

# Removing the Dependency on Dehydrated from the Package

By default if one installs the debian package, one will be prompted to install
**dehydrated** from the official repos. The author recommends this as the **dehydrated**
package on Debian does some nice things like setting up a config drop-in directory
and such.

However, in the case someone is using a newer version of **dehydrated** than is in
the repos and wants to install this hook script with the package this might cause
a problem as it might conflict with however the newer version of **dehydrated** is
installed.

Normally I would recommend you just install this script manually, but in the edge case,
one ***still*** wants to install with the package, it can be gotten around by removing the
dependency on **dehydrated** from the package, do to that - do the following:

First download the "Source Code" archive from the Releases Page [HERE](https://github.com/zoot101/dehydrated-hook-luadns/releases) and extract it.

```bash
tar xvf dehydrated-hook-luadns-0.8.8.tar.gz  # For the tar file
unzip dehydrated-hook-luadns-0.8.8.zip       # For the zip file

cd dehydrated-hook-luadns-0.8.8

# Now edit debian/control to delete the dehydrated depenedency
nano debian/control # or whatever text edit you prefer

# Delete the following line, save and close
 dehydrated (>=0.7.0),

# Then re-build the package like so
dh_make -s --createorig # Accept the default prompts
dpkg-buildpackage -uc -us
```

Then an updated package should be created in the parent directory.

# Credits

I came across a similar script written back in 2017 by Greg Brackley for use with [Luadns.com](https://luadns.com),
which is still hosted and available for download [HERE](https://plone.lucidsolutions.co.nz/web/pki/letsencrypt/letsencrypt-with-dehydrated-using-dns-01-on-centos-v7).

This script worked well for simple certificates with no alternative names, but had issues with
wildcard certificates and did not do any check for propagation. Although the above script is
entirely the author's work, this script did serve as a starting point - Many thanks to him.

And also many thanks to lukas2511 for [Dehydrated](https://github.com/dehydrated-io/dehydrated) itself!

