# Mail Server ansible roles

## Overview

This set of playbooks will turn a general Ubuntu LTS (16.04) host into a mail server. The project was started as a way to implement http://arstechnica.com/information-technology/2014/02/how-to-run-your-own-e-mail-server-with-your-own-domain-part-1/ in a simple, repeatable way.

At the moment, this only includes a primary MX (mail mailserver with mailboxes) role, but backup MX (stores mail when primary is down and delivers to primary when it's back up) is planned for RealSoonNow. 

This project is aimed at hobbyists who want to host their own mail, or groups who need to host mail for a large number of domains cheaply, but at the expense of limited management tools and a number of unfavourable security compromises.

### Known issues
A future version of this project will move mail domains, aliases and passwords to a database. Shortly after, a web application will be provided to manage this information.

#### passwd.db file
At the moment, users and passwords are stored in a single file that is used by dovecot. Password hashes must be generated prior to populating this file. This has a number of drawbacks:
   * Users must be able to run a command to generate a new password hash and provide this correctly to an admin
   * Users cannot change their password - someone with appropriate rights on the server has to do this
  
#### Lookup items
Items such as virtual-mailbox-domains, virtual users and other hashtables are currently stored in flat files within postfix configuration directories. This means that only host admins can change these. 

Once this project moves to a database for these items, it will be possible to allow domain administrators to make changes without requiring host admins to carry them out.

#### Single OpenDKIM key used to sign all domains
A single OpenDKIM key is generated and used to sign all domains. A future version will include per-domain keys. This makes it easier to change the key for a single domain, and is better practice.

---

## Requirements
The Ansible controller should be running the latest version from https://github.com/ansible/ansible and the target must be an Ubuntu 16.04 LTS. CentOS and RedHat Enterprise Linux 7 will be added once backup MX functionality is complete.

You will also need 
   * One or more domains that you control and can edit DNS for
   * A certificate for each domain
   * Intermediate and root certificates, depending on mail clients and SSL provider
   * Password hash and username for each user that you wish to accept mail delivery for


#### Python 2.7
Ansible is sadly still tied to Python 2.7, and some Linux distributions are excluding this now. To install this on your target host, run `sudo apt-get install -y python` as a user with rights to sudo to root.

#### Domains
You will need a domain and the ability to edit DNS for this domain. Specifically, you're going to need to be able to edit set the mailserver (MX) entries and entries for SPF and DKIM. Depending on your certificate provider, you may also need to be able to add entries for domain validation.

I use 123-reg as their control panel is simple and DNS hosting is free, but any other provider that allows you to modify DNS will suffice.


#### Certificates
I use StartSSL at the moment, but I am considering moving to LetsEncrypt once the other big outstanding items for this project are resolved. StartSSL provides a single download that includes your certificate, the intermediate certificate and a root certificate.

Key and certificate management is outside of the scope of this documentation, so please follow the instructions for your chosen certificate provider.

As an example, for a mailserver on mymailserver.mydomain.com, I would download a zip file from StartSSL that contains another zip file named `OtherServer.zip`. This in turn contains
   * root.crt - The root certificate
   * 1_Intermediate.crt - intermediate certificate
   * 2_mymailserver.mydomain.com.crt - our mailserver's certificate

These playbooks allow you to specify the location of these files, and will create a combined certificate as part of the deployment process.

Be warned - StartSSL charge a fee for revoking certificates. This means that while it's free up front, if you have a key compromised or need to revoke for any reason, there will be a cost. 

##### Generating key and CSR
This is not exhaustive, and you should follow your certificate provider's instructions. But this is how I create my key, generate my CSR and remove the password from my key.

> $ openssl req -newkey rsa:2048 -keyout mailserver.mydomain.com.key -out mailserver.mydomain.com.csr  
> $ openssl rsa -in mailserver.mydomain.com.key -out mailserver.mydomain.com.key.NOPASSWORD  
> $ mv mailserver.mydomain.com.key mailserver.mydomain.com.key.WITHPASSWORD  
> $ mv mailserver.mydomain.com.NOPASSWORD mailserver.mydomain.com.key  

These keys **must** be stored securely and protected.

#### Password hashes 
Each user requires an entry in the `passwd.db` file. This file uses the following format
`user@domain:generatedhash`

Users can generate a hash by running `doveadm pw -s SSHA512` which is available in the dovecot-core package. You will need to install this somewhere that your users can run the command.

An example entry for _myuser@mydomain.com_ would be
`myuser@mydomain.com:{SSHA512}OeR5ulGD3LZ0OHuj9muNqSvKB7hxsxnTquSd8AjK8QXrtOAGqxhxdRs093Czcua=`

**Removing the need for this step is a high-priority for this project.**

---

## Basic Usage
This assumes that all variables and templates are already exactly as you would like them. **This will not be the case!** For more information about customization and options available, *please* read the rest of this README.


#### Host to deploy 
For primary mail servers, you need to either edit your ansible inventory and create a new group named `primarymx` that includes all hosts that you want to run this against *and* you need to edit `primarymx.yml` and change the `myprimarymx` in the hosts line to the name of the host or group that you wish to use.

#### Deployment
Simply run `ansible-playbook primarymx.yml` and watch the output. 

---

## Variable files and templates 
### primarymx
#### variables - roles/primarymx/vars/main.yml
This role assumes that you are planning to host mail for a number of domains on a host that is not part of one of those domains. 

If you're hosting mail from a host in the same domain, remove the contents of the extra_mail_domains list.

In the examples below, we'll assume that we're hosting our mail on _primarymx.mydomain.com_ and that we want to host mail for _kfdfsj.uk_ and _hfjdhfufnmds.uk_

The following variables must be completed with information relevant to your environment:
   * fqdn - fully qualified domain name of host that will be the primary mx server (e.g. primarymx.mydomain.com). This needs to match the certificate secured in the _Requirements_ section
   * default_domain - domain part of fqdn above (e.g. mydomain.com)
   * extra_mail_domains - A list of domains that we will host mail for
   * cert_files_dir - the directory where your key and certificates (including root and intermediate certificates) will be placed
   * mail_cert_file and mail_key_file assume that you've used the fqdn of your host in the filename. Change these to explicit file names.

The remaining varaibles are optional, and can be left empty.
   * extra_relay_networks - list of networks that can send mail through this host without needing authentication. Ideally, you should limit this to as small a set of hosts as possible! 
   * intermediate_cert_file and root_cert_file can be removed if you're providing a combined certificate in your _mail_cert_file_ or if these are not needed. 

Finally, there is a section that must be left alone. 
   * unix_users - this is used to create the user required for mail hosting

##### templates - roles/primarymx/templates
##### etc_aliases
This is a map of accounts that the system may try to deliver mail to by default and the real account that the mail should instead go to. Some applications will send mail and assume that it will be delivered locally, so this is where you should ensure that this mail goes somewhere useful.

In this template, we'll assume that all local delivery should go to an address not hosted on this server, but within the _mydomain.com_ domain, since you're providing and managing these domains from there.

##### etc_postfix_canonical
This should be a map of user accounts on your mail server to 'real' mail accounts, either served by this host, or hosted elsewhere. 

_Example_
`www-data@{{ fqdn }} myuser@mydomain.com`

##### etc_postfix_virtual
This is used by postfix as the _virtual_alias_maps_ and is a way to have aliases without mailboxes that redirect mail to real accounts. 

If you have a real mailbox named _junk_ on every domain you host, and you want to send all mail addressed to _spamtrap_@thedomain to the _junk_ mailbox, you could replace the contents of the example template with the following
```
{% for domain in extra_mail_domains %}
spamtrap@{{ domain }} junk@{{ domain }} 
{% endfor %}
```

##### etc_postfix_virtual-mailbox-domains
This is a list of domains that the host will accept mail for. By default, all extra domains that you include in the vars file will be added here.

##### etc_postfix_virtual-mailbox-users
This is a 1-1 map of e-mail addresses and the virtual mailboxes that mail for these addresses will be delivered to. We need this because we're delivering mail to virtual mailboxes instead of real user accounts. 

By default, the template creates a webmaster address for every domain listed in the extra domains entry in the vars file. 

Any users that you create in the passwd.db file need to appear here in the format `user@domain   user@domain`

---

## Final Steps
#### Setting up DKIM for a domain
You need to add a TXT/SPF entry to your domain's DNS that includes your DKIM public key. The source for this can be found at /etc/opendkim/mail.txt. An example entry would be
```default._domainkey     IN TXT    "v=DKIM1; h=sha256; k=rsa; s=email; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADC432KBgQDfS9rm7OaBKKEmB4RfAf1dS4/8p+YMurNlF4vW1iqwy5M7/sA3kMpgjJTplmtej2CKotcileHyMI7zWx1tcnVQM5JccCrMAGuRH432i6AvRp45qmikdRX68MIfPypIYtvyYVL83ofdsaT0IDMSv1tfuom+pJ4H9x/GxHeiQjT6wIDAQAB"```

#### SPF 
An SPF record is the next thing we want to add to our DNS - more information about SPF can be found at https://en.wikipedia.org/wiki/Sender_Policy_Framework and there's a helpful wizard at http://spfwizard.com/

An example SPF record for kfdfsj.uk using mailserver.mydomain.com as a permitted sender would be
```@      IN TXT     "v=spf1 mx a:mailserver.mydomain.com"```