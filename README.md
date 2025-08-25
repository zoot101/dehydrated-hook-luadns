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
appends "\_acme-challenge." to the start, deploys it to the Luadns REST API as a TXT record
with what was given by **dehyrated**, checks for propagation by querying the Luadns nameservers
to see if the record just deployed is live, and then hands control back to **dehydrated**.

For more details a detailed desription is provided in the manual entry that is
included as part of the debian package installation.

# Contents

- [Requirements](#requirements)
- [Usage and Installation](#usage-and-installation)
  - [Package Installation](#package-installation)
  - [Manual Installation](#manual-installation)
- [Dehydrated Config Settings](#dehydrated-config-settings)
  - [deploy\_cert\_command](#deploy_cert_command)
  - [deploy\_cert\_script](#deploy_cert_script)
- [Using an Alternative Domain for Certificate Issue](#using-an-alternative-domain-for-certificate-issue)
- [Testing Directly](#testing-directly)
  - [Supported Actions](#supported-actions)
    - [deploy\_challenge](#deploy_challenge)
    - [clean\_challenge](#clean_challenge)
    - [deploy\_cert](#deploy_cert)
- [Sample Outputs](#sample-outputs)
- [Removing the Dependency on Dehydrated from the Package](#removing-the-dependency-on-dehydrated-from-the-package)
- [Links](#links)
  - [Luadns Rest API and Documentation](#luadns-rest-api-and-documentation)
  - [Dehydrated Documentation](#dehydrated-documentation)
- [Issues](#issues)
- [Credits](#credits)

# Requirements

To use this script with **dehydrated**, the following is required up front:

* Ownership of a valid domain name with its DNS hosted at [Luadns.com](https://luadns.com).    
  - Luadns servers (ns[1-4].luadns.net) configured for the Domain at your Domain Registrar.   
* API access enabled in the Luadns.com account settings for the Domain.    
* A valid API Key created for the zone in question
  - Example: For record.example.org, **example.org** is the Zone)    

# Usage and Installation

If running Debian or one of its derivaties its recommended to install the package so all of
the dependencies (including **dehydrated** itself) will be installed automatically.

## Package Installation

To install the package, first download the latest package from the relases page [HERE](https://github.com/zoot101/dehydrated-hook-luadns/releases)

Note this has only been tested on Debian Bookworm and Trixie as of the time of writing, but it should work on other Debian based
distros also, since the dependencies are very simple.

Install the package like so. It's better to use **apt** instead of **dpkg** so the dependencies are installed automatically.

```bash
sudo apt install ./dehydrated-hook-luadns_1.0.1-1_amd64.deb
```

Then, continue from the [Dehydrated Config Section](#dehydrated-config-settings) section below.

## Manual Installation

Otherwise to install manually, download the latest source code archive from the releases page [HERE](https://github.com/zoot101/dehydrated-hook-luadns/releases) and extract it

```bash
unzip dehydrated-hook-luadns-1.0.1.zip      # For the Zip File
tar xvf dehydrated-hook-luadns-1.0.1.tar.gz # For the Tar File

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
default, so no other action should be required other than downloading the script. There is
also no need to install the manual entry as the **man** command is not included in 
[IPFire](https://ipfire.org) by default.

# Dehydrated Config Settings

Regardless of whether the script is installed manually or via the package, the following
is required in the main **dehydrated** config to start using the script.

```bash
# Select the DNS-01 challenge
CHALLENGETYPE="dns-01"

# Point Dehydrated to the Hook Script, change the
# path if you placed it elsewhere 
HOOK="/usr/bin/dehydrated-hook-luadns"

# Enable Hook Chain (Recommended)
HOOK_CHAIN="yes"

################################
# Dehydrated Hook - Luadns.com Parameters
################################
# Luadns.com Credentials
export lua_email="email@example.com"
export lua_api_key="124...abc....uidlsj"  

# Issue a Custom Command if a new Certificate is Deployed (Optional)
export deploy_cert_command="systemctl reload nginx"

# Call a Custom Script if a new Certificate is Deployed (Optional)
# (Must be executable)
export deploy_cert_script="/path/to/script/here" 
```

The **CHALLENGETYPE, HOOK**, and **HOOK_CHAIN** options are all
covered in the wider dehydrated documentation and are not
considered here.

* [https://github.com/dehydrated-io/dehydrated](https://github.com/dehydrated-io/dehydrated)

In the above example config, **lua_email** is your login email for **Luadns.com** and **lua_api_key** is an
API generated via the Luadns.com WebUI that has access to the Domain (or DNS Zone) you wish to issue certificates
for. Note that one needs to enable API access.

The **HOOK_CHAIN** option is supported for certificates with
multiple alternative names. This is useful to reduce the
number of calls to the script. It is recommeneded to enable
this option within the dehydrated config. However the script
will also work with **HOOK_CHAIN="no"**

## deploy\_cert\_command

The **deploy_cert_command** parameter is optional above. If specified the
script will execute this command when the script is called by **dehydrated**
with the **deploy_cert** argument which is done for each new certificate
**dehydrated** creates.

As per the **dehydrated** hook example [HERE](https://github.com/dehydrated-io/dehydrated/blob/master/docs/examples/hook.sh), this is
intended to allow the user to reload a webserver using any new certificates created by **dehydrated**. But it
also can be any custom command the user desires. Comment out or omit if not using.

Some examples could be:

* export deploy\_cert\_command="systemctl reload nginx"   
* export deploy\_cert\_command="/etc/init.d/apache restart"

## deploy\_cert\_script

The **deploy_cert_script** parameter is also optional. This allows the user to
get the hook script to call a further custom script of their own. This differs from the
**deploy_cert_command** parameter in that it calls the script and passes in all
of the arguments given by **dehydrated** for the **deploy_cert** function detailed [HERE](https://github.com/dehydrated-io/dehydrated/blob/master/docs/examples/hook.sh).
Must be executable.

This could be used for example to copy the new certificate(s) created by **dehydrated** to
a new location and reload the webserver using then. It also could be used to convert the
certificates to a different format if desired. It doesn't have to be a bash script,
python, perl, an executable or anything that can be called from the command line should be OK.

If you wish to get the hook script to reload your webserver, the **deploy_cert_command** is what
to use. If you want to do something more like copy the certificates to a new location or
something, then use the **deploy_cert_script** option. Both can also be used if desired.

Note the script can also be passed to **dehydrated** directly
by using the **-k** or **\--hook** options. See the dehydrated documentation.

After the above lines are deployed in the wider dehydrated
config, that should allow one to solve the DNS-01 challenge
via **Luadns.com** with **dehydrated**!

# Using an Alternative Domain for Certificate Issue

The script supports using an alternative domain for certificate issue using a CNAME
record. To illustrate this, lets take an example.

Say one is seeking a certificate for highly critical domain (critial-example.org). In
this case one may not want to store an API key on the Webserver that has full access
to this domain. If the Webserver is compromised, then the domain should be taken
to be compromised also. If this is a critically important domain with lots of users and
possibly revenue dependent on it, then this could be an unacceptable security risk.

However, there is a way around this and **LetsEncrypt** or other free SSL certificate
providers accept this method also.

That is, to use a completely seperate and less important domain for the issue of
certificates instead. Expanding the example, lets say the less important domain is 
**non-critical-example.org**.

* Taking the example above - **critical-example.org**, let us say that one wants a certficate for
  **record1.critical.example.org**.   
* Then one can create a CNAME record (\_acme-challenge.record1.critical-example.org) at the DNS
  provider for **critical-example.org**  that points to **certificate-issue.non-critical-example.org**.    
* This way, the deployment of the TXT record for certificate issue can instead be
  done on **non-critical-example.org** instead of **critical-example.org**.    
* This gives the system admin some peace of mind to know that if the Webserver gets 
  compromised, only **non-critical-example.org** is at risk instead, as an API key can
  created and stored on the Webserver for that instead of **critical-example.org**.    

This gives additional flexibility as the critical domain does not require its DNS records
to be hosted at **Luadns.com**, they could instead be hosted elsewhere. All that is required
is that an appropriate CNAME record be created under the critically important domain.

Needless to say the non critical domain for use with this script must have its DNS hosted at Luadns.com.

This hook script has been written to account for this situation. It does a check up front
on any records to deploy or clean that it receives from **dehydrated** if they resolve as
CNAME records before being deployed or cleaned (removed).

If it finds a CNAME record, it will proceed to communicate with the Luadns.com REST API for
what the CNAME points to instead.

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

If **deploy_cert** is used as an **ACTION**, in this case the script will
issue the **deploy_cert_command** if specified, and also call the **deploy_cert_script**
if specified and pass in all of the arguments given by **dehyrated** [HERE](https://github.com/dehydrated-io/dehydrated/blob/master/docs/examples/hook.sh)

```bash
# To test the deploy_cert_command, call the script directly like so:
dehydrated-hook-luadns deploy_cert

# To test the deploy_cert_script, call the script and specify the
# arguments like below
dehydrated-hook-luadns deploy_cert "domain" "keyfile" "certfile" "fullchainfile" "chainfile" "timestamp"
```

# Sample Outputs

Shown below are some sample outputs that will be seen when being used by dehydrated.

```bash
root : server @ ~ # dehydrated-hook-luadns deploy_challenge test1.example.com file1 token-value
 + Hook: ############################
 + Hook: # Dehydrated-Hook-Luadns Version: 1.0.1
 + Hook: ############################
 + Hook: + deploy_challenge: 1 of 1
 + Hook: ############################
 + Hook: Name:_acme-challenge.test1.example.com, Type:TXT, Value:token-value
 + Hook: Zone Name: example.com
 + Hook: Querying Luadns Nameservers for example.com NS Records...
 + Hook: --> Querying Nameserver 1/4: ns1.luadns.net
 + Hook: ----> Success: Obtained 4 NS records for example.com
 + Hook: Deploying record via https://api.luadns.com/v1...
 + Hook: --> Getting Zone ID...
 + Hook: --> Zone ID: 1234
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
 + Hook: # Dehydrated-Hook-Luadns Version: 1.0.1
 + Hook: ############################
 + Hook: + clean_challenge: 1 of 1
 + Hook: ############################
 + Hook: Name:_acme-challenge.test1.example.com, Type:TXT, Value:token-value
 + Hook: Zone Name: example.com
 + Hook: Deleting record via https://api.luadns.com/v1...
 + Hook: --> Getting Zone ID...
 + Hook: --> Zone ID: 1234
 + Hook: --> Getting Record ID...
 + Hook: --> Record ID: 12345
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
tar xvf dehydrated-hook-luadns-1.0.1.tar.gz  # For the tar file
unzip dehydrated-hook-luadns-1.0.1.zip       # For the zip file

cd dehydrated-hook-luadns-1.0.1

# Now edit debian/control to delete the dehydrated depenedency
nano debian/control # or whatever text edit you prefer

# Delete the following line, save and close
 dehydrated (>=0.7.0),

# Then re-build the package like so
dh_make -s --createorig # Accept the default prompts
dpkg-buildpackage -uc -us
```

Then an updated package should be created in the parent directory.

# Links

## Luadns Rest API and Documentation

* [https://api.luadns.com/v1](https://api.luadns.com/v1)   
* [https://www.luadns.com/api.html](https://www.luadns.com/api.html)

## Dehydrated Documentation

* [https://github.com/dehydrated-io/dehydrated](https://github.com/dehydrated-io/dehydrated)    
* [https://github.com/dehydrated-io/dehydrated/blob/master/docs/dns-verification.md](https://github.com/dehydrated-io/dehydrated/blob/master/docs/dns-verification.md)    
* [https://github.com/dehydrated-io/dehydrated/blob/master/docs/hook\_chain.md](https://github.com/dehydrated-io/dehydrated/blob/master/docs/hook_chain.md)   
* [https://github.com/dehydrated-io/dehydrated/blob/master/docs/examples/hook.sh](https://github.com/dehydrated-io/dehydrated/blob/master/docs/examples/hook.sh)    

# Issues

Bug reports here on Github are welcome - don't hestitate if you find something wrong.

* [https://github.com/zoot101/dehydrated-hook-luadns/issues](https://github.com/zoot101/dehydrated-hook-luadns/issues)

# Credits

I came across a similar script written back in 2017 by Greg Brackley for use with [Luadns.com](https://luadns.com),
which is still hosted and available for download [HERE](https://plone.lucidsolutions.co.nz/web/pki/letsencrypt/letsencrypt-with-dehydrated-using-dns-01-on-centos-v7).

This script worked well for simple certificates with no alternative names, but had issues with
wildcard certificates and did not do any check for propagation. Although the above script is
entirely the author's work, this script did serve as a starting point - Many thanks to him.

And also many thanks to lukas2511 for [Dehydrated](https://github.com/dehydrated-io/dehydrated) itself!

